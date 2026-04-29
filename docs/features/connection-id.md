# Connection ID (Source / Sink by Reference)

Ontul stores physical connection details (endpoint, credentials, properties) once in the encrypted **ConnectionStore**, then lets jobs and catalogs reference them by a short, stable **connection ID**. This keeps secrets out of code, configuration files, and job payloads — sources and sinks just point at a registered connection.

The same connection ID works across all three execution paths: SQL/CTAS catalog access, batch DataFrame jobs, and streaming jobs.

## Why Connection IDs

| Inline credentials (anti-pattern) | Connection ID (recommended) |
|------------------------------------|------------------------------|
| Secrets embedded in job code, SQL, or REST payloads | Secrets stored once, encrypted at rest via KMS |
| Rotating a key requires editing every job | Rotate in one place; all jobs pick up the new value |
| Credentials visible in job logs, history, and audit trails | Only the connection ID appears in payloads/logs |
| Each environment (dev/stage/prod) needs its own copy of every job | Same job + per-environment connection IDs |

## Lifecycle

```
1. Register a connection  →  POST /admin/connections   (returns id)
2. Use the id from a source/sink  →  Source.s3(...).connection(id) / Sink.kafka(id, topic)
3. Rotate or update                →  PUT /admin/connections/{id}
4. Retire                          →  DELETE /admin/connections/{id}
```

Credentials are sealed with a KMS data key (envelope encryption) before being persisted. Workers decrypt on demand when a job opens the connection — they never receive raw credentials over the wire.

## Registering a Connection

### S3

```bash
curl -X POST http://localhost:8080/admin/connections \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "id": "warehouse-s3",
    "type": "s3",
    "properties": {
      "endpoint": "http://minio:9000",
      "accessKey": "ACCESS_KEY",
      "secretKey": "SECRET_KEY",
      "region": "us-east-1",
      "pathStyle": "true"
    }
  }'
```

### JDBC

```bash
curl -X POST http://localhost:8080/admin/connections \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "id": "analytics-pg",
    "type": "jdbc",
    "properties": {
      "url": "jdbc:postgresql://pg:5432/analytics",
      "username": "etl",
      "password": "secret"
    }
  }'
```

### Kafka

```bash
curl -X POST http://localhost:8080/admin/connections \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "id": "events-kafka",
    "type": "kafka",
    "properties": {
      "bootstrap.servers": "kafka:9092",
      "security.protocol": "SASL_SSL",
      "sasl.mechanism": "PLAIN",
      "sasl.jaas.config": "org.apache.kafka.common.security.plain.PlainLoginModule required username=\"u\" password=\"p\";"
    }
  }'
```

### Connection Types

| Type | Required properties | Used by |
|------|---------------------|---------|
| `s3` | `endpoint`, `accessKey`, `secretKey`, `region`, `pathStyle` | S3 source/sink, Iceberg storage |
| `jdbc` | `url`, `username`, `password` | JDBC source/sink, NeorunBase JDBC mode |
| `kafka` | `bootstrap.servers` (+ optional security props) | Kafka source/sink |
| `neorunbase` | `endpoint`, `jdbcUrl`, `username`, `password` | NeorunBase catalog + sink |
| `http` | `url` (+ optional `headers`) | HTTP / Webhook sink |
| `elasticsearch` | `endpoint` (+ optional auth) | Elasticsearch sink |

## Using Connection IDs in Sources and Sinks

### Java SDK

```java
// S3 source by connection ID — no inline credentials
session.source(Source.s3("s3://warehouse/raw/sales", "parquet")
        .connection("warehouse-s3"))
    .filter("quantity > 0")
    .withColumn("total", "quantity * unit_price")
    .sink(Sink.s3("s3://warehouse/curated/sales", "parquet")
        .connection("warehouse-s3"));

// JDBC source/sink
session.source(Source.jdbc("analytics-pg", "SELECT * FROM public.orders WHERE dt = CURRENT_DATE"))
    .sink(Sink.table("ice.staging.orders"));

session.source(Source.sql("SELECT * FROM ice.warehouse.daily_summary"))
    .sink(Sink.jdbc("analytics-pg", "public.daily_summary"));

// Kafka streaming source by connection ID
session.streamSource(
        Source.kafka("events-kafka", "user-events")
            .groupId("ontul-stream-1")
            .property("auto.offset.reset", "earliest")
            .format("json"))
    .filter("event_type <> 'logout'")
    .sink(Sink.kafka("events-kafka", "filtered-events").keyField("event_id"))
    .commitInterval(5_000)
    .start(Duration.ofMinutes(30));
```

The first argument to `Source.kafka(...)` / `Sink.kafka(...)` is the **connection ID**. Pass the literal string `"inline"` only when supplying credentials directly via `.property(...)` — production jobs should always use a registered ID.

### Python SDK

```python
# S3 source/sink with connection ID
(session.source_s3("s3://warehouse/raw/sales", "parquet", connection="warehouse-s3")
    .filter("quantity > 0")
    .with_column("total", "quantity * unit_price")
    .sink_s3("s3://warehouse/curated/sales", "parquet", connection="warehouse-s3"))

# Kafka streaming source by connection ID
job_id = (session.stream_source("user-events",
              connection="events-kafka",
              format="json",
              auto_offset_reset="earliest")
    .filter("event_type <> 'logout'")
    .sink_kafka("filtered-events", connection="events-kafka", key_field="event_id")
    .commit_interval(5000)
    .start(duration_sec=1800))
```

### REST API (Streaming Job)

Reference connections through `source.connectionId` / `sink.connectionId` instead of repeating credentials in the job config:

```json
{
  "name": "kafka-to-iceberg",
  "type": "STREAMING",
  "config": {
    "source.type": "kafka",
    "source.connectionId": "events-kafka",
    "source.kafka.topic": "user-events",
    "source.kafka.group.id": "ontul-stream-1",
    "source.kafka.auto.offset.reset": "earliest",
    "source.kafka.format": "json",

    "sink.type": "kafka",
    "sink.connectionId": "events-kafka",
    "sink.kafka.topic": "filtered-events",
    "sink.kafka.transactional": "true"
  },
  "operations": [
    {"type": "FILTER", "value": "event_type <> 'logout'"}
  ]
}
```

Equivalent for JDBC and S3 sinks:

```json
{
  "sink": {"type": "jdbc", "connectionId": "analytics-pg", "tableName": "events", "batchSize": 500}
}
```

```json
{
  "sink": {"type": "s3", "connectionId": "warehouse-s3", "path": "s3://warehouse/output", "format": "parquet"}
}
```

When a `connectionId` is present, any per-property override (`source.kafka.bootstrap.servers`, `accessKey`, `password`, …) is **merged on top** of the stored properties. This is the recommended way to override a single value (e.g., a per-job consumer group) without restating credentials.

## Catalogs Reference Connections by ID

Catalogs use the same mechanism — the catalog config carries the connection ID, not raw credentials:

```json
POST /admin/catalogs
{
  "name": "neorun",
  "config": {
    "connector": "neorunbase",
    "connectionId": "neorunbase-prod"
  }
}
```

```json
POST /admin/catalogs
{
  "name": "ice",
  "config": {
    "connector": "iceberg",
    "catalog-type": "rest",
    "uri": "http://polaris:8181/api/catalog",
    "warehouse": "my_catalog",
    "s3.connectionId": "warehouse-s3"
  }
}
```

Once registered, queries use fully qualified names (`ice.db.events`, `neorun.public.orders`) — the engine resolves the connection ID and decrypts credentials transparently.

## Overriding Stored Properties

Inline `.property(...)` values **win** over stored connection properties when both are supplied. Use this to override a single field per job (consumer group, fetch size, query timeout) without forking the connection:

```java
Source.kafka("events-kafka", "user-events")
    .groupId("backfill-2026-04-30")            // override stored group.id
    .property("max.poll.records", "5000");      // override stored fetch tuning
```

## Auditing Connections

```bash
# List all connections (no credentials returned)
GET /admin/connections

# Test a connection without exposing secrets
POST /admin/connections/test
{"id": "warehouse-s3"}
```

The list response returns IDs, types, and non-secret properties only — encrypted fields (`secretKey`, `password`, `sasl.jaas.config`, …) are masked.

## Best Practices

- **One connection per logical system, per environment.** Name them `<role>-<system>`: `warehouse-s3`, `events-kafka`, `analytics-pg`. Avoid `prod-2` and other mystery IDs.
- **Never embed credentials in SQL, job classes, REST payloads, or `conf/ontul.properties`.** If you find yourself writing `accessKey` into a job, register a connection instead.
- **Rotate at the connection level.** `PUT /admin/connections/{id}` updates all jobs, catalogs, and streaming pipelines that reference it on the next open.
- **Use IAM to gate connection management.** `CONNECTION:CREATE`, `CONNECTION:UPDATE`, and `CONNECTION:DELETE` are admin-only by default; grant `CONNECTION:READ` to operators that only need to list IDs.
