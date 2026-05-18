# Data Lineage

Ontul automatically tracks **data lineage** — which tables and columns a table was derived from — at both table and column granularity. Lineage is extracted from the actual Calcite query plan, not from heuristic SQL parsing, so joins, sub-queries, CTEs, and aggregations are resolved accurately.

## Automatic Capture

When a table is created or populated, Ontul records its lineage — no configuration, no annotations. Capture happens on:

- `CREATE TABLE ... AS SELECT` (CTAS)
- `INSERT INTO ... SELECT`
- `MERGE INTO`
- `CREATE VIEW`

Capture is **best-effort**: it runs after the statement succeeds and never blocks or fails the query.

## Table-Level Lineage

Table-level lineage records the set of source tables a table was built from — every table scanned by the defining query, across joins and unions.

```sql
CREATE TABLE iceberg_catalog.demo.cust_orders AS
SELECT c.c_name,
       count(o.o_orderkey) AS order_count,
       sum(o.o_totalprice) AS total_spend
FROM tpch.tiny.customer c
JOIN tpch.tiny.orders o ON c.c_custkey = o.o_custkey
GROUP BY c.c_name;
```

```sql
SHOW LINEAGE iceberg_catalog.demo.cust_orders;
-- tpch.tiny.customer
-- tpch.tiny.orders
```

## Column-Level Lineage

For each output column, Ontul records the source `(table, column)` it derives from. A column that passed through an expression — `SUM(...)`, `CAST(...)`, an aggregate — is flagged **`derived`**; a straight projection of a base column is not. Column origins come from Calcite column-origin metadata, so they are tracked correctly through joins and `GROUP BY`.

```sql
SHOW LINEAGE COLUMNS iceberg_catalog.demo.cust_orders;
-- c_name       ←  tpch.tiny.customer.c_name
-- order_count  ←  tpch.tiny.orders.o_orderkey      [derived]
-- total_spend  ←  tpch.tiny.orders.o_totalprice    [derived]
```

## SQL Commands

| Command | Result |
|---|---|
| `SHOW LINEAGE <table>` | Upstream source tables |
| `SHOW LINEAGE COLUMNS <table>` | Column-level lineage edges |
| `SHOW LINEAGE DOWNSTREAM <table>` | Tables built from this one |

## REST API

Read-only lineage endpoints (any authenticated user):

| Endpoint | Returns |
|---|---|
| `GET /admin/lineage/graph?root=<table>&depth=<n>&direction=<up\|down\|both>` | Lineage graph `{nodes, edges}` |
| `GET /admin/lineage/columns?table=<table>` | Column-level lineage |
| `GET /admin/lineage/impact?table=<table>` | Transitive downstream tables |
| `GET /admin/lineage/pii-flow?table=<table>&column=<col>` | Downstream flow of a column |

## Admin UI

The **Data Lineage** page renders an interactive graph (sources on the left, derived tables on the right). Enter a table, choose direction and depth, and trace its lineage. Click any node to open a side panel showing its column-level lineage — with the `derived` flag — and its downstream impact.

## PII Flow Detection

Given a sensitive column, Ontul traces every downstream `(table, column)` it transitively flows into, via column-level lineage:

```
GET /admin/lineage/pii-flow?table=pg.public.users&column=ssn
```

This answers "where did this PII go?" — for example, finding that `users.ssn` was copied by an ETL job into a table that has no access policy. Ontul **surfaces the flow for an operator to review**; it does not automatically modify IAM policies, since lineage is not exhaustive (opaque UDFs, dynamic SQL) and silent policy changes would be surprising.

## Cross-Catalog Lineage

Because every catalog's tables — Iceberg, JDBC, Kafka, TPC-H — are first-class scan sources, lineage **crosses systems automatically**. A table built `FROM jdbc_catalog.public.users JOIN kafka_catalog.events.clicks` records both as upstream sources, giving cross-system lineage with no extra configuration.

## OpenLineage Emission

Ontul can emit [OpenLineage](https://openlineage.io)-spec events so external catalogs (Marquez, DataHub, Atlan, …) ingest Ontul lineage through the industry-standard wire format.

Set `ontul.lineage.openlineage.url` to an HTTP endpoint; each CTAS/INSERT/MERGE/VIEW then emits a `COMPLETE` RunEvent with `inputs`, `outputs`, and a standard `columnLineage` facet. When unset, emission is disabled.

## Storage

Lineage is persisted in the leader's metadata store and replicated to followers — the same mechanism used for catalogs and IAM — so it survives restarts and leadership changes.
