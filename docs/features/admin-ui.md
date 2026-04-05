# Admin UI

Ontul includes a built-in web-based Admin UI for monitoring, managing, and operating the cluster.

## Pages

### Dashboard

Cluster overview with real-time metrics: query throughput, latency, active queries, worker status, and JVM heap usage.

### Topology

Visual overview of the cluster — active Masters and Workers with node status and health information.

### SQL Query

Built-in SQL editor with syntax highlighting, `Ctrl+Enter` execution, result table, and query history.

### Catalog Browser

Explore registered catalogs, schemas, tables, and columns. Preview table data directly from the UI.

### Catalogs

Register, unregister, and manage data source catalogs. View connector type, connection ID, table count, and configuration for each catalog.

### Connections

Manage physical connections (S3, JDBC, Kafka) — create, update, delete, and list. Credentials are encrypted at rest via KMS.

### Jobs

Monitor active and completed jobs. Submit new batch or streaming jobs, view real-time logs, and kill running jobs.

### IAM

Manage users, groups, and policies. Includes a visual policy editor for creating structured IAM policies with column-level and row-level security rules.

### KMS

Key management interface for viewing and managing encryption keys.

### Maintenance

Configure and monitor Iceberg table maintenance — snapshot expiration, data compaction, manifest rewrite. Per-table configuration with job history.

### Worker Dashboard

Per-worker metrics with auto-refresh: heap usage, active tasks, and performance indicators.

### Backup & Restore

Backup the cluster state (RocksDB checkpoint) to local storage or S3, and restore from backups.

## REST API

All operations available in the Admin UI are also accessible via the REST API, enabling automation and integration with external tools. The full API is documented in OpenAPI 3.0 format.

## Prometheus Metrics

Ontul exposes metrics at `GET /metrics` in Prometheus text format:

- `ontul_queries_total` / `ontul_queries_failed` / `ontul_queries_active`
- `ontul_latency_ms_sum`
- `ontul_workers_total` / `ontul_workers_ready`
- JVM heap and thread metrics
