# Batch Guide

Ontul batch processing handles bounded datasets — ETL pipelines, data migration, periodic aggregation, and data export. Jobs are submitted via REST API, shell script, or SDK.

## Job Types

| Type | Description | How to Submit |
|------|-------------|---------------|
| **BATCH** | SQL-based (CTAS, INSERT INTO SELECT) | REST API, SDK |
| **CLASS** | Custom Java class with dependencies | REST API, shell, SDK |
| **PYTHON** | Python script execution | REST API, shell |

## Job Lifecycle

```
SUBMITTED → PLANNING → RUNNING → COMPLETED
                                → FAILED
                                → KILLED
```

---

## Submitting Jobs

### 1. REST API

#### BATCH Job (SQL)

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

#### CLASS Job (Custom Java)

```bash
# Step 1: Upload dependency JARs
curl -X POST http://localhost:8080/v1/api/deps \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-File-Name: my-etl-job.jar" \
  -H "Content-Type: application/octet-stream" \
  --data-binary @my-etl-job.jar
# Response: {"path": "deps/my-etl-job.jar", "size": 245760}

# Step 2: Submit CLASS job
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
# Step 1: Upload Python script
curl -X POST http://localhost:8080/v1/api/deps \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-File-Name: etl_pipeline.py" \
  -H "Content-Type: application/octet-stream" \
  --data-binary @etl_pipeline.py

# Step 2: Submit PYTHON job
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

Ontul ships with `bin/submit.sh` for submitting CLASS and PYTHON jobs from the command line. The script automatically handles dependency diffing, upload, job submission, and log streaming.

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
  --jars /path/to/job.jar,/path/to/dep1.jar \
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

#### All Options

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
  2026-04-28 10:00:30 INFO  Wrote 50000 rows
  2026-04-28 10:00:30 INFO  Job completed

=== Job COMPLETED (ID: class-1714300000000) ===
```

### 3. Java SDK

#### DataFrame API (Client Mode)

All computation happens on the Ontul cluster. The client only sends/receives Arrow data.

```java
OntulSession session = OntulSession.builder()
    .master("localhost", 47470)
    .token("your-jwt-token")
    .build();

// Read → Transform → Write
session.source(Source.s3("s3://bucket/raw/sales", "parquet")
        .property("endpoint", "http://minio:9000")
        .property("accessKey", "minioadmin")
        .property("secretKey", "minioadmin")
        .property("pathStyle", "true"))
    .filter("quantity > 0 AND unit_price > 0")
    .withColumn("total_amount", "quantity * unit_price")
    .sink(Sink.table("ice.warehouse.enriched_sales"));
```

#### Submit Custom Job (Server Mode)

Submit a custom Java class. The SDK auto-diffs dependencies and uploads missing JARs.

```java
JobHandle job = session.submit(MyEtlJob.class, Map.of(
    "input", "s3://bucket/raw/data",
    "output", "ice.warehouse.processed"
));

// Wait for completion (streams logs to console)
String finalStatus = job.waitForCompletion(300); // 300s timeout
System.out.println("Job finished: " + finalStatus);

// Or poll manually
String status = job.status();  // SUBMITTED, RUNNING, COMPLETED, FAILED, KILLED

// Get logs
job.logs().forEach(System.out::println);

// Kill if needed
job.kill();
```

### 4. Python SDK

```python
from ontul.session import OntulSession

session = OntulSession(host="localhost", port=47470, token="your-jwt-token")

# SQL batch job
session.execute(
    "CREATE TABLE ice.warehouse.top_customers AS "
    "SELECT c_custkey, c_name, c_acctbal "
    "FROM tpch.sf1.customer WHERE c_acctbal > 5000"
)

# S3 → Iceberg
raw = session.source_s3("s3://bucket/raw/sales", "parquet", s3_config)
session.sink(raw, "ice.staging.sales")

# Verify
df = session.sql_pandas("SELECT COUNT(*) AS cnt FROM ice.staging.sales")
print(df)
```

---

## Job Management

### Get Job Status

```bash
curl http://localhost:8080/v1/api/job/status/{jobId} \
  -H "Authorization: Bearer $TOKEN"
```

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

### Stream Job Logs

```bash
# All logs
curl http://localhost:8080/v1/api/job/{jobId}/log \
  -H "Authorization: Bearer $TOKEN"

# Incremental polling (from offset)
curl "http://localhost:8080/v1/api/job/{jobId}/log?from=100" \
  -H "Authorization: Bearer $TOKEN"
```

### Kill a Running Job

```bash
curl -X POST http://localhost:8080/v1/api/job/kill/{jobId} \
  -H "Authorization: Bearer $TOKEN"
```

### List Jobs

```bash
curl http://localhost:8080/admin/jobs \
  -H "Authorization: Bearer $TOKEN"
```

---

## Source/Sink by Connection ID

Production batch jobs should reference a registered connection by ID instead of repeating endpoints and credentials in every call. Register once via `POST /admin/connections`, then use the ID anywhere a source or sink is built.

```java
// S3 source/sink driven by a stored connection — no credentials in code
session.source(Source.s3("s3://warehouse/raw/sales", "parquet")
        .connection("warehouse-s3"))
    .filter("quantity > 0")
    .withColumn("total", "quantity * unit_price")
    .sink(Sink.s3("s3://warehouse/curated/sales", "parquet")
        .connection("warehouse-s3"));

// JDBC source / sink
session.source(Source.jdbc("analytics-pg", "SELECT * FROM public.orders WHERE dt = CURRENT_DATE"))
    .sink(Sink.table("ice.staging.orders"));

session.source(Source.sql("SELECT * FROM ice.warehouse.daily_summary"))
    .sink(Sink.jdbc("analytics-pg", "public.daily_summary"));
```

Inline `.property(...)` values supplied alongside `.connection(id)` are merged on top of the stored connection — handy for per-job overrides (consumer group, query timeout, batch size) without forking the connection. See [Connection ID](../features/connection-id.md) for the full reference.

---

## DataFrame API

### Source Types

| Source | Description | Example |
|--------|-------------|---------|
| `Source.sql(query)` | SQL query | `Source.sql("SELECT * FROM tpch.tiny.customer")` |
| `Source.s3(path, format)` | S3 files (inline or by connection ID) | `Source.s3("s3://bucket/data", "parquet").connection("warehouse-s3")` |
| `Source.jdbc(connId, query)` | JDBC query by connection ID | `Source.jdbc("analytics-pg", "SELECT * FROM orders")` |
| `Source.data(list)` | In-memory data | `Source.data(List.of(Map.of("id", 1, "name", "a")))` |

### Transformations

```java
DataFrame result = session.source(Source.sql("SELECT * FROM ice.raw.sales"))
    .filter("quantity > 0")                          // row filter
    .filter("unit_price > 0")                        // chain multiple filters
    .select("sale_id", "product_id", "quantity")     // column projection
    .withColumn("total", "quantity * unit_price")     // computed column
    .orderBy("total", false)                          // sort (false = DESC)
    .limit(100);                                      // row limit
```

### JOIN

```java
var sales = session.source(Source.sql("SELECT * FROM ice.raw.sales"));
var products = session.source(Source.sql("SELECT * FROM ice.dim.products"));

var joined = sales
    .join(products, "l.product_id = r.product_id")
    .select("sale_id", "product_name", "category", "quantity", "total_amount");
```

### groupBy + agg

```java
var summary = joined
    .groupBy("category", "region")
    .agg("SUM(total_amount) AS revenue, COUNT(*) AS order_count, " +
         "CAST(AVG(total_amount) AS DECIMAL(10,2)) AS avg_order")
    .orderBy("revenue", false);
```

### Sink Types

| Sink | Description | Example |
|------|-------------|---------|
| `Sink.table(name)` | Iceberg/catalog table | `Sink.table("ice.warehouse.results")` |
| `Sink.s3(path, format)` | S3 output (inline or by connection ID) | `Sink.s3("s3://bucket/output", "parquet").connection("warehouse-s3")` |
| `Sink.jdbc(connId, table)` | JDBC table by connection ID | `Sink.jdbc("analytics-pg", "public.daily_summary")` |

---

## Complete Example: S3 → Transform → Iceberg

### Setup

```bash
cd ontul-1.0.0-SNAPSHOT
docker compose -f examples/docker-compose-iceberg.yml up -d
examples/bin/setup-iceberg.sh
```

### Run

```bash
examples/bin/run-iceberg-batch.sh
```

### Java Code

```java
OntulSession session = OntulSession.builder()
    .master("localhost", 47470).build();

// 1. Write raw data to S3
session.source(Source.data(salesData))
    .sink(Sink.s3("s3://iceberg-warehouse/raw/sales", "parquet")
        .property("endpoint", "http://localhost:9000")
        .property("accessKey", "minioadmin")
        .property("secretKey", "minioadmin")
        .property("pathStyle", "true"));

// 2. Read from S3
var raw = session.source(Source.s3("s3://iceberg-warehouse/raw/sales", "parquet")
    .property("endpoint", "http://localhost:9000")
    .property("accessKey", "minioadmin")
    .property("secretKey", "minioadmin")
    .property("pathStyle", "true"));

// 3. Transform
var enriched = raw
    .filter("quantity > 0 AND unit_price > 0")
    .withColumn("total_amount", "quantity * unit_price");

// 4. Join with dimension
var products = session.source(Source.s3("s3://iceberg-warehouse/raw/products", "parquet")
    .property("endpoint", "http://localhost:9000")
    .property("accessKey", "minioadmin")
    .property("secretKey", "minioadmin")
    .property("pathStyle", "true"));

var joined = enriched
    .join(products, "l.product_id = r.product_id")
    .select("sale_id", "product_name", "category", "quantity",
            "unit_price", "total_amount", "region");

// 5. Aggregate → Iceberg
joined.groupBy("category", "region")
    .agg("SUM(total_amount) AS revenue, COUNT(*) AS order_count")
    .sink(Sink.table("ice.examples.revenue_by_category_region"));

// 6. CTAS from TPC-H
session.execute("CREATE TABLE ice.examples.nations AS SELECT * FROM tpch.tiny.nation");

// 7. Cross-catalog JOIN
var federation = session.source(Source.sql(
    "SELECT r.region, r.revenue, n.n_name AS nation " +
    "FROM ice.examples.revenue_by_category_region r " +
    "JOIN tpch.tiny.nation n ON r.region_key = n.n_regionkey"))
    .show();

// 8. INSERT INTO + MERGE INTO
session.execute("INSERT INTO ice.examples.orders " +
    "SELECT * FROM tpch.tiny.orders LIMIT 100");

session.execute("MERGE INTO ice.examples.nations AS t " +
    "USING tpch.tiny.nation AS s ON t.n_nationkey = s.n_nationkey " +
    "WHEN MATCHED THEN UPDATE SET n_comment = 'merged'");
```

### Python Code

```python
from ontul.session import OntulSession

session = OntulSession()
s3_config = {
    'endpoint': 'http://localhost:9000',
    'accessKey': 'minioadmin', 'secretKey': 'minioadmin',
    'pathStyle': 'true'
}

# Write to S3
session.sink_s3(sales_data, "s3://iceberg-warehouse/raw/sales_py", "parquet", s3_config)

# Read from S3 → Iceberg staging
raw = session.source_s3("s3://iceberg-warehouse/raw/sales_py", "parquet", s3_config)
session.sink(raw, "ice.examples.stg_sales_py")

# Transform + aggregate → Iceberg
session.execute(
    "CREATE TABLE ice.examples.rev_py AS "
    "SELECT category, SUM(quantity * unit_price) AS revenue "
    "FROM ice.examples.stg_sales_py "
    "WHERE quantity > 0 AND unit_price > 0 "
    "GROUP BY category"
)

# Verify
df = session.sql_pandas("SELECT * FROM ice.examples.rev_py ORDER BY revenue DESC")
print(df)
```

---

## Driver Delegation

Ontul automatically determines where to run the driver:

| Job Type | Driver | Reason |
|----------|--------|--------|
| Small batch (< 100K rows) | Master | Fast, no dispatch overhead |
| Large batch | Worker | Offload heavy computation |
| CLASS | Worker | Custom code isolation |
| PYTHON | Worker | Python subprocess management |

---

## Cleanup

```bash
bin/stop-example-servers.sh
docker compose -f examples/docker-compose-iceberg.yml down -v
```
