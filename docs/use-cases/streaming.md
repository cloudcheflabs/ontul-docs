# Streaming: Kafka to Iceberg

This tutorial demonstrates real-time streaming ingestion from Kafka to Iceberg tables using Ontul's streaming pipeline.

**What you will learn:**

- Produce JSON events to Kafka
- Submit a streaming job: Kafka consume, filter, select, and write to Iceberg
- Verify ingested data with interactive SQL queries
- Use the StreamDataFrame API for declarative streaming pipelines

## Prerequisites

- Java 17
- Docker (for MinIO + Polaris + Kafka)

## 1. Setup

Follow the [Installation Guide](../installation/installation.md) to download and set up Ontul, then:

```bash
cd ontul-1.0.0-SNAPSHOT

# Start MinIO + Polaris + Kafka
docker compose -f examples/docker-compose-iceberg.yml up -d

# Start Ontul cluster + register catalogs
examples/bin/setup-iceberg.sh
```

The docker-compose starts Kafka (KRaft mode, no ZooKeeper) with the `user-events` topic pre-created.

## 2. Run the Streaming Example

```bash
examples/bin/run-iceberg-streaming.sh
```

---

## How It Works

The streaming pipeline:

```
Kafka (user-events)
  → JSON parsing (auto-infer schema)
  → Filter: event_type <> 'logout'
  → Select: event_id, event_type, user_name, amount, event_time
  → Iceberg (ice.examples.stream_events)
```

### Step 1: Produce Events to Kafka

100 JSON events are produced to the `user-events` topic:

```json
{"event_id":1,"event_type":"click","user_name":"alice","amount":74.16,"event_time":1712345678100}
{"event_id":2,"event_type":"view","user_name":"bob","amount":13.95,"event_time":1712345678200}
{"event_id":3,"event_type":"logout","user_name":"eve","amount":8.69,"event_time":1712345678300}
...
```

Event types: `click`, `view`, `purchase`, `login`, `logout`

### Step 2: Submit Streaming Job

The job is submitted using Ontul's StreamDataFrame API. The pipeline declaration is:

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

**Key behaviors:**

- **JSON auto-inference**: Schema is automatically inferred from the first JSON message
- **Filter**: `logout` events are excluded before writing to Iceberg
- **Select**: Only the specified columns are written (not raw Kafka metadata)
- **Commit interval**: Iceberg commits happen every 5 seconds
- **Kafka offset commit**: Offsets are committed only after Iceberg data is safely persisted (no data loss on failure)
- **Duration**: The job runs for 20 seconds, then stops

### Step 3: Verify Ingested Data

After the streaming job completes, query the Iceberg table:

**Total events (logout excluded):**

```sql
SELECT count(*) AS total_events FROM ice.examples.stream_events
```

```
total_events
-----------
78
```

100 events produced, ~22 `logout` events filtered out = ~78 events ingested.

**Event type distribution:**

```sql
SELECT event_type, COUNT(*) AS cnt,
       CAST(SUM(amount) AS DECIMAL(12,2)) AS total_amount
FROM ice.examples.stream_events
GROUP BY event_type
ORDER BY cnt DESC
```

```
event_type | cnt | total_amount
-----------+-----+-------------
click      |  22 |       965.98
view       |  18 |       886.62
login      |  18 |       915.53
purchase   |  16 |       722.16
```

Note: `logout` is not present — the filter works correctly.

**User activity:**

```sql
SELECT user_name, COUNT(*) AS events,
       CAST(AVG(amount) AS DECIMAL(10,2)) AS avg_amount
FROM ice.examples.stream_events
GROUP BY user_name
ORDER BY events DESC
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

**Sample rows:**

```sql
SELECT event_id, event_type, user_name, amount
FROM ice.examples.stream_events
ORDER BY event_id
LIMIT 10
```

```
event_id | event_type | user_name | amount
---------+------------+-----------+-------
1        | click      | diana     |  68.32
2        | click      | alice     |  27.71
4        | view       | charlie   |  27.57
5        | view       | alice     |  78.29
...
```

---

## Example Code (Java)

The full example is in `examples/java/IcebergStreamingExample.java`.

### Produce Events

Events are produced to Kafka using `docker exec kafka-console-producer`:

```java
private static void produceEvents(int count) throws Exception {
    StringBuilder sb = new StringBuilder();
    for (int i = 1; i <= count; i++) {
        sb.append(String.format(
            "{\"event_id\":%d,\"event_type\":\"%s\",\"user_name\":\"%s\",\"amount\":%.2f,\"event_time\":%d}\n",
            i, randomEventType(), randomUser(), randomAmount(), baseTime + i * 100));
    }

    ProcessBuilder pb = new ProcessBuilder(
        "docker", "exec", "-i", "ontul-example-kafka",
        "/opt/kafka/bin/kafka-console-producer.sh",
        "--bootstrap-server", "localhost:9092",
        "--topic", "user-events");
    Process proc = pb.start();
    proc.getOutputStream().write(sb.toString().getBytes());
    proc.getOutputStream().close();
    proc.waitFor();
}
```

### Verify with DataFrame API

After the streaming job completes, use the DataFrame API to analyze results:

```java
// Event type distribution
var byType = session.source(Source.sql("SELECT * FROM ice.examples.stream_events"))
    .groupBy("event_type")
    .agg("COUNT(*) AS cnt, CAST(SUM(amount) AS DECIMAL(12,2)) AS total_amount")
    .orderBy("cnt", false);

// User activity
var byUser = session.source(Source.sql("SELECT * FROM ice.examples.stream_events"))
    .groupBy("user_name")
    .agg("COUNT(*) AS events, CAST(AVG(amount) AS DECIMAL(10,2)) AS avg_amount")
    .orderBy("events", false);
```

---

## Python Version

The same example is available in Python: `examples/python/iceberg_streaming_example.py`

```bash
examples/bin/run-iceberg-streaming-python.sh
```

```python
from ontul.session import OntulSession
import time

session = OntulSession()

# Submit streaming job using StreamDataFrame API
result = (session.stream_source("user-events",
                bootstrap_servers="localhost:9092",
                format="json",
                auto_offset_reset="earliest")
    .filter("event_type <> 'logout'")
    .select("event_id", "event_type", "user_name", "amount", "event_time")
    .sink("ice.examples.stream_events_py")
    .commit_interval(5000)
    .start(duration_sec=20))

# Wait and verify
time.sleep(45)
table = session.source("SELECT count(*) AS total FROM ice.examples.stream_events_py")
print(table.to_pandas())
```

---

## Architecture: Streaming Pipeline Internals

```
Client (SDK)
  │
  ├── SUBMIT STREAMING {config JSON}
  │       │
  │       ▼
  │   Master (JobManager)
  │       │
  │       ├── Dispatch to Worker(s) via NIO
  │       │       │
  │       │       ▼
  │       │   Worker(s) (StreamingJobExecutor, dedicated thread per job)
  │       │       │
  │       │       ├── KafkaStreamSource (consumer group, multi-worker partition distribution)
  │       │       │       ├── JSON auto-schema inference
  │       │       │       └── Continuous poll (100ms) — Flink-style, NOT micro-batch
  │       │       │
  │       │       ├── Operations pipeline
  │       │       │       ├── FILTER: evaluate predicates
  │       │       │       ├── SELECT: project output columns
  │       │       │       ├── WINDOW: TUMBLING / SLIDING / SESSION
  │       │       │       ├── GROUP_BY + AGG: windowed aggregation (with alias)
  │       │       │       └── Multi-worker hash shuffle (STREAM_SHUFFLE opcode)
  │       │       │
  │       │       ├── StreamSink (pluggable)
  │       │       │       ├── Iceberg: Parquet write → AppendFiles commit
  │       │       │       ├── NeorunBase: REST bulk-insert (high throughput)
  │       │       │       ├── Kafka: producer (transactional mode for exactly-once)
  │       │       │       ├── JDBC / Elasticsearch / HTTP / Console
  │       │       │       └── Sink commit tied to barrier checkpoint (exactly-once)
  │       │       │
  │       │       └── Exchange Manager
  │       │               ├── State snapshot (Kafka offsets + window state)
  │       │               └── KMS envelope encryption
  │       │
  │       ├── CheckpointCoordinator (Master)
  │       │       ├── Periodic CHECKPOINT_TRIGGER → all Workers
  │       │       ├── Collect CHECKPOINT_COMPLETE acks
  │       │       └── Globally complete → prune old snapshots
  │       │
  │       └── Catalog refresh → Calcite schema update
  │
  └── Query Iceberg table to verify results
```

**Key design decisions:**

- **Flink-style continuous processing**: Events are processed as soon as they arrive from Kafka (100ms poll), NOT Spark-style micro-batch
- **Barrier checkpoint**: Master-coordinated distributed checkpoint — sink commit → offset commit → state snapshot (exactly-once for transactional sinks)
- **auto.commit = false**: Kafka offsets are committed only after sink transaction is safely committed
- **JSON schema inference**: The first message's structure determines the Arrow schema
- **Multi-worker shuffle**: Windowed aggregation with GROUP BY distributes records by hash key across Workers
- **Exchange Manager**: All checkpoint state is KMS envelope-encrypted

---

## Cleanup

```bash
bin/stop-example-servers.sh
docker compose -f examples/docker-compose-iceberg.yml down -v
```
