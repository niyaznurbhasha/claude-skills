---
name: structured-logging-setup
description: structlog setup with JSON/pretty console modes, context binding, per-component loggers, and noisy library suppression. Drop-in for any Python app. Use when setting up logging, when user says "add logging", "structured logs", or needs observable log output.
metadata:
  author: niyaznurbhasha
  version: 1.0.0
  category: infrastructure
  tags: [logging, structlog, observability, json, python]
---

# Structured Logging Setup

Configure structlog for any Python application with JSON output for production and pretty console output for development, per-component loggers, context binding, and stdlib noise suppression.

## When to Use

- Setting up logging for any Python project
- Replacing `print()` statements with structured logs
- Need JSON logs for log aggregation (ELK, Datadog, CloudWatch)
- Want pretty colored console output during development
- Multiple components need their own logger identity

## Step-by-Step Implementation

### Step 1: Create the logging module

One `setup_logging()` call at startup, one `get_logger()` factory for components.

```python
import logging
import sys
from typing import Any

import structlog


def setup_logging(level: str = "INFO", json_output: bool = False) -> None:
    """Configure structlog. Call once at application startup."""
    log_level = getattr(logging, level.upper(), logging.INFO)

    shared_processors: list[Any] = [
        structlog.contextvars.merge_contextvars,
        structlog.stdlib.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
    ]

    if json_output:
        processors = shared_processors + [
            structlog.processors.dict_tracebacks,
            structlog.processors.JSONRenderer(),
        ]
    else:
        processors = shared_processors + [
            structlog.dev.ConsoleRenderer(colors=True),
        ]

    structlog.configure(
        processors=processors,
        wrapper_class=structlog.make_filtering_bound_logger(log_level),
        context_class=dict,
        logger_factory=structlog.PrintLoggerFactory(file=sys.stderr),
        cache_logger_on_first_use=True,
    )

    # Silence noisy stdlib loggers
    logging.getLogger("httpx").setLevel(logging.WARNING)
    logging.getLogger("httpcore").setLevel(logging.WARNING)
    logging.getLogger("websockets").setLevel(logging.WARNING)
    logging.getLogger("urllib3").setLevel(logging.WARNING)


def get_logger(name: str, **initial_ctx: Any) -> structlog.stdlib.BoundLogger:
    """Get a named logger with optional initial context."""
    return structlog.get_logger(name).bind(**initial_ctx)
```

### Step 2: Call setup at application startup

```python
from logging_setup import setup_logging, get_logger

# Dev mode — pretty console output
setup_logging(level="DEBUG", json_output=False)

# Production — JSON for log aggregation
setup_logging(level="INFO", json_output=True)
```

### Step 3: Create per-component loggers

Each module or class gets its own named logger. The name appears in every log line, making it easy to filter.

```python
# In any module:
from logging_setup import get_logger

log = get_logger("api.users")

def create_user(email: str):
    log.info("creating_user", email=email)
    # ...
    log.info("user_created", email=email, user_id=user_id)
```

### Step 4: Use context binding for request-scoped data

Bind context once (e.g., request ID, user ID) and it appears in all subsequent log calls.

```python
import structlog

# At the start of a request handler:
structlog.contextvars.clear_contextvars()
structlog.contextvars.bind_contextvars(
    request_id=request_id,
    user_id=user_id,
)

# All log calls in this request context now include request_id and user_id
log.info("processing_order", order_id=order_id)
# Output: {"event": "processing_order", "order_id": "abc", "request_id": "xyz", "user_id": "u1", ...}
```

### Step 5: Use structured key-value pairs, not string formatting

```python
# Good — structured, searchable, parseable
log.info("order_completed", order_id=order_id, total=total, items=item_count)

# Bad — unstructured string
log.info(f"Order {order_id} completed for ${total} with {item_count} items")
```

### Step 6: Suppress noisy libraries

Add any library that floods your logs at DEBUG/INFO level:

```python
logging.getLogger("sqlalchemy.engine").setLevel(logging.WARNING)
logging.getLogger("boto3").setLevel(logging.WARNING)
logging.getLogger("botocore").setLevel(logging.WARNING)
logging.getLogger("asyncio").setLevel(logging.WARNING)
```

## Output Examples

**Console mode** (development):
```
2024-01-15T10:30:45Z [info     ] creating_user    email=user@example.com  [api.users]
2024-01-15T10:30:45Z [info     ] user_created     email=user@example.com user_id=u123  [api.users]
```

**JSON mode** (production):
```json
{"event": "creating_user", "email": "user@example.com", "level": "info", "timestamp": "2024-01-15T10:30:45Z", "logger": "api.users"}
```

## Key Principles

- **One setup call**: `setup_logging()` runs once at startup. Never configure logging in individual modules.
- **Named loggers**: Every component gets a name via `get_logger("component.name")`. This makes filtering trivial.
- **Structured data**: Always use keyword arguments, never format strings. `log.info("event", key=value)` not `log.info(f"event: {value}")`.
- **Context binding**: Use `contextvars` for request-scoped data that should appear in every log line automatically.
- **Stderr output**: Logs go to stderr so stdout remains clean for actual program output.

## Dependencies

```
structlog>=23.0
```
