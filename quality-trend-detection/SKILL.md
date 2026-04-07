---
name: quality-trend-detection
description: Track quality metrics over time across evaluation runs, detect improvements/regressions/plateaus, and alert on drift. Use when user wants to monitor LLM quality over time, detect quality regressions, set up quality gates, track whether changes actually improved things, or automate responses to quality plateaus.
metadata:
  author: niyaznurbhasha
  version: 1.0.0
  category: evaluation-and-quality
  tags: [quality, metrics, trends, regression, drift, monitoring, plateau, alerting, time-series]
---

# Quality Trend Detection

Track quality scores across evaluation runs over time. Detect plateaus, regressions, and improvements. Trigger automated responses (strategy changes, alerts, escalations) based on trend patterns. Ties quality data back to code changes so you know what caused shifts.

## When to Use This

- You run evals regularly and want to know if quality is improving or degrading
- You need automated reactions when quality plateaus (e.g., try a different approach)
- You want to correlate quality changes with code/config changes
- You need quality gates that block deploys when scores regress

## Core Architecture

```
[Eval Run] --> [Score] --> [Record Metric] --> [Trend Store]
                                                    |
                                          [Trend Analyzer]
                                           /      |      \
                                    [Plateau] [Regression] [Achievement]
                                        |          |            |
                                  [Strategy    [Alert /     [Log success /
                                   Change]     Revert]      advance goal]
```

## Step 1: Metric Storage

Store every quality score with metadata linking it to the goal, task, and recent code changes. Use SQLite, Postgres, or even a JSONL file.

### Schema

```sql
CREATE TABLE IF NOT EXISTS metrics (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    goal TEXT NOT NULL,          -- what is being evaluated (e.g., "advice_quality")
    task_id TEXT,                -- which eval task produced this score
    metric_type TEXT NOT NULL,   -- "quality_score", "latency_ms", "cost_usd", etc.
    value REAL NOT NULL,
    metadata TEXT,               -- JSON: task details, code changes, config
    created_at REAL NOT NULL
);

CREATE TABLE IF NOT EXISTS runs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    pipeline TEXT NOT NULL,
    config_json TEXT,
    quality_score REAL,
    notes TEXT,
    created_at REAL NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_metrics_goal_type ON metrics(goal, metric_type);
CREATE INDEX IF NOT EXISTS idx_runs_pipeline ON runs(pipeline);
```

### Recording metrics

```python
import json
import time
import sqlite3
from pathlib import Path

DB_PATH = Path("quality_metrics.db")

def get_conn():
    conn = sqlite3.connect(str(DB_PATH))
    conn.row_factory = sqlite3.Row
    return conn

def record_metric(goal: str, metric_type: str, value: float,
                  metadata: dict = None, task_id: str = None):
    """Record a single metric data point."""
    conn = get_conn()
    conn.execute(
        "INSERT INTO metrics (goal, task_id, metric_type, value, metadata, created_at) "
        "VALUES (?, ?, ?, ?, ?, ?)",
        (goal, task_id, metric_type, value,
         json.dumps(metadata) if metadata else None, time.time()),
    )
    conn.commit()
    conn.close()

def record_run(pipeline: str, quality_score: float,
               config_json: str = None, notes: str = None) -> int:
    """Record an evaluation run with its overall score."""
    conn = get_conn()
    cur = conn.execute(
        "INSERT INTO runs (pipeline, config_json, quality_score, notes, created_at) "
        "VALUES (?, ?, ?, ?, ?)",
        (pipeline, config_json, quality_score, notes, time.time()),
    )
    run_id = cur.lastrowid
    conn.commit()
    conn.close()
    return run_id
```

### Tying scores to code changes

When recording a quality score, include recent code changes as metadata. This creates an audit trail for diagnosing regressions.

```python
def record_quality_with_context(goal: str, score: float, task_id: str,
                                 score_details: dict = None):
    """Record quality score with recent code change context."""
    # Get recent changes from your changelog/git log
    recent_changes = get_recent_changes(goal, limit=3)
    changed_files = []
    for change in recent_changes:
        if change.get("files_changed"):
            changed_files.extend(change["files_changed"])

    record_metric(goal, "quality_score", score, {
        "task_id": task_id,
        "details": score_details,
        "recent_code_changes": changed_files[:10],
    }, task_id=task_id)
```

## Step 2: Trend Retrieval

```python
def get_quality_trend(pipeline: str, limit: int = 5) -> list[float]:
    """Get recent quality scores for a pipeline, newest first."""
    conn = get_conn()
    rows = conn.execute(
        "SELECT quality_score FROM runs "
        "WHERE pipeline = ? AND quality_score IS NOT NULL "
        "ORDER BY id DESC LIMIT ?",
        (pipeline, limit),
    ).fetchall()
    conn.close()
    return [r["quality_score"] for r in rows]

def get_metrics(goal: str = None, metric_type: str = None,
                limit: int = 50) -> list[dict]:
    """Query metrics with optional filters."""
    conn = get_conn()
    clauses, params = [], []
    if goal:
        clauses.append("goal = ?")
        params.append(goal)
    if metric_type:
        clauses.append("metric_type = ?")
        params.append(metric_type)

    where = f"WHERE {' AND '.join(clauses)}" if clauses else ""
    rows = conn.execute(
        f"SELECT * FROM metrics {where} ORDER BY created_at DESC LIMIT ?",
        params + [limit],
    ).fetchall()
    conn.close()
    return [dict(r) for r in rows]
```

## Step 3: Trend Detection

### Plateau detection

A plateau means quality has stopped improving despite continued effort. Trigger a strategy change.

```python
def detect_plateau(scores: list[float], tolerance: float = 1.0) -> bool:
    """Detect if recent scores are plateaued (within tolerance of each other).

    Args:
        scores: Recent scores, newest first. Needs at least 3.
        tolerance: Max spread to consider a plateau.

    Returns:
        True if scores are plateaued.
    """
    if len(scores) < 3:
        return False
    return (max(scores) - min(scores)) <= tolerance
```

### Regression detection

A regression means the latest score is significantly worse than the recent trend.

```python
def detect_regression(scores: list[float], threshold: float = 0.1) -> bool:
    """Detect if the latest score regressed vs. recent average.

    Args:
        scores: Recent scores, newest first. Needs at least 2.
        threshold: Minimum drop (as fraction) to flag regression.

    Returns:
        True if the latest score is a regression.
    """
    if len(scores) < 2:
        return False
    latest = scores[0]
    previous_avg = sum(scores[1:]) / len(scores[1:])
    if previous_avg == 0:
        return False
    drop = (previous_avg - latest) / previous_avg
    return drop > threshold
```

### Achievement detection

Quality crossed a meaningful threshold.

```python
def detect_achievement(score: float, goal_threshold: float) -> bool:
    """Check if quality has reached the target level."""
    return score >= goal_threshold
```

### Declining trend detection

Not just a single regression, but a sustained downward trend.

```python
def detect_declining_trend(scores: list[float], window: int = 3) -> bool:
    """Detect if scores have been declining for N consecutive evaluations.

    Args:
        scores: Recent scores, newest first.
        window: Number of consecutive declines to flag.
    """
    if len(scores) < window + 1:
        return False
    # Check if each score is lower than its predecessor (reversed for newest-first)
    for i in range(window):
        if scores[i] >= scores[i + 1]:
            return False
    return True
```

## Step 4: Automated Responses

Wire trend detection into your eval pipeline to trigger automated actions.

```python
import logging

logger = logging.getLogger(__name__)

# Configuration
QUALITY_THRESHOLD = 7.0      # Score that means "good enough"
MAX_STRATEGY_CHANGES = 3     # Max pivots before skipping a goal
PLATEAU_WINDOW = 3           # Number of scores to check for plateau

def handle_quality_score(goal: str, score: float, task_id: str,
                          score_details: dict = None) -> str:
    """Process a new quality score and take appropriate action.

    Returns action taken: "done", "strategy_change", "skipped",
    "regression_alert", or "continue".
    """
    # Record the score
    record_quality_with_context(goal, score, task_id, score_details)
    record_run(pipeline=goal, quality_score=score,
               notes=json.dumps(score_details) if score_details else None)

    logger.info("Quality score for %s: %.1f", goal, score)

    # Achievement: target quality reached
    if detect_achievement(score, QUALITY_THRESHOLD):
        logger.info("Goal %s achieved target quality (score %.1f)!", goal, score)
        return "done"

    # Get trend
    recent_scores = get_quality_trend(goal, limit=PLATEAU_WINDOW)

    # Regression: latest score dropped significantly
    if detect_regression(recent_scores):
        logger.warning("Quality REGRESSION for %s: %.1f (recent avg: %.1f)",
                       goal, score,
                       sum(recent_scores[1:]) / max(len(recent_scores) - 1, 1))
        alert_regression(goal, score, recent_scores)
        return "regression_alert"

    # Plateau: quality stuck
    plateaued = detect_plateau(recent_scores)
    if plateaued:
        logger.warning("Quality plateaued for %s: scores %s", goal, recent_scores)

    # Decide on action based on score + trend
    strategy_changes = count_strategy_changes(goal)
    failed_attempts = count_failed_attempts(goal)

    if plateaued or score < (QUALITY_THRESHOLD * 0.7) or failed_attempts >= 3:
        if strategy_changes >= MAX_STRATEGY_CHANGES:
            # Exhausted all strategies -- skip this goal
            logger.warning("Goal %s exhausted %d strategies. Skipping.",
                           goal, strategy_changes)
            return "skipped"

        # Try a fundamentally different approach
        logger.info("Goal %s stuck -- triggering strategy change #%d",
                    goal, strategy_changes + 1)
        trigger_strategy_change(goal, score, recent_scores, score_details)
        return "strategy_change"

    return "continue"
```

### Strategy change implementation

When quality plateaus, generate a new approach using context about what has already been tried.

```python
def trigger_strategy_change(goal: str, current_score: float,
                             recent_scores: list[float],
                             score_details: dict = None):
    """Generate a new strategy based on what has been tried and failed."""
    # Gather context about previous attempts
    previous_strategies = get_memory(goal, "strategies_tried")
    issues = score_details.get("issues", []) if score_details else []

    context = (
        f"Goal: {goal}\n"
        f"Current score: {current_score}, Recent scores: {recent_scores}\n"
        f"Issues identified: {issues}\n"
        f"Previously tried: {previous_strategies}\n\n"
        f"Generate a fundamentally different approach. "
        f"Do not retry anything from the 'previously tried' list."
    )

    # Use LLM to generate new strategy
    new_strategy = call_llm(context)

    # Store the new strategy
    save_memory(goal, "strategies_tried",
                f"{previous_strategies}; Strategy #{len(previous_strategies)+1}: {new_strategy[:200]}")

    # Create tasks for the new strategy
    create_tasks_from_strategy(goal, new_strategy)
```

### Regression alerting

```python
def alert_regression(goal: str, score: float, recent_scores: list[float]):
    """Alert when quality regresses. Includes context for diagnosis."""
    # Get what changed recently
    recent_changes = get_recent_changes(goal, limit=5)

    alert = {
        "type": "quality_regression",
        "goal": goal,
        "current_score": score,
        "previous_avg": sum(recent_scores[1:]) / max(len(recent_scores) - 1, 1),
        "recent_scores": recent_scores,
        "recent_changes": recent_changes,
        "message": (
            f"Quality regression detected for '{goal}': "
            f"score dropped to {score} from avg {sum(recent_scores[1:])/max(len(recent_scores)-1,1):.1f}. "
            f"Recent changes: {[c.get('description') for c in recent_changes[:3]]}"
        ),
    }

    logger.warning(alert["message"])
    # Extend with your notification system: Slack, email, dashboard, etc.
    send_alert(alert)
```

## Step 5: Metrics Dashboard / CLI

Provide a way to view metrics and trends.

```python
def print_metrics_summary(goal: str = None):
    """Print aggregated metrics summary."""
    metrics = get_metrics(goal=goal, metric_type="quality_score", limit=100)

    if not metrics:
        print("No metrics recorded yet.")
        return

    # Group by goal
    by_goal = {}
    for m in metrics:
        by_goal.setdefault(m["goal"], []).append(m["value"])

    print(f"\nQuality Trends:")
    print(f"{'Goal':<25} {'Latest':>8} {'Avg':>8} {'Min':>8} {'Max':>8} {'N':>5} {'Trend':>10}")
    print("-" * 75)

    for goal_name, values in sorted(by_goal.items()):
        latest = values[0]  # newest first
        avg = sum(values) / len(values)
        trend = "---"
        if len(values) >= 3:
            if detect_plateau(values[:3]):
                trend = "PLATEAU"
            elif detect_regression(values[:3]):
                trend = "REGRESS"
            elif detect_declining_trend(values[:5]):
                trend = "DECLINING"
            elif values[0] > values[1]:
                trend = "IMPROVING"
            else:
                trend = "stable"

        print(f"  {goal_name:<23} {latest:>6.1f}  {avg:>6.1f}  "
              f"{min(values):>6.1f}  {max(values):>6.1f}  {len(values):>4}  {trend:>10}")
```

## Step 6: Quality Gates

Block deployments or goal progression when quality regresses.

```python
def quality_gate(goal: str, min_score: float = None,
                 no_regression: bool = True) -> tuple[bool, str]:
    """Check if quality meets gate criteria.

    Returns:
        (passed, reason) tuple.
    """
    recent = get_quality_trend(goal, limit=5)

    if not recent:
        return False, "No quality data available"

    latest = recent[0]

    # Minimum score gate
    if min_score and latest < min_score:
        return False, f"Score {latest:.1f} below minimum {min_score:.1f}"

    # Regression gate
    if no_regression and len(recent) >= 2:
        if detect_regression(recent):
            prev_avg = sum(recent[1:]) / len(recent[1:])
            return False, (f"Regression detected: {latest:.1f} vs "
                          f"previous avg {prev_avg:.1f}")

    return True, f"Quality gate passed: score {latest:.1f}"


# Usage in CI/CD or deployment scripts
passed, reason = quality_gate("advice_quality", min_score=7.0)
if not passed:
    print(f"BLOCKED: {reason}")
    sys.exit(1)
```

## Integration with Eval Frameworks

After each eval run, feed scores into the trend system:

```python
def post_eval_hook(eval_results: list, goal: str):
    """Call after an eval run to record metrics and check trends."""
    for result in eval_results:
        record_metric(
            goal=goal,
            metric_type="quality_score",
            value=result["scores"]["overall"],
            metadata={
                "task_id": result["task_id"],
                "category": result.get("test_category"),
                "scores": result["scores"],
            },
            task_id=result["task_id"],
        )

    # Record aggregate
    avg_score = sum(r["scores"]["overall"] for r in eval_results) / len(eval_results)
    action = handle_quality_score(goal, avg_score, task_id="aggregate")

    if action == "regression_alert":
        print(f"WARNING: Quality regression detected for {goal}")
    elif action == "strategy_change":
        print(f"INFO: Strategy change triggered for {goal}")
    elif action == "done":
        print(f"SUCCESS: {goal} achieved target quality!")
```

## Configuration Reference

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `QUALITY_THRESHOLD` | 7.0 | Score that means "good enough" (goal achieved) |
| `MAX_STRATEGY_CHANGES` | 3 | Max strategy pivots before skipping a goal |
| `PLATEAU_WINDOW` | 3 | Number of recent scores to check for plateau |
| `PLATEAU_TOLERANCE` | 1.0 | Max spread in scores to count as plateau |
| `REGRESSION_THRESHOLD` | 0.1 | Min fractional drop to flag regression |
| `DECLINE_WINDOW` | 3 | Consecutive declines to flag a trend |

## Anti-Patterns to Avoid

- **No code change context** -- recording scores without linking to what changed makes regressions impossible to diagnose. Always include recent commits/changes in metadata.
- **Reacting to single-point noise** -- one low score is not a regression. Require 2+ data points or a significant drop before triggering alerts.
- **Never changing strategy** -- if quality has plateaued for 3+ evaluations, the current approach is not working. Automate the pivot.
- **Infinite strategy changes** -- cap the number of strategy pivots. If 3 fundamentally different approaches all plateau, the problem is likely deeper (bad data, wrong model, wrong eval criteria).
- **Only tracking aggregate scores** -- always track per-dimension scores too. An aggregate improvement can hide a safety regression.
- **No persistence** -- in-memory metrics are useless. Write to SQLite at minimum so you can analyze trends across sessions.
