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
    "s3.endpoint": "http://s3-host:9000",
    "s3.path-style-access": "true",
    "s3.accessKey": "ACCESS_KEY",
    "s3.secretKey": "SECRET_KEY"
  }
}
```

If the REST catalog requires OAuth2 `client_credentials`, add `credential` and `scope` (Admin UI: "OAuth / Polaris auth (advanced)" toggle). Polaris-side table IAM is bypassed — Ontul IAM is the access-control boundary; see [Iceberg Integration](iceberg-integration.md#access-control).

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

### Catalog Connectors (Query/Batch)

Registered via `POST /admin/catalogs`. Used in SQL queries and batch pipelines.

| Connector | Read | Write | Description |
|-----------|------|-------|-------------|
| Iceberg | O | O | Open table format, REST catalog (Polaris), S3 direct access |
| NeorunBase | O | O | JDBC pg-wire read, REST bulk-insert write |
| JDBC | O | O | PostgreSQL, MySQL, etc. with HikariCP pooling |
| File | O | O | CSV, Parquet, JSON, Avro, ORC on S3 |
| TPC-H | O | | Built-in benchmark data (configurable scale) |
| TPC-DS | O | | Built-in benchmark data (configurable scale) |

### Streaming Sinks

Used in streaming jobs. Configured via job config, not catalog registration.

| Sink | Transactional | Description |
|------|:---:|-------------|
| Iceberg (table) | O | Parquet write + AppendFiles commit |
| Kafka | O | Producer with optional Kafka Transactions |
| JDBC | O | Batch INSERT with transaction commit |
| NeorunBase | O/X | JDBC mode (tx) or REST mode (at-least-once) |
| Elasticsearch | X | Bulk API with optional upsert |
| HTTP / Webhook | X | JSON POST to REST endpoint |
| Console | X | Debug output to stdout |

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

See [Connection ID (Source/Sink by Reference)](connection-id.md) for the full reference: registering connections, using IDs from Java/Python SDKs and the REST API, overriding stored properties, and rotation.
