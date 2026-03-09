---
name: goal-task-decomposition
description: Break high-level goals into executable subtasks with dependency tracking, priority scheduling, type-based dispatch, and state machine lifecycle. Use when building an autonomous agent, orchestrator, or task runner that needs to decompose goals into work items, when someone says "break this into tasks", "build a task queue", or needs an agent that can plan and execute multi-step work.
metadata:
  author: niyaznurbhasha
  version: 1.0.0
  category: agent-architecture
  tags: [goal-decomposition, task-queue, orchestrator, state-machine, autonomous-agent, priority]
---

# Goal-Task Decomposition

Build an autonomous system that decomposes high-level goals into typed, prioritized subtasks with dependency ordering, dispatches them to specialized executors, and tracks lifecycle state.

## Architecture Overview

```
Goals (human-defined, high-level)
  |
  v
Seed Phase (LLM decomposes goal -> task list)
  |
  v
Task Queue (priority-ordered, typed, stateful)
  |
  v
Dispatcher (routes by type to specialized executors)
  |
  v
Result Handler (success -> done, failure -> retry/escalate)
  |
  v
Score/Evaluate (measure quality, decide: iterate or done)
```

## Step 1: Define Goals

Goals are high-level objectives defined by the user. Each goal has a priority, a seed prompt that tells the LLM how to decompose it, and an ID for tracking:

```python
GOALS = [
    {
        "id": "improve_search",
        "priority": 10,  # higher = more important
        "seed": (
            "Analyze the current search pipeline. Identify performance bottlenecks, "
            "missing features, and ranking quality issues. "
            "Break down into implementation tasks as JSON: "
            '[{"description": "...", "type": "research|code|run|evaluate", "priority": N}]'
        ),
    },
    {
        "id": "add_auth",
        "priority": 8,
        "seed": (
            "Design and implement user authentication. Consider OAuth, JWT, "
            "session management, and rate limiting. "
            "Break down into implementation tasks as JSON: "
            '[{"description": "...", "type": "research|code|run|evaluate", "priority": N}]'
        ),
    },
]
```

### Goal Design Rules

- **Seed prompts end with a JSON format instruction.** The LLM must return structured tasks, not prose.
- **Priority is a band, not a rank.** Tasks within a goal get sub-priorities. Goal priority * 10 + task position = effective priority.
- **Goals are persistent.** They live in config or DB, not in ephemeral state.

## Step 2: Define Task Types and Lifecycle

Every task has a type that determines its executor and a status that tracks its lifecycle:

```python
# Task types — each maps to a different execution strategy
TASK_TYPES = {
    "research": "LLM reads/searches to gather information. Output is text analysis.",
    "code":     "LLM edits source files. Use for any task requiring code changes.",
    "run":      "Execute a shell command. Description must include the exact command.",
    "evaluate": "Quality evaluation — run tests, generate previews, score output.",
}

# Task lifecycle states
# pending -> in_progress -> done
#                        -> failed -> (retry as pending, or final failure)
#                        -> blocked (waiting for approval or dependency)
TASK_STATUSES = ["pending", "in_progress", "done", "failed", "blocked"]
```

### Task Schema

```sql
CREATE TABLE tasks (
    id SERIAL PRIMARY KEY,
    goal TEXT NOT NULL,
    description TEXT NOT NULL,
    type TEXT NOT NULL,         -- research, code, run, evaluate
    priority INT DEFAULT 0,
    status TEXT DEFAULT 'pending',
    attempt INT DEFAULT 0,
    result TEXT,
    session_id TEXT,            -- for resumable sessions
    cost_usd REAL DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    started_at TIMESTAMP,
    completed_at TIMESTAMP
);
```

## Step 3: Seed Goals into Tasks

Use an LLM to decompose each goal into concrete tasks. This is the intelligence layer:

```python
def seed_goal(goal: dict) -> list[dict]:
    """Use an LLM to decompose a goal into concrete tasks."""
    # Inject any relevant memory/context
    memory = get_memory(goal["id"])
    memory_ctx = "\n".join(f"- {m['key']}: {m['value']}" for m in memory)

    prompt = (
        f"{goal['seed']}\n\n"
        f"Relevant context:\n{memory_ctx}\n\n" if memory_ctx else f"{goal['seed']}\n\n"
        "Return ONLY a JSON array. No prose.\n"
        "Format: [{\"description\": \"specific actionable task\", "
        "\"type\": \"research|code|run|evaluate\", "
        "\"priority\": 1-10}]"
    )

    result = call_llm(prompt)
    tasks = parse_task_list(result)

    if not tasks:
        log_action("seed_goal_empty", goal["id"])
        return []

    # Sort: research first, then code, then run, evaluate last
    type_order = {"research": 0, "code": 1, "run": 2, "evaluate": 3}
    tasks.sort(key=lambda t: type_order.get(t.get("type", "research"), 0))

    # Create tasks with effective priority = goal_band + position
    created = []
    for i, t in enumerate(tasks):
        effective_priority = goal["priority"] * 10 + (len(tasks) - i)
        task_id = create_task(
            goal=goal["id"],
            description=t["description"],
            task_type=t.get("type", "research"),
            priority=effective_priority,
        )
        created.append(task_id)

    return created
```

### Robust JSON Parsing

LLMs often wrap JSON in markdown fences or add prose. Handle it:

```python
def parse_task_list(text: str) -> list[dict]:
    """Extract JSON task array from LLM response."""
    cleaned = text.strip()
    # Strip markdown fences
    if cleaned.startswith("```"):
        cleaned = cleaned.split("\n", 1)[1] if "\n" in cleaned else cleaned[3:]
        cleaned = cleaned.rsplit("```", 1)[0].strip()

    # Try direct parse
    try:
        data = json.loads(cleaned)
        if isinstance(data, list):
            return data
        if isinstance(data, dict) and "tasks" in data:
            return data["tasks"]
    except (json.JSONDecodeError, TypeError):
        pass

    # Find JSON array in text via bracket matching
    idx = text.find("[")
    if idx == -1:
        return []
    depth = 0
    for i, ch in enumerate(text[idx:], idx):
        if ch == "[": depth += 1
        elif ch == "]":
            depth -= 1
            if depth == 0:
                try:
                    return json.loads(text[idx:i+1])
                except json.JSONDecodeError:
                    break
    return []
```

## Step 4: Dispatch by Type

Route each task to its specialized executor:

```python
DISPATCH = {
    "research": dispatch_research,
    "code":     dispatch_code,
    "run":      dispatch_run,
    "evaluate": dispatch_evaluate,
}

def dispatch_research(task: dict) -> dict:
    """LLM reads and analyzes — returns findings."""
    system = build_system_prompt(task, "You are analyzing the codebase. Return structured findings.")
    return call_llm_with_tools(task["description"], system_prompt=system, max_turns=25)

def dispatch_code(task: dict) -> dict:
    """LLM makes targeted code changes."""
    system = build_system_prompt(task, "You are making targeted code changes. Verify changes work.")
    return call_llm_with_tools(task["description"], system_prompt=system, max_turns=25)

def dispatch_run(task: dict) -> dict:
    """Execute a shell command."""
    desc = task["description"]
    if "Run:" not in desc:
        # No explicit command — reclassify as code task
        update_task(task["id"], type="code")
        return dispatch_code(task)

    cmd = desc.split("Run:", 1)[1].strip()
    proc = subprocess.run(cmd, shell=True, capture_output=True, text=True, timeout=1800)
    return {
        "result": proc.stdout[-2000:] if proc.returncode == 0 else proc.stderr[-2000:],
        "is_error": proc.returncode != 0,
    }

def dispatch_evaluate(task: dict) -> dict:
    """Run quality evaluation and return scores."""
    return call_llm_with_tools(task["description"], max_turns=5)
```

### Inject Context into Every Dispatch

Every dispatch should include relevant memory and failure history:

```python
def build_system_prompt(task: dict, role_intro: str) -> str:
    """Build system prompt with memory, failure history, and coding principles."""
    memory = get_memory(task["goal"])
    memory_lines = [f"- [{m['key']}] {m['value']}" for m in memory]

    failures = get_failure_history(task["goal"], limit=5)
    failure_lines = [f"- FAILED: {f['description'][:100]} -- {f['result'][:100]}"
                     for f in failures]

    parts = [role_intro]
    if memory_lines:
        parts += ["", "Relevant learnings:", *memory_lines]
    if failure_lines:
        parts += ["", "Previous failures (don't repeat these):", *failure_lines]
    return "\n".join(parts)
```

## Step 5: Handle Results with Retry Logic

```python
RETRY_DELAYS = [30, 120, 600]  # exponential backoff in seconds

def handle_result(task: dict, result: dict) -> None:
    attempt = task["attempt"]

    if result.get("is_error"):
        error_text = result["result"][:2000]

        if attempt < len(RETRY_DELAYS) + 1:
            delay = RETRY_DELAYS[min(attempt - 1, len(RETRY_DELAYS) - 1)]
            update_task(task["id"], status="pending")  # back to queue
            time.sleep(delay)
        else:
            # Final failure — record the learning
            update_task(task["id"], status="failed", result=error_text)
            upsert_memory(
                task["goal"], f"failure_{task['id']}",
                f"Task failed: {task['description'][:100]} -- Error: {error_text[:200]}",
                confidence=0.8,
            )
    else:
        update_task(task["id"], status="done", result=result["result"][:4000])
```

## Step 6: Dedup Similar Tasks

Before dispatching, check if a similar task was already completed:

```python
def find_similar_done_task(description: str, goal: str) -> dict | None:
    """Check if a semantically similar task was already done for this goal."""
    done_tasks = list_tasks(status="done", goal=goal)
    for t in done_tasks:
        # Simple overlap check — upgrade to embedding similarity if needed
        if _word_overlap(description, t["description"]) > 0.7:
            return t
    return None

# In the main loop:
similar = find_similar_done_task(task["description"], task["goal"])
if similar:
    update_task(task["id"], status="done",
                result=f"Deduped: similar to task {similar['id']}")
    continue
```

## Step 7: The Main Loop

```python
def run_loop():
    """Main orchestrator loop."""
    while True:
        # 1. Seed goals that have no pending tasks
        for goal in GOALS:
            if count_tasks_for_goal(goal["id"]) == 0:
                seed_goal(goal)

        # 2. Pick highest-priority pending task
        task = get_next_task()  # ORDER BY priority DESC, created_at ASC
        if not task:
            time.sleep(10)
            continue

        # 3. Dedup check
        similar = find_similar_done_task(task["description"], task["goal"])
        if similar:
            update_task(task["id"], status="done",
                        result=f"Deduped: similar to task {similar['id']}")
            continue

        # 4. Dispatch
        update_task(task["id"], status="in_progress",
                    attempt=task["attempt"] + 1)
        dispatcher = DISPATCH.get(task["type"])
        if not dispatcher:
            update_task(task["id"], status="failed",
                        result=f"Unknown type: {task['type']}")
            continue

        result = dispatcher(task)

        # 5. Handle result
        handle_result(task, result)
```

## Step 8: Strategy Changes When Stuck

When a goal's quality score plateaus or tasks keep failing, re-seed with a fundamentally different approach:

```python
MAX_STRATEGY_CHANGES = 3

def reseed_with_new_strategy(goal_id: str, context: dict) -> None:
    """Clear pending tasks and re-seed with a different approach."""
    # Mark all pending tasks as failed
    for t in list_tasks(status="pending", goal=goal_id):
        update_task(t["id"], status="failed", result="Cleared for strategy change")

    prompt = (
        f"STRATEGY CHANGE for goal '{goal_id}'.\n\n"
        f"Previous approach stalled. Context: {json.dumps(context)}\n\n"
        "Try a FUNDAMENTALLY DIFFERENT approach -- not incremental tweaks.\n"
        "Return ONLY a JSON array of tasks."
    )
    result = call_llm(prompt)
    tasks = parse_task_list(result)
    for i, t in enumerate(tasks):
        create_task(goal=goal_id, description=t["description"],
                    task_type=t.get("type", "code"), priority=50 + len(tasks) - i)
```

## Design Principles

1. **Goals are decomposed by LLM, not hardcoded.** The LLM sees the codebase and context, so it produces better task lists than static decomposition.

2. **Type-based dispatch keeps executors simple.** Each executor handles one type of work. The dispatcher routes. Adding a new type is adding one function.

3. **Priority = goal_band + position.** This ensures high-priority goals' tasks run first, and within a goal, tasks run in dependency order (research before code before evaluate).

4. **Memory and failure history flow into every dispatch.** The LLM never repeats a mistake because previous failures are injected into its system prompt.

5. **Dedup before dispatch.** Decomposition can produce overlapping tasks. Check before executing to avoid wasted work.

6. **Strategy changes, not infinite retries.** After N failed attempts or a quality plateau, don't keep trying the same approach. Re-seed the goal with explicit instructions to try something fundamentally different.

7. **Auto-commit after code tasks.** Each code task gets its own git commit. If a later task breaks something, you can revert to the exact point before it.
