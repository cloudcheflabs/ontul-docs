# Java SDK

The Ontul Java SDK connects to Ontul Master via Arrow Flight SQL and provides a DataFrame-like API for batch processing, streaming, and interactive queries.

## Dependency

The Java SDK and all dependencies are included in the Ontul distribution. Add `lib/*` to your classpath:

```bash
java -cp "lib/*" com.example.MyApp
```

## OntulSession

Entry point for all SDK operations.

```java
import com.cloudcheflabs.ontul.sdk.*;

OntulSession session = OntulSession.builder()
    .master("localhost", 47470)
    .token("your-jwt-token")
    .build();
```

### Builder Options

| Method | Description |
|--------|-------------|
| `.master(host, port)` | Ontul Master Flight SQL endpoint |
| `.token(jwtToken)` | JWT Bearer token for authentication |
| `.accessKey(key, secret)` | AccessKey authentication (alternative to JWT) |
| `.tls(true)` | Enable TLS for Flight SQL connection |

## Execute SQL

```java
// DDL
session.execute("CREATE SCHEMA IF NOT EXISTS iceberg.analytics");
session.execute("CREATE TABLE iceberg.analytics.events (id BIGINT, name VARCHAR, amount DOUBLE)");

// DML
session.execute("INSERT INTO iceberg.analytics.events SELECT * FROM tpch.tiny.lineitem LIMIT 100");
```

## DataFrame API (Query)

```java
// Read from any catalog
DataFrame df = session.source(Source.sql("SELECT * FROM tpch.tiny.customer"));

// Transformations
DataFrame result = df
    .filter("c_acctbal > 1000")
    .select("c_custkey", "c_name", "c_acctbal")
    .orderBy("c_acctbal", false)
    .limit(10);

// Execute and get results
result.execute();  // prints to console
result.show();     // formatted table output
```

### Source Types

```java
// SQL query
Source.sql("SELECT * FROM catalog.schema.table")

// S3 files
Source.s3("s3://bucket/path", "parquet").connection("my-s3-conn")

// Kafka (for streaming)
Source.kafka("connectionId", "topic").format("json").groupId("my-group")
```

## Write to Sink

```java
// Write to Iceberg table
session.source(Source.sql("SELECT * FROM tpch.tiny.customer"))
    .sink(Sink.table("iceberg.warehouse.customers"));

// Write to S3
session.source(Source.sql("SELECT * FROM tpch.tiny.lineitem"))
    .sink(Sink.s3("s3://bucket/output", "parquet").connection("my-s3-conn"));
```

## Streaming

```java
// Kafka → filter + windowed aggregation → Iceberg
String jobId = session.streamSource(
        Source.kafka("inline", "user-events")
            .property("bootstrap.servers", "kafka:9092")
            .groupId("ontul-stream")
            .property("auto.offset.reset", "earliest"))
    .filter("event_type <> 'logout'")
    .select("event_id", "event_type", "user_name", "amount", "event_time")
    .sink(Sink.table("iceberg.analytics.stream_events"))
    .commitInterval(10_000)
    .start(Duration.ofMinutes(30));

System.out.println("Streaming job started: " + jobId);
```

### Windowed Aggregation

```java
session.streamSource(
        Source.kafka("inline", "events")
            .property("bootstrap.servers", "kafka:9092")
            .groupId("agg-group"))
    .filter("amount > 100")
    .window(StreamDataFrame.WindowSpec.tumbling(Duration.ofSeconds(10)), "event_time")
    .groupBy("category")
    .agg("SUM(amount) as total", "COUNT(*) as cnt")
    .sink(Sink.table("iceberg.analytics.event_summary"))
    .commitInterval(10_000)
    .start();
```

### Window Types

```java
// Tumbling window (fixed size, non-overlapping)
StreamDataFrame.WindowSpec.tumbling(Duration.ofSeconds(10))

// Sliding window (fixed size, overlapping)
StreamDataFrame.WindowSpec.sliding(Duration.ofMinutes(5), Duration.ofMinutes(1))

// Session window (gap-based)
StreamDataFrame.WindowSpec.session(Duration.ofSeconds(30))
```

### Sink Types

```java
// Iceberg table
Sink.table("iceberg.schema.table")

// Kafka topic
Sink.kafka("connectionId", "output-topic").keyField("id")

// NeorunBase (REST bulk-insert)
Sink.neorunBase("http://neorunbase:8080", "table_name")
    .username("admin").password("password").batchSize(500)
```

## Server Mode (Submit Jobs)

For long-running jobs that continue after the client disconnects:

```java
// Submit a custom job class
String jobId = session.submit("com.example.MyBatchJob")
    .deps("my-job.jar", "dependency.jar")
    .arg("input", "s3://bucket/data")
    .arg("output", "iceberg.db.results")
    .execute();
```

## User-Defined Functions (UDFs)

```java
// Register a temporary UDF (session-scoped)
session.execute("CREATE TEMPORARY FUNCTION double_it AS 'com.example.DoubleUdf'");

// Use in SQL
DataFrame result = session.source(
    Source.sql("SELECT id, double_it(amount) as doubled FROM iceberg.db.sales"));

// Register a persistent UDF (survives restarts)
session.execute("CREATE FUNCTION my_func AS 'com.example.MyFunc'");
```

## Cross-Catalog Federation

```java
// Join across Iceberg, NeorunBase, and TPC-H in a single query
DataFrame result = session.source(Source.sql(
    "SELECT n.name, c.c_name, i.total " +
    "FROM neorun.public.orders n " +
    "JOIN tpch.tiny.customer c ON n.customer_id = c.c_custkey " +
    "JOIN iceberg.analytics.order_summary i ON n.order_id = i.order_id"));
```

## Error Handling

```java
try {
    session.execute("INSERT INTO iceberg.db.table SELECT ...");
} catch (RuntimeException e) {
    System.err.println("Query failed: " + e.getMessage());
} finally {
    session.close();
}
```

## Complete Example

```java
import com.cloudcheflabs.ontul.sdk.*;
import java.time.Duration;

public class OntulExample {
    public static void main(String[] args) {
        try (OntulSession session = OntulSession.builder()
                .master("localhost", 47470)
                .token("your-jwt-token")
                .build()) {

            // Interactive query
            session.source(Source.sql(
                "SELECT c_name, c_acctbal FROM tpch.tiny.customer ORDER BY c_acctbal DESC LIMIT 5"))
                .show();

            // Batch ETL: TPC-H → Iceberg
            session.execute("CREATE SCHEMA IF NOT EXISTS iceberg.warehouse");
            session.execute(
                "CREATE TABLE iceberg.warehouse.top_customers AS " +
                "SELECT c_custkey, c_name, c_acctbal FROM tpch.sf1.customer " +
                "WHERE c_acctbal > 5000");

            // Streaming: Kafka → Iceberg
            String jobId = session.streamSource(
                    Source.kafka("inline", "events")
                        .property("bootstrap.servers", "kafka:9092")
                        .groupId("demo"))
                .filter("amount > 0")
                .sink(Sink.table("iceberg.warehouse.events"))
                .commitInterval(5_000)
                .start(Duration.ofMinutes(10));

            System.out.println("Streaming job: " + jobId);
        }
    }
}
```
