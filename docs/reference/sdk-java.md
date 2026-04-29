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

// S3 files — by connection ID (recommended)
Source.s3("s3://bucket/path", "parquet").connection("warehouse-s3")

// S3 files — inline credentials (development only)
Source.s3("s3://bucket/path", "parquet")
    .property("endpoint", "http://minio:9000")
    .property("accessKey", "ACCESS_KEY")
    .property("secretKey", "SECRET_KEY")
    .property("pathStyle", "true")

// JDBC by connection ID
Source.jdbc("analytics-pg", "SELECT * FROM public.orders WHERE dt = CURRENT_DATE")

// Kafka by connection ID (streaming) — first arg is the connection ID
Source.kafka("events-kafka", "user-events").format("json").groupId("my-group")
```

### Source/Sink by Connection ID

Connection IDs are the recommended way to reference external systems. They come from `POST /admin/connections` and keep credentials out of code, REST payloads, and job logs:

```java
// Read from S3, write to JDBC — both endpoints by connection ID
session.source(Source.s3("s3://warehouse/raw/sales", "parquet")
        .connection("warehouse-s3"))
    .filter("quantity > 0")
    .sink(Sink.jdbc("analytics-pg", "public.sales"));

// Streaming: Kafka source + Kafka sink, same stored connection
session.streamSource(
        Source.kafka("events-kafka", "user-events")
            .groupId("ontul-stream-1")
            .property("auto.offset.reset", "earliest")
            .format("json"))
    .filter("event_type <> 'logout'")
    .sink(Sink.kafka("events-kafka", "filtered-events").keyField("event_id"))
    .commitInterval(5_000)
    .start(Duration.ofMinutes(30));
```

`.property(...)` values supplied alongside `.connection(id)` override the corresponding stored property — useful for per-job tuning (consumer group, batch size) without forking the connection. See the [Connection ID feature reference](../features/connection-id.md) for details.

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

// Kafka topic — first arg is the connection ID
Sink.kafka("events-kafka", "output-topic").keyField("id")

// JDBC by connection ID
Sink.jdbc("analytics-pg", "public.daily_summary")

// S3 output by connection ID
Sink.s3("s3://bucket/output", "parquet").connection("warehouse-s3")

// NeorunBase by connection ID (recommended)
Sink.neorunBase("neorunbase-prod", "table_name").batchSize(500)

// NeorunBase inline (development only)
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

Ontul supports three UDF scopes:

| Scope | Visibility | Lifetime |
|-------|-----------|----------|
| **TEMPORARY** | Current session only | Session end |
| **USER** | All sessions of the user | Persistent |
| **GLOBAL** | All authenticated users | Persistent (IAM-controlled) |

### Step 1: Write a UDF Class

UDFs are plain `public static` methods — no interface to implement, no `Serializable` required.

```java
public class MyUdfs {

    /** Single-arg: trim + lowercase an email. */
    public static String cleanEmail(String email) {
        if (email == null) return null;
        return email.trim().toLowerCase();
    }

    /** Multi-arg: wrap a string with configurable padding. */
    public static String bracket(String s, Long padLen) {
        if (s == null) return null;
        int pad = padLen != null ? padLen.intValue() : 1;
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < pad; i++) sb.append('[');
        sb.append(s);
        for (int i = 0; i < pad; i++) sb.append(']');
        return sb.toString();
    }

    /** Returns boolean. */
    public static boolean isBot(String userAgent) {
        if (userAgent == null) return false;
        String ua = userAgent.toLowerCase();
        return ua.contains("bot") || ua.contains("crawl") || ua.contains("spider");
    }

    /** Complex scoring logic. */
    public static long emailScore(String email) {
        if (email == null) return 0;
        String e = email.toLowerCase();
        long score = 0;
        if (e.endsWith(".com")) score += 100;
        if (e.endsWith(".io"))  score += 80;
        if (e.contains("admin")) score += 25;
        return score;
    }
}
```

### Step 2: Register UDF (Method-Reference)

```java
// Register with class + method name (recommended)
session.registerUdf("clean_email", MyUdfs.class, "cleanEmail");
session.registerUdf("email_score", MyUdfs.class, "emailScore");
session.registerUdf("bracket",     MyUdfs.class, "bracket");
session.registerUdf("is_bot",      MyUdfs.class, "isBot");
```

### Step 3: Use in SQL

```java
// SELECT with UDFs
DataFrame df = session.source(Source.sql(
    "SELECT n_name, "
    + "clean_email('USER@' || n_name || '.com') AS clean, "
    + "email_score('admin@' || LOWER(n_name) || '.com') AS score, "
    + "bracket(n_name, 2) AS padded "
    + "FROM tpch.tiny.nation ORDER BY n_nationkey LIMIT 5"));

// UDF in WHERE clause
DataFrame filtered = session.source(Source.sql(
    "SELECT n_name FROM tpch.tiny.nation "
    + "WHERE email_score('admin@' || LOWER(n_name) || '.com') > 100"));

// UDF via DataFrame API
DataFrame result = session.source(Source.sql("SELECT n_name FROM tpch.tiny.nation"))
    .withColumn("padded", "bracket(n_name, 1)")
    .withColumn("cleaned", "clean_email('user@' || n_name || '.com')")
    .filter("email_score('a@' || LOWER(n_name) || '.com') > 50")
    .limit(5);
```

### Inline Lambda UDF

For quick one-off UDFs, use a lambda (requires `Udf1`/`Udf2`/`Udf3` functional interface):

```java
import com.cloudcheflabs.ontul.sql.udf.Udf1;

session.registerUdf("upper_pad",
    (Udf1<String, String>) s -> s == null ? null : "[" + s.toUpperCase() + "]");

DataFrame df = session.source(Source.sql(
    "SELECT upper_pad(n_name) AS p FROM tpch.tiny.nation LIMIT 3"));
```

### Global UDF (IAM-Controlled)

Global UDFs are visible to all users. Registration requires `UDF:CREATE_GLOBAL` permission.

```java
// Admin registers a global UDF
session.registerGlobalUdf("global_upper", MyUdfs.class, "cleanEmail");

// Any authenticated user with UDF:EXECUTE policy can call it
DataFrame df = session.source(Source.sql(
    "SELECT global_upper(n_name) FROM tpch.tiny.nation LIMIT 5"));

// Admin drops it
session.unregisterGlobalUdf("global_upper");
```

### Unregister UDF

```java
session.unregisterUdf("clean_email");

// After unregister, SQL will fail with "function not found"
```

### SHOW FUNCTIONS

```java
// List all registered UDFs (name, args, return_type, language, scope)
DataFrame df = session.source(Source.sql("SHOW FUNCTIONS"));
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
