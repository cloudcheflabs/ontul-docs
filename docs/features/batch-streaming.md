# Batch & Streaming Processing

Ontul supports distributed batch and streaming data processing through its SDK, enabling ETL pipelines, data transformations, and continuous streaming jobs within the same cluster used for interactive SQL.

## Job Types

### Batch Jobs

Batch jobs process a bounded dataset and complete when all data has been processed. Use cases include ETL pipelines, data migration, periodic aggregation, and data export.

- Small batch jobs run with the Master as the driver
- Large batch jobs are automatically delegated to a Worker as the driver, keeping the Master lightweight

### Streaming Jobs

Streaming jobs process unbounded data continuously and run until explicitly stopped. Use cases include real-time ingestion from Kafka, CDC pipelines, and continuous ETL.

- Streaming jobs are always delegated to a Worker as the driver
- Support watermark and window operations (tumbling, sliding)

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
