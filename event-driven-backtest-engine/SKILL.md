---
name: event-driven-backtest-engine
description: Streaming event-driven backtesting engine with a pluggable strategy ABC, DuckDB partition pruning over Parquet data, settlement interleaving, and batch streaming. Use when building a backtesting system for any asset class (prediction markets, equities, crypto, sports betting).
metadata:
  author: niyaznurbhasha
  version: 1.0.0
  category: backtesting
  tags: [backtest, event-driven, strategy, duckdb, parquet, streaming, settlement, replay, trading]
---

# Event-Driven Backtest Engine

A streaming replay engine that reads historical trade data from hive-partitioned Parquet files via DuckDB, interleaves settlement events in chronological order, and dispatches all events to a pluggable strategy. The strategy sees events exactly as they would have arrived in real time (point-in-time safe).

## Architecture

### Event Types

Define a closed set of event types as frozen dataclasses:

```python
from dataclasses import dataclass
from datetime import datetime
from typing import Union

@dataclass(frozen=True)
class TradeEvent:
    timestamp: datetime
    market_id: str
    trade_id: str
    price: float
    size: float
    side: str           # "buy" | "sell"
    notional_value: float

@dataclass(frozen=True)
class MarketSettledEvent:
    timestamp: datetime
    market_id: str
    resolution: str     # outcome (e.g., "yes", "no", "win", "loss")

@dataclass(frozen=True)
class BacktestCompleteEvent:
    timestamp: datetime
    markets_seen: int
    trades_processed: int

Event = Union[TradeEvent, MarketSettledEvent, BacktestCompleteEvent]
```

Frozen dataclasses prevent strategies from accidentally mutating events. Use `Union` (not inheritance) to keep the type set explicit.

### Strategy ABC

Strategies implement a single method. This is the entire interface:

```python
from abc import ABC, abstractmethod

class Strategy(ABC):
    @abstractmethod
    def on_event(self, event: Event) -> None: ...
```

The strategy maintains its own internal state (positions, P&L, order book). The engine never inspects strategy state -- it only pushes events.

### Backtest Config

```python
from dataclasses import dataclass
from datetime import date
from pathlib import Path

@dataclass
class BacktestConfig:
    data_dir: Path          # root dir with hive-partitioned Parquet files
    metadata_db: Path       # SQLite/DB for market metadata + settlements
    start_date: date
    end_date: date
    market_ids: set[str] | None = None  # None = all markets
    batch_size: int = 10_000            # rows per fetchmany() call

    def __post_init__(self):
        if self.start_date > self.end_date:
            raise ValueError("start_date must be <= end_date")
        if self.batch_size < 1:
            raise ValueError("batch_size must be >= 1")
```

### Backtest Result

The engine tracks metadata; the strategy tracks its own P&L:

```python
@dataclass
class BacktestResult:
    trades_processed: int = 0
    markets_seen: set[str] = field(default_factory=set)
    markets_settled: set[str] = field(default_factory=set)
    markets_unresolved: set[str] = field(default_factory=set)
    errors: list[str] = field(default_factory=list)
```

## Engine Implementation

### Step 1: Pre-load settlements

Load all settlement records for the date range before streaming trades. Settlements are small (metadata), so loading them entirely into memory is safe:

```python
settlements = metadata_db.get_settlements_in_range(start_dt, end_dt, market_ids)
# Returns list of (timestamp, market_id, resolution) sorted by timestamp
settle_ptr = 0
```

### Step 2: Stream trades via DuckDB with partition pruning

```python
trades_glob = str(config.data_dir / "trades" / "**" / "*.parquet")

# Build SQL with hive partition pruning
filters = [
    f"date >= '{config.start_date}'",
    f"date <= '{config.end_date}'",
]
if config.market_ids:
    ids_sql = ", ".join(f"'{m}'" for m in sorted(config.market_ids))
    filters.append(f"market_id IN ({ids_sql})")

query = f"""
    SELECT timestamp, market_id, trade_id, price, size, side, notional_value
    FROM read_parquet('{trades_glob}', hive_partitioning=true)
    WHERE {" AND ".join(filters)}
    ORDER BY timestamp ASC
"""
```

### Step 3: Interleave settlements (point-in-time safe)

As trades stream in chronologically, emit any settlements that occurred at or before the current trade's timestamp. This ensures the strategy sees settlements in the correct temporal order:

```python
conn = duckdb.connect(":memory:")
try:
    cursor = conn.execute(query)
    while True:
        batch = cursor.fetchmany(config.batch_size)
        if not batch:
            break
        for row in batch:
            ts, market_id, trade_id, price, size, side, notional = row

            # Emit settlements that happened at or before this trade
            while (settle_ptr < len(settlements)
                   and settlements[settle_ptr][0] <= ts):
                s_ts, s_market, s_resolution = settlements[settle_ptr]
                safe_call(strategy, MarketSettledEvent(
                    timestamp=s_ts, market_id=s_market, resolution=s_resolution,
                ), result)
                result.markets_settled.add(s_market)
                settle_ptr += 1

            # Emit the trade
            safe_call(strategy, TradeEvent(
                timestamp=ts, market_id=market_id, trade_id=trade_id,
                price=float(price), size=float(size), side=side,
                notional_value=float(notional),
            ), result)
            result.trades_processed += 1
            result.markets_seen.add(market_id)
finally:
    conn.close()
```

### Step 4: Drain remaining settlements

After all trades are processed, emit any settlements that occurred after the last trade:

```python
while settle_ptr < len(settlements):
    s_ts, s_market, s_resolution = settlements[settle_ptr]
    safe_call(strategy, MarketSettledEvent(
        timestamp=s_ts, market_id=s_market, resolution=s_resolution,
    ), result)
    result.markets_settled.add(s_market)
    settle_ptr += 1

result.markets_unresolved = result.markets_seen - result.markets_settled
```

### Step 5: Emit completion event

```python
safe_call(strategy, BacktestCompleteEvent(
    timestamp=datetime.now(UTC),
    markets_seen=len(result.markets_seen),
    trades_processed=result.trades_processed,
), result)
```

### Error isolation

Never let a strategy exception crash the engine. Capture and log:

```python
def safe_call(strategy: Strategy, event: Event, result: BacktestResult) -> None:
    try:
        strategy.on_event(event)
    except Exception as exc:
        result.errors.append(f"{type(event).__name__}: {exc}")
```

## Writing a Strategy

```python
class MomentumStrategy(Strategy):
    def __init__(self):
        self.positions: dict[str, float] = {}
        self.pnl: float = 0.0
        self.trade_log: list[dict] = []

    def on_event(self, event: Event) -> None:
        match event:
            case TradeEvent():
                self._handle_trade(event)
            case MarketSettledEvent():
                self._handle_settlement(event)
            case BacktestCompleteEvent():
                self._finalize(event)

    def _handle_trade(self, event: TradeEvent) -> None:
        # Your logic: track prices, enter/exit positions, etc.
        pass

    def _handle_settlement(self, event: MarketSettledEvent) -> None:
        # Resolve positions, compute realized P&L
        pass

    def _finalize(self, event: BacktestCompleteEvent) -> None:
        # Compute final metrics, generate report
        pass
```

## Running a Backtest

```python
config = BacktestConfig(
    data_dir=Path("data/"),
    metadata_db=Path("data/metadata.db"),
    start_date=date(2024, 1, 1),
    end_date=date(2024, 6, 30),
    market_ids={"MARKET_A", "MARKET_B"},  # or None for all
    batch_size=10_000,
)

engine = ReplayEngine(config)
strategy = MomentumStrategy()
result = engine.run(strategy)

print(f"Processed {result.trades_processed} trades across {len(result.markets_seen)} markets")
print(f"Settled: {len(result.markets_settled)}, Unresolved: {len(result.markets_unresolved)}")
print(f"Errors: {len(result.errors)}")
```

## Design Principles

1. **Strategy knows nothing about storage.** The engine handles DuckDB, Parquet, and metadata. The strategy only sees typed events.
2. **Point-in-time safety.** Settlements are interleaved chronologically with trades. A strategy never sees a settlement before it would have been observable.
3. **Error isolation.** Strategy exceptions are captured, not propagated. The engine continues processing. Review `result.errors` after the run.
4. **Batch streaming.** `fetchmany(N)` keeps memory bounded regardless of dataset size. Tune `batch_size` for your workload.
5. **Immutable events.** Frozen dataclasses prevent strategies from mutating shared state.
6. **Engine is reusable.** Same engine instance can run multiple strategies sequentially (strategy is passed to `run()`, not `__init__()`).
7. **Timezone awareness.** Always attach UTC timezone to timestamps. If the source data lacks timezone info, explicitly set it.

## Dependencies

- `duckdb` -- streaming SQL over Parquet
- Python `dataclasses`, `datetime`, `pathlib` -- standard library
