---
name: fastapi-sse-live-dashboard
description: FastAPI + Jinja2 + Server-Sent Events for real-time web dashboards with demo mode and HTMX compatibility. Use when building any live-updating web dashboard, when user says "live dashboard", "SSE", "real-time web UI", or needs a self-updating browser dashboard.
metadata:
  author: niyaznurbhasha
  version: 1.0.0
  category: dashboard
  tags: [fastapi, sse, dashboard, jinja2, htmx, real-time, python]
---

# FastAPI SSE Live Dashboard

Build real-time web dashboards using FastAPI, Jinja2 templates, and Server-Sent Events (SSE). Includes demo mode for development and HTMX-compatible partial rendering.

## When to Use

- Building a browser-based monitoring dashboard
- Need live-updating data without WebSocket complexity
- Want a demo mode that works without backend infrastructure
- Need both full-page renders and partial (SSE) updates

## Architecture

```
project/
  dashboard.py          # FastAPI app + CLI
  templates/
    index.html          # Full page (loads partials)
    market.html         # Detail page
    partials/
      table_rows.html   # Partial: just the <tr> elements
      stats.html        # Partial: just the stats block
```

SSE flow:
```
Browser                    Server
  |--- GET /stream/data ---->|
  |<--- event: update -------|  (HTML fragment)
  |    [swap into DOM]       |
  |<--- event: update -------|  (every N seconds)
  |    [swap into DOM]       |
  ...
```

## Step-by-Step Implementation

### Step 1: Create the DataService with demo/live modes

Separate data access from rendering. Demo mode generates synthetic data so you can develop the UI without a live backend.

```python
from pathlib import Path
import time
import random

class DataService:
    def __init__(self, data_dir: Path, demo: bool = False) -> None:
        self.data_dir = data_dir
        self.demo = demo
        self._start_time = time.time()
        if not demo:
            self._init_live()

    def _init_live(self) -> None:
        """Connect to real data sources (databases, files, APIs)."""
        # e.g., open SQLite, connect to DuckDB, etc.
        pass

    def get_items(self) -> list[dict]:
        if self.demo:
            return self._demo_items()
        return self._live_items()

    def get_health(self) -> dict:
        if self.demo:
            return {"status": "demo", "uptime": time.time() - self._start_time}
        return self._live_health()

    def _demo_items(self) -> list[dict]:
        """Generate synthetic data with slight randomization."""
        return [
            {"name": "Item A", "value": round(random.gauss(100, 5), 2), "updated": "3s ago"},
            {"name": "Item B", "value": round(random.gauss(200, 10), 2), "updated": "7s ago"},
        ]

    def _live_items(self) -> list[dict]:
        # Query real database
        pass

    def _live_health(self) -> dict:
        # Check heartbeat file, count records, etc.
        pass
```

### Step 2: Set up FastAPI with Jinja2 templates

```python
import json
from fastapi import FastAPI, Request
from fastapi.responses import HTMLResponse
from fastapi.templating import Jinja2Templates
from pathlib import Path

app = FastAPI(title="My Dashboard")
templates = Jinja2Templates(directory=str(Path(__file__).parent / "templates"))

_svc: DataService | None = None

def _get_svc() -> DataService:
    assert _svc is not None, "DataService not initialized"
    return _svc
```

### Step 3: Create full-page routes

These render the complete page with initial data. SSE will update sections afterward.

```python
@app.get("/", response_class=HTMLResponse)
async def index(request: Request) -> HTMLResponse:
    items = _get_svc().get_items()
    return templates.TemplateResponse("index.html", {
        "request": request,
        "items": items,
        "demo": _svc.demo,
    })

@app.get("/detail/{item_id}", response_class=HTMLResponse)
async def detail(request: Request, item_id: str) -> HTMLResponse:
    svc = _get_svc()
    item = svc.get_item(item_id)
    return templates.TemplateResponse("detail.html", {
        "request": request,
        "item": item,
        "demo": svc.demo,
    })
```

### Step 4: Implement SSE stream helper

The core pattern: an async generator that yields rendered HTML partials at a fixed interval.

```python
import asyncio
from collections.abc import AsyncGenerator
from sse_starlette.sse import EventSourceResponse

async def _sse_stream(
    request: Request,
    render_fn,
    interval: float,
) -> AsyncGenerator[dict]:
    """Generic SSE stream: render HTML partial at interval."""
    while True:
        if await request.is_disconnected():
            break
        try:
            html = render_fn()
            yield {"event": "update", "data": html}
        except Exception as exc:
            log.error("sse_render_error", error=str(exc))
        await asyncio.sleep(interval)
```

### Step 5: Create SSE endpoints that return HTML partials

Each SSE endpoint renders a Jinja2 partial template and streams it to the browser.

```python
@app.get("/stream/items")
async def stream_items(request: Request) -> EventSourceResponse:
    def render() -> str:
        items = _get_svc().get_items()
        return templates.get_template("partials/item_rows.html").render(items=items)
    return EventSourceResponse(_sse_stream(request, render, interval=2.0))

@app.get("/stream/health")
async def stream_health(request: Request) -> EventSourceResponse:
    def render() -> str:
        health = _get_svc().get_health()
        return templates.get_template("partials/health_stats.html").render(health=health)
    return EventSourceResponse(_sse_stream(request, render, interval=5.0))

@app.get("/stream/detail/{item_id}")
async def stream_detail(request: Request, item_id: str) -> EventSourceResponse:
    def render() -> str:
        data = _get_svc().get_detail(item_id)
        return templates.get_template("partials/detail_body.html").render(data=data)
    return EventSourceResponse(_sse_stream(request, render, interval=1.0))
```

### Step 6: Client-side SSE consumption

In your HTML template, use `EventSource` to subscribe and swap content:

```html
<!-- templates/index.html -->
<table>
  <thead><tr><th>Name</th><th>Value</th><th>Updated</th></tr></thead>
  <tbody id="items-body">
    {% for item in items %}
    <tr><td>{{ item.name }}</td><td>{{ item.value }}</td><td>{{ item.updated }}</td></tr>
    {% endfor %}
  </tbody>
</table>

<div id="health-stats">Loading...</div>

<script>
// Subscribe to SSE and swap innerHTML
function connectSSE(url, targetId) {
  const source = new EventSource(url);
  source.addEventListener("update", (e) => {
    document.getElementById(targetId).innerHTML = e.data;
  });
  source.onerror = () => {
    // Auto-reconnect is built into EventSource
    console.warn("SSE connection lost, reconnecting...");
  };
}

connectSSE("/stream/items", "items-body");
connectSSE("/stream/health", "health-stats");
</script>
```

Or with HTMX (even simpler):

```html
<tbody id="items-body"
       hx-ext="sse"
       sse-connect="/stream/items"
       sse-swap="update"
       hx-swap="innerHTML">
  <!-- initial content -->
</tbody>
```

### Step 7: CLI with demo flag

```python
import typer
import uvicorn

cli = typer.Typer(add_completion=False)

@cli.command()
def main(
    data_dir: Path = typer.Option(Path("data"), help="Data directory"),
    demo: bool = typer.Option(False, "--demo", help="Use synthetic demo data"),
    port: int = typer.Option(8000, help="HTTP port"),
    host: str = typer.Option("0.0.0.0", help="Bind host"),
) -> None:
    global _svc
    _svc = DataService(data_dir=data_dir, demo=demo)
    mode = "DEMO" if demo else "LIVE"
    print(f"Dashboard starting [{mode}] on http://{host}:{port}")
    uvicorn.run(app, host=host, port=port, log_level="info")

if __name__ == "__main__":
    cli()
```

## Key Principles

- **SSE over WebSocket for dashboards**: SSE is simpler (HTTP, auto-reconnect, no handshake protocol), works through proxies, and is sufficient for server-to-client updates.
- **Render HTML on the server**: The SSE endpoint returns rendered HTML fragments, not JSON. The client just swaps innerHTML. No client-side templating needed.
- **Demo mode from day one**: Build the UI with synthetic data first. Demo mode makes it possible to develop, test, and demo without backend infrastructure.
- **Partial templates**: Full-page templates include partials. SSE endpoints render the same partials. This avoids duplicating rendering logic.
- **Interval per endpoint**: Different data has different update frequencies. Item list every 2s, health every 5s, detail view every 1s.

## Dependencies

```
fastapi
uvicorn
jinja2
sse-starlette
typer
```
