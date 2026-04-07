---
name: statistical-signal-detection
description: Time-series signal detection using z-scores, EWMA, Foster monotony, Mann-Kendall trend tests, PELT change points, Spearman correlation, and isolation forest. Use when analyzing any time-series data for anomalies, trends, regime shifts, or cross-domain correlations.
metadata:
  author: niyaznurbhasha
  version: 1.0.0
  category: time-series-analytics
  tags: [signal-detection, anomaly-detection, time-series, z-score, ewma, mann-kendall, pelt, changepoint, isolation-forest, spearman, trend-analysis]
---

# Statistical Signal Detection

Detects meaningful patterns in time-series data using a layered suite of statistical methods. Each method targets a different class of signal (point anomalies, monotonic trends, regime shifts, cross-series correlations, multivariate outliers). Results are ranked by criticality and score so downstream consumers (dashboards, LLMs, alerts) can prioritize.

## Architecture

### Signal dataclass

Every detected signal should carry:

```python
@dataclass
class Signal:
    id: str            # stable hash of (domain + key) for dedup across runs
    domain: str        # logical grouping (e.g., "sales", "infra", "health")
    key: str           # machine-readable name (e.g., "latency_spike")
    title: str         # human-readable one-liner with real numbers
    description: str   # what the data shows, with context and thresholds
    criticality: str   # "critical" | "notable" | "informational"
    score: float       # 0-1, higher = more important (for ranking)
    data_state: str    # hash of underlying values (for change detection between runs)
    metadata: dict     # extra key-value pairs for downstream use
```

Use `hashlib.md5(f"{domain}:{key}".encode()).hexdigest()[:12]` for stable IDs. Use a `data_state` hash so consumers can detect when a signal's underlying data changed between runs without re-running detection.

### Criticality weights

Assign weights for delivery priority:

```python
CRITICALITY_WEIGHT = {"critical": 10, "notable": 4, "informational": 1}
```

## Detection Methods (Step by Step)

### 1. Personal Z-Score (point anomaly)

Compares the most recent value against a rolling personal baseline.

```python
def personal_zscore(current: float, history: list[float], min_n: int = 7) -> float | None:
    if len(history) < min_n:
        return None
    mu = statistics.mean(history)
    sigma = statistics.stdev(history)
    if sigma < 0.001:
        return 0.0
    return (current - mu) / sigma
```

**When to fire:** |z| > 1.5 for "notable", |z| > 2.5 for "critical". Adjust thresholds per domain.

**Use for:** Any metric where "unusual compared to my own history" matters. Revenue, latency, engagement, biometrics.

### 2. EWMA Ratio (acute-vs-chronic load)

Exponentially weighted moving average comparing short-term vs long-term load. Originally from sports science (Hulin et al. 2017), generalizes to any workload metric.

```python
def ewma(series: list[float], lam: float) -> float:
    if not series:
        return 0.0
    val = series[0]
    for x in series[1:]:
        val = lam * x + (1 - lam) * val
    return val

# Short window (e.g., 7 days): lam_acute = 2 / (7 + 1)
# Long window (e.g., 28 days): lam_chronic = 2 / (28 + 1)
ratio = ewma_acute / ewma_chronic  # > 1.5 = danger, 0.8-1.3 = sweet spot
```

**Use for:** Any domain where sudden load increases relative to recent baseline indicate risk. Server traffic, order volume, training intensity.

### 3. Foster Monotony + Strain

Detects dangerously uniform workload patterns. High monotony (mean / stdev of daily load) means no variability, which impairs adaptation.

```python
daily_loads = [compute_daily_load(day) for day in recent_7_days]
mu = statistics.mean(daily_loads)
sigma = statistics.stdev(daily_loads)
monotony = mu / sigma          # > 2.0 = danger zone
strain = sum(daily_loads) * monotony  # total accumulated stress
```

**Use for:** Any repeating process where too-uniform input degrades output. Manufacturing, training programs, content publishing cadence.

### 4. Mann-Kendall Trend Test (monotonic trend)

Non-parametric test for monotonic trends. No distributional assumptions. Uses the `pymannkendall` library.

```python
import pymannkendall as mk

def mann_kendall_trend(series: list[float]) -> tuple[str, float]:
    if len(series) < 5:
        return "no trend", 1.0
    result = mk.original_test(series)
    return result.trend, result.p  # trend: "increasing"/"decreasing"/"no trend"
```

**When to fire:** p < 0.10 combined with meaningful magnitude change. Always pair with effect size (how much did it actually change?) to avoid flagging statistically significant but trivially small trends.

**Use for:** Any metric where directional drift matters. Revenue trends, error rates, engagement decay.

### 5. PELT Change Point Detection

Finds regime shifts in a time series where the statistical properties (mean, variance) change abruptly. Uses the `ruptures` library with RBF kernel.

```python
import ruptures as rpt
import numpy as np

def pelt_changepoint(series: list[float], min_size: int = 5) -> list[int]:
    if len(series) < min_size * 2:
        return []
    signal = np.array(series).reshape(-1, 1)
    model = rpt.Pelt(model="rbf", min_size=min_size, jump=1)
    model.fit(signal)
    bkps = model.predict(pen=3)  # pen controls sensitivity (lower = more breakpoints)
    return bkps[:-1]  # drop final sentinel
```

**Use for:** Detecting when a system entered a new regime. Pricing changes, infrastructure migrations, policy shifts, market structure breaks.

### 6. Spearman Cross-Domain Correlation

Non-parametric correlation between two time series. Detects monotonic relationships without assuming linearity.

```python
from scipy.stats import spearmanr

def spearman_corr(x: list[float], y: list[float]) -> tuple[float, float] | None:
    if len(x) < 10 or len(x) != len(y):
        return None
    rho, p = spearmanr(x, y)
    return float(rho), float(p)
```

**When to fire:** |rho| > 0.4 and p < 0.05. Consider lagged correlations (shift one series by N days) to find delayed effects.

**Use for:** Connecting metrics across domains. Sleep quality vs. productivity, marketing spend vs. signups, deploy frequency vs. incident rate.

### 7. Isolation Forest (multivariate anomaly)

Detects multivariate anomalies that look normal on any single dimension but are unusual as a combination. Gate behind a minimum data requirement (60+ observations) to avoid false positives.

```python
from sklearn.ensemble import IsolationForest
from sklearn.preprocessing import StandardScaler

def isolation_forest_score(feature_matrix: list[list[float]], min_rows: int = 60) -> list[float] | None:
    if len(feature_matrix) < min_rows:
        return None
    X = np.array(feature_matrix)
    X = StandardScaler().fit_transform(X)
    clf = IsolationForest(n_estimators=100, contamination=0.08, random_state=42)
    clf.fit(X)
    return clf.decision_function(X).tolist()  # negative = more anomalous
```

**When to fire:** Latest row's score < -0.15. Report which dimensions contribute.

**Use for:** Multi-signal health checks where individual metrics pass but the combination is suspicious. System monitoring, user behavior analysis, portfolio risk.

## Orchestration Pattern

### Step 1: Build individual detectors

Write one function per signal type. Each returns `Signal | None` or `list[Signal]`. Keep detectors independent so they can be tested and toggled individually.

### Step 2: Run all detectors

```python
def detect_signals(data: dict, config: dict) -> list[Signal]:
    signals: list[Signal] = []

    # Single-signal detectors (return Signal | None)
    for result in [
        detect_metric_anomaly(data),
        detect_load_ratio(data),
        detect_monotony(data),
        detect_trend(data),
    ]:
        if result:
            signals.append(result)

    # Multi-signal detectors (return list[Signal])
    signals.extend(detect_cross_domain(data))
    signals.extend(detect_composition_signals(data))

    # Dedup by id, sort by score DESC
    seen = set()
    unique = []
    for s in sorted(signals, key=lambda x: x.score, reverse=True):
        if s.id not in seen:
            seen.add(s.id)
            unique.append(s)

    return unique
```

### Step 3: Fingerprint for change detection

Generate a stable hash of the active signal set so consumers can detect when signals changed between runs without re-processing:

```python
def signals_fingerprint(signals: list[Signal]) -> str:
    parts = sorted(f"{s.id}:{s.criticality}:{s.data_state}" for s in signals)
    return hashlib.md5("|".join(parts).encode()).hexdigest()[:16]
```

## Design Principles

1. **Gate behind minimum data.** Every method has a `min_n` or `min_rows` guard. Never fire on sparse data.
2. **Pair statistical significance with effect size.** A p < 0.05 trend that moved 0.1% is noise. Always check magnitude alongside p-values.
3. **Descriptions include real numbers.** "Sleep declining" is useless. "Sleep declined from 7.2 to 5.1/10 over 14 days (p=0.03)" is actionable.
4. **Graceful degradation.** If `pymannkendall`, `ruptures`, `scipy`, or `sklearn` are not installed, log a warning and skip that method. Never crash.
5. **Stable IDs for dedup.** Hash domain + key so the same signal across runs has the same ID. Use `data_state` to detect when the underlying numbers changed.
6. **Cross-domain signals are the highest value.** Any single metric can be explained away. Two correlated metrics from different domains are a strong signal.

## Dependencies

- `pymannkendall` -- Mann-Kendall trend test
- `ruptures` -- PELT change point detection
- `scipy` -- Spearman correlation, statistical distributions
- `scikit-learn` -- Isolation Forest
- `numpy` -- array operations

All are optional with graceful fallback.
