# Semantic Layer

Ontul's semantic layer turns a regular Calcite view into a curated business surface. The view's `baseSql` is paired with metric definitions, dimension annotations, multilingual synonyms, governance metadata, mandatory filters, and conformed-dimension joins. The engine then rewrites queries on the fly so callers — Tableau dashboards, LLM agents, SQL Runner — can write the names they already know (`revenue`, `profit_margin`, `customer.region`) and the planner expands them into the right aggregation, filter, and JOIN.

The goal is one definition of truth per measure, enforced server-side, consumable through SQL / JDBC / Arrow Flight / MCP / REST without any of those clients duplicating the formula.

```text
LLM       ─┐
Tableau   ─┼─►  Ontul semantic layer  ─►  Calcite planner  ─►  Iceberg / JDBC / NeorunBase ...
DBeaver   ─┘    (rewrite + RBAC +
                 filter injection)
```

## What a semantic view binds

| Field | Purpose |
| --- | --- |
| `baseSql` | The plain SELECT the view exposes. Cross-catalog joins are allowed — federation is one of Ontul's strengths. |
| `description` | One-paragraph human description of the view's subject area. |
| `metrics[]` | Aggregations defined against `baseSql`. Each has a `name`, `expr` (the SQL fragment), `description`, `synonyms[]`, and optionally `allowedRoles[]` (metric-level RBAC) and `mandatoryFilters[]` (per-metric row scoping). |
| `dimensions[]` | "Slice-by" columns from `baseSql`. Names + descriptions + synonyms feed BI / MCP / search. |
| `joins[]` | Conformed-dimension joins — declared once, auto-injected only when their columns are referenced. |
| `mandatoryFilters[]` | View-level row-scoping predicates with `${user.id}` / `${user.roles}` / `${user.attr.X}` templating. |
| `tags[]`, `status`, `certifiedBy`, `certifiedAt`, `owner` | Governance metadata surfaced through the Admin UI, MCP, and BI tools. |

The store keeps every definition verbatim. The engine takes responsibility for expansion at query time: `SELECT revenue, customer.region FROM tpch.tiny.lineitem_sales` becomes an aggregated, joined, filtered query that the validator and the workers actually execute.

## What the engine does at query time

When a SELECT references a semantic view, the rewriter runs **before** validation and applies these passes in order:

1. **Scope discovery** — every FROM-clause table is checked against the semantic catalog. Plain tables stay untouched; semantic views contribute their metric / dimension / join metadata to a per-query scope.
2. **`SELECT *` expansion** — for a single in-scope semantic view, `*` is expanded to its explicit metric + dimension list. `<alias>.*` works the same way. Mixed `*` across a semantic view and a plain table is left alone (the validator handles the plain part).
3. **Conformed-dimension join injection** — any reference of the form `<joinName>.<column>` is matched against declared joins. Each used join becomes a `LEFT JOIN <target> AS <joinName> ON <resolved-template>` appended to FROM. Unused joins stay out of the plan.
4. **Metric expansion** — bare or qualified metric identifiers in the SELECT list and HAVING clause are replaced with their parsed `expr`, recursively (derived metrics expand fully, with cycle detection capped at 16 hops).
5. **RBAC** — any metric whose `allowedRoles` excludes the caller raises a denial before the expression is parsed, so the formula doesn't leak via error messages.
6. **HAVING rewrite** — metric references inside `HAVING revenue > 100` get the same expansion as the SELECT list. `WHERE` is intentionally not touched (aggregations in WHERE are illegal anyway).
7. **Mandatory filter injection** — view-level and per-metric mandatory predicates are AND'd into the WHERE clause, with `${user.id}` / `${user.roles}` / `${user.attr.X}` substitution from the authenticated session. Anonymous (system) queries skip this step.
8. **Auto `GROUP BY`** — when at least one metric was injected and the user didn't supply a GROUP BY, every non-metric expression in the SELECT list becomes a grouping key.

The pass order matters: star expansion fires first so its identifiers go through normal metric resolution; join injection fires next so the FROM clause is final by the time auto-grouping inspects column references.

## Quick walkthrough: define and query a view

This is the minimum surface needed to register, see, and query a semantic view. The full Phase 3 capabilities — joins, derived metrics, mandatory filters, RBAC — are layered on top later in the page.

### 1. Acquire a token

```bash
TOKEN=$(curl -sf -X POST http://localhost:8090/admin/auth/login \
    -H 'Content-Type: application/json' \
    -d '{"username":"admin","password":"Admin123!"}' \
    | python3 -c "import sys,json; print(json.load(sys.stdin)['accessToken'])")
```

### 2. Register the built-in TPC-H catalog

```bash
curl -sf -X POST http://localhost:8090/admin/catalogs \
    -H "Authorization: Bearer $TOKEN" \
    -H 'Content-Type: application/json' \
    -d '{"name":"tpch","config":{"connector":"tpch","scale.factor":"0.01"}}'
```

### 3. Register a semantic view

```bash
curl -sf -X POST http://localhost:8090/api/v1/semantic-views \
    -H "Authorization: Bearer $TOKEN" \
    -H 'Content-Type: application/json' \
    -d '{
      "catalog": "tpch", "schema": "tiny", "name": "lineitem_sales",
      "description": "Per-line sales facts for analytics.",
      "baseSql": "SELECT l_orderkey, l_extendedprice, l_discount, l_quantity, l_shipdate FROM tpch.tiny.lineitem",
      "metrics": [
        {
          "name": "revenue",
          "expr": "SUM(l_extendedprice * (1 - l_discount))",
          "description": "Total net revenue",
          "synonyms": ["매출", "net_revenue", "sales_amount"]
        },
        {
          "name": "units",
          "expr": "SUM(l_quantity)",
          "description": "Units shipped",
          "synonyms": ["quantity", "수량"]
        }
      ],
      "dimensions": [
        {"name": "l_shipdate", "description": "Ship date", "synonyms": ["ship_date", "출하일"]}
      ]
    }'
```

The response is `201 Created` with `{"status":"ok","fqn":"tpch.tiny.lineitem_sales"}`.

### 4. Query with the metric's name directly

```sql
SELECT l_shipdate, revenue
  FROM tpch.tiny.lineitem_sales
 WHERE l_shipdate >= DATE '1995-01-01'
 ORDER BY l_shipdate;
```

The planner expands `revenue` to `SUM(l_extendedprice * (1 - l_discount))`, adds `GROUP BY l_shipdate`, and returns one row per ship date. No client-side templating needed.

### 5. `SELECT *` returns the curated columns

```sql
SELECT * FROM tpch.tiny.lineitem_sales;
```

Equivalent to `SELECT revenue, units, l_shipdate FROM tpch.tiny.lineitem_sales` — measures first, dimensions next, all aggregated and grouped automatically. The raw `baseSql` columns are not exposed; the semantic abstraction is the API.

### 6. Find the metric by Korean synonym

```bash
curl -sf "http://localhost:8090/api/v1/semantic-metrics/search?q=%EB%A7%A4%EC%B6%9C&limit=5" \
    -H "Authorization: Bearer $TOKEN"
```

```json
[ {"fqn":"tpch.tiny.lineitem_sales","metricName":"revenue","score":100,"matchedOn":"synonym:매출"} ]
```

## Query rewriting in detail

### Bare metric reference

```sql
SELECT l_shipdate, revenue
  FROM tpch.tiny.lineitem_sales;
```

is rewritten to

```sql
SELECT l_shipdate,
       SUM(l_extendedprice * (1 - l_discount)) AS revenue
  FROM tpch.tiny.lineitem_sales
 GROUP BY l_shipdate;
```

The output column keeps the user-typed name (`revenue`), so a downstream BI tool sees the column it expects.

### Qualified metric reference

```sql
SELECT s.l_shipdate, s.revenue
  FROM tpch.tiny.lineitem_sales s
 WHERE s.l_shipdate >= DATE '2024-01-01';
```

The qualifier `s` is preserved in the rewritten SQL. Multiple views in one query can disambiguate their metrics with their own aliases.

### `HAVING` on a metric

```sql
SELECT l_shipdate, revenue
  FROM tpch.tiny.lineitem_sales
 GROUP BY l_shipdate
HAVING revenue > 1000000;
```

becomes

```sql
SELECT l_shipdate,
       SUM(l_extendedprice * (1 - l_discount)) AS revenue
  FROM tpch.tiny.lineitem_sales
 GROUP BY l_shipdate
HAVING SUM(l_extendedprice * (1 - l_discount)) > 1000000;
```

Metric filters belong in `HAVING`; using them in `WHERE` is an aggregation-in-WHERE error from Calcite, which is the standard SQL behavior — the semantic layer doesn't try to be cleverer than the language.

### User-supplied alias is preserved

```sql
SELECT l_shipdate, revenue AS daily_total
  FROM tpch.tiny.lineitem_sales;
```

→ `... SUM(...) AS daily_total`. The user's alias wins over the metric's name.

### Mixed metric + raw column reference

```sql
SELECT customer.c_nation, revenue
  FROM tpch.tiny.lineitem_sales;
```

Here `customer.c_nation` is a conformed-dimension reference (covered below). `c_nation` becomes a group-by column; `revenue` becomes the aggregated measure. The planner handles both in one pass.

## Phase 3 capabilities

### Mandatory filters with user-context templating

Mandatory filters are predicate templates AND'd into every query against the view. They support three substitution tokens:

| Token | Resolves to |
| --- | --- |
| `${user.id}` | The authenticated caller's user id, single-quoted. |
| `${user.roles}` | Comma-separated quoted role list, suitable for `role IN (${user.roles})`. |
| `${user.attr.<key>}` | The named attribute from the IAM user record. Missing keys render as `''` — fail-closed. |

A multi-tenant view definition might look like

```json
{
  "catalog": "saas", "schema": "core", "name": "orders",
  "baseSql": "SELECT order_id, tenant_id, total, status, ship_date FROM saas.core.raw_orders",
  "metrics": [
    {"name": "revenue", "expr": "SUM(total)",
     "mandatoryFilters": ["status = 'COMPLETED'"]}
  ],
  "dimensions": [{"name": "ship_date"}],
  "mandatoryFilters": [
    "tenant_id = ${user.attr.tenant_id}"
  ]
}
```

For a caller whose IAM record has `attributes.tenant_id = 'acme-co'`, the query

```sql
SELECT ship_date, revenue FROM saas.core.orders;
```

becomes

```sql
SELECT ship_date, SUM(total) AS revenue
  FROM saas.core.orders
 WHERE tenant_id = 'acme-co'      -- view-level
   AND status = 'COMPLETED'        -- per-metric, applied because revenue is referenced
 GROUP BY ship_date;
```

The per-metric filter only fires when `revenue` is part of the SELECT. A query that asks for `attempts` (a metric without the COMPLETED scope) wouldn't get the status filter — measures stay independent.

Anonymous / system-internal contexts (no user id) skip the entire injection pass — internal admin queries are trusted to know what they read.

The same `${user.id}` / `${user.roles}` / `${user.attr.<key>}` substitution powers [IAM column masking](iam.md#column-masking) — mandatory filters keep the row in or out, masking transforms the columns. Combining the two gives full multi-tenant + PII coverage in one policy stack.

### Derived metrics

A metric expression may reference other metrics from the same view. The rewriter walks the parsed expression tree and recursively splices the referenced metric's own expansion, with cycle detection.

```json
"metrics": [
  {"name": "revenue",        "expr": "SUM(amount)"},
  {"name": "cost",           "expr": "SUM(unit_cost * quantity)"},
  {"name": "profit",         "expr": "revenue - cost"},
  {"name": "profit_margin",  "expr": "(revenue - cost) / revenue"}
]
```

A query like

```sql
SELECT region, profit_margin FROM tpch.tiny.sales;
```

expands to

```sql
SELECT region,
       (SUM(amount) - SUM(unit_cost * quantity))
         / SUM(amount) AS profit_margin
  FROM tpch.tiny.sales
 GROUP BY region;
```

Cycles (`a → b → a`) raise a `DerivedMetricCycleException` with the chain so the operator can see exactly which metrics loop. Recursion is capped at depth 16 as a belt-and-braces safeguard.

### Conformed-dimension joins

A `joins[]` entry declares "if a query references columns prefixed by `<name>`, auto-add this JOIN":

```json
{
  "catalog": "tpch", "schema": "tiny", "name": "lineitem_sales",
  "baseSql": "SELECT l_orderkey, l_partkey, l_custkey, l_extendedprice, l_discount, l_shipdate FROM tpch.tiny.lineitem",
  "metrics": [
    {"name": "revenue", "expr": "SUM(l_extendedprice * (1 - l_discount))"}
  ],
  "dimensions": [{"name": "l_shipdate"}],
  "joins": [
    {
      "name": "customer",
      "target": "tpch.tiny.customer",
      "onTemplate": "${this}.l_custkey = ${customer}.c_custkey",
      "type": "LEFT"
    },
    {
      "name": "part",
      "target": "tpch.tiny.part",
      "onTemplate": "${this}.l_partkey = ${part}.p_partkey",
      "type": "LEFT"
    }
  ]
}
```

`${this}` resolves to the source view's alias (the alias the caller wrote in FROM, or the bare view name if unaliased), and `${customer}` / `${part}` resolve to the join's local name. The templating means the same declaration works whether the caller writes `FROM lineitem_sales` or `FROM tpch.tiny.lineitem_sales AS ls`.

```sql
SELECT customer.c_nation, part.p_brand, revenue
  FROM tpch.tiny.lineitem_sales;
```

becomes (LEFT JOINs added in declaration order, only for joins actually referenced)

```sql
SELECT customer.c_nation, part.p_brand,
       SUM(l_extendedprice * (1 - l_discount)) AS revenue
  FROM tpch.tiny.lineitem_sales
  LEFT JOIN tpch.tiny.customer customer
    ON lineitem_sales.l_custkey = customer.c_custkey
  LEFT JOIN tpch.tiny.part part
    ON lineitem_sales.l_partkey = part.p_partkey
 GROUP BY customer.c_nation, part.p_brand;
```

Joins that aren't referenced never get added — declaring ten possible joins doesn't cost ten table scans if the user only touches one. `LEFT` is the default; `INNER` is available when you want fact rows dropped on dimension misses.

### Metric-level RBAC

Each metric carries an optional `allowedRoles[]` set. Empty (the default) means the metric is visible to every authenticated user. Non-empty means the caller's group set must intersect.

```json
"metrics": [
  {"name": "revenue",      "expr": "SUM(amount)"},
  {"name": "vip_revenue",  "expr": "SUM(amount) FILTER (WHERE tier='VIP')",
   "allowedRoles": ["analyst", "finance"]}
]
```

A user in the `viewer` group running `SELECT vip_revenue FROM ...` is denied before the formula is parsed, so the error message never leaks `tier='VIP'` to an unauthorized user. RBAC is also applied to `SELECT *` expansions — a user without access to `vip_revenue` triggers the denial even when they didn't type the metric name explicitly.

System-internal queries (no authenticated user — the planner's anonymous context) bypass RBAC. This is the intentional escape hatch for internal admin work that legitimately bypasses IAM.

### Governance metadata

Each semantic view carries lifecycle, ownership, and labeling fields used by the Admin UI, MCP, and BI tools to surface trust signals.

| Field | Meaning |
| --- | --- |
| `status` | `DRAFT` (default) / `CERTIFIED` / `DEPRECATED`. BI clients can filter by this; LLMs should prefer CERTIFIED. |
| `certifiedBy` | User id stamped when a view is certified via `POST /certify`. |
| `certifiedAt` | Epoch-ms instant of certification. |
| `owner` | User id that registered the view; auto-stamped on POST. |
| `tags[]` | Free-form labels — `["finance", "gold-tier", "pii"]`. Used by UI filtering. |

Two endpoints manage governance independently of schema changes:

```bash
# Patch tags / status / description in one call
curl -sf -X PATCH http://localhost:8090/api/v1/semantic-views/tpch.tiny.lineitem_sales \
    -H "Authorization: Bearer $TOKEN" \
    -H 'Content-Type: application/json' \
    -d '{"tags":["finance","gold-tier"],"status":"DRAFT"}'

# Stamp certification audit
curl -sf -X POST http://localhost:8090/api/v1/semantic-views/tpch.tiny.lineitem_sales/certify \
    -H "Authorization: Bearer $TOKEN"
```

The `/certify` endpoint is separate from the generic PATCH so audit trails treat certification as a distinct event — the caller's id becomes `certifiedBy`, not a free-text field.

### Lineage integration

When a semantic view is registered, the master synthesizes `SELECT <every metric expr>, <every dim> FROM (baseSql) t`, plans it, and feeds the result through both:

- **Table-level lineage** — every SCAN node's source table is recorded against the view's FQN (`semantic-view` operation type).
- **Column-level lineage** — each metric / dimension output column is traced back to its `(table, column)` origin via Calcite's column-origin metadata, with `derived=true` flagged when the value passes through an expression.

So `/admin/lineage/columns?table=tpch.tiny.lineitem_sales&column=revenue` returns the underlying `l_extendedprice` and `l_discount` columns. The same OpenLineage emission path that handles CTAS / INSERT INTO covers semantic views — no extra integration needed for downstream catalogs like DataHub or OpenMetadata.

## Discovery: synonyms, search, MCP

The same metric is often called different things — "revenue" / "net revenue" / "매출" / "sales_amount" — and BI users / LLM agents need to find it by whatever name they thought of.

The Phase 1 ranker is intentionally simple, fast, and dependency-free:

| Match kind | Score |
| --- | --- |
| Synonym exact (case-insensitive) | 100 |
| Synonym substring | 80 |
| Metric name substring | 60 |
| Description substring | 40 |

The same ranker powers `GET /api/v1/semantic-metrics/search` and the MCP `ontul_search_metrics` tool. Results are IAM-filtered server-side — entries the caller can't SELECT never appear in search results.

### MCP discovery flow

When an LLM agent receives _"show me revenue by ship date"_:

1. `ontul_search_metrics({"query":"revenue"})` → `[{fqn:"tpch.tiny.lineitem_sales", metricName:"revenue", score:100, matchedOn:"synonym:매출"}]`
2. `ontul_describe_semantic_view({"fqn":"tpch.tiny.lineitem_sales"})` → full definition including `expr`, dimensions, status, certifiedBy, joins, mandatoryFilters.
3. `ontul_query({"sql":"SELECT l_shipdate, revenue FROM tpch.tiny.lineitem_sales"})` → the rewriter expands the metric and adds GROUP BY. The agent never has to re-derive the formula.

See [MCP Server](../reference/mcp-server.md) for full tool surface, env vars, and BI tool integration via Flight SQL is covered in [BI Integration](bi-integration.md).

## BI tool exposure

A semantic view appears to JDBC / ODBC introspection as if it were a regular VIEW whose columns are the metrics and dimensions. Tableau, Power BI, Looker, DBeaver, JetBrains DataGrip can browse and query without ever seeing the underlying `baseSql` schema.

`DESCRIBE tpch.tiny.lineitem_sales` returns the logical column list with a `Kind` discriminator:

```text
| Column     | Type      | Nullable | Kind     |
|------------|-----------|----------|----------|
| revenue    | AGGREGATE | YES      | MEASURE  |
| units      | AGGREGATE | YES      | MEASURE  |
| l_shipdate | DIMENSION | YES      | DIMENSION|
```

Flight SQL `CommandGetTables` marks the view as `VIEW` type (vs. `TABLE` for plain Iceberg / JDBC / NeorunBase tables) so BI tools render the correct icon and pull the column list from the semantic surface.

See [BI Integration](bi-integration.md) for end-to-end Tableau / DBeaver / Power BI setup.

## End-to-end example: tenant-scoped sales dashboard

A more realistic Phase-3 view that exercises mandatory filters, derived metrics, conformed dimensions, and governance simultaneously.

### Define the view

```bash
curl -sf -X POST http://localhost:8090/api/v1/semantic-views \
    -H "Authorization: Bearer $TOKEN" \
    -H 'Content-Type: application/json' \
    -d '{
      "catalog": "saas", "schema": "core", "name": "sales",
      "description": "Per-tenant sales facts — multi-tenant row-scoping enforced.",
      "baseSql": "SELECT order_id, tenant_id, customer_id, part_id, amount, unit_cost, quantity, status, ship_date FROM saas.core.raw_orders",
      "metrics": [
        {"name":"revenue", "expr":"SUM(amount)",
         "description":"Total net revenue",
         "synonyms":["매출","net_revenue"],
         "mandatoryFilters":["status = '\''COMPLETED'\''"]
        },
        {"name":"cost",    "expr":"SUM(unit_cost * quantity)",
         "description":"Total cost"},
        {"name":"profit",         "expr":"revenue - cost",
         "description":"Net profit (derived)"},
        {"name":"profit_margin",  "expr":"(revenue - cost) / revenue",
         "description":"Profit / revenue ratio"},
        {"name":"vip_revenue",    "expr":"SUM(amount) FILTER (WHERE status = '\''VIP'\'')",
         "description":"Revenue from VIP segment",
         "allowedRoles":["finance","analyst"]
        }
      ],
      "dimensions":[
        {"name":"ship_date","synonyms":["출하일"]}
      ],
      "joins":[
        {"name":"customer","target":"saas.core.customer",
         "onTemplate":"${this}.customer_id = ${customer}.id","type":"LEFT"},
        {"name":"part","target":"saas.core.part",
         "onTemplate":"${this}.part_id = ${part}.id","type":"LEFT"}
      ],
      "mandatoryFilters":["tenant_id = ${user.attr.tenant_id}"],
      "tags":["finance","gold-tier"]
    }'
```

### Certify it for production use

```bash
curl -sf -X POST http://localhost:8090/api/v1/semantic-views/saas.core.sales/certify \
    -H "Authorization: Bearer $TOKEN"
# → status: CERTIFIED, certifiedBy: admin, certifiedAt: 1779999999999
```

### One query touches every Phase-3 feature

```sql
SELECT customer.region, part.category, profit_margin
  FROM saas.core.sales
 WHERE ship_date >= DATE '2024-01-01'
 ORDER BY profit_margin DESC;
```

For caller `alice` whose IAM record has `attributes.tenant_id = 'acme-co'` and groups `[analyst]`, the rewriter produces

```sql
SELECT customer.region, part.category,
       (SUM(amount) - SUM(unit_cost * quantity)) / SUM(amount) AS profit_margin
  FROM saas.core.sales
  LEFT JOIN saas.core.customer customer
    ON sales.customer_id = customer.id
  LEFT JOIN saas.core.part     part
    ON sales.part_id     = part.id
 WHERE ship_date >= DATE '2024-01-01'
   AND tenant_id = 'acme-co'        -- view-level mandatory filter
   AND status = 'COMPLETED'          -- per-metric filter (revenue is part of profit_margin)
 GROUP BY customer.region, part.category
 ORDER BY profit_margin DESC;
```

If alice asks for `vip_revenue` she gets a result; the same query from a user in only the `viewer` group is denied at rewrite time with a `Forbidden: vip_revenue` error.

## Storage and replication

| Concern | Implementation |
| --- | --- |
| Persistence | RocksDB-backed `MetadataStore` with key prefix `semantic:`. Same store as catalog configs and dynamic settings. |
| Master replication | Rides standard `exportSnapshot` / `importSnapshot` — every `semantic:*` key syncs to follower masters with the rest of the cluster metadata. |
| Calcite view registration | On POST, the master auto-registers an in-memory view. On catalog reload, `MasterServer.reregisterSemanticViews()` re-walks the store and puts each view back. |
| Hot-path lookup | The `SemanticViewStore` fronts RocksDB with an in-memory cache; a query that touches N tables pays one disk read per FQN, not N. |
| Rolling-upgrade safety | `SemanticViewDef` and its nested records are `@JsonIgnoreProperties(ignoreUnknown=true)`; older masters silently ignore newer fields. |

## Authentication and IAM

| Operation | Required permission |
| --- | --- |
| `GET` list / get / search | Authenticated user whose IAM policies grant `data:SelectTable` on the view's FQN. Entries the caller can't SELECT are filtered out (invisible, not 403). |
| `POST` register, `PATCH` governance, `DELETE` | Administrator group membership (`AuthManager.hasAdministratorAccess`). |
| `POST /certify` | Administrator. The caller's id becomes `certifiedBy`. |
| Per-metric `allowedRoles` | Enforced by the rewriter against the planner's `SemanticContext`. The denial fires before parsing the metric expression. |
| Mandatory filter templating | The caller's `userId` / `groups` / `attributes` come from `AuthManager.getUser(...)` and are substituted into the WHERE clause server-side. |

The MCP server inherits IAM from the user token it's launched with — no extra access control layer at the MCP boundary.

## Validation behavior

| Failure | Status | Reason |
| --- | --- | --- |
| Missing `catalog` / `schema` / `name` / `baseSql` | 400 | Required fields. |
| `baseSql` fails Calcite parsing | 400 | Parse-only — table existence is not checked. |
| Any `metric.expr` fails Calcite parsing | 400 | Wrapped as `SELECT <expr> FROM (<baseSql>) t` and parsed. |
| `metric.name` or `metric.expr` missing | 400 | — |
| Invalid `status` on PATCH | 400 | Must be `DRAFT` / `CERTIFIED` / `DEPRECATED`. |
| Derived-metric cycle at query time | runtime error | `DerivedMetricCycleException` carries the chain. |
| RBAC denial at query time | runtime error | `MetricAccessDeniedException` — metric name only, no expression. |
| Calcite view registration throws on POST | logged warning, **registration still succeeds** | Metadata is already persisted; operator can `refreshCatalog` later. |

Validation deliberately stops at parsing. Full semantic validation (column types, function signatures) would require planning against the live catalog and would block registration for tables that don't exist yet. That's left to query time, where errors surface naturally.

## Scope and roadmap

**Shipped:**

- Phase 1: metric / dimension / synonym / description metadata, REST CRUD + IAM-filtered search, MCP tools (`ontul_list_semantic_views`, `ontul_describe_semantic_view`, `ontul_search_metrics`), auto-registration of the underlying Calcite view, follower re-registration on catalog reload.
- Phase 2: server-side query rewriting (metric expansion, auto `GROUP BY`, HAVING rewrite), `SELECT *` and `<alias>.*` expansion, metric-level RBAC, governance metadata (status, certify, tags, owner), lineage integration (table + column edges via OpenLineage), BI metadata exposure (Flight SQL `VIEW` type, `DESCRIBE` measure-first), `/api/v1/bi/connection-info` endpoint.
- Phase 3: mandatory filters with `${user.id}` / `${user.roles}` / `${user.attr.X}` templating (view-level + per-metric), derived metrics with recursive expansion and cycle detection, conformed-dimension joins with declarative ON templates (`${this}` / `${join-name}`), unused-join elimination.

**Permanently excluded:**

- Streaming / batch-job "semantic" metadata. The industry concept is query-only — Airflow / dbt / Dagster model descriptions are documentation, not metrics.
- Materialization of semantic views (reflections / pre-aggregations). Ontul's federated Iceberg + Arrow path is fast enough that the maintenance cost of materialization outweighs the gains; CTAS + lineage already exist if a specific dashboard ever proves the need.

**Under consideration when usage signals demand it:**

- Time intelligence DSL (`yoy`, `mtd`, `rolling_7d`) as first-class metric annotations rather than user-authored window functions.
- Hierarchies (`region → country → city`) as declarative drill paths surfaced through BI manifests.
- Definition-as-code CLI (`ontul semantic apply file.yaml`) over the existing REST API.
- Metric unit-test framework — `assertMetric("revenue","2024-01-01", expected=...)`.
- Smarter natural-language matching (TF-IDF / embedding) — Phase 1 substring ranking is intentionally minimal.

## Operational notes

- `semantic:` keys in MetadataStore are tiny (one JSON document per view) and ride the same RocksDB encryption that catalog configs use. They are included in master backup / restore automatically.
- A corrupt JSON value at a `semantic:*` key is logged and skipped — it does not break `LIST`. Use `DELETE` on the malformed FQN to recover.
- On every catalog reload the master logs `Re-registered N semantic view(s) into Calcite`. Watch this line to confirm propagation after a `loadPersistedCatalogs` event.
- `notifyWorkersCatalogChanged` is called after each successful `POST` so workers refresh their cached catalog snapshot. Plan-time rewriting happens on the master, but the worker-side cache is kept consistent.
- Rewriting failure modes are conservative: if the rewriter hits an unexpected runtime error on a specific SELECT, the original tree is returned and the planner proceeds. Phase 1 behavior (LLM-expanded SQL) keeps working even if a Phase 2 / 3 edge case trips. RBAC denials and derived-metric cycles are the two exceptions — those always surface so admins notice them.

## See also

- [BI Integration](bi-integration.md) — Tableau / DBeaver / Power BI / Looker setup.
- [MCP Server](../reference/mcp-server.md) — semantic-discovery tools for LLM agents.
- [IAM](iam.md) — user attributes feed `${user.attr.X}` templating; column masking and row-level filters are enforced server-side alongside metric RBAC.
- [Data Lineage](data-lineage.md) — metric → base-column edges are recorded automatically.
