---
name: idea-extraction-batch-pipeline
description: Extract structured records from large conversation/document corpora using parallel LLM agents, with batch creation, merge, deduplication, taxonomy grouping, and interactive dashboard generation. Use when mining insights, ideas, or structured data from large text corpora (chat exports, meeting transcripts, research notes, support tickets) at scale.
metadata:
  author: niyaznurbhasha
  version: 1.0.0
  category: data-pipelines
  tags: [extraction, llm, batch-processing, parallel-agents, grouping, taxonomy, dashboard, conversation-mining]
---

# Idea Extraction Batch Pipeline

A batch pipeline for extracting structured records from large text corpora using parallel LLM agents. Handles batching, parallel extraction, merge, deduplication, taxonomy-based grouping, and dashboard generation.

## Architecture Overview

```
Source Corpus (JSONL chunks)
    |
    v
[1. BATCH] — Split into batches of 50 chunks
    |
    v
[2. EXTRACT] — N parallel LLM agents, one per batch
    |
    v
[3. MERGE] — Append new records to master file
    |
    v
[4. GROUP] — Assign records to taxonomy groups (batches of ~45)
    |
    v
[5. VALIDATE] — Cross-check assignments, flag mismatches
    |
    v
[6. DASHBOARD] — Generate interactive HTML from grouped data
```

## Step 1: Prepare the Source Corpus

Your source data should be a JSONL file where each line is a "chunk" — a unit of text to extract from. Use Python (not sed) for slicing, especially on WSL/NTFS:

```python
import itertools

def create_batches(
    corpus_path: str,
    output_dir: str,
    start_offset: int,
    num_batches: int = 10,
    batch_size: int = 50,
):
    """Create batch files from a JSONL corpus.

    Each batch is a JSONL file with `batch_size` chunks.
    Use Python's itertools.islice — sed is extremely slow on WSL/NTFS.
    """
    with open(corpus_path) as f:
        lines = list(itertools.islice(f, start_offset, start_offset + num_batches * batch_size))

    for i in range(num_batches):
        batch = lines[i * batch_size : (i + 1) * batch_size]
        batch_num = (start_offset // batch_size) + i
        out_path = f"{output_dir}/batch_{batch_num}.jsonl"
        with open(out_path, "w") as out:
            out.writelines(batch)
        print(f"  Created {out_path} ({len(batch)} chunks)")
```

## Step 2: Define the Extraction Schema

Define a clear schema for what you're extracting. Each record needs a unique ID, core fields, and evidence pointers back to the source:

```python
RECORD_SCHEMA = {
    "record_id": str,          # unique: "batch{N}_{seq}" (e.g., "batch42_3")
    "title": str,              # short descriptive title
    "one_liner": str,          # single-sentence summary
    "description": str,        # full description (2-5 sentences)
    "problem": str,            # what problem this addresses
    "proposed_solution": str,  # how to solve it
    "tags": list[str],         # classification tags
    "stack": list[str],        # technologies/tools mentioned
    "blockers": list[dict],    # obstacles: {"type": str, "detail": str}
    "evidence": {              # pointer back to source
        "chunk_id": str,
        "conversation_title": str,
        "timestamp_start": float,
    },
}
```

### Critical Rules for LLM Extraction

Include these in your extraction prompt:
- **Blockers must be explicitly stated** — never infer or assume blockers. Only include obstacles that are clearly described as problems in the source text.
- **One record per distinct concept** — don't merge separate ideas into one record.
- **ID format must be deterministic** — `batch{N}_{seq}` prevents collisions across parallel agents.

## Step 3: Parallel Agent Extraction

Launch multiple LLM agents in parallel, each processing one batch:

```python
import json
import concurrent.futures

def extract_batch(batch_path: str, output_path: str, llm_client):
    """Process a single batch file with an LLM agent.

    Reads chunks, sends to LLM with extraction prompt, writes records.
    """
    with open(batch_path) as f:
        chunks = [json.loads(line) for line in f]

    batch_num = batch_path.split("batch_")[1].split(".")[0]

    prompt = build_extraction_prompt(chunks, batch_num)
    response = llm_client.invoke(prompt)
    records = parse_extraction_response(response)

    with open(output_path, "w") as out:
        for record in records:
            out.write(json.dumps(record) + "\n")

    return {"batch": batch_num, "records": len(records)}


def run_parallel_extraction(batch_dir: str, output_dir: str, max_workers: int = 10):
    """Run extraction on all batch files using parallel agents.

    10 parallel agents works but causes some contention; 7 is safer
    if you see slowdowns.
    """
    import glob
    batch_files = sorted(glob.glob(f"{batch_dir}/batch_*.jsonl"))

    results = []
    with concurrent.futures.ThreadPoolExecutor(max_workers=max_workers) as pool:
        futures = {}
        for batch_file in batch_files:
            batch_num = batch_file.split("batch_")[1].split(".")[0]
            out_file = f"{output_dir}/batch_{batch_num}_records.jsonl"
            future = pool.submit(extract_batch, batch_file, out_file)
            futures[future] = batch_num

        for future in concurrent.futures.as_completed(futures):
            batch_num = futures[future]
            try:
                result = future.result()
                results.append(result)
                print(f"  Batch {batch_num}: {result['records']} records extracted")
            except Exception as e:
                print(f"  Batch {batch_num}: FAILED — {e}")
                # Note failure and move on; don't retry in same session
                results.append({"batch": batch_num, "records": 0, "error": str(e)})

    total = sum(r["records"] for r in results)
    failed = sum(1 for r in results if r.get("error"))
    print(f"\nExtraction complete: {total} records from {len(results)} batches "
          f"({failed} failed)")
    return results
```

## Step 4: Merge Into Master File

After each extraction round, append new records to the master file:

```python
def merge_into_master(
    batch_output_dir: str,
    master_path: str,
    batch_pattern: str = "batch_*_records.jsonl",
):
    """Append all new batch records to the master file.

    The master file is append-only — never overwrite it.
    """
    import glob
    new_files = sorted(glob.glob(f"{batch_output_dir}/{batch_pattern}"))

    new_count = 0
    with open(master_path, "a") as master:
        for fpath in new_files:
            with open(fpath) as f:
                for line in f:
                    master.write(line)
                    new_count += 1

    # Report total
    total = sum(1 for _ in open(master_path))
    print(f"Merged {new_count} new records. Master total: {total}")
    return {"new": new_count, "total": total}
```

## Step 5: Taxonomy Grouping

Group records into a taxonomy. CRITICAL: never give an LLM more than ~50 records at once — it hallucinates after ~260. Always pass actual data as text input, never rely on agent memory.

```python
def group_records(
    master_path: str,
    taxonomy: list[dict],
    output_path: str,
    group_batch_size: int = 45,
):
    """Assign records to taxonomy groups using LLM agents.

    Process in batches of ~45 records to avoid LLM hallucination.
    Always pass the actual record data as text input.
    """
    with open(master_path) as f:
        records = [json.loads(line) for line in f]

    # Prepare lightweight summaries for grouping
    summaries = [
        {"record_id": r["record_id"], "title": r["title"], "tags": r.get("tags", [])}
        for r in records
    ]

    assignments = {}

    for i in range(0, len(summaries), group_batch_size):
        batch = summaries[i:i + group_batch_size]

        # Build prompt with actual data embedded as text
        prompt = build_grouping_prompt(batch, taxonomy)
        response = llm_client.invoke(prompt)
        batch_assignments = parse_grouping_response(response)

        for record_id, group_id in batch_assignments.items():
            assignments[record_id] = group_id

    # Validate: every record assigned, all group codes valid
    valid_groups = {g["group_id"] for g in taxonomy}
    unassigned = [r["record_id"] for r in records if r["record_id"] not in assignments]
    invalid = {rid: gid for rid, gid in assignments.items() if gid not in valid_groups}

    if unassigned:
        print(f"WARNING: {len(unassigned)} unassigned records")
    if invalid:
        print(f"WARNING: {len(invalid)} records assigned to invalid groups")

    # Build grouped output
    groups = {g["group_id"]: {**g, "records": []} for g in taxonomy}
    for record in records:
        gid = assignments.get(record["record_id"])
        if gid and gid in groups:
            groups[gid]["records"].append(record)

    with open(output_path, "w") as f:
        for group in groups.values():
            f.write(json.dumps(group) + "\n")

    return groups
```

## Step 6: Validation and Correction Pass

After grouping, run a correction pass where agents review full descriptions and flag misassignments:

```python
def run_correction_pass(
    grouped_path: str,
    num_reviewers: int = 6,
):
    """Review grouped records and flag misassignments.

    Each reviewer agent gets a subset of groups to review.
    They flag records that seem like poor fits.
    """
    with open(grouped_path) as f:
        groups = [json.loads(line) for line in f]

    # Split groups among reviewers
    per_reviewer = max(1, len(groups) // num_reviewers)
    corrections = []

    for i in range(0, len(groups), per_reviewer):
        reviewer_groups = groups[i:i + per_reviewer]
        # Send full descriptions (not just titles) to the reviewer
        prompt = build_review_prompt(reviewer_groups)
        response = llm_client.invoke(prompt)
        batch_corrections = parse_corrections(response)
        corrections.extend(batch_corrections)

    print(f"Correction pass: {len(corrections)} reassignments suggested")
    return corrections
```

## Step 7: Taxonomy Health Check

After grouping and corrections, check whether the taxonomy itself needs updating:

```python
def taxonomy_health_check(grouped_path: str, taxonomy: list[dict]):
    """Scan all groups for:
    (a) records that are poor fits in their current group
    (b) clusters of records that suggest a new group should be created

    Run this after every regroup cycle.
    """
    prompt = build_health_check_prompt(grouped_path, taxonomy)
    response = llm_client.invoke(prompt)

    suggestions = parse_health_check(response)
    # suggestions = {"poor_fits": [...], "new_groups_suggested": [...]}

    if suggestions["new_groups_suggested"]:
        print(f"New groups suggested: {len(suggestions['new_groups_suggested'])}")
        for sg in suggestions["new_groups_suggested"]:
            print(f"  - {sg['proposed_name']}: {sg['reason']}")

    return suggestions
```

## Step 8: Dashboard Generation

Generate an interactive HTML dashboard from the grouped data:

```python
import json
from collections import Counter, defaultdict
from pathlib import Path

def generate_dashboard(
    groups_path: str,
    records_path: str,
    output_path: str,
):
    """Generate an interactive HTML dashboard from grouped records.

    Panels:
    1. Connection graph (shared tags/stack between groups)
    2. Timeline heatmap (when records were created, by group)
    3. Blocker dashboard (what to fix first)
    4. Priority scorer (weighted ranking of what to work on next)
    """
    with open(groups_path) as f:
        groups = [json.loads(line) for line in f]
    with open(records_path) as f:
        records = [json.loads(line) for line in f]

    # Compute stats
    tags = Counter()
    for record in records:
        for t in record.get("tags", []):
            tags[t.lower()] += 1

    stats = {
        "records": len(records),
        "groups": len(groups),
        "with_blockers": sum(1 for g in groups if g.get("blockers")),
        "top_tag": tags.most_common(1)[0][0] if tags else "N/A",
    }

    # Build visualizations (using plotly, matplotlib, or raw HTML/JS)
    html = build_dashboard_html(groups, records, stats)

    Path(output_path).parent.mkdir(parents=True, exist_ok=True)
    Path(output_path).write_text(html, encoding="utf-8")
    print(f"Dashboard written to {output_path}")
```

### Priority Scoring Formula

Rank groups by a weighted combination of signals:

```python
def score_group(group: dict, records: list[dict]) -> dict:
    """Score a group for prioritization.

    Weights:
      Depth (30%) — how many sub-records exist (more = more thinking done)
      Blocker Freedom (25%) — fewer blockers = easier to start
      Stack Overlap (25%) — uses tools you already know
      Recency (20%) — more recently discussed = still top of mind
    """
    depth = min(group["record_count"] / 3.0, 10)
    blocker_score = max(0, 10 - len(group.get("blockers", [])) * 3)

    # Stack overlap with your most-used tools
    group_stack = set(s.lower() for s in group.get("stack", []))
    overlap = len(group_stack & known_stack)
    stack_score = min(overlap * 2.5, 10)

    # Recency from timestamps
    recency = compute_recency_score(group, records)

    total = (depth * 0.30 + blocker_score * 0.25 +
             stack_score * 0.25 + recency * 0.20)

    return {
        "group": group["title"],
        "total": round(total, 1),
        "depth": round(depth * 0.30, 1),
        "blocker_freedom": round(blocker_score * 0.25, 1),
        "stack_overlap": round(stack_score * 0.25, 1),
        "recency": round(recency * 0.20, 1),
    }
```

## Cadence Rules

For large corpora (10,000+ chunks), follow this cadence to manage context and quality:

| Every...          | Do...                                                |
|-------------------|------------------------------------------------------|
| 10 batches (500 chunks)  | Merge into master, report totals              |
| 20 batches (1000 chunks) | Regroup ALL records + regenerate dashboard    |
| 30-40 batches            | Start a fresh conversation to reset context   |

## Lessons Learned (Hard-Won)

1. **Never use sed on WSL/NTFS** for large file slicing — Python `itertools.islice` is orders of magnitude faster
2. **Never give an LLM 500+ records to group at once** — it hallucinates after ~260. Keep batches at 45-50 max.
3. **Always pass actual data as text input** to grouping agents — don't rely on agent memory or references to earlier context
4. **If an agent fails, don't retry in the same conversation** — note the failure and retry in a fresh session
5. **10 parallel agents works** but causes contention; 7 is safer if you see slowdowns
6. **Context bloat is real** — by batch 80+ the conversation slows noticeably. Start fresh every 30-40 batches.
7. **Group summaries must be written from full descriptions**, not just titles+tags. Titles-only gives ~85% quality.
8. **Master file is append-only** — never overwrite. Grouped file gets regenerated each cycle.
9. **Track everything in a state section** — batch number, chunk count, record count, last regroup. If context compresses, the state survives.

## Common Pitfalls

1. **No unique IDs** — records from parallel agents will collide if you use sequential IDs. Use `batch{N}_{seq}` format.
2. **Blockers from hallucination** — LLMs love inventing obstacles. Explicitly instruct: "only include blockers stated as problems in the source text."
3. **Grouping drift** — Taxonomy needs periodic health checks. Groups that grow too large should be split; orphan groups should be merged.
4. **Dashboard from stale data** — Always regenerate the dashboard after regrouping, not just after extraction.
5. **Lost progress on crash** — Keep the master file append-only and track state (last batch, counts) in a config file that survives context resets.
