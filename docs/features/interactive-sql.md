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
- **Queries**: `JOIN` (INNER, LEFT, RIGHT, FULL — one equality in `ON`), `GROUP BY`, `ORDER BY`, `LIMIT`, `HAVING`, `SELECT DISTINCT`, CTEs (`WITH`) and derived tables, `CASE WHEN`, `COALESCE`, `NULLIF`, `CAST`, `LIKE`, `IN`, `BETWEEN`, `IS NULL`
- **Aggregations**: `COUNT(*)`, `COUNT(col)`, `SUM`, `AVG`, `MIN`, `MAX` (MIN/MAX also over text and dates)
- **Not executable**: window functions (`OVER`), subqueries in `SELECT`/`WHERE`, `INTERSECT` / `EXCEPT` / de-duplicating `UNION`, `COUNT(DISTINCT)`, `OFFSET`, composite/non-equi join conditions. These are rejected at plan time with an explicit error rather than returning a wrong answer — see [SQL Capability Gate](sql-capability-gate.md) for the full list and the rewrite for each.
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
