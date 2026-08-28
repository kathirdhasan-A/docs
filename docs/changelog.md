# Changelog

All notable changes to BlancoByte CDC Connector are documented here.

---

## [v1.2] — 2026-05-04

### Added

- **Couchbase source** — stream document mutations from Couchbase via DCP (Database Change Protocol). Uses the official Couchbase Kafka Connector, built from source automatically during `docker-compose build`.
- **Couchbase demo data** — bundled `bb-couchbase` container pre-seeded with 5 employee documents (`emp:1001` through `emp:1005`) in the `sourcedb` bucket.
- **Couchbase schema introspection** — column types inferred by sampling existing documents via `USE KEYS` queries. No N1QL index required.
- **Couchbase type mapper** — dedicated `couchbase_schema_manager.py` and `couchbase_type_mapper.py`, completely separate from existing database scripts to avoid regressions.
- **Query Editor persistence** — ClickHouse Query Editor now saves the last query to `localStorage`. Switching pages and returning no longer clears the editor.

### Fixed

- **Pipeline disappearing on Start** — fixed a bug where clicking Start caused the pipeline to vanish from the list while the connector was registering. Pipeline now immediately shows “Starting…” and remains visible throughout startup.
- **Non-blocking start endpoint** — `/api/pipelines/{id}/start` now returns immediately. All heavy work (Debezium registration, ClickHouse table creation, Kafka topic wait) runs in a background thread. The FastAPI event loop is no longer blocked.
- **`list_pipelines` crash** — fixed `AttributeError: 'PipelineMetrics' object has no attribute 'finished_at'`. The missing field was causing `list_pipelines` to throw on every poll interval, making the pipeline list appear empty in the UI.
- **Internal service hostnames** — fixed all hardcoded hostnames (`postgres`, `kafka`, `clickhouse`, `debezium`) to their correct Docker network names (`bb-postgres`, `bb-kafka`, `bb-clickhouse`, `bb-debezium`). This was causing 502 errors and test connection failures.
- **Kafka topic format** — added `schema_name` field to `PipelineConfig`. Topics are now built as `{prefix}.{schema}.{table}` instead of `{prefix}.{table}`, fixing “topic not ready” loops for PostgreSQL pipelines.
- **docker-compose service names** — all service keys and `depends_on` references standardised to `bb-` prefix. Previously, `yaml.dump()` had stripped the prefix during edits, causing `no such service: sink` errors.
- **nginx upstream** — fixed `nginx.conf` upstream from `sink:8080` to `bb-sink:8080`. The old value caused `bb-ui` to fail on startup with “host not found in upstream”.
- **UI default hostnames** — updated all default connection values in the pipeline wizard to use `bb-` prefixed hostnames (`bb-postgres`, `bb-mysql`, `bb-clickhouse`, etc.).
- **Zookeeper healthcheck** — removed volume mount that caused permission errors. Set `ZOOKEEPER_DATA_DIR=/tmp/zookeeper` and increased healthcheck `start_period` to avoid false failures.
- **MongoDB init hostname** — fixed `host: "mongodb:27017"` to `host: "bb-mongodb:27017"` in `init/mongodb/01-init.js`.

---

## [v1.1] — 2026-04-28

### Added

- **Column filtering** — exclude specific columns from replication per table directly in the pipeline wizard. PK columns are always protected and cannot be excluded.
- **MongoDB source** — stream document-level changes via native Change Streams. Requires a replica set (single-node `rs0` included in the bundled setup). Nested objects and arrays are stored as JSON strings in ClickHouse.
- **Batch auto-stop** — batch pipelines now detect end-of-partition (EOF) across all Kafka partitions and stop automatically. Final status shows ✓ Complete or ✗ Failed with duration and avg events/sec.
- **Alerting** — configure a webhook URL and lag threshold in the pipeline wizard. Notifications are sent to Slack, Teams, or any HTTP endpoint when the pipeline falls behind.
- **Pipeline scheduling** — run batch pipelines on a cron expression (daily, hourly, or custom).
- **S3 / object storage sink** — optionally write CDC events as Parquet files to S3 alongside ClickHouse. Configurable bucket, region, prefix, and credentials.
- **External Kafka support** — connect to your own Kafka cluster (MSK, Confluent Cloud, or self-hosted) instead of the bundled one.
- **ClickHouse Query Editor** — dedicated SQL editor in the UI for running queries directly against the destination ClickHouse instance.
- **Dead letter queue** — failed messages are written to `cdc._bb_dlq` with full error details for later inspection.
- **ErrorBoundary** — React error boundary prevents white screen crashes in the UI. Errors are shown inline with a retry button.

### Fixed

- **`n.filter is not a function`** — `GET /api/pipelines` occasionally returned a non-array response. The UI now guards with `Array.isArray()` and the backend wraps `list_pipelines` in a per-pipeline try/except so a single failure never wipes the full list.
- **PK column exclusion crash** — UI prevents PK columns from being excluded; backend has a matching safety check.
- **MongoDB connection error** — fixed connection string to include `directConnection=true&replicaSet=rs0` with a 5-second timeout.
- **Schema mismatch on restart** — ClickHouse tables are dropped and recreated on pipeline start to ensure schema consistency.
- **Insert fails on column-filtered tables** — sink now checks `DESCRIBE TABLE` before insert and silently drops columns not present in the destination table.
