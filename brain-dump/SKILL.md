---
name: brain-dump
description: Captures a stream-of-consciousness brain dump and organizes it into structured journal entries across projects. Use when user dumps multiple ideas, tasks, and thoughts at once, says "brain dump", "here's everything on my mind", "let me just get this all out", or provides a long unstructured message spanning multiple topics.
metadata:
  author: niyaznurbhasha
  version: 1.0.0
  category: productivity
  tags: [brain-dump, capture, organization, triage, intake]
---

# Brain Dump

Parses an unstructured stream of ideas, tasks, decisions, and thoughts into structured journal entries, correctly categorized by project and type.

## Critical Rule: Never Write Before Review

This skill operates in TWO phases. **Never skip Phase 1 and go straight to writing.**

## Phase 1: Parse & Propose

### Step 1: Extract individual items

Read the brain dump carefully. Each distinct idea, task, decision, or observation becomes a separate item. Signs of a new item:
- Topic shift (different project, different domain)
- "Also", "and then", "another thing", "oh and"
- Shift from idea → task → reflection

Be thorough — users often bury important items in the middle of tangents.

### Step 2: Classify each item

For each extracted item, determine:

**Type** (use journal-capture type rules):
- Is it a choice/decision being made? → `decision`
- Is it an idea or brainstorm? → `brainstorm`
- Did something ship/complete? → `progress`
- Is it a task/to-do? → `brainstorm` (as an action_item)
- Is it a realization? → `reflection`
- Is it analyzing someone else's system? → `teardown`

**Project assignment:**
1. Check the project registry (see references/projects.md or ask user for their project list)
2. Match based on topic, not just keywords
3. If an item doesn't fit any existing project, flag it as `[NEW PROJECT?]` — do NOT silently force it into an existing one
4. If an item spans 2+ projects, pick the primary one and note the secondary

**Grouping:**
- Multiple items for the same project and same type → group into one entry
- Individual to-dos for the same project → group as action_items in one brainstorm
- Keep decisions separate even if same project (each decision deserves its own entry)

### Step 3: Present the proposal table

Output a table for user review:

```
## Brain Dump Review — [N] items extracted

| # | Item | Type | Project | Notes |
|---|------|------|---------|-------|
| 1 | Fix advice verbosity | brainstorm (action_item) | coaching | Grouped with #2, #3 |
| 2 | Add user feedback mechanism | brainstorm (action_item) | coaching | Grouped with #1, #3 |
| 3 | iPhone widget for logging | brainstorm (action_item) | coaching | Grouped with #1, #2 |
| 4 | Focus on harness, not models | decision | megabot | |
| 5 | Anonymized trust-building app | brainstorm | [NEW PROJECT?] | Doesn't fit existing projects |
| 6 | Finished the website | progress | website | Mark as completed |
| 7 | Research speech analysis APIs | brainstorm (action_item) | [AMBIGUOUS] coaching or new? | User should clarify |

### New projects detected:
- **Item 5** suggests a new project around anonymized social interactions. Create new project?

### Completed items detected:
- **Item 6** — should this be logged as a progress entry?

### Ambiguous assignments:
- **Item 7** — speech analysis could be a coaching domain expansion or its own project. Which?

**Please review and tell me what to change. Say "looks good" to write all entries.**
```

### Accuracy heuristics

To maximize first-pass accuracy:

1. **When in doubt, flag it** — an `[AMBIGUOUS]` tag that gets corrected is better than a silent miscategorization
2. **New > forced** — if something doesn't clearly fit an existing project, suggest a new one rather than cramming it in
3. **Read the project registry descriptions** — match on scope/purpose, not just name keywords
4. **Tasks vs ideas** — if someone says "I need to X", that's a to-do (action_item). If they say "what if we X", that's an idea (brainstorm content)
5. **Completed items** — watch for past tense ("I already did", "that's done", "I finished") — these are progress entries, not tasks
6. **Decisions in disguise** — "I think we should X instead of Y" or "focus on A not B" are decisions even if the user doesn't say "I decided"

## Phase 2: Confirm & Write

Only after user confirms (or provides corrections):

### Step 1: Apply corrections

Update the categorization based on user feedback:
- Move items between projects as directed
- Create new projects if approved
- Merge or split items as needed
- Change types as corrected

### Step 2: Write entries

For each group, write a journal entry following the journal-capture schema:
- One brainstorm per project (with grouped action_items)
- One entry per decision
- One entry per progress item
- Tag with correct project slug

### Step 3: Report what was written

```
## Written [N] entries:
- [brainstorm] coaching: 8 action items (product fixes, demos, PT partnership...)
- [decision] megabot: Focus on harness layer, not custom models
- [progress] website: Website finished with Mii-style character scene
- [brainstorm] [NEW] discourse: 3 action items (anonymized interaction design...)

New projects created: discourse (Discourse)
```

## Handling Large Brain Dumps

For dumps with 15+ items:
- Still extract everything — don't summarize or drop items
- Group aggressively by project to keep entry count manageable
- Present the table in project-grouped sections for easier review
- If the dump mentions many new project ideas, group them under a single "New Ideas" brainstorm rather than creating 5 new projects

## Learning From Corrections

When the user corrects a categorization, note the pattern:
- "That's not megabot, that's research" → research tasks get mixed with the project they feed into
- "That's a new project" → watch for items that feel like they fit but are actually distinct enough to be separate
- "That's already done" → listen for past tense more carefully

These corrections improve future brain dumps in the same session.
