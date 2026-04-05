# Interactive SQL

Ontul provides an interactive SQL query engine with federation across multiple data sources. Clients connect via Arrow Flight SQL and execute standard SQL queries against any registered catalog.

## Arrow Flight SQL Interface

Ontul exposes an Arrow Flight SQL endpoint (default port 47470) that supports:

- **JDBC drivers**: Connect from DBeaver, DataGrip, or any JDBC-compatible tool using the Arrow Flight SQL JDBC driver
- **Python**: Connect via `pyarrow.flight` or the Ontul Python SDK
- **Programmatic clients**: Any Arrow Flight SQL client library (Java, Go, Rust, Node.js, etc.)

## SQL Support

Ontul uses Apache Calcite for SQL parsing and query planning, supporting:

- **DML**: `SELECT`, `INSERT INTO`, `UPDATE`, `DELETE`, `MERGE INTO`, `CREATE TABLE AS SELECT`
- **DDL**: `CREATE TABLE`, `DROP TABLE`, `ALTER TABLE`, `CREATE VIEW`, `DROP VIEW`, `CREATE SCHEMA`, `DROP SCHEMA`
- **Queries**: `JOIN` (INNER, LEFT, RIGHT, FULL), `GROUP BY`, `ORDER BY`, `LIMIT`, `HAVING`, subqueries, CTEs (`WITH`), window functions, `CASE WHEN`, `CAST`, `LIKE`, `IS NULL`
- **Aggregations**: `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`
- **Window Functions**: `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `COUNT`, `SUM`, `AVG`, `MIN`, `MAX` with `PARTITION BY` and `ORDER BY`
- **Metadata**: `SHOW CATALOGS`, `SHOW SCHEMAS`, `SHOW TABLES`, `DESCRIBE`, `EXPLAIN`
- **Transactions**: `BEGIN`, `COMMIT`, `ROLLBACK`
- **Session**: `SET` session variables

## Federation Queries

Ontul supports cross-catalog queries using fully qualified table names (`catalog.schema.table`). A single SQL query can join data across different data sources — for example, joining an Iceberg table with a JDBC database table.

## Query Optimization

- **Predicate Pushdown**: Filters are pushed down to connectors, reducing the amount of data read from sources
- **Plan Caching**: Execution plans are cached (SHA-256 based LRU) to avoid re-planning repeated queries
- **Split Pruning**: Only relevant data splits are assigned to Workers based on query predicates
- **Query Resource Management**: Global and per-user concurrency control with timeout enforcement

## Distributed Execution

Queries are automatically distributed across the cluster:

1. The Master parses and plans the query
2. Data splits are resolved from connectors and assigned to Workers
3. Each Worker executes its portion of the plan independently
4. Workers communicate directly with each other for shuffles (no Master bottleneck)
5. Results stream back to the client via Arrow Flight
