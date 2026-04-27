# Remote Mode

Ontul Remote Mode enables client applications to compose and execute data pipelines entirely on the Ontul cluster. The client sends only a lightweight SDK JAR (or Python package) — no Spark, Hadoop, or heavy dependencies. All computation, I/O, and storage happens server-side.

## Architecture

```
┌──────────────────────┐                  ┌──────────────────────────────────────────┐
│  Client Application  │                  │           Ontul Cluster                  │
│                      │  Arrow Flight    │                                          │
│  ontul-sdk.jar       │◄────────────────►│  Master → Workers → S3 / Iceberg / Kafka │
│  (few MB)            │  (zero-copy)     │                                          │
└──────────────────────┘                  └──────────────────────────────────────────┘
```

- Client connects via Arrow Flight SQL (port 47470)
- All DataFrame operations are composed as SQL on the client, executed on the cluster
- Results stream back as Arrow RecordBatches — no serialization overhead
- Client never touches S3, HDFS, or Iceberg directly

## Why Remote Mode?

| Traditional (Spark) | Ontul Remote Mode |
|--------------------|--------------------|
| 500MB+ client dependencies | ~5MB SDK JAR |
| Cluster resources allocated per client | Shared cluster, managed by Master |
| Client needs S3/HDFS credentials | Cluster holds all credentials |
| Spark context startup 30-60s | Session connect <100ms |
| Data moves to client for processing | Data stays server-side |

## Java SDK — DataFrame API

### Connect

```java
import com.cloudcheflabs.ontul.sdk.*;

OntulSession session = OntulSession.builder()
    .master("localhost", 47470)
    .token("your-jwt-token")
    .build();
```

### DataFrame Operations

All operations compose lazily — execution happens server-side when you call `.execute()` or `.show()`.

```java
// source → select → filter → orderBy → limit
DataFrame result = session.source(Source.sql("SELECT * FROM tpch.tiny.customer"))
    .filter("c_acctbal > 1000")
    .select("c_custkey", "c_name", "c_acctbal")
    .orderBy("c_acctbal", false)
    .limit(10);
result.show();
```

### JOIN

```java
var nation = session.source(Source.sql("SELECT * FROM tpch.tiny.nation"))
    .select("n_name", "n_regionkey");
var region = session.source(Source.sql("SELECT * FROM tpch.tiny.region"))
    .select("r_regionkey", "r_name");

var joined = nation.join(region, "l.n_regionkey = r.r_regionkey")
    .select("n_name", "r_name")
    .orderBy("r_name");
joined.show();
```

### groupBy + agg

```java
var stats = session.source(Source.sql("SELECT * FROM tpch.tiny.customer"))
    .join(nationDf, "l.c_nationkey = r.n_nationkey")
    .groupBy("n_name")
    .agg("COUNT(*) AS customer_count, AVG(c_acctbal) AS avg_balance")
    .orderBy("avg_balance", false);
stats.show();
```

### withColumn (Computed Columns)

```java
var enriched = session.source(Source.sql("SELECT * FROM sales"))
    .filter("quantity > 0 AND unit_price > 0")
    .withColumn("total_amount", "quantity * unit_price")
    .withColumn("discount_price", "unit_price * 0.9");
```

### Cross-Catalog Federation

```java
// Join data across Iceberg, TPC-H, and NeorunBase in a single query
var federation = session.source(Source.sql(
    "SELECT r.region, r.total_revenue, n.n_name AS sample_nation " +
    "FROM ice.analytics.region_summary r " +
    "JOIN tpch.tiny.nation n ON r.region_key = n.n_regionkey " +
    "ORDER BY r.total_revenue DESC"));
federation.show();
```

---

## End-to-End Pipeline

A complete data pipeline: generate data → write to S3 → transform → join → aggregate → save to Iceberg → query.

```java
OntulSession session = OntulSession.builder()
    .master("localhost", 47470).build();

// 1. Write raw data to S3 as Parquet
session.source(Source.data(salesData))
    .sink(Sink.s3("s3://bucket/raw/sales", "parquet")
        .property("endpoint", "http://minio:9000")
        .property("accessKey", "minioadmin")
        .property("secretKey", "minioadmin")
        .property("pathStyle", "true"));

// 2. Read from S3 + Transform (server-side)
var rawSales = session.source(Source.s3("s3://bucket/raw/sales", "parquet")
    .property("endpoint", "http://minio:9000")
    .property("accessKey", "minioadmin")
    .property("secretKey", "minioadmin")
    .property("pathStyle", "true"));

var enriched = rawSales
    .filter("quantity > 0 AND unit_price > 0")
    .withColumn("total_amount", "quantity * unit_price");

// 3. Join with dimension data
var products = session.source(Source.s3("s3://bucket/raw/products", "parquet")
    .property("endpoint", "http://minio:9000")
    .property("accessKey", "minioadmin")
    .property("secretKey", "minioadmin")
    .property("pathStyle", "true"));

var joined = enriched
    .join(products, "l.product_id = r.product_id")
    .select("sale_id", "product_name", "category", "quantity",
            "unit_price", "total_amount", "region");

// 4. Aggregate + Save to Iceberg
joined.groupBy("category", "region")
    .agg("SUM(total_amount) AS revenue, COUNT(*) AS order_count")
    .sink(Sink.table("ice.analytics.revenue_summary"));

// 5. Query Iceberg results
session.source(Source.sql(
    "SELECT * FROM ice.analytics.revenue_summary ORDER BY revenue DESC"))
    .show();
```

---

## Python SDK

### Connect

```python
from ontul.session import OntulSession

session = OntulSession(host="localhost", port=47470, token="your-jwt-token")
```

### DataFrame Operations

```python
# source → select → filter → orderBy → limit
result = (session.source("SELECT * FROM tpch.tiny.customer")
    .filter("c_acctbal > 1000")
    .select("c_custkey", "c_name", "c_acctbal")
    .order_by("c_acctbal", ascending=False)
    .limit(10)
    .to_pandas())
print(result)
```

### JOIN + groupBy

```python
nation = session.source("SELECT n_name, n_regionkey FROM tpch.tiny.nation")
region = session.source("SELECT r_regionkey, r_name FROM tpch.tiny.region")

result = (nation.join(region, "l.n_regionkey = r.r_regionkey")
    .select("r_name")
    .group_by("r_name")
    .agg("COUNT(*) AS nation_count")
    .order_by("nation_count", ascending=False)
    .to_pandas())
print(result)
```

### End-to-End Pipeline

```python
# 1. Write to S3
session.sink_s3(sales_data, "s3://bucket/raw/sales", "parquet", s3_config)

# 2. Read from S3 → Transform (server-side)
raw = session.source_s3("s3://bucket/raw/sales", "parquet", s3_config)
session.sink(raw, "ice.staging.sales")

valid = (source(session, "ice.staging.sales")
    .filter("quantity > 0 AND unit_price > 0")
    .with_column("total_amount", "quantity * unit_price"))

# 3. Join + Aggregate → Save to Iceberg
products = source(session, "ice.staging.products")
joined = valid.join(products, "l.product_id = r.product_id")

(joined
    .group_by("category", "region")
    .agg("SUM(total_amount) AS revenue", "COUNT(*) AS orders")
    .save_as_table("ice.analytics.revenue_summary"))

# 4. Query results
print(source(session, "ice.analytics.revenue_summary")
    .order_by("revenue", ascending=False)
    .to_pandas())
```

---

## Distributed Map (UDF + Pipeline)

Combine Remote Mode with UDFs for ML/ETL workloads. The map function runs on Workers in parallel:

```java
// Register UDF
session.registerUdf("ocr", PipelineUdfs.class, "ocr");

// Read binary images from S3
DataFrame images = session.source(Source.s3("s3://bucket/images", "binary")
    .property("endpoint", "http://minio:9000")
    .property("accessKey", "minioadmin")
    .property("secretKey", "minioadmin"));

// Apply distributed map — runs on Workers
DataFrame results = images.map((MapFunction) row -> {
    byte[] bytes = (byte[]) row.get("data");
    OcrResult r = PipelineUdfs.ocr(bytes);
    Map<String, Object> out = new LinkedHashMap<>();
    out.put("name",       row.get("name"));
    out.put("text",       r.text);
    out.put("confidence", r.confidence);
    return out;
});

// Save to Iceberg
results.sink(Sink.table("ice.analytics.ocr_results"));

// Query with UDF in SQL
session.source(Source.sql(
    "SELECT name, text, length_class(text) AS cls FROM ice.analytics.ocr_results"))
    .show();
```

---

## Source Types

| Source | Description |
|--------|-------------|
| `Source.sql(query)` | Any SQL query across registered catalogs |
| `Source.s3(path, format)` | Read from S3 (parquet, json, csv, binary) |
| `Source.data(list)` | In-memory data (client-side List of Maps) |
| `Source.kafka(conn, topic)` | Kafka stream source (for streaming mode) |

## Sink Types

| Sink | Description |
|------|-------------|
| `Sink.table(catalog.schema.table)` | Write to Iceberg/catalog table |
| `Sink.s3(path, format)` | Write to S3 (parquet, json, csv, binary) |
| `Sink.kafka(conn, topic)` | Write to Kafka topic |
| `Sink.neorunBase(url, table)` | Write to NeorunBase (REST bulk-insert) |

---

## Key Benefits

- **Thin client**: Only ontul-sdk needed (~5MB). No Spark/Hadoop/Flink on client side.
- **Server-side execution**: All computation, I/O, joins, aggregations run on Ontul Workers.
- **Zero-copy transfer**: Arrow Flight streams data without serialization overhead.
- **Credential isolation**: S3/Kafka/Iceberg credentials stay on the cluster, never exposed to clients.
- **Sub-second connect**: No context startup delay. Session connect in <100ms.
- **Cross-catalog federation**: JOIN across Iceberg, TPC-H, NeorunBase, S3 in a single pipeline.
- **UDF integration**: Register Java/Python UDFs and use them in both SQL and distributed map.

---

## SDK Reference

- [Java SDK](../reference/sdk-java.md)
- [Python SDK](../reference/sdk-python.md)
- [UDF Documentation](../features/udf.md)
