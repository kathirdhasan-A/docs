# About

**BlancoByte CDC Connector** is an open-source Change Data Capture pipeline that streams real-time database changes directly into ClickHouse — with zero custom code required.

Built on top of Debezium and Apache Kafka, it captures every INSERT, UPDATE, and DELETE from your source databases the moment they happen, and delivers them to ClickHouse as structured, queryable events. Whether you’re building real-time analytics dashboards, keeping a data warehouse in sync, or auditing every change across your systems, BlancoByte CDC handles the heavy lifting.

## What it does

Traditional ETL pipelines run on schedules — hourly, nightly, or weekly. CDC is different. Instead of polling your database, it reads directly from the transaction log, which means your ClickHouse tables stay within milliseconds of your source data at all times.

BlancoByte CDC supports three source databases out of the box:

- **PostgreSQL** — via logical replication and the pgoutput plugin
- **MySQL** — via binlog CDC
- **MariaDB** — via binlog CDC

All changes land in ClickHouse with four automatic metadata columns appended to every row: `_cdc_op` (the operation type), `_cdc_ts` (the timestamp), `_cdc_version` (for deduplication), and `_cdc_deleted` (soft delete flag). This means you can query both the current state of your data and its full change history from a single table.

## The interface

Everything is managed through a clean web UI — no YAML files to edit, no command line required after setup. You create a pipeline by pointing it at a source database, selecting which tables to replicate, and clicking Start. The connector registers itself with Debezium automatically, creates the destination tables in ClickHouse, and begins streaming.

A built-in Query Editor lets you run SQL directly against your ClickHouse database. The Monitor view shows live event throughput, per-table operation counts, and latency metrics. Console Log streams real-time logs from the pipeline process.

![](https://blancobyte.com/wp-content/uploads/2026/04/Screenshot-2026-04-25-at-18.12.35-1024x641.png)

## Architecture

```
Source DB → Debezium → Apache Kafka → BlancoByte Sink → ClickHouse
```

The sink is a FastAPI service that consumes Kafka topics, applies type coercion, and batch-inserts into ClickHouse using the native protocol. Tables are created with `ReplacingMergeTree` by default, which deduplicates rows on `_cdc_version` — giving you the latest state of each record without manual cleanup.
