# Connector Architecture

Ontul is a processing engine, not a storage engine. Data lives in external systems and is accessed through a plugin-based connector architecture. This design separates three distinct concerns: physical connections, table metadata, and data serialization.

## Three-Layer Design

### Connectors (Physical Connections)

Connectors represent physical connections to external systems. Credentials are stored in the encrypted ConnectionStore.

- **S3**: endpoint, access key, secret key, region, path style
- **JDBC**: URL, username, password
- **Kafka**: bootstrap servers, consumer/producer properties
- **NeorunBase**: REST API endpoint + JDBC (pg-wire)

### Catalogs (Table Metadata)

Catalogs manage table metadata and discovery. Each catalog references a Connector by connection ID.

- **Iceberg**: REST catalog (Polaris) — S3 direct access for data, Polaris for metadata
- **NeorunBase**: REST API for table discovery, JDBC (pg-wire) for reads, REST bulk-insert for writes
- **JDBC**: References a JDBC connector — discovers tables from the database schema
- **TPC-H / TPC-DS**: Built-in benchmark data generators (no external connection needed)

### Formats (Data Serialization)

Formats define how data is read from and written to files:

- Parquet, ORC, CSV, JSON, Avro

## Dynamic Catalog Registration

Catalogs can be registered and unregistered at runtime through the Admin API or Admin UI without restarting the cluster.

**Iceberg catalog:**
```json
POST /admin/catalogs
{
  "name": "iceberg",
  "config": {
    "connector": "iceberg",
    "catalog-type": "rest",
    "uri": "http://polaris:8181/api/catalog",
    "warehouse": "my_catalog",
    "s3.endpoint": "http://s3-host:9000",
    "s3.path-style-access": "true",
    "s3.accessKey": "ACCESS_KEY",
    "s3.secretKey": "SECRET_KEY"
  }
}
```

If the REST catalog requires OAuth2 `client_credentials`, add `credential` and `scope` (Admin UI: "OAuth / Polaris auth (advanced)" toggle). Polaris-side table IAM is bypassed — Ontul IAM is the access-control boundary; see [Iceberg Integration](../iceberg/integration.md#access-control).

**NeorunBase catalog:**
```json
POST /admin/catalogs
{
  "name": "neorun",
  "config": {
    "connector": "neorunbase",
    "endpoint": "http://neorunbase-host:8080",
    "jdbcUrl": "jdbc:postgresql://neorunbase-host:5432/neorunbase?preferQueryMode=simple",
    "username": "admin",
    "password": "password"
  }
}
```

Once registered, tables are immediately queryable via fully qualified names: `iceberg.db.my_table`, `neorun.public.events`.

**A registration is checked before anything is stored** (`POST /admin/catalogs`, `PUT /admin/catalogs/<name>`).
Every refusal is `400` with `{"error": "..."}` and leaves no catalog behind (an update leaves the catalog as it was):

| Request | Answer |
|---|---|
| `"type": "ICEBERG_REST"` or `config.connector` not a known type | `unknown catalog type 'ICEBERG_REST' (accepted: elasticsearch, file, iceberg, jdbc, lance, neorunbase, rest-operation, tpcds, tpch)` (plus loaded plugin types) |
| no type at all | `catalog type is required: config.connector (accepted: ...)` |
| `config.connector` and `type` naming different types | `conflicting catalog types: 'iceberg' and type 'jdbc'` |
| a top-level field other than `name`, `catalogName`, `config`, `connector`, `type` | `unknown field 'warehouse' (accepted: catalogName, config, connector, name, type)` |
| a key the connector cannot work without | `config.catalog.rest.uri is required for catalog type 'iceberg'` (jdbc: `url`, elasticsearch: `hosts`, neorunbase: `endpoint` + `jdbcUrl`, rest-operation: `baseUrl` — unless a `connectionId` supplies them; lance: `warehouse`, or `polaris.uri` + `polaris.catalog`) |
| `config.connectionId` of no stored connection | `Connection not found: <id>` |
| an object or array as a config value | `config.<key> must be a string (...)` |
| `config.catalog.type` other than `rest` for Iceberg | `config.catalog.type 'hadoop' is not supported (accepted: rest)` |

The type is matched ignoring case (`"ICEBERG"` is `iceberg`) and stored by its canonical name; there are no other
aliases. The type may be given as `config.connector` or the top-level `type` / `connector`. Earlier releases answered
an unknown or missing type with `201` and created a `file` catalog.

Connections follow the same rules (`POST` / `PUT /admin/connections`): the type is one of `DECIDE, ELASTICSEARCH,
JDBC, KAFKA, NEORUNBASE, REST, RERANK, S3` (case ignored, stored upper-case), otherwise
`unknown connection type 'S4' (accepted: ...)`; only `connectionId`, `type`, `description` and `properties` are
accepted; `properties` values are strings; `connectionId` is required and cannot be changed by an update.

**`POST` creates, `PUT` updates.** `POST /admin/catalogs` only registers a new catalog. A name that is already
registered — compared ignoring case, as SQL resolves names (`ICEBERG` is `iceberg`), the built-in `tpch` / `tpcds`
included — is refused and the existing catalog is left exactly as it was (config, connector, queries):

```
409 {"error": "Catalog 'iceberg' already exists; update it with PUT /admin/catalogs/iceberg"}
```

To change a catalog, send the whole registration to `PUT /admin/catalogs/<name>`: it is checked like a
registration, replaces the stored config and rebuilds the catalog on every master (the workers drop their cached
connector). A secret sent back as the list's mask `***` keeps the stored value; a config that cannot connect is
refused (`400`) and the previous catalog keeps serving. A `PUT` whose body names another existing catalog (a rename
onto it) is `409` the same way. `PUT` of a name that is not registered registers it, so a script that must be
re-runnable and apply edits uses `PUT`. The Admin UI's **Edit catalog** panel sends `PUT`; **Register Catalog** sends
`POST` and shows the `409` text in the panel. Earlier releases replaced the catalog on a `POST` of an existing name.

Connections work the same way: `POST /admin/connections` with a `connectionId` that exists is

```
409 {"error": "Connection 'warehouse-s3' already exists; update it with PUT /admin/connections/warehouse-s3"}
```

and the stored connection is unchanged (earlier releases answered `400 Connection already exists: <id>`). Update it
with `PUT /admin/connections/<id>` (**Edit connection** in the Admin UI).

**An update reloads the catalogs that use the connection.** Every catalog whose config names the `connectionId` is
rebuilt from the updated connection on every master, and every worker drops its connector for it — no **Reload**
needed (earlier releases kept serving the old settings until someone pressed Reload). The answer lists them:

```
200 {"status": "ok", "connectionId": "pg-sales", "updated": true, "reloadedCatalogs": ["pg"], "failedCatalogs": []}
```

If a catalog cannot load with the new settings on some master (a wrong password, an unreachable host), the update
is **refused as a whole**: the previous connection is put back, the catalogs are rebuilt with it everywhere, and
the answer says which catalog failed where and why:

```
422 {"connectionId": "pg-sales", "updated": false,
     "error": "Connection 'pg-sales' was not updated: catalog 'pg' cannot load with it (master-1: FATAL: password authentication failed for user \"flow\", master-2: FATAL: password authentication failed for user \"flow\"). The previous connection is kept; PUT with ?force=true stores it anyway.",
     "failedCatalogs": [{"catalog": "pg", "node": "master-1", "error": "FATAL: password authentication failed for user \"flow\""}, ...],
     "restoredCatalogs": ["pg"]}
```

Rejecting rather than keeping the new settings keeps the cluster consistent: a master whose reload fails keeps
serving the connector it had, while the workers would build theirs from the new, failing settings — two masters
and the workers each on different credentials. When the new settings are right but the database is unreachable
for now (a password rotated during an outage), `PUT /admin/connections/<id>?force=true` stores them anyway and
answers `200` with the failures in `failedCatalogs` and a `warning`; the catalog keeps its connector until a later
reload succeeds. A master that does not answer is listed in `unreachableMasters`; it reads the stored connection
when it is back.

`DELETE /admin/connections/<id>` does not reload anything (that would take the catalogs down at once). Catalogs
that still use the connection keep their loaded connectors until reloaded or restarted, then fail with
`Connection not found: <id>`; the answer names them so they can be re-pointed or dropped:

```
200 {"status": "ok", "catalogsUsingIt": ["pg"], "warning": "catalog(s) pg still use connection 'pg-sales'; they keep working until reloaded or restarted, then fail with \"Connection not found: pg-sales\""}
```

## Built-in Connectors

### Catalog Connectors (Query/Batch)

Registered via `POST /admin/catalogs`. Used in SQL queries and batch pipelines.

| Connector | Read | Write | Description |
|-----------|------|-------|-------------|
| Iceberg | O | O | Open table format, REST catalog (Polaris), S3 direct access |
| NeorunBase | O | O | JDBC pg-wire read, REST bulk-insert write |
| JDBC | O | O | PostgreSQL, MySQL, etc. with HikariCP pooling |
| File | O | O | CSV, Parquet, JSON, Avro, ORC on S3 |
| Elasticsearch | O | O | Query an ES/OpenSearch index as a table and write to an index via `_bulk`; see below |
| Lance | O | O | Vector / multimodal columnar format |
| TPC-H | O | | Built-in benchmark data (configurable scale) |
| TPC-DS | O | | Built-in benchmark data (configurable scale) |

#### Elasticsearch source connector

Register an Elasticsearch (or OpenSearch) cluster as a catalog and query its indices as tables over the normal SQL / SDK path:

```json
{
  "name": "es",
  "properties": {
    "connector": "elasticsearch",
    "hosts": "http://es-host:9200",
    "user": "elastic",
    "password": "…",
    "scrollSize": "1000",
    "scrollTimeout": "1m"
  }
}
```

Once registered, each index (and alias) shows up as a table — e.g. `SELECT * FROM es.default.my_index`. Behaviour:

- **Schema discovery.** The connector reads the index mapping (`GET /{index}/_mapping`) and maps ES field types to Arrow types (`integer/long → Int`, `float/double → FloatingPoint`, `boolean → Bool`, `date → Timestamp`; `text`, `keyword`, `ip`, `geo_point`, nested/object → `Utf8`). The metadata fields `_id`, `_index`, `_score` are also exposed.
- **Predicate pushdown (type-aware).** Because the connector keeps each field's real ES type from the mapping, it pushes exactly the predicates that ES evaluates identically to SQL:
    - Comparisons (`=`, `!=`, `>`, `>=`, `<`, `<=`) on **numeric** fields → `term` / `range`.
    - Equality/inequality on **boolean** fields → `term`.
    - Equality/inequality on **`keyword` / `ip`** (exact) fields → `term`.
    - Comparisons on **`date`** fields → `range` (date-string literals).
    - `col IN (…)` on numeric / `keyword` / `ip` fields → `terms`.

    Predicates on analyzed **`text`** fields (and range comparisons on `keyword`, which would be lexicographic) are **not** pushed — they stay in the engine so pushdown never silently drops matching rows. Anything not pushed is returned as a residual the engine re-applies, so pushdown is always sound.
- **Parallel reads.** A scan is parallelized with the ES sliced-scroll API — one split per slice (`slice.id` / `slice.max`) — so multiple workers scroll disjoint portions of the index concurrently. The slice count is clamped to the index's primary-shard count (slicing beyond shard count is inefficient in ES); a single shard falls back to a plain scroll.
- **Writes (sink).** The connector also implements a sink: `INSERT INTO es.default.my_index SELECT …` (and `CREATE TABLE … AS SELECT`) writes documents via the ES `_bulk` API. When the target index doesn't exist it is created with an **explicit mapping derived from the Arrow schema** (numeric/boolean/date columns keep their ES type instead of relying on ES dynamic mapping; strings map to `keyword`). Set `idField` to the column whose value becomes the document `_id`, giving upsert-by-id (replace) semantics; without it ES auto-generates ids (append). This is the batch write path — streaming jobs can instead use the Elasticsearch streaming sink. So the Elasticsearch connector is a full **source + sink**.

Because the connector is registered as a catalog, the Java/Python SDK can also read and write it by the catalog name — `session.source(Source.sql("SELECT * FROM es.default.my_index"))` to load a DataFrame, and `INSERT INTO es.default.my_index SELECT …` (or a connection-id-referenced sink) to write.

### Streaming Sinks

Used in streaming jobs. Configured via job config, not catalog registration.

| Sink | Transactional | Description |
|------|:---:|-------------|
| Iceberg (table) | O | Parquet write + AppendFiles commit |
| Kafka | O | Producer with optional Kafka Transactions |
| JDBC | O | Batch INSERT with transaction commit |
| NeorunBase | O/X | JDBC mode (tx) or REST mode (at-least-once) |
| Elasticsearch | X | Bulk API with optional upsert |
| HTTP / Webhook | X | JSON POST to REST endpoint |
| Console | X | Debug output to stdout |

## Cross-Engine Federation

Multiple catalogs can be joined in a single query:

```sql
SELECT n.name, c.c_name
FROM neorun.public.events n
JOIN tpch.tiny.customer c ON n.id = c.c_custkey
```

## Plugin Connectors

External connectors can be loaded as plugins via `URLClassLoader` with full dependency isolation. Each plugin JAR is loaded in its own classloader, preventing dependency conflicts between connectors.

## Connection Management

Connections are managed through the Admin UI or REST API:

- Create, update, delete, and list connections
- Credentials are encrypted at rest via KMS (envelope encryption)
- Jobs and catalogs reference connections by ID — no credentials in code or configuration files

See [Connection ID (Source/Sink by Reference)](connection-id.md) for the full reference: registering connections, using IDs from Java/Python SDKs and the REST API, overriding stored properties, and rotation.
