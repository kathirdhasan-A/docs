# BlancoByte CDC Connector

**Real-time Change Data Capture pipeline: Source DB → Debezium → Kafka → ClickHouse**

Stream every INSERT, UPDATE, and DELETE from your databases into ClickHouse within milliseconds — no code required. Manage everything through a clean web UI.

**Supported sources:** PostgreSQL · MySQL · MariaDB · MongoDB
**Destination:** ClickHouse
**Version:** v1.1

![Screenshot 2026-04-25 at 18 12 21](https://github.com/user-attachments/assets/3c991ee3-178c-401e-bc52-ea747253553c)

![Screenshot 2026-04-25 at 18 12 35](https://github.com/user-attachments/assets/8a632032-ab6b-45b2-9f5b-b76697bb003d)

![Screenshot 2026-04-28 at 18 09 16](https://github.com/user-attachments/assets/afe990c5-d2c8-4465-bfde-8eae0bc00d8e)

## Quick Start

```
# 1. Extract the project
unzip BlancoByte-CDC-v3.zip
cd blancobyte-cdc

# 2. Set required permissions
chmod 400 init/mongodb/keyfile

# 3. Build and start all services
docker-compose build --no-cache
docker-compose up -d

# 4. Wait ~60 seconds for all services to become healthy
docker-compose ps

# 5. Open the UI
open http://localhost:3000
```

## Architecture

```
PostgreSQL ──┐
MySQL ────────┤
MariaDB ─────┼──► Debezium (Kafka Connect) ──► Kafka ──► BlancoByte Sink ──► ClickHouse
MongoDB ─────┘                                                    │
                                                             FastAPI REST API
                                                                  │
                                                             React UI :3000
```

**CDC Modes:**

- **Streaming** — Continuous, always-on. Every INSERT / UPDATE / DELETE is replicated in real time.
- **Batch** — Full snapshot then stops. Useful for one-time migrations.

**Dead Letter Queue:** Failed messages are written to `cdc._bb_dlq` with full error details.

## Services & Credentials

| Service            | URL / Host                 | Port  | Username | Password | Database |
| ------------------ | -------------------------- | ----- | -------- | -------- | -------- |
| BlancoByte UI      | http://localhost:3000      | 3000  | —        | —        | —        |
| Sink API (Swagger) | http://localhost:8080/docs | 8080  | —        | —        | —        |
| Debezium REST API  | http://localhost:8083      | 8083  | —        | —        | —        |
| Kafka              | localhost                  | 9093  | —        | —        | —        |
| Schema Registry    | http://localhost:8081      | 8081  | —        | —        | —        |
| ClickHouse HTTP    | http://localhost:8123      | 8123  | bbuser   | bbpass   | cdc      |
| ClickHouse Native  | localhost                  | 9000  | bbuser   | bbpass   | cdc      |
| PostgreSQL         | localhost                  | 5432  | bbuser   | bbpass   | sourcedb |
| MySQL              | localhost                  | 3306  | bbuser   | bbpass   | sourcedb |
| MariaDB            | localhost                  | 3307  | bbuser   | bbpass   | sourcedb |
| MongoDB            | localhost                  | 27017 | bbuser   | bbpass   | sourcedb |

## Demo Data

All four source databases are pre-seeded with realistic sample data on first startup. No setup required.

### PostgreSQL — E-commerce dataset

| Table         | Description                                 |
| ------------- | ------------------------------------------- |
| `users`       | Customer accounts with email, balance, role |
| `products`    | Product catalog with SKU, price, stock      |
| `orders`      | Order records linked to users               |
| `order_items` | Line items per order                        |
| `events`      | User activity event log                     |

### MySQL — Retail dataset

| Table          | Description                                  |
| -------------- | -------------------------------------------- |
| `customers`    | Customer profiles with credit and VIP status |
| `inventory`    | Stock items with warehouse location          |
| `transactions` | Payment transaction records                  |

### MariaDB — B2B sales dataset

| Table               | Description                                 |
| ------------------- | ------------------------------------------- |
| `members`           | Member accounts with country and balance    |
| `catalog_items`     | Product catalog with SKU and stock quantity |
| `sales_orders`      | Sales orders linked to members              |
| `sales_order_lines` | Line items per sales order                  |

### MongoDB — SaaS dataset

| Collection  | Description                               |
| ----------- | ----------------------------------------- |
| `customers` | Customer documents with tier and balance  |
| `products`  | Product catalog with nested tags          |
| `orders`    | Order documents with embedded items array |

> **Note:** MongoDB requires a replica set for Change Streams. The bundled container starts with `--replSet rs0` and is bootstrapped automatically on first run.

## Creating Your First Pipeline

1. Open **http://localhost:3000** and click **New Pipeline**
2. **Source** — Select PostgreSQL, MySQL, MariaDB, or MongoDB. Credentials are pre-filled for demo databases. Click **Test Connection**.
3. **Tables** — Select one or more tables or collections to replicate
4. **Kafka** — Leave defaults (`kafka:9092`, topic prefix auto-detected)
5. **ClickHouse** — Leave defaults (`clickhouse:9000`, database `cdc`)
6. **Type Mapping** — Review auto-detected column types, override if needed
7. Click **Create Pipeline** → then **Start** on the Pipelines page

The snapshot runs automatically. Watch events appear live in **Monitor → Live Event Stream**.

## Testing CDC in Action

### PostgreSQL

```
# INSERT
docker exec -it bb-postgres psql -U bbuser -d sourcedb -c \
  "INSERT INTO users (email, full_name, balance) VALUES ('test@demo.io', 'Test User', 999.00);"

# UPDATE
docker exec -it bb-postgres psql -U bbuser -d sourcedb -c \
  "UPDATE users SET balance = 1234.56 WHERE email = 'test@demo.io';"

# DELETE
docker exec -it bb-postgres psql -U bbuser -d sourcedb -c \
  "DELETE FROM users WHERE email = 'test@demo.io';"
```

### MySQL

```
# INSERT
docker exec -it bb-mysql mysql -u bbuser -pbbpass sourcedb -e \
  "INSERT INTO customers (email, full_name, status) VALUES ('test@demo.io', 'Test User', 'active');"

# UPDATE
docker exec -it bb-mysql mysql -u bbuser -pbbpass sourcedb -e \
  "UPDATE customers SET status = 'vip' WHERE email = 'test@demo.io';"

# DELETE
docker exec -it bb-mysql mysql -u bbuser -pbbpass sourcedb -e \
  "DELETE FROM customers WHERE email = 'test@demo.io';"
```

### MariaDB

```
# INSERT
docker exec -it bb-mariadb mariadb -u bbuser -pbbpass sourcedb -e \
  "INSERT INTO members (email, full_name, balance) VALUES ('test@demo.io', 'Test User', 500.00);"

# UPDATE
docker exec -it bb-mariadb mariadb -u bbuser -pbbpass sourcedb -e \
  "UPDATE members SET balance = 9999.00 WHERE email = 'test@demo.io';"

# DELETE
docker exec -it bb-mariadb mariadb -u bbuser -pbbpass sourcedb -e \
  "DELETE FROM members WHERE email = 'test@demo.io';"
```

### MongoDB

```
# INSERT
docker exec -it bb-mongodb mongosh \
  -u bbuser -p bbpass --authenticationDatabase admin \
  --eval 'db.getSiblingDB("sourcedb").customers.insertOne({
    customer_id: "C006", email: "test@demo.io",
    full_name: "Test User", country: "NL",
    tier: "gold", balance: 5000.00, is_active: true, created_at: new Date()
  })'

# UPDATE
docker exec -it bb-mongodb mongosh \
  -u bbuser -p bbpass --authenticationDatabase admin \
  --eval 'db.getSiblingDB("sourcedb").customers.updateOne(
    { customer_id: "C001" },
    { $set: { balance: 99999.00, tier: "platinum" } }
  )'

# DELETE
docker exec -it bb-mongodb mongosh \
  -u bbuser -p bbpass --authenticationDatabase admin \
  --eval 'db.getSiblingDB("sourcedb").customers.deleteOne(
    { customer_id: "C004" }
  )'
```

### Query in ClickHouse

```
-- Current state (deduplicated, no deleted rows)
SELECT * FROM cdc.users FINAL WHERE _cdc_deleted = 0 ORDER BY _cdc_ts DESC LIMIT 10;

-- Full change history for a record
SELECT _cdc_op, _cdc_ts, * FROM cdc.users WHERE id = 1 ORDER BY _cdc_ts;

-- Operation counts by table
SELECT _cdc_op, count() FROM cdc.users GROUP BY _cdc_op;

-- MongoDB: parse nested items array
SELECT order_id, status,
    JSONExtractString(item, 'sku')   AS sku,
    JSONExtractFloat(item, 'price')  AS price
FROM cdc.orders FINAL
ARRAY JOIN JSONExtractArrayRaw(items) AS item
WHERE _cdc_deleted = 0;
```

## CDC Metadata Columns

Every replicated table receives these four columns automatically:

| Column         | Type                   | Description                                              |
| -------------- | ---------------------- | -------------------------------------------------------- |
| `_cdc_op`      | LowCardinality(String) | Operation: `INSERT` · `UPDATE` · `DELETE` · `SNAPSHOT`   |
| `_cdc_ts`      | DateTime               | Timestamp when the event occurred in the source DB       |
| `_cdc_version` | Int64                  | Kafka offset — used for ReplacingMergeTree deduplication |
| `_cdc_deleted` | UInt8                  | `0` = live row · `1` = deleted (soft delete)             |

## Type Mapping Reference

### PostgreSQL → ClickHouse

| PostgreSQL                    | ClickHouse            |
| ----------------------------- | --------------------- |
| `int2` / `smallint`           | `Int16`               |
| `int4` / `integer`            | `Int32`               |
| `int8` / `bigint`             | `Int64`               |
| `float4` / `real`             | `Float32`             |
| `float8` / `double precision` | `Float64`             |
| `numeric(p,s)`                | `Decimal(p,s)`        |
| `varchar` / `text`            | `String`              |
| `bool`                        | `Bool`                |
| `date`                        | `Date32`              |
| `timestamp`                   | `DateTime64(3)`       |
| `timestamptz`                 | `DateTime64(3,'UTC')` |
| `uuid`                        | `UUID`                |
| `json` / `jsonb`              | `String`              |

### MySQL → ClickHouse

| MySQL              | ClickHouse               |
| ------------------ | ------------------------ |
| `TINYINT(1)`       | `Bool`                   |
| `TINYINT`          | `Int8`                   |
| `SMALLINT`         | `Int16`                  |
| `INT`              | `Int32`                  |
| `BIGINT`           | `Int64`                  |
| `FLOAT`            | `Float32`                |
| `DOUBLE`           | `Float64`                |
| `DECIMAL(p,s)`     | `Decimal(p,s)`           |
| `DATETIME`         | `DateTime64(3)`          |
| `TIMESTAMP`        | `DateTime64(3,'UTC')`    |
| `VARCHAR` / `TEXT` | `String`                 |
| `ENUM`             | `LowCardinality(String)` |
| `JSON`             | `String`                 |

### MariaDB → ClickHouse

Same type mapping as MySQL. MariaDB uses the MySQL Debezium connector and shares identical type semantics.

### MongoDB → ClickHouse

| MongoDB / BSON | ClickHouse      |
| -------------- | --------------- |
| `string`       | `String`        |
| `int32`        | `Int32`         |
| `int64`        | `Int64`         |
| `double`       | `Float64`       |
| `boolean`      | `Bool`          |
| `date`         | `DateTime64(3)` |
| `objectId`     | `String`        |
| `array`        | `String` (JSON) |
| `object`       | `String` (JSON) |

> Nested documents and arrays are stored as JSON strings in ClickHouse. Use `JSONExtractString`, `JSONExtractFloat`, and `JSONExtractArrayRaw` to query them.

## Directory Structure

```
blancobyte-cdc/
├── docker-compose.yml
├── debezium/
│   ├── Dockerfile
│   ├── connectors/
│   │   ├── postgres-connector.json
│   │   ├── mysql-connector.json
│   │   ├── mariadb-connector.json
│   │   └── mongodb-connector.json
│   └── register.sh
├── init/
│   ├── postgres/01-init.sql
│   ├── mysql/01-init.sql
│   ├── mariadb/01-init.sql
│   ├── mongodb/01-init.js
│   └── clickhouse/01-init.sql
├── sink/
│   ├── main.py
│   ├── consumer.py
│   ├── clickhouse_writer.py
│   ├── schema_manager.py
│   └── type_mapper.py
└── ui/
    ├── public/logo.png
    └── src/
        ├── components/
        └── pages/
```

## Troubleshooting

**Connector registration failed:**

```
docker logs bb-connector-init
docker logs bb-debezium --tail 50
```

**MongoDB not connecting:**

```
docker logs bb-mongodb-init
# Ensure keyfile permissions are correct:
chmod 400 init/mongodb/keyfile
```

**No events arriving in ClickHouse:**

```
docker logs bb-sink --tail 50
docker exec bb-kafka kafka-topics.sh --bootstrap-server localhost:9092 --list
```

**TypeMismatch or insert errors:**

```
docker logs bb-sink --tail 100 | grep ERROR
```

**Full reset:**

```
docker-compose down -v
docker-compose up -d
```

## Stopping & Restarting

```
# Stop — data preserved
docker-compose down

# Stop + wipe all data
docker-compose down -v

# Restart
docker-compose up -d
```

*BlancoByte CDC Connector — v1.1*
