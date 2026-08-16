# Ontology (Objects, Links & Actions)

The **ontology** is Ontul's typed semantic model of your business — a layer of
**object types**, **link types**, and **action types** that sit *above* the
physical tables, catalogs, and graphs the engine reads and writes. Where a
[semantic view](semantic-layer.md) curates *analytics* (metrics + dimensions)
and a [retriever](retrievers.md) curates *retrieval* (vector / graph / full-text),
the ontology curates the *entities and operations* an application or agent works
with: a `Customer`, an `Order`, the `places` relationship between them, and the
governed `approve_invoice` write.

The design goal is a credible, open ontology + Action layer: reads win
structurally (open Iceberg as the source of truth, single-engine graph RAG via
NeorunBase, a semantic layer, IAM, and an MCP surface), and the ontology adds the
typed object/link model plus **governed write-back Actions** on top.

```text
Agent (MCP) / App / Admin UI
        │
        ▼
   ┌──────────────────────── Ontology ────────────────────────┐
   │  ObjectType   ──places──►  ObjectType        (type graph) │
   │   (Customer)               (Order)                        │
   │      │  read: query / traverse    │  write: invoke action │
   └──────┼─────────────────────────────┼─────────────────────┘
          ▼                             ▼
   read runtime                   write runtime
   ├─ ObjectSet query   → engine  ├─ DML       → SoT (Iceberg / JDBC)
   └─ Link traversal              └─ OPERATION → REST operational system
      ├─ JOIN  → engine SQL          (ERP / CRM / payment / internal service …)
      └─ GRAPH → NeorunBase GRAPH_NEIGHBORS
```

Two principles run through the whole layer:

- **Iceberg is the system of record; NeorunBase is derived, rebuildable serving.**
  Reads may come from the derived serving layer; **writes go to the source of
  record (Iceberg's ACID / write-audit-publish) or to an external operational
  system over REST — an ERP, CRM, payment gateway, ticketing or internal service —
  never to the derived layer**, which is then rebuilt from the SoT. Writing critical
  action-writes straight into a lakehouse table, or via raw JDBC into an operational
  database, is an anti-pattern; the real write goes to the system of record's own
  API. (ERP is just one example of such a system, not the framing.)
- **Two graphs.** The ontology *type graph* (object/link *definitions*) is small
  metadata in the cluster store. The *instance graph* (actual edges between
  millions of rows) lives in NeorunBase; a `GRAPH` link binds a link type to it
  for traversal.

All three primitives are persisted in the cluster metadata store (key prefixes
`objecttype:`, `linktype:`, `actiontype:`) and replicate to follower masters with
the rest of the cluster metadata, exactly like semantic views and retrievers.

---

## Object types

An **object type** maps a business entity onto a physical read source, naming its
properties in business terms and declaring where writes to it should land.

| Field | Purpose |
| --- | --- |
| `catalog`, `schema`, `name` | The object type's identity; its FQN is `catalog.schema.Name`. |
| `readSource` | The physical table (a `catalog.schema.table` registered in Ontul) instances are read from. |
| `primaryKey[]` | One or more **property names** (not physical columns) that identify an instance. |
| `properties[]` | Each property: `name` (logical), `type` (`string`/`long`/`double`/`boolean`/…), `column` (the physical column it maps to), plus optional `synonyms` (NL/agent matching) and `pii`. |
| `writeTarget` | Where a write action lands: `DML` (a table — `catalog.schema.table`) or `OPERATION` (a connector operation). |
| `allowedRoles[]`, `tags[]`, `status` | Governance + lifecycle metadata. Read `effectiveStatus` (below) rather than `status` for trust. |

The property → column mapping is the crux: callers, agents, and SDKs reference
**logical property names**, and Ontul resolves them to physical columns when it
generates SQL. A `primaryKey` entry must name a declared property — registering a
primary key that references a physical column name (e.g. `o_orderkey` instead of
the property `orderkey`) is rejected.

```json
{
  "catalog": "lake", "schema": "sales", "name": "Order",
  "readSource": "lake.sales.orders",
  "primaryKey": ["orderkey"],
  "properties": [
    { "name": "orderkey", "type": "long",   "column": "o_orderkey" },
    { "name": "status",   "type": "string", "column": "o_orderstatus", "synonyms": ["상태"] },
    { "name": "custkey",  "type": "long",   "column": "o_custkey" }
  ],
  "writeTarget": { "mode": "DML", "catalog": "lake", "schema": "sales", "table": "orders" }
}
```

---

## Link types

A **link type** is a typed relationship between two object types — the edges of
the ontology's type graph. It declares a **binding** that says how the
relationship resolves at query time.

| Field | Purpose |
| --- | --- |
| `fromObjectType`, `toObjectType` | The endpoint object-type FQNs. |
| `cardinality` | `ONE_TO_ONE` / `ONE_TO_MANY` / `MANY_TO_ONE` / `MANY_TO_MANY`. |
| `binding.mode` | `JOIN` (relational key equality) or `GRAPH` (a native NeorunBase edge). |
| `binding.fromKey`, `binding.toKey` | *(JOIN)* the property names on each side whose values must match. |
| `binding.graphCatalog`, `binding.edgeTable`, `binding.edgeLabel`, `binding.direction` | *(GRAPH)* the NeorunBase catalog, the edge/relations table, the edge label, and traversal direction. |

- A **JOIN** binding resolves through the query engine: `Customer places Order`
  where `Customer.custkey = Order.custkey`.
- A **GRAPH** binding is pushed down to NeorunBase's graph engine — the same
  instance graph a [retriever](retrievers.md) reaches — so traversal *is* the
  graph engine, not an application-side loop.

```json
{
  "catalog": "lake", "schema": "sales", "name": "places",
  "fromObjectType": "lake.sales.Customer",
  "toObjectType": "lake.sales.Order",
  "cardinality": "ONE_TO_MANY",
  "binding": { "mode": "JOIN", "fromKey": "custkey", "toKey": "custkey" }
}
```

---

## Reading the ontology

The read runtime is the resolver that turns an ontology intent — expressed
against object/link definitions — into injection-safe SQL the existing engines
execute, and returns typed rows. It is the read counterpart to Actions (write).

### ObjectSet query

`POST /api/v1/object-types/{fqn}/query` returns instances of one object type,
filtered and projected **by property name**. The body accepts `filters`
(property → value equality predicates, ANDed), `select` (property names to
project; omit for all), `orderBy`, `desc`, and `limit`.

```bash
curl -X POST "$ADMIN/api/v1/object-types/lake.sales.Order/query" \
  -H "Authorization: Token $TOKEN" -H "Content-Type: application/json" \
  -d '{ "filters": { "status": "OPEN" }, "select": ["orderkey","status"], "limit": 50 }'
```

Ontul resolves each property to its column, builds
`SELECT "o_orderkey" AS "orderkey", … FROM lake.sales.orders WHERE "o_orderstatus" = 'OPEN' LIMIT 50`,
executes it through the query engine, and returns `{ columns, rows, rowCount }`
where the columns are the **logical property names**. Referencing an undeclared
property is rejected with `400` before any SQL runs.

### Link traversal

`POST /api/v1/link-types/{fqn}/traverse` returns the related to-object instances
reachable from one source object. The body takes `sourceKey` (the source
object's primary-key value), `select`, `limit`, and — for GRAPH links — an
optional `maxDepth`.

- **JOIN** links resolve relationally. When the link's `fromKey` *is* the
  from-object's primary key (the common case), traversal collapses to a single
  filter on the to-object (`toKey = sourceKey`) — no join needed; otherwise a
  relational join resolves the key.
- **GRAPH** links are pushed down to NeorunBase's
  `GRAPH_NEIGHBORS(edge_table, seed, max_depth, edge_filter)` table-valued
  function; the returned neighbour ids are joined back to the to-object's table so
  you get typed instances, not bare ids.

```bash
# Walk the graph: relations of entity 1, along 'is_a' edges, up to depth 2.
curl -X POST "$ADMIN/api/v1/link-types/nb.public.is_a/traverse" \
  -H "Authorization: Token $TOKEN" -H "Content-Type: application/json" \
  -d '{ "sourceKey": "1", "maxDepth": 2, "select": ["id","name"], "limit": 50 }'
```

---

## Actions (governed write-back)

An **action type** is a typed, parameterized, **governed** write operation over
an object type — how an application or agent *changes* data, in contrast to the
read runtime above. An agent invokes a named action (`approve_invoice`) instead
of composing raw SQL; the platform validates parameters, authorizes the caller,
enforces idempotency, executes against the write system-of-record, and audits the
run.

| Field | Purpose |
| --- | --- |
| `objectType` | The object type this action operates on (its write context). |
| `mode` | `DML` (execute rendered SQL against the SoT) or `OPERATION` (call a connector operation). |
| `parameters[]` | Each: `name`, `type` (`STRING`/`LONG`/`DOUBLE`/`BOOLEAN`/`IDENT`), `required`, `description`. |
| `sqlTemplate` | *(DML)* parameterized SQL with `${param}` placeholders. **Admin-authored and trusted.** |
| `operationCatalog`, `operation` | *(OPERATION)* the operation-surface catalog + operation id to invoke. |
| `allowedRoles[]`, `requiresApproval` | Governance — role gating + an optional human-in-the-loop gate. |
| `tags[]`, `status`, `owner` | Lifecycle metadata. |

### Invoking an action

An action can be invoked two ways — REST and SQL — and **both run the same code**
(`ActionInvoker`). That is deliberate: an action writes to a system of record, so a second
entry point must not become a weaker path around the governance. The only difference recorded
is `via=REST` / `via=SQL` in the audit entry.

```bash
# REST
curl -X POST "$ADMIN/api/v1/action-types/lake.sales.approve_invoice/invoke" \
  -H "Authorization: Token $TOKEN" -H "Content-Type: application/json" \
  -d '{ "args": { "id": 42, "reason": "reviewed" }, "idempotencyKey": "req-42" }'
```

```sql
-- SQL (JDBC / Arrow Flight SQL / MCP) — see "Calling an action from SQL" below
CALL lake.sales.approve_invoice(id => 42, reason => 'reviewed', idempotency_key => 'req-42');
```

Either way the invoke is **fail-closed and governed**. In order:

1. **Authorization** — the caller must be an administrator *or* hold an explicit
   `ontology:InvokeAction` grant on the action's FQN. Denied by default. An `OPERATION`-mode
   action additionally requires `ontology:InvokeOperation` on
   `operation:<operationCatalog>:<operation>` — writing into an external system is a different
   power from reading the warehouse (`data:table:…`), so it is granted separately. That second
   check is report-only until `ontul.iam.operation.permission.enforced=true`; while it is off, a
   missing grant is audited as `action:invoke:grant-missing` naming the exact resource, so the
   grants can be added before enforcement is switched on.
2. **Idempotency** — an optional `idempotencyKey` is checked against a ledger
   (`actionrun:`); a retried key returns the prior result without re-applying the
   write.
3. **Parameter validation** — every required parameter must be present, else `400`.
4. **Approval gate** — if `requiresApproval` is set, the invoke is *staged*
   (returns `202 pending_approval`) rather than applied.
5. **Execution**:
     - **DML** — the template is rendered into injection-safe SQL and executed
       through the query engine against the source of record; the result is
       audited and the idempotency outcome recorded.
     - **OPERATION** — the action's `operationCatalog` is resolved to a connector
       exposing an operation surface (see below) and the named operation is
       invoked; the result is audited and recorded identically.

### Calling an action from SQL

Ontul's catalog concept comes from Trino, where a catalog is the namespace of **everything a
connector exposes** — procedures included, not only tables. An action is reachable as a
procedure:

```sql
CALL erp.default.post_invoice(order_id => '1001', amount => 250.00);

-- retry-safe: the same key returns the prior outcome instead of posting twice
CALL erp.default.post_invoice(order_id => '1001', idempotency_key => 'req-42');
```

- **Named arguments only.** Positional arguments are rejected: an action writes to a system of
  record, and a mis-ordered positional argument is exactly the mistake that is expensive to
  discover afterwards.
- **`idempotency_key` is reserved**, not an action parameter. It carries the same de-duplication
  token the REST body passes and shares the same ledger, so retrying a statement — or switching
  from REST to SQL — never re-applies the write.
- Returns one row: `action | status | detail`, where `detail` is the JSON document the REST
  endpoint returns. `status` is `OK`, `REPLAYED` (idempotent retry), or `PENDING_APPROVAL` —
  the last one means the call was *staged for a human*, not applied.
- Approval gates, validation, audit and lineage behave exactly as on the REST path.

Discover what can be called:

```sql
SELECT routine_catalog, routine_schema, routine_name, mode, target, parameters,
       requires_approval, status
FROM information_schema.routines;
```

| Column | Meaning |
| --- | --- |
| `ROUTINE_CATALOG` / `_SCHEMA` / `_NAME` | the action's FQN, i.e. what `CALL` addresses |
| `ROUTINE_TYPE` | always `PROCEDURE` |
| `MODE` | `DML` (writes the warehouse) or `OPERATION` (writes an external system) |
| `TARGET` | for `OPERATION`: `<operationCatalog>:<operation>` |
| `PARAMETERS` | declared parameters with type and nullability |
| `REQUIRES_APPROVAL` | `YES` → a `CALL` returns `PENDING_APPROVAL`, it does not apply |
| `STATUS` | lifecycle status of the action definition |

The same rows back JDBC `DatabaseMetaData.getProcedures()`, so a BI tool or an agent finds
actions without a side channel. This is also why an operation catalog shows no tables: its
surface is procedures, and that surface is listed here.

### Injection safety

The same explicit model as retrievers applies: **the template is admin-authored
and trusted; the arguments are caller-supplied and untrusted.** Each parameter's
declared `type` governs how its value is rendered — `STRING` becomes a
single-quote-escaped literal (`'…''…'`), `LONG`/`DOUBLE`/`BOOLEAN` are validated
and inlined bare, and `IDENT` is restricted to `[A-Za-z0-9_.]`. A caller cannot
smuggle SQL through an argument, and an argument referencing an undeclared
parameter is dropped.

---

## Action workflows (Saga / DAG)

A single write is one action. A real operation is often **several** — reserve
stock, charge payment, create a shipment — that must run **as one governed unit**
and, if a later step fails, **undo** what already ran. An **action workflow** is
that unit: a **DAG of action types** executed server-side as a **Saga**.

Because the writes target external systems of record over REST (which cannot join
a distributed transaction and cannot be rolled back), "all-or-nothing" here means
**run to completion, or compensate what ran** — a best-effort *logical* inverse
call per completed step, not a database rollback. Orchestration, compensation,
idempotency, and audit are the platform's responsibility, so a caller — an agent,
an app, the admin UI — invokes **one** workflow and gets the whole guarantee.

### DAG, not just a chain

Each step names an already-registered action and its place in the graph:

| Step field | Purpose |
| --- | --- |
| `id` | Step id (referenced by other steps' `requires`). |
| `action` | FQN of a registered action type to run. |
| `args` | `param → workflow-input key` (or `=literal`) — builds the action's args from the workflow's `input`. |
| `requires` | Ids of steps that must complete first. Empty = a root. |
| `compensate` | *(optional)* FQN of a registered action that **undoes** this step on rollback. |
| `compensateArgs` | *(optional)* arg mapping for the compensate action; defaults to this step's own `args`. |

Steps run in **topological order**: a step runs once its `requires` are done, so
independent branches (a "diamond") both run and a join step waits for both. A
workflow where **no** step declares `requires` degenerates to declared (linear)
order — so a simple sequence is just the trivial DAG. Cycles and references to
unknown steps are **rejected at registration** (`400`).

### Authoring in YAML

Workflows are authored as **YAML** (the admin UI is a full-width YAML editor; the
same document round-trips via the REST API). A step's `compensate` is **optional**
— omit it and that step is simply skipped on rollback.

```yaml
catalog: erp
schema: ops
name: fulfill_order
title: Fulfill order
steps:
  - id: reserve
    action: erp.ops.reserve_stock
    args: { orderId: orderId }
    compensate: erp.ops.release_stock          # registered action; undoes reserve
  - id: charge
    action: erp.ops.charge_payment
    args: { orderId: orderId, amount: amount }
    requires: [reserve]
    compensate: erp.ops.refund_payment
    compensateArgs: { orderId: orderId }        # optional; else the step's own args
  - id: ship
    action: erp.ops.create_shipment
    args: { orderId: orderId }
    requires: [charge]                           # no compensate → nothing to undo
```

Register it with `POST /api/v1/action-workflows/yaml` (body = the YAML) and export
the editable YAML back with `GET /api/v1/action-workflows/{fqn}/yaml` (server-
managed/computed fields — `fqn`, `createdAt`, `owner`, … — are stripped so you see
only what you can edit). A JSON form of the same document works via
`POST /api/v1/action-workflows`.

### Invoke → run, or roll back

`POST /api/v1/action-workflows/{fqn}/invoke` runs the Saga:

1. **Authorization** — gated on `ontology:InvokeWorkflow` for the workflow FQN
   (admin-bypass, fail-closed). An optional `requiresApproval` stages it instead.
2. **Idempotency** — an optional `idempotencyKey` returns the prior run's result.
3. **Forward pass** — steps execute in topological order; each result records
   `ok`, and where it ran (`driver`).
4. **Failure → compensation** — the first hard failure (a step that isn't
   `continueOnError`) stops the forward pass and runs each **completed** step's
   `compensate` action in **reverse order**. Final status is `completed`,
   `completed_with_errors`, or `rolled_back`.

```bash
curl -X POST "$ADMIN/api/v1/action-workflows/erp.ops.fulfill_order/invoke" \
  -H "Authorization: Token $TOKEN" -H "Content-Type: application/json" \
  -d '{ "input": { "orderId": 42, "amount": 199.90 }, "idempotencyKey": "order-42" }'
# → { "status": "rolled_back", "steps": [...], "compensations": [ {"via":"erp.ops.refund_payment",...}, {"via":"erp.ops.release_stock",...} ] }
```

### Governance — every action stays governed

Compensation is a **workflow** concern, so it lives on the step (`compensate`), not
as a property of the action type. And crucially, a `compensate` must be a
**registered action** — never inline ad-hoc REST — because both the forward step
**and** the compensation run through the same governed path: each is IAM-checked on
**its own** `ontology:InvokeAction` (fail-closed, admin-bypass), parameter-
validated, and audited. RBAC is thereby delegated to each registered action: a
compensation runs only if the caller may invoke that action. There is no way to
smuggle an ungoverned call into a rollback.

### Execution — master coordinates, workers act

The master is the **logical driver**: it owns the DAG scheduling, compensation
order, idempotency ledger, IAM, and audit. The **actual** work — the (possibly
slow) outbound REST call of an `OPERATION` action — is **delegated to a worker**,
exactly the way batch/streaming jobs are dispatched (`TASK_ASSIGN`). The master
resolves the concrete request (base URL + auth from the operation catalog /
[Connection ID](connection-id.md)) and hands the worker a self-contained call to
perform; it falls back to running the call in-process only when no worker is ready.
So a long-running action doesn't tie up the master, and action execution scales
across the worker fleet.

Relevant timeouts are configurable:

```properties
# ontul.properties
ontul.action.exec.timeout.ms=600000       # master awaiting a worker's action result
ontul.action.http.timeout.seconds=60      # the action's outbound HTTP request
```

---

## The operation surface (REST connector)

Not every write is SQL DML. To let an Action write back to an external operational
system of record over REST, Ontul provides a generic **`rest-operation` connector**
— the substrate for `OPERATION`-mode actions. A catalog registered with this
connector exposes a set of named, parameterized HTTP operations. Any operational
system with an API fits: ERP, CRM, payment, ticketing, an internal service. An ERP
integration, for instance, is simply this connector plus an ERP **profile**
(default headers / conventions) and the ERP's operation set — the connector is
generic, and a "profile" adapts it to a given system.

Config keys for a `rest-operation` catalog:

| Key | Purpose |
| --- | --- |
| `baseUrl` | The external base URL (e.g. `https://erp.example.com/api`). |
| `authType` | `NONE` / `BASIC` / `BEARER` / `HEADER`; credentials via `user`/`password`, `token`, or `authHeaderName`/`authHeaderValue` (resolvable through a [Connection ID](connection-id.md)). |
| `profile` | `generic` / `sap-odata` / … — supplies default request headers (content type / accept conventions). |
| `operations` | A JSON array of operations, each `{ id, method, path, body, headers?, successCodes? }` with `${param}` placeholders. |

Rendering is injection-safe: path values are URL-encoded and body values are
JSON-escaped (the template author supplies the surrounding JSON structure and
quoting, the same contract as the SQL template). The connector is a write target,
not a readable source, so it exposes no tables.

```json
{
  "connector": "rest-operation",
  "baseUrl": "https://erp.example.com/api",
  "profile": "generic", "authType": "BEARER", "token": "…",
  "operations": "[{\"id\":\"post_invoice\",\"method\":\"POST\",\"path\":\"/invoices\",\"body\":\"{\\\"id\\\": ${id}, \\\"amount\\\": ${amount}}\",\"successCodes\":[200,201]}]"
}
```

---

## Agent access (MCP)

Every primitive is exposed on the [Ontul MCP server](../reference/mcp-server.md)
so an agent discovers and uses the ontology through one governed surface — the
read tools and the write tool are symmetric:

| Tool | Purpose |
| --- | --- |
| `ontul_list_object_types` / `ontul_describe_object_type` | Discover object types + their property contracts. |
| `ontul_query_object_type` | Read instances (ObjectSet query) by property filters. |
| `ontul_list_link_types` / `ontul_describe_link_type` | Discover relationships. |
| `ontul_traverse_link` | Walk a relationship from a source object to its related objects. |
| `ontul_list_action_types` / `ontul_describe_action_type` | Discover write actions + their parameter contracts. |
| `ontul_invoke_action_type` | Invoke a governed write-back (DML or OPERATION). |
| `ontul_list_action_workflows` / `ontul_describe_action_workflow` | Discover multi-action workflows + their step DAG. |
| `ontul_invoke_action_workflow` | Run a workflow as one governed Saga (with reverse compensation on failure). |

Because governance (IAM, idempotency, audit) is enforced by the master beneath
these tools, an agent gets the ontology's read/write power without a path around
its controls.

---

## Admin UI

The admin UI manages the whole ontology under dedicated pages — **Object Types**,
**Link Types**, **Action Types** (with a parameter editor and an invoke tester),
and **Action Workflows** (a full-width YAML editor with a diamond-DAG starter
template, plus a DAG visualization and an invoke panel that shows the run outcome
and any compensations). The **Catalogs** page gains a *REST* catalog type (base
URL, profile, auth, and a JSON operations editor) for the operation surface, and a
**Drivers** page manages uploaded JDBC driver JARs (one JAR per driver class;
uploading a newer version replaces the old one, streamed to every worker). Both
REST and NeorunBase catalogs can reference a registered [Connection ID](connection-id.md)
instead of inlining credentials.

---

## Row caps

An ObjectSet query or link traversal is bounded by each request's own `limit` and
by a hard cluster-wide ceiling — the effective cap is the smaller of the two:

```properties
# ontul.properties — hard cluster-wide cap on rows returned by an ontology read
# (ObjectSet query or link traversal). Applied on top of each request's own limit.
ontul.ontology.read.max.rows.ceiling=10000
```

Both the relational (JOIN / ObjectSet) path and the GRAPH pushdown honour this
ceiling.

---

## Certification (trust an agent can act on)

Every ontology definition carries a `status` of `DRAFT` / `CERTIFIED` / `DEPRECATED`. On its own
a status is only a label, so Ontul adds two things that make it mean something, and folds the
result into one read-only field: **`effectiveStatus`**.

| `effectiveStatus` | Meaning |
| --- | --- |
| `CERTIFIED` | Signed off, unchanged since, and everything it is built on is certified too. |
| `STALE` | Was certified — then the definition changed underneath the signature. |
| `DRAFT` | Never certified (or capped there by a dependency). |
| `DEPRECATED` | Retired. |

**Certifying** is its own endpoint, never a field you send:

```bash
curl -XPOST $ADMIN/api/v1/object-types/ontology.sales.Customer/certify -H "$AUTH" -d '{}'
# → { …, "status": "CERTIFIED", "certifiedBy": "alice", "effectiveStatus": "CERTIFIED" }

# Undo, or retire:
curl -XPOST $ADMIN/api/v1/object-types/ontology.sales.Customer/decertify -H "$AUTH" \
  -d '{"status":"DEPRECATED"}'
```

The certifier is **the caller** — `status`, `certifiedBy`, `certifiedAt` and the fingerprint are
server-owned and ignored on register/update, so nobody can register a definition that declares
itself certified, or name someone else as the approver.

### Certification breaks when the definition changes

Certifying stamps a fingerprint of the parts that decide what the definition *resolves to* — the
read source, the property→column bindings, the primary key, the join keys. Change any of them and
`effectiveStatus` drops to `STALE` on the next read:

```json
{ "fqn": "ontology.sales.Customer", "status": "CERTIFIED",
  "effectiveStatus": "STALE",
  "certificationNote": "the definition changed after it was certified" }
```

Wording is deliberately excluded: editing a description, a title, a synonym or a tag never revokes
a sign-off. Re-certify to sign off on the new shape.

### Trust cannot exceed what it is built on

`effectiveStatus` is capped by dependencies:

- an **object type** cannot be more trusted than the semantic view it reads from;
- a **link type** cannot be more trusted than either of the object types it connects;
- an **action type** cannot be more trusted than the object type it mutates.

So a CERTIFIED object type over a DRAFT view reports `DRAFT`, with the reason in
`certificationNote`. An agent asking "is everything behind this answer certified?" reads one field
per object instead of walking the graph itself.

### Where it shows up

- **Reads carry it.** `GET /api/v1/object-types`, `GET .../{fqn}`, the ObjectSet
  `POST .../query` response and the link `POST .../traverse` response all include
  `effectiveStatus` (+ `certificationNote`), so a client rendering rows knows what they are worth
  without a second round-trip.
- **MCP** returns the same fields through `list_object_types` / `describe_object_type`.
- **Admin UI** shows the badge on the Object Types page, with a Certify button and the reason as
  a tooltip on a STALE badge.

### Who may certify

Certification is an ownership decision, not an editing one, so it has its own IAM action —
`ontology:Certify` — separate from the administrator rights needed to *edit* the ontology. That
lets a domain owner bless their own subtree without being made an admin:

```json
{ "Sid": "CertifySalesOntology", "Effect": "Allow",
  "Action": "ontology:Certify", "Resource": "ontology.sales.*" }
```

Administrators may always certify. Every certify / decertify is written to the
[Audit Log](audit-log.md) as `ontology:certify` / `ontology:decertify`, and a refusal as
`ontology:certify:denied`.

## REST API summary

All routes are under the master's admin HTTP port and reuse the same IAM token
as the rest of the admin surface. Reads are IAM-filtered on the backing read
source; registration/deletion is administrator-gated; action invoke is gated on
`ontology:InvokeAction`; certification on `ontology:Certify`.

| Method + path | Purpose |
| --- | --- |
| `GET/POST /api/v1/object-types` | List / register object types. |
| `GET/DELETE /api/v1/object-types/{fqn}` | Fetch / delete one. |
| `POST /api/v1/object-types/{fqn}/query` | ObjectSet query (filters / select / limit). |
| `GET/POST /api/v1/link-types` | List / register link types. |
| `GET/DELETE /api/v1/link-types/{fqn}` | Fetch / delete one. |
| `POST /api/v1/link-types/{fqn}/traverse` | Traverse from a source object (JOIN or GRAPH). |
| `GET/POST /api/v1/action-types` | List / register action types. |
| `GET/DELETE /api/v1/action-types/{fqn}` | Fetch / delete one. |
| `POST /api/v1/action-types/{fqn}/invoke` | Governed write-back (DML or OPERATION). |
| `GET/POST /api/v1/action-workflows` | List / register workflows (JSON). |
| `POST /api/v1/action-workflows/yaml` | Register a workflow from YAML. |
| `GET /api/v1/action-workflows/{fqn}` · `/yaml` | Fetch one (JSON) · export editable YAML. |
| `DELETE /api/v1/action-workflows/{fqn}` | Delete one. |
| `POST /api/v1/action-workflows/{fqn}/invoke` | Run the Saga (gated on `ontology:InvokeWorkflow`). |
| `POST /api/v1/{object-types,link-types,action-types,action-workflows,retrievers}/{fqn}/certify` | Sign off (gated on `ontology:Certify`). |
| `POST /api/v1/…/{fqn}/decertify` | Return to `DRAFT`, or `{"status":"DEPRECATED"}` to retire. |

## Related

- [Semantic Layer](semantic-layer.md) — metrics + dimensions the engine rewrites.
- [Retrievers](retrievers.md) — vector / graph / full-text retrieval pushdown.
- [Connector Architecture](connector-architecture.md) — how catalogs and
  connectors (including `rest-operation`) are registered.
- [IAM](iam.md) — the actions and grants the ontology's governance builds on.
