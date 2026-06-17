# Iceberg Table Maintenance

Ontul keeps Iceberg tables healthy with the same maintenance operations as Apache Spark — small-file
compaction, snapshot expiry, manifest rewrite, orphan-file cleanup, position-delete consolidation, and
rollback — available both as **on-demand SQL procedures** and as a **scheduled auto-maintenance service**
(configurable per table from the Admin UI, including a UNIX cron schedule). Parameter names mirror the
[Apache Iceberg Spark procedures](https://iceberg.apache.org/docs/latest/spark-procedures/).

## Auto-maintenance & scheduling

A built-in maintenance service runs periodic tasks per registered Iceberg table:

- **Expire snapshots** beyond a retention window (default 7 days), optionally keeping the last *N* (`retain_last`)
- **Rewrite data files** — small-file compaction, delete-aware; optionally windowed (`window_hours`) and gated by `min_input_files`
- **Rewrite manifests** for faster planning
- **Remove orphan files** that no live snapshot references (with a `dry_run` preview)
- **Rewrite position-delete files** — consolidate v2 delete files

The same operations are available as on-demand SQL (see [procedure reference](#procedure-reference) for the full per-procedure parameter list and examples):

```sql
ALTER TABLE ice.ns.t EXECUTE optimize(file_size_threshold_mb => 128, min_input_files => 5);
ALTER TABLE ice.ns.t EXECUTE expire_snapshots(retain_last => 100);
ALTER TABLE ice.ns.t EXECUTE remove_orphan_files(dry_run => true);
```

**Auto-maintenance (Admin UI → Iceberg Maintenance).** Each table has a config — per-operation
toggles, target file size, `window_hours`, `min_input_files`, snapshot retention + `retain_last`,
the orphan safety window, and the compaction **concurrency-safety** controls (window cooldown
minutes, sequence margin, skip-active-partitions, max commit retries) — plus a **schedule**: either
a fixed interval (hours) or, when set, a 5-field **UNIX cron** expression that overrides the interval
(e.g. `0 */2 * * *` = every 2 hours). This makes incremental, scheduled compaction practical for
streaming tables: e.g. a cron of `0 * * * *` with `window_hours => 1` compacts just the last hour's
small files every hour. Unlike ad-hoc `EXECUTE optimize` (safety filters OFF by default), scheduled
auto-maintenance applies **safe defaults** (cooldown 10 min, seq_margin 1, skip-active on, retries 3)
so it never conflicts with in-flight writes — see
[Running compaction alongside live writes](#running-compaction-alongside-live-writes). The same
finer-grained parameters can be set optionally per table, and a **Manual Trigger** runs any single
operation (or all) on demand against a wildcard table pattern.

For tests and quick teardown, set `polaris.config.drop-with-purge.enabled=true` on the Polaris catalog so `DROP TABLE` actually deletes the underlying S3 data.


## Procedure reference

All maintenance ops are on-demand `ALTER TABLE <table> EXECUTE <procedure>(param => value, …)`
procedures (Trino-style named arguments). The same operations run on a schedule via the
[auto-maintenance service](#auto-maintenance-scheduling) and the Admin UI. Parameter names mirror the
[Apache Iceberg Spark procedures](https://iceberg.apache.org/docs/latest/spark-procedures/) so the
syntax is familiar; a few ontul-specific extensions and convenience aliases are called out per
procedure. Parameters are passed by name and are order-independent; quote string/timestamp values.

Every procedure below is verified end-to-end — `tests/test-iceberg-maintenance-e2e.sh` (Ontul-only)
and `tests/test-iceberg-maintenance-trino-e2e.sh` (run via Ontul, **read back through Trino** against
the same Polaris REST catalog + S3 to prove the resulting metadata is valid for any engine).

---

### `optimize` — compact small files (`rewrite_data_files`)

Reads each data file together with its applicable position- and equality-delete files, writes the
surviving rows into one compacted file, and drops the consumed delete files in a single atomic
`RewriteFiles` commit. This is **delete-aware**: rows previously `DELETE`d/`UPDATE`d are not
resurrected, and the table format (`write.format.default`) is preserved. Run it to fix the
small-file problem caused by frequent commits (especially streaming).

| Parameter | Type | Default | Description |
|---|---|---|---|
| `file_size_threshold_mb` | int | `128` | Target file size. Files smaller than half this are treated as "small" and eligible for compaction. |
| `min_input_files` | int | `5` (`2` for the `OPTIMIZE` shorthand) | Skip compaction unless at least this many small files would be combined (Spark `min-input-files`). Avoids churning an already-optimized table. |
| `window_hours` | int | — (whole table) | **ontul extension.** Compact only the append-only files added by commits in the last *N* hours. Delete-bearing files are skipped (run a full `optimize` for those). Ideal for streaming tables — you never rescan history. |
| `window_cooldown_minutes` | int | `0` (ad-hoc) | **Concurrency-safety.** Exclude files added in the last *N* minutes — the "hot zone" an active writer is appending to. Pulls the window's upper bound back to `now − N min`. |
| `seq_margin` | int | `0` (ad-hoc) | **Concurrency-safety.** Only rewrite files whose data sequence number is `≤ tip − margin`, i.e. committed strictly behind the live tip. Clock-independent "behind the writer". |
| `skip_active_partitions` | bool | `false` (ad-hoc) | **Concurrency-safety.** Skip any partition written within the cooldown window, so compaction never touches a partition an active writer is appending to. |
| `max_commit_retries` | int | `3` (always on) | Bounded re-plan + retry on a concurrent-commit conflict (`CommitFailedException`/`ValidationException`). Pure safety net; absorbs transient races so nothing surfaces to the caller. |

```sql
-- Full-table compaction: combine small files into ~128 MB files, only if ≥5 small files exist
ALTER TABLE ice.sales.orders EXECUTE optimize(file_size_threshold_mb => 128, min_input_files => 5);

-- Shorthand for a whole-table compaction with defaults
OPTIMIZE ice.sales.orders;

-- Incremental compaction for a streaming table: only files written in the last hour / last day
ALTER TABLE ice.sales.orders EXECUTE optimize(window_hours => 1,  min_input_files => 2);
ALTER TABLE ice.sales.orders EXECUTE optimize(window_hours => 24);

-- Concurrency-safe compaction while writers are active (single-committer-style avoidance)
ALTER TABLE ice.sales.orders EXECUTE optimize(
  window_cooldown_minutes => 10, seq_margin => 1, skip_active_partitions => true, max_commit_retries => 5);
```

#### Running compaction alongside live writes

Compaction commits a `RewriteFiles` that replaces existing data files. If that races with a concurrent
append / `MERGE` / `UPDATE` / `DELETE`, Iceberg rejects the loser with `CommitFailedException` /
`ValidationException`. ontul avoids the conflict instead of fighting it — a *single-committer-style*
strategy that keeps compaction strictly **behind** the writer:

- **Cooldown** (`window_cooldown_minutes`) and **active-partition skip** (`skip_active_partitions`) keep
  the rewrite out of the time/partition region a writer is actively touching.
- **Sequence margin** (`seq_margin`) only rewrites files at least *N* sequence numbers behind the live
  tip — clock-independent, so it's robust even with clock skew or bursty commits.
- **Bounded retry** (`max_commit_retries`) re-plans and retries on any residual conflict, so transient
  races never surface to the caller.

Defaults differ by caller: **ad-hoc `EXECUTE optimize` keeps the safety filters OFF** (predictable —
rewrites the whole table just as before, but now with retry on), while **scheduled auto-maintenance
passes SAFE defaults** (cooldown 10 min, seq_margin 1, skip-active on, retries 3) from the per-table
`MaintenanceConfig`. With the filters ON, a full `optimize` becomes *selective* — it leaves hot/active
files untouched rather than collapsing the whole table into one file.

> Verified: 5 small files (10 rows) → 1 file with all 10 rows preserved; windowed run reports
> `Optimized (window 1h): 3 recent small files → 1 file (6 rows)`. Concurrency-safety verified e2e
> (`tests/test-compaction-concurrent-e2e.sh`): a writer appended continuously to a partitioned table
> while safe-default compaction ran **38 rounds concurrently** — **no row loss** (final = baseline +
> every appended row), **zero failures escaping to the client**, and **0 commit conflicts even arose**
> (the cold-file / seq-margin selection kept compaction behind the live writer; the retry net was not
> even needed).

---

### `expire_snapshots` — drop old snapshots and their files

Removes snapshots (table versions) that are no longer needed and physically deletes the data/metadata
files that only those expired snapshots referenced. Keeps time-travel storage bounded. Files still
referenced by a retained snapshot are never touched.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `older_than` | timestamp | — | Expire snapshots older than this instant (Spark `older_than`). Accepts ISO offset (`2026-06-01T00:00:00Z`), ISO local (`2026-06-01T00:00:00` / `2026-06-01 00:00:00`), or date-only (`2026-06-01`). |
| `retain_last` | int | — | Always keep at least the *N* most recent snapshots regardless of age (Spark `retain_last`). |
| `retention_hours` | int | `168` (7 days) | **ontul convenience alias** for `older_than` expressed as "now − N hours". Ignored if `older_than` is given. |

```sql
-- Keep the last 7 days of snapshots
ALTER TABLE ice.sales.orders EXECUTE expire_snapshots(retention_hours => 168);

-- Keep the last 100 snapshots regardless of age (good for high-frequency streaming commits)
ALTER TABLE ice.sales.orders EXECUTE expire_snapshots(retain_last => 100);

-- Combine: expire anything before a date, but never drop below 10 snapshots
ALTER TABLE ice.sales.orders EXECUTE expire_snapshots(older_than => '2026-06-01T00:00:00', retain_last => 10);

-- Shorthand
EXPIRE SNAPSHOTS ice.sales.orders RETAIN LAST 5;
```

> Verified: `expire_snapshots(retention_hours => 0, retain_last => 2)` reduces a 7-snapshot table to
> exactly 2 snapshots, current data (10 rows) intact and readable by Trino.

---

### `rewrite_manifests` — compact manifest metadata

Rewrites the table's manifest files so each scan plans over fewer, well-clustered manifests. Cheap
metadata-only operation; useful after many small commits have fragmented the manifest list.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `spec_id` | int | all specs | Rewrite only manifests belonging to the given partition-spec id (Spark `spec_id`). Use after partition evolution to consolidate one spec's manifests. |

```sql
ALTER TABLE ice.sales.orders EXECUTE rewrite_manifests();
ALTER TABLE ice.sales.orders EXECUTE rewrite_manifests(spec_id => 0);
```

---

### `rewrite_position_delete_files` — consolidate v2 delete files

For Iceberg v2 tables, consolidates multiple Parquet **position-delete** files that target the same
data file into a single delete file per data file, committed via `RowDelta`. Reduces the per-scan
delete-merge cost on tables with heavy `DELETE`/`UPDATE`/`MERGE` traffic. Puffin deletion-vector
files are left as-is.

*No parameters.*

```sql
ALTER TABLE ice.sales.orders EXECUTE rewrite_position_delete_files();
```

---

### `remove_orphan_files` — delete unreferenced S3 files

Lists every object under the table's location and deletes those that **no live snapshot references**
(left behind by failed writes, aborted compactions, etc.). Reachability is computed from current
table metadata, so this complements `expire_snapshots` (which removes snapshots but can leave their
now-unreferenced physical files behind on some object stores).

| Parameter | Type | Default | Description |
|---|---|---|---|
| `older_than` | timestamp | — | Only consider files last-modified before this instant (Spark `older_than`). Same formats as `expire_snapshots`. |
| `older_than_hours` | int | `72` | **ontul convenience alias** — safety window "now − N hours". The default 72 h ensures files from in-flight writes are never deleted. Ignored if `older_than` is given. |
| `dry_run` | bool | `false` | Scan and **report** the files that would be deleted without deleting anything (Spark `dry_run`). Always preview first on a new table. |

```sql
-- Preview only — counts and reports orphans, deletes nothing
ALTER TABLE ice.sales.orders EXECUTE remove_orphan_files(dry_run => true);

-- Actually delete orphans older than the 72 h safety window (default)
ALTER TABLE ice.sales.orders EXECUTE remove_orphan_files(older_than_hours => 72);

-- Shorthand
REMOVE ORPHAN FILES ice.sales.orders;
```

> Verified: after compaction + expiry, `remove_orphan_files(dry_run => true)` reports
> `scanned=23 wouldDelete=14` yet the table still has all 10 rows (nothing deleted).
>
> ⚠️ `remove_orphan_files` requires S3 credentials in the catalog config and uses the table location
> as the listing root — point catalogs at a dedicated prefix per table so the listing is bounded.

---

### `rollback_to_timestamp` / `rollback_to_snapshot` — revert the table

Resets the table's current snapshot to an earlier version — to recover from a bad write, or to
publish/inspect a known-good state. `rollback_to_timestamp` selects the snapshot that was current at
the given instant; `rollback_to_snapshot` targets an explicit snapshot id (find ids in time-travel
history).

| Procedure | Parameter | Type | Description |
|---|---|---|---|
| `rollback_to_timestamp` | `timestamp` (required) | timestamp | Roll back to the snapshot current at this instant. Same formats as `expire_snapshots`. |
| `rollback_to_snapshot` | `snapshot_id` (required) | long | Roll back to this exact snapshot id. |

```sql
ALTER TABLE ice.sales.orders EXECUTE rollback_to_timestamp(timestamp => '2026-06-15T23:19:25');
ALTER TABLE ice.sales.orders EXECUTE rollback_to_snapshot(snapshot_id => 3821550127947089009);
```

> Verified: a table grown to 5 rows rolls back to its 2-row first snapshot; Trino reading the same
> catalog afterward also sees exactly 2 rows (the rolled-back rows are invisible to every engine).

---

### Cross-engine guarantee

Because every procedure commits standard Iceberg metadata through the REST catalog, the results are
readable by any Iceberg engine. `tests/test-iceberg-maintenance-trino-e2e.sh` runs the full set via
Ontul and then queries the same tables through **Trino** (`trinodb/trino`, same Polaris catalog +
ShannonStore S3): row counts and aggregates match exactly, and a `rollback_to_timestamp` performed in
Ontul is reflected in Trino's reads.

For tests and quick teardown, set `polaris.config.drop-with-purge.enabled=true` on the Polaris catalog so `DROP TABLE` actually deletes the underlying S3 data.

