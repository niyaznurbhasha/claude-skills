---
name: system-health-monitoring
description: Heartbeat file + cron watchdog + push notification alerts for long-running services. Use when deploying any background service, when user says "monitoring", "heartbeat", "watchdog", "health check", or needs to know if a service is alive.
metadata:
  author: niyaznurbhasha
  version: 1.0.0
  category: infrastructure
  tags: [monitoring, heartbeat, watchdog, health-check, cron, python]
---

# System Health Monitoring

Implement heartbeat files, health status tracking, and external watchdog alerting for long-running background services.

## When to Use

- Deploying a long-running service (collector, worker, daemon)
- Need to detect stalled or crashed processes
- Want push notifications when a service goes down
- Building a health endpoint or status page

## Architecture

```
Service Process
  |-- REST poll task
  |-- WebSocket task
  |-- Heartbeat task  <-- writes JSON file every N seconds
        |
        v
  data/.heartbeat     <-- JSON file on disk
        |
        v
  Cron watchdog        <-- reads file, alerts if stale
        |
        v
  Push notification    <-- ntfy, Pushover, email, etc.
```

## Step-by-Step Implementation

### Step 1: Write a rich heartbeat file from your service

The heartbeat is not just a timestamp. Include operational metrics so the watchdog (or a dashboard) can assess health at a glance.

```python
import asyncio
import json
import time
from pathlib import Path


class HealthMonitor:
    """Writes periodic heartbeat files with operational metrics."""

    def __init__(self, heartbeat_path: Path, interval_sec: int = 30) -> None:
        self._path = heartbeat_path
        self._interval = interval_sec
        self._path.parent.mkdir(parents=True, exist_ok=True)

        # Metrics — updated by other components
        self.active_items: int = 0
        self.pending_rows: int = 0
        self.last_poll_ts: float = 0.0
        self.last_event_ts: float = 0.0
        self.error_count: int = 0

    async def run(self) -> None:
        """Heartbeat loop — run as an asyncio task."""
        while True:
            self._write()
            await asyncio.sleep(self._interval)

    def _write(self) -> None:
        self._path.write_text(json.dumps({
            "ts": time.time(),
            "active_items": self.active_items,
            "pending_rows": self.pending_rows,
            "last_poll": self.last_poll_ts,
            "last_event": self.last_event_ts,
            "error_count": self.error_count,
        }))
```

### Step 2: Launch the heartbeat as a background task

```python
import asyncio
import signal

class ServiceRunner:
    def __init__(self, data_dir: Path) -> None:
        self._health = HealthMonitor(data_dir / ".heartbeat")

    async def run(self) -> None:
        stop_event = asyncio.Event()
        loop = asyncio.get_running_loop()
        for sig in (signal.SIGINT, signal.SIGTERM):
            loop.add_signal_handler(sig, stop_event.set)

        tasks = [
            asyncio.create_task(self._main_work_loop(), name="worker"),
            asyncio.create_task(self._health.run(), name="heartbeat"),
        ]

        # If any task crashes, trigger shutdown
        def _on_crash(task: asyncio.Task) -> None:
            if not task.cancelled() and task.exception():
                stop_event.set()

        for task in tasks:
            task.add_done_callback(_on_crash)

        try:
            await stop_event.wait()
        finally:
            for task in tasks:
                task.cancel()
            await asyncio.gather(*tasks, return_exceptions=True)
```

### Step 3: Read heartbeat from a dashboard or health endpoint

```python
import time
from pathlib import Path

def check_service_health(heartbeat_path: Path) -> dict:
    """Read heartbeat file and determine service status."""
    if not heartbeat_path.exists():
        return {"status": "unknown", "heartbeat_age_sec": None}

    try:
        data = json.loads(heartbeat_path.read_text())
    except (json.JSONDecodeError, OSError):
        return {"status": "error", "heartbeat_age_sec": None}

    age = time.time() - data.get("ts", 0)

    if age < 60:
        status = "running"
    elif age < 300:
        status = "stale"
    else:
        status = "dead"

    return {
        "status": status,
        "heartbeat_age_sec": round(age, 1),
        "active_items": data.get("active_items", 0),
        "pending_rows": data.get("pending_rows", 0),
        "error_count": data.get("error_count", 0),
    }
```

### Step 4: Create a cron watchdog script

This runs every 5 minutes via cron. If the heartbeat is stale, it sends an alert.

```python
#!/usr/bin/env python3
"""Watchdog: check heartbeat file, alert if stale."""

import json
import sys
import time
from pathlib import Path

HEARTBEAT_PATH = Path("data/.heartbeat")
MAX_AGE_SEC = 120  # Alert if older than 2 minutes
NTFY_TOPIC = "my-service-alerts"  # or use Pushover, email, etc.


def check_and_alert() -> None:
    if not HEARTBEAT_PATH.exists():
        send_alert("Heartbeat file missing — service may not be running")
        sys.exit(1)

    try:
        data = json.loads(HEARTBEAT_PATH.read_text())
    except Exception:
        send_alert("Heartbeat file corrupted")
        sys.exit(1)

    age = time.time() - data.get("ts", 0)
    if age > MAX_AGE_SEC:
        send_alert(
            f"Service heartbeat stale ({age:.0f}s old). "
            f"Last poll: {data.get('last_poll', 'unknown')}, "
            f"Errors: {data.get('error_count', '?')}"
        )
        sys.exit(1)

    print(f"OK: heartbeat {age:.0f}s old")


def send_alert(message: str) -> None:
    """Send push notification via ntfy.sh (free, no signup)."""
    import urllib.request
    try:
        req = urllib.request.Request(
            f"https://ntfy.sh/{NTFY_TOPIC}",
            data=message.encode(),
            headers={"Title": "Service Alert", "Priority": "high"},
        )
        urllib.request.urlopen(req)
        print(f"ALERT sent: {message}")
    except Exception as e:
        print(f"Failed to send alert: {e}", file=sys.stderr)


if __name__ == "__main__":
    check_and_alert()
```

### Step 5: Set up the cron job

```bash
# Check every 5 minutes
*/5 * * * * cd /path/to/project && /path/to/python watchdog.py >> /var/log/watchdog.log 2>&1
```

### Step 6: Display health in a dashboard

```python
def format_health_display(health: dict) -> str:
    """Format health data for display."""
    status = health["status"]
    age = health["heartbeat_age_sec"]

    if status == "running":
        status_line = f"RUNNING (heartbeat {age:.1f}s ago)"
    elif status == "stale":
        status_line = f"STALE (heartbeat {age:.1f}s ago — may be stuck)"
    elif status == "dead":
        status_line = f"DEAD (heartbeat {age:.1f}s ago)"
    else:
        status_line = "UNKNOWN (no heartbeat file)"

    return (
        f"Status:       {status_line}\n"
        f"Active items: {health.get('active_items', '?')}\n"
        f"Pending rows: {health.get('pending_rows', '?')}\n"
        f"Errors:       {health.get('error_count', '?')}"
    )
```

## Key Principles

- **Rich heartbeats**: Include operational metrics (active items, pending work, error counts), not just a timestamp. This lets you diagnose problems without SSH-ing in.
- **Three-tier status**: `running` (<60s), `stale` (<300s), `dead` (>300s). Adjust thresholds based on your service's expected heartbeat interval.
- **File-based, not network-based**: Heartbeat files are simpler and more reliable than HTTP health endpoints for background services. No port conflicts, no firewall issues.
- **Error isolation**: The heartbeat task is independent. If the main work loop crashes, the heartbeat stops updating, and the watchdog detects it.
- **Crash triggers shutdown**: Use `add_done_callback` on tasks so a crash in any task triggers clean shutdown of all tasks. This way systemd/supervisord can restart the whole service.

## Dependencies

Minimal — only stdlib for the core pattern. Optional: `ntfy.sh` (free push notifications, no signup).
