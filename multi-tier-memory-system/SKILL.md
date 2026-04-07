---
name: multi-tier-memory-system
description: Implements three-tier agent memory (short-term cache, medium-term DB, long-term semantic) with dedup, TTL, and context window loading. Use when building any agent that needs persistent memory across conversations, when someone says "add memory to my agent", "the agent forgets everything", or needs to recall past interactions.
metadata:
  author: niyaznurbhasha
  version: 1.0.0
  category: agent-architecture
  tags: [memory, agent, redis, postgres, semantic, dedup, ttl, context-window]
---

# Multi-Tier Memory System

Build a three-tier memory system for LLM agents: short-term (Redis/in-memory), medium-term (relational DB), and long-term (semantic facts with confidence scoring). Each tier serves a different access pattern and retention window.

## Architecture Overview

```
                  Agent Turn
                     |
         +-----------+-----------+
         |           |           |
    Short-Term   Medium-Term   Long-Term
    (Redis)      (Postgres)    (Semantic DB)
    ~6 turns     50 messages   Permanent facts
    TTL: hours   TTL: days     Confidence-scored
```

**Short-term** holds raw recent messages for immediate context (follow-ups, pronoun resolution). **Medium-term** holds conversation summaries and message history. **Long-term** holds extracted semantic facts about the user that persist across sessions.

## Step 1: Define the Short-Term Buffer

The short-term buffer stores the last N raw messages so the model can see verbatim recent turns. This solves follow-up problems like:
- User: "my shoulder hurts" -> Coach: "rate 1-10" -> User: "it's a 10"
- Without the buffer, the model loses "shoulder" context by the third turn.

```python
import json
import redis
from typing import List, Dict

r = redis.Redis.from_url(os.getenv("REDIS_URL"))

BUFFER_KEY_PREFIX = "st_buffer:"
BUFFER_MAXLEN = 12  # ~6 full turns (user + assistant)
BUFFER_TTL_DAYS = 3

def push_buffer_message(thread_id: str, role: str, content: str) -> None:
    """Push a message onto the short-term buffer (newest first)."""
    if not thread_id or not role or not content:
        return
    key = f"{BUFFER_KEY_PREFIX}{thread_id}"
    msg = json.dumps({"role": role, "content": content})
    try:
        r.lpush(key, msg)
        r.ltrim(key, 0, BUFFER_MAXLEN - 1)  # cap length
        r.expire(key, BUFFER_TTL_DAYS * 86400)
    except Exception:
        pass  # best-effort — don't break the agent if cache is down

def get_buffer_messages(thread_id: str) -> List[Dict[str, str]]:
    """Return recent messages in chronological order (oldest first)."""
    key = f"{BUFFER_KEY_PREFIX}{thread_id}"
    try:
        raw = r.lrange(key, 0, BUFFER_MAXLEN - 1)
    except Exception:
        return []
    out = []
    for item in reversed(raw):  # stored newest-first, return oldest-first
        try:
            d = json.loads(item.decode() if isinstance(item, bytes) else str(item))
            if d.get("role") and d.get("content"):
                out.append({"role": d["role"], "content": d["content"]})
        except Exception:
            continue
    return out
```

### Short-Term Scratch and Summaries

Also store ephemeral per-run scratchpad data and rolling conversation summaries:

```python
def set_summary(thread_id: str, text: str, ttl_days: int = 3) -> None:
    """Store a rolling conversation summary."""
    r.setex(f"st_summary:{thread_id}", ttl_days * 86400, text)

def get_summary(thread_id: str) -> str:
    v = r.get(f"st_summary:{thread_id}")
    return v.decode() if v else "No prior summary"

def set_scratch(run_id: str, key: str, value: str, ttl_s: int = 7200) -> None:
    """Per-run scratch space for intermediate node outputs."""
    r.setex(f"scratch:{run_id}:{key}", ttl_s, value)

def get_scratch(run_id: str, key: str) -> str | None:
    v = r.get(f"scratch:{run_id}:{key}")
    return v.decode() if v else None
```

## Step 2: Define the Medium-Term Store (Relational DB)

The medium-term layer stores full message history, events, and structured logs with proper indexing for retrieval.

```python
import psycopg2
from psycopg2.extras import RealDictCursor, Json
from psycopg2.pool import ThreadedConnectionPool
from contextlib import contextmanager

pool = ThreadedConnectionPool(2, 20, os.getenv("DATABASE_URL"))

@contextmanager
def get_conn():
    c = pool.getconn()
    try:
        c.autocommit = True
        yield c
    finally:
        pool.putconn(c)

def write_message(thread_id: str, role: str, content: str,
                  intent: str = None, metadata: dict = None) -> None:
    with get_conn() as conn:
        with conn.cursor() as c:
            c.execute(
                """INSERT INTO messages (id, thread_id, role, content, intent, metadata)
                   VALUES (%s, %s, %s, %s, %s, %s)""",
                (uuid4(), thread_id, role, content, intent,
                 Json(metadata) if metadata else None)
            )

def read_messages(thread_id: str, limit: int = 50) -> list[dict]:
    """Load recent messages for a thread (newest last)."""
    with get_conn() as conn:
        with conn.cursor(cursor_factory=RealDictCursor) as c:
            c.execute(
                """SELECT role, content, created_at, metadata FROM messages
                   WHERE thread_id = %s
                   ORDER BY created_at DESC LIMIT %s""",
                (thread_id, limit)
            )
            rows = c.fetchall()
    return [{"role": r["role"], "content": r["content"]} for r in reversed(rows)]
```

### Event Logging with Dedup

Events represent structured actions the user took. Dedup by payload hash to prevent duplicates:

```python
import hashlib

def write_event(user_id: str, type_: str, payload: dict) -> None:
    """Idempotent event insert (deduped by stable payload hash)."""
    payload_json = json.dumps(payload, sort_keys=True, separators=(",", ":"))
    payload_hash = hashlib.sha256(payload_json.encode()).hexdigest()[:16]
    with get_conn() as conn:
        with conn.cursor() as c:
            c.execute(
                """INSERT INTO events (id, user_id, ts, type, payload, payload_hash)
                   VALUES (%s, %s, now(), %s, %s, %s)
                   ON CONFLICT (user_id, type, payload_hash) DO NOTHING""",
                (uuid4(), user_id, type_, Json(payload), payload_hash)
            )
```

## Step 3: Define the Long-Term Semantic Layer

Semantic facts persist across all sessions. Each fact has a confidence score and is keyed by a semantic key so updates overwrite stale data rather than duplicating.

```python
def upsert_fact(user_id: str, key: str, value: str, confidence: float = 0.8) -> None:
    """Insert or update a semantic fact. Key acts as dedup — same key overwrites."""
    with get_conn() as conn:
        with conn.cursor() as c:
            c.execute(
                """INSERT INTO semantic_facts
                       (id, user_id, fact_key, fact_value, confidence, updated_at)
                   VALUES (%s, %s, %s, %s, %s, now())
                   ON CONFLICT (user_id, fact_key)
                   DO UPDATE SET fact_value = EXCLUDED.fact_value,
                                 confidence = EXCLUDED.confidence,
                                 updated_at = now(),
                                 version = semantic_facts.version + 1""",
                (uuid4(), user_id, key, str(value), confidence)
            )

def list_facts(user_id: str, limit: int = 20) -> list[dict]:
    """Return most-recent semantic facts for context injection."""
    with get_conn() as conn:
        with conn.cursor(cursor_factory=RealDictCursor) as c:
            c.execute(
                """SELECT fact_key, fact_value, confidence, updated_at
                   FROM semantic_facts WHERE user_id = %s
                   ORDER BY updated_at DESC LIMIT %s""",
                (user_id, limit)
            )
            return c.fetchall()

def search_facts(user_id: str, query: str, limit: int = 10) -> list[dict]:
    """Keyword search over fact keys and values."""
    like = f"%{query}%"
    with get_conn() as conn:
        with conn.cursor(cursor_factory=RealDictCursor) as c:
            c.execute(
                """SELECT fact_key, fact_value, confidence, updated_at
                   FROM semantic_facts
                   WHERE user_id = %s
                     AND (fact_key ILIKE %s OR fact_value ILIKE %s)
                   ORDER BY updated_at DESC LIMIT %s""",
                (user_id, like, like, limit)
            )
            return c.fetchall()
```

## Step 4: Load Context for Each Turn

At the start of each agent turn, assemble the context window from all three tiers:

```python
def load_memory_context(thread_id: str, user_id: str) -> dict:
    """Assemble multi-tier memory for prompt injection."""
    return {
        # Tier 1: raw recent messages (verbatim follow-up context)
        "short_term_buffer": get_buffer_messages(thread_id),
        # Tier 1: rolling summary of conversation so far
        "conversation_summary": get_summary(thread_id),
        # Tier 2: full message history (for RAG or retrieval)
        "message_history": read_messages(thread_id, limit=50),
        # Tier 3: long-term facts about this user
        "semantic_facts": list_facts(user_id, limit=20),
    }
```

Inject into the system prompt like:

```python
def build_memory_prompt(ctx: dict) -> str:
    parts = []
    if ctx["conversation_summary"] != "No prior summary":
        parts.append(f"Conversation so far: {ctx['conversation_summary']}")
    if ctx["semantic_facts"]:
        facts = "\n".join(f"- {f['fact_key']}: {f['fact_value']}" for f in ctx["semantic_facts"])
        parts.append(f"What you know about this user:\n{facts}")
    return "\n\n".join(parts)
```

## Step 5: Write-Back After Each Turn

After the agent responds, update all three tiers:

```python
def post_turn_writeback(thread_id: str, user_id: str, user_msg: str,
                        assistant_msg: str, extracted_facts: list[dict]) -> None:
    # Tier 1: push both messages to short-term buffer
    push_buffer_message(thread_id, "user", user_msg)
    push_buffer_message(thread_id, "assistant", assistant_msg)

    # Tier 2: persist messages to DB
    write_message(thread_id, "user", user_msg)
    write_message(thread_id, "assistant", assistant_msg)

    # Tier 3: upsert any extracted semantic facts
    for fact in extracted_facts:
        upsert_fact(user_id, fact["key"], fact["value"], fact.get("confidence", 0.8))
```

## Step 6: Caching Layer for Expensive Computations

Cache embeddings, person models, or any expensive computation in Redis with TTL:

```python
def cache_embedding(key: str, embedding: list, ttl_s: int = 86400) -> None:
    try:
        r.setex(key, ttl_s, json.dumps(embedding))
    except Exception:
        pass

def get_cached_embedding(key: str) -> list[float] | None:
    try:
        v = r.get(key)
        return json.loads(v.decode()) if v else None
    except Exception:
        return None

def get_person_model_cache(user_id: str) -> str | None:
    """Cache computed person models (expensive LLM aggregation) for 5 minutes."""
    v = r.get(f"person_model:{user_id}")
    return v.decode() if v else None

def set_person_model_cache(user_id: str, model_str: str, ttl: int = 300) -> None:
    r.setex(f"person_model:{user_id}", ttl, model_str)

def invalidate_person_model_cache(user_id: str) -> None:
    """Invalidate when new facts are written that would change the model."""
    r.delete(f"person_model:{user_id}")
```

## Step 7: Follow-Up Context (Short-Lived State)

For multi-turn flows where the agent asks a question and needs to remember what it asked:

```python
FOLLOWUP_TTL = 600  # 10 minutes

def set_follow_up_context(thread_id: str, context: dict) -> None:
    r.setex(f"follow_up:{thread_id}", FOLLOWUP_TTL, json.dumps(context))

def get_follow_up_context(thread_id: str) -> dict | None:
    """Retrieve and consume follow-up context (auto-deleted after read)."""
    key = f"follow_up:{thread_id}"
    v = r.get(key)
    if v is None:
        return None
    r.delete(key)  # consume on read
    return json.loads(v.decode())
```

## Schema Reference

Minimum tables needed:

```sql
CREATE TABLE messages (
    id UUID PRIMARY KEY,
    thread_id TEXT NOT NULL,
    role TEXT NOT NULL,
    content TEXT NOT NULL,
    intent TEXT,
    metadata JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_messages_thread ON messages(thread_id, created_at);

CREATE TABLE events (
    id UUID PRIMARY KEY,
    user_id TEXT NOT NULL,
    ts TIMESTAMPTZ DEFAULT NOW(),
    type TEXT NOT NULL,
    payload JSONB,
    payload_hash TEXT,
    UNIQUE(user_id, type, payload_hash)
);

CREATE TABLE semantic_facts (
    id UUID PRIMARY KEY,
    user_id TEXT NOT NULL,
    fact_key TEXT NOT NULL,
    fact_value TEXT NOT NULL,
    confidence FLOAT DEFAULT 0.8,
    version INT DEFAULT 1,
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(user_id, fact_key)
);
CREATE INDEX idx_facts_user ON semantic_facts(user_id, updated_at DESC);
```

## Design Principles

1. **Best-effort short-term**: Redis operations use try/except with pass. If Redis goes down, the agent still works — it just loses recent context. Never let cache failures crash the agent.
2. **Dedup everything**: Events use payload hashing. Facts use upsert on semantic key. Messages are append-only but idempotent via UUID.
3. **TTL by tier**: Short-term (hours), summaries (days), facts (permanent). Match TTL to how long the information stays relevant.
4. **Confidence scoring**: Facts track confidence so the agent can weight recent high-confidence facts over old low-confidence ones.
5. **Consume-on-read for follow-ups**: Follow-up context auto-deletes after retrieval to prevent stale state from leaking into future turns.
6. **Invalidate caches when source data changes**: When new facts are written, invalidate any cached aggregations (person models, insight summaries) that depend on them.
