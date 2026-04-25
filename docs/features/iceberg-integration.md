# Iceberg Integration

Ontul integrates with Apache Iceberg as a first-class catalog, supporting both read and write operations with full table management, automated maintenance, and streaming ingestion.

## Catalog Type

Ontul supports the Iceberg **REST Catalog only**, connecting to REST catalog servers such as Polaris with OAuth2 authentication. S3 storage is accessed directly using credentials from the catalog configuration (no credential vending/STS dependency).

Multiple Iceberg catalogs can be registered simultaneously, each with independent REST catalog and S3 configurations.

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

## DDL Operations

- **CREATE SCHEMA IF NOT EXISTS**: Create Iceberg namespaces
- **CREATE TABLE IF NOT EXISTS**: Create tables with Arrow-to-Iceberg type mapping
- **DROP TABLE IF EXISTS**: Drop tables with metadata cleanup
- **DROP SCHEMA IF EXISTS**: Drop namespaces

## Read Operations

- Query Iceberg tables using standard SQL with fully qualified names (`iceberg.schema.table`)
- Snapshot isolation — queries read from a consistent snapshot
- Schema evolution detection — Ontul automatically detects and reflects column changes
- Parquet data file format

## Write Operations

- **INSERT INTO**: Append data to existing Iceberg tables
- **CREATE TABLE AS SELECT (CTAS)**: Create new Iceberg tables from query results (with immediate catalog refresh)
- **MERGE INTO**: Upsert operations using Iceberg's OverwriteFiles API
- **Streaming writes**: Continuous ingestion from Kafka directly into Iceberg tables with exactly-once semantics (barrier checkpoint + snapshot isolation)
- Tables are auto-created if they don't exist when writing via the SDK

## NeorunBase Iceberg CDC

NeorunBase supports Iceberg CDC (Change Data Capture), automatically synchronizing NeorunBase table changes to Iceberg tables via Polaris. Ontul can query this CDC data:

```sql
-- NeorunBase CDC data accessible via Iceberg catalog
SELECT * FROM iceberg.public.cdc_table
```

## Automated Maintenance

Ontul includes a built-in Iceberg maintenance service that runs scheduled tasks:

- **Expire Snapshots**: Remove old snapshots beyond the configured retention period (default 7 days)
- **Rewrite Data Files**: Compact small files into larger ones for better read performance
- **Rewrite Manifests**: Optimize manifest files for faster query planning
- **Remove Orphan Files**: Clean up data files that are no longer referenced by any snapshot

Maintenance can be configured per table through the Admin UI or triggered manually via `ALTER TABLE EXECUTE` SQL commands.

## S3-Compatible Storage

Iceberg data files are stored in S3-compatible object storage. Ontul supports ShannonStore, AWS S3, MinIO, and other S3-compatible services. S3 credentials are part of the catalog configuration — Polaris is used for metadata only (no credential vending).
