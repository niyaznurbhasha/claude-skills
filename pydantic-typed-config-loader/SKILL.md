---
name: pydantic-typed-config-loader
description: Environment-based config via pydantic-settings with type safety, fail-fast validation, and environment separation (dev/prod). Use when setting up any Python project configuration, when user says "add config", "environment variables", "settings", or needs typed configuration management.
metadata:
  author: niyaznurbhasha
  version: 1.0.0
  category: infrastructure
  tags: [config, pydantic, settings, environment, validation, python]
---

# Pydantic Typed Config Loader

Set up typed, validated, environment-aware configuration for Python projects using pydantic-settings. Fail-fast on bad config at startup, not at runtime.

## When to Use

- Starting a new Python project that reads environment variables
- Replacing raw `os.getenv()` calls with typed config
- Setting up dev/prod environment separation
- Any service that needs validated configuration at startup

## Architecture

```
project/
  config.py          # All settings classes + load_settings()
  secrets/
    .env.shared      # Shared defaults (committed)
    .env.demo        # Demo/dev overrides (committed)
    .env.production   # Prod secrets (gitignored)
```

## Step-by-Step Implementation

### Step 1: Define an environment enum

Create an enum for environment modes. This prevents typos and gives autocomplete.

```python
from enum import Enum

class AppEnv(str, Enum):
    DEV = "dev"
    PRODUCTION = "production"
```

### Step 2: Create settings classes with `pydantic-settings`

Each logical group of config gets its own class. Use `env_prefix` to namespace environment variables.

```python
from pathlib import Path
from typing import Optional
from pydantic import field_validator
from pydantic_settings import BaseSettings, SettingsConfigDict

class AppSettings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="APP_")

    env: AppEnv = AppEnv.DEV
    data_dir: Path = Path("data")
    log_level: str = "INFO"

    @property
    def is_production(self) -> bool:
        return self.env == AppEnv.PRODUCTION


class DatabaseSettings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="DB_")

    host: str = "localhost"
    port: int = 5432
    name: str = "myapp"
    user: str = "postgres"
    password: str = ""

    @property
    def url(self) -> str:
        return f"postgresql://{self.user}:{self.password}@{self.host}:{self.port}/{self.name}"


class ExternalAPISettings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="API_")

    key: str = ""
    base_url: str = "https://api.example.com/v1"
    rate_limit_rps: int = 10
```

### Step 3: Add validators for complex fields

Use `field_validator` for paths that need expansion, files that must exist, or values that need transformation.

```python
class CredentialSettings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="CRED_")

    key_path: Path = Path.home() / ".credentials" / "key.pem"

    @field_validator("key_path", mode="before")
    @classmethod
    def expand_path(cls, v: str | Path) -> Path:
        return Path(v).expanduser()

    def validate_key_exists(self) -> None:
        """Call at startup to fail fast on missing credentials."""
        if not self.key_path.exists():
            raise FileNotFoundError(f"Key not found: {self.key_path}")
        mode = self.key_path.stat().st_mode & 0o777
        if mode & 0o077:
            raise PermissionError(
                f"Key {self.key_path} has insecure permissions ({oct(mode)}). "
                f"Run: chmod 600 {self.key_path}"
            )
```

### Step 4: Module-level singletons with explicit initialization

Never auto-construct settings. Use a `load_settings()` function that must be called once at startup. Accessor functions raise if called before init.

```python
_app: Optional[AppSettings] = None
_db: Optional[DatabaseSettings] = None

def settings() -> AppSettings:
    if _app is None:
        raise RuntimeError("Call load_settings() at startup before accessing settings.")
    return _app

def db_settings() -> DatabaseSettings:
    if _db is None:
        raise RuntimeError("Call load_settings() at startup before accessing settings.")
    return _db
```

### Step 5: Implement `load_settings()` with environment layering

Load `.env` files in order: shared defaults first, then environment-specific overrides. Validate everything before returning.

```python
import os
import sys

def load_settings(env_override: Optional[str] = None) -> AppSettings:
    """Load and validate all settings. Fails fast on missing/invalid config."""
    global _app, _db
    from dotenv import load_dotenv

    env = env_override or os.getenv("APP_ENV", "dev")
    os.environ.setdefault("APP_ENV", env)

    secrets_dir = Path("secrets")
    load_dotenv(secrets_dir / ".env.shared", override=False)
    load_dotenv(secrets_dir / f".env.{env}", override=True)

    _app = AppSettings()
    _db = DatabaseSettings()

    if _app.is_production:
        _production_confirmation()

    return _app


def _production_confirmation() -> None:
    """Safety gate for production mode."""
    print("=" * 60, file=sys.stderr)
    print("  WARNING: RUNNING IN PRODUCTION MODE", file=sys.stderr)
    print("=" * 60, file=sys.stderr)
    confirm = input("Type 'PRODUCTION' to confirm: ")
    if confirm != "PRODUCTION":
        sys.exit("Aborted.")
```

### Step 6: Use in application entrypoint

```python
# main.py
from config import load_settings, settings, db_settings

def main():
    cfg = load_settings()
    print(f"Running in {cfg.env.value} mode")
    print(f"Database: {db_settings().url}")
    print(f"Log level: {settings().log_level}")
```

## Key Principles

- **Fail fast**: All validation happens at startup in `load_settings()`, not when a field is first accessed at runtime.
- **Explicit init**: Settings are `None` until `load_settings()` is called. Accessing them early raises immediately rather than silently using defaults.
- **Environment layering**: `.env.shared` for defaults, `.env.{env}` for overrides. Production secrets never committed.
- **Type safety**: Pydantic coerces and validates types automatically. `port: int` rejects `"abc"` at load time.
- **Grouped by concern**: Each external service or subsystem gets its own settings class with its own `env_prefix`.

## Dependencies

```
pydantic>=2.0
pydantic-settings>=2.0
python-dotenv
```
