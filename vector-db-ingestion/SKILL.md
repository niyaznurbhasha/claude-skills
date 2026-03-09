---
name: vector-db-ingestion
description: Build a bulk data ingestion pipeline for vector databases (pgvector, Pinecone, Qdrant, etc.) with batched embeddings, schema validation, deduplication, and error handling. Use when building RAG knowledge bases, semantic search systems, or any application that needs to embed and store text chunks at scale.
metadata:
  author: niyaznurbhasha
  version: 1.0.0
  category: data-pipelines
  tags: [vector-db, embeddings, rag, ingestion, pgvector, dedup, batch-processing]
---

# Vector DB Ingestion Pipeline

Ingests text chunks into a vector database with validation, batched embedding generation, cosine-similarity deduplication, and atomic insertion. Handles both single-record ingestion and bulk loading.

## Architecture Overview

```
Source Data (JSON files, APIs, docs)
    |
    v
[1. Load] — Read chunks from batch files
    |
    v
[2. Validate] — Schema checks, required fields, token estimation
    |
    v
[3. Embed] — Batched embedding generation (100 at a time)
    |
    v
[4. Dedup] — Cosine similarity check against existing vectors
    |
    v
[5. Insert] — Upsert into vector DB with ON CONFLICT handling
    |
    v
[6. Verify] — Count check, domain breakdown
```

## Step 1: Define the Chunk Schema and Validation

Every chunk needs a consistent schema. Validate before embedding to avoid wasting API calls:

```python
from dataclasses import dataclass

@dataclass
class ValidationResult:
    valid: bool
    errors: list[str]

REQUIRED_FIELDS = ["domain", "topic", "content", "summary", "source_citation",
                   "source_tier", "evidence_strength"]

def validate_chunk(chunk: dict) -> ValidationResult:
    """Validate a chunk has all required fields and passes quality checks."""
    errors = []

    for field in REQUIRED_FIELDS:
        if not chunk.get(field):
            errors.append(f"Missing required field: {field}")

    content = chunk.get("content", "")
    if len(content) < 50:
        errors.append("Content too short (< 50 chars)")
    if len(content) > 10000:
        errors.append("Content too long (> 10000 chars)")

    return ValidationResult(valid=len(errors) == 0, errors=errors)


def estimate_tokens(text: str) -> int:
    """Rough token count estimate (~4 chars per token)."""
    return len(text) // 4
```

## Step 2: Embedding Generation (Batched)

Always embed in batches to respect rate limits and maximize throughput. Use summary + content for richer embeddings:

```python
import time

def embed_texts(texts: list[str], model="text-embedding-3-small") -> list[list[float]]:
    """Embed a batch of texts. Wraps your embedding provider."""
    from openai import OpenAI
    client = OpenAI()
    response = client.embeddings.create(input=texts, model=model)
    return [item.embedding for item in response.data]


def embed_text(text: str) -> list[float]:
    """Embed a single text."""
    return embed_texts([text])[0]


def batch_embed(chunks: list[dict], batch_size: int = 100) -> list[list[float]]:
    """Embed all chunks in batches with progress reporting."""
    all_embeddings = []
    t0 = time.time()

    for i in range(0, len(chunks), batch_size):
        batch = chunks[i:i + batch_size]
        # Combine summary + content for richer embedding
        texts = [f"{c['summary']}\n\n{c['content']}" for c in batch]
        embs = embed_texts(texts)
        all_embeddings.extend(embs)

        elapsed = time.time() - t0
        done = i + len(batch)
        rate = done / elapsed
        print(f"  {done}/{len(chunks)} embedded ({elapsed:.0f}s, {rate:.1f}/s)")

        time.sleep(0.2)  # rate limit buffer

    return all_embeddings
```

## Step 3: Deduplication via Cosine Similarity

Before inserting, check if a near-duplicate already exists:

```python
def find_near_duplicates(
    embedding: list[float],
    domain: str,
    threshold: float = 0.95,
    db_conn=None,
) -> list[dict]:
    """Find existing chunks with cosine similarity above threshold."""
    vec_str = f"[{','.join(str(v) for v in embedding)}]"

    with db_conn.cursor() as c:
        c.execute(
            """
            SELECT id, topic,
                   1 - (embedding <=> %s::vector) AS cosine_sim
            FROM knowledge_chunks
            WHERE domain = %s
              AND 1 - (embedding <=> %s::vector) > %s
            ORDER BY cosine_sim DESC
            LIMIT 3
            """,
            (vec_str, domain, vec_str, threshold),
        )
        return [
            {"id": str(row[0]), "topic": row[1], "cosine_sim": float(row[2])}
            for row in c.fetchall()
        ]
```

## Step 4: Single-Record Ingestion (with Dedup)

For real-time ingestion (e.g., adding one chunk at a time):

```python
def ingest_chunk(chunk: dict, skip_dedup: bool = False, db_conn=None) -> dict:
    # Validate
    qa = validate_chunk(chunk)
    if not qa.valid:
        return {"status": "rejected", "errors": qa.errors}

    # Embed
    embed_input = f"{chunk['summary']}\n\n{chunk['content']}"
    try:
        embedding = embed_text(embed_input)
    except Exception as e:
        return {"status": "error", "errors": [f"embedding failed: {e}"]}

    # Dedup check
    if not skip_dedup:
        dupes = find_near_duplicates(embedding, chunk["domain"], db_conn=db_conn)
        if dupes:
            return {
                "status": "duplicate",
                "similar_to": dupes[0]["id"],
                "cosine_sim": dupes[0]["cosine_sim"],
            }

    # Insert
    vec_str = f"[{','.join(str(v) for v in embedding)}]"
    sql = """
        INSERT INTO knowledge_chunks
            (domain, topic, subtopic, content, summary,
             source_citation, source_tier, embedding, token_count)
        VALUES (%s,%s,%s,%s,%s,%s,%s,%s::vector,%s)
        ON CONFLICT DO NOTHING
        RETURNING id
    """
    with db_conn.cursor() as c:
        c.execute(sql, (
            chunk["domain"],
            chunk["topic"],
            chunk.get("subtopic"),
            chunk["content"],
            chunk["summary"],
            chunk["source_citation"],
            chunk["source_tier"],
            vec_str,
            estimate_tokens(chunk["content"]),
        ))
        row = c.fetchone()

    return {"status": "ok", "id": str(row[0])} if row else {"status": "conflict"}
```

## Step 5: Bulk Ingestion (Initial Load)

For loading large datasets, skip per-record dedup and use batched embeddings:

```python
import json
from pathlib import Path

def load_all_chunks(data_dir: Path) -> list[dict]:
    """Load chunks from batch JSON files."""
    all_chunks = []
    for f in sorted(data_dir.glob("batch_*.json")):
        data = json.loads(f.read_text())
        chunks = data if isinstance(data, list) else data.get("chunks", [])
        all_chunks.extend(chunks)
    return all_chunks


def bulk_insert(chunks: list[dict], embeddings: list[list[float]], db_conn) -> dict:
    """Insert pre-embedded chunks in bulk."""
    results = {"ok": 0, "errors": 0}
    sql = """
        INSERT INTO knowledge_chunks
            (domain, topic, subtopic, content, summary,
             source_citation, source_tier, embedding, token_count)
        VALUES (%s,%s,%s,%s,%s,%s,%s,%s::vector,%s)
        ON CONFLICT DO NOTHING
    """
    with db_conn.cursor() as c:
        for chunk, emb in zip(chunks, embeddings):
            try:
                vec_str = f"[{','.join(str(v) for v in emb)}]"
                c.execute(sql, (
                    chunk["domain"],
                    chunk["topic"],
                    chunk.get("subtopic"),
                    chunk["content"],
                    chunk["summary"],
                    chunk["source_citation"],
                    chunk["source_tier"],
                    vec_str,
                    estimate_tokens(chunk["content"]),
                ))
                results["ok"] += 1
            except Exception as e:
                results["errors"] += 1
                print(f"  Insert error for {chunk.get('topic')}: {e}")
    return results


def run_bulk_ingest(data_dir: Path, db_conn, batch_size: int = 100):
    """Full bulk ingestion pipeline."""
    # Load
    print("Loading chunks...")
    all_chunks = load_all_chunks(data_dir)
    print(f"  Loaded {len(all_chunks)} chunks")

    # Validate
    print("Validating...")
    good, bad = [], []
    for chunk in all_chunks:
        qa = validate_chunk(chunk)
        (good if qa.valid else bad).append(chunk)
    print(f"  Valid: {len(good)}, Invalid: {len(bad)}")

    # Embed in batches
    print(f"Embedding {len(good)} chunks...")
    all_embeddings = batch_embed(good, batch_size=batch_size)

    # Insert
    print(f"Inserting {len(good)} chunks...")
    results = bulk_insert(good, all_embeddings, db_conn)
    print(f"  OK: {results['ok']}, Errors: {results['errors']}")

    # Verify
    with db_conn.cursor() as c:
        c.execute("SELECT COUNT(*) FROM knowledge_chunks")
        total = c.fetchone()[0]
        c.execute("SELECT domain, COUNT(*) FROM knowledge_chunks GROUP BY domain")
        by_domain = c.fetchall()
    print(f"\nFinal DB count: {total}")
    for domain, cnt in by_domain:
        print(f"  {domain}: {cnt}")
```

## Step 6: Batch Ingestion (Multiple Records, with Dedup)

For ongoing ingestion where dedup matters but you have multiple records:

```python
def ingest_batch(chunks: list[dict], skip_dedup: bool = False, db_conn=None) -> dict:
    """Ingest a batch of chunks with per-record status tracking."""
    results = {"ok": 0, "rejected": 0, "duplicate": 0, "errors": 0, "details": []}

    for i, chunk in enumerate(chunks):
        r = ingest_chunk(chunk, skip_dedup=skip_dedup, db_conn=db_conn)
        results[r["status"]] = results.get(r["status"], 0) + 1
        if r["status"] != "ok":
            results["details"].append({"index": i, **r})

    return results
```

## pgvector Table Setup

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE knowledge_chunks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    domain          TEXT NOT NULL,
    topic           TEXT NOT NULL,
    subtopic        TEXT,
    content         TEXT NOT NULL,
    summary         TEXT NOT NULL,
    source_citation TEXT NOT NULL,
    source_tier     TEXT NOT NULL,
    source_year     INTEGER,
    embedding       vector(1536),   -- match your embedding model dimension
    token_count     INTEGER,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Cosine similarity index for fast dedup and search
CREATE INDEX ON knowledge_chunks
    USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);

-- Domain filter index
CREATE INDEX ON knowledge_chunks (domain);
```

## Common Pitfalls

1. **Embedding the wrong text** — Use summary + content combined for richer embeddings, not just content alone
2. **No rate limiting** — Add `time.sleep(0.2)` between batches to avoid 429s from embedding APIs
3. **Skipping validation** — Invalid chunks waste embedding API calls; validate first
4. **Dedup threshold too low** — 0.95 cosine similarity is a good starting point; lower catches more dupes but risks false positives
5. **ON CONFLICT DO NOTHING silently drops** — Log conflicts so you know when inserts are being skipped
6. **Vector string formatting** — pgvector expects `[0.1,0.2,...]` format cast to `::vector`; don't pass Python lists directly
7. **Token estimation** — Track token counts per chunk so you can enforce context window limits during retrieval
