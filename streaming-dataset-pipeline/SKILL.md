---
name: streaming-dataset-pipeline
description: Build a multi-stage streaming data pipeline that downloads, extracts, filters, joins, and assembles a final dataset with resumable checkpoints at each stage. Use when creating datasets from external sources (APIs, data dumps, web scrapes) that require multiple transformation steps and may fail mid-run.
metadata:
  author: niyaznurbhasha
  version: 1.0.0
  category: data-pipelines
  tags: [dataset, pipeline, streaming, etl, resumable, checkpoint, jsonl, data-processing]
---

# Streaming Dataset Pipeline

A multi-stage pipeline pattern for building datasets from external sources. Each stage reads from the previous stage's output, streams records to avoid memory issues, and supports resume-from-checkpoint on failure.

## Architecture Overview

```
Stage 1: DOWNLOAD
  API/Dump → data/raw/{source}_{type}.jsonl
  (resumable by last timestamp)

Stage 2: EXTRACT
  raw JSONL → data/extracted/{source}_filtered.jsonl
  (streaming filter: keep only matching records)

Stage 3: JOIN
  extracted submissions + comments → data/joined/paired.jsonl
  (load one side into memory, stream the other)

Stage 4: ASSEMBLE
  paired data + assets → data/final/dataset.jsonl
  (verify assets exist, produce final schema)
```

Each stage is an independent CLI script. Stages can be re-run independently. JSONL is the interchange format between stages.

## Stage 1: Download with Resume Support

Download data from APIs with pagination, rate limit handling, and resume capability:

```python
import json
import os
import time
import requests

SESSION = requests.Session()
SESSION.headers.update({"User-Agent": "DataPipeline/1.0 (research)"})


def download_paginated(endpoint: str, params: dict, out_path: str):
    """Download all records from a paginated API with resume support.

    Resumes from the last record's timestamp if the output file exists.
    """
    # Resume: find last timestamp from existing output
    last_cursor = 0
    existing_count = 0
    if os.path.exists(out_path):
        with open(out_path) as f:
            for line in f:
                existing_count += 1
                try:
                    record = json.loads(line)
                    ts = record.get("created_utc", 0)
                    if ts > last_cursor:
                        last_cursor = ts
                except json.JSONDecodeError:
                    continue
        if existing_count > 0:
            print(f"  Resuming from {existing_count:,} records (after={last_cursor})")

    total = existing_count
    consecutive_empty = 0
    cursor = last_cursor

    with open(out_path, "a") as f:  # append mode for resume
        while True:
            request_params = {**params, "sort": "asc", "limit": "auto"}
            if cursor > 0:
                request_params["after"] = cursor

            # Retry with exponential backoff
            for attempt in range(5):
                try:
                    resp = SESSION.get(endpoint, params=request_params, timeout=30)
                    if resp.status_code == 429:
                        wait = int(resp.headers.get("Retry-After", 10))
                        print(f"  Rate limited, waiting {wait}s...")
                        time.sleep(wait)
                        continue
                    resp.raise_for_status()
                    break
                except requests.RequestException as e:
                    if attempt < 4:
                        time.sleep(2 ** attempt)
                    else:
                        print(f"  ERROR: Failed after 5 retries: {e}")
                        return total

            results = resp.json().get("data", [])

            if not results:
                consecutive_empty += 1
                if consecutive_empty >= 3:
                    break
                # Jump forward to skip gaps in the data
                cursor += 30 * 86400
                time.sleep(0.5)
                continue

            consecutive_empty = 0

            for record in results:
                f.write(json.dumps(record) + "\n")
                total += 1

            # Advance cursor past last result
            cursor = max(r.get("created_utc", 0) for r in results)

            if total % 1000 < len(results):
                print(f"  {total:,} records downloaded (cursor={cursor})")

            time.sleep(0.3)  # respect rate limits

    return total
```

Key patterns:
- **Append mode** — open output in `"a"` mode so resume doesn't overwrite existing data
- **Cursor from file** — scan existing output to find the last cursor value on startup
- **Consecutive empty handling** — APIs may have gaps; skip forward instead of stopping immediately
- **Exponential backoff** — `2^attempt` seconds between retries, with 429-specific handling

## Stage 2: Streaming Extract and Filter

Stream through raw data, apply domain-specific filters, and output only qualifying records:

```python
import json

def stream_jsonl(path: str):
    """Stream records from a JSONL file."""
    with open(path) as f:
        for line in f:
            line = line.strip()
            if not line:
                continue
            try:
                yield json.loads(line)
            except json.JSONDecodeError:
                continue


def is_qualifying_record(record: dict) -> bool:
    """Domain-specific filter. Customize for your use case."""
    # Example: keep only image posts with 2+ comments
    url = record.get("url", "")
    has_image = any(domain in url for domain in ["i.redd.it", "i.imgur.com"])
    has_engagement = record.get("num_comments", 0) >= 2
    return has_image and has_engagement


def extract_relevant_fields(record: dict) -> dict:
    """Reduce record to only the fields we need downstream."""
    return {
        "id": record.get("id", ""),
        "title": record.get("title", ""),
        "url": record.get("url", ""),
        "score": record.get("score", 0),
        "created_utc": record.get("created_utc", 0),
        "num_comments": record.get("num_comments", 0),
    }


def run_extract(input_path: str, output_path: str):
    """Stream through input, filter, and write qualifying records."""
    count = 0
    kept = 0

    with open(output_path, "w") as out:
        for record in stream_jsonl(input_path):
            count += 1
            if count % 100_000 == 0:
                print(f"  processed {count:,}, kept {kept:,}...")

            if not is_qualifying_record(record):
                continue

            cleaned = extract_relevant_fields(record)
            out.write(json.dumps(cleaned) + "\n")
            kept += 1

    print(f"Extracted: {count:,} total, {kept:,} kept ({kept/max(count,1)*100:.1f}%)")
    return kept
```

## Stage 3: Memory-Efficient Join

When joining two datasets, load the smaller one into memory and stream the larger one:

```python
from collections import defaultdict


def load_primary_index(data_dir: str) -> dict:
    """Load primary records (e.g., submissions) into a dict keyed by ID."""
    index = {}
    for fname in os.listdir(data_dir):
        if not fname.endswith(".jsonl"):
            continue
        with open(os.path.join(data_dir, fname)) as f:
            for line in f:
                record = json.loads(line)
                # Only load primary records (submissions have a distinguishing field)
                if "title" not in record:
                    continue
                index[record["id"]] = record
    print(f"Indexed {len(index):,} primary records")
    return index


def stream_secondary(data_dir: str):
    """Stream secondary records (e.g., comments)."""
    for fname in sorted(os.listdir(data_dir)):
        if not fname.endswith(".jsonl"):
            continue
        with open(os.path.join(data_dir, fname)) as f:
            for line in f:
                record = json.loads(line)
                if "parent_id" not in record:
                    continue
                yield record


def score_match(record: dict) -> float:
    """Score how good a match is (for ranking). Customize per domain."""
    base_score = record.get("score", 0)
    length_bonus = min(len(record.get("body", "")) / 100, 5)
    return base_score + length_bonus


def join_datasets(
    data_dir: str,
    output_path: str,
    max_matches: int = 5,
    min_matches: int = 1,
):
    """Join primary and secondary records.

    Loads primary into memory, streams secondary, collects top matches.
    """
    primary = load_primary_index(data_dir)
    matches_by_id = defaultdict(list)

    total_secondary = 0
    total_matched = 0

    for record in stream_secondary(data_dir):
        total_secondary += 1
        if total_secondary % 500_000 == 0:
            print(f"  {total_secondary:,} secondary processed, "
                  f"{total_matched:,} matched...")

        # Extract the primary record ID from the reference field
        ref_id = record.get("link_id", "").lstrip("t3_")
        if ref_id not in primary:
            continue

        total_matched += 1
        matches_by_id[ref_id].append({
            "body": record["body"],
            "score": record.get("score", 0),
            "match_score": score_match(record),
        })

    # Build joined output — keep top N matches per primary record
    paired = []
    for primary_id, matches in matches_by_id.items():
        if len(matches) < min_matches:
            continue

        matches.sort(key=lambda m: m["match_score"], reverse=True)
        top_matches = matches[:max_matches]

        rec = primary[primary_id].copy()
        rec["matches"] = [
            {"body": m["body"], "score": m["score"]}
            for m in top_matches
        ]
        paired.append(rec)

    with open(output_path, "w") as f:
        for record in paired:
            f.write(json.dumps(record) + "\n")

    print(f"Joined: {len(paired):,} paired records")
    return paired
```

## Stage 4: Final Assembly with Asset Verification

The final stage produces the clean dataset, verifying that all referenced assets (images, files) actually exist:

```python
def assemble_dataset(
    paired_path: str,
    assets_dir: str,
    output_path: str,
    require_assets: bool = False,
):
    """Build final dataset, verifying asset availability."""
    # Index available assets
    available_assets = set()
    if os.path.exists(assets_dir):
        available_assets = set(os.listdir(assets_dir))
    print(f"Found {len(available_assets):,} available assets")

    total = 0
    kept = 0
    total_assets = 0

    with open(paired_path) as fin, open(output_path, "w") as fout:
        for line in fin:
            record = json.loads(line)
            total += 1

            # Resolve local asset paths
            local_assets = []
            for idx, url in enumerate(record.get("asset_urls", [])):
                for ext in (".jpg", ".png", ".webp", ".json"):
                    fname = f"{record['id']}_{idx}{ext}"
                    if fname in available_assets:
                        local_assets.append(os.path.join("assets", fname))
                        break

            if require_assets and not local_assets:
                continue

            output = {
                "id": record["id"],
                "title": record.get("title", ""),
                "assets": local_assets,
                "asset_urls": record.get("asset_urls", []),
                "matches": record.get("matches", []),
                "score": record.get("score", 0),
                "created_utc": record.get("created_utc", 0),
            }
            fout.write(json.dumps(output) + "\n")
            kept += 1
            total_assets += len(local_assets)

    print(f"\nDataset: {kept:,}/{total:,} records, {total_assets:,} assets")
    if kept > 0:
        print(f"  Avg assets/record: {total_assets / kept:.1f}")
        print(f"  Avg matches/record: "
              f"{sum(len(json.loads(l).get('matches', [])) for l in open(output_path)) / kept:.1f}")
```

## Pipeline Runner

Tie all stages together with a runner script:

```python
import argparse

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--stage", choices=["download", "extract", "join", "assemble", "all"],
                        default="all")
    parser.add_argument("--data-dir", default="data")
    args = parser.parse_args()

    stages = {
        "download": run_download,
        "extract": run_extract_all,
        "join": run_join,
        "assemble": run_assemble,
    }

    if args.stage == "all":
        for name, fn in stages.items():
            print(f"\n{'='*50}\nStage: {name}\n{'='*50}")
            fn(args.data_dir)
    else:
        stages[args.stage](args.data_dir)
```

## Design Principles

1. **JSONL everywhere** — Every stage reads and writes JSONL. It's streamable, appendable, and line-level corruptions don't break the whole file.

2. **Stages are idempotent** — Each stage can be re-run. Download appends with resume. Extract and later stages overwrite output.

3. **Stream the big side, index the small side** — In joins, load submissions (thousands) into memory, stream comments (millions).

4. **Progress every N records** — Print progress at regular intervals. For million-record datasets, use `count % 100_000 == 0` or similar.

5. **Explicit counts at every stage** — Log total input, kept, skipped-by-reason. This makes debugging filter logic trivial.

## Common Pitfalls

1. **No resume = start over on failure** — Always track a cursor (timestamp, offset, page token) and resume from it
2. **Loading everything into memory** — Stream with generators (`yield`); only load the smaller dataset into memory for joins
3. **Silent filter drops** — Track skip reasons separately (e.g., `skipped_no_image`, `skipped_low_score`) so you can debug filter thresholds
4. **Rate limit ignorance** — Respect `Retry-After` headers; use exponential backoff; add `time.sleep()` between requests
5. **Single output file** — Keep one JSONL per stage, not per-source; it simplifies downstream stages
6. **No gap handling** — APIs have data gaps; don't stop on first empty response, try jumping the cursor forward
