---
name: event-study-stats-suite
description: Canonical event study statistical tests including BMP cross-sectional t-test, Kolari-Pynnonen adjusted test, Corrado rank test, bootstrap CARs, and Benjamini-Hochberg FDR correction. Use when running a formal event study in finance, economics, or social science and need to test whether abnormal returns (or any event-window metric) are statistically significant.
metadata:
  author: niyaznurbhasha
  version: 1.0.0
  category: analytics
  tags: [event-study, statistics, bmp-test, kolari-pynnonen, corrado-rank, bootstrap, fdr, benjamini-hochberg, abnormal-returns, car, caar]
---

# Event Study Stats Suite

A complete statistical testing toolkit for event studies. Covers the standard parametric test (BMP), its cross-correlation adjustment (Kolari-Pynnonen), a nonparametric alternative (Corrado rank test), a small-sample bootstrap, and multiple testing correction (Benjamini-Hochberg FDR).

## Data Model

### Market Model Estimation

Estimate the normal return relationship using OLS over an estimation window:

```python
@dataclass(frozen=True)
class MarketModelFit:
    alpha: float        # intercept
    beta: float         # market sensitivity
    sigma: float        # residual standard deviation
    r_squared: float    # model fit quality
    residuals: np.ndarray  # estimation-window residuals (needed for Kolari-Pynnonen)

def estimate_market_model(stock_rets: np.ndarray, market_rets: np.ndarray) -> MarketModelFit:
    n = len(stock_rets)
    X = np.column_stack([np.ones(n), market_rets])
    coeffs, _, _, _ = np.linalg.lstsq(X, stock_rets, rcond=None)
    alpha, beta = coeffs[0], coeffs[1]
    predicted = X @ coeffs
    resid = stock_rets - predicted
    ss_res = np.sum(resid ** 2)
    ss_tot = np.sum((stock_rets - np.mean(stock_rets)) ** 2)
    r_squared = 1.0 - ss_res / ss_tot if ss_tot > 0 else 0.0
    sigma = np.sqrt(ss_res / max(n - 2, 1))
    return MarketModelFit(alpha=float(alpha), beta=float(beta),
                          sigma=float(sigma), r_squared=float(r_squared),
                          residuals=resid)
```

### Abnormal Returns and CARs

```python
def compute_abnormal_returns(stock_rets, market_rets, fit):
    return stock_rets - (fit.alpha + fit.beta * market_rets)

@dataclass(frozen=True)
class CARResult:
    entity_id: str     # ticker, user, unit -- whatever the cross-section is
    event_id: str
    car: float         # cumulative abnormal return over event window
    sar: float         # standardized: CAR / (sigma * sqrt(window_length))
    daily_ar: np.ndarray
    fit: MarketModelFit
    est_days: int
    evt_start: int     # relative to event date (e.g., -1)
    evt_end: int       # relative to event date (e.g., +3)
```

Standardize CARs into SARs for cross-sectional tests: `SAR = CAR / (sigma * sqrt(event_window_length))`.

### CAAR (aggregate)

```python
@dataclass
class CAARResult:
    event_id: str
    window_spec: str
    caar: float               # mean of individual CARs
    cars: np.ndarray          # individual CARs
    sars: np.ndarray          # individual SARs
    n_firms: int
    included: list[str]
    excluded: list[str]       # entities dropped for insufficient data
```

## Statistical Tests

### 1. BMP Test (Boehmer-Musumeci-Poulsen 1991)

The standard cross-sectional t-test for event studies. Tests H0: mean SAR = 0 using the cross-sectional variance of SARs.

```python
def bmp_test(sars: np.ndarray) -> tuple[float, float]:
    n = len(sars)
    if n < 2:
        return 0.0, 1.0
    mean_sar = np.mean(sars)
    std_sar = np.std(sars, ddof=1)
    if std_sar == 0:
        return 0.0, 1.0
    t_stat = mean_sar * np.sqrt(n) / std_sar
    p_value = 2.0 * stats.t.sf(abs(t_stat), df=n - 1)
    return float(t_stat), float(p_value)
```

**When to use:** Default test for any event study. Robust to event-induced variance increases because it uses cross-sectional (not time-series) standard error.

### 2. Kolari-Pynnonen Test (2010)

Adjusts the BMP t-statistic for cross-correlation in estimation-window residuals. Critical when events are clustered (multiple entities share the same event date).

```python
def kolari_pynnonen_test(sars: np.ndarray, estimation_residuals: list[np.ndarray]) -> tuple[float, float]:
    n = len(sars)
    if n < 2:
        return 0.0, 1.0

    t_bmp, _ = bmp_test(sars)

    # Average pairwise correlation of estimation-window residuals
    min_len = min(len(r) for r in estimation_residuals)
    resid_matrix = np.column_stack([r[:min_len] for r in estimation_residuals])
    corr_matrix = np.corrcoef(resid_matrix, rowvar=False)
    n_entities = corr_matrix.shape[0]
    mask = np.triu(np.ones((n_entities, n_entities), dtype=bool), k=1)
    r_bar = float(np.mean(corr_matrix[mask]))

    adjustment = 1.0 - r_bar
    if adjustment <= 0:
        return 0.0, 1.0

    t_adj = t_bmp * np.sqrt(adjustment)
    p_value = 2.0 * stats.t.sf(abs(t_adj), df=n - 1)
    return float(t_adj), float(p_value)
```

**When to use:** Always use alongside BMP when events are clustered in calendar time. If events are spread across different dates, BMP alone suffices.

### 3. Corrado Rank Test (1989)

Nonparametric test that ranks abnormal returns within each entity's full series (estimation + event window). No distributional assumptions.

```python
def corrado_rank_test(full_ar_series: list[np.ndarray], event_window_indices: list[int]) -> tuple[float, float]:
    n_entities = len(full_ar_series)
    if n_entities < 2:
        return 0.0, 1.0

    evt_len = len(event_window_indices)
    rank_deviations = np.zeros((n_entities, evt_len))

    for i, ar_series in enumerate(full_ar_series):
        T = len(ar_series)
        ranks = stats.rankdata(ar_series)
        expected_rank = (T + 1) / 2.0
        std_rank = np.std(ranks, ddof=0)
        if std_rank == 0:
            continue
        for j, idx in enumerate(event_window_indices):
            if idx < T:
                rank_deviations[i, j] = (ranks[idx] - expected_rank) / std_rank

    mean_deviation = np.mean(rank_deviations, axis=0)
    z_stat = float(np.sum(mean_deviation) * np.sqrt(n_entities))
    p_value = 2.0 * stats.norm.sf(abs(z_stat))
    return z_stat, float(p_value)
```

**When to use:** As a robustness check alongside parametric tests. If BMP rejects but Corrado does not, the result may be driven by a few extreme outliers.

### 4. Bootstrap CAAR

Resamples individual CARs with replacement to build a distribution of CAAR under the null. Robust with small samples.

```python
def bootstrap_caar(cars: np.ndarray, n_boot: int = 10_000, seed: int = 42) -> tuple[float, float, tuple[float, float]]:
    n = len(cars)
    if n < 2:
        observed = float(np.mean(cars)) if n == 1 else 0.0
        return observed, 1.0, (observed, observed)

    rng = np.random.default_rng(seed)
    observed = float(np.mean(cars))

    boot_caars = np.empty(n_boot)
    for b in range(n_boot):
        sample = rng.choice(cars, size=n, replace=True)
        boot_caars[b] = np.mean(sample)

    # Two-sided p-value
    if observed >= 0:
        p_value = float(np.mean(boot_caars <= 0)) * 2
    else:
        p_value = float(np.mean(boot_caars >= 0)) * 2
    p_value = min(p_value, 1.0)

    ci_lower = float(np.percentile(boot_caars, 2.5))
    ci_upper = float(np.percentile(boot_caars, 97.5))
    return observed, p_value, (ci_lower, ci_upper)
```

**When to use:** Always, as it provides confidence intervals and handles non-normal CAR distributions. Especially valuable when N < 20.

### 5. Benjamini-Hochberg FDR Correction

When testing multiple events or windows, control the false discovery rate rather than the family-wise error rate:

```python
def benjamini_hochberg(p_values: list[float], alpha: float = 0.10) -> list[bool]:
    m = len(p_values)
    if m == 0:
        return []

    indexed = sorted(enumerate(p_values), key=lambda x: x[1])
    rejected = [False] * m

    max_k = -1
    for rank_idx, (orig_idx, p) in enumerate(indexed):
        k = rank_idx + 1
        threshold = k / m * alpha
        if p <= threshold:
            max_k = rank_idx

    if max_k >= 0:
        for rank_idx in range(max_k + 1):
            orig_idx = indexed[rank_idx][0]
            rejected[orig_idx] = True

    return rejected
```

**When to use:** Whenever you run multiple tests (multiple events, multiple windows, multiple test types). Collect all p-values, run BH, then report which survive correction.

## Recommended Test Battery

For each event, run all four tests and apply BH correction across events:

| Test | What it catches | Weakness |
|------|----------------|----------|
| BMP | Standard significance | Assumes no cross-correlation |
| Kolari-Pynnonen | Clustered events | Needs estimation residuals |
| Corrado | Non-normal returns | Lower power than parametric |
| Bootstrap | Small samples, CIs | Computationally heavier |

**Confirmation rule:** Consider an event "confirmed" when both KP-adjusted BMP AND Corrado reject at the FDR-corrected level. This ensures robustness to both cross-correlation and distributional violations.

## Dependencies

- `numpy` -- array operations, OLS
- `scipy` -- t-distribution, normal distribution, rankdata
