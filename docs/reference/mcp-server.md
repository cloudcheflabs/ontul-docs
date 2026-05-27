# MCP Server

`ontul-mcp` is a [Model Context Protocol](https://modelcontextprotocol.io) server that lets an LLM run SQL against the Ontul Unified Data Engine. It speaks JSON-RPC 2.0 over stdio (the standard MCP transport), connects to Ontul Master via Arrow Flight SQL (for queries and catalog introspection) and the admin REST port (for the semantic layer), and exposes eight read-side tools.

Authorization is enforced server-side by Ontul IAM using the access token configured at startup. The MCP layer adds no additional access control — anything the token's policies allow, the LLM can run; anything they deny, the master rejects with an error that the LLM sees verbatim.

The semantic layer rewriting kicks in transparently: an LLM can issue `SELECT revenue, customer.region FROM saas.core.sales` and the master expands the metric, injects the conformed-dimension JOIN, applies the user's mandatory filters, and derives the GROUP BY before execution. The LLM doesn't need to inline the aggregation formula — though it still can, and Phase 1 LLM-expanded SQL keeps working. See [Semantic Layer](../features/semantic-layer.md) for the full rewrite pipeline.

The `ontul-mcp` launcher is shipped as part of the Ontul community-edition distribution. After unpacking the release archive it is available at `bin/ontul-mcp` alongside the master / worker / ZooKeeper launchers.

## Tools

| Tool | Equivalent SQL | Purpose |
| --- | --- | --- |
| `ontul_query` | _arbitrary_ | Run any SQL the IAM token is authorized for. Result returned as a markdown table; row count capped by the `max_rows` argument (default 100, hard ceiling 1000). |
| `ontul_list_catalogs` | `SHOW CATALOGS` | List every registered catalog. |
| `ontul_list_schemas` | `SHOW SCHEMAS FROM <catalog>` | List schemas in a catalog. |
| `ontul_list_tables` | `SHOW TABLES FROM <catalog>.<schema>` | List tables in a schema. |
| `ontul_describe_table` | `DESCRIBE <catalog>.<schema>.<table>` | Show columns and Arrow types. |
| `ontul_list_semantic_views` | `GET /api/v1/semantic-views` | List semantic views (curated business definitions on top of regular SQL views) the IAM token can SELECT from. Optional `catalog` / `schema` filter. See [Semantic Layer](../features/semantic-layer.md). |
| `ontul_describe_semantic_view` | `GET /api/v1/semantic-views/{fqn}` | Fetch one semantic view's full definition — `baseSql`, metrics (with `expr`, `allowedRoles`, `mandatoryFilters`), dimensions, synonyms, description, governance metadata (`status`, `certifiedBy`, `certifiedAt`, `tags`, `owner`), conformed-dimension `joins[]`, view-level `mandatoryFilters`. |
| `ontul_search_metrics` | `GET /api/v1/semantic-metrics/search` | Natural-language metric finder. Matches the query against metric names, synonyms and descriptions; returns ranked `{fqn, metricName, score, matchedOn}` hits. |

The four `_list/_describe_table` tools are thin wrappers around `ontul_query` for ergonomics — the LLM does not need to recall Ontul's metadata-command syntax. `ontul_query` itself is the only tool needed to drive any read or write the IAM token allows; the shortcuts merely make catalog discovery cheap.

The three `_semantic_*` tools talk to the master's admin HTTP port instead of Flight SQL because the semantic CRUD surface is REST. Read-side semantic _consumption_ — the actual `SELECT` against a semantic view — flows through Flight SQL like any other query and benefits from the master-side rewriting. The same `ONTUL_USER_TOKEN` is reused (sent as `Authorization: Token <token>` on REST), and IAM filtering happens server-side just like for the Flight tools. The admin URL is derived from `ONTUL_HOST` (or `ONTUL_ADMIN_URL` to override).

### What `ontul_describe_semantic_view` returns

The full definition is exposed verbatim so the LLM can decide how to use a metric. A representative response shape:

```json
{
  "catalog": "saas",
  "schema":  "core",
  "name":    "sales",
  "fqn":     "saas.core.sales",
  "description": "Per-tenant sales facts — multi-tenant row-scoping enforced.",
  "owner": "admin",
  "status": "CERTIFIED",
  "certifiedBy": "admin",
  "certifiedAt": 1779999999999,
  "tags": ["finance", "gold-tier"],
  "baseSql": "SELECT order_id, tenant_id, customer_id, amount, status, ship_date FROM saas.core.raw_orders",
  "metrics": [
    {"name": "revenue", "expr": "SUM(amount)",
     "synonyms": ["매출", "net_revenue"],
     "mandatoryFilters": ["status = 'COMPLETED'"]},
    {"name": "profit_margin", "expr": "(revenue - cost) / revenue"},
    {"name": "vip_revenue", "expr": "SUM(amount) FILTER (WHERE tier='VIP')",
     "allowedRoles": ["finance", "analyst"]}
  ],
  "dimensions": [{"name": "ship_date"}],
  "joins": [
    {"name": "customer", "target": "saas.core.customer",
     "onTemplate": "${this}.customer_id = ${customer}.id", "type": "LEFT"}
  ],
  "mandatoryFilters": ["tenant_id = ${user.attr.tenant_id}"]
}
```

LLM guidance derived from this payload:

- **Prefer `status: CERTIFIED` over `DRAFT`** when multiple views could answer the question — certified definitions have been reviewed.
- **Avoid metrics where `allowedRoles` excludes the caller**; the rewriter will reject the query with `Forbidden: <metric>` and an unhelpful trace.
- **Expand metrics inline only when the engine rewriter can't help** (e.g. the LLM is generating a query for a non-Ontul engine). For Ontul targets, `SELECT <metric> FROM <fqn>` is enough — the rewriter handles it.
- **Cite tags and certifiedBy in narrative answers** ("revenue is the CERTIFIED measure on saas.core.sales, owned by ‘finance‘") — improves user trust in the LLM's output.

## Configuration

The server reads the same environment variables the Java SDK already honors so a single token can be reused across SDK clients and the MCP server.

| Env var | Default | Purpose |
| --- | --- | --- |
| `ONTUL_HOST` | `localhost` | Master host — or whichever endpoint your reverse proxy fronts. |
| `ONTUL_PORT` (or `ONTUL_MASTER_FLIGHT_PORT`) | `47470` | Arrow Flight SQL port. |
| `ONTUL_ADMIN_URL` | `http://${ONTUL_HOST}:8080` | Admin REST base URL, used only by the three semantic-layer tools. Override when the admin port is fronted by a reverse proxy on a different host or port. |
| `ONTUL_USER_TOKEN` | _unset_ | IAM token attached as the `Authorization` header on every Flight call and as `Authorization: Token <token>` on the admin REST calls. **Required.** Without it the master rejects every call. Issue tokens via the Admin UI ("IAM → Access Keys") or `POST /admin/iam/keys`. |
| `ONTUL_USE_TLS` | `false` | Set to `true` for `grpc+tls` transport. |

Logs are pinned to **stderr** by the bundled logback config. `stdout` is reserved for the MCP JSON-RPC stream — never log there.

## Issuing a token

The MCP server uses Ontul's existing IAM token (one of the three IAM credential forms — Access Key, Secret Key, Token). Issue a token from the Admin UI's IAM page or directly:

```bash
JWT=$(curl -sX POST http://<admin>:8090/admin/auth/login \
    -H 'Content-Type: application/json' \
    -d '{"username":"admin","password":"<your-password>"}' \
    | jq -r .accessToken)

curl -sX POST http://<admin>:8090/admin/iam/keys \
    -H "Authorization: Bearer $JWT" \
    -H 'Content-Type: application/json' \
    -d '{"username":"<the-iam-user>"}'
# → {"accessKeyId":"AKIA...","secretAccessKey":"...","token":"OTOK..."}
```

Use the returned `token` as `ONTUL_USER_TOKEN`. Whatever IAM policies attach to that user — column masks, row filters, catalog allow-lists — apply automatically when the LLM issues queries through the MCP server. No separate authorization layer.

## Wiring into an MCP client

`ontul-mcp` follows the standard MCP stdio convention, so any MCP-aware AI agent or IDE plugin can use it. The integration boils down to telling the client three things:

- the **command** to launch (`/abs/path/to/ontul/bin/ontul-mcp`)
- the **environment variables** to inject (`ONTUL_HOST`, `ONTUL_PORT`, `ONTUL_USER_TOKEN`, optionally `ONTUL_USE_TLS`)
- the **transport**: stdio (no HTTP, no SSE, no socket configuration)

Most clients accept this configuration as a JSON entry under an `mcpServers` (or similarly named) object. A representative shape:

```json
{
  "mcpServers": {
    "ontul": {
      "command": "/abs/path/to/ontul/bin/ontul-mcp",
      "env": {
        "ONTUL_HOST": "localhost",
        "ONTUL_PORT": "47470",
        "ONTUL_USER_TOKEN": "OTOK..."
      }
    }
  }
}
```

Refer to your specific client's MCP integration guide for the exact configuration file path and field names. Once registered the eight tools become available and the agent can issue queries like:

> Show me the top 10 product categories by sales count across `iceberg.warehouse.store_sales` and `iceberg.warehouse.item`.

## Manual smoke test

The protocol is line-delimited JSON-RPC, so a stdio test does not need any client at all:

```bash
export ONTUL_USER_TOKEN=OTOK...
( \
  echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{}}'; \
  echo '{"jsonrpc":"2.0","method":"notifications/initialized"}'; \
  echo '{"jsonrpc":"2.0","id":2,"method":"tools/list"}'; \
  echo '{"jsonrpc":"2.0","id":3,"method":"tools/call",\
"params":{"name":"ontul_list_catalogs"}}'; \
) | bin/ontul-mcp
```

Three responses arrive (the notification produces no reply): the `initialize` result with `serverInfo`, the `tools/list` array, and a markdown rendering of `SHOW CATALOGS`.

## Result formatting

`ontul_query` and the four shortcut tools return a single text payload — a markdown table — so the LLM sees the same column-aligned view a human would in the SQL Runner. NULL cells render as `NULL`, byte-array cells render as a short hex preview followed by the total byte length, and very wide cells are ellipsized at 200 characters. When the result exceeds `max_rows` the table is truncated and a footer line tells the LLM how to widen the cap or tighten the query:

```text
| i_category  | sales_count |
|-------------|-------------|
| Toys        | 4132        |
| Garden      | 3721        |
…
(showing 100 of ≥ 4127 rows; tighten the query or raise max_rows)
```

## Behind a load balancer

When Ontul Master runs HA with multiple instances behind NGINX, the Flight SQL upstream block must use **sticky load balancing** for the MCP server (and for any Flight SQL client). A single Flight query is two RPCs — `GetFlightInfo` followed by `DoGet` — and round-robin sends them to different masters. The second master cannot resolve the result handle minted by the first and the client sees `No results for handle: <uuid>`.

Pin every gRPC connection from one client to one master with `hash $remote_addr consistent;`:

```nginx
upstream ontul_flight {
    hash $remote_addr consistent;
    server ontul-master-1:47470;
    server ontul-master-2:47470;
    keepalive 16;
}
```

For the Admin REST upstream the same caveat applies for stateful flows (catalog registration, IAM mutations) — the simplest pattern is to also hash by client there.
