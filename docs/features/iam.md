# Identity and Access Management

Ontul includes a built-in IAM system that provides authentication, authorization, and fine-grained access control down to the column and row level. Column masking, row-level filters, and metric-level RBAC (when the semantic layer is in use) all read from the same policy store, so a single statement governs what a user sees regardless of whether the query arrives over Arrow Flight SQL, REST, or the MCP server.

## Authentication

Ontul supports four authentication methods that can co-exist on the same cluster:

| Method | When to use | Header / field |
| --- | --- | --- |
| **Username / Password** | Interactive UI / SQL clients (Tableau, DBeaver) that only know Basic auth. PBKDF2-hashed passwords. | `authorization: Basic base64(user:pass)` |
| **JWT Bearer Token** | Short-lived sessions (default 15 min). Issued by `POST /admin/auth/login`. Used by the Admin UI and Java SDK. | `authorization: Bearer <jwt>` |
| **Access Keys** (`AKIA…`) | Long-lived service-account credentials with separate `accessKeyId` + `secretAccessKey`. Issued via `POST /admin/iam/keys`. | `authorization: AccessKey AKIA…:<secret>` |
| **STS Temporary Credentials** (`ASIA…`) | Short-lived AKIA-style credentials with configurable expiry — used by transient workloads (CI runners, ETL jobs). | Same as Access Keys, prefix differs. |

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
| Create user | `POST /admin/iam/users` `{username, password}` |
| List users | `GET  /admin/iam/users` |
| Delete user | `DELETE /admin/iam/users/{username}` |
| Add to group | `POST /admin/iam/add-user-to-group` `{username, groupName}` |
| Create group | `POST /admin/iam/groups` `{groupName}` |
| Issue access key | `POST /admin/iam/keys` `{username, expiresAt?}` |

!!! note "Field-name aliases"
    `POST /admin/auth/login` accepts the user identifier as either `username` or `user`, and `POST /admin/iam/keys` accepts either `username` or `userId`. The canonical field is `username` in both cases; the aliases exist so clients and SDKs that use a different field name interoperate without a mapping layer.

## Policy-Based Access Control

Ontul uses AWS-style JSON policies to manage permissions. Actions live in the `data:` namespace (table reads, DML, admin verbs) and `UDF:` namespace; table resources are addressed as `data:table:<catalog>.<schema>.<table>` and accept wildcards.

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

- **Action verbs in `data:` namespace** — runtime SQL: `data:Select`, `data:Insert`, `data:Update`, `data:Delete`, `data:Merge`. Catalog DDL: `data:CreateTable`, `data:DropTable`, `data:AlterTable`. Operational: `data:KillJob`, `data:CancelQuery`. The metadata-only `data:SelectTable` verb gates discovery surfaces (e.g. semantic-view listing, table catalog browse) — distinct from the runtime `data:Select` that gates actual scans.
- **UDF actions** stay in the `UDF:` namespace — see [UDF Permissions](#udf-permissions).
- **Resources**: `data:table:<catalog>.<schema>.<table>`, `data:schema:<catalog>.<schema>`, `data:job:*`, `data:query:*`, `udf:<name>`. Wildcards `*` and `?` are supported.
- **Attachment**: policies attach to users (direct) or groups (via group membership).
- **Deny rules take precedence** over allow rules.
- **Deny-by-default**: no matching policies means access is denied.

### Creating and updating policies

The same endpoint handles both — `POST /admin/iam/policies` is upsert. Re-posting with an existing `name` overwrites the document and propagates the new content to follower masters via `pushUpdate()`. The next query the affected user issues sees the new policy; no cache invalidation is needed because effective policies are computed per-query.

```bash
# Initial creation
curl -X POST $ADMIN_URL/admin/iam/policies -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d '{"name":"AnalystAccess","document": { … v1 … }}'

# Update — same `name`, new document
curl -X POST $ADMIN_URL/admin/iam/policies -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d '{"name":"AnalystAccess","document": { … v2 … }}'
```

Both calls return `201 Created`. Distinguish create-vs-update on the client side by `GET /admin/iam/policies` first if needed; the server doesn't differentiate.

| Operation | Endpoint |
| --- | --- |
| Create / update policy | `POST   /admin/iam/policies` |
| List policies | `GET    /admin/iam/policies` |
| Delete policy | `DELETE /admin/iam/policies/{name}` |
| Attach policy to group | `POST   /admin/iam/attach-group-policy` `{groupName, policyName}` |
| Detach policy from group | `POST   /admin/iam/detach-group-policy` `{groupName, policyName}` |

The `AdministratorAccess` policy is reserved — `deletePolicy` rejects attempts to remove it.

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
$ curl -X POST $ADMIN_URL/admin/iam/policies -d '{"name":"Bad","document":{
    "Version":"2024-01-01","Statement":[{
      "Sid":"BadMask","Effect":"Mask","Action":"data:Select",
      "Resource":"data:table:x",
      "MaskedColumns":{"c":"SUBSTR("}
    }]}}'
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
