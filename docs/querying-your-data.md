# Querying Your Data

Get the current state of a table

```
SELECT *
FROM cdc.users
FINAL
WHERE _cdc_deleted = 0
ORDER BY id;
```

Get the latest value of each row without FINAL

```
SELECT
    id,
    argMax(email, _cdc_version)      AS email,
    argMax(full_name, _cdc_version)  AS full_name,
    argMax(balance, _cdc_version)    AS balance,
    argMax(_cdc_deleted, _cdc_version) AS is_deleted
FROM cdc.users
GROUP BY id
HAVING is_deleted = 0;
```

View the full change history of a record

```
SELECT _cdc_op, _cdc_ts, id, email, balance
FROM cdc.users
WHERE id = 42
ORDER BY _cdc_ts ASC;
```

Find all rows deleted in the last 24 hours

```
SELECT id, email, _cdc_ts AS deleted_at
FROM cdc.users
FINAL
WHERE _cdc_deleted = 1
  AND _cdc_ts >= now() - INTERVAL 1 DAY
ORDER BY deleted_at DESC;
```

Count operations by type

```
SELECT _cdc_op, count() AS total
FROM cdc.users
GROUP BY _cdc_op
ORDER BY total DESC;
```

Measure replication latency

```
SELECT
    table AS source_table,
    max(_cdc_ts)                         AS last_event_time,
    dateDiff('second', max(_cdc_ts), now()) AS seconds_behind
FROM cdc.users
GROUP BY table;
```

Compare row counts between source and ClickHouse

```
# Count in PostgreSQL
docker exec -it bb-postgres psql -U bbuser -d sourcedb -c \
  "SELECT count(*) FROM users;"

# Count in ClickHouse
curl "http://localhost:8123/?user=bbuser&password=bbpass" \
  --data "SELECT count() FROM cdc.users FINAL WHERE _cdc_deleted = 0"
```

Query changes in a time window

```
SELECT _cdc_op, _cdc_ts, id, email
FROM cdc.users
WHERE _cdc_ts BETWEEN '2026-01-01 00:00:00' AND '2026-01-02 00:00:00'
ORDER BY _cdc_ts;
```
