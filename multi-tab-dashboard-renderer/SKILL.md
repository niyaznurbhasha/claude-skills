---
name: multi-tab-dashboard-renderer
description: Generate multi-tab HTML dashboards from heterogeneous data sources with consistent styling and tab navigation. Use when building static HTML dashboards, when user says "HTML dashboard", "static dashboard", "generate report", or needs a self-contained HTML file with tabs.
metadata:
  author: niyaznurbhasha
  version: 1.0.0
  category: dashboard
  tags: [html, dashboard, static, tabs, rendering, python]
---

# Multi-Tab Dashboard Renderer

Generate self-contained, multi-tab HTML dashboards from Python. Each tab is an independent renderer module that returns HTML, CSS, JS, and data. A shell assembles them into a single file with tab navigation and shared styling.

## When to Use

- Building a static HTML dashboard (opens in any browser, no server needed)
- Multiple data sources need their own tab/view
- Want a single `.html` file you can share, email, or open from disk
- Dashboard grows over time — need to add tabs without touching existing ones

## Architecture

```
project/
  render.py              # Thin wrapper (backward compat)
  renderers/
    __init__.py          # Assembler: collects tabs, writes HTML
    shell.py             # HTML shell: header, tab bar, shared CSS, tab-switching JS
    tab_items.py         # Tab 1: items/trends view
    tab_journal.py       # Tab 2: journal/log view
    tab_health.py        # Tab 3: health/metrics view
    tab_settings.py      # Tab 4: config editor
  output/
    dashboard.html       # Generated output
```

Each tab renderer is a module with a `render()` function that returns:
```python
{
    "html": "<div>...</div>",      # Tab panel content
    "css": "/* tab-specific */",    # Scoped CSS
    "js": "(function(){...})()",    # Tab-specific JS (IIFE)
    "data_js": "const DATA = [...]" # Data embedded as JS variables
}
```

## Step-by-Step Implementation

### Step 1: Create the shell module

The shell provides the HTML skeleton, shared CSS, tab bar, and tab-switching JavaScript.

```python
# renderers/shell.py
import html as html_mod
from typing import List, Tuple


def assemble(tabs: List[Tuple[str, str, dict]], now_str: str) -> str:
    """Build full HTML document from tab renderers.

    Each tab is (id, label, {html, css, js, data_js}).
    """
    all_css = [_shared_css()]
    all_data_js = []
    all_js = []
    tab_buttons = []
    tab_panels = []

    for tab_id, label, content in tabs:
        if content.get("css"):
            all_css.append(content["css"])
        if content.get("data_js"):
            all_data_js.append(content["data_js"])
        if content.get("js"):
            all_js.append(content["js"])

        tab_buttons.append(
            f'<button class="tab-btn" data-tab="{tab_id}">'
            f'{html_mod.escape(label)}</button>'
        )
        tab_panels.append(
            f'<div class="tab-panel" id="tab-{tab_id}" style="display:none">'
            f'{content.get("html", "")}'
            f'</div>'
        )

    css_block = "\n".join(all_css)
    data_js_block = "\n".join(all_data_js)
    js_block = "\n".join(all_js)
    tabs_html = "\n    ".join(tab_buttons)
    panels_html = "\n".join(tab_panels)

    return f"""<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8"/>
<meta name="viewport" content="width=device-width, initial-scale=1"/>
<title>Dashboard</title>
<style>
{css_block}
</style>
</head>
<body>

<div class="shell-header">
  <h1>Dashboard</h1>
  <div class="shell-meta">
    <span>Last refresh: {html_mod.escape(now_str)}</span>
  </div>
</div>

<div class="tab-bar">
  <div class="tab-bar-inner">
    {tabs_html}
  </div>
</div>

{panels_html}

<script>
{data_js_block}

// --- Tab switching ---
(function() {{
  const btns = document.querySelectorAll('.tab-btn');
  const panels = document.querySelectorAll('.tab-panel');

  function switchTab(tabId) {{
    btns.forEach(b => b.classList.toggle('active', b.dataset.tab === tabId));
    panels.forEach(p => p.style.display = p.id === 'tab-' + tabId ? '' : 'none');
    location.hash = tabId;
  }}

  btns.forEach(b => b.addEventListener('click', () => switchTab(b.dataset.tab)));

  // Restore tab from URL hash or default to first
  const hash = location.hash.slice(1);
  const validTab = Array.from(btns).some(b => b.dataset.tab === hash);
  switchTab(validTab ? hash : btns[0]?.dataset.tab || '');
}})();

{js_block}
</script>
</body>
</html>"""


def _shared_css() -> str:
    """CSS shared across all tabs."""
    return """/* Shared Shell CSS */
* { box-sizing: border-box; margin: 0; padding: 0; }
body { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
       background: #0f172a; color: #e2e8f0; }

.shell-header { background: #1e293b; padding: 16px 24px; display: flex;
                align-items: center; justify-content: space-between;
                border-bottom: 1px solid #334155; }
.shell-header h1 { font-size: 20px; font-weight: 600; }
.shell-meta { font-size: 13px; color: #94a3b8; }

/* Tab bar */
.tab-bar { background: #1e293b; border-bottom: 2px solid #334155; padding: 0 24px; }
.tab-bar-inner { display: flex; gap: 0; }
.tab-btn { padding: 12px 24px; font-size: 14px; font-weight: 500; border: none;
           background: none; color: #94a3b8; cursor: pointer;
           border-bottom: 2px solid transparent; margin-bottom: -2px;
           transition: all 0.15s; }
.tab-btn:hover { color: #e2e8f0; background: rgba(255,255,255,0.03); }
.tab-btn.active { color: #60a5fa; border-bottom-color: #60a5fa; }

/* Shared utilities */
.btn { padding: 6px 14px; font-size: 13px; border: 1px solid #475569;
       background: #1e293b; color: #cbd5e1; border-radius: 4px; cursor: pointer; }
.btn:hover { background: #334155; }
.btn.active { background: #3b82f6; border-color: #3b82f6; color: #fff; }
.badge { display: inline-block; padding: 2px 8px; border-radius: 10px;
         font-size: 11px; font-weight: 600; color: #fff; }
.search-box { padding: 6px 12px; font-size: 13px; border: 1px solid #475569;
              background: #0f172a; color: #e2e8f0; border-radius: 4px; width: 260px; }
.card { background: #1e293b; border: 1px solid #334155; border-radius: 8px;
        padding: 16px; transition: border-color 0.15s; }
.card:hover { border-color: #475569; }
.empty-state { text-align: center; padding: 60px 20px; color: #64748b; font-size: 15px; }"""
```

### Step 2: Create a tab renderer module

Each tab is a Python module with a `render()` function. It returns HTML, CSS, JS, and data.

```python
# renderers/tab_items.py
import json
from typing import Dict, List


def render(items: List[dict]) -> dict:
    """Return {html, css, js, data_js} for the Items tab."""

    # Embed data as a JS variable
    data_js = f"const ITEMS_DATA = {json.dumps(items, default=str)};"

    tab_html = """
<div class="items-controls">
  <input class="search-box" id="items-search" placeholder="Search items..." />
  <div class="count" id="items-count"></div>
</div>
<table class="items-table">
  <thead>
    <tr>
      <th data-col="name">Name</th>
      <th data-col="value">Value</th>
      <th data-col="status">Status</th>
      <th data-col="date">Date</th>
    </tr>
  </thead>
  <tbody id="items-tbody"></tbody>
</table>
"""

    tab_css = """/* Items Tab CSS */
.items-controls { padding: 16px 24px; display: flex; gap: 12px; align-items: center;
                  border-bottom: 1px solid #334155; }
.items-table { width: 100%; border-collapse: collapse; }
.items-table thead th { position: sticky; top: 0; background: #1e293b;
                        padding: 10px 12px; text-align: left; font-size: 13px;
                        font-weight: 600; color: #94a3b8; cursor: pointer; }
.items-table tbody tr { border-bottom: 1px solid #1e293b; }
.items-table tbody tr:hover { background: #1e293b; }
.items-table td { padding: 8px 12px; font-size: 13px; }"""

    tab_js = """
// === Items Tab JS ===
(function() {
const DATA = ITEMS_DATA;
let searchQuery = "";

function renderTable() {
  let filtered = DATA;
  if (searchQuery) {
    const q = searchQuery.toLowerCase();
    filtered = filtered.filter(d =>
      (d.name || "").toLowerCase().includes(q) ||
      (d.description || "").toLowerCase().includes(q)
    );
  }

  document.getElementById("items-count").textContent = filtered.length + " items";
  const tbody = document.getElementById("items-tbody");
  let h = "";
  for (const d of filtered) {
    h += "<tr>"
      + "<td>" + (d.name || "").replace(/</g, "&lt;") + "</td>"
      + "<td>" + (d.value || "") + "</td>"
      + "<td>" + (d.status || "") + "</td>"
      + "<td>" + (d.date || "") + "</td>"
      + "</tr>";
  }
  tbody.innerHTML = h;
}

document.getElementById("items-search").addEventListener("input", e => {
  searchQuery = e.target.value;
  renderTable();
});

renderTable();
})();
"""

    return {"html": tab_html, "css": tab_css, "js": tab_js, "data_js": data_js}
```

### Step 3: Create the assembler

The `__init__.py` collects all tabs and writes the final HTML.

```python
# renderers/__init__.py
from datetime import datetime, timezone
from pathlib import Path
from typing import Dict, List, Optional

from . import shell, tab_items, tab_journal, tab_health

OUTPUT_DIR = Path(__file__).resolve().parent.parent / "output"


def render_dashboard(
    items: List[dict],
    journal_entries: Optional[List[dict]] = None,
    health_data: Optional[dict] = None,
    output_path: Optional[Path] = None,
) -> Path:
    """Assemble all tabs and write the dashboard HTML."""
    if output_path is None:
        output_path = OUTPUT_DIR / "dashboard.html"
    output_path.parent.mkdir(parents=True, exist_ok=True)

    now_str = datetime.now(timezone.utc).strftime("%Y-%m-%d %H:%M UTC")

    # Collect tabs — each is (id, label, content_dict)
    tabs = []

    tabs.append(("items", "Items", tab_items.render(items)))
    tabs.append(("journal", "Journal", tab_journal.render(journal_entries or [])))
    tabs.append(("health", "Health", tab_health.render(health_data or {})))

    # Assemble into shell
    html_doc = shell.assemble(tabs, now_str)
    output_path.write_text(html_doc, encoding="utf-8")
    return output_path
```

### Step 4: Create a thin top-level wrapper

```python
# render.py — backward-compatible entry point
from pathlib import Path
from typing import Dict, List, Optional
from renderers import render_dashboard

OUTPUT_DIR = Path(__file__).parent / "output"

def render(items: List[dict], **kwargs) -> Path:
    output_path = kwargs.pop("output_path", OUTPUT_DIR / "dashboard.html")
    return render_dashboard(items=items, output_path=output_path, **kwargs)
```

### Step 5: Adding a new tab

To add a new tab, create a new module and add one line to the assembler:

```python
# renderers/tab_settings.py
def render(config: dict) -> dict:
    data_js = f"const SETTINGS_DATA = {json.dumps(config)};"
    tab_html = """<div class="settings-form">...</div>"""
    tab_css = """/* Settings CSS */"""
    tab_js = """(function() { /* Settings JS */ })();"""
    return {"html": tab_html, "css": tab_css, "js": tab_js, "data_js": data_js}
```

Then in `__init__.py`:
```python
from . import tab_settings
# ...
tabs.append(("settings", "Settings", tab_settings.render(config or {})))
```

## Key Principles

- **Each tab is an isolated module**: A tab renderer knows nothing about other tabs. It returns `{html, css, js, data_js}` and the shell assembles them.
- **Data embedded as JS variables**: Each tab's `data_js` creates `const TAB_DATA = [...]` in a `<script>` block. This makes the HTML fully self-contained — no server needed.
- **IIFE-wrapped JS**: Each tab's JS is wrapped in `(function(){...})()` to avoid variable name collisions between tabs.
- **URL hash for tab state**: The active tab is stored in `location.hash`. Refreshing or sharing the URL preserves the active tab.
- **Self-contained output**: The generated HTML file includes all CSS, JS, and data inline. It opens in any browser from disk (`file://`), no server required.
- **Adding tabs is one-file, one-line**: Create `tab_new.py`, add one line to the assembler. Zero changes to existing tabs or the shell.

## No External Dependencies

This pattern uses only Python stdlib (`json`, `html`, `pathlib`, `datetime`). The generated HTML uses vanilla JavaScript — no frameworks.
