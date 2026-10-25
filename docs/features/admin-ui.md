# Admin UI

Ontul includes a built-in web-based Admin UI for monitoring, managing, and operating the cluster.

## The console shell

- The sidebar groups the pages (Core Infrastructure, Governance, Integrations, Semantic & AI, Analytics) and
  can be collapsed to icons with the button at its top edge; collapsed links keep their name as a tooltip.
- The version shown under the sidebar and on the login page is the running server's own
  (`GET /admin/health` returns it from the build), not a string compiled into the UI.
- The header shows who is signed in and whether they are an **Administrator** or a **User**, a
  **Change password** button (any user, not only administrators) and **Sign out**. Signing out revokes the
  session's refresh token on the server.
- An address the console has no page for shows *Page not found* instead of an empty screen.
- Every side panel (editors, invoke testers) opens at least `min(1100px, 92%` of the window`)` wide.
- The console works the same over plain `http://<host>:<port>` as on `localhost` or `https`: it uses no
  browser API that exists only in a secure context (a query id is made with `crypto.getRandomValues`,
  copy buttons fall back to a hidden text area when the Clipboard API is not available). An error
  thrown inside a page is shown in a red bar at the top of the page instead of only in the browser
  console.
- An expired access token is renewed with the refresh token without leaving the page. Login, refresh
  and logout are always served by the leader master, so behind a load balancer a refresh never lands on
  a master that does not know the session.

Dates, durations and counts are rendered from the server's values; a missing or out-of-range value is
shown as `—` / `Never`, never as `Invalid Date`, `NaN` or a 1970 date.

## Opening pages directly

Every page has its own URL under `/admin/` (for example `/admin/audit`, `/admin/lineage`,
`/admin/maintenance`) and can be bookmarked, shared, or reloaded. Some page URLs are also REST API
paths — `GET /admin/audit` is both the Audit page and the audit-list API — so the server answers by
the kind of request: a browser opening the page gets the UI, while API calls (from the UI itself, the
CLI or scripts) get JSON as before. Paths under `/admin/auth/` (login and SSO redirects) are always
handled by the server.

## Who can change what

Every admin API call is checked on the server, whatever the console shows. Hiding a button is a
convenience; the check is what keeps a non-administrator out.

| Area | Read | Change |
|---|---|---|
| KMS, Backup & Restore, STS, node logs (Topology **Logs**) | administrators only | administrators only |
| IAM (users, groups, policies, keys), SSO, connections, catalogs, drivers, storage and other `/admin/config/*` settings, Iceberg maintenance, health scan, audit settings, cluster maintenance mode | any signed-in user (secrets masked) | administrators only |
| SQL query history, jobs (console and `/v1/api/job/*`: list, history, status, log) | your own entries; administrators see everyone's | your own queries and jobs; killing another user's job needs `data:KillJob` |
| Audit events | your own events; administrators see everyone's | settings: administrators only. Recording an event (`POST /v1/api/audit`) for another user, and recording lineage (`POST /v1/api/lineage`), need administrator access |
| Your own access key download | the key's owner or an administrator | — |

"Administrator" means a user whose policies grant `AdministratorAccess`. A non-administrator who calls
an administrator-only route gets `403 Forbidden`. The check runs after a follower master forwards the
request to the leader, so it gives the same answer on every master. When no leader is known — during
an election — a follower answers `503 Service Unavailable` rather than guessing a master.

### Secrets are never sent back

Connection properties, catalog properties, job configurations, query history and SSO settings are
masked before they leave the server. Any key whose name contains `secret`, `password`, `passwd`,
`token`, `credential`, `apikey` / `api_key`, `accesskey` / `access_key`, `private`, `jaas` or
`authheadervalue` (case-insensitive) is shown as `***`. The same masking is applied inside SQL text, so
`CREATE USER ... PASSWORD '...'` or a `key=value` string inside a job argument appears as
`PASSWORD '***'` in history.

Editing a masked value works in both forms:

- **Leave `***` in place** — the stored secret is kept. You can rename a catalog's description or
  change an endpoint without re-typing its secret key.
- **Type a new value** — it replaces the stored one.
- **Clear the field** (connections) — the key is removed from the connection.

KMS key status lists each key's versions with their creation time and state only. No key material is
returned by any endpoint.

### Access keys and tokens

Each key shows when it was created and when it expires (*No expiry*, *Expires <date>*, or *Expired <date>*
in red). The server accepts an expiry in epoch milliseconds; `0`, a missing value, or a value past year
9999 (some clients send `Long.MAX_VALUE/2` for "forever") means the key never expires, and is listed as
`0`. An expiry already in the past, a non-numeric value, or a key for a user that does not exist is
refused (`400` / `404`) rather than creating a key that is dead on arrival. The **Expiration Date** picker
refuses a date in the past or one that is not a valid date.

Deleting an access key, or a user, revokes the key's token immediately: a request presenting it is
refused from the next call on, not when a cache expires. A token is accepted only while its key exists,
is **Active** and is not past its expiry date. An expiry date picked in the UI means *to the end of that
day*. The download icon for a key is shown only to an administrator and to the key's owner, and the
server enforces the same rule.

## Pages

### Dashboard

Stat cards for the number of masters and workers, and per-node charts of the last hour: **CPU Usage (%)**,
**Memory Usage (MB)** (JVM heap), **Query Count**, **Active Queries**, **Job Count (cumulative)** and **Active
Jobs**, refreshed every 15 seconds. **All / Masters / Workers** (or a click on a stat card) limits the chart
lines to that kind of node. The charts appear once a few samples have been collected.

### Topology

The **Cluster Topology** page lists every node the cluster depends on, refreshed every 10 seconds:

- **Masters** — node ID, host, admin and internal ports, role (**Leader** / Follower) and status, with a
  **Logs** button that tails the node's log (administrators only — node logs carry every user's
  statements). The log viewer polls every two seconds; **LIVE** / **PAUSED** stops and resumes it.
  **Refresh** re-reads every list at once.
- **Workers** — node ID, host, internal and Flight ports, status and **Logs**.

**Logs** shows that node's own log, whichever master served the page: a follower's log is fetched from
the follower itself, and a worker is matched by host *and* port, so two workers on one host do not show
each other's log. The viewer reads complete lines only, follows the file as it grows, and starts again
from the end if the file is truncated or rotated. ZooKeeper servers have no log viewer.
- **ZooKeeper** — every server of the ensemble in `ontul.zk.serverList`, whether or not Ontul is
  currently connected to it. Each row shows:

    | Column | Meaning |
    |---|---|
    | Server | `host:port` from the connect string |
    | Role | `leader`, `follower`, `observer` or `standalone`, as the server reports it; `unknown` when it answers but does not serve (no quorum) or has the `srvr` command disabled |
    | Status | **Online** when the server answered, **Offline** when it could not be reached within `ontul.zk.admin.probeTimeoutMs` — the probe error is shown under the badge |
    | Version | ZooKeeper version |
    | Connections | client connections on that server |
    | Latency min/avg/max | request latency the server measured, in ms |
    | Znodes | number of znodes |
    | Zxid | the last transaction id the server has seen; a follower far behind the leader is lagging |

    The header shows the connect string, how many servers are up (`2/3 UP`), and the **ZooKeeper
    session** of the master that answered — `CONNECTED`, `RECONNECTED`, `SUSPENDED` (connection lost,
    session may still be alive) or `LOST` (session expired) — with its session ID.

The ZooKeeper rows come from ZooKeeper's own `srvr` four-letter command, sent by the master to each
server over a plain socket — the same answer `echo srvr | nc host 2181` gives. Nothing is written to
ZooKeeper. `srvr` is in ZooKeeper's default `4lw.commands.whitelist` since 3.5; an ensemble that removed
it still shows its servers as Online, with the role `unknown` and the server's refusal as the note.
A probe is reused for `ontul.zk.admin.probeCacheTtlMs` (5 s by default), so many open admin pages do not
add load to the ensemble. Any master answers on its own: unlike other admin requests, this one is not
forwarded to the leader, because finding the leader needs ZooKeeper. So the ZooKeeper section keeps
updating for an open page even when the ensemble has lost quorum and the Masters and Workers lists stall —
the case it exists for. (Logging in needs the leader, so open the page before you need it, or use the API
with a token you already hold.) The same data is available as JSON from `GET /admin/nodes/zookeeper`:

```json
{
  "connectString": "zk1:2181,zk2:2181,zk3:2181",
  "servers": [
    {"host": "zk1", "port": 2181, "status": "UP", "mode": "leader", "version": "3.9.1",
     "zxid": "0x100000045", "connections": 5, "outstanding": 0,
     "latencyMinMs": 0, "latencyAvgMs": 0.42, "latencyMaxMs": 7, "nodeCount": 137,
     "probeRttMs": 3, "error": null},
    {"host": "zk3", "port": 2181, "status": "DOWN", "mode": null, "error": "ConnectException: Connection refused", "...": "..."}
  ],
  "session": {"state": "CONNECTED", "connected": true, "since": 1760774400000, "sessionId": "0x1000a3b2c0001"},
  "reportedBy": "ontul-master-1:8080"
}
```

Also home to the **Enter / Exit Maintenance** switch. Turning it on refuses writes across every
Master while leaving reads, settings and job control available; a banner stays on screen for as long
as the window is open. See [Cluster Maintenance Mode](cluster-maintenance.md).

### SQL Query

Built-in SQL editor: `Ctrl+Enter` (or **Execute**) runs the statement, `Tab` inserts two spaces, and the
result grid pages 50 rows at a time (previous / next, rows numbered across pages) with each column's
Arrow type under its name and `NULL` shown as such.

- **One statement, written as in any SQL client.** A trailing `;` (with whitespace or a comment after
  it) is accepted — `select * from iceberg.stream.events;` runs as is. A second statement after the `;`
  is refused with *Only one statement can be run at a time*. A leading comment line is fine.
- **Names** resolve case-insensitively (`SELECT * FROM ICEBERG.STREAM.EVENTS`), and a namespace or column
  that is a reserved SQL word — `stream`, `default`, `year`, `value`, `timestamp` — needs no quoting when
  it is part of a dotted name (`iceberg.stream.events`, `e.timestamp`).
- **Row cap.** The page receives at most `ontul.admin.query.max.rows` rows (default 1000). The cap is
  planned into the query as a `LIMIT` — added when the query has none, lowered when it is larger — so the
  engine stops reading once it has the rows; a `select *` over a large, still-growing streaming table
  answers in seconds instead of reading the whole table into one response (which used to run the master
  out of memory). When the result was cut, the header says *Showing the first 1000 rows (result
  truncated …)*. Aggregates such as `count(*)` are not affected. For a full result use a job, JDBC /
  Flight SQL or `POST /v1/api/sql`, which are not capped.
- **Feedback.** The moment you click **Execute** the button spins and **Cancel** appears.
- An empty result (a `WHERE` that matches nothing, a table a streaming job has not committed to yet)
  shows the column headers and *0 rows*.
- A `DOUBLE` keeps its decimal point (`3.0`, not `3`), the same in the Catalog Browser.

- **Cancel** stops the running query on the cluster, not only in the browser. The page chooses the
  query's ID before sending it, so it can cancel a query whose first result has not arrived yet.
- Statements that return no rows — `INSERT`, `CREATE`, `DROP`, `CALL` and so on — show a banner with
  the command tag and row count instead of an empty table.
- A failed statement shows the error and how long it ran.
- Integers beyond JavaScript's exact range (±2<sup>53</sup>−1, for example a large `BIGINT` ID or an
  exact `SUM`) are sent as strings, so the page shows the digits the engine computed rather than a
  rounded number.
- History lists your own queries (administrators see all), newest first, with their command tag,
  duration, row count (kept by the server, so it is still there after a reload) and the time they ran; clicking one puts it back in the editor. Secrets in the SQL
  text are masked. **Clear view** hides the list in this browser; the server keeps its history.

### Catalog Browser

Point-and-click walk through a catalog: **Catalogs → Schemas → Tables**, then the selected table's
column list and a sample of up to 100 rows. It is the counterpart to typing `SHOW SCHEMAS`,
`SHOW TABLES`, `DESCRIBE` and `SELECT` by hand — for the times you do not yet know what to type.

Names are quoted when the page builds its statements, so a schema called like a reserved word (Lance's
`default` namespace) or an index with a `-` in it browses like any other. `SHOW SCHEMAS`, `SHOW TABLES`
and `DESCRIBE` accept quoted names (`SHOW TABLES FROM "tpch"."tiny"`), and name an unknown catalog as an
error instead of returning an empty list. In queries, a dotted name with `default` as a later part
(`lance.default.events`) needs no quoting.

It drives the same query endpoint the SQL Query page does, because catalog metadata is answered by
the query engine; there is no separate introspection API, and adding one would give two code paths
that could disagree about what a table looks like. A statement that fails is shown with the
statement that failed rather than leaving an empty pane. A table that describes but will not read —
permissions, a broken storage path — keeps its column list and reports only the preview as failed.

### Catalogs

Register, unregister, and manage data source catalogs. View connector type, connection ID, table count, and configuration for each catalog.

- **Register first, persist after.** A new or edited catalog is only saved once its connector has
  started. A catalog whose endpoint or credentials are wrong is rejected with the connector's error,
  and editing an existing catalog with bad settings leaves the previous, working catalog in place.
- **Iceberg** catalogs take a REST (e.g. Polaris), Glue, JDBC or Hadoop catalog. Storage settings come
  either from a [connection](connection-id.md) or inline: S3 endpoint (optional for AWS), region,
  access/secret key and path-style access, which is on by default for S3-compatible stores. When a
  connection is chosen the inline keys are not sent, so the connection stays the only source of
  credentials. The REST form sends `catalog.type=rest`; a catalog that has `catalog.rest.uri` but no
  type is also treated as REST.
- **Lance** with Polaris takes the Polaris server root as its URI plus the warehouse name.
- A connector type the form does not know is edited as raw JSON properties, so nothing is lost by
  opening it in the UI.
- Masked secrets are restored from the stored catalog when you save without retyping them.

### Connections

Manage connections of eight types — S3, JDBC, Kafka, REST, NeorunBase, Elasticsearch, RERANK (retriever
re-ranking endpoint) and DECIDE (`ai_verdict` decision endpoint) — create, update, delete and list.
Credentials are encrypted at rest via KMS. A card shows the first four properties (secrets masked) and
`+N more`.

- Connection IDs may contain letters, digits, `_`, `.` and `-`.
- Secret fields are always shown as password fields and come back as `***`; see
  [Secrets are never sent back](#secrets-are-never-sent-back) for how edits treat them.
- **Test** (the bolt icon on every card) really connects with the stored credentials, and shows the answer
  next to the connection:

    | Type | What Test does |
    |---|---|
    | S3 | checks the configured bucket exists and can be listed (`HeadBucket`, then `ListObjectsV2`), or lists buckets when no bucket is set, with the endpoint, region and path-style settings jobs use |
    | JDBC | opens a connection (uploaded drivers included) and reports the database product and version |
    | NeorunBase | logs in over pg-wire and checks the admin endpoint's `/admin/health` |
    | Kafka | asks the cluster for its brokers, cluster id and topics (`AdminClient`), with the SASL settings of the connection |
    | REST, RERANK, Elasticsearch | `GET` of the base URL / first host; any answer below 500 counts as reachable |
    | DECIDE | asks the endpoint one real question and reports the model that answered |

    Each check gives up after `ontul.connection.test.timeout.ms` (5 s). A type with no check says so.
- A connection type the form does not know keeps all of its properties when edited (token- and
  password-like properties as password fields).
- Saving clears the typed secrets from the form, so they do not stay in the page after the panel closes.

### Semantic Layer

Register and govern [semantic views](semantic-layer.md) — curated metrics and dimensions over a
base SQL view. Each entry shows its trust badge, its metrics (a shield icon marks role-gated
ones), dimensions, tags and mandatory filters. **Certify** signs a view off, stamping the logged-in
user and a fingerprint of its semantics; the badge turns **STALE** (orange) if the view is edited
afterwards, with the reason on hover.

### Retrievers

Manage [retrievers](retrievers.md) — governed multi-modal retrieval (vector / graph / full-text)
pushed down to a NeorunBase catalog. The side panel covers the SQL template, the typed parameter
contract, output columns, an optional re-rank block, synonyms and `allowedRoles`. An **Invoke**
drawer runs a retriever with structured arguments and shows the rendered SQL next to the rows.

### Object Types

The [ontology](ontology.md) entity layer: typed business objects (Customer, Order) mapped onto
physical tables. Each row shows the read source, the property→column bindings, the write target
(system of record) and a trust badge. **Certify** signs the definition off; the badge shows
**STALE** with a tooltip when the read source or a property binding changed after certification.

### Link Types

Typed relationships between object types, bound either to relational join keys (`JOIN`) or to a
native NeorunBase graph edge (`GRAPH`). A link's trust is capped by both endpoint object types, so
a certified link over a draft object type displays as DRAFT with the offending endpoint named.

### Action Types

Governed write-backs — the only way an agent changes data. Each action declares its target object
type, its mode (`DML` against the Iceberg system of record, or `OPERATION` calling a REST/ERP
connector), a typed parameter contract, and whether it requires approval. An **invoke tester**
drawer runs one with real arguments and an optional idempotency key, and shows the server's full answer
(for OPERATION, the system's status code and response; a failed call answers `ok: false`). Parameters
are validated by type (`LONG`, `DOUBLE`, `BOOLEAN`, `IDENT`), so a value cannot inject SQL. An
idempotency key is scoped to the caller and the action: the same key on another action runs that action.

### Action Workflows

Multi-step Sagas composed of action types, authored as kiok-style YAML. Clicking a workflow opens its
YAML in the same vocabulary the editor documents — steps with `id`, `action`, `args`, `requires`,
`compensate`, `compensateArgs` — without server-owned fields (certification, timestamps). Each card draws
the DAG; clicking a step opens the referenced action type to view or edit (or says it is not registered).
**Invoke** runs the workflow with a JSON input and an optional idempotency key and draws the outcome
step by step; on a step failure the completed steps are compensated in reverse order and the run shows
`rolled_back` with the compensations. Idempotency keys are scoped to the caller and the workflow.

### Certification on every ontology page

Retrievers, Link Types, Action Types and Action Workflows show the same trust badge as Object Types
and Semantic Views — **DRAFT**, **CERTIFIED** (with who signed it off), **STALE** (edited after
sign-off; the reason is on hover) or **DEPRECATED** — and administrators get the same **Certify**
button (`POST /api/v1/<kind>/{fqn}/certify`). On a STALE entry the button reads *Re-certify*. The
badge is the server's `effectiveStatus`, so a link over a draft object type still shows DRAFT.
Certifying needs administrator access or the `ontology:Certify` action.

### Ontology Graph

Visual type-graph of object types and the link types connecting them — the schema-level map an
agent traverses, rendered for humans. Nodes list their properties (★ marks the primary key); JOIN edges
are solid, GRAPH edges dashed and animated. Clicking a node shows its read source, property → column
bindings and links (clicking a link moves to the other type); **Focus** isolates the node and its
neighbours; the namespace selector shows one `catalog.schema` at a time.

### Flow

The visual [Ontul Flow](../streaming/flow.md) builder and operator console: a drag-and-drop
`source → transform → sink` canvas backed by an editable JSON/YAML spec (both directions stay in
sync). The list view shows every registered flow with its status, **Records**, **Rejected** and
**Lag** columns, plus Start / Stop / Delete. Opening a flow gives Spec, Data (preview rows from a
source or sink node), Metrics (live throughput sparkline, rejected count, source lag), Logs
(auto-scrolling tail) and History (past runs with duration and rejected counts). A running flow is
read-only — stop it to edit.

The Iceberg sink's write mode (`append`, `upsert`, …) is saved in the spec as `sink.write`, together with
`transactional` and `batchSize`, so it survives a save and reopen. An Iceberg source starts from
**latest** (only new snapshots) or **all** (`earliest` is accepted as a synonym). Saving a new flow
under a name that is already taken answers `409` instead of replacing the other flow.

### Data Lineage

Table-level lineage graph built from executed DML, views, started flows and edges pushed by other
engines: which sources feed which targets, each edge labelled with the operation that introduced it
(`CTAS`, `INSERT_INTO`, `MERGE_INTO`, `VIEW`, …). Enter a table, pick direction and depth; click a node
for its column-level lineage and downstream impact. See [Data Lineage](data-lineage.md).

### Audit

Two pages over the [audit log](audit-log.md):

- **Overview & Graph** — counters, the storage & retention settings (including S3 tiering), and a
  `user → engine → table` relationship graph with drill-down to the related events.
- **Search** — free text plus user, engine, table, read/write, time range and max-results filters.

Both pages take a **time range** (*Any time*, *Last hour*, *Last 24h*, *Last 7 days*, *Last 30 days*,
or **Custom range…** with From / To pickers) and a maximum number of events, and say so when a result
is cut at that maximum.

On the overview, **Fetch max** (100 / 400 / 1000 / 5000, capped by `ontul.audit.viewer.max.page`) is
how many of the newest events are fetched from the server — the toolbar says how many were — and the
**Recent events** table and the drill-down table each have a **Rows** selector (50 / 100 / 500, default
100) with **Prev** / **Next** and *Showing X–Y of N*, so every fetched event can be paged through in
the server's order. A non-administrator sees only their own events, and the settings are read-only
for them. They show local and S3-tiered events as one log, so turning tiering on — even
with no local copy kept — does not empty the page. Denied attempts appear alongside allowed ones, so a
refused certification or an IAM-blocked write is visible here.

### Lance Maintenance

Compaction and cleanup for [Lance](lance-integration.md) datasets: **OPTIMIZE** (fragment compaction;
`target_rows_per_fragment`, streaming window, cooldown, `min_input_files`, `materialize_deletions`,
`defer_index_remap`, `dry_run`) and **VACUUM** (reclaim old versions; `retain_last`, `older_than_days`,
`before_version`). VACUUM asks for confirmation showing the exact statement. Recent runs list each
statement with its result and duration.

### Storage

Switch the dependency, spill, exchange and job-log backends between local disk and S3 **at runtime,
cluster-wide, with no restart** — the change is persisted to the replicated metadata store and pushed to
every node. One S3 connection and base prefix cover all four (`<prefix>/spill`, `<prefix>/job-logs`,
`<prefix>/exchange`). **Test Connection** checks the bucket of the prefix with the stored keys. Job logs of
work running on workers are shipped to `<prefix>/job-logs` once S3 is chosen.

With S3 chosen, the job **history** (each job's final state) is kept on the master's disk and also copied
to `<prefix>/job-logs/jobHistory/` with the same connection's keys, so a job's status reads
`COMPLETED` / `FAILED` from any master behind a load balancer, and after a master is replaced. (It used
to switch the history to S3 through a file-system client with no S3 support and no keys: every job then
ended without a history and its status read `UNKNOWN`.) A status request that lands on a master which
did not run the job asks the master that did before falling back to the S3 log segments.

### Drivers

Upload JDBC driver JARs for federated catalogs. Drivers are stored durably and synced to every
worker, so a new driver does not require rebuilding an image.

- **An uploaded driver wins over a bundled one.** Ontul ships some JDBC drivers (PostgreSQL among them).
  Each uploaded JAR is loaded child-first, and a driver class found in an uploaded JAR is used before
  the bundled copy, so uploading `postgresql-42.7.4.jar` really switches the version. Driver classes are
  read from the JAR's `META-INF/services/java.sql.Driver`.
- Uploading a newer JAR for the same driver class under a different file name **supersedes** the old
  JAR: it is removed on the master and on every worker.
- **Delete** removes the JAR from the master and from every worker, and answers `404` for a name that
  does not exist.
- File names must end in `.jar` and may not contain path separators or `..`.

### Jobs

Monitor active and completed jobs, view real-time logs, and kill running jobs. Jobs are submitted with
the SDK, `bin/submit.sh`, the REST API or `SUBMIT STREAMING` in the SQL editor; the page is for watching
and stopping them. Tabs: **Active**, **History** and **Queries** (each with its count). Rows show status,
type, driver (MASTER / WORKER), user, start / end time and duration; the log icon opens the job's log
(**LIVE** while it runs, then the full log: from S3 when job logs are stored there, otherwise from the
worker's memory, which keeps a finished job's lines for 5 minutes — choose S3 on the Storage page to keep them). **Kill** asks for confirmation with the
job's details, stops the job on every worker running it and reports a failure instead of silently doing
nothing. Clicking a query opens its detail (status, command, duration, rows, time, id, error). **Live** /
**Paused** turns the 3-second refresh on and off. Non-administrators see their own jobs, and secrets in
job configuration (a Flow's sink password, for example) are masked — in the REST API too.

### IAM

Manage users, groups, and policies. Includes a visual policy editor for creating structured IAM policies with column-level and row-level security rules.

- The policy editor covers `Allow`, `Deny` and `Mask` statements. `Mask` takes the columns to mask
  (`MaskedColumns`). Actions are picked from the known action list or typed by hand.
- Opening a policy and saving it keeps every field, including ones the editor has no control for, so
  editing a policy written by the API or the CLI does not drop conditions.
- **New Policy** refuses a name that already exists (`409`) instead of overwriting it.
- The editor stays open with the server's error when a save fails.
- Groups list their members and count them; a member is removed with the × on hover, a policy detached
  with the × on its chip.
- The built-in `admin` user cannot be deleted and is marked *protected*.
- User names `system` (the principal background maintenance runs as) and names containing `/` are
  refused, and passwords must have at least 6 characters — checked by the server, not only the form.
- **STS Sessions**: generate temporary credentials from one of the access keys with a TTL of 1 minute to
  12 hours and an optional session name. The new credentials show the secret and session token on demand
  (eye icon), copy buttons and a CSV download; the table shows each session's creation time, remaining TTL
  and status, and **Clear Expired** removes expired ones. Temporary credentials authenticate over Flight SQL
  and on `POST /v1/api/auth/accesskey` with `accessKey`, `secretKey` and `stsToken`; they stop working
  when their parent key is revoked. The server refuses a duration outside
  `ontul.sts.min.duration.seconds`–`ontul.sts.max.duration.seconds` (60–43200).

### Single Sign-On

Configure OIDC, SAML 2.0 and LDAP / Active Directory, on tabs of their own. Every value is a named
field — nothing here asks for JSON. Secrets (OIDC client secret, LDAP bind password, SAML SP private
key) are never read back; each shows only whether it is set, and typing replaces it.

The SAML tab is an exchange of documents rather than a form: download this cluster's SP metadata for
the provider to import, paste the provider's metadata to have the entity ID, sign-on URL and signing
certificate read out of it, and generate an SP keypair when the provider needs encrypted assertions
or signed requests. Each provider has a **test** button, so a mistake surfaces before anyone tries to
log in with it.

Settings are stored in the replicated metadata store and applied on every master with no restart.
Changing them needs `AdministratorAccess`; everyone else sees the page read-only. See
[Single Sign-On](sso.md).

### KMS

Key management interface for viewing and managing encryption keys: create keys, rotate them and see
each key's versions. Administrators only. Key IDs are trimmed and checked for duplicates before
creation. No key material is ever shown.

### Maintenance

Configure and monitor Iceberg table maintenance — snapshot expiration, data compaction, manifest rewrite, orphan-file cleanup, and position-delete consolidation. Per-table configuration covers per-operation toggles plus Spark-aligned parameters (target file size, compaction `window_hours` and `min_input_files`, snapshot retention and `retain_last`, orphan safety window) and a **schedule** that is either a fixed interval or a 5-field UNIX **cron** (e.g. `0 */2 * * *`) which overrides the interval. A **Manual Trigger** runs any single operation (or all) on demand against a wildcard table pattern, with full job history.

Settings are validated on save, and an invalid value is reported rather than stored: snapshot retention
and `retain_last` at least 1, the orphan-file safety window at least 24 hours (shorter windows can delete
files a running write has not committed yet), and a valid 5-field cron expression. The table field
accepts any table name and suggests tables seen in health data and maintenance history. After **Run
Now**, the history refreshes until the run finishes.

### Backup & Restore

Backup the cluster's KMS / IAM / metadata stores and connections to S3 and restore from any prior backup. The page exposes manual *Backup Now*, a fixed-interval schedule, and a 5-field UNIX **cron schedule** (e.g. `0 2 * * *`); the next-fire timestamp is shown next to an active cron so you can see at a glance when the next run will land. See [Backup & Restore](backup-restore.md) for the full flow.

Backups are listed newest first with their UTC time; **Restore** asks for confirmation listing what is
overwritten (KMS, IAM, metadata, connections). The job history shows every backup and restore with its
time and duration. The form stays disabled until the saved configuration has loaded, so a value typed
early is never overwritten by it. *Backup Now* needs a saved backup configuration. The backup list reports the storage error — a missing
bucket, wrong credentials — instead of showing an empty list. Scheduled and cron backups run on the
leader only, so a multi-master cluster takes one backup per run.

## REST API

All operations available in the Admin UI are also accessible via the REST API, enabling automation and integration with external tools. The full API is documented in OpenAPI 3.0 format.

## Prometheus Metrics

Ontul exposes metrics at `GET /metrics` in Prometheus text format:

- `ontul_queries_total` / `ontul_queries_failed` / `ontul_queries_active`
- `ontul_latency_ms_sum`
- `ontul_workers_total` / `ontul_workers_ready`
- JVM heap and thread metrics
