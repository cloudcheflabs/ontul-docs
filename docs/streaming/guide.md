# Streaming Guide

Ontul streaming follows **Flink's model**: a streaming job (an [Ontul Flow](flow.md)) is a long-running
dataflow that processes every record as soon as it is read — there is no micro-batch trigger — and
recovers from failures through **consistent checkpoints** of its source positions and its state.

This guide covers, in order: how a job runs, sources, operations, event time and watermarks,
windows, state, checkpointing, delivery guarantees and recovery, stopping and resuming, sinks,
monitoring, configuration and current limits.

!!! tip "Coming from Spark Structured Streaming?"
    | Spark | Ontul (Flink model) |
    |---|---|
    | trigger / micro-batch interval | none — records are processed continuously; `checkpointIntervalMs` sets how often a checkpoint commits |
    | `withWatermark(col, delay)` | window spec `\|lateness=<ms>` or `maxOutOfOrdernessMs` — bounded out-of-orderness |
    | output mode `append` for windows | windows emit once, when the watermark passes their end |
    | output modes `update` / `complete` | not supported |
    | `failOnDataLoss` | Kafka source `failOnDataLoss` (default `true`) |
    | `Trigger.AvailableNow` | a bounded run: `durationMs`, which ends with a final checkpoint |
    | checkpoint location | the Exchange Manager (`checkpoints/<stateId>/`), see [Checkpointing](#checkpointing) |
    | `foreachBatch` + idempotent writes | sinks that commit offsets with their data (Iceberg, JDBC, transactional Kafka) |

---

## How a job runs

```
            ┌──────────── worker 0 ────────────┐      ┌──────────── worker 1 ────────────┐
Kafka p0 ──▶│ source ─▶ FILTER/SELECT ─┐        │      │        ┌─ FILTER/SELECT ◀─ source │◀── Kafka p1
Kafka p2 ──▶│                          ├─keyBy──┼──────┼──keyBy─┤                          │◀── Kafka p3
            │ window operator ◀────────┘        │      │        └────────▶ window operator │
            │        └─▶ sink                   │      │                   sink ◀─┘        │
            └───────────────────────────────────┘      └───────────────────────────────────┘
                         ▲  barriers / acks                       ▲
                         └──────────── master CheckpointCoordinator ┘
```

* **Parallelism.** The master runs the job on `numWorkers` workers (all ready workers by default).
  Worker `i` reads Kafka partitions `p` with `p % numWorkers == i` — a fixed assignment, not a
  consumer-group rebalance, so a partition never moves between workers while the job runs.
  More workers than partitions leaves some workers without input.
* **keyBy.** A windowed aggregation hash-partitions records by their `GROUP_BY` values to the worker
  that owns the key (with no `GROUP_BY`, all records go to one worker). A job without a window has
  no exchange: each worker writes its own partitions to the sink.
* **Checkpoints.** The master injects a checkpoint barrier into every worker on a fixed interval;
  see [Checkpointing](#checkpointing).

---

## Submitting a job

### REST — `SUBMIT STREAMING`

```bash
curl -X POST http://localhost:8080/v1/api/sql \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"sql": "SUBMIT STREAMING {
    \"kafka\": { \"connectionId\": \"kafka-prod\", \"topic\": \"user-events\", \"format\": \"json\" },
    \"operations\": [
      { \"type\": \"FILTER\", \"value\": \"amount > 0 AND event_type <> '\''logout'\''\" },
      { \"type\": \"SELECT\", \"value\": \"event_id, event_type, user_name, amount, event_time\" }
    ],
    \"sink\": { \"type\": \"table\", \"table\": \"ice.analytics.events\" },
    \"checkpointIntervalMs\": 5000, \"numWorkers\": 2
  }"}'
```

The response carries one row with the new job's id (column `job_id`), which `/v1/api/job/status`,
`/kill` and `/log` take:

```json
{"status": "ok", "columns": ["job_id"], "rows": [["7f1c…"]], "commandTag": "STREAMING 7f1c…"}
```

Credentials do not belong in the config — reference a registered
[connection](../features/connection-id.md) by `connectionId`.

### Java SDK

```java
String jobId = session.streamSource(Source.kafka("kafka-prod", "user-events").format("json"))
    .filter("amount > 0 AND event_type <> 'logout'")
    .select("event_id", "event_type", "user_name", "amount", "event_time")
    .sink(Sink.table("ice.analytics.stream_events"))
    .checkpointInterval(5_000)
    .workers(2)
    .start();                       // runs until killed; start(Duration) for a bounded run
```

`start()` returns the job id. Builder options: `checkpointInterval`, `commitInterval`, `workers`,
`allowedLateness` / `maxOutOfOrderness`, `idleTimeout`, `stateId`, `exactlyOnce`.
`Sink.table`, `Sink.kafka(connectionId, topic).transactional(true)` and
`Sink.neorunBase(endpoint, table)` are supported; `Sink.s3` is not a streaming sink, and
`map`/`flatMap` throw — a Java function cannot be shipped to the workers, and a job run without it
would produce different data than the program says.

### Python SDK

```python
result = (session.stream_source("user-events", bootstrap_servers="kafka:9092", format="json",
                                auto_offset_reset="earliest")
    .filter("event_type <> 'logout'")
    .sink("ice.analytics.stream_events")
    .checkpoint_interval(5000)
    .start(duration_sec=1800))
print(result["job_id"])
```

---

## Sources

| `type` | Fields |
|--------|--------|
| *(legacy `kafka` node)* | `connectionId`, `topic`, `format` (`json`/`avro`/`raw`), `groupId`, `failOnDataLoss`, `schema`, `properties` |
| `kafka` | as above, plus `bootstrapServers`, `autoOffsetReset`, `partitionDiscoveryMs` |
| `cdc` | `connector` (`postgres`/`mysql`/`sqlserver`/`oracle`/`db2`), `connectionId`, `tables[]`, `snapshot` |
| `iceberg` | `table` (`catalog.schema.table`), `mode` (`append`/`changelog`), `snapshot` (`latest`/`all`) |
| `file` | `path` (`s3://…`, newline-delimited JSON), `maxFilesPerPoll`, `connectionId` |
| `neorunbase` | `mode` (`rest`/`jdbc`), endpoint/`jdbcUrl`, `pollQuery`, `cursorColumn` |

### Kafka: where a partition starts

On every (re)start each partition the worker owns is positioned, in this order:

1. the offset in the restored **checkpoint** (or in the sink, for a sink that stores offsets — see
   [Delivery guarantees](#delivery-guarantees));
2. the consumer group's **committed** offset (`groupId`, default `ontul-stream-<stateId>`);
3. `autoOffsetReset` (`earliest` default, `latest`, or `none` = fail).

A partition added to the topic while the job runs is picked up within `partitionDiscoveryMs`
(default 30 s) and read from its beginning.

**Data loss.** If the offset a job must resume from is no longer in the log — retention deleted it,
or the topic was recreated — the job **fails** with `offset … is outside the log range`
(`failOnDataLoss: true`, the default). Set `failOnDataLoss: false` to resume at the earliest
available offset instead; each such skip is counted in `dataLossEvents`.

### Kafka JSON: schema

Without a declared schema the columns are inferred from the records, as the union of the fields
seen so far (a field that appears later is added, never dropped). A field's type is fixed by the
first value seen: integer → `INT`, or `BIGINT` if it does not fit; number → `DOUBLE`;
`true`/`false` → `BOOLEAN`; anything else, including nested objects (as JSON text) → `VARCHAR`.

A later value that does not fit its column — `68.32` in an integer column, `"abc"` in a number
column — is **rejected** (`schema_mismatch`), never stored as `68` or `0`. To fix the types up front,
declare them:

```json
"kafka": { "connectionId": "kafka-prod", "topic": "orders",
           "schema": { "order_id": "BIGINT", "amount": "DOUBLE", "status": "VARCHAR", "ts": "BIGINT" } }
```

Rejected records are counted (`rejectedRows`, per reason in `rejectsByReason`) and can be kept in
a quarantine table with `"errorSink": {"type": "table", "table": "cat.schema.rejects"}`.

---

## Operations

| Operation | Value | Notes |
|-----------|-------|-------|
| `FILTER` | SQL boolean expression | `AND`/`OR`/`NOT`, parentheses, `= <> != < <= > >=`, `IS [NOT] NULL`, `[NOT] IN (…)`, `[NOT] LIKE`, `[NOT] BETWEEN`. SQL three-valued logic: a comparison with `NULL` does not pass. |
| `SELECT` | `col, col, …` | projection; an unknown column fails the job |
| `WINDOW` | window spec | see [Windows](#windows) |
| `GROUP_BY` | `col, col, …` | keys of the windowed aggregation |
| `AGG` | `SUM(col) AS a; COUNT(*) AS n; …` | `SUM`, `COUNT(col)`, `COUNT(*)`, `AVG`, `MIN`, `MAX`, each with an optional `AS` alias |

Order: `FILTER` runs first; with a window, `SELECT` is ignored (the window output has its own
columns). An expression the executor cannot evaluate — a `FILTER` that does not parse,
`CAST(…)` inside `AGG`, `GROUP_BY` without a `WINDOW`, or any other operation type — **fails the
job at start** with the reason, instead of silently doing something else.

Not supported in the streaming executor: joins (stream–stream and stream–table), deduplication and
user functions. Write the stream to a table and use SQL for those.

---

## Event time and watermarks

A window needs each record's **event time**, taken from the field named in the window spec
(default `event_time`). Accepted values: an Arrow timestamp or date column (any unit/zone), epoch
**milliseconds** as a number or numeric text, or ISO-8601 text (`2026-10-16T09:30:00Z`,
`2026-10-16T09:30:00+09:00`; without an offset it is read as UTC). A record whose event time is
missing or unreadable is rejected (`event_time`), not timed by the wall clock.

The **watermark** says "no record older than this is expected any more". Ontul computes it as
Flink does with bounded out-of-orderness:

* Per Kafka partition: *highest event time seen − bound*. The bound is `|lateness=<ms>` in the
  window spec, or `maxOutOfOrdernessMs` (alias `allowedLatenessMs`, the SDK's `allowedLateness`).
* Per worker: the **minimum** over its partitions — a fast partition cannot declare time a slower
  one has not reached.
* Per window operator: the **minimum** over all workers feeding it through the keyBy exchange.
* It never moves backwards, and it is restored from the checkpoint after a failure.

**Idle partitions.** A partition that never produces holds the watermark — and therefore every
window — back, as in Flink without idleness. Set `idleTimeoutMs`: a partition silent for that
long stops counting until its next record. If event time is held back by a partition that has
never produced, the job logs a warning naming it once a minute.

**Late records.** A record whose window has already fired is dropped and counted in
`lateRecords` (for a sliding window it is still added to the windows that are open). There is no
allowed lateness after firing and no side output; choose the bound to cover your disorder.

---

## Windows

| Window | Spec (REST) | SDK |
|--------|-------------|-----|
| Tumbling | `TUMBLING(SIZE 30 SECONDS)` or `TUMBLING:30000` | `WindowSpec.tumbling(Duration.ofSeconds(30))` |
| Sliding | `SLIDING(SIZE 5 MINUTES SLIDE 1 MINUTE)` or `SLIDING:300000:60000` | `WindowSpec.sliding(…, …)` |
| Session | `SESSION(GAP 30 SECONDS)` or `SESSION:30000` | `WindowSpec.session(Duration.ofSeconds(30))` |

Append `|<event-time-field>` and `|lateness=<ms>`, e.g. `TUMBLING:10000|ts|lateness=3000`.
`GLOBAL` is rejected — it would never close on event time.

* Windows are `[start, end)` aligned to the UTC epoch (no time-zone offset). A sliding window
  covers every `start` that is a multiple of the slide with `start ≤ t < start + size`.
* A session starts at an element and lasts `gap`; sessions of the same key that overlap merge.
* A window **fires once**, when the watermark reaches its end, and its state is dropped. A bounded
  run (`durationMs`) fires every remaining window at the end of input.

Output columns: `window_start`, `window_end` (`BIGINT`, epoch ms), the `GROUP_BY` columns with their
own types, then the aggregates — `COUNT` as `BIGINT`, `SUM`/`AVG`/`MIN`/`MAX` as `DOUBLE`.
`COUNT(col)` counts non-null values; `SUM`/`AVG`/`MIN`/`MAX` ignore nulls and are `NULL` when there
were none. Without an alias the column is named `count` (`COUNT(*)`) or `<fn>_<col>` (`sum_amount`).

---

## State

Open windows are **keyed state** held on the worker's heap (Flink's HashMap state backend) and
written in full into every checkpoint. State lives only as long as its window: it is dropped when
the window fires, so its size is roughly *keys × open windows* — watch `openWindows` and give
workers heap for it. There is no TTL beyond that.

On restore every worker keeps the keys it owns *now*, so a job restarted on a different number of
workers (e.g. after losing one) redistributes its window state by key. State is only restored into
the same window/`GROUP_BY`/`AGG` definition; after changing those the job starts with empty windows
(offsets are still restored).

---

## Checkpointing

```
master                      worker i
  │── barrier n ──────────────▶│ capture source offsets
  │                            │ forward barrier n on every keyBy channel
  │                            │ wait for barrier n from every input channel (alignment)
  │                            │ sink: pre-commit (Iceberg files) or commit (+offsets, JDBC/Kafka-tx)
  │                            │ write checkpoints/<stateId>/chk-n/w<i>  (offsets, watermarks, windows)
  │◀── ack n (+ data files) ───│
  │ all acks → n complete; Iceberg: append every worker's files of n in ONE commit stamped with n
  │── barrier n+1 (completed=n) ──▶ publish n as the restore point, prune older checkpoints
```

* **Interval** — `checkpointIntervalMs` (job), default `ontul.streaming.checkpoint.interval.ms`.
  Older jobs that set only `commitIntervalMs` use it as the interval.
* **Timeout** — `checkpointTimeoutMs` (job), default `ontul.streaming.checkpoint.timeout.ms`. A
  checkpoint missing an ack by then is **aborted**: it never becomes a restore point; Iceberg data
  files that did arrive are committed with the next checkpoint.
* **Barriers are aligned**: a worker does not process records that arrive after a barrier on a
  channel until it has taken that checkpoint, so state and offsets always match.
* **Storage** — the worker's [Exchange Manager](../reference/configuration.md#exchange-manager), under
  `checkpoints/<stateId>/chk-<id>/`, KMS envelope-encrypted. With the default **local-disk**
  exchange a checkpoint lives on its worker's disk and is lost with it: a job moved to another
  worker then resumes from the consumer group's or the sink's offsets, without its window state.
  **Use the S3 exchange (`ontul.exchange.s3.enabled=true`, e.g. on ShannonStore) for failover.**
* Checkpoint ids increase across restarts; only checkpoints from the restore point on are kept.

---

## Delivery guarantees

What a failure costs, per sink. "Exactly-once" means: after any number of worker failures and
restarts, every source record is reflected in the sink exactly once.

| Sink | Without a window | With a window |
|------|------------------|---------------|
| Iceberg `table` (default) | **exactly-once** — the master commits each checkpoint's files once, stamped with the checkpoint id and the offsets | **exactly-once** |
| Iceberg with `ontul.streaming.exactly.once=false` | at-least-once (workers commit on `commitIntervalMs`/row threshold) | at-least-once |
| `jdbc` | **exactly-once** — offsets are written to `ontul_stream_offsets` in the same transaction as the rows | at-least-once; use `mode: upsert` + `keys` (window bounds and group keys) for idempotent re-emits |
| `kafka`, `transactional: true` | **exactly-once** with a Kafka source — the offsets commit inside the Kafka transaction; read it with `isolation.level=read_committed` | at-least-once (uncommitted output is never visible) |
| `kafka` (non-transactional), `elasticsearch`, `http`, `neorunbase`, Lance table | at-least-once | at-least-once |

At-least-once sinks never lose a record: a write failure fails the job (it is no longer logged and
skipped) and the restart replays from the last checkpoint. Offsets are committed to the source only
at a completed checkpoint, never on a timer and never when a job fails or is cancelled.

The JDBC offsets table (`ontul_stream_offsets`, override with `offsetsTable`) is created on first
use; set `"exactlyOnce": false` on the sink to run at-least-once without it. A transactional Kafka
sink uses the stable `transactional.id` `ontul-<stateId>-<worker>`, so a restarted task fences off a
zombie of the previous attempt.

### Failure and restart

A failed task, a dead or restarted worker (checked every 10 s) or a checkpoint commit that fails
three times in a row restarts the job: the master stops the old run everywhere, then redispatches
after an exponential backoff (2 s, 4 s, … 30 s) from the last restore point. After
`maxRetries` (job) / `ontul.streaming.max.retries` failed restarts the job is `FAILED`; a run that
completes `ontul.streaming.retry.reset.checkpoints` checkpoints restores its retry budget.

---

## Stopping and resuming

`POST /v1/api/job/kill/{jobId}` cancels a job **without committing**: it resumes, if started again
under the same `stateId`, from its last completed checkpoint (like a retained checkpoint in Flink).

* A named **Flow** uses `flow-<name>` as its state id, so *Stop* then *Start* resumes where it left
  off. Add `"startFrom": "fresh"` to the config to ignore earlier checkpoints.
* Any job can set `"stateId": "<name>"` to get the same behavior across submissions.
* A bounded run (`durationMs`) ends with a final checkpoint that commits the last output.

---

## Sinks

```json
"sink": { "type": "table", "table": "ice.analytics.events" }
"sink": { "type": "table", "table": "ice.cdc.users", "upsertKeys": ["user_id"] }
"sink": { "type": "jdbc", "connectionId": "analytics-pg", "tableName": "events", "batchSize": 500 }
"sink": { "type": "kafka", "connectionId": "events-kafka", "topic": "out", "keyField": "event_id", "transactional": true }
"sink": { "type": "neorunbase", "mode": "rest", "connectionId": "neorunbase-prod", "tableName": "events" }
"sink": { "type": "elasticsearch", "connectionId": "search-es", "index": "events", "idField": "event_id" }
"sink": { "type": "http", "connectionId": "alerts-webhook", "batchSize": 100 }
"sink": { "type": "console" }
```

**Iceberg upsert.** With `upsertKeys`, each commit is a `RowDelta` of the new data file plus a
key-only equality-delete file, so a newer row replaces the older one at read time without
rescanning the table. Works on partitioned and unpartitioned tables; run `OPTIMIZE` periodically.
A `write` block applies CDC (`mode: cdc`, `delete: hard|soft|ignore`) — see [Ontul Flow](flow.md).

**Schema drift.** An Iceberg sink evolves the table to a drifting source by default (new column
added, `int`→`long` / `float`→`double`); `"schemaEvolution": "none" | "strict"` ignores or fails on
drift.

---

## Monitoring

`GET /v1/api/job/status/{jobId}` (summed or, for time, the slowest worker):

| Field | Meaning |
|-------|---------|
| `processedRows`, `rejectedRows`, `rejectsByReason` | records read; records rejected, by reason |
| `sourceLag`, `sourceLagMs` | Kafka lag in records; age of the newest record read |
| `watermark` | event-time watermark (epoch ms, -1 = none yet) |
| `lateRecords`, `openWindows` | records dropped as late; windows held in state |
| `lastCompletedCheckpointId`, `completedCheckpoints`, `abortedCheckpoints` | checkpoint progress of the current run |
| `checkpointDurationMs`, `checkpointBytes` | last checkpoint's worker-side duration and size |
| `dataLossEvents` | Kafka offsets skipped with `failOnDataLoss: false` |
| `retryCount`, `errorMessage` | restarts so far; last failure |

`GET /v1/api/job/{jobId}/log?from=0` streams the job log: every checkpoint, restore, window
emission batch, rejected record (first N), and a progress line every `ontul.streaming.log.interval.ms`.

---

## Configuration

Job config (top level of the `SUBMIT STREAMING` JSON):

| Key | Default | |
|-----|---------|-|
| `numWorkers` | all ready workers | parallelism |
| `checkpointIntervalMs` | `ontul.streaming.checkpoint.interval.ms` | checkpoint interval |
| `checkpointTimeoutMs` | `ontul.streaming.checkpoint.timeout.ms` | abort a checkpoint after |
| `commitIntervalMs`, `commitRowThreshold` | `ontul.streaming.commit.*` | flush cadence of non-transactional sinks |
| `maxRetries` | `ontul.streaming.max.retries` | restarts before `FAILED` |
| `durationMs` | unbounded | bounded run, ends with a final checkpoint |
| `maxOutOfOrdernessMs` (`allowedLatenessMs`) | 0 | watermark bound (or `\|lateness=` in the window spec) |
| `idleTimeoutMs` | 0 (off) | idle partitions stop holding the watermark |
| `stateId` | `flow-<name>` / job id | checkpoint namespace |
| `startFrom` | — | `fresh` ignores earlier checkpoints |
| `ontul.streaming.exactly.once` | `true` | `false` = legacy per-worker Iceberg commits |
| `errorSink` | — | quarantine table for rejected records |

Server defaults are in [Configuration › Streaming](../reference/configuration.md#streaming).

---

## Current limits

* No stream–stream or stream–table joins, deduplication, user functions, `update`/`complete`
  output, or allowed lateness after a window fires.
* No rate limit on a source. A Kafka poll returns at most `max.poll.records` (default 1000; set it
  in the source `properties`), and a slow keyBy receiver slows its senders (backpressure), but there
  is no records-per-second cap.
* Window state is on the heap; very high key cardinality needs worker memory.
* Failover with state needs the S3 exchange; the default local-disk exchange keeps checkpoints on
  the worker that wrote them.
* CDC sources resume from Debezium's own offset store, which may be ahead of the last checkpoint:
  use a key-based (`upsert`/`cdc`) sink write so a replay is idempotent.

---

## Example: Kafka → Iceberg

```java
session.streamSource(Source.kafka("kafka-prod", "user-events").format("json"))
    .filter("event_type <> 'logout'")
    .select("event_id", "event_type", "user_name", "amount", "event_time")
    .sink(Sink.table("ice.examples.stream_events"))
    .checkpointInterval(5_000)
    .start(Duration.ofSeconds(20));
```

```sql
SELECT event_type, COUNT(*) AS cnt, CAST(SUM(amount) AS DECIMAL(12,2)) AS total_amount
FROM ice.examples.stream_events GROUP BY event_type ORDER BY cnt DESC
```

## Example: windowed revenue

```java
session.streamSource(Source.kafka("kafka-prod", "order-events").format("json"))
    .filter("amount > 0")
    .window(StreamDataFrame.WindowSpec.tumbling(Duration.ofSeconds(30)), "event_time")
    .groupBy("category")
    .agg("SUM(amount) AS total_revenue", "COUNT(*) AS order_count", "AVG(amount) AS avg_amount")
    .sink(Sink.table("ice.analytics.category_revenue_30s"))
    .maxOutOfOrderness(Duration.ofSeconds(5))
    .idleTimeout(Duration.ofMinutes(1))
    .checkpointInterval(10_000)
    .start();
```

Round or cast the aggregates when you read them (`CAST(avg_amount AS DECIMAL(10,2))`) — the
streaming `AGG` does not evaluate expressions.
