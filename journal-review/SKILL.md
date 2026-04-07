---
name: journal-review
description: Reviews and corrects journal entry categorizations — moves items between projects, updates types, marks tasks complete, creates new projects. Use when user says "fix the categorization", "move this to", "that's wrong project", "review my entries", "clean up journal", or provides corrections to previously written entries.
metadata:
  author: niyaznurbhasha
  version: 1.0.0
  category: productivity
  tags: [journal, review, correction, reorganization]
---

# Journal Review

Processes user corrections to journal entries — reassigns projects, fixes types, marks items complete, and creates new project categories.

## When This Activates

- User reviews brain-dump output and wants changes
- User says items are in the wrong project
- User wants to mark things as done
- User wants to split a project into multiple
- User wants to merge entries or create new projects

## Instructions

### Step 1: Understand the corrections

Parse what the user wants changed. Common correction types:

| User Says | Action |
|-----------|--------|
| "Move X to project Y" | Write new entry under project Y with the item, note removal from old project |
| "That's a new project called Z" | Add Z to project registry, write entries under new slug |
| "I already did X" | Write a progress entry marking it complete |
| "That's not a decision, it's a brainstorm" | Write corrected entry with right type |
| "Combine these two" | Merge items into single entry |
| "Split this into A and B" | Write two separate entries |
| "Remove X" / "That's not relevant" | Do not re-write the item |

### Step 2: Show proposed changes

Before writing, show what will change:

```
## Proposed Changes

MOVE:
- "Research speech analysis" → from coaching to domain-expansions
- "Spatial reasoning scan" → from megabot to research

NEW PROJECT:
- discourse (Discourse) — for anonymized interaction items

MARK COMPLETE:
- "Google OAuth setup" → progress entry (already done)
- "Website Mii character scene" → progress entry (already done)

REMOVE:
- "SketchUp character scene" — duplicate of website entry

Proceed?
```

### Step 3: Execute changes

For moves and corrections:
- Write NEW entries with corrected project tags and types
- The old entries remain in the journal (append-only) but the newer entries take precedence since they sort by date

For new projects:
- Add to project registry configuration
- Write entries under the new project slug

For completions:
- Write progress entries that capture what was accomplished
- Note the original brainstorm/task it resolves

### Step 4: Update project registry

If new projects were created, update:
1. The project registry config file (e.g., `config.py` or equivalent)
2. Any project table in CLAUDE.md or similar docs
3. The journal-capture skill's project table (if bundled)

### Step 5: Report changes

```
## Changes Applied

- Moved 3 items to research project
- Created discourse project with 2 entries
- Marked 4 items as complete (progress entries)
- Updated project registry: 12 → 13 projects

Dashboard re-render needed: run `conda run -n discovery python dashboard.py`
```

## Handling Ambiguity

When the user's correction is ambiguous:
- "That's wrong" → Ask: "Which project should it be in?"
- "Fix everything" → Show current state grouped by project, ask for specific corrections
- Partial corrections → Apply what's clear, ask about the rest

## Append-Only Journal

The journal is JSONL (append-only). To "move" an item:
1. Write a new entry with the correct categorization
2. The old entry stays but newer entries appear first in the dashboard
3. If the old entry causes confusion, note the correction in the new entry's content
