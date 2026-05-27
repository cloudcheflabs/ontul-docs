# BI Integration

Ontul speaks Apache Arrow Flight SQL natively, which is the same protocol the modern BI stack — Tableau 2023+, Power BI (via ADBC), Looker, DBeaver, JetBrains DataGrip, Apache Superset — uses for live database connections. Any tool that can mount an Arrow Flight SQL JDBC driver can connect to Ontul without a vendor-specific adapter.

The semantic layer makes that connection useful. Views registered through `/api/v1/semantic-views` appear in BI tools as regular VIEWs whose columns are the **measures** and **dimensions** you defined — the underlying `baseSql` schema is hidden. A dashboard built against `tpch.tiny.lineitem_sales` drags `revenue` and `l_shipdate` onto the canvas and Ontul handles the aggregation, the GROUP BY, the JOIN to conformed dimensions, the row-level filter, and the RBAC check server-side.

```text
   Tableau / Power BI / Looker / DBeaver
                    │
                    │   Arrow Flight SQL (gRPC)
                    ▼
   ┌──────────────────────────────────────┐
   │  Ontul Master (Flight SQL endpoint)  │
   │  - VIEW type tagging                 │
   │  - DESCRIBE → measures + dimensions  │
   │  - rewrite metric refs, inject joins │
   │  - apply mandatory filters / RBAC    │
   └──────────────────────────────────────┘
                    │
                    ▼
   Iceberg  /  NeorunBase  /  JDBC catalogs
```

## What BI tools see

When a BI client introspects the catalog, three things change versus a plain Iceberg / JDBC connection:

| Surface | Behavior |
| --- | --- |
| `CommandGetTables` (Flight SQL) | Every semantic view is tagged `table_type = 'VIEW'`. Plain tables stay `TABLE`. BI tools render the right icon and filter accordingly. |
| `DESCRIBE catalog.schema.view` | Returns the logical column list — metrics first (as `MEASURE`), dimensions next (as `DIMENSION`) — not the raw `baseSql` schema. Tableau uses this to pre-classify fields as measures vs. dimensions on import. |
| `SELECT * FROM view` | Expands to the metric + dimension list with full aggregation and GROUP BY. Tableau's "preview data" step shows the curated result, not the underlying fact rows. |

The Kind discriminator visible from `DESCRIBE`:

```text
| Column     | Type      | Nullable | Kind      |
|------------|-----------|----------|-----------|
| revenue    | AGGREGATE | YES      | MEASURE   |
| units      | AGGREGATE | YES      | MEASURE   |
| l_shipdate | DIMENSION | YES      | DIMENSION |
```

## One-stop connection info

Every Ontul master serves a single self-describing endpoint that BI users hit before configuring a connection. It returns the JDBC URL, the gRPC URL, the driver Maven coordinates / direct-download URL, and per-BI-tool setup hints:

```bash
curl -s http://localhost:8090/api/v1/bi/connection-info
```

```json
{
  "server": "Ontul",
  "version": "1.0.0",
  "arrowFlightSql": {
    "host": "localhost",
    "port": 47470,
    "grpcUrl": "grpc://localhost:47470",
    "jdbcUrl": "jdbc:arrow-flight-sql://localhost:47470",
    "driverArtifact": "org.apache.arrow:flight-sql-jdbc-driver:18.1.0",
    "driverDownloadUrl": "https://repo1.maven.org/maven2/org/apache/arrow/flight-sql-jdbc-driver/18.1.0/flight-sql-jdbc-driver-18.1.0.jar",
    "biTools": {
      "tableau":  "Tableau Desktop 2023.1+ → Other Databases (JDBC) → URL: jdbc:arrow-flight-sql://localhost:47470 …",
      "powerBi":  "Power BI via ADBC driver — install arrow-adbc, then ODBC DSN …",
      "looker":   "Looker LookML 'dialect: arrow_flight_sql' …",
      "dbeaver":  "DBeaver Community → New Connection → Apache Arrow Flight SQL → URL: jdbc:arrow-flight-sql://localhost:47470"
    }
  },
  "adminRest": {
    "baseUrl": "http://localhost:8090",
    "semanticListUrl": "http://localhost:8090/api/v1/semantic-views"
  },
  "auth": {
    "methods": "Bearer JWT, Basic user/password, Access Key AKIA/SK, Token OTOK",
    "notes": "Pass the bearer token in the 'authorization' Flight call header. BI tools that only support user/password should use the Basic auth form."
  }
}
```

The endpoint is **public** — it leaks no secrets and BI users need it _before_ they have credentials. If your cluster sits behind nginx, the returned host / port reflect the externally-routable values (configured via `flight-sql.advertised-host` and `master-flight-sql-port` in the master config).

## Authentication for BI clients

Ontul accepts four credential forms over Flight SQL. Pick whichever the BI tool offers:

| Form | Header / field | Notes |
| --- | --- | --- |
| **Bearer JWT** | `authorization: Bearer <jwt>` | From `POST /admin/auth/login`. Short-lived (15 min default). Best for interactive use. |
| **Basic** | `authorization: Basic base64(user:pass)` | What Tableau / Power BI / Looker offer out-of-the-box. Master logs in on each Flight session. |
| **Access Key** | `authorization: AccessKey AKIA…:<secret>` | Long-lived, suitable for service accounts. Issue via `POST /admin/iam/keys`. |
| **Token** | `authorization: Token OTOK…` | Long-lived bearer token, also from `POST /admin/iam/keys`. |

Whatever IAM policies attach to the credential — column masks, row filters, semantic metric `allowedRoles`, mandatory-filter `${user.attr.X}` substitutions — apply automatically. The BI tool doesn't need (and shouldn't have) a side channel into IAM.

## Tableau Desktop / Server

Tableau 2023.1+ ships the Arrow Flight SQL JDBC driver wiring; older versions need it added manually.

### One-time driver setup

1. Download `flight-sql-jdbc-driver-18.1.0.jar` (URL is returned by `/api/v1/bi/connection-info`).
2. Copy it to Tableau's drivers directory:
    - macOS: `~/Library/Tableau/Drivers/`
    - Windows: `C:\Program Files\Tableau\Drivers\`
    - Linux: `/opt/tableau/tableau_driver/jdbc/`
3. Restart Tableau Desktop.

### Connect

1. Tableau → **More…** → **Other Databases (JDBC)**.
2. URL: `jdbc:arrow-flight-sql://<host>:47470/`
3. Dialect: **Generic SQL**.
4. Username / password: any of the IAM credential forms.
5. Sign in. Tableau introspects the catalog tree; semantic views appear with the VIEW icon.

### What you get

- Drag `revenue` (auto-classified as **measure** because Ontul returned `Kind=MEASURE` in DESCRIBE) and `l_shipdate` onto a viz. Tableau emits `SELECT l_shipdate, revenue FROM tpch.tiny.lineitem_sales`. Ontul rewrites it server-side to the full aggregation + GROUP BY.
- Drag `customer.c_nation` from a conformed-dim view — Ontul auto-injects `LEFT JOIN tpch.tiny.customer customer ON ...` and Tableau sees the joined column without a Tableau-side data-source join.
- Tenant-scoped mandatory filters fire transparently. A finance user in tenant `acme-co` sees `acme-co` rows only; the same dashboard signed in as `globex` shows `globex` rows. No Tableau-side row-level security configuration.

### Sample Tableau-emitted SQL → rewritten

| Tableau emits | Ontul rewrites to |
| --- | --- |
| `SELECT * FROM tpch.tiny.lineitem_sales LIMIT 1000` | `SELECT SUM(...) AS revenue, SUM(...) AS units, l_shipdate FROM ... GROUP BY l_shipdate LIMIT 1000` |
| `SELECT l_shipdate, revenue FROM tpch.tiny.lineitem_sales` | `... SUM(...) AS revenue ... GROUP BY l_shipdate` |
| `SELECT customer.c_nation, revenue FROM tpch.tiny.lineitem_sales` | `... LEFT JOIN tpch.tiny.customer customer ON ls.l_custkey = customer.c_custkey ... GROUP BY customer.c_nation` |

## DBeaver Community

DBeaver Community 23.3+ has a built-in **Apache Arrow Flight SQL** driver entry. No manual jar wrangling.

1. **Database → New Database Connection** → search "Arrow Flight SQL".
2. URL: `jdbc:arrow-flight-sql://<host>:47470/`
3. Test Connection — if the driver isn't downloaded yet DBeaver fetches it.
4. Connect with username / password (Basic) or an IAM-issued token in the URL: `jdbc:arrow-flight-sql://<host>:47470/?token=OTOK…`.

DBeaver's database tree shows catalogs → schemas → tables, with semantic views under the `VIEW` node. Right-click a view → **View Data** issues `SELECT * FROM <view>` which Ontul rewrites to the curated measures + dimensions.

## Power BI

Power BI uses [ADBC](https://arrow.apache.org/adbc) (Arrow Database Connectivity) to consume Arrow Flight SQL. As of Power BI Desktop 2024.6+:

1. Install the [arrow-adbc-flight-sql ODBC driver](https://github.com/apache/arrow-adbc) for your platform.
2. Configure an ODBC DSN with:
    - Driver: ADBC Flight SQL
    - URI: `grpc://<host>:47470`
    - Authentication: Basic (username/password) or `token=OTOK…`
3. Power BI Desktop → **Get Data** → **ODBC** → select the DSN.

Power BI's data model sees semantic-view measures as numeric columns. To make them behave like proper Power BI measures, mark them as such in the model view — Power BI then issues `SELECT <measure> GROUP BY <dim>` queries instead of trying to scan and aggregate client-side. Ontul does the actual aggregation.

## Looker

Looker can consume Arrow Flight SQL via the `arrow_flight_sql` dialect (Looker 23.18+).

In a `connection.yml`:

```yaml
- connection: ontul
  dialect: arrow_flight_sql
  host: ontul-master.internal
  port: 47470
  database: tpch          # default catalog
  username: looker_svc
  password: <password>
  ssl: false
```

LookML `view` files map onto Ontul semantic views. The trick is to declare LookML measures as **passthroughs** since Ontul does the aggregation itself:

```lookml
view: lineitem_sales {
  sql_table_name: tpch.tiny.lineitem_sales ;;

  dimension: ship_date {
    type: date
    sql: ${TABLE}.l_shipdate ;;
  }

  measure: revenue {
    type: number          # NOT type: sum — Ontul aggregates
    sql: ${TABLE}.revenue ;;
  }

  measure: profit_margin {
    type: number
    sql: ${TABLE}.profit_margin ;;
    value_format_name: percent_2
  }
}
```

When a Looker dashboard requests `revenue` by `ship_date`, the emitted SQL is `SELECT l_shipdate, revenue FROM tpch.tiny.lineitem_sales GROUP BY l_shipdate` and Ontul handles the rest server-side — including conformed joins, mandatory filters, and RBAC.

## Apache Superset

Superset 4.0+ has experimental Arrow Flight SQL support. Configure a database with SQLAlchemy URL:

```
adbc+flightsql://<user>:<password>@<host>:47470/
```

Bind the database, browse the catalog tree, and build dashboards. The same semantic surface applies — Superset sees VIEWs with measure / dimension columns.

## JetBrains DataGrip / IntelliJ Database Tools

1. **Database → New → Data Source → Other → Apache Arrow Flight SQL** (DataGrip 2024.1+).
2. URL: `jdbc:arrow-flight-sql://<host>:47470/`
3. User / password.
4. Test connection → introspect.

Useful for power users running ad-hoc SQL while a BI tool drives the production dashboard.

## SQL clients (jdbc-arrow-flight-sql)

For any Java app, the standalone driver is on Maven Central:

```xml
<dependency>
  <groupId>org.apache.arrow</groupId>
  <artifactId>flight-sql-jdbc-driver</artifactId>
  <version>18.1.0</version>
</dependency>
```

```java
String url = "jdbc:arrow-flight-sql://localhost:47470/";
Properties props = new Properties();
props.setProperty("user", "alice");
props.setProperty("password", "secret");

try (Connection conn = DriverManager.getConnection(url, props);
     Statement stmt = conn.createStatement();
     ResultSet rs = stmt.executeQuery(
         "SELECT customer.region, profit_margin FROM saas.core.sales")) {
    while (rs.next()) {
        System.out.println(rs.getString("region") + "\t" + rs.getDouble("profit_margin"));
    }
}
```

The result columns are exactly what the SELECT named — semantic-view column metadata flows through JDBC `ResultSetMetaData` as well.

## Python clients

`adbc_driver_flightsql` is the recommended Python entry point:

```bash
pip install adbc_driver_flightsql adbc_driver_manager pyarrow pandas
```

```python
import adbc_driver_flightsql.dbapi as flightsql

conn = flightsql.connect(
    "grpc://localhost:47470",
    db_kwargs={
        "username": "alice",
        "password": "secret",
    },
)
cur = conn.cursor()
cur.execute("SELECT customer.region, profit_margin FROM saas.core.sales")
df = cur.fetch_arrow_table().to_pandas()
print(df)
```

The query is rewritten server-side just like the BI tool path; the Python client receives one pandas DataFrame with the aggregated results.

## HA and load balancing

When multiple Ontul masters run behind nginx, the Flight SQL upstream must use **sticky load balancing**. A single Flight query is two RPCs — `GetFlightInfo` followed by `DoGet` — and round-robin sends them to different masters. The second master cannot resolve the result handle minted by the first, and the client sees `No results for handle: <uuid>`.

```nginx
upstream ontul_flight {
    hash $remote_addr consistent;
    server ontul-master-1:47470;
    server ontul-master-2:47470;
    keepalive 16;
}

server {
    listen 47470 http2;
    grpc_pass grpc://ontul_flight;
}
```

The admin REST upstream needs the same treatment for stateful flows (catalog registration, semantic view PATCH / certify, IAM mutations). Read-only routes like `/api/v1/bi/connection-info` are stateless and tolerate round-robin, but the simplest pattern is to hash by client there too.

## Troubleshooting

| Symptom | Likely cause / fix |
| --- | --- |
| Tableau "test connection" returns "No results for handle …" | nginx isn't using sticky load balancing — see the previous section. |
| BI tool browses the catalog but a semantic view shows raw `baseSql` columns instead of measures | The BI tool is running `SHOW COLUMNS` against the connector, not Ontul's DESCRIBE. Re-issue using `DESCRIBE <fqn>` from the SQL Runner to verify Ontul returns measures — if it does, the BI tool's introspection is too low-level. Filing an issue with the Kind discriminator usually resolves it. |
| `revenue` is recognized but `customer.c_nation` returns an error | The semantic view doesn't declare a `customer` join, or the join's `target` table is missing. Re-fetch the view via `GET /api/v1/semantic-views/<fqn>` to confirm the `joins[]` array, and check the target exists in the catalog. |
| "Forbidden: vip_revenue" in the BI tool | The IAM user isn't in any of the metric's `allowedRoles`. Add the user to one of the listed groups, or remove the metric from their view. |
| Empty rows for a tenant-scoped view | The `${user.attr.tenant_id}` template resolved to `''` because the IAM user has no `tenant_id` attribute. Set it via the IAM admin UI or `POST /admin/iam/users/<id>/attributes`. |

## See also

- [Semantic Layer](semantic-layer.md) — definition model, query rewriting, governance, RBAC.
- [MCP Server](../reference/mcp-server.md) — same metadata surface, exposed as MCP tools for LLM agents.
- [IAM](iam.md) — user attributes feed the `${user.attr.X}` templating used by mandatory filters.
- [High Availability](high-availability.md) — full nginx config for multi-master clusters.
