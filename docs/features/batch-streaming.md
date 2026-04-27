# Batch & Streaming Processing

Ontul supports distributed batch and streaming data processing through its SDK and REST API. ETL pipelines, data transformations, and continuous streaming jobs all run within the same cluster used for interactive SQL.

## Job Types

| Type | Description | Driver | Lifetime |
|------|-------------|--------|----------|
| **BATCH** | Process bounded dataset (SQL-based) | Master or Worker | Completes when done |
| **STREAMING** | Continuous Kafka consume → transform → sink | Worker (always) | Runs until stopped or duration expires |
| **CLASS** | Custom Java class with dependencies | Worker | Completes when done |
| **PYTHON** | Python script execution | Worker | Completes when done |

---

## Job Lifecycle

```
SUBMITTED → PLANNING → RUNNING → COMPLETED
                                → FAILED
                                → KILLED
```

- **SUBMITTED**: Job accepted by Master, queued for execution
- **PLANNING**: Master analyzing query plan, determining driver (Master vs Worker)
- **RUNNING**: Executing on assigned Workers
- **COMPLETED**: Finished successfully
- **FAILED**: Error during execution
- **KILLED**: Stopped by user via REST API or Admin UI

---

## Submitting Jobs

### 1. REST API

#### Batch Job (SQL)

```bash
curl -X POST http://localhost:8080/v1/api/job/submit \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "daily-etl",
    "type": "BATCH",
    "sql": "CREATE TABLE ice.warehouse.daily_summary AS SELECT category, SUM(amount) AS total FROM ice.raw.transactions WHERE dt = CURRENT_DATE GROUP BY category"
  }'
```

Response:
```json
{"jobId": "batch-1714300000000", "status": "SUBMITTED"}
```

#### Streaming Job

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

#### CLASS Job (Custom Java)

```bash
# 1. Upload dependency JARs
curl -X POST http://localhost:8080/v1/api/deps \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-File-Name: my-etl-job.jar" \
  -H "Content-Type: application/octet-stream" \
  --data-binary @my-etl-job.jar

# Response: {"path": "deps/my-etl-job.jar"}

# 2. Submit CLASS job
curl -X POST http://localhost:8080/v1/api/job/submit \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "custom-etl",
    "type": "CLASS",
    "className": "com.example.MyEtlJob",
    "deps": ["deps/my-etl-job.jar"],
    "args": {
      "input": "s3://bucket/raw/data",
      "output": "ice.warehouse.processed"
    }
  }'
```

#### PYTHON Job

```bash
# 1. Upload Python script
curl -X POST http://localhost:8080/v1/api/deps \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-File-Name: etl_pipeline.py" \
  -H "Content-Type: application/octet-stream" \
  --data-binary @etl_pipeline.py

# 2. Submit PYTHON job
curl -X POST http://localhost:8080/v1/api/job/submit \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "python-etl",
    "type": "PYTHON",
    "scriptPath": "deps/etl_pipeline.py",
    "args": {
      "input_table": "ice.raw.events",
      "output_table": "ice.analytics.summary"
    }
  }'
```

### 2. Shell Script (`bin/submit.sh`)

Ontul ships with `bin/submit.sh` for submitting CLASS and PYTHON jobs from the command line. The script handles dependency diffing, upload, job submission, and log streaming automatically.

#### CLASS Job

```bash
bin/submit.sh \
  --master localhost:8080 \
  --class com.example.job.MyEtlJob \
  --jar-dirs /path/to/project/build/libs,/path/to/project/build/deps \
  --token $ONTUL_USER_TOKEN \
  --args "input=s3://bucket/raw,output=ice.warehouse.processed"
```

Or specify individual JARs:

```bash
bin/submit.sh \
  --master localhost:8080 \
  --class com.example.job.ImageProcessingJob \
  --jars /path/to/job.jar,/path/to/dep1.jar,/path/to/dep2.jar \
  --token $ONTUL_USER_TOKEN
```

#### PYTHON Job

```bash
bin/submit.sh \
  --master localhost:8080 \
  --python /path/to/etl_pipeline.py \
  --pydeps /path/to/requirements.txt \
  --token $ONTUL_USER_TOKEN \
  --args "input_table=ice.raw.events,output_table=ice.analytics.summary"
```

#### Options

| Option | Description |
|--------|-------------|
| `--master host:port` | Master admin endpoint (required) |
| `--class className` | Java class to execute (CLASS job) |
| `--python script.py` | Python script to execute (PYTHON job) |
| `--jars jar1,jar2,...` | Comma-separated JAR files |
| `--jar-dirs dir1,dir2,...` | Comma-separated directories (all `*.jar` collected) |
| `--pydeps requirements.txt` | Python requirements file |
| `--token TOKEN` | JWT token (or env `ONTUL_USER_TOKEN`) |
| `--accesskey AK` | AccessKey auth (or env `ONTUL_USER_ACCESS_KEY`) |
| `--secretkey SK` | SecretKey auth (or env `ONTUL_USER_SECRET_KEY`) |
| `--args "k1=v1,k2=v2"` | Job arguments as key=value pairs |
| `--no-wait` | Submit and exit without waiting for completion |

#### What the Script Does

```
Step 1: Compare client JARs with server deps (GET /v1/api/deps)
        → SKIP JARs already on server
Step 2: Upload missing JARs (POST /v1/api/deps)
Step 3: Submit job (POST /v1/api/job/submit)
Step 4: Poll status + stream logs until COMPLETED / FAILED / KILLED
```

Example output:

```
=== Ontul Job Submit ===
Master:  http://localhost:8080
Class:   com.example.job.MyEtlJob
JARs:    5 files

Step 1: Comparing deps with server...
  SKIP (on server): guava-33.4.0-jre.jar
  SKIP (on server): jackson-core-2.18.2.jar
  Missing: 2 JARs to upload

Step 2: Uploading missing JARs...
  Uploaded: my-etl-job.jar (245760 bytes) → deps/my-etl-job.jar
  Uploaded: custom-lib.jar (102400 bytes) → deps/custom-lib.jar

Step 3: Submitting CLASS job...
  Job ID: class-1714300000000

Step 4: Waiting for completion...
  2026-04-28 10:00:01 INFO  Starting MyEtlJob
  2026-04-28 10:00:05 INFO  Reading from s3://bucket/raw
  2026-04-28 10:00:30 INFO  Wrote 50000 rows to ice.warehouse.processed
  2026-04-28 10:00:30 INFO  Job completed

=== Job COMPLETED (ID: class-1714300000000) ===
```

### 3. Java SDK

#### Batch (DataFrame API)

```java
OntulSession session = OntulSession.builder()
    .master("localhost", 47470)
    .token("your-jwt-token")
    .build();

// Read → Transform → Write (runs on cluster)
session.source(Source.s3("s3://bucket/raw/data", "parquet")
        .property("endpoint", "http://minio:9000")
        .property("accessKey", "minioadmin")
        .property("secretKey", "minioadmin"))
    .filter("quantity > 0 AND unit_price > 0")
    .withColumn("total_amount", "quantity * unit_price")
    .sink(Sink.table("ice.warehouse.enriched_sales"));
```

#### Streaming (StreamDataFrame API)

```java
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

#### Windowed Streaming

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
    .start();
```

Window types:

| Window | Description | Example |
|--------|-------------|---------|
| Tumbling | Fixed size, non-overlapping | `WindowSpec.tumbling(Duration.ofSeconds(30))` |
| Sliding | Fixed size, overlapping | `WindowSpec.sliding(Duration.ofMinutes(5), Duration.ofMinutes(1))` |
| Session | Gap-based | `WindowSpec.session(Duration.ofSeconds(30))` |

#### CLASS Job (Custom Java Class)

Write a custom job class and submit it to the cluster. The SDK automatically diffs client JARs against server, uploads missing dependencies, and submits the job.

```java
// Submit custom job class — deps auto-uploaded
JobHandle job = session.submit(MyEtlJob.class, Map.of(
    "input", "s3://bucket/raw/data",
    "output", "ice.warehouse.processed"
));

// Wait for completion (stream logs to console)
String finalStatus = job.waitForCompletion(300); // 300s timeout
System.out.println("Job finished: " + finalStatus);

// Or poll manually
while (true) {
    String status = job.status();
    if ("COMPLETED".equals(status) || "FAILED".equals(status)) break;
    Thread.sleep(5000);
}

// Get logs
job.logs().forEach(System.out::println);

// Kill if needed
job.kill();
```

### 4. Python SDK

#### Batch

```python
from ontul.session import OntulSession

session = OntulSession(host="localhost", port=47470, token="your-jwt-token")

# SQL batch
session.execute(
    "CREATE TABLE ice.warehouse.top_customers AS "
    "SELECT c_custkey, c_name, c_acctbal "
    "FROM tpch.sf1.customer WHERE c_acctbal > 5000"
)

# Read from S3 → Write to Iceberg
raw = session.source_s3("s3://bucket/raw/sales", "parquet", s3_config)
session.sink(raw, "ice.staging.sales")
```

#### Streaming

```python
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

---

## Job Management

### Get Job Status

```bash
curl http://localhost:8080/v1/api/job/status/{jobId} \
  -H "Authorization: Bearer $TOKEN"
```

Response:
```json
{
  "jobId": "batch-1714300000000",
  "name": "daily-etl",
  "type": "BATCH",
  "status": "RUNNING",
  "username": "admin",
  "startTime": 1714300000000,
  "assignedWorkerId": "worker-1"
}
```

### Get Job Logs

```bash
# All logs
curl http://localhost:8080/v1/api/job/{jobId}/log \
  -H "Authorization: Bearer $TOKEN"

# From offset (for incremental polling)
curl "http://localhost:8080/v1/api/job/{jobId}/log?from=100" \
  -H "Authorization: Bearer $TOKEN"
```

Response:
```json
{
  "lines": [
    "2026-04-28 10:00:01 INFO  Starting batch job daily-etl",
    "2026-04-28 10:00:02 INFO  Reading from ice.raw.transactions",
    "2026-04-28 10:00:15 INFO  Wrote 50000 rows to ice.warehouse.daily_summary",
    "2026-04-28 10:00:15 INFO  Job completed successfully"
  ]
}
```

### Kill a Running Job

```bash
curl -X POST http://localhost:8080/v1/api/job/kill/{jobId} \
  -H "Authorization: Bearer $TOKEN"
```

Response:
```json
{"killed": true, "jobId": "streaming-1714300000000"}
```

### List Jobs

```bash
# All active jobs
curl http://localhost:8080/admin/jobs \
  -H "Authorization: Bearer $TOKEN"

# Job history
curl http://localhost:8080/admin/jobs/history \
  -H "Authorization: Bearer $TOKEN"
```

---

## Streaming Details

### Flink-Style Continuous Processing

Ontul streaming uses Flink-style continuous processing, NOT Spark-style micro-batch:

- Events processed as soon as they arrive (poll interval: 100ms)
- No batch boundaries — true event-at-a-time processing
- Lower latency than micro-batch approaches

### Multi-Worker Distribution

Streaming jobs distribute across Workers based on Kafka partition assignment:

- Each Worker receives a subset of Kafka partitions via the consumer group
- `workers(n)` in the SDK controls how many Workers are used (default: all)
- Partition count should be >= Worker count for optimal distribution

```java
// Use 2 workers (each gets some partitions)
session.streamSource(Source.kafka("inline", "events")
        .property("bootstrap.servers", "kafka:9092")
        .groupId("my-group"))
    .sink(Sink.table("ice.analytics.events"))
    .workers(2)  // optional: limit worker count
    .start();
```

### Exactly-Once Semantics

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
| NeorunBase (JDBC) | O | O |
| Kafka (transactional=true) | O | O |
| NeorunBase (REST) | X | at-least-once |
| Kafka (non-tx) | X | at-least-once |
| Console | X | at-least-once |

### Streaming Operations

| Operation | Description | Example |
|-----------|-------------|---------|
| FILTER | Row-level predicate | `"amount > 0"` |
| SELECT | Column projection | `"event_id, event_type, amount"` |
| GROUP_BY | Group columns for aggregation | `"category"` |
| WINDOW | Window specification | `TUMBLING(SIZE 30 SECONDS)` |
| AGG | Aggregation expressions | `"SUM(amount) AS total, COUNT(*) AS cnt"` |

---

## Driver Delegation

Ontul automatically determines where to run the driver for each job:

| Job Type | Driver Location | Reason |
|----------|----------------|--------|
| Small batch (< 100K rows) | Master | Fast, no dispatch overhead |
| Large batch | Worker | Offload heavy computation |
| Streaming | Worker (always) | Long-running, dedicated resources |
| CLASS | Worker | Custom code isolation |
| PYTHON | Worker | Python subprocess management |

This keeps the Master responsive for query planning and administrative operations while offloading heavy computation to Workers.

---

## SDK Reference

- [Java SDK](../reference/sdk-java.md) — Full Java API documentation
- [Python SDK](../reference/sdk-python.md) — Full Python API documentation
- [REST API](../reference/rest-api.md) — REST API reference
- [Remote Mode](../features/remote-mode.md) — DataFrame API for pipelines
