# Claude Skills

A collection of 38 reusable [Claude Skills](https://docs.anthropic.com/en/docs/build-with-claude/skills) for productivity, research, and development workflows. Extracted from building 12 projects.

## Skills

### Thinking & Productivity

| Skill | Description |
|-------|-------------|
| [brain-dump](./brain-dump/) | Dump everything on your mind — auto-organizes into structured entries with a review step before writing |
| [journal-capture](./journal-capture/) | Structured capture of decisions, reflections, brainstorms, progress, and teardowns |
| [journal-review](./journal-review/) | Review and correct categorizations — move items between projects, fix types, mark complete |
| [project-insights](./project-insights/) | Deep project health analysis across 5 dimensions |
| [daily-digest](./daily-digest/) | Personalized daily briefing from journal + project data |
| [idea-extraction-batch-pipeline](./idea-extraction-batch-pipeline/) | Batch LLM extraction from conversation history with parallel agents, merge, and dedup |
| [idea-grouping-taxonomy](./idea-grouping-taxonomy/) | Taxonomy-based grouping of 100+ items using LLM agents with validation and correction passes |

### Agent Architecture

| Skill | Description |
|-------|-------------|
| [multi-tier-memory-system](./multi-tier-memory-system/) | Three-tier agent memory (short-term cache, medium-term DB, long-term semantic) |
| [langgraph-agentic-pipeline](./langgraph-agentic-pipeline/) | Multi-node LangGraph StateGraph with interactive/background/full execution modes |
| [goal-task-decomposition](./goal-task-decomposition/) | Break goals into subtasks with dependency tracking and dispatch routing |
| [failure-learning-system](./failure-learning-system/) | Log failures with context, extract patterns, apply learned corrections |
| [simtom-knowledge-isolation](./simtom-knowledge-isolation/) | Isolate LLM character knowledge via boundary re-injection every N turns |

### Evaluation & Quality

| Skill | Description |
|-------|-------------|
| [llm-as-judge-eval-framework](./llm-as-judge-eval-framework/) | Rubric-based LLM evaluation with deterministic/hybrid/judge modes |
| [ablation-study-framework](./ablation-study-framework/) | Systematic eval of augmentation layers (baseline → +RAG → +rules) |
| [quality-trend-detection](./quality-trend-detection/) | Track quality metrics over time, detect regressions and plateaus |

### Data Pipelines

| Skill | Description |
|-------|-------------|
| [structured-data-extraction-pipeline](./structured-data-extraction-pipeline/) | LLM extraction from free-form input with validation and canonicalization |
| [vector-db-ingestion](./vector-db-ingestion/) | Bulk ingest with batched embeddings, validation, dedup, pgvector insertion |
| [parquet-storage-with-wal](./parquet-storage-with-wal/) | Thread-safe append buffer with WAL, atomic Parquet flush, Hive partitions |
| [streaming-dataset-pipeline](./streaming-dataset-pipeline/) | Multi-stage pipeline (download → extract → filter → join → assemble) with resumable checkpoints |

### Time-Series & Analytics

| Skill | Description |
|-------|-------------|
| [statistical-signal-detection](./statistical-signal-detection/) | z-scores, EWMA, Mann-Kendall, PELT change points, isolation forest |
| [duckdb-analytics-layer](./duckdb-analytics-layer/) | DuckDB + Parquet with hive-partition pruning and streaming batch processing |

### Finance & Backtesting

| Skill | Description |
|-------|-------------|
| [event-study-stats-suite](./event-study-stats-suite/) | BMP t-test, Kolari-Pynnonen, Corrado rank, bootstrap CARs, BH-FDR correction |
| [event-driven-backtest-engine](./event-driven-backtest-engine/) | Streaming backtests with pluggable strategy ABC and settlement interleaving |
| [event-study-full-pipeline](./event-study-full-pipeline/) | End-to-end: define events → fetch prices → compute CARs → statistical tests → report |

### Infrastructure

| Skill | Description |
|-------|-------------|
| [pydantic-typed-config-loader](./pydantic-typed-config-loader/) | Environment-based config via pydantic-settings with fail-fast validation |
| [structured-logging-setup](./structured-logging-setup/) | structlog with JSON/pretty modes, context binding, library suppression |
| [async-http-websocket-patterns](./async-http-websocket-patterns/) | Rate-limited async HTTP + WebSocket with backoff and reconnection |
| [database-reset-and-seed](./database-reset-and-seed/) | Idempotent DB reset with migrations, seeding, and partial rollback |
| [system-health-monitoring](./system-health-monitoring/) | Heartbeat file + watchdog + push alerts for long-running services |
| [safe-commit](./safe-commit/) | Security-checked git commits with change classification |

### Dashboards & UI

| Skill | Description |
|-------|-------------|
| [fastapi-sse-live-dashboard](./fastapi-sse-live-dashboard/) | FastAPI + SSE for real-time dashboards with demo mode and HTMX |
| [textual-tui-reactive](./textual-tui-reactive/) | Textual TUI with DataTables, reactive properties, async workers |
| [multi-tab-dashboard-renderer](./multi-tab-dashboard-renderer/) | Multi-tab HTML dashboard from heterogeneous data sources |

### Interview & Research

| Skill | Description |
|-------|-------------|
| [high-information-gain-interviewer](./high-information-gain-interviewer/) | LLM interviewer maximizing info gain with dynamic investigation plans |
| [interview-belief-extractor](./interview-belief-extractor/) | Post-interview belief extraction with behavioral confidence scoring |
| [research-gap-finder](./research-gap-finder/) | Discover open research gaps from academic papers via arxiv + Semantic Scholar |

### Misc

| Skill | Description |
|-------|-------------|
| [keyword-filter-framework](./keyword-filter-framework/) | Compiled regex filtering with bot exclusion and configurable thresholds |
| [chaos-controlled-behavior-system](./chaos-controlled-behavior-system/) | Single parameter (0-1) → 10+ derived behavior metrics for interactive systems |

## Installation

### Claude.ai
1. Download/clone this repo
2. Zip the skill folder you want (e.g., `safe-commit/`)
3. Go to Claude.ai → Settings → Capabilities → Skills
4. Upload the zip

### Claude Code
Place skill folders in your project directory or a shared skills directory. Claude Code will discover them automatically.

### API
Use the `/v1/skills` endpoint. See [Skills API docs](https://docs.anthropic.com/en/docs/build-with-claude/skills).

## Customization

Each skill's `SKILL.md` is designed to be forked and adapted. Skills are domain-agnostic — swap in your own schemas, project names, and configurations.

## License

MIT
