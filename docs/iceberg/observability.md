# Iceberg Observability (Table Health)

Iceberg already records everything you need to know about a table's physical condition — file
counts and sizes in the snapshot summary, per-file sizes in the manifests, the metadata log, the
snapshot chain. What it does not do is *watch* any of it. Ontul turns that metadata into a
continuously collected time series, a health score with named findings, and — the part a dashboard
built on `$snapshots` cannot give you — a **record of what each maintenance run actually changed**.

The Admin UI page is **Analytics → Iceberg Health**. Everything on it is also available over REST
and as Prometheus gauges.

## Two collection tiers

Not all Iceberg metadata costs the same to read, and treating it as if it did is how an
observability layer becomes the load it was meant to observe. Ontul collects in two tiers:

| Tier | Default | Reads | Produces |
|---|---|---|---|
| **fast** | every 60s | the already-loaded `metadata.json` — current snapshot summary, snapshot list, metadata log, manifest list | file/delete/record counts, total and average file size, snapshot count, commit interval, manifest count and bytes, metadata generations |
| **deep** | every 30min | plans a file scan — reads **every manifest** | per-file size distribution: small-file ratio, p50/p90, min/max, an 8-bucket histogram |

The fast tier is effectively free: those fields are already in memory once the table is loaded, so
it is safe to run often. The deep tier is the expensive one — the `$files` equivalent — so it runs
on its own slow schedule and its result is cached and carried forward between runs.

A freshly elected leader has no cached distribution for any table, so scheduled deep scans are
additionally capped at `deep.max.per.cycle` (default 3) and spread over subsequent cycles rather
than scanning the whole warehouse in one tick. A scan requested from the UI bypasses the cap.

## Health score and findings

Each table gets a composite score from 100. Every deduction produces a **finding** that names the
maintenance operation which fixes it, so the UI can offer the exact remedy instead of just showing a
red number.

| Code | Condition | Severity / penalty | Remedy |
|---|---|---|---|
| `SMALL_FILES` | avg file size vs. target: `<12.5%` / `<25%` / `<50%` | critical −25 / warning −15 / info −7 | `optimize` (`rewrite_data_files`) |
| `SMALL_FILE_RATIO` | share of files under `target ÷ 4`: `>50%` / `>25%` | critical −20 / warning −10 | `optimize` |
| `SNAPSHOT_BLOAT` | retained snapshots `>500` / `>200` / `>100` | critical −15 / warning −10 / info −5 | `expire_snapshots` |
| `DELETE_DEBT` | delete files as a share of all files: `>30%` / `>10%` | critical −20 / warning −10 | `rewrite_position_delete_files` |
| `METADATA_BLOAT` | `metadata.json` generations `>200` / `>100` | warning −10 / info −5 | `expire_snapshots`, or set `write.metadata.previous-versions-max` |
| `MANIFEST_FRAGMENTATION` | `>50` manifests **and** more than one manifest per 4 data files | warning −8 | `rewrite_manifests` |
| `NO_MAINTENANCE` | auto-maintenance off on a table with `>50` data files | warning −8 | configure a schedule |
| `STALE_MAINTENANCE` | last successful run older than 3× the table's schedule | warning −10 | run maintenance |
| `MAINTENANCE_ERRORS` | any failed run in the last 24h | critical −12 | run maintenance |

Grades: **healthy** ≥ 85, **degraded** ≥ 60, **critical** below that.

The small-file threshold is `targetFileSizeMB ÷ small.file.divisor` — the table's own maintenance
target, not a global constant, so a table configured for 512 MB files is judged against 512 MB.
Tables holding a single data file are excluded from small-file ratios, which have no meaning there.

## The remediation loop

The point of the page is not the diagnosis, which `$snapshots` and `$files` could already give you.
It is closing the loop:

**detect → threshold → remediate → verify.**

Ontul owns the maintenance service, so the last step is possible: every maintenance run is bracketed
with a health sample before it starts and another once it commits. The result is stored as a
**maintenance effect** and shown under *What maintenance changed*:

```
rewrite_data_files   ok   1.1s
  data files 9 → 1     avg file 1.0 KB → 777 B     snapshots 13 → 14
  manifests 12 → 13    deletes 3 → 0               score 65 → 100
```

Two details make this measurement honest:

- An operation that rewrites the physical layout (`rewrite_data_files`,
  `rewrite_position_delete_files`, `remove_orphan_files`) **invalidates the cached size
  distribution**, so the "after" sample runs a real deep scan. Inheriting the pre-compaction
  histogram would report a successful fix as having changed nothing.
- A table maintained before its first collection tick still gets a "before" sample, taken inline, so
  the first — often largest — cleanup is not the one that goes unrecorded.

A run that had nothing to do is recorded as such rather than hidden, which is how you tell "the
schedule is working" apart from "the schedule never fired". Note that the before/after pair brackets
the operation in wall-clock time; on a table under concurrent write load, other commits can land
inside that window and appear in the delta.

## Admin UI

**Analytics → Iceberg Health**

- **Fleet rollup** — average score, healthy/degraded/critical counts, total data files, total size,
  average file size, fleet small-file ratio, collection cycle time.
- **Needs attention** — a card per table below the healthy threshold, showing its top finding and a
  one-click button that runs the remedy for it.
- **All tables** — score bar, data files, size, average file size, small-file ratio, snapshots,
  delete files, last commit, maintenance status.
- **Detail drawer** (click any row) — findings with per-finding remedy buttons, the file-size
  histogram with an on-demand **Deep scan**, a trend chart of file count and average file size,
  the maintenance effect history, and the table's recent maintenance jobs.

## REST API

All routes are under the admin API and follow the usual leader-forwarding rule — a request that
lands on a follower is forwarded to the leader transparently.

| Method | Path | Notes |
|---|---|---|
| `GET` | `/admin/iceberg/health` | fleet summary + every observed table + histogram bucket labels |
| `GET` | `/admin/iceberg/health/table?table=<fqn>[&refresh=true]` | one table; `refresh=true` re-reads it now (fast tier) |
| `GET` | `/admin/iceberg/health/series?table=<fqn>[&since=<epochMs>]` | time series, default the last hour |
| `GET` | `/admin/iceberg/health/effects[?table=<fqn>][&limit=N]` | before/after records, newest first |
| `POST` | `/admin/iceberg/health/scan` | `{"tableKey":"…","deep":true}` — `deep` reads every manifest |

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  http://localhost:8080/admin/iceberg/health | jq '.summary'

curl -s -X POST -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"tableKey":"ice.sales.orders","deep":true}' \
  http://localhost:8080/admin/iceberg/health/scan | jq '.smallFileRatio, .sizeHistogram'
```

## Prometheus metrics

Exposed on `/metrics` alongside the cluster and JVM metrics. Per-table gauges carry a `table` label
holding the fully-qualified `catalog.schema.table`.

| Metric | Meaning |
|---|---|
| `ontul_iceberg_table_data_files` | data files in the current snapshot |
| `ontul_iceberg_table_delete_files` | delete files in the current snapshot |
| `ontul_iceberg_table_records` | total records |
| `ontul_iceberg_table_size_bytes` | total data-file size |
| `ontul_iceberg_table_avg_file_size_bytes` | average data-file size |
| `ontul_iceberg_table_small_file_ratio` | fraction below the small-file threshold; `-1` until deep-scanned |
| `ontul_iceberg_table_delete_file_ratio` | delete files as a fraction of all files |
| `ontul_iceberg_table_snapshots` | retained snapshots |
| `ontul_iceberg_table_manifests` | manifests in the current snapshot |
| `ontul_iceberg_table_metadata_files` | retained `metadata.json` generations |
| `ontul_iceberg_table_health_score` | composite score, 0–100 |
| `ontul_iceberg_table_last_commit_age_seconds` | seconds since the last commit |
| `ontul_iceberg_table_maintenance_last_success_age_seconds` | seconds since the last successful maintenance run; `-1` if never |
| `ontul_iceberg_tables_total`, `…_degraded`, `…_critical` | fleet counts |
| `ontul_iceberg_findings_total` | open findings across all tables |
| `ontul_iceberg_collector_leader` | `1` on the master that owns collection, `0` elsewhere |

Useful alerts:

```promql
# tables whose layout has degraded
min by (table) (ontul_iceberg_table_health_score) < 60

# compaction is falling behind on a table that is being written
max by (table) (ontul_iceberg_table_small_file_ratio) > 0.5
  and max by (table) (ontul_iceberg_table_last_commit_age_seconds) < 3600

# maintenance stopped running
max by (table) (ontul_iceberg_table_maintenance_last_success_age_seconds) > 86400
```

## Multi-master behaviour

Collection is a **cluster singleton**: only the leader walks the catalogs. Table health is a
property of the table, not of the master that read it, so having every master scan every catalog
would multiply the object-store metadata reads for identical results. `ontul_iceberg_collector_leader`
identifies the owner, and per-table gauges are exported by that master only — aggregate with
`max by (table) (…)` so a leader change does not double-count. The REST surface needs no such care:
it is under the admin API, which already forwards follower requests to the leader.

Because the collector's series and effect history live in the leader's memory, a failover restarts
them: current values return on the next cycle, while the in-memory trend and effect log do not carry
over.

Scheduled maintenance is gated the same way, so two masters never compact the same table in
parallel.

## Configuration

Set in `ontul.properties`; all are optional.

| Property | Default | Meaning |
|---|---|---|
| `ontul.iceberg.observability.enabled` | `true` | master collects Iceberg table health |
| `ontul.iceberg.observability.fast.interval.seconds` | `60` | fast tier period |
| `ontul.iceberg.observability.deep.interval.seconds` | `1800` | deep tier period per table |
| `ontul.iceberg.observability.deep.max.per.cycle` | `3` | deep scans started per cycle |
| `ontul.iceberg.observability.series.retention.seconds` | `86400` | in-memory time-series window |
| `ontul.iceberg.observability.small.file.divisor` | `4` | a file below `target ÷ N` counts as small |

## Verification

End-to-end against ShannonStore (S3) + Polaris (Iceberg REST) + a live Ontul cluster:
`tests/test-iceberg-observability-e2e.sh`. It fragments a table with single-row commits, asserts the
fast tier, the deep tier's histogram, the findings and their remedies, the Prometheus exposition,
and then compacts the table and asserts the recorded effect — file count down, health score up.

## See also

- [Iceberg Table Maintenance](maintenance.md) — the operations the findings point at
- [Metadata Tables](metadata-tables.md) — `$snapshots`, `$files`, `$metadata_files` for ad-hoc SQL
- [High Availability](../features/high-availability.md) — leader ownership and forwarding
