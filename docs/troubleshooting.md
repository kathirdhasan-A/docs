# Troubleshooting

### Connector registration failed

**Symptom:** The pipeline status shows an error immediately after clicking Start.

**Check the connector-init logs:**

```
docker logs bb-connector-init
```

**Check the Debezium REST API directly:**

```
curl http://localhost:8083/connectors
curl http://localhost:8083/connectors/pg-postgres-sourcedb/status
```

**Common cause:** The source database is not yet healthy when the connector tries to register. Try stopping and restarting the pipeline from the UI after 30 seconds.

### No events arriving in ClickHouse

**Symptom:** The pipeline shows as running but the Monitor page shows 0 events.

**Check Kafka topics exist:**

```
docker exec bb-kafka kafka-topics.sh --bootstrap-server localhost:9092 --list
```

**Check the sink consumer is running:**

```
docker logs bb-sink --tail 50
```

**Check the connector is actually running in Debezium:**

```
curl http://localhost:8083/connectors/pg-postgres-sourcedb/status | python3 -m json.tool
```

Look for `"state": "RUNNING"`. If it shows `"state": "FAILED"`, check `docker logs bb-debezium --tail 50`.

### TypeMismatch error on insert

**Symptom:** Errors like `TypeMismatchError: Column id: required argument is not an integer` in the sink logs.

**Cause:** The column type mapping between the source schema and the ClickHouse table does not match. This can happen if a table was created by a previous pipeline with a different schema.

**Fix:** Drop the affected ClickHouse table and restart the pipeline. The table will be recreated with the correct schema.

```
-- Run in ClickHouse (via Query Editor or HTTP API)
DROP TABLE IF EXISTS cdc.your_table;
```

Then stop and restart the pipeline from the UI.

### MariaDB connector not registering

**Symptom:** MariaDB pipeline fails but PostgreSQL and MySQL work fine.

**Check MariaDB is healthy:**

```
docker logs bb-mariadb --tail 20
```

**Check the connector-init output:**

```
docker logs bb-connector-init | grep -i mariadb
```

**Verify binlog is enabled:**

```
docker exec bb-mariadb mariadb -u root -prootpass -e "SHOW VARIABLES LIKE 'log_bin';"
```

### Full reset

If anything is in a broken state, a full reset clears all data and starts fresh.

```
docker-compose down -v
docker-compose up -d
```

This removes all Kafka topics, ClickHouse data, and pipeline state. The demo databases will be re-seeded automatically.
