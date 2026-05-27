# Semantic Layer

Ontul's semantic layer is a thin metadata wrapper that lives on top of a regular Calcite view. It pairs the view's SELECT statement (`baseSql`) with business-meaning metadata — **metrics**, **dimensions**, **descriptions**, and **synonyms** — and persists everything in the master's MetadataStore so that REST clients, the Admin UI, and the MCP server all read from the same authoritative definition.

The goal is to give LLM consumers a curated, IAM-aware "menu" of business measures *before* a query is built — what numbers exist, what they mean, and what they're called in plain language — without forcing every team to re-derive the same `SUM(amount * (1 - discount))` for the tenth time.

This page describes Phase 1 of the implementation. See [Scope and roadmap](#scope-and-roadmap) for what is intentionally deferred.

## What a semantic view looks like

A semantic view binds five pieces of information to one fully-qualified name (`catalog.schema.name`):

| Field | Purpose |
| --- | --- |
| `baseSql` | The plain SELECT that the view exposes. Cross-catalog joins are allowed — federation is one of Ontul's strengths. |
| `description` | One-paragraph human description of the view's subject area. |
| `metrics[]` | Aggregations defined against `baseSql`. Each has a `name`, an `expr` (the actual SQL fragment), a `description`, and a list of `synonyms`. |
| `dimensions[]` | "Slice-by" columns already present in `baseSql`'s result schema. Carries `name`, `description`, and `synonyms` — the column itself already exists, so the engine has nothing extra to do here. |
| `owner` | Who registered it. Stamped from the IAM identity on the POST. |

The store keeps the definitions verbatim. The engine does **not** rewrite `SELECT revenue FROM lineitem_sales` into `SELECT SUM(l_extendedprice * (1 - l_discount)) FROM …` at query time. LLM consumers receive the metric expressions through the MCP tools and generate already-expanded SQL themselves. This trade-off keeps Phase 1 dependency-free of Calcite RelOpt rules; full server-side metric rewriting is a Phase 2 concern.

## How it's stored and synced

| Concern | Implementation |
| --- | --- |
| Persistence | RocksDB-backed `MetadataStore` with key prefix `semantic:`. The same store that holds catalog configs and dynamic settings. |
| Master replication | Rides the standard `exportSnapshot` / `importSnapshot` path — every `semantic:*` key syncs to follower masters with the rest of the cluster metadata. |
| Calcite view registration | On POST, the master auto-registers an in-memory `OntulViewTable` so `SELECT * FROM <fqn>` plans immediately. On every catalog reload (including follower `METADATA_SYNC_PUSH`), `MasterServer.reregisterSemanticViews()` re-walks the store and puts each view back into Calcite — necessary because catalog reload rebuilds the registry from connectors, and semantic views aren't connector-derived. |
| Rolling-upgrade safety | `SemanticViewDef` and its nested `Metric` / `Dimension` records are annotated `@JsonIgnoreProperties(ignoreUnknown=true)`, so a future field addition stays readable by older masters. |

## REST API

Admin HTTP server on the master exposes five endpoints, all returning JSON. Reads (GET) succeed for any authenticated user whose IAM token allows `data:SelectTable` on the view's FQN. Writes (POST / DELETE) require an administrator.

| Method | Path | Body | Description |
| --- | --- | --- | --- |
| `POST` | `/api/v1/semantic-views` | `SemanticViewDef` JSON | Register a new semantic view. Parses `baseSql` and every metric expression to catch malformed SQL up-front (returns 400 with the parser's message); does **not** plan, so a reference to a not-yet-existing table doesn't block registration. On success, auto-registers the underlying Calcite view and notifies workers. |
| `GET` | `/api/v1/semantic-views` | — | List every visible semantic view. Optional `?catalog=…&schema=…` filter. IAM-filtered: entries the caller cannot SELECT are hidden. |
| `GET` | `/api/v1/semantic-views/{catalog}.{schema}.{name}` | — | Fetch one definition. 404 if it doesn't exist, 403 if IAM denies SELECT on the FQN. |
| `DELETE` | `/api/v1/semantic-views/{catalog}.{schema}.{name}` | — | Remove the definition. (Calcite has no remove-view API, so the in-memory `ViewTable` lingers until the next catalog refresh; the metadata that callers actually consume is gone immediately.) |
| `GET` | `/api/v1/semantic-metrics/search` | `?q=<query>&limit=<n>` | Natural-language metric search. Matches against metric names, synonyms, and descriptions across every semantic view the caller can see. Returns ranked hits with `fqn`, `metricName`, `score`, and `matchedOn`. |

### Search ranking

The Phase 1 matcher is intentionally simple. Scoring:

| Match kind | Score |
| --- | --- |
| Synonym exact (case-insensitive) | 100 |
| Synonym substring | 80 |
| Metric name substring | 60 |
| Description substring | 40 |

Smarter ranking (TF-IDF, embedding) is deferred until Phase 1 usage data tells us what queries users actually issue.

## MCP tools

The Ontul MCP server registers three semantic-layer tools alongside its existing read-side wrappers (`ontul_query`, `ontul_list_catalogs`, etc.):

| Tool | Arguments | Purpose |
| --- | --- | --- |
| `ontul_list_semantic_views` | `catalog?`, `schema?` | List every semantic view the caller can see. The LLM uses this to discover what curated business definitions exist before generating SQL. |
| `ontul_describe_semantic_view` | `fqn` (required) | Return the full definition — `baseSql`, metric expressions, dimensions, synonyms, description. The LLM reads this to learn how to expand a metric reference inline. |
| `ontul_search_metrics` | `query` (required), `limit?` | Natural-language metric finder. Useful when the user asks "show me revenue" — the LLM calls this to locate the right `fqn` + `metricName` first, then follows up with `ontul_describe_semantic_view` to grab the expression. |

These tools talk to the master's admin HTTP port over REST, not Flight SQL. The same `ONTUL_USER_TOKEN` that authenticates Flight is accepted as a `Token <token>` Authorization header, so no separate JWT is needed. The admin URL can be overridden with `ONTUL_ADMIN_URL` (default `http://${ONTUL_HOST}:8080`).

## Authentication and IAM

| Operation | Required permission |
| --- | --- |
| `GET` list / get / search | Authenticated user whose IAM policies grant `data:SelectTable` on the view's FQN. Entries the caller can't SELECT are filtered out of list and search results (not returned as 403 — invisible). |
| `POST` register | Caller must be in the administrator group (`AuthManager.hasAdministratorAccess`). Phase 2 will introduce a finer-grained `semantic:Register` action so per-catalog ownership becomes possible. |
| `DELETE` | Same as POST. |

The MCP server inherits IAM from the user token it's launched with. The MCP layer adds no extra access control — the master enforces every filter server-side.

## Example: register and query a sales view

The walkthrough below assumes a default local cluster (admin REST on `http://localhost:8090`, default admin password rotated to `Admin123!`) and the built-in TPC-H connector — no external storage is needed. The exact same flow is exercised by `tests/e2e-semantic-views.sh` in the Ontul source tree.

### 1. Acquire a JWT

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
    -d '{
      "name": "tpch",
      "config": {"connector": "tpch", "scale.factor": "0.01"}
    }'
```

### 3. Register a semantic view over `tpch.tiny.lineitem`

```bash
curl -sf -X POST http://localhost:8090/api/v1/semantic-views \
    -H "Authorization: Bearer $TOKEN" \
    -H 'Content-Type: application/json' \
    -d '{
      "catalog": "tpch",
      "schema":  "tiny",
      "name":    "lineitem_sales",
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

### 4. Fetch it back

```bash
curl -sf "http://localhost:8090/api/v1/semantic-views/tpch.tiny.lineitem_sales" \
    -H "Authorization: Bearer $TOKEN"
```

```json
{
  "catalog": "tpch",
  "schema":  "tiny",
  "name":    "lineitem_sales",
  "fqn":     "tpch.tiny.lineitem_sales",
  "baseSql": "SELECT l_orderkey, l_extendedprice, l_discount, l_quantity, l_shipdate FROM tpch.tiny.lineitem",
  "description": "Per-line sales facts for analytics.",
  "metrics": [
    {"name":"revenue","expr":"SUM(l_extendedprice * (1 - l_discount))","description":"Total net revenue","synonyms":["매출","net_revenue","sales_amount"]},
    {"name":"units",  "expr":"SUM(l_quantity)","description":"Units shipped","synonyms":["quantity","수량"]}
  ],
  "dimensions": [
    {"name":"l_shipdate","description":"Ship date","synonyms":["ship_date","출하일"]}
  ],
  "createdAt": 1779880934424,
  "updatedAt": 1779880934424,
  "owner":     "admin"
}
```

### 5. Find a metric by Korean synonym

```bash
curl -sf "http://localhost:8090/api/v1/semantic-metrics/search?q=%EB%A7%A4%EC%B6%9C&limit=5" \
    -H "Authorization: Bearer $TOKEN"
```

```json
[
  {"fqn":"tpch.tiny.lineitem_sales","metricName":"revenue","score":100,"matchedOn":"synonym:매출"}
]
```

A substring lookup (`q=net_rev`) returns the same metric with `score: 80, matchedOn: "synonym:net_revenue"`.

### 6. Query the auto-registered view

The POST in step 3 already added an in-memory `OntulViewTable` for `tpch.tiny.lineitem_sales`. SELECT works immediately:

```bash
curl -sf -X POST http://localhost:8090/admin/query/execute \
    -H "Authorization: Bearer $TOKEN" \
    -H 'Content-Type: application/json' \
    -d '{"sql":"SELECT COUNT(*) AS c FROM tpch.tiny.lineitem_sales"}'
```

To consume the metric, the caller inlines its `expr`. An LLM that just read the description via MCP would generate:

```sql
SELECT l_shipdate,
       SUM(l_extendedprice * (1 - l_discount)) AS revenue
  FROM tpch.tiny.lineitem_sales
 GROUP BY l_shipdate
 ORDER BY l_shipdate;
```

### 7. Delete

```bash
curl -sf -X DELETE http://localhost:8090/api/v1/semantic-views/tpch.tiny.lineitem_sales \
    -H "Authorization: Bearer $TOKEN"
```

## Example: MCP discovery flow

When an LLM connected to Ontul MCP receives a user request like _"show me revenue by ship date"_:

1. **Locate the metric.** The LLM calls `ontul_search_metrics({"query":"revenue"})`. The server returns the ranked hit `{fqn:"tpch.tiny.lineitem_sales", metricName:"revenue", score:80, matchedOn:"synonym:net_revenue"}`.
2. **Read the definition.** The LLM calls `ontul_describe_semantic_view({"fqn":"tpch.tiny.lineitem_sales"})`. Response includes the `expr: "SUM(l_extendedprice * (1 - l_discount))"` and the available dimensions (`l_shipdate`).
3. **Generate expanded SQL.** Using the description, the LLM produces the GROUP BY query shown above and calls `ontul_query`. Standard IAM, lineage, and result formatting apply.

The point of steps 1 and 2 is that the LLM never sees a raw connector table without the curated business context — what columns mean, what aggregations have already been agreed on, and what the same measure is called in different languages.

## Validation behavior

| Failure | Status | Reason |
| --- | --- | --- |
| Missing `catalog` / `schema` / `name` / `baseSql` | 400 | Required fields. |
| `baseSql` fails Calcite parsing | 400 | Parse-only — table existence is not checked. |
| Any `metric.expr` fails Calcite parsing | 400 | Wrapped as `SELECT <expr> FROM (<baseSql>) t` and parsed. |
| `metric.name` or `metric.expr` missing | 400 | — |
| Calcite view registration throws | logged warning, **registration still succeeds** | The metadata is already persisted; an operator can fix the underlying table and `refreshCatalog` to recover. |

Validation deliberately stops at parsing. Full semantic validation (column types, function signatures) would require planning against the live catalog and would block registration for tables that don't exist yet. That's left to query time, where errors surface naturally.

## Scope and roadmap

**In Phase 1** (this page):

- Metric / dimension / synonym / description metadata, persisted in MetadataStore.
- REST CRUD + search endpoints with IAM filtering.
- MCP tools for list / describe / search.
- Auto-registration of the underlying view in Calcite so SELECT works immediately.
- Follower-master re-registration on every catalog reload.

**Permanently excluded:**

- Streaming / batch-job "semantic" metadata. The industry concept is query-only — Airflow / dbt / Dagster model descriptions are documentation, not metrics — and inventing one here would be feature creep.
- Materialization of semantic views. They stay regular SQL views; consumers run the same query path as any other SELECT.

**Phase 2 (under consideration once Phase 1 has usage data):**

- Engine-side metric rewriting (`SELECT revenue FROM v` → expanded SUM), including cross-table joins and CTE caching. This is the bulk of the remaining work.
- Versioning and approval workflow (who owns "매출"?).
- Bidirectional sync with Iceberg column comments and partition specs.
- Cluster-wide on/off toggle for the semantic surface.
- Smarter natural-language matching (TF-IDF / embedding).
- Per-catalog ACLs (`semantic:Register` action) so semantic ownership can be delegated below the administrator role.

## Operational notes

- The `semantic:` keys in MetadataStore are tiny (one JSON document per view) and ride the same RocksDB encryption that catalog configs use. They are included in master backup / restore automatically.
- A corrupt JSON value at a `semantic:*` key is logged and skipped — it does not break `LIST`. Use `DELETE` on the malformed FQN to recover.
- On every catalog reload the master logs `Re-registered N semantic view(s) into Calcite`. Watch this line in the Master log to confirm propagation after a `loadPersistedCatalogs` event.
- `notifyWorkersCatalogChanged` is called after each `POST` so workers refresh their cached catalog snapshot. Plan-time view resolution happens on the master, but the worker-side cache is kept consistent for any future planning that runs there.
