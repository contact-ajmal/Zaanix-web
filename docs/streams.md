---
title: Streams, CDC and reverse ETL
order: 22
group: Connect
description: Kafka, Kinesis and HTTP pushes appended to tables as events arrive; Postgres and Debezium change data capture; query results sent to databases, files, APIs, Iceberg and Delta tables.
---

## Streams

**Connections → Streams** appends events to a workspace table as they arrive, from one of three sources:

- **Kafka** — any Kafka-compatible broker (Apache Kafka, Confluent, Redpanda, MSK): a topic, an optional consumer group, from the oldest or only new messages, with TLS and SASL (PLAIN or SCRAM).
- **Amazon Kinesis** — every shard, including new ones, with a cloud connection's credentials or the server's own.
- **HTTP pushes** — your app POSTs JSON (an object, an array or newline-delimited) to `/api/streams/:id/push` with the stream's key; it answers `202`.

Messages are written in micro-batches. A JSON object's fields become columns; the first batch creates the table with inferred types, later batches add new fields as columns, and a value that does not fit becomes NULL instead of stopping the stream. Optional metadata columns carry the key, partition, offset and times. Delivery is **at least once**: Kafka offsets and Kinesis checkpoints are saved only after a batch is written, so a restarted stream resumes where it stopped.

## Change data capture

A stream can **mirror** a table instead of appending: inserts and updates replace each key's row, deletes remove it. With *Keep a history of changes*, every change is also appended to `<table>__changes` with its operation.

- **Postgres CDC** reads a table through logical replication. ZAANIX creates the publication and slot, optionally copies the existing rows first, creates the table with the source's types, then applies changes — acknowledging the WAL only after each batch is written. It needs `wal_level = logical` and a user with REPLICATION; *Test connection* checks both. Removing the stream drops its slot.
- **Debezium change events** on a Kafka topic or an HTTP push cover MySQL, SQL Server, Oracle, MongoDB and anything else Debezium captures.
- **Any JSON feed** can mirror too, with *Keep the latest per key*.

Streams appear in lineage (topic → stream → table) and on the live feed.

## Reverse ETL

**Connections → Reverse ETL** sends query results out of a workspace — by hand, on a schedule, from an orchestrator, or from agents (with approval). A reverse sync is one read-only SELECT, a destination and a mode.

| Destination | What happens |
|---|---|
| Database table | A Postgres, MySQL or SQLite connection with read-only turned off; the table is created when missing |
| Iceberg table | A table of an Iceberg REST catalog, AWS Glue or S3 Tables; each run is one Iceberg transaction |
| Delta Lake table | A table folder in the data directory or a bucket |
| Files | Parquet, CSV or JSON, in the data directory or a bucket |
| HTTP API | JSON posted in batches, with encrypted headers and an idempotency key per batch |

| Mode | Sends |
|---|---|
| `replace` | the whole result; the destination holds exactly it |
| `append` | the whole result, added |
| `upsert` | only rows new or changed since the last successful run, matched on key columns |
| `mirror` | upsert, plus the keys that disappeared, deleted or sent as deletes |

The query runs as the sync's author, under their access policies. Change detection keeps a hash of every row delivered and is replaced only after a successful delivery, so a failed run is retried in full. **Preview next run** shows how many rows would be sent and deleted without sending anything. From the workbench, ⋯ → *Send results to…* starts a reverse sync with the tab's SQL.
