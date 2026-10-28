# Identity and Access Management

Ontul includes a built-in IAM system that provides authentication, authorization, and fine-grained access control down to the column and row level. Column masking, row-level filters, and metric-level RBAC (when the semantic layer is in use) all read from the same policy store, so a single statement governs what a user sees regardless of whether the query arrives over Arrow Flight SQL, REST, or the MCP server.

## Authentication

Ontul supports five authentication methods that can co-exist on the same cluster:

| Method | When to use | Header / field |
| --- | --- | --- |
| **Username / Password** | Interactive UI / SQL clients (Tableau, DBeaver) that only know Basic auth. PBKDF2-hashed passwords. | `authorization: Basic base64(user:pass)` |
| **JWT Bearer Token** | Short-lived sessions (default 15 min). Issued by `POST /admin/auth/login`. Used by the Admin UI and Java SDK. | `authorization: Bearer <jwt>` |
| **Access Keys** (`AKIA…`) | Long-lived service-account credentials with separate `accessKeyId` + `secretAccessKey`. Issued via `POST /admin/iam/keys`. | `authorization: AccessKey AKIA…:<secret>` |
| **STS Temporary Credentials** (`ASIA…`) | Short-lived AKIA-style credentials with configurable expiry — used by transient workloads (CI runners, ETL jobs). | Same as Access Keys, prefix differs. |
| **Federated identity** (SSO) | An account that lives in your identity provider rather than here — OIDC, SAML 2.0, or LDAP / Active Directory. Exchanged for a bearer token at `POST /v1/api/auth/sso`, or obtained through a browser redirect in the console. | Same as JWT Bearer Token. See [Single Sign-On](sso.md). |

The MCP server and the SDK accept whichever credential is supplied via `ONTUL_USER_TOKEN`. Whatever IAM policies attach to that credential apply automatically — there is no separate authorization layer to configure on the client.

### Default admin and password rotation

The cluster is bootstrapped with a default `admin` user. When the master starts with the literal password `admin` (the out-of-the-box value), the user is flagged `requirePasswordChange=true` and **every privileged REST and Flight SQL endpoint returns 403 until the password is rotated**. This gate applies only to:

1. The default admin (until first rotation), and
2. Any user whose password was just reset by an administrator via the local admin-socket recovery channel.

Users created normally with `POST /admin/iam/users` start with `requirePasswordChange=false` and can call the REST / Flight SQL APIs immediately. The flag never applies to access keys — those carry their own random secret and don't need rotation before use.

## Users and Groups

Users carry an opaque id, a hashed password, group memberships, access keys, and an optional **attribute map** that backs templated row filters and column masks (`${user.attr.tenant_id}`). Groups exist purely as a policy-attachment vehicle: a policy attached to a group applies to every member, and a user can belong to any number of groups.

| Operation | Endpoint |
| --- | --- |
| Create user | `POST /admin/iam/users` `{username, password, groups?, policyNames?, attributes?}` |
| List users | `GET  /admin/iam/users` |
| Delete user | `DELETE /admin/iam/users/{username}` |
| Add to group | `POST /admin/iam/add-user-to-group` `{username, groupName}` |
| Create group | `POST /admin/iam/groups` `{groupName, policyNames?}` |
| Issue access key | `POST /admin/iam/keys` `{username, expiresAt?}` |

Creating a group can attach its policies in the same call, and creating a user can place it in groups,
attach policies directly and set its attributes:

```bash
curl -s -X POST "http://<master>:8080/admin/iam/groups" -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' -d '{"groupName": "analysts", "policyNames": ["ReadSales"]}'
# 201 {"status":"ok","groupName":"analysts","policyNames":["ReadSales"]}
```

Everything named is checked **before** anything is stored: an unknown policy or group answers `400` with the
reason (`Policy not found: ReadSalez`, `Group not found: x`) and **no group / user is created**. A field the
endpoint does not act on is refused rather than ignored (`unknown field 'members' (accepted: groupName,
policyNames)`), and a missing required field answers `400` naming it (`groupName is required`, `username is
required`, `password is required`). Earlier releases answered `201` for a group created with `policyNames`
while dropping the policies.

In the Admin UI each group row has an **Attach Policy** button (administrators only) next to the delete
button; expanding the row still shows the attached policies with their detach buttons and *Attach Custom*.

!!! note "Field-name aliases"
    `POST /admin/auth/login` accepts the user identifier as either `username` or `user`, and `POST /admin/iam/keys`, `POST /admin/iam/users`, `add-user-to-group` and `remove-user-from-group` accept either `username` or `userId`. The canonical field is `username`; the aliases exist so clients and SDKs that use a different field name interoperate without a mapping layer.

## Policy-Based Access Control

Ontul uses AWS-style JSON policies to manage permissions. Actions live in the `data:` namespace (table reads, DML, admin verbs), the `UDF:` namespace, and the `ontology:` namespace (governed write-backs and certification); table resources are addressed as `data:table:<catalog>.<schema>.<table>` and accept wildcards.

```json
{
  "Version": "2024-01-01",
  "Statement": [
    {
      "Sid": "SalesReadWrite",
      "Effect": "Allow",
      "Action": ["data:Select", "data:Insert"],
      "Resource": "data:table:ice.sales.*"
    },
    {
      "Sid": "HideSalarySsn",
      "Effect": "Deny",
      "Action": "data:Select",
      "Resource": "data:table:ice.hr.salaries",
      "Columns": ["salary", "ssn"]
    }
  ]
}
```

- **Action verbs in `data:` namespace** — runtime SQL: `data:Select`, `data:Insert`, `data:Update`, `data:Delete` `data:Merge`. Catalog DDL: `data:CreateTable`, `data:DropTable`, `data:AlterTable`, `data:CreateSchema`, `data:DropSchema`, `data:CreateView`, `data:DropView` (legacy aliases still accepted, see the table below). Operational: `data:KillJob`, `data:CancelQuery`. The metadata-only `data:SelectTable` verb gates discovery surfaces (semantic views, object types, retrievers: listing and fetching their definitions) — distinct from the runtime `data:Select` that gates actual scans. A `data:Select` grant on a name also makes it discoverable, so a user who may read a table sees the views and object types built on it; reading table data (including Lance blobs) always needs `data:Select`. Catalog / schema / table listings, `DESCRIBE`, `EXPLAIN` and lineage follow `data:Select` (see [BI integration](bi-integration.md)).
- **UDF actions** stay in the `UDF:` namespace — see [UDF Permissions](#udf-permissions).
- **Ontology actions** live in the `ontology:` namespace and gate the [ontology](ontology.md) surface: `ontology:InvokeAction` (perform a governed write-back), `ontology:InvokeWorkflow` (run a multi-step Saga), `ontology:Certify` (sign off on a definition). Administrators are always allowed; for everyone else these are deny-by-default like any other action.
- **Server-side S3 reads, writes and remote functions** — `data:ReadS3` on `data:s3:<bucket>/<key>` gates `read_s3('<s3 path>', ...)` (the master reads where the caller names, with inline keys, a stored connection's or its own credentials); `data:UploadS3` on the same resource gates the Flight `UPLOAD_S3` action and the `WRITE_S3 '<s3 path>' ...` statement (the master writes where the caller names); `data:MapExecute` on `data:compute:map` / `data:compute:py-map` gates the Flight `MAP_EXECUTE` / `PY_MAP_EXECUTE` actions (the caller's function runs on every worker); the Flight `CancelQuery` action needs `data:CancelQuery` on `data:query:<id>`, like `POST /admin/query/cancel`. All deny by default; administrators pass.
- **Resources**: `data:table:<catalog>.<schema>.<table>`, `data:schema:<catalog>.<schema>`, `data:job:*`, `data:query:*`, `data:s3:<bucket>/<key>`, `data:compute:map`, `udf:<name>`. Wildcards `*` and `?` are supported. There is no `data:catalog:` resource: a whole catalog is `data:table:<catalog>.*.*` plus `data:schema:<catalog>.*`. **Semantic-layer and ontology objects are addressed by their BARE fully-qualified name** — `semantic.rag.*`, `ontology.sales.*` — *not* prefixed with `data:table:`. A policy written as `data:table:ontology.sales.*` never matches, which reads like "IAM is broken" rather than "the pattern did not match".
- **Attachment**: policies attach to users (direct) or groups (via group membership).
- **Deny rules take precedence** over allow rules.
- **Deny-by-default**: no matching policies means access is denied.

### Query-time enforcement (reads **and** writes)

Authorization runs inside the query engine on **every** statement, on **every** interface (Arrow Flight SQL, the REST `POST /v1/api/sql`, the admin `POST /admin/query/execute`, and the MCP server) — clients cannot bypass it. The check is **fail-closed**: a user with no effective policy, an explicit `Deny`, or no matching statement (abstain) is rejected before any data is read or written. Administrators carry a `"*"` resource policy and always pass.

Both the tables a query **reads** and the table it **writes** are authorized, each against the action that matches the operation:

| Statement | Authorized resource(s) | Action |
|---|---|---|
| `SELECT … FROM t` | every scanned source `t` | `data:Select` |
| `INSERT INTO t SELECT … FROM s` | source `s` | `data:Select` |
| | **target `t`** | `data:Insert` |
| `CREATE TABLE t AS SELECT … FROM s` | source `s` | `data:Select` |
| | **target `t`** | `data:Insert` |
| `MERGE INTO t …` | **target `t`** | `data:Insert` |
| `UPDATE t SET …` | **target `t`** | `data:Update` |
| `DELETE FROM t …` | **target `t`** | `data:Delete` |

**Catalog DDL is authorized too.** `CREATE SCHEMA`, `DROP SCHEMA`, `DROP TABLE`, `DROP VIEW`, `CREATE VIEW`,
`ALTER TABLE`, `OPTIMIZE`, `VACUUM` and the other maintenance statements need `data:CreateTable`,
`data:DropTable` or `data:AlterTable` on the schema (`data:schema:<catalog>.<schema>`) or table resource —
see the [action namespace](iam-policy-templates.md#action-namespace). A user with no matching policy is
refused, like any other statement. Scheduled table maintenance runs as the internal principal `system`,
which is not an IAM user and cannot be created as one.

The write **target** is authorized in addition to the read sources: granting a user `data:Select` on a table does **not** let them `INSERT`/`UPDATE`/`DELETE` into it — a separate `data:Insert`/`data:Update`/`data:Delete` grant (or a wildcard such as `data:*`) on that table resource is required. So the `SalesReadWrite` example above (`["data:Select","data:Insert"]` on `data:table:ice.sales.*`) permits both reading and inserting across the `ice.sales` schema, while a read-only policy that lists only `data:Select` blocks every write to those tables.

The engine authorization is the enforcement point the [chango Trino / Spark / Flink authz plugins](../index.md) also call over REST — so the same policy governs a table whether it is touched by an external engine or by an Ontul SQL statement.

### Canonical actions and resources

What each operation is authorized as — the exact `(action, resource)` pair the engine evaluates. The same table is
in the Admin UI policy editor (**Action & resource reference**), and a denial names the pair it checked, e.g.
`Access denied: no matching policy for user 'wf-runner' to data:CreateSchema on 'data:schema:iceberg.wf'`: grant
exactly that.

Action names follow the SQL statement — the same names the chango Trino gateway sends to
`/v1/api/authz/check`. Where Ontul used another name before, that name is a **legacy alias**: an Allow of it still
grants the statement, and a Deny of either the action or its legacy alias denies it, so existing policies keep
working and an existing Deny cannot be stepped around by the rename.

| Operation | Action | Resource | Legacy alias |
|---|---|---|---|
| `SELECT`, `EXPLAIN`, `DESCRIBE`, `SHOW …` (listings are filtered) | `data:Select` | `data:table:<catalog>.<schema>.<table>` | |
| `INSERT INTO` an existing table | `data:Insert` | `data:table:<catalog>.<schema>.<table>` | |
| `CREATE TABLE` | `data:CreateTable` | `data:table:<catalog>.<schema>.<table>` | `data:Insert` |
| `CREATE TABLE … AS SELECT` — filling the new table is part of creating it (no separate `data:Insert`), plus `data:Select` on every source table | `data:CreateTable` | `data:table:<catalog>.<schema>.<table>` | `data:Insert` |
| `UPDATE` | `data:Update` | `data:table:<catalog>.<schema>.<table>` | |
| `DELETE FROM` | `data:Delete` | `data:table:<catalog>.<schema>.<table>` | |
| `MERGE INTO` | `data:Merge` | `data:table:<catalog>.<schema>.<table>` | `data:Insert` (+ `data:Delete` with `WHEN MATCHED THEN DELETE`) |
| `CREATE SCHEMA` / `NAMESPACE` / `DATABASE` | `data:CreateSchema` | `data:schema:<catalog>.<schema>` | `data:CreateTable` |
| `DROP SCHEMA` | `data:DropSchema` | `data:schema:<catalog>.<schema>` | `data:DropTable` |
| `CREATE [MATERIALIZED] VIEW` | `data:CreateView` | `data:table:<catalog>.<schema>.<view>` | `data:CreateTable` |
| `DROP VIEW` / `DROP MATERIALIZED VIEW` | `data:DropView` | `data:table:<catalog>.<schema>.<view>` | `data:DropTable` |
| `DROP TABLE` | `data:DropTable` | `data:table:<catalog>.<schema>.<table>` | |
| `ALTER TABLE` / `VIEW`, `OPTIMIZE`, `VACUUM`, `EXPIRE SNAPSHOTS`, `REMOVE ORPHAN FILES`, `REWRITE MANIFESTS` | `data:AlterTable` | `data:table:<catalog>.<schema>.<table>` | |
| `CREATE INDEX` / `DROP INDEX … ON <table>`, `COMMENT ON TABLE` / `COLUMN` | `data:AlterTable` | `data:table:<catalog>.<schema>.<table>` | |
| `PUBLISH WAP` (a WAP branch appended to main) | `data:Insert` | `data:table:<catalog>.<schema>.<table>` | |
| `INSERT … SELECT`, subqueries of `UPDATE` / `DELETE` / `MERGE`: every table read | `data:Select` | `data:table:<catalog>.<schema>.<table>` | |
| `match()` / `vector_search()` (refused when column / row rules apply to the caller) | `data:Select` | `data:table:<catalog>.<schema>.<table>` | |
| `INSERT` / `UPDATE` / `DELETE` on a JDBC catalog (sent to the database): every other table it names | `data:Select` | `data:table:<catalog>.<schema>.<table>` | |
| `WRITE_S3`, Flight `UPLOAD_S3` | `data:UploadS3` | `data:s3:<bucket>/<key>` | |
| `read_s3()` — whatever credentials it carries (inline keys, a `connectionId`, or the server's own default chain) | `data:ReadS3` | `data:s3:<bucket>/<key>` | |
| Flight `MAP_EXECUTE` / `PY_MAP_EXECUTE` | `data:MapExecute` | `data:compute:map` / `data:compute:py-map` | |
| Cancel a query (SQL page, Flight `CancelQuery`) | `data:CancelQuery` | `data:query:*` (Flight: `data:query:<id>`) | |
| Kill a job (the job's owner needs nothing on `/v1/api`) | `data:KillJob` | `data:job:*` | |
| Streaming flow: source table / sink table (+ `data:Update`, `data:Delete` for CDC / upsert) | `data:Select` / `data:Insert` | `data:table:<catalog>.<schema>.<table>` | |
| Semantic views, retrievers, object types: list / get / invoke | `data:SelectTable` | `<catalog>.<schema>.<name>` (no prefix) | |
| UDF in a query / register / drop (`GLOBAL` scope: `_GLOBAL`) | `UDF:EXECUTE` / `UDF:CREATE` / `UDF:DROP` | `udf:<name>` | |
| `ai_verdict()` / `ai_verdict_one()` | `AI:DECIDE` | `connection:<connectionId>` | |
| `CALL` an action / workflow / operation | `ontology:InvokeAction` / `InvokeWorkflow` / `InvokeOperation` | `<catalog>.<schema>.<action>` (operation: `operation:<catalog>:<op>`) | |
| Certify an ontology object | `ontology:Certify` | `ontology.<schema>.<name>` | |

!!! note "Migrating to the statement names"
    Nothing has to change for existing policies: `data:CreateTable` on `data:schema:…` still allows `CREATE SCHEMA`,
    `data:Insert` still allows `CREATE TABLE` / CTAS and (with `data:Delete` for `WHEN MATCHED THEN DELETE`)
    `MERGE`, and a Deny written with the old name still denies. Saving a policy with `data:CreateTable` /
    `data:DropTable` on a `data:schema:` resource is answered with a note naming `data:CreateSchema` /
    `data:DropSchema`. To migrate, replace the legacy name with the action in the table (or add it next to the
    legacy one); the policy templates already use the new names. `/v1/api/authz/check` answers with the same
    rules, and a `data:CreateSchema` / `data:DropSchema` check on the gateway's `data:table:<catalog>.<schema>`
    spelling is also answered by a `data:schema:<catalog>.<schema>` grant.

How matching works:

- Case-insensitive. `*` matches any characters (including `.` and `:`), `?` exactly one. `"*"` alone matches everything.
- There is **no hierarchy**: `data:table:iceberg.wf.*` does not cover `data:schema:iceberg.wf`. A role that creates its
  own schema needs a `data:schema:` statement as well — see the **ETL Schema Owner (one schema)** template below.
- Names are checked as the engine resolves them, not as written: quotes are removed and case is ignored, so
  `"ICEBERG"."NS"."T"` meets a statement on `data:table:iceberg.ns.t`. A `SELECT` must name `catalog.schema.table`
  (Ontul has no default catalog or schema; `SELECT * FROM t` fails to plan). In DDL / DML a two-part `catalog.table`
  is `catalog.default.table` — what the statement acts on — and is checked as that.
- A statement whose target cannot be read from it (e.g. `COMMENT ON SCHEMA …`), and a JDBC-catalog `INSERT` /
  `UPDATE` / `DELETE` that holds more than one statement or names a table not written as
  `catalog.schema.table` of that catalog, is refused for anyone but an administrator.
- S3 resources are `data:s3:<bucket>/<key>` without the scheme; `data:s3:bucket/team/*` grants everything under
  `team/`. A path that is not `s3://` / `s3a://`, or that has a `.` / `..` segment, is refused whatever the policy says
  (Hadoop would have normalized `team/../secret` to `secret`).
- Administration routes (users, groups, policies, catalogs, connections, KMS, backup, …) are not governed by actions:
  they need the built-in `AdministratorAccess` policy.

A grant for an ETL role that owns one schema (create it, create / load / change / drop its tables, nothing else):

```json
{
  "Version": "2024-01-01",
  "Statement": [
    { "Sid": "EtlSchema", "Effect": "Allow", "Action": ["data:CreateSchema", "data:DropSchema"], "Resource": "data:schema:iceberg.etl" },
    { "Sid": "EtlTables", "Effect": "Allow",
      "Action": ["data:Select", "data:Insert", "data:Update", "data:Delete", "data:Merge", "data:CreateTable", "data:DropTable",
                 "data:AlterTable", "data:CreateView", "data:DropView"],
      "Resource": "data:table:iceberg.etl.*" }
  ]
}
```

### Validation and warnings

Every create and update checks the document the way the evaluator will read it. A document it cannot read as
written is **refused with `400`** and a message naming the statement and field:

- `Statement` must be a non-empty array; only `Sid`, `Effect`, `Action`, `Resource`, `Columns`, `Condition` and
  `MaskedColumns` are allowed in a statement (an AWS-style `NotAction` would otherwise be silently ignored).
- `Effect` must be `Allow`, `Deny` or `Mask`; `Action` and `Resource` are a string or a non-empty list of strings
  without spaces; a `Mask` statement needs `MaskedColumns`, and every mask expression must parse.

Vocabulary is **warned about, not refused**: the document is stored and the answer carries `"warnings": [...]`,
which the Admin UI shows after Save. A warning is given when

- an action in a namespace Ontul owns (`data:`, `UDF:`, `AI:`, `ontology:`) is not one Ontul's engine checks —
  with what it checks instead, and that no engine integration sends it either, e.g. `Statement 2:
  'data:CreateNamespace' is not an action Ontul checks — CREATE SCHEMA / NAMESPACE / DATABASE is authorized as
  data:CreateSchema on data:schema:<catalog>.<schema>. No engine integration sends this either — it has no effect and
  can be removed. ...`. `data:ShowSchemas`, `data:ShowTables`, `data:ShowCatalogs` and `data:Describe` are such
  names: listings and DESCRIBE are authorized as `data:Select`;
- a name only the engine integrations send — a note, not a problem: the gateway's schema form
  `data:CreateSchema` / `data:DropSchema` on `data:table:<catalog>.<schema>` answers `... is not checked by Ontul's
  own SQL (it checks data:CreateSchema on data:schema:...); it is sent by the Trino/Spark/Flink gateway integration —
  keep it if you use those`;
- a legacy name is used — a note, not a problem: `Statement 1: data:CreateTable on 'data:schema:iceberg.wf' is the
  legacy alias of data:CreateSchema (still accepted; data:CreateSchema is the name to use)`;
- a `data:` resource is not a form Ontul checks (`data:catalog:…`, `data:tables:…`);
- none of a statement's resources has the form its action is checked on, e.g. `data:Insert` on
  `data:schema:iceberg.etl` or `data:Select` on `data:schema:…` (`data:Select` is checked on `data:table:…`), so the
  statement never applies — "it has no effect and can be removed".

The console's action reference (policy editor and Policies tab) is served by the server
(`GET /admin/iam/policy-reference`) from the same vocabulary these warnings use.

These are warnings rather than errors because the same policies answer external engines (next section), which
may use names of their own.

### External engines

Trino, Spark and Flink (chango) ask Ontul through `POST /v1/api/authz/check` / `check-batch` with the action and
resource **they** build. The chango Trino gateway uses the names in the table above (`data:CreateSchema`,
`data:DropSchema`, `data:Merge`, `data:CreateView`, `data:DropView`, `data:CreateTable`, …), so one policy governs a
statement in Ontul SQL and from Trino alike. The check applies the same legacy aliases (a `data:Merge` check from an
engine, which cannot tell whether the MERGE deletes rows, takes the legacy `data:Insert` only together with
`data:Delete`). The gateway names a schema `data:table:<catalog>.<schema>`; for `data:CreateSchema` /
`data:DropSchema` on such a two-part name the check also asks `data:schema:<catalog>.<schema>` — an Allow on either
allows, a Deny on either denies.

### Why was I denied?

A denial names the exact action and resource that was checked. When the caller's **own** policies hold a near
miss, the message adds a hint (only the caller's own Allow statements are looked at):

```text
Access denied: no matching policy for user 'wf-runner' to data:CreateSchema on 'data:schema:iceberg.wf'.
  Hint: your policy grants data:CreateNamespace on 'data:schema:iceberg.wf', which Ontul does not check —
  CREATE SCHEMA / NAMESPACE / DATABASE is authorized as data:CreateSchema on data:schema:<catalog>.<schema>
Access denied: no matching policy for user 'etl' to data:Insert on 'data:table:iceberg.wf.t'.
  Hint: your policies grant data:Insert only on 'data:table:iceberg.etl.*'
```

No hint means nothing in the caller's policies comes close: the grant is simply missing.

### Creating, updating and deleting policies

| Operation | Endpoint |
| --- | --- |
| Create policy | `POST   /admin/iam/policies` `{name, document, ifNotExists}` |
| Update policy (keeps attachments) | `PUT    /admin/iam/policies/{name}` `{document}` |
| List policies | `GET    /admin/iam/policies` |
| Delete policy | `DELETE /admin/iam/policies/{name}[?detach=true]` |
| Attach policy to group | `POST   /admin/iam/attach-group-policy` `{groupName, policyName}` |
| Detach policy from group | `POST   /admin/iam/detach-group-policy` `{groupName, policyName}` |

All of them are administrator-only, handled by the leader (a follower forwards the request), replicated to every
master, and recorded in the audit log (`iam:CreatePolicy` / `iam:UpdatePolicy` / `iam:DeletePolicy` on
`iam:policy:<name>`).

**Update in place.** `PUT /admin/iam/policies/{name}` replaces the document and keeps the name and **every group and
user attachment**, so fixing a policy reaches its members on their next request — no re-attaching, no new token.
`404` if there is no such policy, `400` for an invalid document, `403` for the built-in `AdministratorAccess`. The
answer lists where the policy is attached:

```bash
curl -X PUT $ADMIN_URL/admin/iam/policies/wf-runner-policy -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" -d '{"document": { … corrected document … }}'
# 200 {"status":"ok","policyName":"wf-runner-policy","attachedGroups":["wf-runners"],"attachedUsers":[]}
```

Do not delete and re-create a policy to change it: a re-created policy is attached to nothing.

**Create.** With `"ifNotExists": true` (what the Admin UI sends) an existing name is refused with
`409 Policy 'x' already exists; update it with PUT /admin/iam/policies/x (or Edit in the console), which keeps its
group and user attachments`. Without it, `POST` still replaces the document of an existing policy (attachments are
kept), as it always has, for scripts that rely on it; use `PUT` for updates.

**Delete.** A policy still attached to a group or user is **refused** with `409`, naming them:

```json
{"error": "Policy 'wf-runner-policy' is attached to group(s) wf-runners; detach it first, or delete with ?detach=true to detach it from all of them. ...",
 "attachedGroups": ["wf-runners"], "attachedUsers": []}
```

Deleting an attached policy changes what every member may do at once — and widens it when the policy holds a
`Deny` — so it is never done silently: detach it first, or send `DELETE …?detach=true` to detach it from all of
them and delete it in one step (the answer lists `detachedGroups` / `detachedUsers`). The Admin UI's **Delete**
shows the groups and users it is attached to and sends `detach=true` only after you confirm. The built-in
`AdministratorAccess` policy cannot be changed or deleted (`403`).

In the Admin UI each policy card shows where it is attached and has **Edit** and **Delete**; **Edit** opens the
policy editor with the document loaded and says "Attached to: … — saving keeps these attachments". The built-in
policy is marked *Built-in · read-only*. `GET /admin/iam/users` lists each user's directly attached policies in
`policyNames`.

### Certification without admin rights

Certifying an ontology or semantic-layer definition is an **ownership** decision, not an editing
one, so it has its own action. That lets a domain owner sign off on their own subtree without
being made an administrator — and, equally, lets someone edit definitions without being able to
bless their own work:

```json
{
  "Version": "2024-01-01",
  "Statement": [{
    "Sid": "CertifySalesOntology",
    "Effect": "Allow",
    "Action": "ontology:Certify",
    "Resource": "ontology.sales.*"
  }]
}
```

With this attached, the user may call `POST /api/v1/object-types/{fqn}/certify` (and
`/decertify`) for anything under `ontology.sales`, and the recorded `certifiedBy` is **their** id.
Without it the call is refused with `403` even if they can otherwise read the object. Every
certify / decertify / refusal is written to the [Audit Log](audit-log.md).

## Column-Level Security

Ontul supports two distinct column-level controls. Both apply server-side inside the query plan; clients cannot bypass them.

| Mechanism | Effect | Output schema | Use when |
| --- | --- | --- | --- |
| **Column Deny** | Column is dropped from the SCAN output. | Column disappears from result. | The column should be inaccessible — analytics on the field aren't allowed at all. |
| **Column Mask** | Column value is replaced by a SQL expression. | Column name and type preserved. | The column structure stays useful (joins, group-by) but the raw value must not reach the caller — phone numbers, SSNs, salaries, emails. |

Precedence on conflict: **Deny &gt; Mask &gt; Allow**. A column listed in both a Deny and a Mask statement disappears entirely; the mask never fires.

### Column Deny

Denied columns vanish from the SCAN output via an injected `PROJECT` node. A `SELECT *` from a user with `Columns: ["ssn"]` in a Deny statement gets every column except `ssn`. An explicit `SELECT ssn FROM ...` planned by Calcite earlier in the pipeline will fail at validation; the deny mechanism is designed for the `SELECT *` and "show me everything" patterns that BI tools emit.

```json
{
  "Sid": "HideSsnFromAnalysts",
  "Effect": "Deny",
  "Action": "data:Select",
  "Resource": "data:table:hr.core.employees",
  "Columns": ["ssn"]
}
```

### Column Masking

Mask statements replace each named column's value with a SQL expression evaluated at the worker. The output column keeps its original name and Arrow type — downstream operations (`WHERE`, joins, aggregations) work, just on the masked values.

```json
{
  "Sid": "MaskHrPii",
  "Effect": "Mask",
  "Action": "data:Select",
  "Resource": "data:table:hr.core.employees",
  "MaskedColumns": {
    "phone":  "'***-***-XXXX'",
    "email":  "MD5(email)",
    "salary": "ROUND(salary, -3)"
  }
}
```

Examples of what each pattern yields:

| Mask expression | Behavior | Example output |
| --- | --- | --- |
| `'REDACTED'` | Literal — every row gets the constant. Cheapest, no leakage. | `REDACTED` |
| `MD5(email)` | Stable hash — same input → same output, so joins and grouping still discriminate users. | `5d41402abc4b2a76…` |
| `ROUND(salary, -3)` | Numeric bucketing — preserves order-of-magnitude utility for analytics while hiding exact value. | `75000` |
| `'XXX-XXX-' \|\| RIGHT(phone, 4)` | Partial reveal — last 4 digits visible. <span style="color:#a06000">Requires worker support for the function — see "Limitations" below.</span> | `XXX-XXX-1234` |
| `CASE WHEN '${user.attr.role}' = 'compliance' THEN ssn ELSE '***-**-XXXX' END` | Conditional unmask — compliance users see raw, others see masked. | `***-**-XXXX` (most users), real value (compliance) |

#### How a mask query looks

A user in `analyst-group` running:

```sql
SELECT name, phone, salary FROM hr.core.employees WHERE department = 'Engineering';
```

receives the rewritten plan (conceptually):

```sql
SELECT name,
       '***-***-XXXX'      AS phone,
       ROUND(salary, -3)   AS salary
  FROM hr.core.employees
 WHERE department = 'Engineering';
```

Same query as an admin (no policies apply) returns the raw `name`, `phone`, and `salary`.

#### User-context templating

The mask expression supports the same templating tokens as the [semantic layer's mandatory filters](semantic-layer.md#mandatory-filters-with-user-context-templating) — `${user.id}`, `${user.roles}`, `${user.attr.<key>}`. The substitution happens after policy load and before SQL parsing, so a single policy can implement role-aware unmasking without one statement per role:

```json
{
  "Sid": "ConditionalSsnMask",
  "Effect": "Mask",
  "Action": "data:Select",
  "Resource": "data:table:hr.core.employees",
  "MaskedColumns": {
    "ssn": "CASE WHEN ${user.id} = 'compliance_admin' THEN ssn ELSE '***-**-' || RIGHT(ssn, 4) END"
  }
}
```

`${user.id}` substitutes to a single-quoted SQL literal — `'compliance_admin'` — so write it bare, **without** wrapping it in your own quotes. The same convention applies to `${user.attr.<key>}`: bare in the expression, the resolver adds the quotes. Missing user attributes resolve to the empty string `''`. The substitution escapes embedded `'` in values — basic SQL-injection defence for a feature whose inputs come from admin DDL, not user prompts.

#### Precedence between Mask statements

When multiple Mask statements target the same column (e.g. two groups both attach masks to `ssn`), the **lexicographically smallest `Sid` wins** — deterministic across cluster restarts and snapshot replays. Attach order, group join order, and policy creation time are intentionally not part of the precedence calculation.

#### Validation

Mask expressions are parse-checked at policy-attach time. `POST /admin/iam/policies` with a syntactically broken expression returns `400 Bad Request` with the parser's message:

```bash
$ curl -X POST $ADMIN_URL/admin/iam/policies -d \
'{
  "name": "Bad",
  "document": {
    "Version": "2024-01-01",
    "Statement": [
      {
        "Sid": "BadMask",
        "Effect": "Mask",
        "Action": "data:Select",
        "Resource": "data:table:x",
        "MaskedColumns": { "c": "SUBSTR(" }
      }
    ]
  }
}'
{"error":"Mask expression for column 'c' fails to parse: SUBSTR("}
```

At query time, the master plans each mask through Calcite against the actual table schema, so a syntactically-valid expression that references a non-existent column or uses an unsupported function fails at the per-table compile step. The failure is logged and the column falls through unmasked — never silently aborts the whole query — but the master log surfaces it:

```
WARN  QueryService - [<queryId>] IAM mask compile failed for
      data:table:hr.core.employees.salary — leaving raw:
      No match found for function signature ROUND(<NUMERIC>, <NUMERIC>)
```

Watch for that line after attaching a new mask policy; it's the only signal that an expression looks valid SQL but the planner doesn't understand it.

#### Limitations

- **Worker expression-engine coverage is partial**. The mask expression is compiled to Calcite's positional `$N` form on the master, then evaluated by the worker's expression engine. Reliable functions include numeric (`ROUND`, `+`, `-`, `*`, `/`, comparisons), string concat (`||`, `CONCAT`), `UPPER`, `LOWER`, `MD5`, `CASE WHEN`, and literal substitutions. Less reliable: 3-argument `SUBSTRING` with named offsets, `SUBSTR` with negative indices, advanced regex. When in doubt, keep the mask simple (literal replacement or a single function call) and verify by running a query as a masked user.
- **Semantic-view columns**. Masks attach to the underlying physical table, not the semantic view that wraps it. If you mask `hr.core.employees.salary`, a query that goes through `hr.curated.employee_summary` (a semantic view) still sees the masked value because the SCAN node it reads from has the mask applied.
- **`SELECT *` is the common path**. Mixed-cardinality writes (a user explicitly selecting a denied column) are still planned by Calcite before IAM runs, so they may fail at validation rather than silently dropping the column.

## Row-Level Security

Apply WHERE-condition filters to restrict which rows a user can see. The condition is AND'd into the query's existing WHERE inside an injected FILTER node, evaluated at the SCAN layer — before any user predicate, before any mask.

```json
{
  "Sid": "ApacOnly",
  "Effect": "Allow",
  "Action": "data:Select",
  "Resource": "data:table:ice.sales.orders",
  "Condition": "region = 'APAC'"
}
```

`Condition` supports the `${user.userId}` substitution for per-user scoping. The semantic layer's mandatory filters use a richer token set (`${user.id}`, `${user.roles}`, `${user.attr.<key>}`) and are the recommended path for new policies — see [Semantic Layer → Mandatory filters](semantic-layer.md#mandatory-filters-with-user-context-templating).

```json
{
  "Sid": "OwnRowsOnly",
  "Effect": "Allow",
  "Action": "data:Select",
  "Resource": "data:table:ice.app.user_events",
  "Condition": "user_id = '${user.userId}'"
}
```

Multiple matching Allow statements with conditions are joined with `AND` — every applicable row filter applies. The combined predicate evaluates against the raw column values, before any column masks, so row filters on PII columns (e.g. "tenant_id = X") work even when those columns are masked in the projection.

## UDF Permissions

User-defined functions are first-class IAM resources. Five action verbs control the UDF lifecycle:

| Action | Required for |
| --- | --- |
| `UDF:EXECUTE` | Calling a UDF in a query (planner enforces this against every UDF the query references) |
| `UDF:CREATE` | Registering a TEMPORARY or USER-scoped UDF |
| `UDF:DROP` | Dropping a USER-scoped UDF |
| `UDF:CREATE_GLOBAL` | Registering a GLOBAL UDF visible to all users |
| `UDF:DROP_GLOBAL` | Dropping a GLOBAL UDF |

UDF resources are addressed as `udf:<name>`, so policies can target an exact function or a wildcard family:

```json
{
  "Effect": "Allow",
  "Action": "UDF:EXECUTE",
  "Resource": ["udf:mask_*", "udf:hash_*", "udf:geohash"]
}
```

The Admin UI policy editor ships with templates *UDF Execute Any*, *UDF Author*, *UDF Sandbox*, *UDF Deny Sensitive*, *UDF Global Admin*, and *UDF Global Read-Only*. See the [UDF feature page](udf.md) and the [UDF tutorial](../use-cases/udf-tutorial.md) for the full lifecycle.

## Decision Endpoint Permissions

[`ai_verdict()` / `ai_verdict_one()`](ai-verdict.md) call an external decision
endpoint once per distinct state in every batch. That makes them the one SQL
functions that spend money per row and send column text to a third party, so
reading the table is not sufficient authorization — the endpoint is a resource
of its own.

| Action | Resource | Required for |
| --- | --- | --- |
| `AI:DECIDE` | `connection:<id>` | Evaluating `ai_verdict()` or `ai_verdict_one()` against that `DECIDE` connection |

```json
{
  "Version": "2024-01-01",
  "Statement": [
    {
      "Sid": "CheapModelOnly",
      "Effect": "Allow",
      "Action": "AI:DECIDE",
      "Resource": "connection:jev-small"
    },
    {
      "Sid": "NoVendorX",
      "Effect": "Deny",
      "Action": "AI:DECIDE",
      "Resource": "connection:vendor-x"
    }
  ]
}
```

Two controls follow from granting per endpoint rather than per function: an
allow list for who may bill the expensive model, and a deny that expresses
"nobody sends data to this vendor from SQL" without touching a single table
grant. As everywhere else here, no matching statement is a denial — the check
runs on the master, before any plan reaches a worker, and the error names both
the action and the endpoint it refused.

The Admin UI ships both shapes as policy templates — **Decision Endpoint
(ai_verdict)** and **Decision Endpoint (Allow One, Deny Another)** — see
[IAM Policy Templates](iam-policy-templates.md#decision-endpoints-ai_verdict).

A `DECIDE` connection marked `external=true` additionally requires each call to
pass `acknowledgeEgress`. That is not an IAM check and cannot be granted away:
the state is an arbitrary SQL expression with no link to an object type's `pii`
flags, so there is nothing for the engine to infer, and the acknowledgement
records a human decision instead of pretending to a check. See
[`ai_verdict`](ai-verdict.md#governance).

## Semantic Layer & Retriever RBAC

The [semantic layer](semantic-layer.md) and [retrievers](retrievers.md) gate access through the same IAM system, on two axes:

1. **Discovery / visibility** — the metadata-only `data:SelectTable` action on the object's fqn governs whether a caller can *list* or *describe* a semantic view or retriever (`ontul_list_semantic_views`, `ontul_list_retrievers`, and the REST `GET` routes are IAM-filtered with it).
2. **Object-level role gating** — a metric's `allowedRoles[]` and a retriever's `allowedRoles[]` name **IAM groups**. Empty means public to any caller that passes axis 1; non-empty means the caller must additionally be a member of at least one listed group. A metric denial happens *at rewrite time*, before the formula is parsed, so the expression never leaks to an unauthorized user; a retriever denial happens before the template is rendered or pushed down.

### Configure a metric with role gating (Admin UI)

In the Admin UI, **Semantic & AI → Semantic Layer → Register View**:

1. Fill `catalog` / `schema` / `name` and the base SQL.
2. Add a metric (e.g. `revenue`) and, in its **allowedRoles** field, list the IAM group(s) allowed to use it — e.g. `finance, analyst`. Leave it empty for a public metric.
3. Optionally set per-metric **mandatoryFilters** (row scoping that applies only when the metric is referenced) and view-level mandatory filters with `${user.attr.X}` templating.
4. Save, then **Certify** (the badge button) to flip `DRAFT → CERTIFIED`.

The equivalent REST call (what the UI issues under the hood — `POST /api/v1/semantic-views`):

```bash
curl -s -X POST "http://<master>:8080/api/v1/semantic-views" \
  -H "Authorization: Token $TOKEN" -H "Content-Type: application/json" -d '{
  "catalog": "tpch", "schema": "sales", "name": "region_sales",
  "baseSql": "SELECT region, amount, discount, status FROM tpch.sales.orders",
  "metrics": [
    {"name": "revenue", "expr": "SUM(amount * (1 - discount))", "synonyms": ["매출"]},
    {"name": "vip_revenue", "expr": "SUM(amount) FILTER (WHERE tier=''VIP'')",
     "allowedRoles": ["finance"], "mandatoryFilters": ["status=''COMPLETED''"]}
  ],
  "dimensions": [{"name": "region"}]
}'
```

A retriever's `allowedRoles` is configured the same way under **Semantic & AI → Retrievers → Register Retriever** (or `POST /api/v1/retrievers`).

### Set up the IAM group that backs `allowedRoles`

`allowedRoles: ["finance"]` only gates access if the `finance` group exists and the caller is a member. Create the group and add the user (Admin UI **IAM** page, or REST — see [Users and Groups](#users-and-groups)):

```bash
# Create the group named in allowedRoles
curl -s -X POST "http://<master>:8080/admin/iam/groups" \
  -H "Authorization: Bearer $JWT" -H "Content-Type: application/json" \
  -d '{"groupName": "finance"}'

# Add the user to it — now `alice` satisfies the metric's allowedRoles
curl -s -X POST "http://<master>:8080/admin/iam/add-user-to-group" \
  -H "Authorization: Bearer $JWT" -H "Content-Type: application/json" \
  -d '{"username": "alice", "groupName": "finance"}'
```

Result: `alice` (in `finance`) can resolve `vip_revenue`; a user in only `viewer` is denied before the `tier='VIP'` formula is parsed — and the same `finance` membership lets `alice` invoke a retriever whose `allowedRoles` includes `finance`. One group membership governs both the analytics metric and the multi-modal retriever, consistently across SQL, BI, and MCP.

!!! note "System-internal queries bypass RBAC"
    The planner's anonymous context (no authenticated user) skips metric RBAC — the intentional escape hatch for internal admin work. Caller-issued queries and retriever invocations always carry the IAM identity and are always gated.

## End-to-end example: tenant-scoped masking + RLS

A complete policy that combines RBAC, masking, and row-level security for a multi-tenant SaaS application:

```json
{
  "Version": "2024-01-01",
  "Statement": [
    {
      "Sid": "BaselineRead",
      "Effect": "Allow",
      "Action": "data:Select",
      "Resource": "data:table:saas.core.*",
      "Condition": "tenant_id = '${user.userId}'"
    },
    {
      "Sid": "MaskCustomerPii",
      "Effect": "Mask",
      "Action": "data:Select",
      "Resource": "data:table:saas.core.customers",
      "MaskedColumns": {
        "email":  "MD5(email)",
        "phone":  "'***-***-XXXX'",
        "ssn":    "'***-**-XXXX'"
      }
    },
    {
      "Sid": "HideInternalCols",
      "Effect": "Deny",
      "Action": "data:Select",
      "Resource": "data:table:saas.core.*",
      "Columns": ["internal_notes", "fraud_score"]
    }
  ]
}
```

For a user `tenant_acme` running `SELECT * FROM saas.core.customers`:

1. The baseline Allow grants read access — gated by `tenant_id = 'tenant_acme'`.
2. The Deny strips `internal_notes` and `fraud_score` from the SCAN output.
3. The Mask replaces `email`, `phone`, `ssn` with their masked expressions.
4. The result: every column except the two denied ones, with PII fields obfuscated, restricted to rows where `tenant_id = 'tenant_acme'`.

A second tenant `tenant_globex` running the identical query sees only their rows, with the same masking applied. An admin running it sees everything raw — no policies attach to the default admin's effective policy set unless they're explicitly granted.

## NeorunBase serving policies (federation)

When **NeorunBase** is used alongside ontul as a low-latency serving layer over the lakehouse, external
applications often connect **directly to NeorunBase** (PostgreSQL wire / JDBC), bypassing ontul. To keep ontul
the single place where access is authored, NeorunBase runs in *federated* mode and **pulls** its IAM from ontul
— ontul itself needs no changes.

You author these policies in ontul exactly like any other, with two conventions so NeorunBase recognizes and
enforces them:

- **Resource** uses NeorunBase's native format: `db:table:<catalog>.<ns>.<table>` for an Iceberg catalog table,
  or `db:table:<schema>.<table>` for a native table. (ontul's own policies use `data:table:…`; the `db:table:`
  prefix is what marks a policy as NeorunBase-targeted, so the two never collide.)
- **Action** uses `pg:Select` (or `SELECT`).

`Allow`/`Deny`, column `Columns`, `Condition` (row filters), and `MaskedColumns` all work exactly as documented
above — NeorunBase imports the policy document verbatim and enforces masking, column deny, and row filtering on
its serving path.

!!! tip "Policy editor template"
    The Admin UI policy editor includes a **"NeorunBase serving policy"** template that pre-fills the
    `db:table:` / `pg:*` format and explains when to use it, so you don't have to remember the conventions.

The external client then authenticates to NeorunBase with an ontul-issued token (single sign-on); NeorunBase
validates it locally with the shared master key and applies the synced policies. See the NeorunBase docs,
*IAM Federation*, for the NeorunBase-side configuration.

## Management

Users, groups, and policies are managed through the Admin UI (with a visual policy editor) or the REST API. The Admin UI's IAM page surfaces `requirePasswordChange`, attached policies, and mask/deny columns per resource so operators can audit a user's effective access without writing a query.
