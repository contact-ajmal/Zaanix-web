---
title: DuckView Agent
order: 40
group: AI and agents
description: The web app for handing data work to an agent — missions, the context panel, capabilities, the canvas and briefs, approvals and the autonomy dial — and how to run it next to DuckView.
---

DuckView Agent is a web app next to the DuckView platform. You sign in with your DuckView account, choose the data, and say what you need. The agent works in your DuckView workspace **as you**: it reads with your access, and DuckView holds its changes until you approve them. Results come back as findings, charts, tables and SQL, and everything it makes opens in DuckView.

DuckView itself is unchanged by it. People who prefer the platform keep using it; the agent is the second way to work.

![A mission: the question, the answer with key findings, results as charts, and the canvas](../assets/img/agent-mission.jpg)

## Run it

The agent is its own server (with its own store for missions) and web app, on port 4300. It needs a running DuckView.

With Docker Compose, from the repository:

```bash
# .env — a model for everyone; leave it out and people bring their own key
# provider: anthropic | openai | openrouter | ollama | openai-compatible
AGENT_MODEL_PROVIDER=anthropic
AGENT_MODEL=claude-sonnet-5
AGENT_MODEL_API_KEY=sk-ant-…

docker compose --profile agent up --build -d
# → DuckView on :4200, DuckView Agent on :4300
```

From source, next to a running DuckView:

```bash
pnpm install && pnpm build
DUCKVIEW_URL=http://localhost:4200 pnpm agent
```

For development, `pnpm --filter @duckview/agent-server dev` runs the API on :4300 and `pnpm --filter @duckview/agent-web dev` the UI on :5174.

## Configuration

| Variable | Default | What it is |
|---|---|---|
| `DUCKVIEW_URL` | `http://localhost:4200` | Where the agent server reaches DuckView |
| `DUCKVIEW_PUBLIC_URL` | `DUCKVIEW_URL` | Where people's browsers reach DuckView ("Open in DuckView") |
| `AGENT_PORT`, `AGENT_HOST` | `4300`, `127.0.0.1` | Where DuckView Agent listens |
| `AGENT_DATA_DIR` | `data/agent` | Its own store: missions, memory, sign-ins — never DuckView's data |
| `AGENT_SECRET` | generated into the data directory | Signs cookies and encrypts the stored DuckView credentials |
| `AGENT_SESSION_HOURS` | `12` | How long a sign-in lasts |
| `AGENT_MODEL_PROVIDER` | none | `anthropic`, `openai`, `openrouter`, `ollama` or `openai-compatible`. Without one, people set their own model in the app |
| `AGENT_MODEL`, `AGENT_MODEL_API_KEY`, `AGENT_MODEL_BASE_URL` | none | The model. Anthropic defaults to `claude-sonnet-5` |
| `AGENT_MAX_STEPS`, `AGENT_MAX_RETRIES`, `AGENT_MAX_OUTPUT_TOKENS` | `12`, `2`, `4096` | How far one request may go |
| `AGENT_PRICING` | none | Cost estimates as JSON, e.g. `{"claude-sonnet":{"input":3,"output":15}}` per million tokens |

**Rate limits.** To DuckView, the agent server is one client for everyone who uses it, so DuckView's per-client limit (`server.rate_limit_per_minute`, 600 a minute) is shared by all agent users. The agent server waits out a 429 and retries, but for a team raise the limit — for example `DUCKVIEW__server__rate_limit_per_minute=6000`.

## The home

The home is one question and one prompt card.

- **Capabilities.** Under the prompt: Analyse data, Build dashboard, Chart, Data quality, Investigate, Clean data and, under *More*, Notebook, Data app and Metrics — only those your role in the workspace allows. Choosing one puts it in the prompt as a pill (with its option, when it has one: *Grid* or *Interactive* for a dashboard; *Streamlit*, *Dash* or *Gradio* for an app), changes the placeholder, shows four **sample prompts** written for the data in your context, and points the agent at that capability's tools first. Every other tool stays available.
- **The composer.** `@` mentions tables, views, files, metrics and dashboards as you can see them (a mentioned table joins the request's data); `/chart`, `/dashboard`, `/check`, `/explain` and `/why` start from a common request. ⌘↵ starts the mission.

![The agent home with a capability chosen, sample prompts and the Context panel](../assets/img/agent-home.jpg)

## The context panel

The data the agent works with, kept apart from the prompt. Anything added here goes with the next request as its chosen data; the agent reads it first and may still find more. **Add data** (or the prompt's **+**) opens four sources:

| Source | What you can add |
|---|---|
| **In the workspace** | Tables, views and files, as DuckView's catalog shows them to you |
| **Server files** | Folders and data files on the DuckView server, as its file jail allows |
| **Remote** | Your connected sources — lakehouses (Databricks, Iceberg catalogs), database connections and cloud buckets, browsed down to a table or file — or any http(s), s3, gs or az URL |
| **From my computer** | Files uploaded into the workspace's `uploads/` folder; you can also drop files on the panel |

Data from outside the catalog is opened with DuckView's inspect, as you, before it is added — a URL that cannot be read is refused in words — and reaches the model with its columns. A lakehouse table that runs on a remote warehouse (Databricks) is read with `lakehouse_query` on that warehouse, and the agent is told so. In a mission, the prompt's **+** adds data for the next message.

## Missions

- **Missions as threads.** Each request is a message; the agent's turn shows its plan and steps (folded away), the answer, key findings and results. "Verified metric" appears when a defined metric answered; otherwise the tables the result read are listed. The data you chose and the data the agent found are shown apart. Follow-up suggestions come after each answer.
- **The canvas.** One result shown large beside the thread, as a chart (bar, line or area; pick the axes), a table or the SQL, with *Where this comes from* and its lineage links. **Pin to dashboard**, **Save as query**, download a **CSV**, or **Open SQL** in DuckView.
- **Briefs.** A mission becomes a report page — summary, KPI tiles, charts, tables and findings. Edit it, refresh its numbers (run again as you), save it as a DuckView dashboard to share it, copy it as Markdown, or print it.
- **The sidebar** is live: missions grouped as *Needs you*, *Running*, *Today* and older, with search.

## Approvals and the autonomy dial

Reads always run. There are three kinds of action, each set to *Ask me* or *Go ahead* by each person in **Agent settings**:

| Kind | Examples | Default |
|---|---|---|
| Create things | Dashboard widgets, quality checks, notebooks, saved queries | Go ahead |
| Change or delete data, or export it | SQL that changes tables, models, writing files | Ask me |
| Publish or reach outside DuckView | Publishing apps or endpoints, alerts, reverse syncs | Ask me |

- **Ask me** is kept by the agent server itself: the call waits for you even where DuckView would let it through.
- **Go ahead** approves DuckView's own hold for you, and the steps say so.
- Dropping, deleting or truncating data **always** asks.
- After you approve several of one kind in a row, the settings page suggests letting the agent go ahead with them.

**Approvals** lists everything waiting for you across missions; **Activity** records what the agent made and every approval — asked, given, declined or automatic.

## Security

- **Identity is DuckView's.** Sign-in is passed through to DuckView. The agent server keeps the person's DuckView session and an **agent token** minted for that sign-in (with the scopes their role allows, revoked at sign-out). Both are encrypted (AES-256-GCM) behind a signed httpOnly cookie; the browser never holds a DuckView credential.
- **Reading uses the person's session**, so the catalog, metrics, dashboards and datasets are exactly what they can see.
- **Tools use the agent token**, so DuckView treats each call as an agent acting for the person: their workspace roles, row filters, column masks, the SQL guard and the file jail apply; changes are held until the person approves them; calls are audited as the agent's.
- **Offering tools is not permission.** A viewer or read-only account is offered reading tools only, and DuckView still checks every call.
- **Missions and briefs are private** to the person who started them. Sharing happens in DuckView: a brief saved as a dashboard is shared there, under DuckView's access rules.
- **The canvas's actions are yours.** Pin to dashboard, Save as query and saving a brief run with your DuckView session; refreshing a brief uses the agent token, so a statement that would change data is held.

## Models

Set one model for everyone with `AGENT_MODEL_*`, or let each person bring their own key (**Your model** in the app; the key stays in their browser and is sent only with their requests). Providers: Anthropic, OpenAI, OpenRouter, Ollama and any OpenAI-compatible endpoint. Besides its own tool protocol, the agent reads the native tool-call formats of open models (Qwen, GLM, Llama, LFM), hides their reasoning text, and asks again when a model replies with nothing. Pick a capable model for real work: free, rotating models vary a lot.
