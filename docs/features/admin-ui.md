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

Configure and monitor Iceberg table maintenance — snapshot expiration, data compaction, manifest rewrite, orphan-file cleanup, and position-delete consolidation. Per-table configuration covers per-operation toggles plus Spark-aligned parameters (target file size, compaction `window_hours` and `min_input_files`, snapshot retention and `retain_last`, orphan safety window) and a **schedule** that is either a fixed interval or a 5-field UNIX **cron** (e.g. `0 */2 * * *`) which overrides the interval. A **Manual Trigger** runs any single operation (or all) on demand against a wildcard table pattern, with full job history.

### Worker Dashboard

Per-worker metrics with auto-refresh: heap usage, active tasks, and performance indicators.

### Backup & Restore

Backup the cluster's KMS / IAM / metadata stores to S3 and restore from any prior backup. The page exposes manual *Backup Now*, a fixed-interval schedule, and a 5-field UNIX **cron schedule** (e.g. `0 2 * * *`); the next-fire timestamp is shown next to an active cron so you can see at a glance when the next run will land. See [Backup & Restore](backup-restore.md) for the full flow.

## REST API

All operations available in the Admin UI are also accessible via the REST API, enabling automation and integration with external tools. The full API is documented in OpenAPI 3.0 format.

## Prometheus Metrics

Ontul exposes metrics at `GET /metrics` in Prometheus text format:

- `ontul_queries_total` / `ontul_queries_failed` / `ontul_queries_active`
- `ontul_latency_ms_sum`
- `ontul_workers_total` / `ontul_workers_ready`
- JVM heap and thread metrics
