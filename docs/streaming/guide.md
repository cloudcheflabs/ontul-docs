# Streaming Guide

Ontul streaming uses **Flink-style continuous processing** — events are processed as soon as they arrive (100ms poll), NOT Spark-style micro-batch. Streaming jobs consume from Kafka, apply transformations, and write to sinks (Iceberg, Kafka, NeorunBase, etc.) with exactly-once semantics for transactional sinks.

## Submitting Streaming Jobs

### 1. REST API

```bash
curl -X POST http://localhost:8080/v1/api/job/submit \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "kafka-to-iceberg",
    "type": "STREAMING",
    "config": {
      "source.kafka.bootstrap.servers": "kafka:9092",
      "source.kafka.topic": "user-events",
      "source.kafka.group.id": "ontul-stream-1",
      "source.kafka.auto.offset.reset": "earliest",
      "source.kafka.format": "json",
      "sink.type": "table",
      "sink.table": "ice.analytics.events"
    },
    "operations": [
      {"type": "FILTER", "value": "event_type <> '\''logout'\''"},
      {"type": "SELECT", "value": "event_id, event_type, user_name, amount, event_time"}
    ]
  }'
```

Response:

```json
{"jobId": "streaming-1714300000000", "status": "SUBMITTED"}
```

#### REST API Config Fields

| Field | Description |
|-------|-------------|
| `source.kafka.bootstrap.servers` | Kafka broker addresses |
| `source.kafka.topic` | Topic to consume |
| `source.kafka.group.id` | Consumer group ID |
| `source.kafka.auto.offset.reset` | `earliest` or `latest` |
| `source.kafka.format` | `json` (auto-infer schema) |
| `sink.type` | `table`, `kafka`, `console` |
| `sink.table` | Target table (for type=table) |
| `sink.kafka.topic` | Target topic (for type=kafka) |

#### REST API Operations

```json
"operations": [
  {"type": "FILTER", "value": "amount > 0"},
  {"type": "SELECT", "value": "event_id, event_type, amount"},
  {"type": "WINDOW", "value": "TUMBLING(SIZE 30 SECONDS)"},
  {"type": "GROUP_BY", "value": "event_type"},
  {"type": "AGG", "value": "SUM(amount) AS total, COUNT(*) AS cnt"}
]
```

### 2. Java SDK (StreamDataFrame API)

```java
OntulSession session = OntulSession.builder()
    .master("localhost", 47470)
    .token("your-jwt-token")
    .build();

String jobId = session.streamSource(
        Source.kafka("inline", "user-events")
            .property("bootstrap.servers", "kafka:9092")
            .groupId("ontul-stream-1")
            .property("auto.offset.reset", "earliest")
            .format("json"))
    .filter("event_type <> 'logout'")
    .select("event_id", "event_type", "user_name", "amount", "event_time")
    .sink(Sink.table("ice.analytics.stream_events"))
    .commitInterval(5_000)
    .start(Duration.ofMinutes(30));

System.out.println("Streaming job started: " + jobId);
```

#### Windowed Aggregation

```java
session.streamSource(
        Source.kafka("inline", "events")
            .property("bootstrap.servers", "kafka:9092")
            .groupId("agg-group"))
    .filter("amount > 0")
    .window(StreamDataFrame.WindowSpec.tumbling(Duration.ofSeconds(30)), "event_time")
    .groupBy("category")
    .agg("SUM(amount) AS total_revenue", "COUNT(*) AS event_count")
    .sink(Sink.table("ice.analytics.windowed_summary"))
    .commitInterval(10_000)
    .start();  // runs indefinitely until killed
```

#### Control Worker Count

```java
// Use 2 workers (each gets some Kafka partitions)
.workers(2)
.start(Duration.ofMinutes(60));
```

### 3. Python SDK

```python
from ontul.session import OntulSession

session = OntulSession()

job_id = (session.stream_source("user-events",
              bootstrap_servers="kafka:9092",
              format="json",
              auto_offset_reset="earliest")
    .filter("event_type <> 'logout'")
    .select("event_id", "event_type", "user_name", "amount", "event_time")
    .sink("ice.analytics.stream_events")
    .commit_interval(5000)
    .start(duration_sec=1800))

print(f"Streaming job started: {job_id}")
```

### 4. Shell Script

`bin/submit.sh` is designed for CLASS and PYTHON jobs. Streaming jobs are submitted via REST API or SDK.

---

## Kafka Source Formats

| Format | Description | Schema |
|--------|-------------|--------|
| **JSON** | JSON messages, schema auto-inferred from first message | Auto-infer from first record |
| **AVRO** | Avro binary with Schema Registry | Schema Registry URL required |
| **RAW** | No parsing — raw key/value/metadata columns | Fixed: `topic`, `partition`, `offset`, `timestamp`, `key`, `value` |

```java
// JSON (default)
Source.kafka("inline", "events").format("json")

// AVRO with Schema Registry
Source.kafka("inline", "events").format("avro")
    .property("schema.registry.url", "http://schema-registry:8081")

// RAW (no parsing)
Source.kafka("inline", "events").format("raw")
```

---

## Operations

| Operation | Description | Example |
|-----------|-------------|---------|
| **FILTER** | Row-level predicate | `"amount > 0 AND event_type <> 'logout'"` |
| **SELECT** | Column projection | `"event_id, event_type, amount"` |
| **WINDOW** | Window specification | `"TUMBLING(SIZE 30 SECONDS)"` |
| **GROUP_BY** | Group columns for aggregation | `"event_type"` |
| **AGG** | Aggregation expressions | `"SUM(amount) AS total, COUNT(*) AS cnt"` |

Operations are applied in order: FILTER → SELECT → WINDOW → GROUP_BY + AGG.

---

## Window Types

| Window | Behavior | SDK | REST |
|--------|----------|-----|------|
| **Tumbling** | Fixed size, non-overlapping | `WindowSpec.tumbling(Duration.ofSeconds(30))` | `TUMBLING(SIZE 30 SECONDS)` |
| **Sliding** | Fixed size, overlapping | `WindowSpec.sliding(Duration.ofMinutes(5), Duration.ofMinutes(1))` | `SLIDING(SIZE 5 MINUTES SLIDE 1 MINUTE)` |
| **Session** | Gap-based | `WindowSpec.session(Duration.ofSeconds(30))` | `SESSION(GAP 30 SECONDS)` |

AGG expressions with aliases are auto-generated if not specified:

```sql
SUM(amount)          → sum_amount
COUNT(*)             → count_star
AVG(price)           → avg_price
SUM(amount) AS total → total
```

---

## Multi-Worker Distribution

Streaming jobs distribute across Workers based on Kafka consumer group partition assignment:

- Each Worker receives a subset of Kafka partitions
- Partition count should be >= Worker count for optimal distribution
- If partition count < Worker count, idle Workers receive no data

Example: 4 Kafka partitions, 2 Workers → each Worker gets 2 partitions.

```java
// Default: use all available Workers
.start(Duration.ofMinutes(30));

// Limit to specific Worker count
.workers(2)
.start(Duration.ofMinutes(30));
```

---

## Exactly-Once Semantics

For transactional sinks, Ontul guarantees exactly-once delivery through barrier checkpoint:

```
1. Master triggers CHECKPOINT_TRIGGER → all Workers
2. Each Worker: flush sink → commit sink transaction
3. Snapshot state (Kafka offsets + window state) to Exchange Manager
4. Commit Kafka consumer offsets (AFTER sink commit — ordering matters)
5. Report CHECKPOINT_COMPLETE → Master
6. Master: all Workers acked → checkpoint globally complete
```

If a Worker fails and restarts, it resumes from the last committed checkpoint — no duplicates, no data loss.

| Sink | Transactional | Exactly-Once |
|------|:---:|:---:|
| Iceberg | O | O |
| JDBC | O | O |
| NeorunBase (JDBC mode) | O | O |
| Kafka (transactional=true) | O | O |
| NeorunBase (REST mode) | X | at-least-once |
| Kafka (non-tx) | X | at-least-once |
| Console | X | at-least-once |

Checkpoint interval is configurable:

```java
.commitInterval(10_000)  // 10 seconds (SDK)
```

```properties
# conf/ontul.properties
ontul.streaming.checkpoint.interval.ms=10000
```

---

## Sink Types

7 sink types are supported (from `StreamSinkFactory`):

### Iceberg (table)

```java
// SDK
.sink(Sink.table("ice.analytics.stream_events"))
```

```json
// REST config
"sink": {"type": "table", "table": "ice.analytics.stream_events"}
```

Writes Parquet files, commits via `AppendFiles`. Exactly-once with barrier checkpoint.

### Kafka

```java
// SDK
.sink(Sink.kafka("connectionId", "output-topic").keyField("event_id"))
```

```json
// REST config
"sink": {"type": "kafka", "bootstrapServers": "kafka:9092", "topic": "output-topic", "keyField": "event_id", "transactional": true}
```

With `transactional=true`, uses Kafka Transactions for exactly-once.

### JDBC

```json
"sink": {"type": "jdbc", "jdbcUrl": "jdbc:postgresql://host:5432/db", "tableName": "events", "username": "user", "password": "pass", "batchSize": 500}
```

Transactional — exactly-once with barrier checkpoint.

### NeorunBase

```json
// REST mode (high throughput, at-least-once)
"sink": {"type": "neorunbase", "mode": "rest", "endpoint": "http://neorunbase:8080", "tableName": "events", "username": "admin", "password": "admin", "batchSize": 500}

// JDBC mode (transactional, exactly-once)
"sink": {"type": "neorunbase", "mode": "jdbc", "jdbcUrl": "jdbc:postgresql://neorunbase:5432/db", "tableName": "events"}
```

### Elasticsearch

```json
"sink": {"type": "elasticsearch", "index": "events", "endpoint": "http://es:9200", "idField": "event_id", "bulkSize": 1000}
```

Also accepts `"es"` as type alias. Non-transactional (at-least-once).

### HTTP / Webhook

```json
"sink": {"type": "http", "url": "https://api.example.com/events", "batchSize": 100, "headers": {"Authorization": "Bearer token123"}}
```

Also accepts `"webhook"` as type alias. Non-transactional (at-least-once).

### Console (Debug)

```json
"sink": {"type": "console"}
```

Prints each batch to stdout. Useful for debugging pipelines.

### Exactly-Once Summary

| Sink | Transactional | Exactly-Once |
|------|:---:|:---:|
| Iceberg (table) | O | O |
| JDBC | O | O |
| Kafka (transactional=true) | O | O |
| NeorunBase (JDBC mode) | O | O |
| NeorunBase (REST mode) | X | at-least-once |
| Kafka (non-tx) | X | at-least-once |
| Elasticsearch | X | at-least-once |
| HTTP / Webhook | X | at-least-once |
| Console | X | at-least-once |

---

## Job Management

### Get Status

```bash
curl http://localhost:8080/v1/api/job/status/{jobId} \
  -H "Authorization: Bearer $TOKEN"
```

### Stream Logs

```bash
curl "http://localhost:8080/v1/api/job/{jobId}/log?from=0" \
  -H "Authorization: Bearer $TOKEN"
```

### Kill a Running Job

Streaming jobs run indefinitely (or until duration expires). Kill manually:

```bash
curl -X POST http://localhost:8080/v1/api/job/kill/{jobId} \
  -H "Authorization: Bearer $TOKEN"
```

Java SDK:

```java
JobHandle job = ...; // from submit
job.kill();
```

---

## Complete Example: Kafka → Iceberg

### Setup

```bash
cd ontul-1.0.0-SNAPSHOT

# Start MinIO + Polaris + Kafka (4 partitions for user-events topic)
docker compose -f examples/docker-compose-iceberg.yml up -d

# Start Ontul cluster + register catalogs
examples/bin/setup-iceberg.sh
```

### Run

```bash
examples/bin/run-iceberg-streaming.sh
```

### What Happens

1. **Produce 100 JSON events** to Kafka topic `user-events`:

```json
{"event_id":1,"event_type":"click","user_name":"diana","amount":68.32,"event_time":1712345678100}
{"event_id":2,"event_type":"click","user_name":"alice","amount":27.71,"event_time":1712345678200}
{"event_id":3,"event_type":"logout","user_name":"eve","amount":8.69,"event_time":1712345678300}
```

2. **Submit streaming job** (Kafka → filter → select → Iceberg):

```java
session.streamSource(
        Source.kafka("inline", "user-events")
            .format("json")
            .groupId("ontul-stream-" + System.currentTimeMillis())
            .property("bootstrap.servers", "localhost:9092")
            .property("auto.offset.reset", "earliest"))
    .filter("event_type <> 'logout'")
    .select("event_id", "event_type", "user_name", "amount", "event_time")
    .sink(Sink.table("ice.examples.stream_events"))
    .commitInterval(5_000)
    .start(Duration.ofSeconds(20));
```

3. **Verify ingested data** — query Iceberg:

```sql
SELECT count(*) AS total_events FROM ice.examples.stream_events
```

```
total_events
-----------
78
```

100 events produced, ~22 `logout` events filtered = 78 events ingested.

```sql
SELECT event_type, COUNT(*) AS cnt,
       CAST(SUM(amount) AS DECIMAL(12,2)) AS total_amount
FROM ice.examples.stream_events
GROUP BY event_type ORDER BY cnt DESC
```

```
event_type | cnt | total_amount
-----------+-----+-------------
login      |  21 |      1088.72
view       |  20 |       961.83
click      |  19 |       966.52
purchase   |  18 |      1052.12
```

No `logout` — filter works correctly.

```sql
SELECT user_name, COUNT(*) AS events,
       CAST(AVG(amount) AS DECIMAL(10,2)) AS avg_amount
FROM ice.examples.stream_events
GROUP BY user_name ORDER BY events DESC
```

```
user_name | events | avg_amount
----------+--------+-----------
bob       |     19 |      58.44
alice     |     16 |      48.52
charlie   |     15 |      43.06
diana     |     14 |      53.41
eve       |     14 |      56.34
```

### Python Version

```python
from ontul.session import OntulSession
import time

session = OntulSession()

job_id = (session.stream_source("user-events",
              bootstrap_servers="localhost:9092",
              format="json",
              auto_offset_reset="earliest")
    .filter("event_type <> 'logout'")
    .select("event_id", "event_type", "user_name", "amount", "event_time")
    .sink("ice.examples.stream_events_py")
    .commit_interval(5000)
    .start(duration_sec=20))

time.sleep(45)

df = session.sql_pandas("SELECT count(*) AS total FROM ice.examples.stream_events_py")
print(df)
```

---

## Windowed Aggregation Example

```java
// Kafka → 30s tumbling window → category revenue → Iceberg
session.streamSource(
        Source.kafka("inline", "order-events")
            .property("bootstrap.servers", "kafka:9092")
            .groupId("window-group")
            .property("auto.offset.reset", "earliest")
            .format("json"))
    .filter("amount > 0")
    .window(StreamDataFrame.WindowSpec.tumbling(Duration.ofSeconds(30)), "event_time")
    .groupBy("category")
    .agg("SUM(amount) AS total_revenue", "COUNT(*) AS order_count",
         "CAST(AVG(amount) AS DECIMAL(10,2)) AS avg_amount")
    .sink(Sink.table("ice.analytics.category_revenue_30s"))
    .commitInterval(10_000)
    .start(Duration.ofHours(1));
```

---

## Exchange Manager

All checkpoint state is stored via the Exchange Manager with KMS envelope encryption:

- **State snapshot**: Kafka consumer offsets + window aggregation state
- **Encryption**: AES-256-GCM with KMS-managed data keys
- **Storage**: `${ontul.exchange.base.dir}/checkpoint/`
- **Pruning**: Old checkpoints pruned after global completion

---

## Cleanup

```bash
bin/stop-example-servers.sh
docker compose -f examples/docker-compose-iceberg.yml down -v
```
