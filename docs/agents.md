---
title: MCP, A2A and ZAANIX AI
order: 41
group: AI and agents
description: The MCP server (stdio, SSE, Streamable HTTP), the REST/OpenAPI façade, registered agents with per-framework snippets, human-in-the-loop approval, and ZAANIX Bot.
---

ZAANIX treats agents as first-class users. **One tool registry** backs three surfaces, so they can never drift:

- **MCP server** — stdio, legacy SSE and Streamable HTTP; 90 tools, with resources and guided prompts.
- **REST façade** — `GET /api/agent/v1/tools` (names, descriptions, JSON-schema inputs) and `POST /api/agent/v1/tools/<tool>` (returns `{text, structured, is_error}`; invalid arguments → 400).
- **OpenAPI 3.0** — `GET /api/agent/openapi.json`, generated on the fly for Bedrock Agents action groups and AgentCore Gateway targets.

## Transports

| Mode | How |
|---|---|
| stdio | `zaanix mcp --token zx_… [--workspace <id>]` (or `ZAANIX_API_TOKEN`) — stdout is JSON-RPC only, logs go to stderr |
| SSE (2024-11-05) | `GET /mcp/sse` + `POST /mcp/messages?sessionId=…` with `Authorization: Bearer zx_…` |
| Streamable HTTP (2025-03-26) | `POST/GET/DELETE /mcp` with `Authorization: Bearer zx_…` and `mcp-session-id` |

Tokens need the `mcp` scope (plus `write` for mutations and `admin` for administrative SQL). A token may be pinned to one workspace; otherwise pass `workspace_id` to tools or `?workspace_id=` on connect.

```bash
claude mcp add --transport http zaanix http://localhost:4200/mcp --header "Authorization: Bearer zx_…"
```

## Tools

| Tool | Purpose |
|---|---|
| `execute_query(sql, workspace_id?, page_size?, page?, dry_run?)` | Runs SQL; returns a Markdown table + typed JSON (`columns`, `rows`, `total_rows`, `truncated`, `rows_changed`). Hard cap 200 rows per call, long strings truncated. Mutations need `dry_run=false`. Read-only results are served from the [result cache](cache.html). |
| `profile_dataset(table_or_path, workspace_id?)` | `SUMMARIZE` stats: types, min/max, approx distinct, null %, quartiles, row count, footprint. |
| `explain_query(sql, workspace_id?, analyze?)` | Physical plan as JSON tree + ASCII with cardinality estimates; `analyze=true` adds measured timings (read-only SQL only). |
| `list_accessible_data(workspace_id?)` | Workspaces (own and shared), tables/views with columns, every data file / Delta / Iceberg table in the jail, and attached lakehouse catalogs. |
| `save_dataset(sql, output_format, target_filename, workspace_id?, dry_run?)` | `COPY (sql) TO` parquet/csv/json inside the jail (`exports/` by default). |
| `browse_storage(provider?, path?, connection_id?, bucket?, catalog?, schema?)` | One level of the data directory, cloud connections → buckets → objects, or lakehouse connections → schemas → tables. |
| `inspect_schema(file_path_or_table, connection_id?)` | Columns/types/nullability for tables, files, remote objects, `.duckdb` files, attached lakehouse tables or a SELECT — no scan. |
| `lakehouse_query(connection_id, sql, page_size?, dry_run?)` | SQL on a Databricks SQL warehouse; non-read statements need `dry_run=false` after approval. |
| `list_dashboards(workspace_id?)` | Dashboards with their widgets and layouts. |
| `create_dashboard_widget(dashboard_id \| dashboard_name, title, sql, widget_type, chart_config?, refresh_interval_sec?)` | Builds dashboards autonomously; the SQL is validated read-only and dry-run first. |
| `create_mosaic_dashboard(spec \| spec_text, name?, description?, dashboard_id?, validate_only?, workspace_id?)` | Creates or updates an interactive Mosaic dashboard from a declarative spec (YAML/JSON). Validated structurally and every dataset/table bound with EXPLAIN before saving; errors come back as a list to fix. |
| `list_data_sources(workspace_id?)` | Every connection with health (connector connections included), the syncs of a workspace, the source-type catalog. |
| `browse_connector(connection_id, path?)` | Walks a warehouse, SaaS or Google connection one level at a time; leaves carry the `resource` for `create_data_sync`. |
| `connector_query(connection_id, sql, limit?)` | Read-only SQL on Snowflake, BigQuery, Redshift or ClickHouse; rows capped. |
| `create_data_sync(name, source, target_table, …, transform_sql?, schedule?, run_now?)` | Scheduled load of a table / connector resource / URL / SELECT into a workspace table with an optional `{{raw}}` transformation, validated first. |
| `update_data_sync(sync_id, transform_sql?, schedule?, mode?, enabled?, run_now?)` | Attach a transformation, change the schedule, pause/resume. |
| `run_data_sync(sync_id)` | Run now; rows, duration, error, recent runs. |
| `list_apps` · `create_app(name, source, …)` · `update_app` · `run_app` · `stop_app` · `get_app_logs` · `preview_app` · `publish_app` | Streamlit [data apps](apps.html): generated from a dashboard, saved queries or code (validated first), run, previewed with a screenshot, published after human approval. |

The table above is the core. The other tools, by area (the **Tools** tab of the Agents page and `GET /api/agent/v1/tools` list all 90 with their inputs):

| Area | Tools |
|---|---|
| Dashboards | `build_dashboard` (a whole dashboard from a checked plan), `get_dashboard`, `update_widget`, `remove_widget`, `snapshot_dashboard` |
| Notebooks and queries | `list_notebooks`, `get_notebook`, `create_notebook`, `run_notebook`, `list_saved_queries`, `get_saved_query`, `save_query`, `query_history` |
| Explore and prepare | `search_workspace`, `search_catalog`, `diff_tables`, `find_joins`, `prepare_data` |
| dbt and metrics | `list_dbt_projects`, `get_dbt_project`, `create_dbt_project`, `write_dbt_files`, `create_dbt_model`, `run_dbt`, `get_dbt_run`, `list_metrics`, `query_metrics`, `define_metric` |
| Quality and monitoring | `list_quality_suites`, `suggest_quality_checks`, `create_quality_suite`, `run_quality_suite`, `list_watches`, `create_watch`, `check_watch`, `detect_anomalies`, `list_insights`, `create_metric_monitor` |
| Governance | `get_lineage`, `annotate_table`, `scan_pii`, `tag_pii`, `protect_pii` |
| Delivery and publishing | `list_alerts`, `create_alert`, `run_alert`, `list_endpoints`, `publish_endpoint`, `list_reverse_syncs`, `create_reverse_sync`, `run_reverse_sync`, `list_streams` |
| Collaboration | `list_comments`, `add_comment`, `list_templates`, `install_template` |
| Agents | `list_agents`, `ask_agent` (ZAANIX agents and remote A2A agents) |
| The workspace | `workspace_health`, `list_backups`, `backup_workspace`, `create_stream`, `git_status`, `git_commit`, `get_usage` |

Anything that changes data, publishes or reaches outside waits for approval (below).

**Resources** — `duckdb://workspaces`, `duckdb://schemas/{workspace_id}` (DDL + column map + files), `duckdb://system/resources` (CPUs, RAM, DuckDB ceiling, spill disk, active engines), `duckdb://guides/mosaic-spec` (how to write a Mosaic dashboard spec).

**Prompts** — `data_quality_audit(table_or_path)`, `sql_optimization(sql)`, `build_mosaic_dashboard(table_or_path, goal?)` and `build_data_pipeline(source, goal?)` encode complete agent workflows over the tools above.

## Human-in-the-loop

Any mutating statement from an MCP / API-token actor returns an `approval_required` challenge — the verbs, per-statement previews and how to proceed — instead of executing. The agent shows it to the human; if approved, it calls the tool again with the identical SQL and `dry_run: false`. Members of a shared workspace with **viewer** access cannot mutate even then.

## Registered agents

The **Agents** page (*MCP clients*) registers the agents that call ZAANIX and gives each one a dedicated, workspace-scoped token (`read + mcp`, optionally `write` — mutations are still held for approval). Every tool call is attributed to the agent (call/error counters, "last seen", the live inspector shows the agent name and whether it came over MCP or REST). Copy-paste snippets are generated per framework with the token substituted:

| Framework | Integration |
|---|---|
| **Strands Agents** | `MCPClient(lambda: streamablehttp_client(url, headers=…))` → `Agent(tools=…)` |
| **LangGraph** / **LangChain** | `langchain-mcp-adapters` `MultiServerMCPClient` → `create_react_agent` / `create_agent` |
| **CrewAI** | `MCPServerAdapter({url, transport: "streamable-http", headers})` |
| **AgentCore Runtime** | `BedrockAgentCoreApp` entrypoint deployable with the starter toolkit; ZAANIX can invoke it back (`InvokeAgentRuntime`) from the Agents page or as a Copilot backend |
| **AgentCore Gateway** | ZAANIX as an MCP server target or an OpenAPI target (API-key credential provider holding the ZAANIX token) |
| **Bedrock Agents (Classic)** | Action group from the generated OpenAPI document + a Lambda forwarder to the REST façade |
| **Custom / HTTP** | `curl`, Python `requests`, or any MCP client config |

Endpoints: `GET/POST /api/agents`, `GET/PATCH/DELETE /api/agents/:id`, `POST /api/agents/:id/rotate-token`, `GET /api/agents/:id/snippets`, `POST /api/agents/:id/test`, `POST /api/agents/:id/invoke` (SSE), `GET /api/agents/discover?kind=…`, `GET /api/agents/frameworks`.

## ZAANIX Bot

An in-app assistant docked beside the workbench and the dashboard builder. Every turn is hydrated with the workspace's tables and views (columns + types), the data files in the jail, the configured cloud buckets, the SQL in the active tab and — for selected files or tables — `SUMMARIZE` statistics.

| Provider | Notes |
|---|---|
| **Anthropic** | Official SDK, streaming, `claude-opus-5` by default |
| **OpenAI** · **Ollama** | `gpt-4o`, or any local model over the OpenAI-compatible endpoint |
| **Amazon Bedrock** | Converse streaming with model / inference-profile discovery |
| **Bedrock Agent** · **AgentCore runtime** | Route the drawer to your deployed agent; ZAANIX passes the workspace context along as `payload.context` |

Providers: Claude, ChatGPT / OpenAI, Gemini, DeepSeek, OpenRouter, Kimi, Groq, Mistral, Grok, a local Ollama, any OpenAI-compatible endpoint, and Amazon Bedrock / Bedrock Agent / AgentCore. **Settings → Copilot** is the console: pick a vendor card, paste a key (linked to the vendor's console), *Test connection*, *Save for everyone* — stored encrypted, applied immediately, overriding `copilot.*` in the config file; people can also bring their own key (kept in the browser, sent per request, never stored). The **Usage** panel lists the sessions running now and tokens per day, model and person. Actions: *Insert into tab*, *New tab*, *Run & inspect* (executes, then explains the result in business language), *Fix my query*, *Suggest questions*, *Build dashboard* (drafts a Mosaic dashboard spec, validated against your data and created in one click — see [Mosaic dashboards](mosaic.html)). Conversations persist with the context snapshot of each turn.

`POST /api/copilot/chat` streams SSE events (`context` → `delta`* → `done` | `error`); `GET /api/copilot/config` · `GET /api/copilot/providers` · `POST /api/copilot/models` · `GET/PUT/DELETE /api/copilot/settings` + `POST /api/copilot/settings/test` (administrators) · `GET /api/copilot/usage` · `GET /api/copilot/conversations` · `GET /api/copilot/messages` · `DELETE /api/copilot/conversations/:id`.

## ZAANIX agents and the marketplace

**Agents → ZAANIX agents** runs agents ZAANIX hosts itself: instructions, a task for scheduled runs, the tools they may use (read-only), a limit on tool calls, a schedule and channels for their reports. The marketplace installs ready-made ones in one click:

| Agent | Does | Default schedule |
|---|---|---|
| Anomaly investigator | finds unusual metrics and breaks each change down | every morning |
| Weekly business review | last week's headline metrics against the week before and the 4-week average | Mondays |
| Data quality auditor | failing checks, important tables without checks, the checks to add | every morning |
| Pipeline watcher | failed syncs, dbt runs, quality checks, alerts and reverse syncs | every morning |
| Catalog writer | table and column descriptions drafted from profiles | when asked |
| dbt reviewer | failing and slow models, models without tests or docs | Mondays |
| Data analyst | answers questions with the metrics or SQL | when asked |

## Agent2Agent (A2A)

ZAANIX speaks A2A (protocol 0.3) both ways. A ZAANIX agent with *Other agents can call it* gets a public Agent Card and a JSON-RPC endpoint (`message/send`, `message/stream`, `tasks/get`, `tasks/cancel`); each call runs **as the caller**, read-only, under their access policies. Under *Agents you can ask*, add a remote agent by its card's URL; `list_agents` and `ask_agent` let ZAANIX's AI and agents hand it a question. Calls go through the same egress guard as webhooks.

## ZAANIX Agent

For people rather than programs, [ZAANIX Agent](agent-app.html) is a separate web app on top of these tools: missions, a context panel, approvals with an autonomy dial, and briefs — working as the signed-in person.

