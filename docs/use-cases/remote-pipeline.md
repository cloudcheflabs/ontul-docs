# Remote Pipeline: S3 → Transform → Iceberg

This tutorial demonstrates Ontul's **Remote mode** — building an end-to-end data pipeline where the client only has a lightweight SDK, and all computation happens on the Ontul cluster.

**What you will learn:**

- Use Ontul Remote mode to run complex ETL from a thin client
- Write raw data to S3 as Parquet (data lake landing zone)
- Read from S3 and apply multi-step transformations (filter, enrich, join)
- Aggregate results and save to Iceberg (data warehouse)
- Query Iceberg tables and display results on the client
- Run cross-catalog federation queries (Iceberg + TPC-H)

## Prerequisites

- Java 17
- Docker (for MinIO + Polaris)

## 1. Setup

```bash
curl -L -O https://github.com/cloudcheflabs/ontul-pack/releases/download/ontul-archive/ontul-1.0.0.tar.gz
tar zxvf ontul-1.0.0.tar.gz && cd ontul-1.0.0

docker compose -f examples/docker-compose-iceberg.yml up -d
examples/bin/setup-iceberg.sh
```

## 2. Run

```bash
# Java
examples/bin/run-remote-pipeline.sh

# Python
examples/bin/run-remote-pipeline-python.sh
```

---

## Pipeline Architecture

```
Client (ontul-sdk.jar / Python SDK)
  │  Client only sends SQL commands + receives Arrow results.
  │  No data passes through the client.
  │
  ├── Step 1: Generate sales + product data (in-memory → VALUES SQL)
  │
  ├── Step 2: Sink.s3() → WRITE_S3 command
  │       └── Master writes Parquet directly to S3 (MinIO)
  │
  ├── Step 3: Source.s3() → read_s3() table function
  │       └── Master reads S3 via FileConnector → Arrow cache in memory
  │       └── filter(qty > 0) → withColumn(total_amount) — executed on Worker
  │
  ├── Step 4: join(sales, products)
  │       └── Worker fetches Arrow cache from Master via NIO
  │       └── HashJoin on Worker — all server-side
  │
  ├── Step 5: groupBy → agg → Sink.table() [Aggregate → Iceberg]
  │       ├── revenue_by_category_region
  │       ├── top_products
  │       └── region_summary
  │
  ├── Step 6: Source.sql("SELECT * FROM ice.examples...") [Query]
  │       └── Worker scans Iceberg → Arrow Flight → Client display
  │
  └── Step 7: Iceberg × TPC-H federation query
```

**All processing happens on the Ontul cluster (Master + Workers).** The client only sends SQL strings via Arrow Flight and receives Arrow RecordBatch results. S3 read/write is performed server-side — data never passes through the client.

---

## Example Code (Java)

### Connect to Ontul Cluster

```java
OntulSession session = OntulSession.builder()
        .master("localhost", 47470)
        .build();
```

### Write Data to S3 as Parquet

```java
var sales = List.of(
    Map.of("sale_id", 1, "product_id", 5, "quantity", 3,
           "unit_price", 299.99, "region", "North America"),
    // ... 500 rows
);

session.source(Source.data(sales))
    .sink(Sink.s3("s3://iceberg-warehouse/raw/sales", "parquet")
        .property("endpoint", "http://localhost:9000")
        .property("accessKey", "minioadmin")
        .property("secretKey", "minioadmin")
        .property("pathStyle", "true"));
```

### Read from S3 + Transform

```java
var rawSales = session.source(Source.s3("s3://iceberg-warehouse/raw/sales", "parquet")
        .property("endpoint", "http://localhost:9000")
        .property("accessKey", "minioadmin")
        .property("secretKey", "minioadmin")
        .property("pathStyle", "true"));

// Filter invalid records + add computed column
var enriched = rawSales
    .filter("quantity > 0")
    .filter("unit_price > 0")
    .withColumn("total_amount", "quantity * unit_price");
```

### Join with Dimension Table

```java
var products = session.source(Source.s3("s3://iceberg-warehouse/raw/products_dim", "parquet")
        .property("endpoint", "http://localhost:9000")
        .property("accessKey", "minioadmin")
        .property("secretKey", "minioadmin")
        .property("pathStyle", "true"));

var joined = enriched
    .join(products, "l.product_id = r.product_id")
    .select("sale_id", "product_name", "category", "quantity",
            "unit_price", "total_amount", "region");
```

### Aggregate + Save to Iceberg

```java
// Revenue by category × region
joined.groupBy("category", "region")
    .agg("SUM(total_amount) AS revenue, COUNT(*) AS order_count, " +
         "CAST(AVG(total_amount) AS DECIMAL(10,2)) AS avg_order_value")
    .sink(Sink.table("ice.examples.revenue_by_category_region"));

// Top products
joined.groupBy("product_name", "category")
    .agg("SUM(total_amount) AS total_revenue, SUM(quantity) AS total_qty")
    .sink(Sink.table("ice.examples.top_products"));
```

### Query Iceberg Results

```java
var top10 = session.source(Source.sql(
        "SELECT * FROM ice.examples.top_products"))
    .orderBy("total_revenue", false)
    .limit(10);
// Results returned as Arrow RecordBatch to client
```

### Federation: Iceberg × TPC-H

```java
var federation = session.source(Source.sql(
    "SELECT r.region, r.total_revenue, n.n_name AS sample_nation " +
    "FROM ice.examples.region_summary r " +
    "JOIN tpch.tiny.nation n ON ... " +
    "ORDER BY r.total_revenue DESC"));
```

---

## Python Version

```python
from ontul.session import OntulSession

session = OntulSession()

s3_config = {
    'endpoint': 'http://localhost:9000',
    'accessKey': 'minioadmin',
    'secretKey': 'minioadmin',
    'pathStyle': 'true',
}

# Write to S3 (server-side — Master writes directly to S3)
session.sink_s3(sales, "s3://bucket/raw/sales", "parquet", s3_config)

# Read from S3 (server-side — Master reads via FileConnector)
raw_sales = session.source_s3("s3://bucket/raw/sales", "parquet", s3_config)

# Sink to Iceberg staging (PyArrow Table → CTAS)
session.sink(raw_sales, "ice.examples.stg_sales")

# Transform (server-side via RemoteDataFrame)
valid = (source(session, "ice.examples.stg_sales")
    .filter("quantity > 0")
    .with_column("total_amount", "quantity * unit_price"))

# Join
joined = valid.join(products_df, "l.product_id = r.product_id")

# Aggregate → Iceberg
(joined.group_by("category", "region")
    .agg("SUM(total_amount) AS revenue", "COUNT(*) AS cnt")
    .save_as_table("ice.examples.revenue_summary"))

# Query results
print(source(session, "ice.examples.revenue_summary")
    .order_by("revenue", ascending=False).to_pandas())
```

---

## Cleanup

```bash
bin/stop-example-servers.sh
docker compose -f examples/docker-compose-iceberg.yml down -v
```
