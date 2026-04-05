# Connector Architecture

Ontul is a processing engine, not a storage engine. Data lives in external systems and is accessed through a plugin-based connector architecture. This design separates three distinct concerns: physical connections, table metadata, and data serialization.

## Three-Layer Design

### Connectors (Physical Connections)

Connectors represent physical connections to external systems. Credentials are stored in the encrypted ConnectionStore.

- **S3**: endpoint, access key, secret key, region, path style
- **JDBC**: URL, username, password
- **Kafka**: bootstrap servers, consumer/producer properties

### Catalogs (Table Metadata)

Catalogs manage table metadata and discovery. Each catalog references a Connector by connection ID.

- **Iceberg**: REST catalog (e.g., Polaris, Nessie) — references an S3 connector for storage
- **JDBC**: References a JDBC connector — discovers tables from the database schema
- **TPC-H / TPC-DS**: Built-in benchmark data generators (no external connection needed)

### Formats (Data Serialization)

Formats define how data is read from and written to files:

- Parquet, ORC, CSV, JSON, Avro

## Dynamic Catalog Registration

Catalogs can be registered and unregistered at runtime through the Admin API or Admin UI without restarting the cluster.

```
POST /admin/catalogs
{
  "name": "my_iceberg",
  "connector": "iceberg",
  "connectionId": "my-s3-conn",
  "config": {
    "catalog.type": "rest",
    "catalog.uri": "http://polaris:8181/api/catalog"
  }
}
```

Once registered, tables are immediately queryable via fully qualified names: `my_iceberg.db.my_table`.

## Built-in Connectors

| Catalog | Read | Write | Streaming | Description |
|---------|------|-------|-----------|-------------|
| Iceberg | O | O | O | Open table format with maintenance automation |
| JDBC | O | | | Any JDBC database with connection pooling (HikariCP) |
| TPC-H | O | | | Built-in benchmark data (configurable scale) |
| TPC-DS | O | | | Built-in benchmark data (configurable scale) |

## Plugin Connectors

External connectors can be loaded as plugins via `URLClassLoader` with full dependency isolation. Each plugin JAR is loaded in its own classloader, preventing dependency conflicts between connectors.

## Connection Management

Connections are managed through the Admin UI or REST API:

- Create, update, delete, and list connections
- Credentials are encrypted at rest via KMS (envelope encryption)
- Jobs and catalogs reference connections by ID — no credentials in code or configuration files
