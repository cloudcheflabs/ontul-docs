# Batch & Streaming Processing

Ontul supports distributed batch and streaming data processing through its SDK, enabling ETL pipelines, data transformations, and continuous streaming jobs within the same cluster used for interactive SQL.

## Job Types

### Batch Jobs

Batch jobs process a bounded dataset and complete when all data has been processed. Use cases include ETL pipelines, data migration, periodic aggregation, and data export.

- Small batch jobs run with the Master as the driver
- Large batch jobs are automatically delegated to a Worker as the driver, keeping the Master lightweight

### Streaming Jobs

Streaming jobs process unbounded data continuously and run until explicitly stopped. Use cases include real-time ingestion from Kafka, CDC pipelines, and continuous ETL.

- **Flink-style continuous processing** — events are processed as soon as they arrive (poll 100ms), NOT Spark-style micro-batch
- Streaming jobs are always delegated to Workers (multi-worker, partition-based distribution)
- Barrier checkpoint: Master-coordinated distributed checkpoint across all Workers
- Window operations: TUMBLING, SLIDING, SESSION (with alias support for AGG expressions)
- Operations: FILTER, GROUP_BY, WINDOW, AGG — configured via REST API
- **Exactly-once semantics** for transactional sinks (Iceberg, JDBC, NeorunBase JDBC, Kafka Transactions)

### Exactly-Once Delivery

For transactional sinks, Ontul guarantees exactly-once semantics through barrier checkpoint:

```
1. Master triggers CHECKPOINT_TRIGGER → all Workers
2. Each Worker: flush sink → commit sink transaction
3. Snapshot state (Kafka offsets + window state) to Exchange Manager
4. Commit Kafka consumer offsets (AFTER sink commit)
5. Report CHECKPOINT_COMPLETE → Master
6. Master: all Workers acked → checkpoint globally complete
```

| Sink | Transactional | Exactly-Once |
|------|:---:|:---:|
| Iceberg | O | O |
| JDBC | O | O |
| NeorunBase (JDBC) | O | O |
| Kafka (transactional=true) | O | O |
| NeorunBase (REST) | X | at-least-once |
| Kafka (non-tx) | X | at-least-once |
| Console/ES/HTTP | X | at-least-once |

## SDK

Ontul provides SDKs for programmatic data processing:

### Java SDK

```java
OntulSession session = OntulSession.builder()
    .master("localhost", 47470)
    .token("your-jwt-token")
    .build();

// Read from S3
DataFrame df = session.source(Source.s3("s3://bucket/data", "parquet")
    .connection("my-s3-conn"));

// SQL query
DataFrame result = session.sql("SELECT * FROM iceberg_catalog.db.my_table");

// Write to Iceberg
df.sink(Sink.table("iceberg_catalog.db.target_table"));
```

### Python SDK

```python
from ontul import OntulSession

session = OntulSession(host="localhost", port=47470, token="your-jwt-token")

# Execute SQL
result = session.sql("SELECT * FROM iceberg_catalog.db.my_table LIMIT 10")

# Get Pandas DataFrame
df = session.sql_pandas("SELECT * FROM iceberg_catalog.db.my_table")
```

## Execution Modes

### Client Mode

- Interactive execution from applications or notebooks
- No dependency upload required — SDK only
- `session.source()` / `df.sink()` — results return to the client
- Source and Sink configurations reference connection IDs or inline properties

### Server Mode

- Upload JARs with custom dependencies to the server
- `session.submit(MyJob.class)` — jobs run asynchronously
- Client can disconnect after submission
- Status and log polling via REST API or Admin UI
- Suitable for scheduling with workflow orchestrators (e.g., Airflow)

## Job Management

Jobs are managed through the REST API and Admin UI:

- Submit, kill, and monitor jobs
- View job status (SUBMITTED → RUNNING → COMPLETED / FAILED / KILLED)
- Access job logs in real-time
- Job history stored locally or on S3

## Driver Delegation

Ontul automatically determines where to run the driver for each job:

| Job Type | Driver Location |
|----------|----------------|
| Small batch | Master |
| Large batch | Worker |
| Streaming | Worker (always) |

This keeps the Master responsive for query planning and administrative operations while offloading heavy computation to Workers.
