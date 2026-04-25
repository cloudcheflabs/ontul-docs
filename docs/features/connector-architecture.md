# Connector Architecture

Ontul is a processing engine, not a storage engine. Data lives in external systems and is accessed through a plugin-based connector architecture. This design separates three distinct concerns: physical connections, table metadata, and data serialization.

## Three-Layer Design

### Connectors (Physical Connections)

Connectors represent physical connections to external systems. Credentials are stored in the encrypted ConnectionStore.

- **S3**: endpoint, access key, secret key, region, path style
- **JDBC**: URL, username, password
- **Kafka**: bootstrap servers, consumer/producer properties
- **NeorunBase**: REST API endpoint + JDBC (pg-wire)

### Catalogs (Table Metadata)

Catalogs manage table metadata and discovery. Each catalog references a Connector by connection ID.

- **Iceberg**: REST catalog (Polaris) — S3 direct access for data, Polaris for metadata
- **NeorunBase**: REST API for table discovery, JDBC (pg-wire) for reads, REST bulk-insert for writes
- **JDBC**: References a JDBC connector — discovers tables from the database schema
- **TPC-H / TPC-DS**: Built-in benchmark data generators (no external connection needed)

### Formats (Data Serialization)

Formats define how data is read from and written to files:

- Parquet, ORC, CSV, JSON, Avro

## Dynamic Catalog Registration

Catalogs can be registered and unregistered at runtime through the Admin API or Admin UI without restarting the cluster.

**Iceberg catalog:**
```json
POST /admin/catalogs
{
  "name": "iceberg",
  "config": {
    "connector": "iceberg",
    "catalog-type": "rest",
    "uri": "http://polaris:8181/api/catalog",
    "warehouse": "my_catalog",
    "credential": "root:secret",
    "scope": "PRINCIPAL_ROLE:ALL",
    "io-impl": "org.apache.iceberg.aws.s3.S3FileIO",
    "s3.endpoint": "http://s3-host:9000",
    "s3.path-style-access": "true",
    "s3.accessKey": "ACCESS_KEY",
    "s3.secretKey": "SECRET_KEY"
  }
}
```

**NeorunBase catalog:**
```json
POST /admin/catalogs
{
  "name": "neorun",
  "config": {
    "connector": "neorunbase",
    "endpoint": "http://neorunbase-host:8080",
    "jdbcUrl": "jdbc:postgresql://neorunbase-host:5432/neorunbase?preferQueryMode=simple",
    "username": "admin",
    "password": "password"
  }
}
```

Once registered, tables are immediately queryable via fully qualified names: `iceberg.db.my_table`, `neorun.public.events`.

## Built-in Connectors

| Catalog | Read | Write | Streaming Sink | Description |
|---------|------|-------|----------------|-------------|
| Iceberg | O | O | O | Open table format, REST catalog (Polaris), S3 direct access |
| NeorunBase | O | O | O | REST bulk-insert sink (high throughput), JDBC pg-wire source |
| JDBC | O | O | O | Any JDBC database with connection pooling (HikariCP) |
| Kafka | | | O | Kafka producer sink (transactional mode supported) |
| Elasticsearch | | | O | Bulk API sink with optional upsert |
| HTTP | | | O | JSON POST to REST endpoint |
| TPC-H | O | | | Built-in benchmark data (configurable scale) |
| TPC-DS | O | | | Built-in benchmark data (configurable scale) |

## Cross-Engine Federation

Multiple catalogs can be joined in a single query:

```sql
SELECT n.name, c.c_name
FROM neorun.public.events n
JOIN tpch.tiny.customer c ON n.id = c.c_custkey
```

## Plugin Connectors

External connectors can be loaded as plugins via `URLClassLoader` with full dependency isolation. Each plugin JAR is loaded in its own classloader, preventing dependency conflicts between connectors.

## Connection Management

Connections are managed through the Admin UI or REST API:

- Create, update, delete, and list connections
- Credentials are encrypted at rest via KMS (envelope encryption)
- Jobs and catalogs reference connections by ID — no credentials in code or configuration files
