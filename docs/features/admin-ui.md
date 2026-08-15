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

### Semantic Layer

Register and govern [semantic views](semantic-layer.md) — curated metrics and dimensions over a
base SQL view. Each entry shows its trust badge, its metrics (a shield icon marks role-gated
ones), dimensions, tags and mandatory filters. **Certify** signs a view off, stamping the logged-in
user and a fingerprint of its semantics; the badge turns **STALE** (orange) if the view is edited
afterwards, with the reason on hover.

### Retrievers

Manage [retrievers](retrievers.md) — governed multi-modal retrieval (vector / graph / full-text)
pushed down to a NeorunBase catalog. The side panel covers the SQL template, the typed parameter
contract, output columns, an optional re-rank block, synonyms and `allowedRoles`. An **Invoke**
drawer runs a retriever with structured arguments and shows the rendered SQL next to the rows.

### Object Types

The [ontology](ontology.md) entity layer: typed business objects (Customer, Order) mapped onto
physical tables. Each row shows the read source, the property→column bindings, the write target
(system of record) and a trust badge. **Certify** signs the definition off; the badge shows
**STALE** with a tooltip when the read source or a property binding changed after certification.

### Link Types

Typed relationships between object types, bound either to relational join keys (`JOIN`) or to a
native NeorunBase graph edge (`GRAPH`). A link's trust is capped by both endpoint object types, so
a certified link over a draft object type displays as DRAFT with the offending endpoint named.

### Action Types

Governed write-backs — the only way an agent changes data. Each action declares its target object
type, its mode (`DML` against the Iceberg system of record, or `OPERATION` calling a REST/ERP
connector), a typed parameter contract, and whether it requires approval. An **invoke tester**
drawer runs one with real arguments.

### Action Workflows

Multi-step Sagas composed of action types, authored as JSON or kiok-style YAML (importable and
exportable). On a step failure the completed steps are compensated in reverse order; the page
shows the run ledger.

### Ontology Graph

Visual type-graph of object types and the link types connecting them — the schema-level map an
agent traverses, rendered for humans.

### Flow

The visual [Ontul Flow](../streaming/flow.md) builder and operator console: a drag-and-drop
`source → transform → sink` canvas backed by an editable JSON/YAML spec (both directions stay in
sync). The list view shows every registered flow with its status, **Records**, **Rejected** and
**Lag** columns, plus Start / Stop / Delete. Opening a flow gives Spec, Data (preview rows from a
source or sink node), Metrics (live throughput sparkline, rejected count, source lag), Logs
(auto-scrolling tail) and History (past runs with duration and rejected counts). A running flow is
read-only — stop it to edit.

### Data Lineage

Dataset-level lineage graph built from executed DML and started flows: which sources feed which
targets, with the job or query that created each edge. See [Data Lineage](data-lineage.md).

### Audit

Search the [audit log](audit-log.md) by user, engine, table, action kind, free text and time
range. Denied attempts appear alongside allowed ones, so a refused certification or an IAM-blocked
write is visible here.

### Lance Maintenance

Compaction and cleanup for [Lance](lance-integration.md) datasets, mirroring the Iceberg
Maintenance page for the vector/AI format.

### Storage

Switch the exchange and job-log backends between local disk and S3 **at runtime, cluster-wide, with
no restart** — the change is persisted to the replicated metadata store and pushed to every node.

### Drivers

Upload JDBC driver JARs for federated catalogs. Drivers are stored durably and synced to every
worker, so a new driver does not require rebuilding an image.

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
