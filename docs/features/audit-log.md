# Audit Log

Ontul keeps a **replicated audit log** of query and access activity that answers, for every recorded operation: **who** (user), **which engine** (source), **which tables**, **what query**, the **authorization outcome**, and the operation **details**. Audit records are collected independently of the query hot-path, survive leader failover, and can be tiered to durable, queryable S3 storage — while the Admin UI keeps showing the full history, local and tiered, as one log.

The audit log is the access/activity half of Ontul governance; its table list on each record is the same edge captured by [Data Lineage](data-lineage.md), so the two views line up — lineage answers *"what was this table built from"*, audit answers *"who and which engine touched it, with what query"*.

## What Is Recorded

Each audit event carries:

| Field | Meaning |
|---|---|
| `eventId` | Unique id of the event (used to de-duplicate local and tiered copies) |
| `userId` | The principal who ran the operation |
| `source` | Originating engine — `ontul` for statements Ontul executes; `trino` / `spark` / `flink` for events pushed by the external-engine authorization plugins |
| `action` | Operation kind, e.g. `data:Select`, `data:Insert`, `data:CreateTable`, `data:Write`, DDL, IAM change |
| `resource` | Primary resource acted on (table / object) |
| `decision` | Authorization outcome — `ALLOW` or `DENY` (empty on events where no allow/deny applies). A **denied** attempt is recorded, not only successful access |
| `tables` | All tables the operation read and/or wrote (the lineage edge) |
| `query` | The SQL text of the operation, when applicable |
| `details` | Free-form context — for a denial, the refusal reason (e.g. `ontul: ABSTAIN`) |
| `timestamp` | Event time (epoch millis) — when the access happened, as reported by the engine |
| `ingestTs` | When the event reached Ontul (server clock). Used to order tiering — see [How tiering picks events](#how-tiering-picks-events) |

## Automatic Capture

Ontul records audit events for the statements it executes, with **no configuration**:

- **Writes and DDL are always audited** — `INSERT`, `CREATE`, `MERGE`, IAM changes, etc.
- **Starting an [Ontul Flow](../streaming/flow.md)** (`SUBMIT STREAMING`) is always audited as a
  `data:Write` with the **sink** as `resource` and every **source and sink dataset** in `tables`;
  an IAM-refused start is recorded with `decision=DENY`.
- **Ontology invocations and certification are always audited** — a governed write-back is
  recorded as `action:invoke` (`action:invoke:pending` when it is held for approval,
  `action:exec:denied` on an IAM refusal), and signing a definition off as `ontology:certify` /
  `ontology:decertify` (`ontology:certify:denied` when refused). The `resource` is the definition's
  fully-qualified name and `userId` is the certifier. This is what makes a CERTIFIED badge
  attributable after the fact — see
  [Ontology → Certification](ontology.md#certification-trust-an-agent-can-act-on).
- **Reads (SELECT) are audited by sampling** — see below. The engine is recorded as `ontul`, and touched tables are extracted from the statement.

Capture is best-effort and never blocks or fails a query.

Ontul's **own internal reads** are not audited: when the Admin UI reads tiered history from the audit table (see [Reading the Full History](#reading-the-full-history)), that query is not itself recorded, so refreshing the viewer does not grow the log.

### Read Auditing and Sampling

Knowing *who ran which query* requires auditing reads, but read traffic is high-volume and would grow the store quickly. Two knobs bound this:

- `ontul.audit.read.enabled` (default `true`) — record `SELECT`s at all.
- `ontul.audit.read.sample.rate` (default `1.0`) — fraction of reads recorded (e.g. `0.1` = 10%).

Both are changeable at runtime from the Admin UI.

### Retention

- `ontul.audit.log.retention.days` (default `90`) — the **absolute local cap**: records older than this are pruned from the local store whether or not tiering is on. This is the primary knob to bound a read-heavy audit store — lower it (e.g. to `1`) when local growth matters. It is editable at runtime from the Admin UI.
- With tiering on, `ontul.audit.tier.retain.local.days` additionally decides how long **already-tiered** records also stay local — see [S3 Tiering](#s3-tiering-iceberg-parquet).

## Authorization Outcome — Allowed and Denied Access

A security audit must answer *"who tried to do what and was **refused**"*, not only *"what succeeded"*. Every access event therefore carries a `decision`:

- `ALLOW` — the access was authorized and executed.
- `DENY` — the access was **refused by IAM** and never ran; the refusal reason is in `details` (e.g. `ontul: ABSTAIN` = no policy grants it).

A denied Ontul query is recorded with `decision=DENY` before the refusal is raised — it does **not** vanish as a generic error. The external engine plugins do the same: a refused Trino/Spark/Flink access is reported as a `DENY` audit event (no lineage is emitted, since nothing was written). Denials are always recorded — they are never subject to the read sampling that bounds high-volume allowed `SELECT`s.

The **Decision** column in the Admin UI makes refused attempts stand out at a glance (a red `DENY` badge), and free-text search matches the refusal reason in `details` — the primary signal for probing, misconfiguration, or a revoked grant still being exercised.

!!! warning "Tiered copies do not carry `decision` yet"
    The Iceberg / Parquet copy written by [S3 tiering](#s3-tiering-iceberg-parquet) does not include
    the `decision` column. While an event is still in the local store its decision is shown; once it
    exists **only** in S3 (after local pruning), the Admin UI shows it without a decision badge, and
    SQL over the tiered table cannot filter on it. Keep `ontul.audit.tier.retain.local.days` long
    enough to cover the window in which you review denials, or keep `DENY` review on the local
    window. Adding the column to the tiered schema is planned.

## External-Engine Ingestion (REST)

External engines push audit events into the same Ontul store over a REST endpoint. Trino, Spark, and Flink authorization plugins (as deployed by [Chango](https://www.cloudchef-labs.com)) already enforce Ontul IAM; the same plugins report the accesses they authorized — **and the ones they refused** — here, so audit from every engine is collected centrally.

The endpoint lives at the server root, **not** under `/admin`:

```
POST /v1/api/audit
Authorization: Token <ontul-token>        # or: Bearer <jwt> / AccessKey
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
    "details": "trino authz plugin",
    "ts":      1792288263000
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

Response: `{"status":"ok","ingested":2}`.

- The body may be a single event or an array.
- `ts` (epoch millis) is the time the access happened. It is optional — when omitted, the server time is used. A plugin that reports asynchronously should send the access time; the event is still tiered correctly even if it arrives after newer events have been tiered (see [How tiering picks events](#how-tiering-picks-events)).
- `decision` is optional (older plugins omit it); when present it is stored and searchable.
- Recording an event for **another** user (`userId` different from the caller) needs administrator access —
  engine plugins use an administrator's token. Any other caller may only record events as themselves, so a
  user cannot forge another person's audit trail (`403`).
- `GET /admin/audit` returns a non-administrator's own events only; changing the audit settings needs
  administrator access.
- Ingested events are indistinguishable from native ones in search, filters, and the Admin UI, other than their `source`.

The companion lineage endpoint is `POST /v1/api/lineage` — see [Data Lineage](data-lineage.md#external-engine-lineage-ingestion).

## Access Model

The audit log deliberately splits its read and write paths:

- **Writes bypass IAM.** The internal component is the sole writer; no IAM principal has write access, so audit records cannot be forged or suppressed through the normal authorization surface. This includes the tiered S3 table: it is written in-process by Ontul's Iceberg handler, not through SQL.
- **Reads enforce IAM.** Audit contains sensitive query text and column names, so listing/searching the log is an authorized admin action. SQL over the tiered table goes through normal IAM like any other table.

All audit and lineage writes are forwarded to the **leader** and replicated to standby masters (below). This includes events recorded on a **follower** — a query or `SUBMIT STREAMING` that a load balancer sent to a standby master: the follower keeps the record and forwards it to the leader, which stores it and replicates it back to every master, so the audit viewer (served by the leader) shows it.

## Storage and Failover

Audit records live in the **replicated metadata store** (the same mechanism as catalogs, IAM, and lineage): the leader owns the store and broadcasts to standbys, so the audit log **survives restarts and leader failover** — a newly elected leader serves the same history.

Forwarding from a follower is asynchronous (it never adds a network hop to the query) and ordered. While the leader is unreachable or an election is in progress, a record is retried `ontul.metadata.forward.max.attempts` times (default 5, with a growing pause); if that fails, or more than `ontul.metadata.forward.queue.capacity` (default 10000) records are waiting, the record stays on that follower only and a warning is logged.

Local collection is independent of any tiering path: if S3 tiering is enabled but temporarily failing, records keep accumulating locally and are tiered later (see below) rather than being lost.

## S3 Tiering (Iceberg / Parquet)

By default audit stays local only. When you need durable, long-term, queryable history, enable tiering from the Admin UI (**Audit → Overview & Graph → Storage & Retention**) or with `ontul.audit.tier.*`. An asynchronous, best-effort sweep offloads records to S3:

- **`iceberg` (recommended).** Records are bulk-written to an Iceberg **v2** table (default `ice.ops.audit_events`) partitioned by `day(event_time)`, using Ontul's own in-process Iceberg writer (the `IcebergConnectorSink` handler) — not the SQL engine and not the raw Iceberg Java API — which avoids any circular dependency on the query path or IAM.
- **`parquet` (fallback).** Records are written as day-partitioned Parquet under an `s3://` prefix when no Iceberg catalog is available, using the S3 connection named by `audit.tier.connectionId`.

### What tiering creates for you

For `iceberg` mode the target is `catalog.schema.table`, where `catalog` is an Iceberg catalog registered in Ontul. The sweep creates what is missing:

- the **namespace** (`schema`) if it does not exist, and
- the **table** (Iceberg v2, partitioned by `day(event_time)`) if it does not exist.

No manual `CREATE SCHEMA` is needed.

### Settings are validated when saved

Saving an `iceberg` target from the Admin UI or `POST /admin/audit/settings` checks it immediately instead of accepting it and failing in the background. The save is rejected with **HTTP 400** — and nothing is persisted — when:

| Problem | Example message |
|---|---|
| Unknown mode | `audit.tier.mode must be one of none, iceberg, parquet: icebreg` |
| Iceberg mode without a table | `audit.tier.iceberg.table is required when audit.tier.mode=iceberg` |
| Malformed target | `audit tier target ice.audit_events is not usable: audit tier iceberg table must be catalog.schema.table: ice.audit_events` |
| Catalog not registered | `audit tier target nosuchcat.x.audit_events is not usable: No catalog config for audit tier catalog 'nosuchcat'` |
| Catalog unreachable | `audit tier target … is not usable: <catalog error>` |

A valid target is accepted, and its namespace is created as part of the check. The settings request may change only some keys (for example only the table); validation merges the request over what is already stored.

### Sweep health

`GET /admin/audit/settings` returns the stored settings plus the leader's sweep status:

```json
{
  "audit.tier.mode": "iceberg",
  "audit.tier.iceberg.table": "ice.ops.audit_events",
  "audit.tier.retain.local.days": "7",
  "tierStatus": {
    "mode": "iceberg",
    "lastSuccessMs": 1792287336163,
    "lastFailureMs": 0,
    "lastError": null,
    "retainLocalDays": 7
  }
}
```

`lastError` is `null` while the sweep is healthy. When a sweep fails, the error is kept here (as well as in the master log, as `Audit tiering sweep failed (retry in Ns); RocksDB keeps collecting: …`) and the sweep retries with exponential backoff (30 s doubling up to 10 min). Records keep accumulating locally in the meantime — nothing is pruned until it has been tiered.

### Only the leader tiers

Audit records are replicated to every master, so every master sees the same backlog. The sweep runs on the **leader only**; after a failover the new leader continues from the replicated watermark. Each event is therefore written to S3 once.

### How tiering picks events

A **watermark** marks the resume point. It is kept over **ingest time** — when an event reached Ontul — not over the event's own `timestamp`:

- Engine plugins report asynchronously with the time the access happened. An event can therefore arrive with a timestamp *older* than events that were already tiered. Ordering by ingest time still picks it up on the next sweep; ordering by event time would skip it forever.
- Events ingested in the last `ontul.audit.tier.ingest.settle.ms` (default 2000 ms) wait for the next sweep, so an event stored in the same millisecond as a sweep is never stranded behind the new watermark.
- One sweep tiers at most `ontul.audit.tier.batch.size` events (default 5000); a batch never splits events that share the same ingest millisecond.

### Local copy after tiering

`audit.tier.retain.local.days` (default `7`) controls how long **already-tiered** records also stay in the local store as a hot copy. Local cleanup deletes a record only when it has been tiered **and** is older than this window — a record that has not reached S3 is never deleted by tiering, whatever its timestamp.

Setting it to `0` keeps only untiered records locally; everything else lives in S3. The Admin UI still shows the full history (next section).

### Changing or disabling tiering

Ontul remembers **every Iceberg table that has received tiered events**. Switching tiering back to `none`, or pointing it at a different table, does not hide history that already left the local store — the viewer keeps reading all of those tables. A table that can no longer be read (for example, dropped) is skipped and reported in the viewer response (`tierNote`).

### The tiered table

Once tiered to Iceberg, the audit table is queryable like any other table — from Ontul, and from any engine attached to the same catalog (e.g. Trino):

| Column | Type | Notes |
|---|---|---|
| `event_id` | string | Same id as the local record |
| `source` | string | Engine |
| `user_id` | string | |
| `action` | string | |
| `resource` | string | |
| `tables` | string | Comma-separated table list |
| `query` | string | |
| `details` | string | |
| `ts` | bigint | Event time, epoch millis |
| `event_time` | timestamp (UTC) | Event time; partition source (`day(event_time)`) |

```sql
SELECT source, count(*) FROM ice.ops.audit_events GROUP BY source;
SELECT user_id, query FROM ice.ops.audit_events
 WHERE event_time >= TIMESTAMP '2026-10-01 00:00:00' AND tables LIKE '%sales.orders%';
SELECT * FROM ice.ops."audit_events$snapshots";   -- commit history
```

### Automatic Table Maintenance (ITM)

The audit table receives one commit per tiering sweep, so it accrues many small data files and snapshots. When tiering to Iceberg is enabled, Ontul **auto-registers the audit table for periodic [Iceberg Table Maintenance](../iceberg/maintenance.md)** — by default an **hourly** job that runs compaction, snapshot expiry (retaining the last 24 commits), and orphan-file cleanup. The registration is refreshed on every settings save (so renaming the target table re-registers the new name) and the entry is visible and editable on the Admin UI **Maintenance** page. See `ontul.audit.tier.maintenance.*` in the configuration table.

## Reading the Full History

`GET /admin/audit` (and both Admin UI pages built on it) returns **one merged log**:

1. Matching events are read from the local store.
2. Matching events are read from every Iceberg table that has received tiered events, with the same filters pushed down as SQL predicates.
3. The two are merged, **de-duplicated by `eventId`** (an event inside the local-retention window exists in both), sorted newest first, and cut to `limit`.

Step 2 is skipped only when it cannot change the answer: when the local store alone fills the page with events newer than the local-retention cutoff — everything that recent is still local, so no tiered-only event can rank above them.

Only **Iceberg** tiers are read back. History tiered in `parquet` mode is durable in S3 but is not shown by the viewer once it has been pruned locally — query the Parquet files directly (for example with `read_s3()`), or keep `audit.tier.retain.local.days` long enough to cover the period you review in the UI.

If a tiered table cannot be read, the local and readable results are still returned, with a `tierNote` explaining what was skipped:

```json
{ "events": [ … ], "count": 200, "tierNote": "tiered audit partly unavailable: ice.old.audit_events: …" }
```

## Search and Filters

The audit log supports rich search over the collected events — the filters apply to the local store and to the tiered tables alike:

| Parameter | Meaning |
|---|---|
| `user` | Exact user id |
| `engine` | Engine (`ontul` / `trino` / `spark` / `flink`), case-insensitive |
| `table` | Substring match against any touched table |
| `action` | `read` (`data:Select`) / `write` (everything else) / `all` |
| `q` | Free-text substring over query, details, resource, user, action, and tables (case-insensitive) |
| `from` / `to` | Time range on the event time, epoch millis, both inclusive |
| `limit` | Max events returned, newest first. Default `200`, capped at `ontul.audit.viewer.max.page` (default `5000`) |

```
GET /admin/audit?engine=trino&action=read&q=orders&limit=200
GET /admin/audit?from=1792281600000&to=1792368000000&table=sales.orders
```

The cap exists because one request merges the local store with a scan of the tiered tables; narrow with `from` / `to` and filters to reach further back rather than raising `limit`.

## Admin UI

The **Audit** section has two pages:

- **Overview & Graph** — summary counters (events, users, engines, tables, reads, writes), the storage & retention settings (retention days, read enable + sample rate, tiering mode with Iceberg/Parquet targets, retain-local days), and an interactive **relationship graph** of `user → engine → table`. Clicking a user, engine, or table node **drills down** to that entity's related read/write audit logs — with expandable query text and details — right below the graph. Engine badges are color-coded (ontul / trino / spark / flink).
  The graph and counters summarize a window you choose in the toolbar: a **time range** and a **Max** number of events (100 / 400 / 1000 / 5000, default 400).
- **Search** — a dedicated page with a free-text search bar plus user / engine / table / type (read·write) / **time range** / **max results** filters over the full log.

On both pages the **time range** offers *Any time*, *Last hour*, *Last 24h*, *Last 7 days*, *Last 30 days*, and **Custom range…**, which shows **From** and **To** date-time pickers (browser local time, minute precision; the whole *To* minute is included). When a result reaches the chosen maximum, the page says so — *"Showing the newest N matching events — narrow the time range or filters to see older ones"* — so a truncated view is never mistaken for everything.

Both pages render each event's authorization outcome as a **Decision** badge — a red `DENY` or green `ALLOW` — so refused attempts stand out at a glance next to the action and touched tables.

Both pages show local and tiered events together (see [Reading the Full History](#reading-the-full-history)); they can be opened directly by URL (`/admin/audit`, `/admin/audit/search`) or reloaded.

## Settings API

The runtime settings behind the Storage & Retention panel:

```
GET  /admin/audit/settings          # stored settings + tierStatus
POST /admin/audit/settings          # change any subset; validated, 400 on a bad target
```

| Key | Meaning |
|---|---|
| `audit.retention.days` | Absolute local cap (overrides `ontul.audit.log.retention.days`); applied immediately |
| `audit.read.enabled` / `audit.read.sample.rate` | Read auditing |
| `audit.tier.mode` | `none` / `iceberg` / `parquet` |
| `audit.tier.iceberg.table` | `catalog.schema.table` for `iceberg` |
| `audit.tier.parquet.path` | `s3://` prefix for `parquet` |
| `audit.tier.connectionId` | S3 connection for `parquet` |
| `audit.tier.retain.local.days` | Local copy window for tiered records |

Settings are stored in the replicated metadata store and take effect on the next sweep — no restart.

## Troubleshooting

| Symptom | Cause / action |
|---|---|
| Settings save returns 400 | The target is not usable — the message names the reason (see [Settings are validated when saved](#settings-are-validated-when-saved)) |
| `tierStatus.lastError` is set | The last sweep failed; the error says why. Records keep collecting locally and are retried with backoff |
| Viewer response carries `tierNote` | A previously used tiered table could not be read; its events are missing from that response only |
| Older events missing from the page | The page is capped at `limit`; use a time range or filters |
| A tiered event shows no Decision badge | Tiered copies do not carry `decision` yet (see the warning above) |

## Configuration

All keys are settable in `ontul.properties`; the runtime-adjustable ones are also editable from the Admin UI.

| Property | Default | Description |
|---|---|---|
| `ontul.audit.log.retention.days` | `90` | Absolute local retention cap (days) |
| `ontul.audit.read.enabled` | `true` | Audit SELECT/read queries |
| `ontul.audit.read.sample.rate` | `1.0` | Fraction of reads recorded (0.0–1.0) |
| `ontul.audit.tier.mode` | `none` | `none` / `iceberg` / `parquet` |
| `ontul.audit.tier.iceberg.table` | `ice.ops.audit_events` | Target table (catalog.schema.table) for `iceberg`; namespace and table are created if missing |
| `ontul.audit.tier.parquet.path` | — | Target `s3://` prefix for `parquet` |
| `ontul.audit.tier.connectionId` | — | S3 connection id used by the `parquet` fallback |
| `ontul.audit.tier.retain.local.days` | `7` | Local hot-copy window for already-tiered records (days); `0` = keep only untiered records locally |
| `ontul.audit.tier.interval.ms` | `300000` | Tiering sweep interval |
| `ontul.audit.tier.batch.size` | `5000` | Max records tiered per sweep |
| `ontul.audit.tier.ingest.settle.ms` | `2000` | Events ingested this recently wait for the next sweep |
| `ontul.audit.viewer.max.page` | `5000` | Max events one `GET /admin/audit` request may return |
| `ontul.audit.tier.maintenance.enabled` | `true` | Auto-register the audit table for ITM when tiering→iceberg |
| `ontul.audit.tier.maintenance.cron` | `0 * * * *` | Cron for the audit table's maintenance (hourly) |
| `ontul.audit.tier.maintenance.snapshot.retain.last` | `24` | Snapshots always retained by expiry |
| `ontul.audit.tier.maintenance.remove.orphan.files` | `true` | Run orphan-file cleanup on the audit table |
