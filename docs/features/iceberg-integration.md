# Iceberg Integration

Ontul integrates with Apache Iceberg as a first-class, **format-version 2 native** catalog: every
write is a v2 snapshot, with hidden partitioning, position-delete `DELETE`, RowDelta `UPDATE`/`MERGE`,
schema and partition evolution, snapshot rollback, branches/tags, time travel, and Iceberg-spec views.
The data path (Parquet read/write, field-id resolution, partition transforms, predicate evaluation,
snapshot commits) is Ontul code distributed across workers; the Iceberg JAR is used only as a metadata
library, and Polaris-side credential vending is bypassed so Ontul IAM stays the policy boundary.

Full documentation now lives in the dedicated **[Iceberg Integration](../iceberg/integration.md)**
section:

- **[Overview](../iceberg/integration.md)** — REST catalog setup (Polaris / AWS Glue), DDL/DML, file
  formats, time travel, views, federation, and access control.
- **[Table Maintenance](../iceberg/maintenance.md)** — on-demand `ALTER TABLE … EXECUTE` procedures
  (`optimize`, `expire_snapshots`, `remove_orphan_files`, `rollback_to_timestamp`, …) and the
  cron-schedulable auto-maintenance service.
- **[Write-Audit-Publish (WAP)](../iceberg/wap.md)** — stage writes on a branch, audit, then publish
  to `main`.
