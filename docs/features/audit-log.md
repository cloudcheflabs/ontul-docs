# Audit Log

Ontul keeps a **replicated audit log** of query and access activity that answers, for every recorded operation: **who** (user), **which engine** (source), **which tables**, **what query**, and the operation **details**. Audit records are collected independently of the query hot-path, survive leader failover, and can optionally be tiered to durable, queryable S3 storage.

The audit log is the access/activity half of Ontul governance; its table list on each record is the same edge captured by [Data Lineage](data-lineage.md), so the two views line up — lineage answers *"what was this table built from"*, audit answers *"who and which engine touched it, with what query"*.

## What Is Recorded

Each audit event carries:

| Field | Meaning |
|---|---|
| `userId` | The principal who ran the operation |
| `source` | Originating engine — `ontul` today; `trino` / `spark` / `flink` when pushed by external authz plugins |
| `action` | Operation kind, e.g. `data:Select`, `data:Insert`, DDL, IAM change |
| `resource` | Primary resource acted on (table / object) |
| `decision` | Authorization outcome — `ALLOW` or `DENY` (empty on events where no allow/deny applies). A **denied** attempt is recorded, not only successful access |
| `tables` | All tables the operation read and/or wrote (the lineage edge) |
| `query` | The SQL text of the operation, when applicable |
| `details` | Free-form context — for a denial, the refusal reason (e.g. `ontul: ABSTAIN`) |
| `timestamp` | Event time (epoch millis) |

## Automatic Capture

Ontul records audit events for the statements it executes, with **no configuration**:

- **Writes and DDL are always audited** — `INSERT`, `CREATE`, `MERGE`, IAM changes, etc.
- **Reads (SELECT) are audited by sampling** — see below. The engine is recorded as `ontul`, and touched tables are extracted from the statement.

Capture is best-effort and never blocks or fails a query.

### Read Auditing and Sampling

Knowing *who ran which query* requires auditing reads, but read traffic is high-volume and would grow the store quickly. Two knobs bound this:

- `ontul.audit.read.enabled` (default `true`) — record `SELECT`s at all.
- `ontul.audit.read.sample.rate` (default `1.0`) — fraction of reads recorded (e.g. `0.1` = 10%).

Both are changeable at runtime from the Admin UI.

### Retention

- `ontul.audit.log.retention.days` (default `90`) — the **absolute local cap**: records older than this are pruned from the local store whether or not tiering is on. This is the primary knob to bound a read-heavy audit store — lower it (e.g. to `1`) when local growth matters. It is editable at runtime from the Admin UI.

## Authorization Outcome — Allowed and Denied Access

A security audit must answer *"who tried to do what and was **refused**"*, not only *"what succeeded"*. Every access event therefore carries a `decision`:

- `ALLOW` — the access was authorized and executed.
- `DENY` — the access was **refused by IAM** and never ran; the refusal reason is in `details` (e.g. `ontul: ABSTAIN` = no policy grants it).

A denied Ontul query is recorded with `decision=DENY` before the refusal is raised — it does **not** vanish as a generic error. The external engine plugins do the same: a refused Trino/Spark/Flink access is reported as a `DENY` audit event (no lineage is emitted, since nothing was written). Denials are always recorded — they are never subject to the read sampling that bounds high-volume allowed `SELECT`s.

The **Decision** column in the Admin UI makes refused attempts stand out at a glance (a red `DENY` badge), and free-text search matches the refusal reason in `details` — the primary signal for probing, misconfiguration, or a revoked grant still being exercised.

## External-Engine Ingestion (REST)

External engines push audit events into the same Ontul store over a REST endpoint. Trino, Spark, and Flink authorization plugins (as deployed by [Chango](https://www.cloudchef-labs.com)) already enforce Ontul IAM; the same plugins report the accesses they authorized — **and the ones they refused** — here, so audit from every engine is collected centrally.

```
POST /admin/v1/api/audit
Authorization: Bearer <token>
Content-Type: application/json

[
  {
    "source":  "trino",
    "userId":  "alice",
    "action":  "data:Select",
    "resource":"hive.sales.orders",
    "query":   "SELECT id,total FROM hive.sales.orders WHERE region='EU'",
    "tables":  ["hive.sales.orders"],
    "decision":"ALLOW",
    "details": "trino authz plugin"
  },
  {
    "source":  "trino",
    "userId":  "bob",
    "action":  "data:CreateTable",
    "resource":"hive.sales.secret",
    "tables":  ["hive.sales.secret"],
    "decision":"DENY",
    "details": "trino authz plugin (ontul: ABSTAIN)"
  }
]
```

The body may be a single event or an array. The `decision` field is optional (older plugins omit it); when present it is stored and searchable. Ingested events are indistinguishable from native ones in search, filters, and the Admin UI, other than their `source`.

## Access Model

The audit log deliberately splits its read and write paths:

- **Writes bypass IAM.** The internal component is the sole writer; no IAM principal has write access, so audit records cannot be forged or suppressed through the normal authorization surface.
- **Reads enforce IAM.** Audit contains sensitive query text and column names, so listing/searching the log is an authorized admin action.

All audit and lineage writes are forwarded to the **leader** and replicated to standby masters (below).

## Storage and Failover

Audit records live in the **replicated metadata store** (the same mechanism as catalogs, IAM, and lineage): the leader owns the store and broadcasts to standbys, so the audit log **survives restarts and leader failover** — a newly elected leader serves the same history.

Local collection is independent of any tiering/emit path: if S3 tiering is enabled but temporarily failing, records keep accumulating locally and are tiered later (see below) rather than being lost.

## S3 Tiering (Iceberg / Parquet)

By default audit stays local only. When you need durable, long-term, queryable history, enable tiering from the Admin UI (or `ontul.audit.tier.*`). An asynchronous, best-effort sweep offloads older records to S3:

- **`iceberg` (recommended).** Records are bulk-written to an Iceberg **v2** table (default `ice.ops.audit_events`) partitioned by `day(event_time)`, using Ontul's own in-process Iceberg writer (the `IcebergConnectorSink` handler) — not the SQL engine and not the raw Iceberg Java API — which avoids any circular dependency on the query path or IAM.
- **`parquet` (fallback).** Records are written as day-partitioned Parquet under an `s3://` prefix when no Iceberg catalog is available.

A **watermark** marks the resume point; the sweep advances it after each successful commit and retries with exponential backoff on failure. `ontul.audit.tier.retain.local.days` (default `7`) controls how long tiered records are also kept locally as a hot copy — durable history lives in S3, recent data stays local for fast queries. (Only records already tiered *and* older than this window are pruned, so no un-tiered record is ever dropped.)

Once tiered to Iceberg, the audit table is queryable like any other Ontul table:

```sql
SELECT source, count(*) FROM ice.ops.audit_events GROUP BY source;
SELECT * FROM ice.ops."audit_events$snapshots";   -- commit history
```

### Automatic Table Maintenance (ITM)

The audit table receives one commit per tiering sweep, so it accrues many small data files and snapshots. When tiering to Iceberg is enabled, Ontul **auto-registers the audit table for periodic [Iceberg Table Maintenance](../iceberg/maintenance.md)** — by default an **hourly** job that runs compaction, snapshot expiry (retaining the last 24 commits), and orphan-file cleanup. The registration is refreshed on every settings save (so renaming the target table re-registers the new name) and the entry is visible and editable on the Admin UI **Maintenance** page. See `ontul.audit.tier.maintenance.*` in the configuration table.

## Search and Filters

The audit log supports rich search over the collected events:

| Filter | Meaning |
|---|---|
| `user` | Exact user id |
| `engine` | Exact engine (`ontul` / `trino` / `spark` / `flink`) |
| `table` | Substring match against any touched table |
| `action` | `read` (SELECT) / `write` (everything else) / `all` |
| `q` | Free-text over query, details, resource, user, and tables |
| `from` / `to` | Time range (epoch millis) |

```
GET /admin/audit?engine=trino&action=read&q=orders&limit=200
```

## Admin UI

The **Audit** section has two pages:

- **Overview & Graph** — summary counters (events, users, engines, tables, reads, writes), the storage & retention settings (retention days, read enable + sample rate, tiering mode with Iceberg/Parquet targets, retain-local days), and an interactive **relationship graph** of `user → engine → table`. Clicking a user, engine, or table node **drills down** to that entity's related read/write audit logs — with expandable query text and details — right below the graph. Engine badges are color-coded (ontul / trino / spark / flink).
- **Search** — a dedicated page with a free-text search bar plus user / engine / table / type (read·write) / time-range filters over the full log.

Both pages render each event's authorization outcome as a **Decision** badge — a red `DENY` or green `ALLOW` — so refused attempts stand out at a glance next to the action and touched tables.

## Configuration

All keys are settable in `ontul.properties`; the runtime-adjustable ones are also editable from the Admin UI.

| Property | Default | Description |
|---|---|---|
| `ontul.audit.log.retention.days` | `90` | Absolute local retention cap (days) |
| `ontul.audit.read.enabled` | `true` | Audit SELECT/read queries |
| `ontul.audit.read.sample.rate` | `1.0` | Fraction of reads recorded (0.0–1.0) |
| `ontul.audit.tier.mode` | `none` | `none` / `iceberg` / `parquet` |
| `ontul.audit.tier.iceberg.table` | `ice.ops.audit_events` | Target table (catalog.schema.table) for `iceberg` |
| `ontul.audit.tier.parquet.path` | — | Target `s3://` prefix for `parquet` |
| `ontul.audit.tier.connectionId` | — | S3 connection id for tiering |
| `ontul.audit.tier.retain.local.days` | `7` | Local hot-copy window after tiering (days) |
| `ontul.audit.tier.interval.ms` | `300000` | Tiering sweep interval |
| `ontul.audit.tier.batch.size` | `5000` | Max records tiered per sweep |
| `ontul.audit.tier.maintenance.enabled` | `true` | Auto-register the audit table for ITM when tiering→iceberg |
| `ontul.audit.tier.maintenance.cron` | `0 * * * *` | Cron for the audit table's maintenance (hourly) |
| `ontul.audit.tier.maintenance.snapshot.retain.last` | `24` | Snapshots always retained by expiry |
| `ontul.audit.tier.maintenance.remove.orphan.files` | `true` | Run orphan-file cleanup on the audit table |
