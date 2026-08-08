# Configuration

Ontul reads its configuration from `conf/ontul.properties`. Every property can also be overridden at runtime, and the same keys apply to both the Master and the Worker (each node simply ignores the sections that do not concern it).

## Configuration Precedence

A value is resolved in the following order, highest priority first:

1. **System Property** — passed with `-D`, e.g. `-Dontul.master.admin.port=8081`
2. **Environment Variable** — see the naming convention below
3. **Properties file** — `conf/ontul.properties`

If a key is set in more than one place, the higher-priority source wins.

## Environment Variable Naming

To set any property through an environment variable, take the property key, replace dots with underscores, and uppercase it:

| Property | Environment Variable |
| --- | --- |
| `ontul.master.admin.port` | `ONTUL_MASTER_ADMIN_PORT` |
| `ontul.zk.serverList` | `ONTUL_ZK_SERVERLIST` |
| `ontul.base.data.dir` | `ONTUL_BASE_DATA_DIR` |

Many path properties default to a subdirectory of `${ontul.base.data.dir}` (for example `kms/`, `iam/`, `metadata/`, `exchange/`, `job-history/`). Changing the base data directory moves all of them at once unless they are overridden individually.

!!! warning "Persist the data directory"
    In containers `ontul.base.data.dir` MUST resolve to a mounted volume (e.g. `/app/data`). Otherwise all cluster state — KMS keys, IAM, metadata — is lost on restart.

## Base

| Property | Default | Description |
| --- | --- | --- |
| `ontul.base.data.dir` | `./data` | Root directory for all node-local persistent state. Most other path properties default to a subdirectory of this via `${ontul.base.data.dir}`. Relative paths resolve against the process working directory. |

## ZooKeeper

| Property | Default | Description |
| --- | --- | --- |
| `ontul.zk.serverList` | `localhost:2181` | Comma-separated ZooKeeper connect string (`host:port[,host:port...]`) used for cluster coordination: master leader election, live worker registry, and shared config watches. All masters and workers must point at the same ensemble. |
| `ontul.zk.rootPath` | `/ontul` | Base znode namespace under which Ontul creates all its coordination nodes. Change this to run multiple independent clusters on one ZK ensemble. |
| `ontul.zk.sessionTimeoutMs` | `30000` | ZooKeeper session timeout (ms). If a node cannot heartbeat within this window the session expires, the leader loses leadership, and workers are deregistered. Larger values tolerate GC/network blips but slow failover. |
| `ontul.zk.connectionTimeoutMs` | `10000` | Maximum time (ms) to wait for the initial TCP connection to ZooKeeper before giving up on a connect attempt. Affects startup and reconnect latency. |

## Master

| Property | Default | Description |
| --- | --- | --- |
| `ontul.master.host` | `0.0.0.0` | Interface the Master binds to. `0.0.0.0` binds all interfaces. Also advertised to clients as the JDBC/Flight SQL host (unless `ontul.flight.sql.advertised.host` overrides it), so behind NAT or a proxy a routable address may be required. |
| `ontul.master.admin.port` | `8080` | TCP port for the Master's Admin HTTP server (REST API + Admin UI) — the main management/control-plane endpoint. |
| `ontul.master.internal.port` | `19999` | TCP port for the Master's internal NIO control-plane server, used for master↔worker control messaging (not bulk data, which goes over Arrow Flight). |
| `ontul.master.admin.worker.threads` | `8` | Size of the thread pool serving Admin HTTP requests. Raise for higher concurrent admin/REST load; each thread handles one request at a time. |
| `ontul.master.admin.ui.static.path` | `./admin-ui` | Filesystem directory containing the pre-built Admin UI static assets (HTML/JS/CSS) served by the Admin HTTP server. Relative to the process working directory. |
| `ontul.master.flight.sql.port` | `47470` | TCP port for the Master's Arrow Flight SQL server — the unified JDBC endpoint (`jdbc:arrow-flight-sql://host:port`) and the SDK data endpoint. |
| `ontul.master.broadcast.timeout.ms` | `5000` | Timeout (ms) for broadcasting a control message from the Master to all workers (e.g. config/key sync). A worker that does not respond within this window is treated as not acknowledging. |
| `ontul.master.leader.deference.window.ms` | `3000` | Sticky-incumbent leader election: on restart, defer this many ms before joining the election if a previous leader is still alive, so it can reclaim leadership without a leadership toggle. Set to `0` to disable. |

## Worker

| Property | Default | Description |
| --- | --- | --- |
| `ontul.worker.host` | `0.0.0.0` | Interface this Worker binds to and the host it registers under in the worker registry. The registered `host:port` (using the internal port) becomes the worker's nodeId. |
| `ontul.worker.internal.port` | `19998` | TCP port for this Worker's internal NIO control-plane server (master↔worker and worker↔worker control messaging). Also forms the worker's nodeId (`host:port`); each worker on the same host needs a distinct value. |
| `ontul.worker.flight.port.offset` | `1000` | Offset added to `ontul.worker.internal.port` to derive the Worker's Arrow Flight data port (`flightPort = internal.port + offset`). With the defaults the Flight port is `19998 + 1000 = 20998`. |
| `ontul.worker.shutdown.drain.timeout.seconds` | `60` | Graceful-shutdown drain timeout (seconds). On shutdown the Worker stops accepting new work and waits up to this long for in-flight tasks to finish before forcing termination. |

## KMS

| Property | Default | Description |
| --- | --- | --- |
| `ontul.kms.enabled` | `true` | Master switch for the built-in Key Management Service. When `true` the KMS provides envelope-encryption keys for metadata, spill data, internal protocol payloads, etc. Disabling turns off at-rest and in-transit envelope encryption that depends on KMS keys. |
| `ontul.kms.rocksdb.path` | `${ontul.base.data.dir}/kms` | Filesystem path to the embedded RocksDB store holding wrapped (encrypted) data encryption keys. Must persist across restarts; losing it makes all KMS-encrypted data unrecoverable. |
| `ontul.kms.master.key.env` | `ONTUL_MASTER_KEY` | Name of the environment variable (not the key value) that holds the KMS master passphrase/key. The master key unwraps all data encryption keys and seeds JWT signing. The named env var must be set identically on every node. |
| `ontul.kms.pbkdf2.iterations` | `200000` | PBKDF2 iteration count used to derive the KMS master key from the passphrase. Higher is stronger against brute force but slower at startup. Must be identical on all nodes or derived keys will not match. |
| `ontul.kms.encrypt.internal.protocol` | `true` | When `true`, internal NIO control-protocol message payloads are KMS envelope-encrypted in transit between master and workers using the key id below. |
| `ontul.kms.internal.protocol.key.id` | `internal-protocol` | KMS key id used to encrypt internal NIO protocol payloads. Only meaningful when `ontul.kms.encrypt.internal.protocol=true`. The Master ensures this key exists. |
| `ontul.kms.fetch.max.retries` | `60` | Maximum retries a Worker performs when fetching KMS keys from the Master at startup (the Master may not have finished key init). Combined with the retry interval this bounds total wait (~60s default). |
| `ontul.kms.fetch.retry.interval.ms` | `1000` | Delay (ms) between successive KMS key-fetch retry attempts (see `ontul.kms.fetch.max.retries`). |

## IAM

| Property | Default | Description |
| --- | --- | --- |
| `ontul.iam.rocksdb.path` | `${ontul.base.data.dir}/iam` | Filesystem path to the embedded RocksDB store for IAM state (users, access/secret keys, roles, policies). Must persist across restarts. |
| `ontul.iam.admin.user` | `admin` | Username of the bootstrap admin account, created on first startup if no users exist. Used to log into the Admin UI/REST API initially. |
| `ontul.iam.admin.password` | `admin` | Initial password for the bootstrap admin account (used only when the account is first created). Change this in production; afterwards rotate via the Admin UI or `bin/ontul-cli.sh iam:reset-password`. |
| `ontul.iam.audit.dir` | `${ontul.base.data.dir}/iam-audit` | Directory for the append-only IAM/security audit log (authentication attempts, access-key and password changes, role/policy edits). Entries are pruned per `ontul.audit.log.retention.days`. Must persist across restarts. |
| `ontul.admin.socket.enabled` | `true` | Whether the Master exposes the out-of-band Unix domain socket used for local admin recovery (`bin/ontul-cli.sh iam:reset-password`). The socket file's OS permission (mode 600) is the only authentication. Set `false` to remove the local recovery path. See [Admin Password Recovery](../features/admin-password-recovery.md). |
| `ontul.admin.socket.path` | `${ontul.base.data.dir}/admin.sock` | Filesystem path of the admin recovery Unix domain socket. Must live on a local filesystem that supports Unix domain sockets (not a networked mount); recreated on each Master start. |
| `ontul.admin.socket.marker.file` | `master.socket` | Name of the file under `<ontul.home>/bin` where the Master writes the socket path it actually bound to. `bin/ontul-cli.sh` prefers that published path over re-deriving one from this file, since `ontul.base.data.dir` can be overridden with `-D` at launch. Removed on shutdown. |

## Metadata

| Property | Default | Description |
| --- | --- | --- |
| `ontul.metadata.rocksdb.path` | `${ontul.base.data.dir}/metadata` | Filesystem path to the embedded RocksDB store holding cluster metadata: catalog definitions, connections, table maintenance config, storage settings, etc. Must persist across restarts. |
| `ontul.metadata.kms.key.id` | `metadata-encryption` | KMS key id used to envelope-encrypt metadata at rest in the RocksDB store. Requires KMS to be enabled. |

## RPC Timeouts

| Property | Default | Description |
| --- | --- | --- |
| `ontul.query.rpc.timeout.ms` | `30000` | Timeout (ms) for query-related RPCs the Master issues to Workers (e.g. dispatching a query stage and awaiting its result). Raise for long-running stages over slow links. |
| `ontul.internal.rpc.timeout.ms` | `10000` | Timeout (ms) for internal (non-query) control-plane RPCs, e.g. small master↔worker coordination calls. Keep relatively short. |

## Semantic Layer / Retrievers

| Property | Default | Description |
| --- | --- | --- |
| `ontul.retriever.max.rows.ceiling` | `1000` | Hard cluster-wide cap on the number of rows a single [retriever](../features/retrievers.md) invocation (`POST /api/v1/retrievers/{fqn}/invoke`) may return. Applied on top of each retriever's own `maxRowsCeiling`, so the effective cap is the smaller of the two. Lets an operator bound retriever result size globally regardless of per-retriever definitions. |

## Worker Health Check

| Property | Default | Description |
| --- | --- | --- |
| `ontul.worker.health.check.interval.ms` | `10000` | Interval (ms) between rounds of the Master's worker health checker. Each round pings every registered worker. Lower means faster failure detection but more background traffic. |
| `ontul.worker.health.check.timeout.ms` | `5000` | Per-worker health-check ping timeout (ms). A ping that does not return within this window counts as one failure for that worker. |
| `ontul.worker.health.max.consecutive.failures` | `3` | Number of consecutive failed health checks before a worker is declared down and removed from the cluster (its in-flight work is rescheduled/failed). With the defaults a worker is evicted after ~3 missed 10s rounds. |
| `ontul.worker.health.check.future.grace.ms` | `2000` | Extra grace (ms) added on top of the per-check timeout when waiting for each `Future` result during a health-check round. Prevents premature `TimeoutException` for checks about to return. |

## Query

| Property | Default | Description |
| --- | --- | --- |
| `ontul.query.history.max.size` | `200` | Maximum number of finished queries retained in the in-memory query history (shown in the Admin UI). Oldest entries are evicted once the cap is reached. |
| `ontul.query.max.global.concurrency` | `100` | Cluster-wide admission limit: max queries allowed to run concurrently across all users. Enforced by a global semaphore; excess queries wait (see acquire timeout) then are rejected. |
| `ontul.query.max.per.user.concurrency` | `10` | Per-user admission limit: max concurrent queries a single IAM identity may run. Prevents one user from starving others within the global limit. |
| `ontul.query.default.timeout.ms` | `300000` | Default per-query execution timeout (ms, 5 minutes) when a query/job does not specify its own. Long-running queries exceeding this are cancelled. |
| `ontul.query.max.memory.bytes` | `1073741824` | Soft memory budget (bytes) per query for in-memory (Arrow) execution before operators spill intermediate data to disk via the exchange manager. Default is 1 GiB. |
| `ontul.query.semaphore.acquire.timeout.seconds` | `30` | Maximum time (seconds) a query waits to acquire a concurrency admission slot before it is rejected as overloaded. |

## Connector

| Property | Default | Description |
| --- | --- | --- |
| `ontul.connector.batch.size` | `4096` | Number of rows per Arrow record batch when connectors read from / write to external sources (the vector batch size). Larger batches improve throughput and vectorization but use more memory per in-flight batch. |

## Iceberg Catalog (REST)

Server-wide defaults for Iceberg REST catalogs. Every value is a **default**: a catalog registered through the Admin UI / `POST /admin/catalogs` can override it with its own `catalog.rest.*` config. These apply on both the Master (interactive query) and Worker (job source/sink) code paths. See [Iceberg Integration](../iceberg/integration.md) for per-catalog config.

| Property | Default | Description |
| --- | --- | --- |
| `ontul.iceberg.polaris.default.scope` | `PRINCIPAL_ROLE:ALL` | Default OAuth2 scope requested from a Polaris REST catalog when a catalog does not set `catalog.rest.scope`. Ignored for the `glue` flavor (not OAuth2). |
| `ontul.iceberg.glue.signing.name` | `glue` | AWS SigV4 signing service name for the AWS Glue Iceberg REST endpoint. Use `s3tables` when pointing at the Amazon S3 Tables REST endpoint. |
| `ontul.iceberg.default.region` | `us-east-1` | Default AWS region used whenever none is otherwise specified — for S3 data access (when `s3.region` is absent) and for SigV4-signing Glue REST requests (when neither `catalog.rest.signing.region` nor `s3.region` is set). The single fallback for every Iceberg AWS region. |
| `ontul.iceberg.rest.credential.vending.bypass` | `true` | Suppress catalog-vended S3 credentials (Polaris STS-subscoped tokens / Glue Lake Formation) so Ontul reads and writes S3 directly with the configured static keys, keeping Ontul IAM the authoritative access boundary. Leave `true` (the only tested path; S3-compatible servers reject the STS temp keys Polaris would otherwise vend). |
| `ontul.iceberg.default.format.version` | `2` | Default Iceberg table format version for newly created tables. `2` = position-delete files; `3` = deletion vectors (Puffin), row lineage, variant. Opt-in: existing tables keep their own version and the library reads/writes all versions. On `3`, `DELETE`/`UPDATE`/`MERGE` write deletion vectors instead of position-delete files. See [Iceberg Integration](../iceberg/integration.md#format-version-3-deletion-vectors). |

## Iceberg Maintenance

| Property | Default | Description |
| --- | --- | --- |
| `ontul.maintenance.interval.hours` | `6` | Cluster-wide default interval (hours) between automatic Iceberg table maintenance runs (expire snapshots + rewrite/compact data files). Runs on the Master leader. Individual tables can override this. |
| `ontul.maintenance.snapshot.retention.hours` | `168` | Default snapshot retention (hours) for expire-snapshots: snapshots older than this are removed during maintenance (default 7 days). Expired snapshots can no longer be time-travelled to. |
| `ontul.maintenance.target.file.size.mb` | `128` | Default target output file size (MB) for the data-file rewrite/compaction step (passed as `file_size_threshold_mb` to `OPTIMIZE`). Larger means fewer, bigger files and fewer small-file reads. |
| `ontul.maintenance.initial.delay.ms` | `60000` | Initial delay (ms) after Master startup before the first maintenance round runs, so the cluster can finish coming up first (default 60s). |
| `ontul.maintenance.history.retention.days` | `30` | Retention (days) for the maintenance job history (records of past expire/compaction runs). Older entries are pruned. |

## Audit Log

| Property | Default | Description |
| --- | --- | --- |
| `ontul.audit.log.retention.days` | `90` | Retention (days) for the query/access audit log and IAM/security entries — the absolute local cap; records older than this are pruned from the local store whether or not tiering is on. Editable at runtime from the Admin UI. |
| `ontul.audit.read.enabled` | `true` | Whether `SELECT`/read queries are audited (who ran which query). Volume is bounded by the sample rate. |
| `ontul.audit.read.sample.rate` | `1.0` | Fraction (0.0–1.0) of read queries recorded when read auditing is on. |
| `ontul.audit.tier.mode` | `none` | Audit tiering target: `none` (local only), `iceberg` (recommended), or `parquet` (fallback). Older records are offloaded to durable, queryable S3. |
| `ontul.audit.tier.iceberg.table` | `ice.ops.audit_events` | Target Iceberg table (`catalog.schema.table`) for `tier.mode=iceberg`. Written as a format-v2 table partitioned by `day(event_time)` via Ontul's in-process Iceberg writer. |
| `ontul.audit.tier.parquet.path` | _(empty)_ | Target `s3://` prefix for the `tier.mode=parquet` fallback. |
| `ontul.audit.tier.connectionId` | _(empty)_ | S3 connection id (from the Connection registry) used by tiering. |
| `ontul.audit.tier.retain.local.days` | `7` | Days a tiered record is also kept locally as a hot copy (durable history lives in S3). Only records already tiered *and* older than this are pruned. |
| `ontul.audit.tier.interval.ms` | `300000` | How often the tiering sweep runs (ms). |
| `ontul.audit.tier.batch.size` | `5000` | Max records tiered per sweep. |
| `ontul.audit.tier.maintenance.enabled` | `true` | When tiering→iceberg, auto-register the audit table for periodic Iceberg Table Maintenance (compaction + expire-snapshots + orphan cleanup). |
| `ontul.audit.tier.maintenance.cron` | `0 * * * *` | 5-field UNIX cron for the audit table's maintenance (default hourly). |
| `ontul.audit.tier.maintenance.snapshot.retain.last` | `24` | `expire_snapshots` retain-last for the audit table — always keep at least this many recent commits. |
| `ontul.audit.tier.maintenance.remove.orphan.files` | `true` | Run `remove_orphan_files` on the audit table (reclaims files left by compaction/expiry). |

See [Audit Log](../features/audit-log.md) for the full feature description.

## Job

| Property | Default | Description |
| --- | --- | --- |
| `ontul.job.history.dir` | `${ontul.base.data.dir}/job-history` | Directory where submitted-job history and job logs are stored. Accepts a local path or an S3 path (e.g. `s3://bucket/ontul/job-history`). If a jobLogs S3 storage connection is configured in the Admin UI, that S3 location overrides this value. |
| `ontul.job.large.row.threshold` | `100000` | Row-count threshold above which a job's result set is treated as "large" and handled via the streaming/spill path rather than buffered fully in memory. |

## Flight SQL

| Property | Default | Description |
| --- | --- | --- |
| `ontul.flight.sql.advertised.host` | _(commented / optional)_ | Specific hostname advertised for nginx-safe result streaming. When behind nginx, set to this Master's direct IP/hostname. Leave empty/unset to reuse the same connection (works without nginx). |

## Logging

| Property | Default | Description |
| --- | --- | --- |
| `ontul.log.path` | `${user.dir}/logs` | Directory where log files are written (defaults to a `logs` folder under the process working directory). Used by the logback configuration to place per-node log files. |
| `ontul.log.output.name` | _(commented / optional)_ | Base name used when naming this node's log/stdout file (e.g. `master` or `worker`). When unset, workers fall back to a name derived from their internal port (`worker-<port>.out`). |

## Exchange Manager

The Exchange Manager handles spill of intermediate shuffle/exchange data and execution-state checkpoints (streaming snapshots). It has two storage modes, selected by `ontul.exchange.s3.enabled`, and all payloads are KMS envelope-encrypted at rest in both modes.

- **Local-disk mode (default).** Data is written to — and read from — a worker-local directory. This is fast, but the data is tied to that worker's disk: a stage that is retried on a **different** worker cannot see it.
- **S3-primary mode (`ontul.exchange.s3.enabled=true`).** Data is read and written **directly to S3** (the object store is the primary exchange storage, not a background copy). Because the spill/snapshot objects live in S3, a task that is reassigned to a different worker reads the same data from S3 — this is what enables worker-independent, fault-tolerant recovery. Object keys are worker-independent (`{prefix}spill/{stageId}/…`, `{prefix}snapshot/{checkpointId}/…`), so any worker configured with the same bucket and prefix resolves them.

!!! note
    S3-primary mode is analogous to a filesystem/object-store-backed exchange manager (`exchange.base-directories=s3://…`) in engines that support fault-tolerant execution: the object store **replaces** local disk for exchange data — it is not merely a backup of it.

| Property | Default | Description |
| --- | --- | --- |
| `ontul.exchange.base.dir` | `${ontul.base.data.dir}/exchange` | Worker-local directory used in **local-disk mode** for spilling intermediate shuffle/exchange data and execution-state checkpoints. Scratch/spill space (safe to be node-local); KMS envelope-encrypted at rest. Ignored in S3-primary mode. |
| `ontul.exchange.s3.enabled` | `false` | `false` selects **local-disk mode**. `true` selects **S3-primary mode**, where spill and snapshot data is read and written directly to S3 (not local disk, and not a background backup). |
| `ontul.exchange.s3.endpoint` | _(commented / optional)_ | S3 endpoint URL for the exchange bucket. Set for S3-compatible stores (e.g. MinIO `http://localhost:9000`); leave unset to use the AWS default endpoint. |
| `ontul.exchange.s3.region` | _(commented / optional, default `us-east-1`)_ | AWS region for the exchange bucket. |
| `ontul.exchange.s3.bucket` | _(commented / optional, default `ontul-exchange`)_ | S3 bucket name used to store exchange objects. |
| `ontul.exchange.s3.prefix` | `exchange/` | Key prefix under which exchange spill/snapshot objects are stored inside the bucket, so multiple clusters can share one bucket. Trailing slash optional. |
| `ontul.exchange.s3.path.style` | _(commented / optional, default `true`)_ | Use path-style S3 addressing (bucket in the URL path) instead of virtual-hosted style. Required by most S3-compatible servers like MinIO. |
| `ontul.exchange.s3.access.key` | _(commented / optional)_ | Static access key for the exchange S3 bucket. Leave empty to use the default AWS credential provider chain (env/instance profile/etc). |
| `ontul.exchange.s3.secret.key` | _(commented / optional)_ | Static secret key paired with the access key above. |
| `ontul.exchange.writer.threads` | `2` | Number of background writer threads the exchange manager uses for spill and state-snapshot writes in **local-disk mode**. More threads means more parallel disk I/O for spill-heavy or fault-tolerant workloads. |
| `ontul.exchange.s3.orphan.ttl.ms` | `86400000` | **S3-primary mode only.** On worker startup, sweep exchange objects older than this TTL (ms) — orphans left by crashed jobs whose stage cleanup never ran. `0` disables the sweep. Default 24h. |
| `ontul.exchange.s3.multipart.threshold.bytes` | `67108864` | **S3-primary mode only.** Object size (bytes) above which uploads switch to multipart — avoids the 5 GB single-PUT limit and improves throughput for large spills. Default 64 MiB. |
| `ontul.exchange.s3.multipart.part.size.bytes` | `16777216` | **S3-primary mode only.** Part size (bytes) for multipart uploads (S3 minimum is 5 MiB). Default 16 MiB. |

## Streaming

| Property | Default | Description |
| --- | --- | --- |
| `ontul.streaming.checkpoint.interval.ms` | `10000` | Default interval (ms) between checkpoint barriers for streaming jobs (how often a fault-tolerant snapshot of stream state is taken). A job may override via `checkpointIntervalMs`. Lower means more frequent checkpoints. |
| `ontul.streaming.max.retries` | `3` | Maximum number of automatic restart attempts for a streaming job that fails before it is marked permanently failed. |
| `ontul.streaming.source.poll.timeout.ms` | `100` | Source poll timeout (ms), e.g. Kafka consumer poll. Lower reduces end-to-end latency but increases CPU/poll churn; a job may override it. |
| `ontul.streaming.commit.interval.ms` | `10000` | Time-based sink commit interval (ms) for non-transactional sinks: the current micro-batch is flushed/committed at least this often. Default when the job does not specify `commitIntervalMs`. |
| `ontul.streaming.commit.row.threshold` | `10000` | Row-count sink commit threshold: the current micro-batch is flushed as soon as this many rows accumulate, even before the commit interval elapses (whichever comes first). |
| `ontul.streaming.log.interval.ms` | `30000` | Interval (ms) between streaming progress log lines (throughput/offset logging). Default when the job does not override it. |
| `ontul.streaming.checkpoint.send.timeout.ms` | `5000` | Timeout (ms) for the checkpoint coordinator to send a checkpoint barrier message to a worker. |
| `ontul.streaming.checkpoint.timeout.ms` | `30000` | Timeout (ms) for a checkpoint round overall — how long the coordinator waits for all workers to acknowledge the barrier before the checkpoint is considered failed. |

## Ontul Flow

Tuning for the named streaming-pipeline registry ([Ontul Flow](../streaming/flow.md)). A running
flow's live status + run history are replicated across masters via the metadata store so the Flow
list and History tab are consistent regardless of which master serves the request.

| Property | Default | Description |
| --- | --- | --- |
| `ontul.flow.status.persist.interval.ms` | `5000` | How often the owning master persists a running flow's live status + records to the replicated metadata store (`flow.status.<name>` / `flow.history.<name>`). Lower = fresher Flow list at the cost of more metadata writes/broadcasts. |
| `ontul.flow.status.freshness.ms` | `15000` | A persisted flow.status snapshot older than this is treated as stale (the flow is shown as not-running until the owner refreshes it). Keep it a few multiples of the persist interval. |
| `ontul.flow.history.max.runs` | `20` | Maximum run records retained per flow in the replicated run history (History tab). |

## Job Logs

Streaming and batch job logs can be flushed to append-only rolling gzip segments on S3
(`{prefix}jobLogs/<jobId>/<seq>-<ts>.log.gz`), so long-running jobs' logs stay durable and queryable
by time range after the in-memory log is gone. By default these settings inherit the Exchange
Manager's S3 target, so turning on S3-primary exchange also puts job logs on S3.

| Property | Default | Description |
| --- | --- | --- |
| `ontul.job.logs.s3.enabled` | _(inherits `ontul.exchange.s3.enabled`)_ | Enable flushing job logs to S3 rolling segments. Defaults to whatever the exchange S3 mode is. |
| `ontul.job.logs.memory.tail` | `1000` | Number of most-recent log lines kept in memory per job (the rest are flushed to S3 segments). |
| `ontul.job.logs.flush.interval.ms` | `10000` | How often the worker/master drains buffered job-log lines to an S3 segment (time-based roll). |

The S3 endpoint/region/bucket/prefix/credentials for job logs default to the exchange S3 target
(`ontul.exchange.s3.*`) and can be overridden with `ontul.job.logs.s3.*`. The backend can also be
switched from the Admin UI **Storage** page at runtime (no restart), which persists to the replicated
metadata store as `storage.jobLogs.*` / `storage.exchange.*` referencing an S3 connection.

## Admin

| Property | Default | Description |
| --- | --- | --- |
| `ontul.admin.token.access.expiry.ms` | `900000` | Lifetime (ms) of short-lived JWT access tokens issued after login. After expiry the client must use its refresh token to obtain a new access token (default 15 minutes). |
| `ontul.admin.token.refresh.expiry.ms` | `86400000` | Lifetime (ms) of JWT refresh tokens, which let a client mint new access tokens without re-authenticating (default 24 hours). Effectively the maximum idle session length. |
| `ontul.admin.http.max.content.size.bytes` | `10485760` | Maximum accepted Admin HTTP request body size (bytes); larger requests are rejected (default 10 MiB). Raise if uploading large dependency JARs/drivers via the Admin API. |

## Python

| Property | Default | Description |
| --- | --- | --- |
| `ontul.python.path` | `python3` | Python interpreter used by Workers to run Python UDFs and Python-based exports (PDF/XLSX/PPTX). Either a command on `PATH` or an absolute path. Must exist on every worker host with the required packages. |

## Cache

| Property | Default | Description |
| --- | --- | --- |
| `ontul.cache.max.bytes` | `268435456` | Maximum size (bytes) of the in-memory Arrow LRU result/data cache. Once full, least-recently-used entries are evicted (default 256 MiB). Larger means more cache hits but higher memory use. |

## Data Lineage

| Property | Default | Description |
| --- | --- | --- |
| `ontul.lineage.openlineage.url` | _(commented / optional)_ | OpenLineage-compatible HTTP endpoint (Marquez, DataHub, ...) to emit a `RunEvent` on every CTAS/INSERT/MERGE/CREATE VIEW. Leave empty to disable emission (lineage is still captured and stored locally). |
| `ontul.lineage.openlineage.namespace` | `ontul` | Namespace string reported in emitted OpenLineage events, used by the lineage backend to group datasets/jobs under this Ontul instance. |
| `ontul.lineage.openlineage.timeout.ms` | `5000` | HTTP timeout (ms) for the POST that emits a single OpenLineage event. Kept short so lineage emission never blocks query completion for long. |

## Client / SDK Properties (`ontul.user.*`)

These properties are **not** used by the server directly. They are set on the client side — by the SDK, JDBC driver, or job submission — via system properties (`-D`) or environment variables. They are listed in `ontul.properties` for reference only.

| Property | Default | Description |
| --- | --- | --- |
| `ontul.job.driver.mode` | _(commented / optional, default `master`)_ | Job driver delegation mode: `master` (interactive) or `worker` (submit). Set per-job in the job config, not as a server property. |
| `ontul.master.flight.port` | _(commented / optional, default `47470`)_ | Flight SQL port used by the SDK to connect to the Master. The server uses `ontul.master.flight.sql.port`; this is the client-side equivalent passed as a system property when launching jobs. |
| `ontul.user.accessKey` | _(commented / optional)_ | Client-side credential: IAM access key id used by the SDK / submitted jobs to authenticate to the Master. Set via `-D` or env var on the client. Paired with `ontul.user.secretKey`. |
| `ontul.user.secretKey` | _(commented / optional)_ | Client-side credential: IAM secret key paired with `ontul.user.accessKey`. Treat as a secret; set on the client side only. |
| `ontul.user.token` | _(commented / optional)_ | Client-side credential: pre-obtained JWT bearer token for SDK/JDBC auth, as an alternative to access/secret key login. Set on the client side only. |
| `ontul.user.stsToken` | _(commented / optional)_ | Client-side credential: short-lived STS temporary token for SDK auth (temporary session credentials). Set on the client side only. |
