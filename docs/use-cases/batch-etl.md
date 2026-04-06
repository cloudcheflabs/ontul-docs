# Batch ETL: S3 Parquet + Iceberg

This tutorial demonstrates how to build a batch ETL pipeline using Ontul with Apache Iceberg and S3 (MinIO).

**What you will learn:**

- Write in-memory data to S3 as Parquet
- Read Parquet files from S3
- Load data into Iceberg tables
- Use the DataFrame API: filter, select, join, groupBy, agg, orderBy
- Execute DDL/DML: CTAS, INSERT INTO, MERGE INTO
- Run cross-catalog federation queries (Iceberg + TPC-H)

## Prerequisites

- Java 17
- Docker (for MinIO + Polaris)

## 1. Download and Install Ontul

```bash
curl -L -O https://github.com/cloudcheflabs/ontul-pack/releases/download/ontul-archive/ontul-1.0.0.tar.gz
tar zxvf ontul-1.0.0.tar.gz
cd ontul-1.0.0
```

## 2. Start Infrastructure

Start MinIO (S3-compatible storage), Apache Polaris (Iceberg REST catalog), and Kafka:

```bash
docker compose -f examples/docker-compose-iceberg.yml up -d
```

This starts:

| Service | Port | Description |
|---------|------|-------------|
| MinIO | 9000 (API), 9001 (Console) | S3-compatible object storage |
| Polaris | 8181 | Iceberg REST catalog |
| Kafka | 9092 | Message broker (for streaming use case) |

Wait for all services to be healthy:

```bash
# Check Polaris setup is complete
docker inspect ontul-example-polaris-setup | grep '"Status"'
```

## 3. Start Ontul Cluster and Register Catalogs

```bash
# Start Ontul (1 Master, 2 Workers, 1 ZooKeeper)
examples/bin/setup-iceberg.sh
```

This script:

1. Starts the Ontul cluster
2. Logs in and changes the default password
3. Registers `tpch` catalog (built-in TPC-H benchmark data)
4. Registers `ice` catalog (Iceberg via Polaris + MinIO)

After setup, the `ONTUL_USER_TOKEN` environment variable is exported.

## 4. Run the Batch ETL Example

```bash
examples/bin/run-iceberg-batch.sh
```

---

## Example Code (Java)

The full example is in `examples/java/IcebergBatchExample.java`.

### Step 1: Write Data to S3 as Parquet

Create in-memory product data and write it to S3:

```java
OntulSession session = OntulSession.builder()
        .master("localhost", 47470)
        .build();

var products = List.of(
    Map.<String, Object>of("product_id", 1, "name", "Laptop", "category", "Electronics", "price", 1299.99, "stock", 50),
    Map.<String, Object>of("product_id", 2, "name", "Mouse",  "category", "Electronics", "price", 29.99,   "stock", 200),
    // ... more products
);

session.source(Source.data(products))
    .sink(Sink.s3("s3://iceberg-warehouse/raw/products", "parquet")
        .property("endpoint", "http://localhost:9000")
        .property("accessKey", "minioadmin")
        .property("secretKey", "minioadmin")
        .property("pathStyle", "true"));
```

### Step 2: Read Parquet from S3

```java
var s3Products = session.source(Source.s3("s3://iceberg-warehouse/raw/products", "parquet")
        .property("endpoint", "http://localhost:9000")
        .property("accessKey", "minioadmin")
        .property("secretKey", "minioadmin")
        .property("pathStyle", "true"))
    .select("product_id", "name", "category", "price", "stock");
```

### Step 3: DataFrame Chain (filter, select, orderBy)

```java
var expensiveElectronics = session.source(Source.s3("s3://iceberg-warehouse/raw/products", "parquet")
        .property("endpoint", "http://localhost:9000")
        .property("accessKey", "minioadmin")
        .property("secretKey", "minioadmin")
        .property("pathStyle", "true"))
    .filter("category = 'Electronics'")
    .filter("price > 100")
    .select("name", "price", "stock")
    .orderBy("price", false);
```

### Step 4: Load into Iceberg Table

Write the S3 Parquet data into an Iceberg table managed by Polaris:

```java
session.source(Source.s3("s3://iceberg-warehouse/raw/products", "parquet")
        .property("endpoint", "http://localhost:9000")
        .property("accessKey", "minioadmin")
        .property("secretKey", "minioadmin")
        .property("pathStyle", "true"))
    .sink(Sink.table("ice.examples.products"));
```

### Step 5: CTAS (Create Table As Select)

Create Iceberg tables from TPC-H benchmark data:

```java
session.execute("CREATE TABLE ice.examples.nations AS SELECT * FROM tpch.tiny.nation");
session.execute(
    "CREATE TABLE ice.examples.orders AS " +
    "SELECT o_orderkey, o_custkey, o_totalprice, o_orderstatus, o_orderdate " +
    "FROM tpch.tiny.orders LIMIT 200");
```

### Step 6: GroupBy + Aggregation on Iceberg

```java
var categorySummary = session.source(Source.sql("SELECT * FROM ice.examples.products"))
    .groupBy("category")
    .agg("COUNT(*) AS product_count, CAST(AVG(price) AS DECIMAL(10,2)) AS avg_price, SUM(stock) AS total_stock")
    .orderBy("product_count", false);
```

### Step 7: Cross-Catalog JOIN (Iceberg + TPC-H)

Query Iceberg orders joined with TPC-H customer data in a single query:

```java
var orders = session.source(Source.sql("SELECT * FROM ice.examples.orders"));
var customers = session.source(Source.sql("SELECT * FROM tpch.tiny.customer"));

var topCustomers = orders
    .join(customers, "l.o_custkey = r.c_custkey")
    .groupBy("c_name")
    .agg("SUM(o_totalprice) AS total_spent")
    .orderBy("total_spent", false)
    .limit(5);
```

### Step 8: INSERT INTO and MERGE INTO

```java
// Append more orders
session.execute(
    "INSERT INTO ice.examples.orders " +
    "SELECT o_orderkey, o_custkey, o_totalprice, o_orderstatus " +
    "FROM tpch.tiny.orders LIMIT 100");

// Upsert nations
session.execute(
    "MERGE INTO ice.examples.nations AS t " +
    "USING (SELECT * FROM tpch.tiny.nation WHERE n_regionkey = 0) AS s " +
    "ON t.n_nationkey = s.n_nationkey " +
    "WHEN MATCHED THEN UPDATE SET n_comment = 'updated' " +
    "WHEN NOT MATCHED THEN INSERT (n_nationkey, n_name, n_regionkey, n_comment) " +
    "VALUES (s.n_nationkey, s.n_name, s.n_regionkey, s.n_comment)");
```

---

## Python Version

The same example is available in Python: `examples/python/iceberg_batch_example.py`

```bash
examples/bin/run-iceberg-batch-python.sh
```

```python
from ontul.session import OntulSession

session = OntulSession()

# Write to S3
session.sink_s3(products, "s3://iceberg-warehouse/raw/products-py", "parquet", s3_config)

# Read from S3
rows = session.source_s3("s3://iceberg-warehouse/raw/products-py", "parquet", s3_config)

# Sink to Iceberg
session.sink(pa.Table.from_pandas(df), "ice.examples.products_py")

# CTAS
session.execute("CREATE TABLE ice.examples.nations_py AS SELECT * FROM tpch.tiny.nation")

# Cross-catalog JOIN (via RemoteDataFrame)
orders = source(session, "ice.examples.orders_py")
customers = source(session, "tpch.tiny.customer")
top = (orders.join(customers, "l.o_custkey = r.c_custkey")
    .group_by("c_name")
    .agg("SUM(o_totalprice) AS total_spent")
    .order_by("total_spent", ascending=False)
    .limit(5))
print(top.to_pandas())
```

---

## Cleanup

```bash
# Stop Ontul cluster
bin/stop-example-servers.sh

# Stop Docker (MinIO + Polaris + Kafka)
docker compose -f examples/docker-compose-iceberg.yml down -v
```
