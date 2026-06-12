# Retrievers (Multi-Modal Retrieval)

A **retriever** is a first-class, governed object in Ontul's semantic layer that wraps a backend-native, multi-modal retrieval query and exposes it for safe, parameterized invocation. Where a [semantic view](semantic-layer.md) curates *analytics* (metrics + dimensions the engine rewrites into Calcite SQL), a retriever curates *retrieval* — vector similarity, graph traversal, and full-text search — and pushes it down to a backing engine that can actually execute it.

Today the backing engine is **[NeorunBase](https://github.com/cloudcheflabs)**, whose single-SQL surface combines vector (ANN), full-text (BM25), and graph (PageRank / neighbor expansion) retrieval — purpose-built for graph-RAG. A retriever lets an agent on the [Ontul MCP server](../reference/mcp-server.md) query *semantic metrics* and *NeorunBase's vector / graph / full-text retrieval* through one governed surface.

```text
Agent (MCP) ─┬─►  ontul_search_metrics / ontul_query        ─►  semantic metrics  (Calcite rewrite)
             └─►  ontul_list/describe/invoke_retriever       ─►  retriever pushdown ─►  NeorunBase
                                                                  (HYBRID_SEARCH / GRAPH_NEIGHBORS / …)
```

## Why retrievers exist

Adding a NeorunBase catalog to Ontul lets you run plain relational `SELECT`s against NeorunBase tables. It does **not** let NeorunBase's special table-valued functions (`HYBRID_SEARCH`, `GRAPH_NEIGHBORS`, `PERSONALIZED_PAGERANK`) or operators (`<=>`, `@@`) flow through — Ontul's own Calcite planner does not recognize them and would strip or reject them.

A retriever solves this by **bypassing Ontul's planner for the retrieval query**: the master renders an admin-authored, NeorunBase-native SQL template into a safe statement and ships it verbatim to the NeorunBase connector's JDBC (pg-wire) passthrough. NeorunBase's own pre-Calcite rewriters then see the TVFs intact and execute them server-side. Ontul contributes what it is good at — discovery, governance (IAM + role gating), injection-safe parameterization, and a uniform MCP/REST surface — without getting in the way of the backend dialect.

## Anatomy of a retriever

A retriever definition (`RetrieverDef`) is persisted in the cluster metadata store (key prefix `retriever:`) and replicates to follower masters with the rest of the cluster metadata.

| Field | Purpose |
| --- | --- |
| `catalog`, `schema`, `name` | The retriever's identity in the semantic namespace (its FQN is `catalog.schema.name`). This is *not* where it executes. |
| `targetCatalog` | The Ontul catalog (registered with the `neorunbase` connector) the rendered SQL is pushed down to. This is where execution lands. |
| `kind` | A label for the modality — `HYBRID`, `VECTOR`, `FTS`, `GRAPH`, `PAGERANK`, `PATH_EXISTS`, or `CUSTOM`. Informational; the template is authoritative. |
| `sqlTemplate` | The backend-native SQL with `${param}` placeholders. **Admin-authored and trusted.** |
| `params[]` | The declared parameter contract. Each has `name`, `type`, `required`, `defaultValue`, `description`. Callers may only supply declared names. |
| `outputColumns[]` | Documentation of the columns the rendered query returns. |
| `defaultMaxRows`, `maxRowsCeiling` | Per-retriever row defaults / cap (further bounded by a cluster-wide cap — see [Row caps](#row-caps)). |
| `synonyms[]`, `description` | Natural-language discovery — what an agent matches against in `ontul_describe_retriever` / search. |
| `allowedRoles[]`, `tags[]`, `status`, `certifiedBy`, `certifiedAt`, `owner` | Governance — RBAC gating + the same lifecycle metadata semantic views carry. |

### Parameter types and injection safety

The security model is explicit: **the template is admin-authored and trusted; the args are caller-supplied and untrusted.** The renderer never lets a caller inject raw SQL. Each declared parameter has a `type` that governs validation and how its value is rendered into the statement:

| `type` | Validation | Rendered as |
| --- | --- | --- |
| `STRING` | any text | single-quote-escaped, quoted literal (`'…''…'`) |
| `INT` | parses as a Java `long` | bare number |
| `NUMBER` | parses as a finite `double` | bare number |
| `BOOL` | `true`/`false`/`1`/`0` | `TRUE` / `FALSE` |
| `VECTOR` | matches `[n, n, …]` of numbers | quoted literal `'[…]'` |
| `IDENT` | matches `[A-Za-z0-9_.]+` | bare identifier |

After substitution the renderer asserts two more invariants before anything reaches the engine:

- **No unfilled placeholder** — a `${…}` that has no value (undeclared, or optional with no default and no arg) is rejected.
- **Single statement only** — a `;` *outside* any quoted string literal is rejected. A `;` *inside* an escaped string (e.g. a payload like `x'); DROP TABLE t; --`) stays one harmless literal, while an admin template typo such as `SELECT 1; DELETE …` is caught.

Two context parameters are always bound for row-level scoping, regardless of the declared list: `${user.id}` (the caller's id) and `${user.roles}` (comma-joined roles). Use them in the template's `WHERE` to scope results per caller.

## REST API

All routes are under the master's admin HTTP port. Authentication reuses the same IAM token as the rest of the admin surface (`Authorization: Token <token>`).

| Method + path | Purpose |
| --- | --- |
| `GET /api/v1/retrievers` | List retrievers (IAM-filtered by `data:SelectTable` on the fqn). Optional `?catalog=&schema=`. |
| `POST /api/v1/retrievers` | Register a retriever (admin-only). Validates that `targetCatalog` is a registered `neorunbase` connector. |
| `GET /api/v1/retrievers/{fqn}` | Fetch one retriever's full definition. |
| `DELETE /api/v1/retrievers/{fqn}` | Delete a retriever (admin-only). |
| `GET /api/v1/retrievers/search?q=&limit=` | Natural-language search over name / synonyms / description. |
| `POST /api/v1/retrievers/{fqn}/invoke` | Render with the caller's args and push down to NeorunBase. Body: `{ "args": { … }, "maxRows": N }`. |

### Authorization on invoke

`/invoke` is gated two ways:

1. The IAM `data:SelectTable` check on the retriever's fqn (same as semantic-view visibility).
2. If the definition lists `allowedRoles`, the caller must additionally be a member of at least one listed group.

Only after both pass does the master render the template (binding `${user.id}` / `${user.roles}` from the authenticated caller) and execute the pushdown.

## MCP tools

The [Ontul MCP server](../reference/mcp-server.md) exposes three retriever tools so an agent discovers and runs retrievers without ever writing SQL:

| Tool | Purpose |
| --- | --- |
| `ontul_list_retrievers` | List retrievers (optional `catalog`/`schema` filter), with kind, target, declared params, and governance. |
| `ontul_describe_retriever` | Fetch one retriever's full definition — the param contract the agent must satisfy. |
| `ontul_invoke_retriever` | Run a retriever by fqn with structured `args` (and optional `max_rows`). Returns the rendered SQL plus the result rows. |

The agent flow mirrors the metric flow: discover (`list` / `describe`), then invoke with declared args. Passing an undeclared arg, a type-mismatched value, or omitting a required param is rejected by the server *before any SQL runs*.

## Register and invoke: a HYBRID example

The example registers a hybrid (FTS + vector) document retriever over a NeorunBase `docs` table, with a region/revenue hard filter, then invokes it. `nb` is an Ontul catalog already registered with the `neorunbase` connector.

### 1. Register

```bash
curl -s -X POST "http://localhost:8080/api/v1/retrievers" \
  -H "Authorization: Token $TOKEN" -H "Content-Type: application/json" -d '{
  "catalog": "semantic", "schema": "rag", "name": "docs_hybrid",
  "description": "Hybrid (FTS+vector) document retrieval with a region/revenue hard filter.",
  "kind": "HYBRID", "targetCatalog": "nb",
  "synonyms": ["find documents", "문서 검색", "hybrid search"],
  "sqlTemplate": "SELECT d.id, d.description, h.score FROM HYBRID_SEARCH(table => '\''public.docs'\'', ts_query => ${q}, ts_index => '\''idx_docs_text'\'', vec_query => ${qvec}, vec_index => '\''idx_docs_emb'\'', alpha => 0.4, beta => 0.6, k => ${k}) h JOIN docs d ON d.id = h.id WHERE d.region = ${region} AND d.revenue >= ${minRevenue} ORDER BY h.score DESC LIMIT ${k}",
  "params": [
    {"name": "q",          "type": "STRING", "required": true,  "description": "keyword query"},
    {"name": "qvec",       "type": "VECTOR", "required": true,  "description": "embedding"},
    {"name": "region",     "type": "STRING", "required": false, "defaultValue": "경기도", "description": "region hard filter"},
    {"name": "minRevenue", "type": "INT",    "required": false, "defaultValue": "100",   "description": "min revenue"},
    {"name": "k",          "type": "INT",    "required": false, "defaultValue": "5",     "description": "top-k"}
  ],
  "outputColumns": [{"name": "id"}, {"name": "description"}, {"name": "score"}],
  "status": "CERTIFIED"
}'
# → {"status":"ok","fqn":"semantic.rag.docs_hybrid"}
```

### 2. Invoke

```bash
curl -s -X POST "http://localhost:8080/api/v1/retrievers/semantic.rag.docs_hybrid/invoke" \
  -H "Authorization: Token $TOKEN" -H "Content-Type: application/json" \
  -d '{"args":{"q":"AI","qvec":"[0.9, 0.1, 0.0, 0.0]","k":5},"maxRows":10}'
```

Response (the rendered SQL is echoed so an operator can see exactly what ran):

```json
{
  "fqn": "semantic.rag.docs_hybrid",
  "sql": "SELECT d.id, d.description, h.score FROM HYBRID_SEARCH(table => 'public.docs', ts_query => 'AI', ts_index => 'idx_docs_text', vec_query => '[0.9, 0.1, 0.0, 0.0]', vec_index => 'idx_docs_emb', alpha => 0.4, beta => 0.6, k => 5) h JOIN docs d ON d.id = h.id WHERE d.region = '경기도' AND d.revenue >= 100 ORDER BY h.score DESC LIMIT 5",
  "columns": ["id", "description", "score"],
  "rows": [[1, "AI R&D innovation roadmap", 0.83], [5, "AI artificial intelligence research", 0.79]],
  "rowCount": 2
}
```

The `region` / `minRevenue` defaults apply because the caller omitted them; the `WHERE` hard-filter drops out-of-region and low-revenue rows; the FTS+vector blend ranks the rest. The injected `'AI'` string and `'[0.9, 0.1, 0.0, 0.0]'` vector are rendered as escaped literals — a caller cannot break out of them.

### Same retriever from an agent (MCP)

```text
ontul_describe_retriever({ "fqn": "semantic.rag.docs_hybrid" })
ontul_invoke_retriever({ "fqn": "semantic.rag.docs_hybrid",
                         "args": { "q": "AI", "qvec": "[0.9, 0.1, 0.0, 0.0]", "k": 5 } })
```

## Agentic example: Ontul MCP → semantic metrics + NeorunBase retrieval

This is the payoff of the design: a single agent connected to the [Ontul MCP server](../reference/mcp-server.md) answers a question that needs *both* governed analytics *and* graph-RAG retrieval — without leaving Ontul, and without writing raw SQL against NeorunBase's special functions. Ontul is the one control plane; the semantic metric lives in a [semantic view](semantic-layer.md), the retrieval lives in a retriever backed by NeorunBase.

**User asks the agent:** *"What's our 경기도 net revenue, and which documents back our top AI initiatives there?"*

The agent decomposes this into one analytics call and one retrieval call over MCP:

```text
# 1. Discover the metric by natural language (semantic layer)
ontul_search_metrics({ "query": "매출" })
# → [{ "fqn": "tpch.sales.region_sales", "metricName": "revenue", "score": 100, "matchedOn": "synonym:매출" }]

# 2. Query the metric directly — the engine rewrites `revenue` into its aggregation,
#    injects GROUP BY, and enforces RBAC + mandatory filters server-side.
ontul_query({ "sql": "SELECT region, revenue FROM tpch.sales.region_sales WHERE region = '경기도'" })
# → | region | revenue |
#   | 경기도  | 4820000 |

# 3. Discover the retriever and its param contract (semantic layer, retrieval side)
ontul_describe_retriever({ "fqn": "semantic.rag.docs_hybrid" })
# → kind=HYBRID, targetCatalog=nb, params=[q:STRING*, qvec:VECTOR*, region:STRING=경기도, minRevenue:INT=100, k:INT=5]

# 4. Invoke it — Ontul renders the template into injection-safe SQL and pushes
#    HYBRID_SEARCH (FTS + vector) down to NeorunBase; results come back through Ontul.
ontul_invoke_retriever({
  "fqn": "semantic.rag.docs_hybrid",
  "args": { "q": "AI", "qvec": "[0.9, 0.1, 0.0, 0.0]", "region": "경기도", "k": 5 }
})
# → rows: [[1, "AI R&D innovation roadmap", 0.83],
#          [5, "AI artificial intelligence research", 0.79]]
```

The agent then synthesizes one answer: *"경기도 net revenue is ₩4.82M. The top AI-related documents there are 'AI R&D innovation roadmap' and 'AI artificial intelligence research' (hybrid FTS+vector match)."*

What made this work end-to-end:

- **One token, one surface.** The same `ONTUL_USER_TOKEN` and the same MCP tool namespace cover both the metric and the retrieval; IAM (`data:SelectTable` + `allowedRoles`) is enforced server-side on both.
- **The metric stayed governed.** `revenue` was never hand-formulated by the agent — the semantic layer expanded it, so the number matches every other consumer's.
- **The NeorunBase TVF actually ran.** `HYBRID_SEARCH` (which Ontul's Calcite planner would otherwise reject) executed natively on NeorunBase because the retriever pushed it down verbatim; the agent only supplied typed, escaped args.
- **No raw SQL crossed the agent boundary for retrieval.** The agent filled a declared param contract; the injection-safe renderer produced the SQL.

The end-to-end flow is exercised by `tests/e2e-neorunbase-retriever.sh` in the Ontul repo (register a NeorunBase catalog + a HYBRID retriever, invoke, assert pushdown rows, hard-filter behavior, and injection safety).

## Row caps

Two layers bound the result size, smaller wins:

- **Per-retriever** — `maxRowsCeiling` (default 1000) and `defaultMaxRows` (default 50) in the definition.
- **Cluster-wide** — `ontul.retriever.max.rows.ceiling` (default 1000) in `ontul.properties`, a hard cap an operator can lower globally regardless of any retriever's own ceiling. See [Configuration](../reference/configuration.md).

A caller's requested `maxRows` is clamped to `min(requested, retriever.maxRowsCeiling, cluster.ceiling)`, floored at 1.

## Storage, replication, governance

Retriever definitions live in the cluster metadata store under `retriever:` and ride the standard `exportSnapshot` / `importSnapshot` replication to follower masters — exactly like semantic views. Unlike semantic views they are **not** registered as Calcite views (they are never planned by Ontul's optimizer); they are rendered and pushed down only at invoke time.

Governance mirrors the semantic layer: `status` (`DRAFT` / `CERTIFIED` / `DEPRECATED`), `tags`, `certifiedBy` / `certifiedAt`, `owner`, and `allowedRoles` for invoke-time RBAC.

## See also

- [Semantic Layer](semantic-layer.md) — metrics, dimensions, and query rewriting.
- [MCP Server](../reference/mcp-server.md) — the agent tool surface.
- [Connector Architecture](connector-architecture.md) — how the `neorunbase` connector and pushdown fit in.
- [Configuration](../reference/configuration.md) — `ontul.retriever.max.rows.ceiling`.
