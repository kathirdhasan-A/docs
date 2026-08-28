# Getting Started

Prerequisites

Before you begin, make sure you have the following installed on your machine:

- **Docker Desktop** (version 24 or later)
- **Docker Compose** (included with Docker Desktop)
- At least **8 GB of available RAM** (ClickHouse and Kafka are memory-hungry)
- **10 GB of free disk space**

An internet connection is required on first run to pull Docker images.

## What’s already in the demo databases

When the stack starts up, each database is automatically seeded with realistic sample data:

**PostgreSQL** includes five tables — `users`, `products`, `orders`, `order_items`, and `events` — with pre-populated rows representing an e-commerce dataset. Users have balances, roles, and activity metadata. Products have SKUs, prices, and inventory counts. Orders and order items are linked by foreign keys.

**MySQL** includes three tables — `customers`, `inventory`, and `transactions` — representing a retail operation with customer profiles, stock records, and payment history.

**MariaDB** includes four tables — `members`, `catalog_items`, `sales_orders`, and `sales_order_lines` — representing a B2B sales dataset with member accounts, a product catalog, and multi-line orders.

All tables are distinct across databases so you can run pipelines against all three simultaneously without any naming conflicts in ClickHouse.

### What happens when you start a pipeline

The moment you click **Start** on a pipeline, three things happen automatically:

1. **Debezium connector is registered** — BlancoByte registers a Debezium connector for your source database via the REST API. The connector begins reading from the transaction log immediately.
2. **ClickHouse tables are created** — Destination tables are created in ClickHouse with the schema you defined in the wizard, plus four CDC metadata columns (`_cdc_op`, `_cdc_ts`, `_cdc_version`, `_cdc_deleted`).
3. **Snapshot runs** — Debezium performs an initial snapshot of all existing rows and publishes them to Kafka. The BlancoByte sink reads these events and batch-inserts them into ClickHouse. For the demo databases, this typically completes within 5–10 seconds.

After the snapshot, the connector switches to streaming mode and captures changes as they happen in real time.

### Your first pipeline in 5 minutes

Once the stack is running (see Installation), open your browser and navigate to `http://localhost:3000`.

**Step 1 — Create a new pipeline**

Click **New Pipeline** in the sidebar. Give your pipeline a name (e.g. `prod-users-to-clickhouse`) and select **Streaming** mode for continuous replication.

**Step 2 — Configure your source**

Select your source database type — PostgreSQL, MySQL, or MariaDB. The host, port, and credentials are pre-filled for the bundled demo databases. Click **Test Connection** to verify, then click Next.

**Step 3 — Select tables**

Choose which tables to replicate. The wizard shows table sizes and column counts to help you decide. Select one or more tables and click Next.

**Step 4 — Kafka settings**

The Kafka broker address and topic prefix are pre-configured. Leave the defaults unless you’re connecting to an external Kafka cluster.

**Step 5 — ClickHouse destination**

Enter your ClickHouse connection details. Click **Test ClickHouse Connection** to verify. Choose a table engine — `ReplacingMergeTree` is recommended for CDC workloads. Click Next.

**Step 6 — Type mapping**

Review the column type mappings. The wizard auto-detects types from your source schema and maps them to appropriate ClickHouse types. You can override any column type before creating the pipeline.

**Step 7 — Start**

Click **Create Pipeline**, then click **Start** on the Pipelines page. Within a few seconds you should see the snapshot events streaming in the Monitor view.

### Verifying data in ClickHouse

Open the **Query Editor** tab and run:

```
SELECT _cdc_op, _cdc_ts, *
FROM cdc.your_table
FINAL
WHERE _cdc_deleted = 0
ORDER BY _cdc_ts DESC
LIMIT 10
```

Testing CDC events

Make a change in your source database and watch it appear in ClickHouse in real time:

```
# PostgreSQL example
docker exec -it bb-postgres psql -U bbuser -d sourcedb -c \
  "INSERT INTO users (email, full_name) VALUES ('test@example.com', 'Test User');"

# MySQL example
docker exec -it bb-mysql mysql -u bbuser -pbbpass sourcedb -e \
  "UPDATE customers SET status = 'vip' WHERE id = 1;"

# MariaDB example
docker exec -it bb-mariadb mariadb -u bbuser -pbbpass sourcedb -e \
  "DELETE FROM members WHERE email = 'test@example.com';"
```

The Monitor page will show the event within milliseconds.
