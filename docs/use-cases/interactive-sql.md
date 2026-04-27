# Interactive SQL: Iceberg via JDBC

This tutorial demonstrates interactive SQL queries on Iceberg tables using Ontul's Arrow Flight SQL endpoint.

**What you will learn:**

- Connect to Ontul via JDBC (Arrow Flight SQL)
- DDL: CREATE TABLE, SHOW CATALOGS/TABLES, DESCRIBE
- DML: INSERT INTO VALUES, SELECT, UPDATE
- Aggregation with GROUP BY, AVG, SUM, COUNT
- Cross-catalog federation queries
- CTAS (Create Table As Select)
- EXPLAIN query plan

## Prerequisites

- Java 17
- Docker (for MinIO + Polaris)
- Ontul cluster running with Iceberg catalog registered (see [Batch ETL](batch-etl.md) for setup)

## 1. Setup

Follow the [Installation Guide](../installation/installation.md) to download and set up Ontul, then:

```bash
cd ontul-1.0.0-SNAPSHOT

# Start MinIO + Polaris + Kafka
docker compose -f examples/docker-compose-iceberg.yml up -d

# Start Ontul cluster + register catalogs
examples/bin/setup-iceberg.sh
```

## 2. Run the Interactive SQL Example

```bash
examples/bin/run-iceberg-query.sh
```

## JDBC Connection

Ontul exposes an Arrow Flight SQL endpoint on port `47470`. Any JDBC client (DBeaver, DataGrip, etc.) can connect:

```
JDBC URL: jdbc:arrow-flight-sql://localhost:47470
Driver: org.apache.arrow.driver.jdbc.ArrowFlightJdbcDriver
```

---

## Example Code (Java)

The full example is in `examples/java/IcebergQueryExample.java`.

### Connect via JDBC

```java
String JDBC_URL = "jdbc:arrow-flight-sql://localhost:47470";

Properties props = new Properties();
props.put("user", "Token " + token);
props.put("password", "");
props.put("useEncryption", "false");

Class.forName("org.apache.arrow.driver.jdbc.ArrowFlightJdbcDriver");
Connection conn = DriverManager.getConnection(JDBC_URL, props);
```

### SHOW CATALOGS

```sql
SHOW CATALOGS
```

```
Catalog
-------
tpcds
tpch
ice
```

### CREATE TABLE (DDL)

Create an Iceberg table with explicit schema:

```sql
CREATE TABLE ice.examples.products (
    product_id INT,
    product_name VARCHAR,
    category VARCHAR,
    price DOUBLE,
    stock INT
)
```

### SHOW TABLES / DESCRIBE

```sql
SHOW TABLES IN ice.examples

DESCRIBE ice.examples.products
```

```
Column        | Type                   | Nullable
--------------+------------------------+---------
product_id    | Int(32, true)          | YES
product_name  | Utf8                   | YES
category      | Utf8                   | YES
price         | FloatingPoint(DOUBLE)  | YES
stock         | Int(32, true)          | YES
```

### INSERT INTO VALUES

```sql
INSERT INTO ice.examples.products VALUES
    (1, 'Laptop', 'Electronics', 1299.99, 50),
    (2, 'Mouse', 'Electronics', 29.99, 200),
    (3, 'Desk', 'Furniture', 399.99, 30),
    (4, 'Chair', 'Furniture', 249.99, 45),
    (5, 'Monitor', 'Electronics', 599.99, 75),
    (6, 'Keyboard', 'Electronics', 79.99, 150),
    (7, 'Bookshelf', 'Furniture', 189.99, 20),
    (8, 'Headphones', 'Electronics', 149.99, 100),
    (9, 'Lamp', 'Furniture', 59.99, 80),
    (10, 'Webcam', 'Electronics', 89.99, 60)
```

### SELECT

```sql
SELECT * FROM ice.examples.products ORDER BY product_id
```

```
product_id | product_name | category    | price   | stock
-----------+--------------+-------------+---------+------
1          | Laptop       | Electronics | 1299.99 | 50
2          | Mouse        | Electronics |   29.99 | 200
3          | Desk         | Furniture   |  399.99 | 30
...
```

### Aggregation

```sql
SELECT category,
       COUNT(*) AS product_count,
       CAST(AVG(price) AS DECIMAL(10,2)) AS avg_price,
       SUM(stock) AS total_stock
FROM ice.examples.products
GROUP BY category
ORDER BY product_count DESC
```

```
category    | product_count | avg_price | total_stock
------------+---------------+-----------+------------
Electronics | 6             | 374.99    | 635
Furniture   | 4             | 224.99    | 175
```

### Cross-Catalog Federation

Query Iceberg tables joined with TPC-H data in a single SQL:

```sql
SELECT n.n_name AS nation, r.r_name AS region
FROM ice.examples.nations n
JOIN tpch.tiny.region r ON n.n_regionkey = r.r_regionkey
ORDER BY r.r_name, n.n_name
LIMIT 10
```

### CTAS from Aggregation

```sql
CREATE TABLE ice.examples.category_summary AS
SELECT category,
       COUNT(*) AS cnt,
       CAST(SUM(price * stock) AS DECIMAL(15,2)) AS total_value
FROM ice.examples.products
GROUP BY category
```

### EXPLAIN

```sql
EXPLAIN SELECT product_name, price
FROM ice.examples.products
WHERE category = 'Electronics'
ORDER BY price DESC
```

Returns the physical execution plan as JSON showing SCAN, FILTER, PROJECT, and SORT operators.

---

## Python Version

The same example is available in Python: `examples/python/iceberg_query_example.py`

```bash
examples/bin/run-iceberg-query-python.sh
```

```python
from ontul.session import OntulSession

session = OntulSession()

# Execute SQL and get Pandas DataFrame
table = session.source("SELECT * FROM ice.examples.products ORDER BY product_id")
print(table.to_pandas())

# DDL
session.execute("CREATE TABLE ice.examples.products_pyq (product_id INT, ...)")

# DML
session.execute("INSERT INTO ice.examples.products_pyq VALUES (1, 'Laptop', ...)")

# Aggregation
table = session.source(
    "SELECT category, COUNT(*) AS cnt FROM ice.examples.products_pyq GROUP BY category")
print(table.to_pandas())
```

---

## Cleanup

```bash
bin/stop-example-servers.sh
docker compose -f examples/docker-compose-iceberg.yml down -v
```
