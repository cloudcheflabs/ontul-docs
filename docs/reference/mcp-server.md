# MCP Server

`ontul-mcp` is a [Model Context Protocol](https://modelcontextprotocol.io) server that lets an LLM run SQL against the Ontul Unified Data Engine. It speaks JSON-RPC 2.0 over stdio (the standard MCP transport), connects to Ontul Master via Arrow Flight SQL, and exposes five read-side tools.

Authorization is enforced server-side by Ontul IAM using the access token configured at startup. The MCP layer adds no additional access control — anything the token's policies allow, the LLM can run; anything they deny, the master rejects with an error that the LLM sees verbatim.

The `ontul-mcp` launcher is shipped as part of the Ontul community-edition distribution. After unpacking the release archive it is available at `bin/ontul-mcp` alongside the master / worker / ZooKeeper launchers.

## Tools

| Tool | Equivalent SQL | Purpose |
| --- | --- | --- |
| `ontul_query` | _arbitrary_ | Run any SQL the IAM token is authorized for. Result returned as a markdown table; row count capped by the `max_rows` argument (default 100, hard ceiling 1000). |
| `ontul_list_catalogs` | `SHOW CATALOGS` | List every registered catalog. |
| `ontul_list_schemas` | `SHOW SCHEMAS FROM <catalog>` | List schemas in a catalog. |
| `ontul_list_tables` | `SHOW TABLES FROM <catalog>.<schema>` | List tables in a schema. |
| `ontul_describe_table` | `DESCRIBE <catalog>.<schema>.<table>` | Show columns and Arrow types. |

The four shortcut tools are thin wrappers around `ontul_query` for ergonomics — the LLM does not need to recall Ontul's metadata-command syntax. `ontul_query` itself is the only tool needed to drive any read or write the IAM token allows; the shortcuts merely make catalog discovery cheap.

## Configuration

The server reads the same environment variables the Java SDK already honors so a single token can be reused across SDK clients and the MCP server.

| Env var | Default | Purpose |
| --- | --- | --- |
| `ONTUL_HOST` | `localhost` | Master host — or whichever endpoint your reverse proxy fronts. |
| `ONTUL_PORT` (or `ONTUL_MASTER_FLIGHT_PORT`) | `47470` | Arrow Flight SQL port. |
| `ONTUL_USER_TOKEN` | _unset_ | IAM token attached as the `Authorization` header on every Flight call. **Required.** Without it the master rejects every call. Issue tokens via the Admin UI ("IAM → Access Keys") or `POST /admin/iam/keys`. |
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

Refer to your specific client's MCP integration guide for the exact configuration file path and field names. Once registered the five tools become available and the agent can issue queries like:

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
