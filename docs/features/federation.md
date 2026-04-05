# Federation Queries

Ontul supports federated queries across multiple data sources through its multi-catalog architecture powered by Apache Calcite.

## Multi-Catalog Architecture

Ontul uses a 3-level schema hierarchy for table resolution:

```
catalog.schema.table
```

Each catalog represents a separate data source. Multiple catalogs can be registered simultaneously, and a single SQL query can reference tables from different catalogs.

## Cross-Source Joins

Join data across different data sources in a single query:

```sql
SELECT c.name, o.total
FROM jdbc_catalog.public.customers c
INNER JOIN iceberg_catalog.sales.orders o
  ON c.id = o.customer_id
WHERE o.status = 'completed';
```

In this example, `customers` comes from a JDBC database and `orders` comes from an Iceberg table — Ontul handles the distributed execution transparently.

## Supported Catalog Types

| Catalog Type | Description |
|-------------|-------------|
| Iceberg | REST catalog (e.g., Polaris, Nessie) for Iceberg tables |
| JDBC | Any relational database accessible via JDBC |
| TPC-H / TPC-DS | Built-in benchmark data |

## Dynamic Registration

Catalogs are registered at runtime through the Admin API or Admin UI. No cluster restart is required. Once registered, all tables within the catalog are immediately discoverable and queryable.

## Schema Introspection

Standard metadata commands work across all catalog types:

```sql
SHOW CATALOGS;
SHOW SCHEMAS FROM my_catalog;
SHOW TABLES FROM my_catalog.my_schema;
DESCRIBE my_catalog.my_schema.my_table;
```

Ontul also provides `information_schema` virtual tables (`tables`, `columns`, `schemata`) for programmatic introspection.
