---
name: textual-tui-reactive
description: Textual TUI with DataTables, reactive properties, @work async tasks, multi-tab layout, and keybindings. Use when building terminal UI dashboards, when user says "TUI", "terminal dashboard", "textual app", or needs a rich terminal interface.
metadata:
  author: niyaznurbhasha
  version: 1.0.0
  category: dashboard
  tags: [textual, tui, terminal, dashboard, reactive, python]
---

# Textual TUI Reactive Dashboard

Build rich terminal UI dashboards using Textual with DataTables, reactive properties, background async workers, multi-tab layouts, and keybindings.

## When to Use

- Building a terminal-based monitoring dashboard
- Need a TUI that auto-refreshes with live data
- Want keyboard navigation, tabs, and interactive tables
- Need a local dashboard that works over SSH

## Architecture

```
project/
  monitor.py       # Textual App + DataService + CSS
```

The Textual app has:
- `DataService` — fetches data (demo or live)
- `MonitorApp(App)` — layout, keybindings, refresh intervals
- `@work` decorated methods — background data fetching
- `call_from_thread` — safely update UI from worker threads
- `reactive` properties — auto-trigger UI updates on change

## Step-by-Step Implementation

### Step 1: Create a DataService

Same pattern as the web dashboard — demo/live modes with a clean public API.

```python
from pathlib import Path

class DataService:
    def __init__(self, data_dir: Path, demo: bool = False) -> None:
        self._data_dir = data_dir
        self._demo = demo

    def get_items(self) -> list[dict]:
        if self._demo:
            return self._demo_items()
        return self._live_items()

    def get_detail(self, item_id: str) -> dict | None:
        if self._demo:
            return self._demo_detail(item_id)
        return self._live_detail(item_id)

    def get_health(self) -> dict:
        if self._demo:
            return self._demo_health()
        return self._live_health()

    def get_item_ids(self) -> list[str]:
        return [item["id"] for item in self.get_items()]

    # ... demo and live implementations
```

### Step 2: Define CSS for the app

Textual uses CSS (a subset of web CSS) for layout and styling.

```python
CSS = """
Screen {
    background: $surface;
}

#main-table {
    height: 1fr;
}

#detail-container {
    height: 1fr;
    padding: 0 1;
}

#selector {
    height: 3;
    margin-bottom: 1;
}

#detail-body {
    height: 1fr;
}

#health-content {
    height: auto;
    padding: 1;
    border: solid $panel;
}

/* Color classes for status indicators */
.status-ok { color: $success; text-style: bold; }
.status-warn { color: $warning; text-style: bold; }
.status-error { color: $error; text-style: bold; }
"""
```

### Step 3: Build the App with TabbedContent

```python
from textual.app import App, ComposeResult
from textual.binding import Binding
from textual.containers import Horizontal, Vertical
from textual.reactive import reactive
from textual.widgets import (
    DataTable, Footer, Header, Select, Static, TabbedContent, TabPane,
)

class MonitorApp(App):
    TITLE = "My Monitor"
    CSS = CSS

    BINDINGS = [
        Binding("q", "quit", "Quit"),
        Binding("r", "refresh_all", "Refresh"),
        Binding("1", "switch_tab('tab-items')", "Items"),
        Binding("2", "switch_tab('tab-detail')", "Detail"),
        Binding("3", "switch_tab('tab-health')", "Health"),
    ]

    # Reactive properties — changing these auto-triggers watchers
    _selected_id: reactive[str] = reactive("")

    def __init__(self, data_service: DataService) -> None:
        super().__init__()
        self._ds = data_service

    def compose(self) -> ComposeResult:
        yield Header(show_clock=False)
        with TabbedContent(id="tabs"):
            with TabPane("Items", id="tab-items"):
                yield DataTable(id="items-table", cursor_type="row")

            with TabPane("Detail", id="tab-detail"):
                with Vertical(id="detail-container"):
                    yield Select(
                        options=[("Loading...", "__loading__")],
                        id="selector",
                        prompt="Select item",
                        allow_blank=False,
                    )
                    with Horizontal(id="detail-body"):
                        yield Static(id="detail-content")
                        yield DataTable(id="detail-table", cursor_type="none")

            with TabPane("Health", id="tab-health"):
                yield Static(id="health-content")

        yield Footer()
```

### Step 4: Set up tables and intervals on mount

```python
def on_mount(self) -> None:
    # Configure table columns
    table = self.query_one("#items-table", DataTable)
    table.add_column("ID", key="id", width=16)
    table.add_column("Name", key="name", width=30)
    table.add_column("Value", key="value", width=10)
    table.add_column("Status", key="status", width=10)
    table.add_column("Updated", key="updated", width=10)

    detail_table = self.query_one("#detail-table", DataTable)
    detail_table.add_column("Time", key="ts", width=10)
    detail_table.add_column("Event", key="event", width=20)
    detail_table.add_column("Value", key="value", width=10)

    # Initial loads
    self.refresh_items()
    self.refresh_health()

    # Periodic refresh intervals
    self.set_interval(1.0, self._tick_clock)
    self.set_interval(2.0, self.refresh_items)
    self.set_interval(1.0, self.refresh_detail)
    self.set_interval(5.0, self.refresh_health)

def _tick_clock(self) -> None:
    from datetime import datetime, timezone
    now = datetime.now(timezone.utc).strftime("%H:%M:%S UTC")
    mode = "DEMO" if self._ds._demo else "LIVE"
    self.sub_title = f"{now}  [{mode}]"
```

### Step 5: Use @work for background data fetching

The `@work` decorator runs the method in a thread. Use `call_from_thread` to safely update the UI.

```python
from textual import work

@work(exclusive=True, thread=True)
def refresh_items(self) -> None:
    """Fetch items in background thread, update table on main thread."""
    items = self._ds.get_items()
    ids = [item["id"] for item in items]
    self.call_from_thread(self._update_items_table, items)
    self.call_from_thread(self._update_selector, ids)

def _update_items_table(self, items: list[dict]) -> None:
    table = self.query_one("#items-table", DataTable)
    table.clear()
    for item in items:
        table.add_row(
            item["id"],
            item["name"][:28],
            f"{item['value']:.2f}",
            item["status"],
            f"{item.get('updated_ago', '?')}s",
            key=item["id"],
        )

def _update_selector(self, ids: list[str]) -> None:
    selector = self.query_one("#selector", Select)
    current = self._selected_id
    options = [(item_id, item_id) for item_id in ids]
    if not options:
        return
    selector.set_options(options)
    if current and current in ids:
        selector.value = current
    else:
        selector.value = ids[0]
        self._selected_id = ids[0]

@work(exclusive=True, thread=True)
def refresh_detail(self) -> None:
    item_id = self._selected_id
    if not item_id:
        return
    detail = self._ds.get_detail(item_id)
    self.call_from_thread(self._update_detail, detail)

@work(exclusive=True, thread=True)
def refresh_health(self) -> None:
    health = self._ds.get_health()
    self.call_from_thread(self._update_health, health)
```

### Step 6: Handle user interactions

```python
def action_switch_tab(self, tab_id: str) -> None:
    tabs = self.query_one("#tabs", TabbedContent)
    tabs.active = tab_id

def action_refresh_all(self) -> None:
    self.refresh_items()
    self.refresh_detail()
    self.refresh_health()

def on_data_table_row_selected(self, event: DataTable.RowSelected) -> None:
    """Click a row in the items table -> switch to detail tab."""
    if event.data_table.id == "items-table":
        item_id = str(event.row_key.value)
        self._selected_id = item_id
        tabs = self.query_one("#tabs", TabbedContent)
        tabs.active = "tab-detail"
        self.refresh_detail()

def on_select_changed(self, event: Select.Changed) -> None:
    """Dropdown selection changed -> refresh detail."""
    if event.select.id == "selector":
        val = event.value
        if val and val != Select.BLANK:
            self._selected_id = str(val)
            self.refresh_detail()
```

### Step 7: Update health display with rich markup

```python
def _update_health(self, h: dict) -> None:
    status = h.get("status", "unknown")

    if status == "demo":
        status_line = "[yellow]DEMO[/yellow] (synthetic data)"
    elif status == "running":
        status_line = "[green]RUNNING[/green]"
    elif status == "stale":
        status_line = "[yellow]STALE[/yellow]"
    else:
        status_line = "[red]DOWN[/red]"

    content = (
        f"Status:   {status_line}\n"
        f"Items:    {h.get('item_count', '?')}\n"
        f"Events:   {h.get('event_count', '?')}\n"
        f"Uptime:   {h.get('uptime_sec', '?')}s"
    )
    widget = self.query_one("#health-content", Static)
    widget.update(content)
```

### Step 8: CLI entrypoint

```python
import typer

cli = typer.Typer(add_completion=False)

@cli.command()
def main(
    data_dir: Path = typer.Option(Path("data"), help="Data directory"),
    demo: bool = typer.Option(False, "--demo", help="Use synthetic demo data"),
) -> None:
    ds = DataService(data_dir, demo=demo)
    app = MonitorApp(ds)
    app.run()

if __name__ == "__main__":
    cli()
```

## Key Principles

- **`@work(exclusive=True, thread=True)`**: Runs data fetching in a background thread so the UI stays responsive. `exclusive=True` cancels any previous running instance of the same worker.
- **`call_from_thread`**: Always use this to update widgets from a worker thread. Direct widget access from threads will crash.
- **`reactive` properties**: Declare with `reactive[str] = reactive("")`. Changing the value auto-triggers `watch_` methods if defined.
- **`set_interval`**: Set up periodic refreshes on mount. Different data gets different intervals (items: 2s, detail: 1s, health: 5s).
- **Keybindings**: Number keys for tab switching (`1`, `2`, `3`), `r` for refresh, `q` for quit. Users expect these.
- **Demo mode**: Always have a demo mode. It makes development, testing, and demos trivial.

## Dependencies

```
textual>=0.40
typer
```
