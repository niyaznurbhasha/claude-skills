---
name: journal-capture
description: Captures decisions, reflections, brainstorms, progress, and teardowns to a structured JSONL journal. Use when user makes a choice, has a realization, brainstorms ideas, ships something, or analyzes a system. Also use when user says "journal this", "log this", "capture this", or signals session end.
metadata:
  author: niyaznurbhasha
  version: 1.0.0
  category: productivity
  tags: [journal, capture, decisions, reflections, thinking-partner]
---

# Journal Capture

Automatically captures significant observations to a structured JSONL journal file.

## Setup

Set your journal path. Default: `data/journal.jsonl` in your project root.

## When to Capture

Trigger on any of these:
- A choice is made (architecture, approach, prioritization, scope, tools) → **decision**
- Something is learned or realized → **reflection**
- Ideas generated (features, strategies, directions) → **brainstorm**
- Something ships that users can actually interact with → **progress**
- A real system is studied in depth → **teardown**

## Type Rules

These are the most common mistakes — follow strictly:

- "Decided to write a plan / do research / create docs" → **decision** (not progress — nobody can use a doc yet)
- "Chose tech stack, architecture, library, approach" → **decision** always
- "Feature ships / new tab live / deployed" → **progress**
- "Wrote code or files" → NOT progress unless something real ships
- "Had a realization mid-build" → **reflection**
- "Strategic conversation about process" → **decision** (meta-decisions count)

## Instructions

### Step 1: Determine entry type

Match the observation to a type using the rules above. When in doubt, prefer decision over progress.

### Step 2: Identify the project

Tag with the appropriate project slug from your registry. Customize the table below for your projects:

| Slug | Name |
|------|------|
| my-app | My Application |
| tools | Internal Tools |
| research | Research |

### Step 3: Dedup check

Read last 5 lines of your journal file. Same type + same project + same topic → replace, don't append.

### Step 4: Write the entry

Append one JSON line to your journal file:

```json
{"id": "<8-char-hex>", "type": "<type>", "created_at": "<iso-datetime>", "tags": ["project:<slug>", ...], ...fields}
```

## Entry Schemas

### decision
```json
{
  "decision": "what was decided",
  "why": "reasoning",
  "expected_outcome": "what should happen",
  "might_be_missing": "blind spots"
}
```

### reflection
```json
{
  "title": "short title",
  "context": "situation",
  "concept": "key concept",
  "what_worked": "positives",
  "what_didnt": "negatives",
  "explanation": "key insight",
  "change": "what to do differently"
}
```

### brainstorm
```json
{
  "title": "short title",
  "content": "full content",
  "insights": ["insight1", "insight2"],
  "action_items": ["todo1", "todo2"]
}
```

### progress
```json
{
  "title": "short title",
  "what_shipped": "what's live",
  "impact": "why it matters",
  "next_steps": ["step1", "step2"]
}
```

### teardown
```json
{
  "system_name": "name",
  "source_url": "url",
  "problem": "what they solved",
  "constraints": "limitations",
  "tradeoffs": "design choices",
  "what_id_do": "personal take"
}
```

## Session End Sweep

When user signals wrapping up ("done", "thanks", "that's it"):
1. Scan back through conversation
2. Identify any uncaptured decisions, realizations, or strategic priorities
3. Write them before responding
