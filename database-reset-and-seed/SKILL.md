---
name: database-reset-and-seed
description: Idempotent database reset script that drops schemas, applies SQL migrations, and seeds reference data with partial rollback on failure. Use when setting up any PostgreSQL dev workflow, when user says "reset database", "seed data", "db setup", or needs a dev database lifecycle script.
metadata:
  author: niyaznurbhasha
  version: 1.0.0
  category: infrastructure
  tags: [database, postgres, reset, seed, migration, sql, python]
---

# Database Reset and Seed

Idempotent database reset script for PostgreSQL: drop and recreate schemas, apply SQL migrations in order, seed reference and test data, with fallback strategies on failure.

## When to Use

- Setting up a development database lifecycle (reset, migrate, seed)
- Need a one-command "nuke and rebuild" for local dev
- Multiple SQL schema files that must be applied in order
- Want idempotent seeding (safe to run multiple times)
- Need separate persistent schemas (reference data) vs ephemeral schemas (user data)

## Architecture

```
project/
  sql/
    core_schema.sql         # Main tables (users, orders, etc.)
    reference_schema.sql    # Lookup tables that survive resets
    domain_schema.sql       # Domain-specific tables
  scripts/
    reset_db.py             # This script
    seed_data.py            # Seed functions
  .env                      # DATABASE_URL
```

## Step-by-Step Implementation

### Step 1: Load environment and read SQL files

```python
import os
import sys
from pathlib import Path
from dotenv import load_dotenv
import psycopg2
from psycopg2.extras import Json


def _load_env() -> None:
    """Find and load .env from repo root."""
    here = Path(__file__).resolve()
    repo_root = here.parents[2]  # adjust depth for your layout
    env_path = repo_root / ".env"
    if env_path.exists():
        load_dotenv(env_path)


def _read_sql(path: Path) -> str:
    return path.read_text()
```

### Step 2: Apply persistent schemas first

Some schemas (reference data, lookup tables) should survive resets. Apply these before dropping anything.

```python
def apply_reference_schema(conn) -> None:
    """Apply reference schema that survives resets."""
    sql_dir = Path(__file__).resolve().parent.parent / "sql"
    ref = sql_dir / "reference_schema.sql"
    with conn.cursor() as c:
        c.execute(_read_sql(ref))
        # Add columns that were added after initial table creation
        c.execute("ALTER TABLE reference.lookup ADD COLUMN IF NOT EXISTS new_col TEXT")
    print("Applied reference schema")
```

### Step 3: Apply core schema files in order

```python
def apply_schema_files(conn) -> None:
    """Apply all schema files in dependency order."""
    sql_dir = Path(__file__).resolve().parent.parent / "sql"

    schema_files = [
        sql_dir / "extensions.sql",       # CREATE EXTENSION IF NOT EXISTS
        sql_dir / "core_schema.sql",       # Users, threads, messages
        sql_dir / "domain_schema.sql",     # Domain-specific tables
    ]

    with conn.cursor() as c:
        for schema_file in schema_files:
            if schema_file.exists():
                c.execute(_read_sql(schema_file))
                print(f"Applied {schema_file.name}")
```

### Step 4: Write idempotent seed functions

Every seed function uses `ON CONFLICT DO NOTHING` so it can run repeatedly without errors.

```python
import uuid

def seed_test_user(conn) -> None:
    """Seed a test user for development/eval."""
    with conn.cursor() as c:
        c.execute(
            """
            INSERT INTO users(id, display_name, email)
            VALUES (%s, %s, %s)
            ON CONFLICT (id) DO NOTHING;
            """,
            ("user_1", "Test User", "test@example.com"),
        )


def seed_reference_data(conn) -> None:
    """Seed lookup/reference tables."""
    with conn.cursor() as c:
        c.execute(
            """
            INSERT INTO reference.categories(id, name)
            VALUES ('cat_1', 'Default Category')
            ON CONFLICT (id) DO NOTHING;
            """,
        )


def seed_default_config(conn, config_path: Path) -> None:
    """Seed a JSON config from a file."""
    if not config_path.exists():
        return
    try:
        config_obj = json.loads(config_path.read_text())
    except Exception:
        return

    config_id = str(uuid.uuid4())
    with conn.cursor() as c:
        c.execute(
            """
            INSERT INTO configs (id, name, body, is_active)
            VALUES (%s, %s, %s, true)
            """,
            (config_id, config_obj.get("name", "default"), Json(config_obj)),
        )
```

### Step 5: Implement the reset function with fallback

Primary strategy: DROP and CREATE the public schema (fastest, cleanest). Fallback: TRUNCATE all tables (works when DROP is not permitted).

```python
def reset_db(database_url: str) -> None:
    conn = psycopg2.connect(database_url)
    conn.autocommit = True

    def _seed_all(conn_obj):
        """Run all seed functions with error isolation."""
        try:
            seed_reference_data(conn_obj)
            print("Seeded reference data")
        except Exception as e:
            print(f"WARNING: Failed to seed reference data: {e}")

        try:
            seed_test_user(conn_obj)
            print("Seeded test user")
        except Exception as e:
            print(f"WARNING: Failed to seed test user: {e}")

    with conn.cursor() as c:
        # 1. Apply persistent schemas first (survive reset)
        apply_reference_schema(conn)

        # 2. Drop and recreate public schema
        try:
            c.execute("SELECT current_user")
            user = c.fetchone()[0]

            c.execute("DROP SCHEMA public CASCADE;")
            c.execute("CREATE SCHEMA public;")
            c.execute(f'GRANT ALL ON SCHEMA public TO "{user}";')
            c.execute("GRANT ALL ON SCHEMA public TO public;")

            print("Reset: DROP/CREATE public schema")
            apply_schema_files(conn)
            _seed_all(conn)
            return
        except Exception as e:
            print(f"DROP/CREATE failed, falling back to TRUNCATE: {e}")

        # 3. Fallback: truncate all public tables
        c.execute("""
            SELECT tablename FROM pg_tables
            WHERE schemaname = 'public'
            ORDER BY tablename
        """)
        tables = [r[0] for r in c.fetchall()]
        if not tables:
            print("No tables found. Nothing to reset.")
            return

        joined = ", ".join(f'"{t}"' for t in tables)
        c.execute(f"TRUNCATE TABLE {joined} CASCADE;")
        apply_schema_files(conn)
        _seed_all(conn)
        print(f"Reset: TRUNCATED {len(tables)} tables")
```

### Step 6: Add companion service resets

If you use Redis, Elasticsearch, or other stores alongside PostgreSQL, reset them too.

```python
def reset_cache() -> None:
    """Flush Redis cache."""
    import redis
    redis_url = os.getenv("REDIS_URL")
    if not redis_url:
        print("WARNING: REDIS_URL not set, skipping cache reset")
        return
    try:
        r = redis.from_url(redis_url)
        r.flushall()
        print("Redis flushed")
    except Exception as e:
        print(f"WARNING: Failed to flush Redis: {e}")
```

### Step 7: CLI entrypoint

```python
def main() -> None:
    _load_env()
    db_url = os.getenv("DATABASE_URL")
    if not db_url:
        print("ERROR: DATABASE_URL must be set.", file=sys.stderr)
        sys.exit(1)

    reset_db(db_url)


if __name__ == "__main__":
    main()
```

## SQL Schema Patterns

Use `IF NOT EXISTS` and `IF NOT NULL` everywhere for idempotency:

```sql
CREATE TABLE IF NOT EXISTS users (
    id TEXT PRIMARY KEY,
    display_name TEXT,
    email TEXT UNIQUE,
    created_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX IF NOT EXISTS idx_users_email ON users(email);

-- Add columns safely after table exists
ALTER TABLE users ADD COLUMN IF NOT EXISTS phone TEXT;
```

## Key Principles

- **Persistent vs ephemeral schemas**: Reference/lookup data in a separate schema that survives resets. User data in `public` schema that gets dropped.
- **Fallback strategy**: Try DROP CASCADE first (cleanest). Fall back to TRUNCATE CASCADE if permissions block DROP.
- **Idempotent seeds**: Every INSERT uses `ON CONFLICT DO NOTHING`. Safe to run 100 times.
- **Error isolation in seeding**: Each seed function is wrapped in try/except. One failure does not block others. Always print warnings, never silently fail.
- **`autocommit = True`**: Schema DDL (DROP SCHEMA, CREATE SCHEMA) requires autocommit in psycopg2.

## Dependencies

```
psycopg2-binary
python-dotenv
```
