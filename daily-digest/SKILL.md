---
name: daily-digest
description: Generates a personalized daily briefing from project data. Use when user says "daily digest", "morning briefing", "what should I focus on", "give me my digest", "what's important today". Requires discovery-dashboard for full functionality.
compatibility: Best with discovery-dashboard (github.com/niyaznurbhasha/discovery-dashboard). Works standalone with any JSONL journal.
metadata:
  author: niyaznurbhasha
  version: 1.0.0
  category: productivity
  tags: [digest, briefing, daily, priorities]
---

# Daily Digest

Generates a concise daily briefing by reading journal entries, project data, and open tasks to surface the highest-priority items.

## Instructions

### Step 1: Read recent journal entries

Read your journal file (JSONL format) — last 7 days of entries.

### Step 2: Gather project state

For each active project, check:
- Open action_items from brainstorm entries
- Pending next_steps from progress entries
- Unresolved decisions older than 14 days
- Any predictions past their resolution date

### Step 3: Generate the digest

Structure as follows:

#### Priority Tasks
- Pull action_items from recent brainstorms and next_steps from progress
- Group by project
- Highlight time-sensitive or blocking items

#### Overdue Items
- Predictions past resolution date without outcome
- Projects with 0 entries in 7+ days
- Old unresolved decisions

#### Pattern Alerts
- Same anti-pattern appearing 2+ times this week
- Decisions without reflections in any project
- Projects with lots of brainstorms but no progress entries (ideas without execution)

### Step 4: Deliver as concise pulse

10-15 lines maximum. Lead with the most actionable items. Skip empty sections — don't generate noise.

## Example Output

```
Daily Digest — Mar 9, 2026

PRIORITY TASKS:
- [my-app] Fix response verbosity — logging when user asked for advice
- [tools] Finish remaining edge analyses
- [research] Test autoresearch repo

OVERDUE:
- Prediction "50 users by March" resolves this week — no outcome logged

PATTERNS:
- 4 projects touched this week, 3 went dark
- my-app has 5 brainstorms, 0 progress entries — shipping gap
```

## Customization

Edit the project list and journal path to match your setup. The digest format is intentionally minimal — extend the sections if you need more detail.
