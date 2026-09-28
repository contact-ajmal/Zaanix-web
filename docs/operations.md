---
title: Running DuckView for a team
order: 52
group: Operate
description: Workspace management, backups and bundles, quotas and the idle policy, cluster mode, the Postgres protocol for BI tools, orchestration from Airflow, Dagster and Prefect, and usage and cost.
---

## Workspace management

- **Administration → Workspaces** lists every workspace — owner, storage, size, members, engine, last activity, cost this month — with filters and bulk actions. A destructive bulk action asks you to type the name, or "delete N", to confirm.
- **New workspace** is a wizard: basics, storage (the data directory, any folder on the server, cloud storage, in-memory or MotherDuck), engine settings, a starting point — empty, a [template](delivery.html#templates) or a **clone** of another workspace (its data and its queries, dashboards, notebooks, metrics and checks) — and people.
- **Each workspace has a detail page**: overview and health checks (worst first), members, storage, engine, sources, integrations, usage, audit and lifecycle.

## Backups and bundles

A **bundle** (`.duckview`) is one DuckDB file holding a workspace's tables and its settings and objects. Export one to move a workspace; import it to create one. A **backup** is a bundle kept by the server — by hand or on a schedule. Restoring always takes a safety backup first; restoring the objects (not only the data) is a choice. Compare a backup with now with *Compare* (`diff_tables`).

## Quotas and policies

**Administration → Workspaces → Policies** sets the organisation's rules:

- **Quotas**: storage per workspace, an engine memory cap, and query time per day.
- **Idle policy**: warn owners after N days without use, and archive after M days, telling a channel.
- **Creation**: administrators only, or everyone; a naming pattern with a hint; default memory, threads and query timeout.

## Cluster mode

Several DuckView nodes behind one load balancer share PostgreSQL metadata and a ReadWriteMany data directory:

- **One engine per workspace.** A lease names the node that opens a workspace's file; the other nodes forward its queries there.
- **Scheduled work runs once.** Syncs, alerts, snapshots, checks, dbt runs, monitors and agents are each claimed by exactly one node; streams hold a lease.
- **Failover.** A node that stops heartbeating loses its leases after `lease_seconds`, and the next request opens its workspaces elsewhere.
- **Sticky routing** only for data apps and MCP SSE sessions (`ClientIP` affinity); the UI, REST, Streamable HTTP and the Postgres protocol can go to any node.

Set `cluster.enabled`, the same `cluster.secret`, `JWT_SECRET` and `ENCRYPTION_KEY` on every node, and `cluster.advertise_url` for how the others reach each one. `GET /api/admin/cluster` shows the nodes and what each holds.

## The Postgres protocol

Anything that speaks PostgreSQL can query DuckView: Tableau, Power BI, Metabase, Superset, DBeaver, Excel, psql, JDBC and ODBC drivers, psycopg and node-postgres. Turn it on with `pgwire.enabled` (port 5433 by default); connect with your email and password or an API token, and a workspace as the database. Everything runs through the workbench's path — the SQL guard, your access policies and the audit log. Use TLS (`pgwire.tls_cert`, `pgwire.tls_key`, `pgwire.require_tls`) anywhere but localhost.

## Orchestration

Pipelines start DuckView work and wait for its status: `POST /api/orchestrate/runs {kind, id}` and `GET /api/orchestrate/runs/:id?wait=30`. Kinds: a sync, a dbt command, a quality suite, a reverse sync, a notebook, an alert, a snapshot, a DuckView agent, a metric monitor, or a SQL check that fails when it returns rows. The Python SDK wraps them:

```bash
pip install "duckview[airflow]"     # or [dagster], [prefect]
```

- **Airflow**: operators (`DuckViewSyncOperator`, `DuckViewDbtOperator`, `DuckViewQualityCheckOperator`, `DuckViewSQLCheckOperator`, …) and a *DuckView* connection type.
- **Dagster**: a `DuckViewResource` and `duckview_op(kind, id)`.
- **Prefect**: tasks (`run_sync`, `run_dbt`, `run_quality_suite`, …) and a `DuckViewCredentials` block.

## Usage and cost

**Settings → Usage & cost** shows what DuckView was used for and what it cost at your rates: **compute** (query time from every surface, and pipeline time), **AI** (tokens by model, at list prices you can override; turns paid with a person's own key are shown apart) and **storage**. Administrators see the organisation by day, workspace, person, source and model; everyone else sees their own. **Budgets** — for the organisation or one workspace — notify channels at thresholds of actual or forecast spend. Export as CSV.
