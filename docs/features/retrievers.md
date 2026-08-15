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
| `rerank` | Optional second-stage re-ranking of the rows the template returned — see [Re-ranking](#re-ranking). Absent means the backend's own ordering is final. |
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
  "outputColumns": [{"name": "id"}, {"name": "description"}, {"name": "score"}]
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

## Re-ranking

A retriever's SQL template is a **recall** stage. It can score thousands of rows cheaply because the query and the documents were embedded independently — which is also its limitation: nothing ever compared them directly. Ask "환불 정책은 며칠인가" and every refund-related document lands in roughly the same neighbourhood, because they are all about refunds.

A **cross-encoder** compares them. It takes the query and one document together, in one forward pass, so attention can connect *며칠* in the question to *14일* in the text:

```
"refunds are available within 14 days of purchase"  → 0.94   (answers the question)
"refund reasons explained"                          → 0.31   (about refunds, not about how long)
"refund request contact desk"                       → 0.12
```

That costs a forward pass per row, so it cannot run over the whole corpus — which is exactly why it belongs *after* the template, over the handful of candidates recall already narrowed to.

### Ontul holds the contract, not the model

The `rerank` block never names a model, a runtime or a host. It names a **connection**:

```json
"rerank": {
  "connectionId": "reranker-internal",
  "textColumn": "description",
  "idColumn": "id",
  "queryParam": "q",
  "topN": 5,
  "maxCandidates": 50,
  "maxTextChars": 2000,
  "timeoutMs": 1500,
  "onFailure": "PASSTHROUGH",
  "instruction": "Given a Korean business query, retrieve passages that directly answer it"
}
```

Everything model-shaped — weights, quantisation, GPU, the endpoint's address — lives on that `RERANK` connection. Moving from a 0.6B to a 4B, or from a self-hosted endpoint to a managed one, is an edit on the connection: no retriever definition changes and no Ontul release is involved. The wire contract is deliberately narrow:

```
POST {baseUrl}{path}
{ "query": "...", "documents": ["...", ...], "top_n": 5, "instruction": "...", "model": "..." }
→ { "results": [ { "index": 0, "relevance_score": 0.94 }, ... ] }
```

Response parsing is forgiving on purpose — `relevance_score` or `score`, wrapped in `results`/`data` or a bare array, `index` present or implied by position — because that is the spread of what real endpoints emit. Anything unparseable is a skip with a named reason, never a guess.

!!! note "Dedicated re-rankers only — not generative LLMs"
    Anything that speaks this contract fits: a self-hosted Qwen3-Reranker, Cohere Rerank, Voyage. Generative LLMs do not. They answer in prose (needs parsing, and the parse can fail), take seconds rather than hundreds of milliseconds, and therefore need retries — a loop. A retriever invocation is a single synchronous call, so LLM-based ranking belongs in the agent loop above it. `ontul.rerank.max.timeout.ms` (default 5000) is the mechanical expression of that boundary: a call that needs longer cannot be configured here.

### Fields

| Field | Default | Purpose |
| --- | --- | --- |
| `enabled` | `true` | Off switch that **keeps** the configuration. Toggling this is how you A/B whether re-ranking actually beats the backend's ordering on your data, without deleting and retyping the block. |
| `connectionId` | — | Required. A `ConnectionStore` entry of type `RERANK`. Credentials never appear in the retriever, so a caller who can read the definition still cannot read the endpoint's token. |
| `textColumn` | — | Required. The result column whose text is scored. Resolved against the columns the query **actually returned**, not `outputColumns` (which is documentation and may be stale). |
| `idColumn` | — | Optional. Echoed in the response so an operator can see which rows moved. |
| `queryParam` | first required `STRING` param | Which declared param carries the user's question. Set it explicitly when the template takes more than one string. |
| `topN` | `5` | Rows kept **after** re-ranking. The caller's `maxRows` sizes the *candidate* set; this sizes the answer. |
| `maxCandidates` | `50` | Ceiling on rows sent in one request. Bounds endpoint load and payload size regardless of the `maxRows` a caller asks for. |
| `maxTextChars` | `2000` | Per-document truncation. Cross-encoders take a fixed window (commonly 512 tokens) and silently drop the tail, so **where the cut happens is a ranking-quality decision** — made here by the admin who knows the chunking, rather than implicitly by the model. |
| `instruction` | — | Task instruction sent with the request. Write it in **English even for Korean corpora**: instruction-tuned re-rankers were trained on English instructions and follow them more reliably. |
| `timeoutMs` | `1500` | Per-call budget. This sits inside a synchronous invoke, so it is also the latency a caller pays before a `PASSTHROUGH`. |
| `onFailure` | `PASSTHROUGH` | `PASSTHROUGH` keeps the backend's ordering and flags it. `FAIL` returns **502** — for callers where a wrong ordering is worse than an error. |
| `acknowledgeEgress` | `false` | Required when the connection is marked `external=true`. See [Sending text off-cluster](#sending-text-off-cluster). |

### The response always says what happened

Every invocation carries a `rerank` object, applied or not:

```json
{
  "fqn": "semantic.rag.docs_hybrid",
  "sql": "SELECT d.id, d.description, h.score FROM HYBRID_SEARCH(...)",
  "columns": ["id", "description", "score"],
  "rows": [[4, "refunds are available within 14 days of purchase", 0.71], ...],
  "rowCount": 3,
  "rerank": {
    "applied": true,
    "connectionId": "reranker-internal",
    "model": "Qwen/Qwen3-Reranker-0.6B",
    "candidates": 50,
    "latencyMs": 213,
    "scores": [0.94, 0.31, 0.12],
    "ids": [4, 2, 3]
  }
}
```

When it did not run, `applied` is `false` and `reason` names the cause. This matters more than it looks: **a re-ranker that quietly stopped working is indistinguishable from one that ranks badly.** Without the reason, "results got worse this week" is an unfalsifiable complaint.

| `reason` | Meaning |
| --- | --- |
| `not-configured` | The retriever has no `rerank` block. |
| `disabled` | `rerank.enabled=false` on this retriever. |
| `disabled-cluster-wide` | `ontul.rerank.enabled=false` — the operator's kill switch. |
| `text-column-missing:<name>` | The template no longer returns that column. A **configuration** problem, not an endpoint problem. |
| `connection-missing:<id>`, `connection-type:<T>`, `connection-missing-baseUrl` | The connection was deleted, replaced, or is incomplete. |
| `query-empty` | No query text could be resolved from the args. |
| `timeout`, `http-<code>`, `malformed-response`, `empty-results`, `error:<Type>` | The endpoint let us down. These are the ones `onFailure=FAIL` turns into a 502; misconfigurations above stay a 200 with the reason, because answering 502 for a renamed column sends the operator to the wrong system. |

### Ordering is an authorization property

Re-ranking runs **strictly after** the query. The rows handed to the endpoint have already passed the IAM `data:SelectTable` check, the `allowedRoles` group check, and whatever row-level scoping the template applied through `${user.id}` / `${user.roles}`.

That order is not an implementation detail. Re-ranking first and filtering afterwards would ship rows to an external service that the caller was never entitled to see. Rows the endpoint claims that were never candidates (an out-of-range `index`) are dropped for the same reason.

### Sending text off-cluster

A `RERANK` connection can be marked `external=true`, meaning document text leaves the cluster. Registration then requires `rerank.acknowledgeEgress=true`.

Ontul deliberately does **not** try to decide this for you. A retriever's output columns come from an arbitrary SQL template and carry no link to the `pii` flags on object-type properties — there is nothing to infer from. A check that cannot actually see PII would be theatre; recording a named admin's explicit decision (with `owner` and `updatedAt`) is honest and auditable.

Self-hosting avoids the question entirely: a Qwen3-Reranker behind your own endpoint keeps the text inside the cluster, and the retriever definition is identical either way.

### Cluster-wide limits

In `ontul.properties` — these bound what any single retriever definition may ask of the cluster, and are enforced at **registration** so a bad value is a 400 on the definition rather than a surprise at query time:

| Property | Default | Purpose |
| --- | --- | --- |
| `ontul.rerank.enabled` | `true` | Master kill switch. `false` makes every retriever report `disabled-cluster-wide` — drop re-ranking during an incident without editing retrievers. |
| `ontul.rerank.max.timeout.ms` | `5000` | Ceiling on `rerank.timeoutMs`. Keeps generative LLMs out of the synchronous path. |
| `ontul.rerank.max.candidates` | `200` | Ceiling on `rerank.maxCandidates`. |
| `ontul.rerank.max.text.chars` | `4000` | Ceiling on `rerank.maxTextChars`. |
| `ontul.rerank.http.connect.timeout.ms` | `1000` | TCP connect timeout, separate from the per-call budget: an unreachable host fails fast instead of burning the whole request. |
| `ontul.rerank.min.timeout.ms` | `50` | Floor on `rerank.timeoutMs` — catches a budget that was meant to be seconds. |
| `ontul.rerank.default.path` | `/rerank` | Path appended to a connection's `baseUrl` when the connection sets no `path` of its own. |

### Setting one up

1. **Connections → New**, type **RERANK**. Set `baseUrl` (e.g. `http://reranker:8080`), optionally `path` (default `/rerank`) and `model`, plus auth. Mark `external=true` if the endpoint is outside the cluster.
2. **Retrievers → Edit** the retriever, tick **Re-rank results**, pick the connection, set `textColumn` and `topN`.
3. **Invoke (▶)** to test. The panel shows whether re-ranking applied, the model, the latency, the scores — or the reason it was skipped.
4. Measure before committing: run with `enabled: false`, then `true`, and compare. If the template already does structural re-ranking (PPR, graph paths), the additional gain may be small — the contract is simple enough that adding it later costs nothing.

The end-to-end behaviour is exercised by `tests/e2e-retriever-rerank.sh` in the Ontul repo: a mock endpoint with a deterministic ordering rule proves the rows follow the endpoint rather than the backend, and every failure mode — timeout, 5xx, garbage body, out-of-range index, endpoint stopped — is asserted to degrade into a flagged pass-through.

### Worked example, end to end

**1. Register the endpoint as a connection.** Everything model-shaped stops here.

```bash
curl -X POST http://ontul-master:8080/admin/connections \
  -H "Authorization: Bearer $ADMIN_TOKEN" -H 'Content-Type: application/json' -d '{
    "connectionId": "reranker-internal",
    "type": "RERANK",
    "description": "Qwen3-Reranker-0.6B behind TEI, in-cluster",
    "properties": {
      "baseUrl": "http://reranker.internal:8080",
      "path": "/rerank",
      "model": "Qwen/Qwen3-Reranker-0.6B",
      "authType": "NONE",
      "external": "false"
    }
  }'
```

Name the secret `apiKey` (or `secret`) rather than `token` when the endpoint needs
one — the connections list masks keys containing *secret* / *password* / *key*, and
a property literally named `token` is returned in the clear.

**2. Register the retriever.** `maxRows` at invoke time sizes the candidate set;
`topN` sizes the answer.

```bash
curl -X POST http://ontul-master:8080/api/v1/retrievers \
  -H "Authorization: Token $TOKEN" -H 'Content-Type: application/json' -d '{
    "catalog": "semantic", "schema": "rag", "name": "docs_hybrid",
    "kind": "HYBRID", "targetCatalog": "nb",
    "sqlTemplate": "SELECT d.id, d.description, h.score FROM HYBRID_SEARCH(table => '"'"'public.docs'"'"', ts_query => ${q}, ts_index => '"'"'idx_docs_text'"'"', vec_query => ${qvec}, vec_index => '"'"'idx_docs_emb'"'"', alpha => 0.4, beta => 0.6, k => ${k}) h JOIN docs d ON d.id = h.id ORDER BY h.score DESC LIMIT ${k}",
    "params": [
      {"name": "q",    "type": "STRING", "required": true},
      {"name": "qvec", "type": "VECTOR", "required": true},
      {"name": "k",    "type": "INT",    "required": false, "defaultValue": "50"}
    ],
    "outputColumns": [{"name": "id"}, {"name": "description"}, {"name": "score"}],
    "rerank": {
      "enabled": true,
      "connectionId": "reranker-internal",
      "textColumn": "description",
      "idColumn": "id",
      "queryParam": "q",
      "topN": 5,
      "maxCandidates": 50,
      "maxTextChars": 2000,
      "timeoutMs": 1500,
      "onFailure": "PASSTHROUGH",
      "instruction": "Given a Korean business query, retrieve passages that directly answer it"
    }
  }'
```

**3. Invoke it.**

```bash
curl -X POST http://ontul-master:8080/api/v1/retrievers/semantic.rag.docs_hybrid/invoke \
  -H "Authorization: Token $TOKEN" -H 'Content-Type: application/json' \
  -d '{"args": {"q": "환불 정책은 며칠인가", "qvec": "[0.9, 0.1, 0.0, 0.0]", "k": 50}, "maxRows": 50}'
```

50 rows are recalled and sent to the endpoint; 5 come back:

```json
{
  "rowCount": 5,
  "columns": ["d.id", "d.description", "h.score"],
  "rows": [
    [4, "구매 후 14일 이내 환불 가능합니다", 0.71],
    [2, "환불 사유는 다음과 같습니다", 0.83],
    [3, "환불 담당 부서 연락처", 0.80]
  ],
  "rerank": {
    "applied": true,
    "connectionId": "reranker-internal",
    "model": "Qwen/Qwen3-Reranker-0.6B",
    "candidates": 50,
    "latencyMs": 213,
    "scores": [0.94, 0.31, 0.12],
    "ids": [4, 2, 3]
  }
}
```

Note row 4 is now first even though its hybrid score (`0.71`) is the *lowest* of the
three. That inversion is the whole point: hybrid ranked by topical similarity, the
cross-encoder ranked by whether the passage answers *this* question.

**4. What the endpoint received.**

```json
{
  "query": "환불 정책은 며칠인가",
  "documents": ["환불 사유는 다음과 같습니다", "환불 담당 부서 연락처", "..."],
  "top_n": 5,
  "instruction": "Given a Korean business query, retrieve passages that directly answer it",
  "model": "Qwen/Qwen3-Reranker-0.6B"
}
```

Documents are truncated to `maxTextChars` **before** they leave Ontul, and only the
`textColumn` is sent — not the whole row.

### When the endpoint is down

Same request, re-ranker stopped. The answer still arrives:

```json
{
  "rowCount": 5,
  "rows": [[2, "환불 사유는 다음과 같습니다", 0.83], "..."],
  "rerank": { "applied": false, "reason": "timeout", "latencyMs": 1502,
              "connectionId": "reranker-internal" }
}
```

Rows are back in the backend's order and `reason` says why. Flip `onFailure` to
`FAIL` and the same outage returns **502** instead — pick per retriever, not
globally.

### An external endpoint

```json
"properties": {
  "baseUrl": "https://api.cohere.com/v2",
  "path": "/rerank",
  "model": "rerank-v3.5",
  "authType": "BEARER",
  "apiKey": "…",
  "external": "true"
}
```

With `external=true`, registering a retriever against it fails until the admin adds
`"acknowledgeEgress": true` to the `rerank` block:

```
connection 'cohere-rerank' is marked external=true: document text from
'description' would leave the cluster. Set rerank.acknowledgeEgress=true to
accept that explicitly
```

### Measuring whether it is worth it

The `enabled` flag exists so this is a one-field edit rather than a rewrite:

```bash
# A: recall only
curl -X POST .../api/v1/retrievers -d '{... "rerank": {..., "enabled": false}}'
# run your evaluation set, record hit@5

# B: with re-ranking
curl -X POST .../api/v1/retrievers -d '{... "rerank": {..., "enabled": true}}'
# same evaluation set, compare
```

Worth doing before committing to the latency. If the template already re-ranks
structurally — PPR, graph paths, a well-tuned `alpha`/`beta` — the additional gain
can be small, and `rerank.applied` in each response tells you which arm produced
any given result.

### What an agent sees

`ontul_describe_retriever` surfaces the `rerank` block, and `ontul_invoke_retriever`
passes the `rerank` object through. Both tool descriptions tell the agent not to
re-order rows that came back with `applied: true` — a second opinion from a
generative model would be worse than the first, since the cross-encoder compared
each row against the query directly.

## Row caps

Two layers bound the result size, smaller wins:

- **Per-retriever** — `maxRowsCeiling` (default 1000) and `defaultMaxRows` (default 50) in the definition.
- **Cluster-wide** — `ontul.retriever.max.rows.ceiling` (default 1000) in `ontul.properties`, a hard cap an operator can lower globally regardless of any retriever's own ceiling. See [Configuration](../reference/configuration.md).

A caller's requested `maxRows` is clamped to `min(requested, retriever.maxRowsCeiling, cluster.ceiling)`, floored at 1.

## Configuring in the Admin UI

Retrievers can be managed without REST calls under **Semantic & AI → Retrievers** in the Admin UI:

- **Register Retriever** opens a side panel for `catalog` / `schema` / `name`, `kind`, the **target catalog** (a `neorunbase` catalog), the SQL template, repeatable **parameter** rows (name, type, required, default, description), an optional **Re-rank** block, synonyms, and `allowedRoles`. Certification is not part of the
form — it is a separate, attributed action (see Governance below).
- The **Edit** (✏️) button reopens an existing retriever in the same panel. `catalog` / `schema` / `name` are its identity and are locked there — saving is an upsert on that FQN, so letting them change would create a second retriever instead of updating this one. Editing is what makes the `rerank.enabled` A/B a one-field change.
- Each retriever card shows its kind badge, target catalog, a shield icon when role-gated, a **RERANK top N** badge when re-ranking is configured (greyed out when it is configured but disabled), and its declared params (`name:type*`).
- The **Invoke** (▶) button opens a tester: fill the declared `args` as JSON, run it, and the panel shows the **rendered SQL** that was pushed to NeorunBase, whether re-ranking applied (with model, latency and scores — or the reason it was skipped), plus the returned rows — the same `POST /api/v1/retrievers/{fqn}/invoke` an agent calls. It's the fastest way to validate a template + param contract before agents use it.

Metric / retriever `allowedRoles` map to IAM groups — see [Semantic Layer & Retriever RBAC](iam.md#semantic-layer-retriever-rbac).

## Storage, replication, governance

Retriever definitions live in the cluster metadata store under `retriever:` and ride the standard `exportSnapshot` / `importSnapshot` replication to follower masters — exactly like semantic views. Unlike semantic views they are **not** registered as Calcite views (they are never planned by Ontul's optimizer); they are rendered and pushed down only at invoke time.

Governance mirrors the semantic layer: `status` (`DRAFT` / `CERTIFIED` / `DEPRECATED`), `tags`,
`certifiedBy` / `certifiedAt`, `owner`, and `allowedRoles` for invoke-time RBAC.

`status`, `certifiedBy`, `certifiedAt` and the certification fingerprint are **server-owned**: a
register/update call cannot set them (a payload that tries is ignored), so nothing can declare
itself certified. Certify through the dedicated endpoint instead, which stamps the **caller** as
the certifier and is audited:

```bash
curl -s -X POST "http://localhost:8080/api/v1/retrievers/semantic.rag.docs_hybrid/certify" \
  -H "Authorization: Bearer $TOKEN" -d '{}'
# → { …, "status": "CERTIFIED", "certifiedBy": "alice", "effectiveStatus": "CERTIFIED" }

# Undo (back to DRAFT), or retire it:
curl -s -X POST "http://localhost:8080/api/v1/retrievers/semantic.rag.docs_hybrid/decertify" \
  -H "Authorization: Bearer $TOKEN" -d '{"status":"DEPRECATED"}'
```

Certifying also records a fingerprint of what the retriever *does* — its `kind`, target catalog,
SQL template and parameter contract. Edit any of those afterwards and reads report
`effectiveStatus: "STALE"` with a `certificationNote` explaining why; editing the description or
synonyms does not. **Read `effectiveStatus`, not `status`.** Certification requires the
`ontology:Certify` IAM action (admins always may), which is deliberately separate from the rights
needed to edit a retriever. See
[Ontology → Certification](ontology.md#certification-trust-an-agent-can-act-on).

## See also

- [Semantic Layer](semantic-layer.md) — metrics, dimensions, and query rewriting.
- [MCP Server](../reference/mcp-server.md) — the agent tool surface.
- [Connector Architecture](connector-architecture.md) — how the `neorunbase` connector and pushdown fit in.
- [Configuration](../reference/configuration.md) — `ontul.retriever.max.rows.ceiling`, `ontul.rerank.*`.
