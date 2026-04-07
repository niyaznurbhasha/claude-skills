---
name: parquet-storage-with-wal
description: Build a thread-safe time-series storage layer using an in-memory append buffer, JSON-lines WAL for crash recovery, atomic Parquet flush, and Hive-partitioned date splits with DuckDB for analytics. Use when storing high-throughput time-series data (market data, sensor readings, event streams, logs) that needs both fast writes and efficient analytical queries.
metadata:
  author: niyaznurbhasha
  version: 1.0.0
  category: data-pipelines
  tags: [parquet, wal, time-series, duckdb, storage, crash-recovery, hive-partitioning, analytics]
---

# Parquet Storage with WAL

A write-ahead log (WAL) backed append buffer that periodically flushes to Hive-partitioned Parquet files. Provides crash recovery, atomic writes, and DuckDB-powered analytics over the Parquet files.

## Architecture Overview

```
Write Path:
  record → WAL (JSON-lines, fsync) → in-memory buffer → Parquet flush (atomic rename)

Read Path:
  DuckDB → read_parquet('dir/**/*.parquet', hive_partitioning=true) → SQL queries

Partition Layout:
  data/
    exchange=polymarket/
      date=2026-01-15/
        orderbooks_1705312800000_a1b2c3d4.parquet
        orderbooks_1705316400000_e5f6a7b8.parquet
      date=2026-01-16/
        ...
```

## Step 1: Define Your Arrow Schema

Use PyArrow schemas to enforce types at the storage layer:

```python
import pyarrow as pa

EVENTS_SCHEMA = pa.schema([
    pa.field("timestamp", pa.timestamp("us", tz="UTC")),
    pa.field("event_id", pa.string()),
    pa.field("source", pa.string()),
    pa.field("category", pa.string()),
    pa.field("value", pa.float32()),
    pa.field("metadata", pa.string()),  # JSON string for flexible fields
])
```

## Step 2: Build the ParquetBuffer

The core component: an append buffer with WAL protection and periodic Parquet flushing.

```python
import json
import time
import uuid
from collections import defaultdict
from dataclasses import dataclass, field
from datetime import datetime
from pathlib import Path
from typing import Any

import pyarrow as pa
import pyarrow.parquet as pq


@dataclass
class ParquetBuffer:
    """
    Thread-safe append buffer with JSON-lines WAL for crash recovery.
    Flushes to hive-partitioned Parquet on interval or size threshold.
    """
    name: str                          # e.g., "events", "orderbooks"
    schema: pa.Schema
    output_dir: Path                   # where Parquet files land
    wal_path: Path                     # path to WAL file
    flush_interval_sec: int = 300      # flush every 5 minutes
    max_buffer_rows: int = 50_000      # flush if buffer exceeds this

    _buffer: list[dict] = field(default_factory=list)
    _last_flush: float = field(default_factory=time.time)
    _wal_fh: Any = field(default=None, repr=False)

    def __post_init__(self) -> None:
        self.output_dir.mkdir(parents=True, exist_ok=True)
        self.wal_path.parent.mkdir(parents=True, exist_ok=True)
        self._recover_from_wal()
        self._wal_fh = open(self.wal_path, "a")

    def append(self, record: dict) -> None:
        """Write to WAL first (crash safe), then buffer in memory."""
        self._wal_fh.write(json.dumps(record, default=str) + "\n")
        self._wal_fh.flush()
        self._buffer.append(record)

        if (
            len(self._buffer) >= self.max_buffer_rows
            or time.time() - self._last_flush >= self.flush_interval_sec
        ):
            self.flush()

    def flush(self) -> int:
        """Write buffer to Parquet. Returns number of rows written."""
        if not self._buffer:
            return 0

        rows = len(self._buffer)

        # WAL stores datetimes as ISO strings; coerce back before Arrow
        for rec in self._buffer:
            ts = rec.get("timestamp")
            if isinstance(ts, str):
                rec["timestamp"] = datetime.fromisoformat(ts)

        # Split by partition keys (e.g., source + date)
        partitions: dict[tuple, list] = defaultdict(list)
        for rec in self._buffer:
            ts = rec.get("timestamp")
            date_str = ts.strftime("%Y-%m-%d") if isinstance(ts, datetime) else "unknown"
            source = str(rec.get("source", "unknown"))
            partitions[(source, date_str)].append(rec)

        for (source, date_str), records in partitions.items():
            out_dir = self.output_dir / f"source={source}" / f"date={date_str}"
            out_dir.mkdir(parents=True, exist_ok=True)

            filename = out_dir / (
                f"{self.name}_{int(time.time() * 1000)}_{uuid.uuid4().hex[:8]}.parquet"
            )
            tmp = filename.with_suffix(".parquet.tmp")

            table = pa.Table.from_pylist(records, schema=self.schema)
            pq.write_table(table, tmp, compression="snappy")
            tmp.rename(filename)  # atomic on Linux/macOS

        # Reset buffer and truncate WAL
        self._buffer.clear()
        self._last_flush = time.time()
        if self._wal_fh:
            self._wal_fh.close()
        self._wal_fh = open(self.wal_path, "w")  # truncate

        return rows

    def _recover_from_wal(self) -> None:
        """On startup, replay any records from the WAL that weren't flushed."""
        if not self.wal_path.exists():
            return
        recovered = 0
        with open(self.wal_path) as f:
            for line in f:
                line = line.strip()
                if line:
                    try:
                        self._buffer.append(json.loads(line))
                        recovered += 1
                    except json.JSONDecodeError:
                        pass  # skip corrupt WAL lines
        if recovered:
            self.flush()

    def close(self) -> None:
        """Flush remaining buffer and close WAL file handle."""
        self.flush()
        if self._wal_fh:
            self._wal_fh.close()

    @property
    def pending_rows(self) -> int:
        return len(self._buffer)
```

Key design decisions:
- **WAL-first writes** — every record hits disk (JSON-lines) before memory, so a crash never loses data
- **Atomic rename** — write to `.parquet.tmp`, then `rename()` to final path; readers never see partial files
- **Hive partitioning** — `source=X/date=YYYY-MM-DD/` layout enables partition pruning in queries
- **Unique filenames** — millisecond timestamp + 8-char UUID hex prevents collisions on concurrent flushes

## Step 3: Model-to-Record Conversion

Convert your domain models into flat dicts for the buffer:

```python
def event_to_record(event) -> dict:
    """Convert a domain event object to a flat dict for ParquetBuffer."""
    return {
        "timestamp": event.timestamp,
        "event_id": event.id,
        "source": event.source,
        "category": event.category,
        "value": event.value,
        "metadata": json.dumps(event.extra) if event.extra else "{}",
    }

# Usage:
buffer.append(event_to_record(my_event))
```

## Step 4: DuckDB Analytics Layer

Query the Parquet files using DuckDB with automatic Hive partition pruning:

```python
import duckdb
from typing import Optional

class AnalyticsDB:
    """DuckDB query layer over hive-partitioned Parquet files.
    Read-only analytics — writes go through ParquetBuffer."""

    def __init__(self, data_dir: Path) -> None:
        self._data_dir = data_dir
        self._conn = duckdb.connect(":memory:")
        self._register_views()

    def _register_views(self) -> None:
        """Create DuckDB views over Parquet directories."""
        events_dir = self._data_dir / "events"
        if events_dir.exists() and any(events_dir.rglob("*.parquet")):
            self._conn.execute(f"""
                CREATE OR REPLACE VIEW events AS
                SELECT * FROM read_parquet(
                    '{events_dir}/**/*.parquet',
                    hive_partitioning=true
                )
            """)

    def refresh(self) -> None:
        """Re-register views after new Parquet files are written."""
        self._register_views()

    def query(self, sql: str) -> list[dict]:
        """Execute SQL and return results as list of dicts."""
        result = self._conn.execute(sql)
        cols = [desc[0] for desc in result.description]
        return [dict(zip(cols, row)) for row in result.fetchall()]

    def query_events(
        self,
        source: Optional[str] = None,
        start: Optional[datetime] = None,
        end: Optional[datetime] = None,
        limit: Optional[int] = None,
    ) -> list[dict]:
        """Query events with optional filters. DuckDB prunes partitions."""
        filters = []
        if source:
            filters.append(f"source = '{source}'")
        if start:
            filters.append(f"timestamp >= '{start.isoformat()}'")
        if end:
            filters.append(f"timestamp <= '{end.isoformat()}'")

        where = "WHERE " + " AND ".join(filters) if filters else ""
        limit_clause = f"LIMIT {limit}" if limit else ""
        return self.query(
            f"SELECT * FROM events {where} ORDER BY timestamp {limit_clause}"
        )

    def close(self) -> None:
        self._conn.close()
```

## Step 5: SQLite Metadata Store (Optional)

For storing non-time-series metadata (market info, device registry, config):

```python
import sqlite3

class MetadataDB:
    def __init__(self, db_path: Path) -> None:
        db_path.parent.mkdir(parents=True, exist_ok=True)
        self._conn = sqlite3.connect(str(db_path), check_same_thread=False)
        self._conn.row_factory = sqlite3.Row
        self._migrate()

    def _migrate(self) -> None:
        self._conn.execute("""
            CREATE TABLE IF NOT EXISTS sources (
                source_id   TEXT PRIMARY KEY,
                name        TEXT NOT NULL,
                status      TEXT DEFAULT 'active',
                last_seen   TEXT
            )
        """)
        self._conn.commit()

    def upsert(self, source_id: str, name: str, status: str = "active") -> None:
        self._conn.execute("""
            INSERT INTO sources (source_id, name, status, last_seen)
            VALUES (?, ?, ?, datetime('now'))
            ON CONFLICT(source_id) DO UPDATE SET
                status = excluded.status,
                last_seen = excluded.last_seen
        """, (source_id, name, status))
        self._conn.commit()

    def close(self) -> None:
        self._conn.close()
```

## Putting It All Together

```python
from pathlib import Path

DATA_DIR = Path("data")

# Initialize buffers
events_buffer = ParquetBuffer(
    name="events",
    schema=EVENTS_SCHEMA,
    output_dir=DATA_DIR / "events",
    wal_path=DATA_DIR / "wal" / "events.wal",
    flush_interval_sec=300,
    max_buffer_rows=50_000,
)

# Initialize analytics
analytics = AnalyticsDB(DATA_DIR)

# Write path
events_buffer.append(event_to_record(my_event))

# Read path (after flush)
events_buffer.flush()
analytics.refresh()
recent = analytics.query_events(source="sensor_1", limit=100)

# Cleanup
events_buffer.close()
analytics.close()
```

## Common Pitfalls

1. **datetime serialization roundtrip** — WAL stores datetimes as ISO strings via `json.dumps(default=str)`; you must coerce them back to `datetime` objects before creating Arrow tables
2. **Atomic rename only works same-filesystem** — `.tmp` file must be in the same directory as the final file
3. **WAL truncation timing** — Truncate WAL only AFTER successful Parquet flush, never before
4. **Midnight boundary** — Records near midnight span two dates; split by actual timestamp, not flush time
5. **DuckDB view refresh** — After writing new Parquet files, call `refresh()` before querying or you get stale data
6. **Corrupt WAL lines** — Always wrap WAL replay in try/except per line; a partial write from a crash should be skipped, not crash the recovery
7. **WSL2/NTFS performance** — Atomic rename works but is slow on NTFS-mounted drives; keep data on the Linux filesystem when possible
