---
name: event-study-full-pipeline
description: End-to-end event study pipeline covering event catalog definition, price fetching, date alignment, CAR computation across multiple windows, full statistical test battery (BMP, Kolari-Pynnonen, Corrado, bootstrap), BH-FDR correction, reversion analysis, and formatted reporting. Use when running a complete empirical event study from raw events to publication-ready results.
metadata:
  author: niyaznurbhasha
  version: 1.0.0
  category: analytics
  tags: [event-study, pipeline, car, caar, reversion, statistical-tests, finance, empirical, cli]
---

# Event Study Full Pipeline

Orchestrates the complete event study workflow: define an event catalog, fetch and align price data, compute CARs across sensitivity windows, run a battery of statistical tests, apply multiple-testing correction, optionally measure post-event reversion, and format results for reporting.

## Pipeline Overview

```
1. DEFINE  → Event catalog + universe of affected entities
2. FETCH   → Download/load price data for all entities + benchmark
3. ALIGN   → Match entity dates to benchmark dates (common trading days)
4. COMPUTE → CARs per entity-event pair, across multiple windows
5. TEST    → BMP, Kolari-Pynnonen, Corrado, Bootstrap per event
6. CORRECT → Benjamini-Hochberg FDR across all p-values
7. REPORT  → Formatted summary table + optional per-entity detail
8. REVERT  → (Optional) Paired selloff-recovery reversion analysis
```

## Step 1: Define Events and Universe

### Event catalog

Each event needs an ID, date, description, affected entities, and expected direction:

```python
@dataclass
class EventDef:
    event_id: str
    date: date
    name: str
    category: str
    description: str
    affected_entities: list[str]  # tickers, firms, units
    expected_direction: int       # -1 = negative, +1 = positive, 0 = unknown
```

Build the catalog as a list. Can come from a database, YAML file, or code:

```python
def build_catalog() -> list[EventDef]:
    return [
        EventDef("evt_001", date(2024, 3, 15), "Regulation Announced",
                 "regulatory", "New compliance requirements...",
                 ["FIRM_A", "FIRM_B", "FIRM_C"], expected_direction=-1),
        # ...
    ]
```

### Universe

Define the full set of entities to study, with grouping metadata:

```python
@dataclass
class UniverseEntry:
    ticker: str
    sector: str
    group: str  # sub-classification for cross-sectional analysis
```

## Step 2: Fetch Prices

Load adjusted close prices for all entities and the benchmark. Cache results to avoid redundant fetches:

```python
def load_prices(db, tickers: list[str]) -> dict[str, tuple[list[date], np.ndarray]]:
    result = {}
    for ticker in tickers:
        rows = db.query("SELECT date, adj_close FROM prices WHERE ticker = ? ORDER BY date", [ticker])
        if rows:
            dates = [r["date"] for r in rows]
            prices = np.array([r["adj_close"] for r in rows], dtype=float)
            result[ticker] = (dates, prices)
    return result
```

Always validate data before proceeding:
- Benchmark (e.g., SPY) must be present
- Log which tickers are missing data
- Require minimum history (e.g., 200 trading days) per entity

## Step 3: Align Dates

Entities and the benchmark may have different trading calendars. Build aligned arrays using only common dates:

```python
def align_prices(entity_dates, entity_prices, benchmark_dates, benchmark_prices):
    bench_map = {d: i for i, d in enumerate(benchmark_dates)}
    common_e, common_b = [], []
    for i, d in enumerate(entity_dates):
        if d in bench_map:
            common_e.append(i)
            common_b.append(bench_map[d])

    if len(common_e) < 200:  # minimum data requirement
        return None

    aligned_entity = np.array([entity_prices[i] for i in common_e])
    aligned_bench = np.array([benchmark_prices[i] for i in common_b])
    aligned_dates = [entity_dates[i] for i in common_e]
    return aligned_dates, aligned_entity, aligned_bench
```

### Finding the event date index

Map the event date to the nearest trading day at or after the event:

```python
def find_event_idx(dates: list[date], event_date: date) -> int | None:
    for i, d in enumerate(dates):
        if d >= event_date:
            return i
    return None
```

## Step 4: Compute CARs

Run the market model estimation + CAR computation for each entity-event pair. Use the functions from the event-study-stats-suite skill.

### Sensitivity windows

Always test multiple event windows to check robustness:

```python
WINDOWS = {
    "[-1,+1]": (-1, 1),   # tight: 3-day window
    "[-1,+3]": (-1, 3),   # standard: 5-day window
    "[0,+1]":  (0, 1),    # post-only: 2-day
    "[0,+3]":  (0, 3),    # post-only: 4-day
    "[0,+5]":  (0, 5),    # extended: 6-day
}
```

### Per-event computation

For each event, iterate over affected entities:

```python
def run_single_event(event, entities, price_data, bench_dates, bench_prices, evt_start, evt_end):
    car_results = []
    est_residuals = []
    full_ar_list = []

    for entity_id in entities:
        if entity_id not in price_data:
            continue

        alignment = align_prices(price_data[entity_id][0], price_data[entity_id][1],
                                 bench_dates, bench_prices)
        if alignment is None:
            continue
        aligned_dates, aligned_entity, aligned_bench = alignment

        evt_idx = find_event_idx(aligned_dates, event.date)
        if evt_idx is None:
            continue

        result = compute_car(
            aligned_entity, aligned_bench,
            ticker=entity_id, event_id=event.event_id,
            event_idx=evt_idx, est_days=120, buffer=20,
            evt_start=evt_start, evt_end=evt_end,
        )
        if result is not None:
            car_results.append(result)
            est_residuals.append(result.fit.residuals)

            # Full AR series for Corrado test
            entity_rets = np.diff(aligned_entity) / aligned_entity[:-1]
            bench_rets = np.diff(aligned_bench) / aligned_bench[:-1]
            full_ar = compute_abnormal_returns(entity_rets, bench_rets, result.fit)
            full_ar_list.append(full_ar)

    return car_results, est_residuals, full_ar_list
```

## Step 5: Run Statistical Tests

For each event with at least 2 entity results, run the full test battery:

```python
caar_result = compute_caar(car_results, event.event_id, window_spec)

t_bmp, p_bmp = bmp_test(caar_result.sars)
t_kp, p_kp = kolari_pynnonen_test(caar_result.sars, est_residuals)
z_corr, p_corr = corrado_rank_test(full_ar_list, event_window_indices)
obs_boot, p_boot, (ci_lo, ci_hi) = bootstrap_caar(caar_result.cars)
```

Collect all p-values across events for FDR correction:

```python
all_p_values.extend([p_bmp, p_kp, p_corr, p_boot])
p_value_keys.extend([
    (event.event_id, "bmp"),
    (event.event_id, "kp"),
    (event.event_id, "corrado"),
    (event.event_id, "boot"),
])
```

## Step 6: Apply BH-FDR Correction

After processing all events, apply Benjamini-Hochberg across the full set of p-values:

```python
bh_rejected = benjamini_hochberg(all_p_values, alpha=0.10)

# Map back to events
for event_data in all_event_data:
    eid = event_data["event"].event_id
    indices = [i for i, (e, _) in enumerate(p_value_keys) if e == eid]
    rejections = {p_value_keys[i][1]: bh_rejected[i] for i in indices}
    event_data["bmp_rejected"] = rejections.get("bmp", False)
    event_data["kp_rejected"] = rejections.get("kp", False)
    # ...
```

### Confirmation rule

Count an event as "confirmed" when both KP-adjusted BMP AND Corrado reject at the FDR-corrected level:

```python
confirmed = sum(1 for r in results if r["kp_rejected"] and r["corrado_rejected"])
```

## Step 7: Report Results

Format a summary table with one row per event:

```
Event             | CAAR   | N  | BMP t | BMP p | KP t | KP p | Corr z | Corr p | Boot p | Boot CI
Legal Massacre    | -8.2%  | 12 | -3.41 | 0.006 | -2.89| 0.014| -2.55  | 0.011  | 0.002  | [-11.1%, -5.3%]
```

Include significance markers from BH correction (e.g., asterisks or "REJECT" flags).

## Step 8: Reversion Analysis (Optional)

For events with negative expected direction, measure whether entities that sold off subsequently recover.

### Paired window approach

Use the SAME pre-event market model for both the selloff and recovery windows. This ensures abnormal returns are measured against the same baseline:

```python
SELLOFF_WINDOW = (-1, 3)
RECOVERY_WINDOWS = {
    "[+5,+10]": (5, 10),
    "[+5,+15]": (5, 15),
    "[+5,+20]": (5, 20),
}

# 1. Compute selloff CAR to get the fit
selloff = compute_car(prices, benchmark, ticker, event_id, event_idx,
                      est_days=120, buffer=20, evt_start=-1, evt_end=3)

# 2. Reuse the SAME fit for recovery windows
for win_name, (rs, re) in RECOVERY_WINDOWS.items():
    recovery = compute_car_with_fit(prices, benchmark, fit=selloff.fit,
                                     ticker, event_id, event_idx,
                                     evt_start=rs, evt_end=re)
    if recovery:
        reversion_ratio = -recovery.car / selloff.car  # positive = reverting
```

### Filter criteria for reversion

- Only include entities with negative selloff CAR (they actually sold off)
- Require minimum selloff magnitude (e.g., |CAR| > 2%)
- Group results by sector/category for cross-sectional insights

### Statistical test for reversion

Test whether the mean reversion ratio is significantly greater than zero:

```python
from scipy.stats import ttest_1samp
t_stat, p_val = ttest_1samp(reversion_ratios, 0)
```

## Design Principles

1. **Multiple windows for robustness.** If a result only appears in one window, it may be noise. Consistent significance across windows strengthens the finding.
2. **Minimum N requirements.** Skip events with fewer than 2 entities. Statistical tests are meaningless with N=1.
3. **Align on common dates.** Never assume entity and benchmark have identical trading calendars. Always compute the intersection.
4. **Save intermediate results.** Persist individual CARs to a database so you can re-run statistical tests without re-computing the market model.
5. **BH correction is non-negotiable.** With multiple events and multiple tests, uncorrected p-values will produce false discoveries. Always apply FDR correction before interpreting results.
6. **Reversion reuses the same model.** The recovery window must use the same pre-event estimation to make the selloff and recovery CARs comparable.
7. **Separate data fetching from analysis.** Fetching is slow and network-dependent. Cache aggressively and validate data completeness before running the study.

## Dependencies

- `numpy` -- array operations, OLS
- `scipy` -- statistical distributions, rankdata, t-tests
- A price database (SQLite, Postgres, or any DB with adjusted close prices)
- Optional: `typer` for CLI interface, `structlog` for structured logging
