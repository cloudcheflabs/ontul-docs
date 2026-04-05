# Iceberg Integration

Ontul integrates with Apache Iceberg as a first-class catalog, supporting both read and write operations with full table management and automated maintenance.

## Catalog Type

Ontul supports the Iceberg REST Catalog, connecting to REST catalog servers such as Polaris or Nessie with OAuth2 or static token authentication.

## Read Operations

- Query Iceberg tables using standard SQL with fully qualified names (`iceberg_catalog.schema.table`)
- Snapshot isolation — queries read from a consistent snapshot
- Schema evolution detection — Ontul automatically detects and reflects column changes
- Parquet and ORC data file formats supported

## Write Operations

- **INSERT INTO**: Append data to existing Iceberg tables
- **CREATE TABLE AS SELECT (CTAS)**: Create new Iceberg tables from query results
- **MERGE INTO**: Upsert operations using Iceberg's OverwriteFiles API
- **Streaming writes**: Continuous ingestion from Kafka or other streaming sources directly into Iceberg tables
- Tables are auto-created if they don't exist when writing via the SDK

## Automated Maintenance

Ontul includes a built-in Iceberg maintenance service that runs scheduled tasks:

- **Expire Snapshots**: Remove old snapshots beyond the configured retention period (default 7 days)
- **Rewrite Data Files**: Compact small files into larger ones for better read performance
- **Rewrite Manifests**: Optimize manifest files for faster query planning
- **Remove Orphan Files**: Clean up data files that are no longer referenced by any snapshot

Maintenance can be configured per table through the Admin UI or triggered manually via `ALTER TABLE EXECUTE` SQL commands.

## S3-Compatible Storage

Iceberg data files are stored in S3-compatible object storage. Ontul supports AWS S3, MinIO, and other S3-compatible services, configured through the ConnectionStore.
