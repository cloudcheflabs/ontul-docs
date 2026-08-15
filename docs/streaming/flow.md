# Ontul Flow

**Ontul Flow** is the visual, governed way to build and operate streaming pipelines in Ontul. A
*flow* is a **named, persisted `source → transform → sink` streaming job** you manage from the Admin
UI (**Analytics → Flow**) or the REST API — register it, start/stop it, watch its live metrics and
logs, preview the data it produces, and review its run history.

Under the hood a flow is an [Ontul streaming job](guide.md): the Flow page simply persists the job
definition and submits it as `SUBMIT STREAMING`. Everything the [Streaming Guide](guide.md) documents
about the engine (Flink-style continuous processing, checkpoints, exactly-once for transactional
sinks, the exchange manager) applies to flows.

## Concepts

| | |
|---|---|
| **Flow** | A named pipeline definition, persisted in the replicated cluster metadata store (`flow.def.<name>`). Survives restarts. |
| **Owner** | The user who created the flow. Admins see and manage every flow; other users see and manage only the flows they own. |
| **Run** | One execution of a flow. Each Start creates a run with its own job id; the run is kept in the flow's history so start/stop cycles accumulate. |
| **Status** | `READY` (registered, never run / stopped), `RUNNING`, `STOPPED`/`KILLED`, `FAILED`. |

Runs correlate to a flow by a stable job name `flow-<name>`, so the Flow list, metrics and history are
consistent no matter which master serves the request.

## Lifecycle

```
        create / edit                 Start                    Stop
  (drag-drop or YAML/JSON)  ─────▶  RUNNING  ─────▶  STOPPED  ──┐
        Save (READY)         ◀───── (read-only) ◀────           │
             ▲                                                  │
             └──────────────────────────────────────────────────┘
                         edit only when NOT running
```

- **Create / edit** — build the pipeline on the canvas (pick a source and sink connector, configure
  each node) **or** paste a YAML/JSON spec, then **Save**. A flow is **editable only when it is not
  running**; a running flow is read-only (Stop it to edit).
- **Start** — deploys the flow to the cluster (IAM-enforced, see below). Status becomes `RUNNING`.
- **Stop** — cancels the run cluster-wide; the run is retained in history.
- **Delete** — removes the flow definition (run history is kept for audit).

## The Admin UI Flow page

The first screen is the **flow list**: name, pipeline summary, owner, status, records, and
Start/Stop/Delete actions. **New flow** opens the editor; clicking a flow opens it for viewing/editing.

The editor has three areas:

- **Connector rail** (left) — drag a **source** or **sink** connector onto the matching node, or click
  it. Also shows the running flows.
- **Canvas** (center) — the `source → transform → sink` pipeline. Click a node to configure it in the
  inspector; click a source/sink node to **preview its rows** in the Data tab. Below the canvas a dock
  gives five tabs:
    - **Spec** — the generated `SUBMIT STREAMING` config as **JSON or YAML**, editable. *Apply to
      canvas* parses your edits back onto the nodes, so you can author a flow entirely by pasting a
      spec.
    - **Data** — a live preview of the source/sink table rows (Iceberg/Ontul tables via SQL, JDBC
      sinks queried directly).
    - **Metrics** — status, source phase, records processed, and an ingestion-throughput sparkline.
    - **Logs** — the running job's log, streamed and auto-scrolled (like the cluster topology log
      viewer).
    - **History** — every run of this flow with start/end time, duration, records and who ran it.
- **Inspector** (right) — per-node configuration.

## Connectors

Every connector shown is backed by a real engine `StreamSource` / `StreamSink`.

### Sources

| Source | Description | Key settings |
|---|---|---|
| **Kafka** | Consume a topic | connection (KAFKA), topic, format (`json`/`avro`), consumer group |
| **Database CDC** | Debezium change-data-capture | database (`postgres`/`mysql`/`sqlserver`/`oracle`/`db2`), connection, tables, snapshot (`initial`/`never`) |
| **Iceberg** | Incremental read of an Iceberg table | table (`catalog.schema.table`), mode (`append` or `changelog` — the latter emits I/U/D with a `__op` column), start (`latest`/`earliest`) |
| **File / object** | Poll an S3 prefix for new newline-delimited objects | connection (S3), path (`s3://bucket/prefix/`), format |
| **NeorunBase** | Poll a NeorunBase (Lakebase OLTP) table | mode (`rest`/`jdbc`), endpoint/JDBC URL, poll query, cursor column |

All five relational **Debezium** connectors are bundled — Postgres, MySQL/MariaDB, SQL Server, Oracle
and Db2 — with their JDBC drivers. Credentials resolve from the registered
[connection](../features/connection-id.md) by id, so no database password lives in the flow spec.

### Sinks

| Sink | Description | Write modes |
|---|---|---|
| **Iceberg** | Write to an Iceberg v2 table | append · upsert · **CDC apply** (SCD Type 1/2, hard/soft/ignore delete) |
| **JDBC** | Write to a relational table (Postgres/MySQL/…) | append · **CDC apply** (upsert on c/u/r, delete on d, by key) |
| **Kafka** | Fan out to a topic | at-least-once or transactional |
| **REST / webhook** | HTTP POST batches of rows as JSON | at-least-once |
| **Elasticsearch** | Bulk index | idempotent by doc id |
| **NeorunBase** | Write to NeorunBase (Lakebase OLTP) | rest/jdbc |
| **Console** | Print rows to the worker log | debug |

### CDC apply — Iceberg & JDBC

When the source carries change events (a CDC source, or an Iceberg `changelog` source), the sink can
**materialize a replica** instead of appending every event. Rows carry an operation column (`__op`:
`c`/`u`/`r` = upsert, `d` = delete); the sink applies them by primary key:

- **Delete handling** — `hard` (equality-delete tombstone, row removed at the next commit), `soft`
  (row kept, a deleted-flag column set and a deleted-at timestamp stamped), or `ignore` (drop deletes).
- **Slowly-changing dimension** (Iceberg) — **Type 1** overwrites in place; **Type 2** closes the
  current version (sets `valid_to` + `is_current=false`) and appends a new version, using the CDC
  *before* image so no re-read is needed.

## SUBMIT STREAMING config

The Flow page persists and submits this JSON (also reachable directly via
`POST /v1/api/sql` with `SUBMIT STREAMING <config>`). A source is either a legacy `kafka` node or a
generic `source` node with a `type`; the sink is a `sink` node with a `type` (plus a `write` block for
Iceberg CDC apply).

```json
{
  "source": {
    "type": "cdc",
    "connector": "postgres",
    "connectionId": "pg-sales",
    "tables": ["public.orders"],
    "snapshot": "initial"
  },
  "operations": [
    { "type": "FILTER", "value": "amount > 50" },
    { "type": "SELECT", "value": "order_id, customer_id, amount" }
  ],
  "sink": { "type": "table", "table": "ice.sales.orders", "schemaEvolution": "add" },
  "errorSink": { "type": "table", "table": "ice.sales.orders_rejected" },
  "write": {
    "mode": "cdc",
    "keys": ["order_id"],
    "delete": "soft",
    "opColumn": "__op",
    "deletedFlag": "is_deleted",
    "deletedAt": "deleted_at",
    "scd": { "type": 2, "validFrom": "valid_from", "validTo": "valid_to", "current": "is_current" }
  },
  "commitIntervalMs": 1000,
  "numWorkers": 2,
  "durationMs": 9999999999
}
```

A Kafka → JDBC append flow is simpler:

```json
{
  "kafka": { "connectionId": "kafka-prod", "topic": "orders", "format": "json", "groupId": "orders-cdc" },
  "operations": [{ "type": "FILTER", "value": "amount > 50" }],
  "sink": {
    "type": "jdbc",
    "jdbcUrl": "jdbc:postgresql://pg:5432/app",
    "tableName": "orders_sink",
    "username": "app", "password": "…",
    "batchSize": 1
  },
  "commitIntervalMs": 1000, "numWorkers": 1
}
```

For a JDBC **CDC-apply** sink, put `mode`/`opColumn`/`keys` on the sink node itself (JDBC does not use
a separate `write` block).

## Data quality — rejected records

A record a flow cannot decode (malformed JSON, an undecodable Avro payload, an empty value) is
**rejected**: it never reaches the sink, and it is **counted**. The count is visible as **Rejected**
in the Flow list, the header metrics, the Metrics tab and each run's history row, and the reason is
logged (`json_parse`, `avro_decode`, `empty_payload`).

!!! note "A rejected record is not a processed record"
    `Records` counts rows written to the sink; `Rejected` counts rows that were dropped before it.
    They never overlap, so `Rejected > 0` always means data did not arrive at the target.

To keep the rejected records instead of only counting them, name a **quarantine table** with
`errorSink` (Admin UI: the sink panel's *Rejected records → Quarantine table* field):

```json
{
  "source": { "type": "kafka", "connectionId": "kafka-prod", "topic": "orders", "format": "json" },
  "sink": { "type": "table", "table": "ice.sales.orders" },
  "errorSink": { "type": "table", "table": "ice.sales.orders_rejected" }
}
```

The quarantine table has a **fixed, source-independent schema** — it holds records whose own shape
could not be trusted:

| column | type | |
|---|---|---|
| `__flow` | string | the flow (job) that rejected the record |
| `__reason` | string | `json_parse` / `avro_decode` / `empty_payload` / `schema_mismatch` |
| `__error` | string | the underlying failure message |
| `__source` | string | where it came from, e.g. `orders-0@1523` |
| `__payload` | string | the raw record, verbatim, for replay (truncated at 64 KB) |
| `__ts` | bigint | rejection time, epoch millis |

It is created on first write, appended at each checkpoint, and governed like any other write target:
the flow's owner needs `data:Insert` on it, and it appears in the flow's audit `tables`. Quarantining
is **best-effort** — if the quarantine table itself cannot be written, the flow keeps running and the
records stay counted, with an error in the log.

## Schema drift

A source's shape is not frozen: someone runs `ALTER TABLE … ADD COLUMN` upstream, or a producer
starts emitting a new JSON field. `sink.schemaEvolution` decides what a flow does about it:

| mode | behavior |
|---|---|
| `add` *(default)* | A column the target table does not have is **added** to it, and `int`→`long` / `float`→`double` are promoted, before the batch is written. |
| `none` | The table schema wins — a new source column is ignored. |
| `strict` | Drift **fails the flow** instead of guessing. Use when the target schema is a contract. |

```json
{ "sink": { "type": "table", "table": "ice.sales.orders", "schemaEvolution": "add" } }
```

Only additive, read-compatible changes are ever applied — exactly the set Iceberg can make without
rewriting committed data files. A column the source **dropped** is deliberately left in the table
(later rows carry `NULL`), because one absent field in one batch is not evidence the column is gone.
Anything narrowing or incompatible is refused: `strict` fails, otherwise the table keeps its type.

Schema drift applies to the **Iceberg sink**. A JDBC sink writes to a table whose schema the target
database owns — add the column there.

## Lag

**Lag** answers "is this flow keeping up":

- **records** — Kafka consumer lag (log end offset − our position), summed over the partitions this
  flow's workers own.
- **time** — how old the newest record processed is. For CDC this is the source database's own change
  timestamp, and it is the only meaningful lag for a change log (there is no row count to be behind
  by).

Both are surfaced in the Flow list and Metrics tab (`1,204 rec · 2.3s`), in the periodic progress line
in the flow's log, and on the job status API as `sourceLag` / `sourceLagMs`. A flow's worker count is
fixed for a run: to scale one out, **Stop → change `numWorkers` → Start** — the run resumes from its
last committed checkpoint, so no data is reprocessed or lost.

## Keeping the sink table fast

A streaming flow commits on an interval, so it naturally produces many small files. Ontul compacts
them **in the background, concurrently with the flow** — no need to stop ingestion:

- Enable per-table compaction (cron or fixed interval) under **Iceberg → Maintenance** in the Admin
  UI, or with `ALTER TABLE … EXECUTE optimize(…)`.
- Concurrent-write safety is built in: a cooldown window excludes the files an active writer is
  still appending to, `skip_active_partitions` leaves a partition a flow is currently writing
  untouched, and commits retry with a sequence-number margin.

See **[Iceberg Maintenance](../iceberg/maintenance.md)** for the full set of operations (compaction,
expire snapshots, remove orphan files, rewrite position deletes) and their tuning.

## Governance (IAM)

Flows are governed like the rest of Ontul — see [IAM](../features/iam.md).

- **Starting a flow** runs it as the user and enforces the same table-level RBAC as SQL DML: an
  Ontul-catalog `table` sink needs `data:Insert` (plus `data:Update`/`data:Delete` for a cdc/upsert
  write mode); an Ontul-catalog `table` source needs `data:Select`. External systems
  (Kafka/CDC/JDBC/…) are reached through their registered [connection](../features/connection-id.md)
  credentials.
- **Visibility & management** are **owner-scoped**: admins see and manage all flows; a non-admin user
  sees and can start/stop/edit/delete only the flows they own.

## Lineage & audit

Starting a flow is captured by Ontul governance exactly like SQL DML — no configuration:

- **[Audit Log](../features/audit-log.md)** — the start is recorded as a `data:Write` event with the
  **sink** as the `resource` and **every source and sink dataset** in `tables`, alongside who started
  it and the full `SUBMIT STREAMING` config. An IAM-refused start is recorded with `decision=DENY`.
- **[Data Lineage](../features/data-lineage.md)** — a lineage edge is recorded from the flow's
  **source(s) → sink** (and emitted as an OpenLineage event when `ontul.lineage.openlineage.url` is
  set). Ontul-catalog tables appear under their `catalog.schema.table` name so a flow's edge lines up
  with SQL-DML lineage; external systems use a scheme-prefixed dataset id — `kafka://<topic>`,
  `cdc://<connector>/<table>`, `jdbc://<table>`, `es://<index>`, `http://<endpoint>`,
  `neorunbase://<table>`, or the S3 path for a file source.

Both are best-effort and never block or fail the flow submission.

## Storage & failover

A flow is a distributed streaming job; if a worker dies, another resumes the job from its last
checkpoint via the **exchange manager**. For real deployments run the exchange (and job logs) on **S3**
so state is not tied to a single worker's disk — configurable at runtime from **Storage** in the Admin
UI, applied cluster-wide without a restart. See [Configuration](../reference/configuration.md) for the
`ontul.exchange.*`, `ontul.job.logs.*` and `ontul.flow.*` properties.

## See also

- [Streaming Guide](guide.md) — the engine, SUBMIT STREAMING, SDK APIs, windowing, Iceberg sink modes.
- [Kafka integration](../features/kafka-integration.md), [Connections](../features/connection-id.md),
  [IAM](../features/iam.md), [Iceberg integration](../features/iceberg-integration.md).
- [Iceberg Maintenance](../iceberg/maintenance.md) — compaction that runs alongside a live flow.
