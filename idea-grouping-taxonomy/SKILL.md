---
name: idea-grouping-taxonomy
description: Taxonomy-based grouping of large collections (100+) of unstructured items using LLM agents in controlled batches with validation passes, correction cycles, and health checks. Use when you have a large set of extracted ideas, features, research papers, support tickets, or any items that need meaningful categorization at scale without hallucination.
metadata:
  author: niyaznurbhasha
  version: 1.0.0
  category: research
  tags: [taxonomy, grouping, categorization, llm-agents, batch-processing, idea-management, clustering]
---

# Idea Grouping Taxonomy

A battle-tested process for organizing large collections of unstructured items (ideas, features, papers, tickets) into a meaningful taxonomy using LLM agents. Designed to prevent hallucination, catch misassignments, and maintain taxonomy health as the collection grows.

## When to Use

You have 100+ items that need meaningful categorization, and:
- Simple keyword matching is insufficient (items are semantically rich)
- You need human-readable group names and summaries, not just cluster IDs
- The taxonomy should evolve as new items arrive
- Accuracy matters — you cannot afford silent misassignment

## Critical Constraints (Learned the Hard Way)

Before starting, internalize these rules:

1. **Never give an LLM more than 50 items to group at once.** Beyond ~50, LLMs start hallucinating item titles, inventing items that do not exist, and silently dropping items. Keep batches at 45-50 max.
2. **Always pass actual data as text input.** Do not rely on LLM memory or context from previous turns. Every batch must include the full text of the items being grouped.
3. **Validate everything.** After every grouping pass, verify: every item was assigned, all group codes are valid, no titles were hallucinated.
4. **Group summaries must be written from full descriptions**, not just titles and tags. Title-only summaries achieve roughly 85% quality — you lose nuance.
5. **If an agent fails, do not retry in the same conversation.** Note the failure, move on, and retry in a fresh context.

## The Process

### Step 1: Define the Initial Taxonomy

Before grouping anything, define your group taxonomy. Each group needs:
- **Code**: Short identifier (e.g., G01, G02, CAT-A, CAT-B)
- **Name**: Human-readable label (e.g., "Finance/Trading", "Developer Workflow")
- **Description**: 1-2 sentence scope definition — what belongs here and what does not

Start with 10-20 groups based on a quick scan of 50-100 items. You will refine later.

```
G01: 3D Rendering — Ideas related to mesh texturing, shading, real-time rendering
G02: Finance — Trading strategies, portfolio management, market analysis
G03: Developer Tools — IDE plugins, CLI tools, workflow automation
...
```

The taxonomy is a living document. It will grow and split as you process more items.

### Step 2: Batch the Items

Split your collection into batches of ~45 items each.

```python
import itertools

def create_batches(items, batch_size=45):
    it = iter(items)
    batch_num = 0
    while True:
        batch = list(itertools.islice(it, batch_size))
        if not batch:
            break
        yield batch_num, batch
        batch_num += 1
```

### Step 3: Run Grouping Agents

For each batch, send the full item data (not just titles) along with the complete taxonomy definition. The agent assigns each item to exactly one group.

**Agent prompt structure:**

```
Here is the current taxonomy:
[Full taxonomy with codes, names, and descriptions]

Here are the items to categorize. For each item, assign exactly one group code.

[Full text of each item: ID, title, description, tags]

Output format:
{"assignments": [{"item_id": "...", "group": "G01"}, ...]}
```

**Parallelization**: Run 5-7 agents in parallel. More than 10 causes contention and slowdowns without meaningful speedup.

### Step 4: Validate Assignments

After all batches complete, run validation checks:

```python
def validate_assignments(assignments, all_item_ids, valid_groups):
    errors = []

    assigned_ids = {a["item_id"] for a in assignments}
    valid_group_set = set(valid_groups)

    # Every item must be assigned
    missing = all_item_ids - assigned_ids
    if missing:
        errors.append(f"Unassigned items: {missing}")

    # No hallucinated items
    extra = assigned_ids - all_item_ids
    if extra:
        errors.append(f"Hallucinated items: {extra}")

    # All group codes must be valid
    invalid = {a["group"] for a in assignments} - valid_group_set
    if invalid:
        errors.append(f"Invalid group codes: {invalid}")

    # No duplicates
    if len(assigned_ids) != len(assignments):
        errors.append("Duplicate assignments detected")

    return errors
```

If validation fails, reprocess only the failing batches.

### Step 5: Run Correction Pass

Even with good initial grouping, expect 5-15% misassignment. Run a correction pass:

1. Split all grouped items into review batches (6-8 agents)
2. Each agent receives items WITH their full descriptions and current group assignment
3. Agent task: "Review each assignment. Flag any item that is a poor fit for its current group. For flagged items, suggest the correct group and explain why."
4. Apply corrections where the agent's reasoning is sound

**Correction agent prompt:**

```
You are reviewing group assignments for accuracy.

For each item below, you see the item's full description and its assigned group.
Flag any item where the assignment is wrong. For flagged items, provide:
- The item ID
- Current (wrong) group
- Suggested (correct) group
- Brief reason

Only flag items you are confident are misassigned. When in doubt, leave the current assignment.

[Items with descriptions and current groups]
[Full taxonomy for reference]
```

### Step 6: Write Group Summaries

For each group, generate a 2-3 sentence summary from the full descriptions of all items in the group. This summary describes the theme, scope, and typical items found in the group.

Do NOT write summaries from titles alone — they miss nuance and produce generic descriptions.

```
G01: 3D Mesh Texturing (144 items)
Summary: Techniques for applying textures to 3D meshes, including UV mapping,
procedural texturing, and neural texture synthesis. Covers both real-time game
assets and offline rendering pipelines. Heavy overlap with ControlNet-based
generation approaches.
```

### Step 7: Taxonomy Health Check

After every major grouping cycle, run a health check agent:

**Task A — Misfit Detection**: Scan all groups for items that are poor fits. These are items the correction pass missed, or items that fit poorly because the group definition is too broad.

**Task B — Cluster Detection**: Look for clusters of 5+ items within a group that share a distinct sub-theme. These suggest a new group should be created.

**Task C — Merge Detection**: Look for groups with fewer than 5 items that could be merged into a related group.

Apply the health check results: create new groups, merge small groups, reassign misfits.

## Cadence for Growing Collections

When processing items incrementally (e.g., extracting from a large corpus in rounds):

| Milestone | Action |
|-----------|--------|
| Every 500 items processed | Merge new items into master, report totals |
| Every 1000 items processed | Full regroup of ALL items + taxonomy health check |
| Every 1500-2000 items | Start a fresh context to avoid LLM context degradation |

## Handling Taxonomy Evolution

As your collection grows, the taxonomy must evolve:

1. **Splitting**: When a group exceeds 100 items, check if it has distinct sub-clusters. Split if sub-themes are clearly different.
2. **Merging**: When a group has fewer than 5 items and a close neighbor exists, merge them.
3. **Renaming**: As a group's composition becomes clearer, update the name and description to match reality rather than original intent.
4. **New groups**: The health check (Step 7) will surface clusters that need new groups. Create them proactively.

## Output Artifacts

At the end of a grouping cycle, you should have:

1. **Taxonomy definition**: All group codes, names, descriptions, and item counts
2. **Assignment file**: Every item mapped to exactly one group
3. **Group summaries**: 2-3 sentence descriptions written from full item data
4. **Validation report**: Confirmation that all items are assigned, no hallucinations, all codes valid

## Adapting to Different Item Types

This process works for any collection that needs categorization:

- **Research papers**: Groups become research areas or methodologies
- **Support tickets**: Groups become issue categories or product areas
- **Feature requests**: Groups become product themes or user needs
- **Code snippets**: Groups become patterns or domains
- **Meeting notes**: Groups become projects or decision areas

The process is identical. Only the taxonomy definitions and item schemas change.
