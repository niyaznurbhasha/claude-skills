---
name: duckdb-analytics-layer
description: DuckDB + Parquet analytics with hive-partition pruning, streaming batch processing, and SQL-based filtering. Use when building any analytical workload on columnar data, market filtering, or high-performance OLAP queries over Parquet files.
metadata:
  author: niyaznurbhasha
  version: 1.0.0
  category: analytics
  tags: [duckdb, parquet, hive-partitioning, analytics, filtering, olap, columnar, batch-processing, market-filter]
---

# DuckDB Analytics Layer

Pattern for high-performance analytics over Parquet files using DuckDB. Covers hive-partition pruning for predicate pushdown, streaming batch reads to control memory, SQL-based universe filtering with composable criteria, and stats computation from raw columnar data.

## Core Pattern: Hive-Partitioned Reads with Pruning

### Directory layout

Structure Parquet files in hive-partition format so DuckDB can skip irrelevant partitions:

```
data/
  trades/
    exchange=binance/
      date=2024-01-01/
        part-001.parquet
      date=2024-01-02/
        part-001.parquet
    exchange=coinbase/
      date=2024-01-01/
        part-001.parquet
```

### Query with partition pruning

DuckDB's `read_parquet` with `hive_partitioning=true` pushes date/exchange filters down to file-level pruning -- only files matching the predicate are opened:

```python
import duckdb

conn = duckdb.connect(":memory:")
trades_glob = "data/trades/**/*.parquet"

# Partition columns (date, exchange) are available as regular columns
query = f"""
    SELECT timestamp, market_id, price, size
    FROM read_parquet('{trades_glob}', hive_partitioning=true)
    WHERE date >= '2024-01-01'
      AND date <= '2024-03-31'
      AND exchange = 'binance'
    ORDER BY timestamp ASC
"""
cursor = conn.execute(query)
```

### Dynamic filter construction

Build WHERE clauses programmatically from a config object:

```python
def build_where(config) -> str:
    filters = [
        f"date >= '{config.start_date}'",
        f"date <= '{config.end_date}'",
    ]
    if config.ids:
        ids_sql = ", ".join(f"'{m}'" for m in sorted(config.ids))
        filters.append(f"market_id IN ({ids_sql})")
    return " AND ".join(filters)
```

## Streaming Batch Processing

For large datasets that exceed memory, use `fetchmany` to process in controlled batches:

```python
cursor = conn.execute(query)
while True:
    batch = cursor.fetchmany(10_000)  # configurable batch size
    if not batch:
        break
    for row in batch:
        process(row)
```

This pattern streams rows from DuckDB without materializing the full result set. Tune `batch_size` based on per-row processing cost and available memory.

## Universe Filtering Pattern

When working with a large universe of entities (markets, stocks, sensors), apply systematic programmatic filtering before any analysis.

### Step 1: Define stats and filter config as dataclasses

```python
@dataclass
class EntityStats:
    entity_id: str
    avg_daily_volume: float
    avg_daily_count: float
    days_of_history: int
    avg_spread: float | None = None
    days_to_expiry: int | None = None
    latest_price: float | None = None

@dataclass
class FilterConfig:
    min_daily_volume: float = 500.0
    max_spread: float = 0.05
    min_daily_count: float = 10.0
    min_history_days: int = 7
    min_days_to_expiry: int = 3
    max_days_to_expiry: int = 180
    min_price: float = 0.03
    max_price: float = 0.97
```

### Step 2: Filter function collects ALL failures

Do not short-circuit. Collect every failure reason so the caller knows exactly why an entity was excluded:

```python
@dataclass
class FilterResult:
    entity_id: str
    passes: bool
    failures: list[str]

def filter_entity(stats: EntityStats, config: FilterConfig) -> FilterResult:
    failures = []

    if stats.avg_daily_volume < config.min_daily_volume:
        failures.append(
            f"avg_daily_volume={stats.avg_daily_volume:.0f} < min={config.min_daily_volume:.0f}"
        )

    if stats.avg_daily_count < config.min_daily_count:
        failures.append(
            f"avg_daily_count={stats.avg_daily_count:.1f} < min={config.min_daily_count:.1f}"
        )

    if stats.days_of_history < config.min_history_days:
        failures.append(
            f"days_of_history={stats.days_of_history} < min={config.min_history_days}"
        )

    if stats.avg_spread is not None and stats.avg_spread > config.max_spread:
        failures.append(
            f"avg_spread={stats.avg_spread:.4f} > max={config.max_spread:.4f}"
        )

    # Range filters: check both bounds
    if stats.days_to_expiry is not None:
        if stats.days_to_expiry < config.min_days_to_expiry:
            failures.append(f"days_to_expiry={stats.days_to_expiry} < min")
        elif stats.days_to_expiry > config.max_days_to_expiry:
            failures.append(f"days_to_expiry={stats.days_to_expiry} > max")

    if stats.latest_price is not None:
        if stats.latest_price < config.min_price:
            failures.append(f"latest_price={stats.latest_price:.4f} < min")
        elif stats.latest_price > config.max_price:
            failures.append(f"latest_price={stats.latest_price:.4f} > max")

    return FilterResult(entity_id=stats.entity_id, passes=len(failures) == 0, failures=failures)
```

### Step 3: Batch filter a universe

```python
def filter_universe(stats_list: list[EntityStats], config: FilterConfig | None = None):
    if config is None:
        config = FilterConfig()
    results = [filter_entity(s, config) for s in stats_list]
    passing = [s for s, r in zip(stats_list, results) if r.passes]
    return passing, results
```

## Stats Computation from Parquet

Compute per-entity statistics using DuckDB aggregate queries over hive-partitioned data:

```python
def compute_entity_stats(
    con: duckdb.DuckDBPyConnection,
    data_dir: str,
    entity_id: str,
    as_of: datetime,
    lookback_days: int = 30,
) -> EntityStats:
    since = as_of - timedelta(days=lookback_days)
    glob_path = f"{data_dir}/**/*.parquet"

    row = con.execute(f"""
        SELECT
            COUNT(DISTINCT date_trunc('day', timestamp)) AS days_with_data,
            COALESCE(SUM(size), 0.0)                     AS total_volume,
            COUNT(*)                                      AS total_count
        FROM read_parquet('{glob_path}', hive_partitioning=true)
        WHERE entity_id = ?
          AND timestamp >= ?
          AND timestamp <= ?
    """, [entity_id, since, as_of]).fetchone()

    days = max(int(row[0] or 0), 1)
    return EntityStats(
        entity_id=entity_id,
        avg_daily_volume=float(row[1] or 0) / days,
        avg_daily_count=int(row[2] or 0) / days,
        days_of_history=int(row[0] or 0),
    )
```

## Design Principles

1. **In-memory DuckDB.** Use `duckdb.connect(":memory:")` for analytics. No persistent database needed -- the Parquet files are the source of truth.
2. **Partition pruning is free performance.** Structure data as `key=value/` directories. DuckDB skips entire directory trees that don't match the WHERE clause.
3. **Batch fetch, don't materialize.** Use `fetchmany(N)` for streaming. Never `fetchall()` on unbounded queries.
4. **Collect all failures.** Filters should report every reason an entity failed, not just the first. This makes debugging filter configs trivial.
5. **Config as dataclass.** FilterConfig with sensible defaults lets callers override only what they need. Validate in `__post_init__`.
6. **Always close connections.** Use try/finally to ensure `conn.close()` even on exceptions.
7. **Glob patterns handle growth.** `**/*.parquet` with hive partitioning means new partitions (dates, exchanges) are automatically included without code changes.

## Dependencies

- `duckdb` -- analytical SQL engine
- `pyarrow` or `fastparquet` -- Parquet read support (DuckDB bundles its own reader, but these help for writing)
