# Architecture

BlancoByte CDC Connector is built on a battle-tested open-source stack. Each component has a single responsibility, making the system easy to debug, scale, and extend.

### Full data flow

```
Source DB ──► Debezium ──► Kafka ──► BlancoByte Sink ──► ClickHouse
               (Kafka        (Message    (Python +              │
               Connect)      Bus)        FastAPI)          React UI
                                              │
                                         REST API :8080
```

### Component breakdown

**Debezium** acts as the bridge between your source database and Kafka. It reads directly from the database transaction log — PostgreSQL’s Write-Ahead Log (WAL) for logical replication, and the binary log (binlog) for MySQL and MariaDB. This means zero impact on your database performance: Debezium never runs queries against your tables. It simply tails the log and publishes every change as a structured event to Kafka.

**Apache Kafka** is the message bus that decouples the source from the destination. Events land in Kafka topics — one topic per table, named by prefix and schema (e.g. `pg.public.users`). Kafka buffers events durably, so if the sink is temporarily unavailable, no data is lost. Events are replayed from the last committed offset when the sink reconnects.

**BlancoByte Sink** is a custom Python service built with FastAPI. It consumes Kafka topics, applies type coercion to match ClickHouse column types, and batch-inserts rows using the ClickHouse native protocol. It also exposes the REST API that the UI calls to create, start, stop, and monitor pipelines. All pipeline state is stored on disk inside the sink container.

**ClickHouse** is the destination. Tables are created with `ReplacingMergeTree` by default, which deduplicates rows on `_cdc_version` (the Kafka offset). This means you can replay events without ending up with duplicate rows. The `FINAL` keyword in queries triggers deduplication at query time for fully consistent results.

**React UI** is served by nginx at port 3000. It communicates exclusively with the BlancoByte Sink API — it never connects directly to Kafka, Debezium, or ClickHouse. All pipeline operations, monitoring, and query execution are proxied through the sink API.

### CDC modes

**Streaming** — The connector runs continuously. After the initial snapshot, every change in the source database is captured within milliseconds and delivered to ClickHouse. This is the default mode and the primary use case for BlancoByte CDC.

**Batch** — The connector takes a full snapshot of the selected tables, publishes all rows to Kafka, and then stops. Useful for one-time data migrations or backfilling ClickHouse from an existing database.
