---
name: failure-learning-system
description: Log failures with structured context, categorize them, extract recurring patterns, and inject learned corrections into future attempts. Use when building an autonomous agent or pipeline that retries tasks, when someone says "the agent keeps making the same mistake", "add failure tracking", or needs an agent that learns from errors.
metadata:
  author: niyaznurbhasha
  version: 1.0.0
  category: agent-architecture
  tags: [failure-learning, error-tracking, retry, pattern-detection, autonomous-agent, resilience]
---

# Failure Learning System

Build a system where an autonomous agent logs every failure with structured context, categorizes errors into types, detects recurring patterns, and injects failure history into future prompts so the agent never repeats the same mistake.

## Architecture Overview

```
Task Execution
     |
     v (failure occurs)
Categorize Failure
     |
     v
Record to Failure Store (structured: type, context, error, attempt)
     |
     v
Pattern Detection (same type recurring? same goal stuck?)
     |
     v
Inject into Future Prompts
  - "Previous failures (don't repeat these): ..."
  - Trigger strategy change if pattern detected
```

## Step 1: Categorize Failures

Map raw error text into structured failure types. This enables aggregation and pattern detection:

```python
def categorize_failure(error_text: str) -> str:
    """Categorize an error string into a structured failure type."""
    lower = error_text.lower()

    if "oom" in lower or "out of memory" in lower or "cuda" in lower:
        return "gpu_oom"
    if "timeout" in lower:
        return "timeout"
    if "connection" in lower or "network" in lower or "ssh" in lower:
        return "network"
    if "permission" in lower or "access denied" in lower:
        return "permission"
    if "import" in lower or "module" in lower or "no module" in lower:
        return "import_error"
    if "file not found" in lower or "no such file" in lower:
        return "missing_file"
    if "json" in lower or "parse" in lower or "decode" in lower:
        return "parse_error"
    if "syntax" in lower or "indentation" in lower:
        return "syntax_error"
    if "rate limit" in lower or "429" in lower or "quota" in lower:
        return "rate_limit"

    return "unknown"
```

### Extend for Your Domain

Add domain-specific categories. The key is that each category implies a different remediation strategy:

```python
# Category -> default remediation
FAILURE_REMEDIATION = {
    "gpu_oom":      "Reduce batch size, use gradient checkpointing, or use a larger GPU",
    "timeout":      "Increase timeout, break task into smaller units",
    "network":      "Retry with backoff, check connectivity",
    "permission":   "Check file/service permissions, verify credentials",
    "import_error": "Install missing package, check environment activation",
    "missing_file": "Verify path exists, check if previous task created the file",
    "parse_error":  "Validate input format, add error handling",
    "rate_limit":   "Add delay between requests, reduce concurrency",
    "syntax_error": "Review generated code, run linter before execution",
    "unknown":      "Review full error log, escalate if recurring",
}
```

## Step 2: Record Failures with Context

Store failures with enough context to learn from them:

```python
def record_failure(goal: str, task_id: int, task_type: str,
                   description: str, error_text: str, attempt: int) -> None:
    """Record a structured failure for future learning."""
    failure_type = categorize_failure(error_text)

    # Store as a structured metric
    record_metric(goal, "failure", 1, {
        "task_id": task_id,
        "failure_type": failure_type,
        "error_excerpt": error_text[:300],
        "attempt": attempt,
        "task_type": task_type,
        "description": description[:200],
    }, task_id=task_id)

    # Also store as a memory entry (used in future prompt injection)
    if attempt >= 2:  # only persist after multiple failures
        upsert_memory(
            goal, f"failure_{task_id}",
            f"Task failed ({failure_type}): {description[:100]} -- {error_text[:200]}",
            confidence=0.8,
        )
```

### Failure Schema

```sql
CREATE TABLE failure_log (
    id SERIAL PRIMARY KEY,
    goal TEXT NOT NULL,
    task_id INT,
    failure_type TEXT NOT NULL,
    error_excerpt TEXT NOT NULL,
    description TEXT,
    attempt INT DEFAULT 1,
    task_type TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);
CREATE INDEX idx_failure_goal ON failure_log(goal, created_at DESC);
CREATE INDEX idx_failure_type ON failure_log(failure_type);

-- Also store learnings in a key-value memory table
CREATE TABLE memory (
    id SERIAL PRIMARY KEY,
    category TEXT NOT NULL,
    key TEXT NOT NULL,
    value TEXT NOT NULL,
    confidence FLOAT DEFAULT 0.8,
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(category, key)
);
```

## Step 3: Inject Failure History into Future Prompts

The core learning mechanism: every time a task is dispatched, its prompt includes recent failures for the same goal. The LLM sees what went wrong and avoids repeating it:

```python
def build_system_prompt(task: dict, role_intro: str) -> str:
    """Build system prompt with failure history injected."""
    parts = [role_intro]

    # Inject relevant learnings
    memory = get_memory(task["goal"])
    memory_lines = [f"- [{m['key']}] {m['value']}" for m in memory]
    if memory_lines:
        parts += ["", "Relevant learnings:", *memory_lines]

    # Inject failure history — THE KEY LEARNING MECHANISM
    failures = get_failure_history(task["goal"], limit=5)
    failure_lines = [
        f"- FAILED: {f['description'][:100]} -- {f['error_excerpt'][:100]}"
        for f in failures
    ]
    if failure_lines:
        parts += ["", "Previous failures (don't repeat these):", *failure_lines]

    return "\n".join(parts)
```

This single pattern — **injecting past failures into every future prompt** — is what makes the agent actually learn. Without it, the agent has no memory of what went wrong.

## Step 4: Retry with Exponential Backoff

Not all failures are permanent. Transient failures (network, rate limits) should be retried:

```python
RETRY_DELAYS = [30, 120, 600]  # seconds between retries
MAX_ATTEMPTS = len(RETRY_DELAYS) + 1

def handle_task_failure(task: dict, error_text: str) -> str:
    """Handle a failed task. Returns 'retry', 'failed', or 'escalate'."""
    attempt = task["attempt"]
    failure_type = categorize_failure(error_text)

    # Record the failure
    record_failure(
        goal=task["goal"], task_id=task["id"], task_type=task["type"],
        description=task["description"], error_text=error_text, attempt=attempt
    )

    # Transient failures get retried
    if failure_type in ("network", "timeout", "rate_limit") and attempt < MAX_ATTEMPTS:
        delay = RETRY_DELAYS[min(attempt - 1, len(RETRY_DELAYS) - 1)]
        update_task(task["id"], status="pending")
        time.sleep(delay)
        return "retry"

    # Non-transient failures: retry with reduced attempt budget
    if attempt < MAX_ATTEMPTS:
        delay = RETRY_DELAYS[min(attempt - 1, len(RETRY_DELAYS) - 1)]
        update_task(task["id"], status="pending")
        time.sleep(delay)
        return "retry"

    # Max attempts exceeded
    update_task(task["id"], status="failed", result=error_text)
    upsert_memory(
        task["goal"], f"failure_{task['id']}",
        f"Task permanently failed ({failure_type}): {task['description'][:100]} -- {error_text[:200]}",
        confidence=0.9,
    )

    # Check if this goal is stuck
    failed_count = count_failed_attempts(task["goal"])
    if failed_count >= 3:
        return "escalate"

    return "failed"
```

## Step 5: Detect Recurring Patterns

Aggregate failures to detect systemic issues:

```python
def detect_failure_patterns(goal: str) -> list[dict]:
    """Detect recurring failure patterns for a goal."""
    failures = get_failure_history(goal, limit=20)
    if not failures:
        return []

    # Count by type
    type_counts = {}
    for f in failures:
        ft = f["failure_type"]
        type_counts[ft] = type_counts.get(ft, 0) + 1

    patterns = []
    for ft, count in type_counts.items():
        if count >= 3:
            patterns.append({
                "type": ft,
                "count": count,
                "remediation": FAILURE_REMEDIATION.get(ft, "Unknown"),
                "severity": "high" if count >= 5 else "medium",
            })

    return patterns

def check_quality_plateau(goal: str) -> bool:
    """Check if quality scores have plateaued (no improvement over 3+ evaluations)."""
    scores = get_quality_trend(goal, limit=3)
    if len(scores) < 3:
        return False
    return max(scores) - min(scores) <= 1  # within 1 point = plateau
```

## Step 6: Trigger Strategy Changes

When patterns indicate the current approach is stuck, pivot:

```python
MAX_STRATEGY_CHANGES = 3

def maybe_change_strategy(goal: str, current_score: float,
                          recent_scores: list, score_details: dict) -> str:
    """Decide whether to iterate, change strategy, or give up."""
    patterns = detect_failure_patterns(goal)
    plateaued = check_quality_plateau(goal)
    failed_count = count_failed_attempts(goal)
    strategy_changes = count_strategy_changes(goal)

    # Still making progress -> keep iterating
    if not plateaued and failed_count < 3 and current_score >= 5:
        return "iterate"

    # Exhausted all strategies -> skip this goal
    if strategy_changes >= MAX_STRATEGY_CHANGES:
        log_action("goal_skipped",
                   f"score={current_score}, strategies_tried={strategy_changes}")
        upsert_memory(goal, "skipped",
                     f"Skipped after {strategy_changes} strategy changes. "
                     f"Best score: {max(recent_scores) if recent_scores else current_score}",
                     confidence=0.9)
        return "skip"

    # Stuck -> try a fundamentally different approach
    reseed_with_new_strategy(goal, {
        "current_score": current_score,
        "recent_scores": recent_scores,
        "failure_patterns": patterns,
        "issues": score_details.get("issues", []),
    })
    return "strategy_change"
```

## Step 7: Self-Review for Regressions

Periodically review recent changes to catch regressions that individual task results might miss:

```python
REVIEW_EVERY_N_TASKS = 5

def self_review(recent_tasks: list[dict]) -> dict:
    """Review recent completed tasks for conflicts and regressions."""
    prompt = (
        f"Review these {len(recent_tasks)} recently completed tasks:\n"
        + "\n".join(f"- {t['description'][:100]}" for t in recent_tasks)
        + "\n\nCheck for:\n"
        "1. Breaking changes\n"
        "2. Conflicting changes (two tasks edited same thing differently)\n"
        "3. Regressions (a fix undid something working)\n"
        "4. Dead code added\n\n"
        'Return JSON: {"ok": true/false, "issues": ["..."], "severity": "none|low|high"}'
    )
    result = call_llm(prompt)
    try:
        review = json.loads(result)
    except json.JSONDecodeError:
        review = {"ok": True, "issues": [], "severity": "none"}

    # Auto-create fix tasks for high-severity issues
    if review.get("severity") == "high":
        for issue in review.get("issues", [])[:3]:
            create_task(
                goal="self_review_fix",
                description=f"Fix review issue: {issue}",
                task_type="code",
                priority=200,  # high priority
            )

    return review
```

## Step 8: User Feedback Integration

Allow humans to inject learnings that bypass the automated detection:

```python
def handle_user_feedback(feedback: str, goal: str = "user_feedback") -> None:
    """Store user-provided learning with maximum confidence."""
    upsert_memory(goal, f"fb_{int(time.time())}", feedback, confidence=1.0)
    log_action("user_feedback", feedback)
```

## Complete Failure Handling Flow

Putting it all together in the main loop:

```python
def handle_result(task: dict, result: dict, tasks_since_review: int) -> int:
    """Process task result. Returns updated tasks_since_review counter."""

    if result.get("is_error"):
        action = handle_task_failure(task, result["result"])
        if action == "escalate":
            decision = maybe_change_strategy(
                task["goal"], current_score=0,
                recent_scores=[], score_details={}
            )
            log_action("escalation", f"goal={task['goal']}, decision={decision}")
        return tasks_since_review

    # Success path
    update_task(task["id"], status="done", result=result["result"][:4000])
    tasks_since_review += 1

    # Periodic self-review
    if tasks_since_review >= REVIEW_EVERY_N_TASKS:
        recent = list_tasks(status="done", limit=REVIEW_EVERY_N_TASKS)
        self_review(recent)
        tasks_since_review = 0

    # Handle evaluation scores
    if task["type"] == "evaluate":
        score_data = parse_score(result["result"])
        decision = maybe_change_strategy(
            task["goal"], score_data.get("score", 0),
            get_quality_trend(task["goal"]), score_data
        )

    return tasks_since_review
```

## Design Principles

1. **Failures are data, not just errors.** Every failure gets categorized, stored, and counted. This transforms random errors into actionable intelligence.

2. **Prompt injection is the learning mechanism.** The LLM has no persistent memory. The only way it "learns" is by seeing failure history in its system prompt. This means failure records must be concise and actionable.

3. **Categorization enables remediation.** A raw error string is noise. A failure type ("gpu_oom", "parse_error") implies a specific fix. Build your categories around what the remediation would be.

4. **Escalation beats infinite retry.** Three retries of the same approach with the same context will likely produce the same result. After N failures, escalate to a strategy change rather than retry.

5. **Strategy changes must be fundamental.** When re-seeding after a plateau, explicitly instruct the LLM to try a "fundamentally different approach, not incremental tweaks." Otherwise it generates slightly reworded versions of the same tasks.

6. **Self-review catches what individual tasks miss.** Two tasks might each succeed individually but conflict with each other. Periodic review catches regressions that no single task result would reveal.

7. **Human feedback at maximum confidence.** User-provided learnings get confidence=1.0 because humans see things the automated system cannot. Always provide a feedback channel.
