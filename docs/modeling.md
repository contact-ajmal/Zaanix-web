---
title: Models, metrics and quality
order: 30
group: Model and govern
description: dbt projects run in the workspace's own engine, a semantic layer of metrics defined once, data quality checks on a schedule, metric monitors, data prep recipes and join discovery.
---

Everything on this page lives under **Data** in the platform: *Prepare*, *Models*, *Metrics* and *Quality*. Each part is also a set of tools for agents, and ZAANIX AI reads all of it.

## dbt projects

**Data → Models** keeps dbt projects in a workspace and runs them in the workspace's own engine. Start from the starter project (a seed, a staging view, a table on top, descriptions and tests) or import a project folder.

How a run works: real **dbt Core** (with dbt-duckdb) compiles the project against a *shadow* database — empty tables with the workspace's real schemas and columns, so `is_incremental()` and introspection work — and ZAANIX then runs the compiled SQL **in the workspace's engine, as the person running it**. The SQL guard, access policies, the audit log and the cache apply as for any query.

- **Commands**: `build`, `run`, `test`, `seed` and `compile`, with `--select`, `--exclude`, `--full-refresh` and the project's vars.
- **Materialisations**: view, table, incremental (append, or delete + insert on `unique_key`) and ephemeral; data tests with `severity`, `warn_if` and `error_if`. As in `dbt build`, a failure skips everything downstream.
- **Schedules**: every N minutes or cron (UTC); scheduled runs run as the project's creator.
- **After a run**, descriptions and tags become catalog notes, and lineage shows the project building its models.
- **From the workbench**, *dbt model* saves the tab's SELECT as a model (references to the project's own models become `ref()`).
- **Agents** create projects, write files and run them; building creates tables, so an agent's build waits for approval with a list of every relation it would create.

dbt Core is installed into a virtualenv on the first run (or ahead of time by an administrator). Not run yet: snapshots, Python models, unit tests and custom materialisations.

## The semantic layer

**Data → Metrics** holds metrics defined once, in dbt's MetricFlow shape — written by hand, imported from every dbt project that declares them, or scaffolded from a table:

```yaml
semantic_models:
  - name: orders
    table: orders
    default_time_dimension: order_date
    entities:
      - { name: order, type: primary, expr: order_id }
      - { name: customer, type: foreign, expr: customer_id }
    dimensions:
      - { name: order_date, type: time, granularity: day }
      - { name: region, type: categorical }
    measures:
      - { name: revenue, agg: sum, expr: amount }
      - { name: order_count, agg: count }
metrics:
  - { name: total_revenue, label: Revenue, type: simple, measure: revenue }
  - { name: orders, type: simple, measure: order_count }
  - { name: aov, type: ratio, numerator: total_revenue, denominator: orders }
```

A query — metrics, group by (a dimension, a time grain such as `order_date__month`, or a dimension of another model through an entity), filters, order and limit — compiles to one SELECT and runs as the caller, under their access policies. Saving validates every model and metric against the engine.

The same definitions answer **the Metrics explorer** (with a question box: "revenue by region last quarter"), **ZAANIX AI** (it answers with a metric query, computed exactly as defined, never a guess), **dashboards** and **agents** (`list_metrics`, `query_metrics`).

## Data quality

**Data → Quality** keeps suites of checks on tables:

| Check | Fails on |
|---|---|
| `not_null`, `unique` | nulls; values that appear more than once |
| `accepted_values`, `range` | any other value; values outside the bounds |
| `relationships` | values with no match in another table |
| `expression`, `custom_sql` | rows where a condition is false; any row a SELECT returns |
| `row_count`, `freshness` | too few or too many rows; a newest value that is too old |

Each check can be limited to some rows, allow a tolerance and be an error or a warning. Every check compiles to one SELECT of the failing rows, so a failing check shows them and opens in the workbench. **Suggest** profiles a table and proposes the checks it passes today. Suites run by hand, from agents, from an orchestrator or on a schedule, and tell their channels when the status changes. The Data explorer shows each table's quality status; dbt test results sit alongside.

**Watches** (under Quality) tell you when a table's columns change or its data stops arriving.

## Metric monitors

**Data → Metrics → Monitors** watches metrics for unusual values. A monitor computes a metric per day, week or month and compares the latest complete period with the ones before it (the median and a robust spread; daily numbers against the same weekday). With *explain by* a dimension, each change says which segments drove it — *"Most of the drop came from region = EU (−180, 100% of the change)."* Each finding is recorded once as an **insight**, delivered to channels, shown on Home, and one click from *Ask AI why*.

## Prepare data

**Data → Prepare** builds a recipe from a source table and steps — filter, keep or drop columns, rename, change type, fill empty values, clean text, find and replace, add a column, split a column, remove duplicates, sort — compiled to one readable SELECT with a step per CTE. The preview shows the result and how many rows each step leaves. Save the recipe as a view, a table or a dbt model.

## How tables join

**Find joins** lists how a workspace's tables relate: declared foreign keys, and relationships inferred from names (`orders.customer_id → customers.id`) and then confirmed on the data — the share of values that match, the rows an inner join would drop, and the cardinality.
