# Query Guide

Ontul provides an interactive SQL query engine with cross-catalog federation. Clients connect via Arrow Flight SQL (port 47470) and execute standard SQL against any registered catalog.

## Connecting

### JDBC (DBeaver, DataGrip, Custom App)

```
JDBC URL:  jdbc:arrow-flight-sql://localhost:47470
Driver:    org.apache.arrow.driver.jdbc.ArrowFlightJdbcDriver
```

Java JDBC connection:

```java
Properties props = new Properties();
props.put("user", "Token " + jwtToken);
props.put("password", "");
props.put("useEncryption", "false");  // set "true" for TLS via Nginx

Class.forName("org.apache.arrow.driver.jdbc.ArrowFlightJdbcDriver");
Connection conn = DriverManager.getConnection(
    "jdbc:arrow-flight-sql://localhost:47470", props);
```

AccessKey authentication:

```java
props.put("user", "AccessKey " + accessKey + ":" + secretKey);
```

### Python SDK

```python
from ontul.session import OntulSession

session = OntulSession(host="localhost", port=47470, token="your-jwt-token")

# Execute and get Pandas DataFrame
df = session.sql_pandas("SELECT * FROM tpch.tiny.customer LIMIT 10")
print(df)
```

### Java SDK

```java
OntulSession session = OntulSession.builder()
    .master("localhost", 47470)
    .token("your-jwt-token")
    .build();

DataFrame result = session.source(Source.sql(
    "SELECT * FROM tpch.tiny.customer ORDER BY c_acctbal DESC LIMIT 10"));
result.show();
```

---

## Catalog System

Ontul uses a 3-level naming convention: `catalog.schema.table`.

### Registering Catalogs

Catalogs are registered via Admin REST API:

```bash
# TPC-H (built-in benchmark data)
curl -X POST http://localhost:8080/admin/catalogs \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"tpch","config":"{\"connector\":\"tpch\",\"scale.factor\":\"0.01\"}"}'

# Iceberg (via Polaris REST catalog + MinIO S3)
curl -X POST http://localhost:8080/admin/catalogs \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"ice","config":"{\"connector\":\"iceberg\",\"catalog.type\":\"rest\",\"catalog.rest.uri\":\"http://localhost:8181/api/catalog\",\"catalog.name\":\"ontul_iceberg\",\"catalog.rest.credential\":\"root:s3cr3t\",\"catalog.rest.scope\":\"PRINCIPAL_ROLE:ALL\",\"header.Polaris-Realm\":\"POLARIS\",\"catalog.warehouse\":\"ontul_iceberg\",\"s3.accessKey\":\"minioadmin\",\"s3.secretKey\":\"minioadmin\",\"s3.endpoint\":\"http://localhost:9000\",\"s3.pathStyle\":\"true\"}"}'
```

### Metadata Commands

```sql
-- List all registered catalogs
SHOW CATALOGS

-- List schemas in a catalog
SHOW SCHEMAS IN ice

-- List tables in a schema
SHOW TABLES IN ice.examples

-- Describe table structure
DESCRIBE ice.examples.products
```

Output:

```
Column        | Type                   | Nullable
--------------+------------------------+---------
product_id    | Int(32, true)          | YES
product_name  | Utf8                   | YES
category      | Utf8                   | YES
price         | FloatingPoint(DOUBLE)  | YES
stock         | Int(32, true)          | YES
```

---

## SQL Reference

### DDL

```sql
-- Create schema
CREATE SCHEMA IF NOT EXISTS ice.analytics

-- Create table
CREATE TABLE ice.examples.products (
    product_id INT,
    product_name VARCHAR,
    category VARCHAR,
    price DOUBLE,
    stock INT
)

-- Create Table As Select
CREATE TABLE ice.examples.category_summary AS
SELECT category, COUNT(*) AS cnt, SUM(price * stock) AS total_value
FROM ice.examples.products GROUP BY category

-- Drop
DROP TABLE ice.examples.products
DROP SCHEMA ice.analytics
```

### DML

```sql
-- Insert values
INSERT INTO ice.examples.products VALUES
    (1, 'Laptop', 'Electronics', 1299.99, 50),
    (2, 'Mouse',  'Electronics',   29.99, 200),
    (3, 'Desk',   'Furniture',    399.99, 30)

-- Insert from query
INSERT INTO ice.examples.orders
SELECT o_orderkey, o_custkey, o_totalprice FROM tpch.tiny.orders LIMIT 200

-- Merge (upsert)
MERGE INTO ice.examples.nations AS t
USING (SELECT * FROM tpch.tiny.nation WHERE n_regionkey = 0) AS s
ON t.n_nationkey = s.n_nationkey
WHEN MATCHED THEN UPDATE SET n_comment = 'updated'
WHEN NOT MATCHED THEN INSERT (n_nationkey, n_name, n_regionkey, n_comment)
VALUES (s.n_nationkey, s.n_name, s.n_regionkey, s.n_comment)
```

### Queries

```sql
-- Filter + order + limit
SELECT product_name, price, stock
FROM ice.examples.products
WHERE category = 'Electronics' AND price > 100
ORDER BY price DESC
LIMIT 10

-- Aggregation
SELECT category,
       COUNT(*) AS product_count,
       CAST(AVG(price) AS DECIMAL(10,2)) AS avg_price,
       SUM(stock) AS total_stock
FROM ice.examples.products
GROUP BY category
ORDER BY product_count DESC

-- Window function
SELECT product_name, category, price,
       ROW_NUMBER() OVER (PARTITION BY category ORDER BY price DESC) AS price_rank
FROM ice.examples.products
ORDER BY category, price_rank

-- Subquery
SELECT * FROM ice.examples.products
WHERE price > (SELECT AVG(price) FROM ice.examples.products)

-- CTE
WITH high_value AS (
    SELECT * FROM ice.examples.products WHERE price > 100
)
SELECT category, COUNT(*) AS cnt FROM high_value GROUP BY category
```

### Supported Syntax

| Category | Syntax |
|----------|--------|
| Joins | `INNER JOIN`, `LEFT JOIN`, `RIGHT JOIN`, `FULL OUTER JOIN` |
| Aggregations | `COUNT`, `SUM`, `AVG`, `MIN`, `MAX` |
| Window Functions | `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `COUNT`, `SUM`, `AVG` with `PARTITION BY` / `ORDER BY` |
| Predicates | `WHERE`, `HAVING`, `LIKE`, `IN`, `BETWEEN`, `IS NULL`, `CASE WHEN` |
| Set Operations | `UNION`, `UNION ALL`, `INTERSECT`, `EXCEPT` |
| Types | `CAST(expr AS type)`, `INT`, `BIGINT`, `DOUBLE`, `DECIMAL`, `VARCHAR`, `BOOLEAN`, `DATE`, `TIMESTAMP` |

### EXPLAIN

```sql
EXPLAIN SELECT product_name, price
FROM ice.examples.products
WHERE category = 'Electronics'
ORDER BY price DESC
```

Returns the physical execution plan showing SCAN, FILTER, PROJECT, and SORT operators.

---

## Cross-Catalog Federation

Join data across different catalogs in a single SQL query:

```sql
-- Iceberg + TPC-H federation
SELECT n.n_name AS nation, r.r_name AS region
FROM ice.examples.nations n
JOIN tpch.tiny.region r ON n.n_regionkey = r.r_regionkey
ORDER BY r.r_name, n.n_name
LIMIT 10
```

```sql
-- Iceberg orders + TPC-H customers
SELECT c.c_name, SUM(o.total_amount) AS total_spent
FROM ice.analytics.orders o
JOIN tpch.tiny.customer c ON o.customer_id = c.c_custkey
GROUP BY c.c_name
ORDER BY total_spent DESC
LIMIT 5
```

---

## REST API SQL Execution

SQL can also be executed via REST API without JDBC:

```bash
# Execute SQL directly
curl -X POST http://localhost:8080/v1/api/sql \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"sql": "SELECT * FROM tpch.tiny.nation ORDER BY n_nationkey LIMIT 5"}'

# Via Admin endpoint
curl -X POST http://localhost:8080/admin/query/execute \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"sql": "SHOW CATALOGS"}'
```

---

## Available Connectors

Queries can access data through 6 built-in connectors:

| Connector | Description | Example Catalog Config |
|-----------|-------------|----------------------|
| **Iceberg** | Apache Iceberg tables via REST catalog | `{"connector":"iceberg","catalog.type":"rest",...}` |
| **TPCH** | TPC-H benchmark data (built-in) | `{"connector":"tpch","scale.factor":"0.01"}` |
| **TPCDS** | TPC-DS benchmark data (built-in) | `{"connector":"tpcds","scale.factor":"0.01"}` |
| **NeorunBase** | JDBC read (pg-wire) + REST bulk-insert write | `{"connector":"neorunbase","jdbc.url":"..."}` |
| **JDBC** | PostgreSQL, MySQL, etc. | `{"connector":"jdbc","jdbc.url":"..."}` |
| **File** | CSV, Parquet, JSON, Avro, ORC on S3 | `{"connector":"file","base.path":"s3://..."}` |

---

## Query Configuration

Key configuration properties in `conf/ontul.properties`:

```properties
# Max concurrent queries across all users
ontul.query.max.global.concurrency=100

# Max concurrent queries per user
ontul.query.max.per.user.concurrency=10

# Query timeout (default 5 minutes)
ontul.query.default.timeout.ms=300000

# Max memory per query
ontul.query.max.memory.bytes=1073741824
```

---

## Query Optimization

| Optimization | Description |
|-------------|-------------|
| Predicate Pushdown | Filters pushed down to connectors — less data read from sources |
| Plan Caching | SHA-256 based LRU cache avoids re-planning repeated queries |
| Split Pruning | Only relevant data splits assigned to Workers based on predicates |
| Distributed Execution | Workers execute independently, shuffle directly peer-to-peer |

---

## Complete Example

### Setup

```bash
cd ontul-1.0.0-SNAPSHOT

# Start MinIO + Polaris + Kafka
docker compose -f examples/docker-compose-iceberg.yml up -d

# Start Ontul cluster + register catalogs
examples/bin/setup-iceberg.sh
```

### Run

```bash
examples/bin/run-iceberg-query.sh
```

### Java JDBC Code

```java
Connection conn = DriverManager.getConnection(JDBC_URL, props);

// 1. Show catalogs
query(conn, "SHOW CATALOGS");

// 2. Create table
exec(conn, "CREATE TABLE ice.examples.products (" +
    "product_id INT, product_name VARCHAR, category VARCHAR, " +
    "price DOUBLE, stock INT)");

// 3. Insert data
exec(conn, "INSERT INTO ice.examples.products VALUES " +
    "(1, 'Laptop', 'Electronics', 1299.99, 50), " +
    "(2, 'Mouse',  'Electronics',   29.99, 200), " +
    "(3, 'Desk',   'Furniture',    399.99, 30)");

// 4. Query
query(conn, "SELECT * FROM ice.examples.products ORDER BY product_id");

// 5. Aggregation
query(conn, "SELECT category, COUNT(*) AS cnt, " +
    "CAST(AVG(price) AS DECIMAL(10,2)) AS avg_price " +
    "FROM ice.examples.products GROUP BY category");

// 6. Window function
query(conn, "SELECT product_name, category, price, " +
    "ROW_NUMBER() OVER (PARTITION BY category ORDER BY price DESC) AS rank " +
    "FROM ice.examples.products");

// 7. Cross-catalog federation
query(conn, "SELECT n.n_name, r.r_name " +
    "FROM ice.examples.nations n " +
    "JOIN tpch.tiny.region r ON n.n_regionkey = r.r_regionkey " +
    "ORDER BY r.r_name LIMIT 10");

// 8. CTAS
exec(conn, "CREATE TABLE ice.examples.category_summary AS " +
    "SELECT category, COUNT(*) AS cnt FROM ice.examples.products GROUP BY category");

// 9. EXPLAIN
query(conn, "EXPLAIN SELECT * FROM ice.examples.products WHERE category = 'Electronics'");
```

### Python Code

```python
from ontul.session import OntulSession

session = OntulSession()

# DDL
session.execute("CREATE TABLE ice.examples.products_py (product_id INT, product_name VARCHAR, category VARCHAR, price DOUBLE, stock INT)")

# DML
session.execute("INSERT INTO ice.examples.products_py VALUES (1, 'Laptop', 'Electronics', 1299.99, 50)")

# Query → Pandas
df = session.sql_pandas("SELECT * FROM ice.examples.products_py ORDER BY product_id")
print(df)

# Aggregation
df = session.sql_pandas("SELECT category, COUNT(*) AS cnt FROM ice.examples.products_py GROUP BY category")
print(df)

# Cross-catalog
df = session.sql_pandas(
    "SELECT n.n_name, r.r_name FROM tpch.tiny.nation n "
    "JOIN tpch.tiny.region r ON n.n_regionkey = r.r_regionkey LIMIT 10")
print(df)
```

### Admin UI

SQL queries can also be executed from the built-in Admin UI SQL Runner at `http://localhost:8080/admin/`.

---

## Cleanup

```bash
bin/stop-example-servers.sh
docker compose -f examples/docker-compose-iceberg.yml down -v
```
