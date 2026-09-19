# Cluster Maintenance Mode

A cluster-wide switch that stops Ontul accepting **writes**, so an operator can restore a backup, rotate keys, or rewrite tables out of band without a write landing in the middle of the work.

It does not close the cluster. Reads keep being served, settings stay editable, and running jobs are not stopped. That is deliberate, and the next two sections are why.

!!! note "Not Iceberg maintenance"
    Ontul already uses the word "maintenance" for **table** maintenance — compaction and snapshot expiry, under `/admin/maintenance/…` and the `ontul.maintenance.*` properties. That is a different thing, and it **keeps running** during a window. This feature lives under `/admin/cluster/maintenance` and `ontul.cluster.maintenance.*` to keep the two apart.

## What it refuses

| Refused with `503` | Left open |
| --- | --- |
| SQL `INSERT` / `MERGE` / `UPDATE` / `DELETE` / CTAS / DDL | `SELECT`, `SHOW`, `DESCRIBE`, `EXPLAIN` |
| `CALL` actions from SQL | Retriever search, `traverse`, object-type `query`, `preview` |
| Ontology action-type and action-workflow `invoke` | **`POST /v1/api/authz/check` and `check-batch`** |
| `POST /v1/api/job/submit` and `/v1/api/job/upload` | `job/kill`, `job/status`, `job/list`, `job/…/log` |
| | Catalogs, connections, IAM, KMS, drivers, ontology definitions |

The rule in one line: **writes to data are refused; settings are not.**

Three of those entries are the ones worth explaining, because a blunter switch would get them wrong.

**`/v1/api/authz/check` stays open.** Trino, Spark and Flink call it through their Ontul authorization plugins before every statement. Refusing it makes those engines fail *closed* — every query in the lakehouse starts erroring, not just Ontul's. That is a blast radius out of all proportion to closing Ontul for maintenance, and the call is a read that races nothing.

**Job kill stays open.** The first thing an operator does after opening a window is stop the jobs that are still writing. A switch that blocked `kill` would be blocking the work it exists to enable.

**Settings stay editable.** Catalogs, connections, IAM, KMS and ontology type definitions are schema and configuration, not data — and editing them is usually the reason the window was opened. ShannonStore makes the same call for the same reason.

## Where the setting lives

In the cluster metadata store — the encrypted RocksDB config store — under a single key, `cluster.maintenance.mode`. It replicates to follower masters through the same `exportSnapshot`/`importSnapshot` path as catalogs, ontology types and semantic views, and `putConfig` broadcasts the resulting snapshot immediately, so the change lands cluster-wide without a separate push.

ZooKeeper would have been the other candidate and is the wrong one: **ZooKeeper holds node state** (membership, leadership, readiness) while **settings live in RocksDB**. That split is the same across every Cloud Chef Labs product.

Three consequences follow, and all three are why it is stored rather than held in memory:

- **It survives a master restart.** A master restarted during a window reloads it on boot and comes back still refusing writes.
- **It survives a full cluster restart.** Every master reloads independently.
- **A master that joins mid-window picks it up** from the leader's snapshot rather than accepting writes because it happened to boot.

Each master also keeps the value in memory as a read cache, because every statement consults it and a per-statement store read would put the setting in the hot path. The store is the source of truth; the cache is what the request path reads. The cache is refreshed after every snapshot import — including after a backup restore, which replaces every config key, this one among them.

## Where the check runs

Once, in `QueryService.executeQuery`, on the write branches only. That one place covers **three client surfaces**: the REST `/v1/api/sql` endpoint, **Arrow Flight SQL** (which is how BI tools and JDBC reach Ontul), and MCP — all three funnel through it.

It also runs a second time, in the HTTP layer, for `/v1/api/sql` specifically. That looks like duplication and is worth the two lines: a refusal from inside the query path comes back as `200` with an error in the body, because Flight SQL and MCP have no HTTP status to carry. Over HTTP a client cannot tell "closed for maintenance, come back" from "your SQL is wrong". So the HTTP endpoint classifies the statement itself and answers a real `503` with `Retry-After`.

## Turning it on and off

From the Admin UI: **Cluster Topology → Enter Maintenance**. A confirmation appears first, and a banner stays on screen while the window is open.

Or over REST, against any master — every admin request is forwarded to the leader, which owns the store:

```bash
TOKEN=$(curl -sf -X POST http://localhost:8090/admin/auth/login \
    -H 'Content-Type: application/json' \
    -d '{"username":"admin","password":"…"}' | jq -r .accessToken)

# status
curl -sf http://localhost:8090/admin/cluster/maintenance -H "Authorization: Bearer $TOKEN"

# on
curl -sf -X POST http://localhost:8090/admin/cluster/maintenance \
    -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
    -d '{"enabled":true}'

# off
curl -sf -X POST http://localhost:8090/admin/cluster/maintenance \
    -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
    -d '{"enabled":false}'
```

## What clients see

```
HTTP/1.1 503 Service Unavailable
Retry-After: 30
Content-Type: application/json

{"error":"Cluster is in maintenance mode; writes are refused.",
 "maintenanceMode":true,"retryAfterSeconds":30}
```

Most HTTP clients and SDKs read `503` plus `Retry-After` as a back-off signal and reschedule after the interval the server asked for, so a short window shows up in an application as a pause rather than as failures.

Over Flight SQL and MCP there is no status code to carry, so the same refusal arrives as a query error whose message says the cluster is in maintenance mode and when to retry.

`Retry-After` is configurable:

```properties
# How long a client is told to wait before retrying a write that a maintenance
# window refused (seconds). Set it to roughly how long a window usually lasts.
ontul.cluster.maintenance.retry.after.seconds=30
```

## When to use it

| Use it for | Don't use it for |
| --- | --- |
| Restoring a backup over live catalog / IAM state | Adding or removing a master (leadership handles it) |
| KMS rotation steps that need a quiet write path | Routine config tuning |
| Rewriting or migrating tables outside Ontul | Iceberg table maintenance (that has its own schedule and is safe concurrently) |
| Investigating a data problem without racing new writes | Anything finishing inside a client's retry budget |

The mental model is *defence*: close the write path exactly when a write landing midway through your action would be a problem. It is not a step in routine operations.

## What it does not do

- **It does not stop jobs already running.** A running job keeps writing. Kill it from the Jobs page — that route stays open precisely so you can.
- **It does not stop reads,** including reads that a table rewrite is racing. Iceberg's snapshot isolation already covers that.
- **It does not pause Iceberg table maintenance,** the streaming engine's internal commits, or audit tiering.
- **It does not block settings changes.**
- **It does not drain connections.** Open connections stay open; the `503` is simply the answer to the next write on each.

## Verifying it

`tests/e2e-cluster-maintenance.sh` in the product repository runs the cycle against the two-master compose cluster: a write succeeds, the window opens, the **same write is refused with `503` + `Retry-After` on both masters** — hit directly rather than through nginx, because a test that only talks to the load balancer can pass while a follower quietly keeps taking writes — reads and authz checks and job kill keep working, a master restarted mid-window comes back still refusing, and then the window closes and the same write succeeds again.

That last step matters as much as the refusal. A test that only asserts "refused" passes against an implementation that breaks writes permanently.

## See also

- [High Availability](high-availability.md) — leadership, and what is and is not replicated between masters.
- [Backup & Restore](backup-restore.md) — the operation a window is most often opened for.
- [IAM](iam.md) — the authorization check that deliberately keeps answering.
