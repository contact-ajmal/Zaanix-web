---
title: Data apps
order: 13
group: Work with data
description: Streamlit, Dash and Gradio apps on a workspace's data — a Python SDK, an editor with a live preview, runtimes from a process to Kubernetes or the viewer's browser, scaling to zero, and publishing with review.
---

**Apps** turns a workspace into a place where analysts build applications, not only dashboards: a **Streamlit**, **Dash** or **Gradio** app that reads the workspace's tables and files through the `zaanix` SDK, edited in ZAANIX next to a live preview, run by ZAANIX, and opened by the workspace's members.

## Write one

```python
import streamlit as st
from zaanix.streamlit import connect, query, table_picker, viewer

st.title("Explorer")
dv = connect()                                   # from the runner's environment
rel = table_picker(dv)                           # tables, views and data files of the workspace
df = query(f"SELECT * FROM {rel} LIMIT 1000")    # DuckDB SQL → pandas, cached for five minutes
st.caption(f"{len(df):,} rows · viewing as {viewer()['email']}")
st.dataframe(df)
```

*New app* in the gallery starts from a **template** (the table explorer: pick a dataset, filter, grid, chart — or blank), **from a dashboard**, or **from saved queries**. A Mosaic dashboard is turned into an app deterministically — its datasets become SQL relations, its menus and sliders sidebar filters, its KPI marks metric cards, its bar / area / line / scatter / heat-map marks aggregating SQL rendered with Altair, its tables `st.dataframe` — so the app answers the same questions the dashboard does and the code is yours to take further. The editor has Python highlighting, ⌘S saves and restarts the running app, the preview on the right is the real app, *Check* runs the static checks (compiles, imports streamlit, no tokens), *Draft* lets Copilot write `app.py` for a goal with the SDK guide as its contract, and the logs are one click away.

## The SDK

`pip install zaanix` — no dependencies; add `[streamlit]` for streamlit, pandas and pyarrow.

```python
import zaanix
dv = zaanix.connect(url, token, workspace)     # or ZAANIX_URL / ZAANIX_TOKEN / ZAANIX_WORKSPACE
dv.query("SELECT zone, avg(fare) FROM trips GROUP BY 1")   # pandas; format="records" | "polars" | "result"
dv.query_arrow("SELECT * FROM trips")             # pyarrow through the Arrow export — the fast path for large results
dv.tables(); dv.files(); dv.catalog()
dv.table("trips").where("fare > 10").order_by("fare DESC").limit(100).to_df()
dv.copilot("Which zones have the highest average fare?")["text"]
dv.tools(); dv.call_tool("profile_dataset", file_path_or_table="trips")   # the agent façade, for LangChain / CrewAI / Strands
```

Everything goes through ZAANIX's HTTP API with a bearer token — the SDK never opens the `.duckdb` file, so the engine keeps its lock and every read carries the caller's role, the sandbox and the audit trail.

## Frameworks

| Framework | Entry | Starts with |
|---|---|---|
| Streamlit (default) | `app.py` | `streamlit run` — templates *Table explorer* and *Blank* |
| Dash | calls `app.run()` | `python app.py` — template *Dash explorer* |
| Gradio | calls `demo.launch()` | `python app.py` — template *Gradio query* |

The framework is fixed when the app is created. The visitor's identity reaches every framework in `X-ZAANIX-User / -Email / -Role` headers (`viewer()` in Streamlit, `zaanix.viewer_from_headers(...)` in Dash, `request.headers` in Gradio).

## Where apps run

Each app gets a token minted for its creator on every start — the `read` scope, **scoped to the app's workspace**, expiring, revoked when the app stops — and a minimal environment (`ZAANIX_URL`, `ZAANIX_TOKEN`, `ZAANIX_WORKSPACE`), never the server's secrets. An app can query what a viewer could and nothing else. `apps.runtime` picks where it runs:

| Runtime | Instance | Isolation |
|---|---|---|
| `subprocess` (default) | a process on the server, from a shared virtualenv created on the first start | a separate process with a minimal environment |
| `docker` | a container of `anbproject/zaanix-app-runtime` | read-only root, all capabilities dropped, non-root, memory and CPU limits |
| `kubernetes` | a pod in the server's namespace | non-root, read-only root, no service-account token, resource limits; a NetworkPolicy fences app pods |

**In the viewer's browser.** A Streamlit app can run with `execution: browser` on stlite (Streamlit on Pyodide): no process at all, and it reads **as the viewer**, with a read-only token for the app's workspace.

**Scaling.** Apps scale to zero: unused apps stop after `apps.idle_stop_minutes`, and the next page load starts them again. At `apps.max_running`, the least recently used idle app makes room. Administrators mark apps **always on**: started with the server, never idled out, restarted after a crash.

## Publishing and isolation

- Apps are for the workspace's members until **published** to everyone signed in. With `apps.publish_requires_approval` (the default) an editor's request waits for an administrator's review under **Settings → Data apps**; a code change to an approved app sends it back for review.
- Apps are **never served from the UI's origin**: a second listener (`apps.port`, the UI's port + 1 — `:4201` by default; `apps.public_url` behind a proxy) serves them, and the browser gets its app cookie there through a one-time handoff. App links can be shared as they are.
- `apps.enabled` is on in full filesystem mode and **off in sandboxed mode**: apps execute Python next to the server, so an administrator decides.

## For agents and MCP clients

Everything above is a tool. From Claude Code, Claude Desktop, Cursor or any MCP client connected to ZAANIX:

| Tool | Does |
|---|---|
| `list_apps(workspace_id?)` | Apps with status, URL, visibility, errors. |
| `create_app(name, source, description?, visibility?, run_now?)` | `source: {dashboard_id}` generates from a dashboard; `{saved_query_ids}` / `{queries: [{name, sql}]}` a query browser; `{template}`; `{code, requirements?}` code as written. Validated (compiles, imports streamlit, no tokens) before it is saved, started right away, returns the code and the URL. |
| `update_app(app_id, code?, requirements?, name?, description?, run_now?)` | Re-validated; a running app restarts and the call waits until it is healthy. |
| `run_app` · `stop_app` · `get_app_logs` | Lifecycle and diagnostics (tracebacks, install output). |
| `preview_app(app_id, wait_ms?)` | A headless browser opens the app as a signed-in visitor, waits for Streamlit to render, and returns the visible text **and a screenshot** (an MCP image; needs Chrome/Chromium on the server) — the way an agent checks its own work. |
| `publish_app(app_id, audience, dry_run)` | Human-in-the-loop: `dry_run` (default) says what would change; `dry_run=false` after approval makes the app visible to everyone signed in (`org`) or back to the workspace. Audited. |

The resource `duckdb://guides/data-app` is the SDK contract an agent reads before writing code, and the prompt **`build_data_app(goal, data?)`** is the end-to-end recipe: connect the data and keep it fresh with a sync → profile it → build a Mosaic dashboard → `create_app` from it → `preview_app` → refine with `update_app` → `publish_app` once a person approves. The REST façade (`/api/agent/v1/tools/<name>`) exposes the same tools to LangChain, CrewAI and Strands agents, with screenshots as `images[].data_base64`.
