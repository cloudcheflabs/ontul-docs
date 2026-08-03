# Iceberg Metadata Tables

Ontul exposes an Iceberg table's internal metadata as **virtual metadata tables**, queryable with plain SQL by appending a `$<kind>` suffix to the table name (Trino/Spark convention). Use them to inspect commit history, diagnose small-file problems, trace write patterns, and find the snapshot id to roll back to — all without leaving SQL.

```sql
SELECT * FROM ice.sales."orders$snapshots";
```

Quote the table reference (`"orders$snapshots"`) so the `$` is taken literally.

## Available tables

| Table | What it shows |
| --- | --- |
| `$snapshots` | One row per snapshot — commit time, operation, and write statistics (see below). |
| `$history` | The snapshot log: `made_current_at`, `snapshot_id`, `parent_id`, `is_current_ancestor`. |
| `$refs` | Branches and tags — `name`, `type`, `snapshot_id`, retention settings. |
| `$manifests` | Manifest files of the current snapshot with per-manifest file/row counts. |
| `$files` | Data files of the current snapshot — `file_path`, `file_format`, `record_count`, `file_size_in_bytes`, column stats. |
| `$partitions` | Per-partition `record_count`, `file_count`, `total_data_file_size_in_bytes`. |
| `$metadata_files` | The `metadata.json` log with `table_uuid` — detects table **recreation** (Ontul-specific; see below). |

## `$snapshots` — write statistics

`$snapshots` carries the standard Iceberg summary fields **flattened into typed columns**, so stats queries need no map access or `CAST`:

| Column | Type | Summary key |
| --- | --- | --- |
| `committed_at` | timestamp | — |
| `snapshot_id`, `parent_id` | bigint | — |
| `operation` | varchar | `append` / `overwrite` / `delete` / `replace` |
| `manifest_list` | varchar | — |
| `summary` | varchar (JSON) | the full map, for the long tail of non-standard keys |
| `add_rec` / `del_rec` | bigint | added / deleted-records |
| `add_df` / `del_df` | bigint | added / deleted-data-files |
| `add_size` / `rm_size` | bigint | added-files-size / removed-files-size |
| `tot_rec` / `tot_df` / `tot_size` | bigint | total-records / total-data-files / total-files-size |
| `chg_part` | int | changed-partition-count |

### Snapshot trace

```sql
SELECT committed_at, operation,
       add_rec, del_rec, add_df, del_df,
       tot_rec, tot_df, tot_size, chg_part, snapshot_id
FROM ice.sales."orders$snapshots"
ORDER BY committed_at;
```

### Write-pattern shift — average added file size

Spot where writes changed shape (e.g. 100 MiB → 400 MiB files):

```sql
SELECT committed_at, operation,
       add_df AS files,
       add_size / NULLIF(add_df, 0) / 1048576 AS avg_file_mb
FROM ice.sales."orders$snapshots"
WHERE operation IN ('append', 'overwrite')
ORDER BY committed_at;
```

### Empty commits and compactions

```sql
-- Empty commits (state-preserving commits)
SELECT * FROM ice.sales."orders$snapshots"
WHERE COALESCE(add_df, 0) = 0 AND COALESCE(del_df, 0) = 0;

-- Compactions: files replaced but row count unchanged
SELECT committed_at, del_df AS removed, add_df AS added
FROM ice.sales."orders$snapshots"
WHERE operation = 'replace' AND add_rec = del_rec;
```

The raw `summary` JSON is still available for any key not flattened above:

```sql
SELECT snapshot_id, summary FROM ice.sales."orders$snapshots";
```

## `$files` — small-file diagnosis

Before running `ALTER TABLE … EXECUTE optimize`, size up the problem:

```sql
SELECT count(*) AS small_files,
       sum(file_size_in_bytes) / 1048576 AS mb
FROM ice.sales."orders$files"
WHERE file_size_in_bytes < 64 * 1024 * 1024;
```

## `$metadata_files` — table recreation / UUID generations

Iceberg's `table_uuid` changes when a table is **dropped and recreated at the same path** — the storage location is reused but it is a different logical table. No engine's `$snapshots` shows this, because `$snapshots` only reflects the *current* table. `$metadata_files` exposes the `metadata.json` log with each file's `table_uuid`, so a change in `table_uuid` marks a new **generation**:

| Column | Type | Meaning |
| --- | --- | --- |
| `metadata_file` | varchar | Path of the `metadata.json`. |
| `created_at` | timestamp | When that metadata file was written. |
| `table_uuid` | varchar | The table uuid recorded in that file (best-effort read; null if the file was expired/removed). |
| `is_latest` | boolean | Whether this is the current metadata file. |

```sql
-- Number each generation in time order: a new number appears whenever a new
-- table_uuid is first seen (i.e. the table was recreated at the same path).
SELECT metadata_file, created_at, table_uuid,
       DENSE_RANK() OVER (ORDER BY first_seen) AS generation
FROM (
  SELECT metadata_file, created_at, table_uuid,
         MIN(created_at) OVER (PARTITION BY table_uuid) AS first_seen
  FROM ice.sales."orders$metadata_files"
)
ORDER BY created_at;
```

A single `generation` value across all rows means the table was never recreated; two or more values mark recreation boundaries (the article's `g1` / `g2`).

!!! note
    `table_uuid` is read from each historical `metadata.json` on demand; files removed by `expire_snapshots` / `remove_orphan_files` return `null`. `is_latest = true` marks the one current metadata file.

## Related

- [Table Maintenance](maintenance.md) — `optimize`, `expire_snapshots`, `rewrite_manifests`, `remove_orphan_files`, `rollback_*`. Use `$snapshots` / `$files` to decide when to run them, and `$history` / `$refs` to find the snapshot id for a rollback.
- [Write-Audit-Publish](wap.md).
