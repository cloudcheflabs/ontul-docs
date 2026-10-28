# Iceberg Integration

Ontul integrates with Apache Iceberg as a first-class catalog. The integration is **format-version 2** native — every write is a v2 snapshot — and includes the full operator surface that production tables need: hidden partitioning, position-delete DELETE, RowDelta UPDATE/MERGE, schema evolution with safe-type promotion, partition evolution, snapshot rollback, branches and tags, time travel, and Iceberg-spec views.

The data path is implemented in Ontul. The Apache Iceberg JAR is used only as a metadata library (catalog client, manifest schema, snapshot bookkeeping). Parquet writes, parquet reads with Iceberg field-id resolution, partition transforms, partition path encoding, predicate evaluation against position deletes, and snapshot commits are all Ontul code. This is what lets Ontul distribute every write and read across workers and bypass Polaris-side credential vending entirely.

> **In this section:** this page covers catalog setup and SQL (DDL/DML, formats, time travel, views, federation, access control). See also **[Table Maintenance](maintenance.md)** (compaction, snapshot expiry, orphan cleanup, rollback — on-demand and scheduled) and **[Write-Audit-Publish (WAP)](wap.md)** (branch-staged writes with audit before publish).

## Catalog Type

Ontul supports the Iceberg **REST catalog** only. Two REST catalog flavors are supported, selected with `catalog.rest.flavor`: **Apache Polaris** (OAuth2 client-credentials, the default) and the **AWS Glue Iceberg REST endpoint** (AWS SigV4 request signing). Storage is S3-compatible (ShannonStore, AWS S3, MinIO). Multiple Iceberg catalogs can be registered side-by-side, each with its own REST endpoint and credentials.

```json
POST /admin/catalogs
{
  "name": "ice",
  "config": {
    "connector": "iceberg",
    "catalog.type": "rest",
    "catalog.rest.uri": "http://polaris:8181/api/catalog",
    "catalog.name": "my_catalog",
    "catalog.warehouse": "my_catalog",
    "catalog.rest.flavor": "polaris",
    "catalog.rest.client_id": "root",
    "catalog.rest.client_secret": "s3cr3t",
    "catalog.rest.scope": "PRINCIPAL_ROLE:ALL",
    "s3.accessKey": "ACCESS_KEY",
    "s3.secretKey": "SECRET_KEY",
    "s3.endpoint": "http://nginx:80",
    "s3.pathStyle": "true",
    "s3.region": "us-east-1"
  }
}
```

`catalog.rest.flavor: polaris` switches on Polaris-specific defaults (default scope `PRINCIPAL_ROLE:ALL`, OAuth2 client-credentials grant). Other REST catalogs work without it.

### AWS Glue REST catalog

AWS Glue exposes an Iceberg REST catalog at `https://glue.<region>.amazonaws.com/iceberg`. Because it is a standard Iceberg REST catalog, Ontul reaches it through the same `RESTCatalog` path as Polaris — only the authentication differs: Glue requires **AWS SigV4** request signing instead of OAuth2. Set `catalog.rest.flavor: glue` (or `catalog.rest.auth: sigv4` on any REST endpoint) to switch the catalog client to SigV4 signing.

```json
POST /admin/catalogs
{
  "name": "glue_cat",
  "config": {
    "connector": "iceberg",
    "catalog.type": "rest",
    "catalog.rest.flavor": "glue",
    "catalog.rest.uri": "https://glue.us-east-1.amazonaws.com/iceberg",
    "catalog.warehouse": "123456789012",
    "catalog.rest.signing.region": "us-east-1",
    "catalog.rest.aws.accessKey": "AKIA...",
    "catalog.rest.aws.secretKey": "SECRET",
    "s3.accessKey": "AKIA...",
    "s3.secretKey": "SECRET",
    "s3.region": "us-east-1"
  }
}
```

Glue-specific keys:

| Key | Meaning |
| --- | --- |
| `catalog.warehouse` | The AWS account id (or `<account-id>:s3tablescatalog/<bucket>` for [S3 Tables](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables.html)). |
| `catalog.rest.uri` | `https://glue.<region>.amazonaws.com/iceberg`. |
| `catalog.rest.signing.region` | SigV4 signing region. Falls back to `s3.region`, then to `ontul.iceberg.default.region`. |
| `catalog.rest.signing.name` | SigV4 service name. Defaults to `ontul.iceberg.glue.signing.name` (`glue`); use `s3tables` for the S3 Tables endpoint. |
| `catalog.rest.aws.accessKey` / `.secretKey` / `.sessionToken` | AWS credentials used to sign catalog requests. **Optional** — when absent they fall back to the `s3.*` keys, then to the default AWS provider chain (instance profile / environment / SSO), which is the usual case under an IAM role. |

As with Polaris, table data is read and written directly with the `s3.*` credentials; Glue's Lake Formation credential vending is bypassed (`ontul.iceberg.rest.credential.vending.bypass`, default `true`) so Ontul IAM remains the authoritative policy boundary. See [Configuration](../reference/configuration.md) for the server-wide Iceberg catalog defaults.

The S3 credential keys accept either the `s3.`-prefixed spelling (`s3.accessKey`, `s3.secretKey`, `s3.endpoint`, `s3.region`, `s3.pathStyle`) or the bare spelling (`accessKey`, `secretKey`, …) used by [Connection IDs](../features/connection-id.md), so a catalog can be backed by a stored S3 connection (`"connectionId": "..."`) without renaming keys.

### Lookup-on-miss for externally created tables

A table created in a shared REST catalog by **another engine** (for example Trino) is picked up automatically: when a query references `catalog.ns.table` and the planner does not yet have it cached, Ontul refreshes that catalog and re-plans once before failing. No manual `POST /admin/catalogs/<name>/refresh` is needed for the new table to become queryable. (The refresh endpoint still exists for forcing a full re-list on demand.)

### Catalog availability

Each master builds its own connector for every catalog. If the REST catalog cannot be reached when a master starts or becomes leader (for example, Polaris is up but its database is not), that catalog is **unavailable** on that master:

- A query that names it fails. It does not return an empty result:

    ```json
    {"status": "error",
     "error": "Catalog 'iceberg' is unavailable: Unable to process (code: 500) ... Connection to db:5432 refused (since 2026-10-20T10:09:20Z, 3 attempt(s); retried every 30s, POST /admin/catalogs/iceberg/reload retries now)"}
    ```

- `GET /admin/catalogs` reports each catalog's state, as seen by every master. `available` is `true` only when every master that answered has the catalog loaded, so it can be used in a health check:

    ```json
    [{"name": "iceberg", "connectorType": "iceberg", "available": false,
      "unavailableReason": "Unable to process (code: 500) ... Connection to db:5432 refused",
      "unavailableSince": "2026-10-20T10:09:20Z",
      "masters": [{"node": "master-1", "available": false, "reason": "...", "attempts": 3, "lastAttemptAt": "..."},
                  {"node": "master-2", "available": false, "reason": "...", "attempts": 3, "lastAttemptAt": "..."}]}]
    ```

- Recovery does not need a restart:
    - Every master retries the catalog every `ontul.catalog.reload.retry.seconds` (default 30).
    - A query that names the catalog starts a retry in the background (at most once every 5 seconds). The query itself fails at once and does not wait for it.
    - To retry at once on every master, call `POST /admin/catalogs/<name>/reload`, or `POST /admin/catalogs/reload` for every unavailable catalog. Either call needs an administrator. It answers `200` when the catalog is loaded on every master, `503` (with each master's reason) when it is not, and `404` for a catalog that is not registered.
    - `POST /admin/catalogs/<name>/refresh` on an unavailable catalog does the same as `reload`.

A reload of a catalog that is still serving keeps the current connector if the new one cannot be built.

### Polaris catalog: `stsUnavailable: true` is required

When the Polaris catalog itself is created (a one-time admin step on the Polaris side), its `storageConfigInfo` **must** include `stsUnavailable: true`:

```json
{
  "catalog": {
    "name": "my_catalog",
    "type": "INTERNAL",
    "storageConfigInfo": {
      "storageType": "S3",
      "endpoint": "http://localhost:8000",
      "endpointInternal": "http://nginx:80",
      "pathStyleAccess": true,
      "stsUnavailable": true,
      "allowedLocations": ["s3://my-warehouse/"]
    }
  }
}
```

Without `stsUnavailable`, Polaris transparently calls AssumeRole on its bootstrap credentials and hands the resulting `ASIA*`-prefixed temporary key to the Iceberg client. ShannonStore and other S3-compatible servers that don't share an STS endpoint with Polaris cannot validate those temporary keys, and every S3 request returns `401 Forbidden: The security token included in the request is invalid`. The `POLARIS_FEATURES_SKIP_CREDENTIAL_SUBSCOPING_INDIRECTION` environment flag only suppresses the second AssumeRole; the first still happens unless the catalog itself opts out via `stsUnavailable: true`.

### No vended credentials

Ontul sends `header.X-Iceberg-Access-Delegation: ""` on every catalog call so Polaris does not vend credentials, and additionally **reflectively replaces the `io` field of `RESTTableOperations`** with a `S3FileIO` wired to the static `s3.accessKey` / `s3.secretKey` from the catalog config. This second step is required because Iceberg's commit path writes the new `metadata.json` through the operations' internal `io` field directly, not via the public `io()` accessor a wrapper can override. With both layers in place, every read and write — including the metadata write that lands a new snapshot — uses Ontul-managed credentials. Polaris-side table IAM is intentionally bypassed; Ontul IAM is the authoritative policy boundary across catalogs/jobs/sources/sinks.

## DDL

| Statement | Notes |
|---|---|
| `CREATE SCHEMA [IF NOT EXISTS] catalog.namespace` | Creates an Iceberg namespace. An existing one is an error (`Schema 'c.ns' already exists`) unless `IF NOT EXISTS`. |
| `CREATE TABLE [IF NOT EXISTS] catalog.ns.t (cols) [USING iceberg] [PARTITIONED BY (...)] [SORT BY (...)] [WITH (props) \| TBLPROPERTIES (props)]` | Format v2, with optional hidden partitioning and sort order. `USING iceberg` (Spark style) is accepted and ignored; `WITH` and `TBLPROPERTIES` are interchangeable property clauses. An existing table is an error (`Table 'c.ns.t' already exists`) unless `IF NOT EXISTS`, which leaves it unchanged. |
| `CREATE TABLE [IF NOT EXISTS] catalog.ns.t [PARTITIONED BY (...)] AS SELECT ...` | CTAS, distributed across workers. On an existing table, plain CTAS **appends** the rows (unlike Trino, which refuses it) and `IF NOT EXISTS` is a no-op answering `CREATE TABLE 0 (table catalog.ns.t already exists; nothing written)` — see [Missing objects and failures](#missing-objects-and-failures). |
| `DROP TABLE [IF EXISTS] catalog.ns.t` | Honors Polaris `polaris.config.drop-with-purge.enabled` for actual S3 cleanup. A missing table is an error unless `IF EXISTS`; see [Missing objects and failures](#missing-objects-and-failures). |
| `DROP SCHEMA [IF EXISTS] catalog.namespace` | Namespace must be empty (`Cannot drop non-empty schema 'c.ns'`, with or without `IF EXISTS`). |
| `CREATE [OR REPLACE] VIEW catalog.ns.v AS <SELECT>` | Persisted via Iceberg's view spec; see [Views](#views). |
| `DROP VIEW [IF EXISTS] catalog.ns.v` | Stops resolving on every master at once; see [Views](#views). A missing view is an error unless `IF EXISTS`. |
| `COMMENT ON TABLE catalog.ns.t IS '...'` | |
| `COMMENT ON COLUMN catalog.ns.t.col IS '...'` | |

### Hidden partitioning

`PARTITIONED BY (...)` accepts the standard Iceberg transforms — `identity`, `year(col)`, `month(col)`, `day(col)`, `hour(col)`, `bucket(N, col)`, `truncate(L, col)`, `void`. The transforms and partition-path encoding are implemented in Ontul (Murmur3_32 for `bucket`, ISO date math for the temporal transforms), so writes do not need a Spark/Trino runtime and the same logic runs identically on master and worker. Partition sources of type `date` and `timestamp` (identity, or the temporal transforms) are written with Iceberg's own partition values — an `INSERT` into a table `PARTITIONED BY (dt)` on a `DATE` column lands one file per day under `dt=YYYY-MM-DD/`.

Column types in `CREATE TABLE`: the primitive types, `DECIMAL(p, s)`, `TIMESTAMP [WITH TIME ZONE]` and `ARRAY<...>` are created as the matching Iceberg types; a CTAS of an `ARRAY[...]` expression creates an Iceberg `list` column. `MAP<...>`, `ROW(...)` and `STRUCT<...>` columns are refused with a message, because they cannot be written yet — they used to become a `VARCHAR` column without a word.

```sql
CREATE TABLE ice.events.events (
  id BIGINT, ts TIMESTAMP, region STRING, amount INTEGER
)
PARTITIONED BY (day(ts), bucket(8, region));
```

Predicates against the source columns are pushed into the Iceberg scan and used for partition pruning before the worker even opens a parquet file.

#### Distributed-write partition round-trip

Every Ontul write is distributed: each worker writes its own data files and ships per-file
metadata — path, size, record count, and **partition value** — to the controller, which performs
one atomic `AppendFiles` (or `RowDelta`) commit. Only the controller commits a given table, so
no concurrent-writer `CommitFailedException` can occur regardless of parallelism.

The partition value travels as a path string (`name=value/...`) that the controller re-parses
with Iceberg's `Conversions.fromPartitionString(resultType, s)`. The encoding is therefore
**result-type dependent**, not a single human-readable form:

| Transform | Partition result type | Serialized form |
|---|---|---|
| `year`, `month`, `hour`, `bucket(N,…)`, `truncate(L,int)` | `int` | the raw ordinal value (e.g. `hour → 494822`) — `Integer.parseInt` |
| `day`, identity on a `date` column | `date` | ISO date string (e.g. `2024-03-15`) — `LocalDate.parse` |
| identity / truncate on string | `string` | the value itself |

Using the wrong form breaks the commit silently per transform: the human form
(`hour → "2026-06-13-22"`) fails `Integer.parseInt`, while the raw epoch-day int for `day` fails
`LocalDate.parse`. Ontul encodes each field by its result type so `year/month/day/hour/bucket/
truncate/identity` all round-trip correctly through the distributed commit, on both the
`INSERT`/`CTAS` data-file path and the `MERGE`/`DELETE` delete-file path.

### Sort order

`SORT BY (col [ASC|DESC] [NULLS FIRST|LAST], ...)` enforces a per-file sort. Rows are buffered per partition and sorted before the data file is flushed — useful for clustering on filter columns to improve later column-filter predicate pushdown.

## File formats

Iceberg data files can be **Parquet, ORC, or Avro**, and Ontul both reads and writes all three. A single table may even hold a mix of formats written by different engines.

**Write.** The per-table `write.format.default` property chooses the data-file format for every write path — `INSERT`, CTAS, `MERGE`, `UPDATE`, and `OPTIMIZE` compaction. Parquet is the default when the property is absent.

```sql
CREATE TABLE ice.ns.events (id BIGINT, payload STRING) USING iceberg
TBLPROPERTIES ('format-version'='2', 'write.format.default'='orc');
```

**Read.** Ontul reads Parquet, ORC and Avro data files regardless of which engine produced them, so a table created by Trino, Spark or Flink in any of these formats is queryable directly. The format is detected **per data file** from the Iceberg manifest — not from the table default — so reads work transparently across a format change or a mixed-format table.

**Cross-engine interoperability.** Files Ontul writes are readable by other Iceberg engines and vice-versa (verified end-to-end against Trino on a Polaris catalog). Ontul records the true on-disk `file_size_in_bytes` on every data file, so engines that locate the Parquet footer from Iceberg metadata read it correctly; ORC/Avro files written elsewhere are read through Iceberg's generic readers with field-id projection.

**Row-level DML across formats.** `DELETE`, `UPDATE`, and `MERGE` work on tables whose data files are Parquet, ORC, or Avro: the handlers scan the matched data file in its own format to find the rows, then write new data files in the table's `write.format.default` and the deletes. Reads apply deletes to any data-file format too — a Parquet, ORC, or Avro data file with position deletes, equality deletes, or a deletion vector attached is read with those deletes applied, including delete files another engine wrote in ORC or Avro (the delete reader dispatches on the delete file's own format).

!!! note "Delete files Ontul writes are always Parquet"
    Ontul writes its own equality- and position-**delete** files as Parquet (independent of `write.format.default`), and its v3 deletion vectors as Puffin. Other engines (e.g. Trino) write delete files in the table's data format — Ontul reads those in whatever format they are.

## Read

- Standard SQL with fully qualified names (`catalog.ns.table`).
- Snapshot isolation — every query reads a single Iceberg snapshot.
- Position-delete files and equality-delete files are applied at read time. The fast-path `SELECT count(*)` shortcut is automatically suppressed when a snapshot has any deletes; the count goes through a real scan so the answer is correct.
- **Field-ID-based parquet reads.** Columns are resolved by Iceberg field id, not name, so files written before an `ALTER TABLE RENAME COLUMN` or `SET DATA TYPE` continue to read correctly. Type-widening (`INT32 → INT64`, `FLOAT → DOUBLE`) is performed on the fly when an older file's primitive is narrower than the current schema's column type.
- **Types are Iceberg's types.** `date`, `time`, `timestamp`, `timestamptz` and `decimal(p, s)` are read and written as those types, not as epoch numbers or doubles: a table Ontul writes reads in Trino as `date` / `timestamp(6)` / `decimal(12,2)`, and the reverse. See [Data Types](../features/data-types.md).
- **`DECIMAL` in any physical encoding.** Parquet stores a decimal as `INT32`, `INT64` or unscaled bytes (`fixed_len_byte_array` / `binary`) depending on the writer — Trino writes every decimal as bytes. Ontul reads all three at the column's scale, as an exact `DECIMAL`.
- Distributed scan: each worker reads its assigned splits; the master combines per-worker partial aggregates (`COUNT/SUM/MIN/MAX/AVG`, grouped or not, with `HAVING`) — see [Splitting a Query Across Workers](../features/query-performance.md#splitting-a-query-across-workers).

### Time travel

```sql
SELECT * FROM ice.ns.t FOR VERSION AS OF 5723102914918334234;
SELECT * FROM ice.ns.t FOR VERSION AS OF 'pre_change';      -- branch or tag name
SELECT * FROM ice.ns.t FOR TIMESTAMP AS OF '2026-04-15 12:00:00';
```

Both forms are stripped by a SQL preprocessor and routed through `TableScan.useSnapshot` / `useRef` / `asOfTime`, scoped to the current query thread.

## Write

| Statement | Implementation |
|---|---|
| `INSERT INTO catalog.ns.t SELECT/VALUES ...` | Workers write data-file shards (in the table's `write.format.default`, see [File formats](#file-formats)) in parallel, master commits a single `AppendFiles` snapshot. Partition routing happens on the worker. |
| `CREATE TABLE catalog.ns.t [PARTITIONED BY (...)] AS SELECT ...` | Same path as INSERT, plus catalog refresh so the new table is queryable immediately. |
| `DELETE FROM catalog.ns.t WHERE <pred>` | Merge-on-read. Workers scan their assigned files, evaluate the predicate row-by-row, and write deletes — position-delete parquet on v2 tables, **deletion vectors** on v3 (see [Format version 3](#format-version-3-deletion-vectors)); master commits one `RowDelta`. |
| `UPDATE catalog.ns.t SET col = expr [WHERE <pred>]` | RowDelta — original rows marked deleted (position-delete on v2, deletion vector on v3) plus a new data file with updated values. |
| `MERGE INTO catalog.ns.t [[AS] t] USING (<query>) \| <table> [[AS] s] ON <cond> WHEN MATCHED [AND <cond>] THEN UPDATE SET ... \| DELETE ... WHEN NOT MATCHED [AND <cond>] THEN INSERT [(...)] VALUES (...)` | Source is a subquery / `VALUES` or a table name (any catalog). Several `WHEN` clauses, evaluated in order (SQL standard / Trino). Composite-key joins and non-equi remainder predicates are supported. Single-RowDelta commit. |
| `OPTIMIZE catalog.ns.t` | Delete-aware compaction: small data files are read with their position-/equality-deletes applied, written as one or more compacted files, deletes dropped. |

**Writing into many partitions.** An `INSERT`, CTAS or streaming sink into a partitioned table used to keep one
open Parquet writer per partition until commit. Each holds per-column buffers and its whole row group, so one
statement spanning a few thousand day partitions ran a worker with `-Xmx896m` out of heap. The writer now works
like Spark's clustered write: rows are clustered by partition in memory and each partition is written as one
file, at most `ontul.iceberg.writer.max.open.partitions` (default 8) at a time. When the rows buffered by all
writers of the JVM exceed the budget (`ontul.iceberg.writer.max.buffered.bytes`, by default a quarter of the
heap), the writer that crossed it writes its largest partitions to files right away; a partition that receives
more rows later gets another file. Memory and open files stay bounded at any partition count (verified with
2,556 and 8,805 day partitions from one statement on workers with `-Xmx896m`). A write that fits the budget gives
one file per partition per worker task; larger writes give one file per partition per flush, which `OPTIMIZE`
compacts. A table with thousands of one-row files is slow to read, as with any engine — compact it.

`MERGE INTO` parsing tolerates aliases without `AS` (`MERGE INTO t alias ...`), nested-paren source subqueries (`USING (SELECT * FROM (VALUES ...))`), and composite ON conditions like `t.id = s.id AND t.region = s.region`.

**The source can be a table.** `USING <table>` needs no parentheses and no `SELECT * FROM`: the name may be
`table`, `schema.table` or fully qualified, in any catalog (`USING tpch.tiny.nation n`). Without an alias,
columns are qualified by the tables' own names, as in Trino:

```sql
MERGE INTO ice.sales.orders USING ice.staging.order_updates
ON orders.id = order_updates.id
WHEN MATCHED THEN UPDATE SET total = order_updates.total;
```

**Several WHEN clauses, with conditions, and DELETE.** A MERGE takes any number of
`WHEN MATCHED [AND <condition>] THEN UPDATE SET ... | DELETE` and
`WHEN NOT MATCHED [AND <condition>] THEN INSERT [(<columns>)] VALUES (...)` clauses. As in the SQL standard
and Trino they are evaluated **in the order written**: for a matched target row the first `WHEN MATCHED`
clause whose condition holds acts on it (later ones are not looked at); if none holds, the row is left as it
is. A source row that matched a target row is never inserted. For an unmatched source row the first
`WHEN NOT MATCHED` clause whose condition holds inserts it; if none holds, nothing is inserted.

```sql
MERGE INTO ice.sales.orders AS t
USING ice.staging.changes AS c
ON t.id = c.id
WHEN MATCHED AND c.op = 'D' THEN DELETE                          -- checked first
WHEN MATCHED AND (c.op = 'U' OR c.total > t.total) THEN UPDATE SET total = c.total
WHEN MATCHED THEN UPDATE SET status = 'seen'                       -- every other matched row
WHEN NOT MATCHED AND c.op <> 'D' THEN INSERT (id, total, status) VALUES (c.id, c.total, 'new');
```

- A condition may use target and source columns: comparisons with literals or with another column,
  `IS [NOT] NULL`, `IN`, `AND` / `OR` / `NOT`. An unqualified column name is resolved to whichever side has
  it; a name both sides have must be qualified. A condition Ontul cannot evaluate row by row fails the
  MERGE before anything is written, instead of being ignored.
- `INSERT VALUES (...)` without a column list fills every target column in table order.
- `WHEN MATCHED ... THEN DELETE` needs the `data:Delete` privilege on the target in addition to
  `data:Insert`.
- `WHEN NOT MATCHED BY SOURCE` (not in the SQL standard or Trino) is refused with an error.
- `SET` and `INSERT VALUES` take literals and column references; compute anything else in the `USING`
  query and assign the resulting column.

The source relation may carry a **Trino-style column-alias list** so a bare `VALUES` block can be used directly as the MERGE source without an inner `SELECT ... AS`:

```sql
MERGE INTO ice.ns.events t
USING (VALUES (1, 'US', 100), (2, 'EU', 200)) AS s (id, region, amount)
ON t.id = s.id
WHEN MATCHED THEN UPDATE SET amount = s.amount
WHEN NOT MATCHED THEN INSERT (id, region, amount) VALUES (s.id, s.region, s.amount);
```

The alias column list (`s (id, region, amount)`) is applied positionally to the source columns, so `ON` / `SET` / `INSERT` can reference them by name. Both `AS s (...)` and `s (...)` spellings are accepted.

**How rows match, and what MERGE refuses.**

- Key columns are compared as values of the **target column's type**: `DECIMAL` `1.5` matches a stored
  `1.50`, a `DATE` from the source matches the stored date, a `DOUBLE` `1.0` matches an `INT` `1`. (They
  used to be compared as text of whatever each side decoded to, so such a row read as *not matched* and
  `WHEN NOT MATCHED THEN INSERT` added it a second time.) A source value that cannot be converted to the
  key column's type matches nothing.
- A target row matched by **more than one source row** fails the MERGE — `MERGE: one target row matched
  more than one source row (...); make the USING query return at most one row per key` — as in the SQL
  standard and Trino. The first source row used to win and the others were inserted as duplicates.
- An `UPDATE` that changes a **partition column** writes the row into its new partition.
- The commit is validated against concurrent writers: the data files it deletes from must still exist,
  and no conflicting data or delete files for its keys may have been committed since it read the table.
  When a compaction (`OPTIMIZE`, maintenance `rewrite_data_files`), a streaming append/upsert or another
  MERGE got there first, the MERGE re-reads the table and tries again, up to `ontul.merge.commit.attempts`
  (default 4); after that it fails and changes nothing. (Without the check, a compaction committed during
  a MERGE left the old row in the compacted file next to the updated one.) `UPDATE` and `DELETE` validate
  the same way and fail instead of committing over a rewritten file.
- The SQL page's row cap (`ontul.admin.query.max.rows`) applies to the rows a `SELECT` shows only; the
  `USING` query of a MERGE (and any query a DML statement runs) always reads every row. It used to inherit
  the cap, so a MERGE run from the SQL page or `/admin/query/execute` processed only the first 1001
  source rows.
- On a v3 table, a repeated MERGE replaces the deletion vector only of the data files it deletes from;
  the vectors of other files stay as they are.

### Format version 3 — deletion vectors

Ontul reads and writes Iceberg **format-version 3**. The headline v3 feature is the **deletion vector**, which replaces position-delete files: instead of writing a parquet file of `(file_path, pos)` rows, a v3 table stores a roaring bitmap of deleted row positions for each data file inside a [Puffin](https://iceberg.apache.org/puffin-spec/) file (`deletion-vector-v1` blob). Deletion vectors are more compact and far cheaper to apply at read time (a bitmap membership test instead of a join against delete rows).

On a v3 table, `DELETE` / `UPDATE` / `MERGE` write deletion vectors (one per touched data file, packed into a Puffin file) and commit a `RowDelta`; the read path detects the `PUFFIN`-format delete files and applies the bitmap. A repeated row-level operation on the same data file reads the existing vector, unions the new positions, and supersedes the old vector (a data file carries at most one deletion vector). The deletion-vector codec is hand-rolled to the Iceberg/Delta byte layout and verified against Iceberg's own reference reader, so other v3 engines read Ontul-written vectors.

Format version is **per-table** and **opt-in**: v2 (position-delete files) is the default. Set the server default to create new tables on v3:

```properties
# ontul.properties
ontul.iceberg.default.format.version=3
```

Existing v2 tables keep their version, and the library reads/writes v1/v2/v3 regardless — upgrading the engine never drops v2 support. Equality deletes (used by streaming upsert) remain valid in v3.

### Streaming ingestion

Ontul streaming jobs can write directly into Iceberg tables. Barrier checkpoints align with snapshot boundaries; each barrier produces one Iceberg snapshot with exactly-once semantics on the source side and atomic visibility on the read side. See [Streaming Guide](../streaming/guide.md).

### Streaming upsert (equality-delete)

Append-only is the default streaming sink mode. For sources whose rows can be re-emitted with the same key (CDC, "latest snapshot per id" feeds, late-arriving corrections), Ontul also supports v2 **equality-delete upsert** keyed on a user-supplied column list:

```java
df.sink(Sink.table("ice.ns.users").upsertKeys("id"))
  .commitInterval(5_000)
  .start();
```

Or via SUBMIT STREAMING JSON:

```json
{
  "kafka": { ... },
  "sink": { "type": "table", "table": "ice.ns.users", "upsertKeys": ["id"] }
}
```

Per commit window the sink emits a single `RowDelta` containing the new data file plus a small key-only equality-delete file. Iceberg's MOR rules apply the delete to all earlier-sequence data files in the same partition spec, so a row with a previously-seen key is hidden at read time without scanning the target — same effect as `MERGE INTO`, but the cost is **one append + one tiny delete file** rather than a full target rescan.

Supported on both unpartitioned and partitioned tables (one per-partition equality-delete file per commit). Partition columns and equality-key columns can differ; standard Iceberg semantics apply.

## Schema evolution

| Statement | Behavior |
|---|---|
| `ALTER TABLE t ADD COLUMN c TYPE [COMMENT '...']` | Field id assigned, existing rows read as NULL. |
| `ALTER TABLE t DROP COLUMN c` | Soft-drop — field id kept in the schema history; older files still readable. |
| `ALTER TABLE t RENAME COLUMN old TO new` | Field id preserved, files written under `old` continue to read under `new`. |
| `ALTER TABLE t ALTER COLUMN c SET DATA TYPE bigint` | Iceberg-allowed widenings only (`int → bigint`, `float → double`). Older parquet files are read at their physical type and widened in-flight. |
| `ALTER TABLE t SET PROPERTIES k=v, ...` | |

After every schema-mutating DDL, the Calcite-side catalog entry is re-registered (column list, types) so subsequent SELECTs in the same session see the new schema without restart.

## Snapshot management

```sql
-- Branches and tags
ALTER TABLE ice.ns.t EXECUTE create_branch(name => 'pre_change');
ALTER TABLE ice.ns.t EXECUTE create_branch(name => 'pre_change', snapshot_id => 1234);
ALTER TABLE ice.ns.t EXECUTE create_tag(name => 'v1', snapshot_id => 1234);
ALTER TABLE ice.ns.t EXECUTE drop_branch(name => 'pre_change');
ALTER TABLE ice.ns.t EXECUTE drop_tag(name => 'v1');

-- Rollback the main branch to an earlier snapshot
ALTER TABLE ice.ns.t EXECUTE rollback_to_snapshot(snapshot_id => 1234);
```

Branches and tags are visible to time travel: `SELECT * FROM t FOR VERSION AS OF 'pre_change'`.

## Partition evolution

```sql
ALTER TABLE ice.ns.t EXECUTE add_partition_field(field => 'region');
ALTER TABLE ice.ns.t EXECUTE drop_partition_field(field => 'region');
```

### Missing objects and failures

DDL answers like Trino. Dropping an object that does not exist is an error naming it, and `IF EXISTS` makes it a
no-op; creating one that exists is an error, and `IF NOT EXISTS` makes it a no-op -- except a plain CTAS, which
appends to an existing table (below). A statement that fails is an error carrying the cause -- never a success:

| Statement | Answer |
|---|---|
| `CREATE TABLE IF NOT EXISTS ice.ns.t AS SELECT ...` (t exists) | `CREATE TABLE 0 (table ice.ns.t already exists; nothing written)` — nothing is read or written |
| `CREATE TABLE ice.ns.t AS SELECT ...` (t exists) | **appends** the rows: `CREATE TABLE <n>`. This differs from Trino (an error); the SDK table sink relies on it |
| `REFRESH MATERIALIZED VIEW ice.ns.mv` | **replaces** the stored rows in one `overwrite` snapshot: `REFRESH MATERIALIZED VIEW <n>` (see [materialized views](../query/guide.md#materialized-views)) |
| `CREATE MATERIALIZED VIEW ice.ns.mv ...` (exists) | `Materialized view 'ice.ns.mv' already exists` |
| `REFRESH` / `DROP MATERIALIZED VIEW ice.ns.missing` | `Materialized view 'ice.ns.missing' does not exist` |
| `DROP TABLE ice.ns.mv` (a materialized view) | `'ice.ns.mv' is a materialized view; use DROP MATERIALIZED VIEW ice.ns.mv` |
| `DROP TABLE ice.ns.missing` | `Table 'ice.ns.missing' does not exist` |
| `DROP TABLE ice.missing` | `Table 'ice.default.missing' does not exist` (a two-part name is read in `default`) |
| `DROP VIEW ice.ns.missing` | `View 'ice.ns.missing' does not exist` |
| `DROP SCHEMA ice.missing` | `Schema 'ice.missing' does not exist` |
| `DROP SCHEMA [IF EXISTS] ice.ns` (with tables) | `Cannot drop non-empty schema 'ice.ns'` |
| `CREATE SCHEMA ice.ns` (exists) | `Schema 'ice.ns' already exists` |
| `CREATE TABLE ice.ns.t (...)` (exists) | `Table 'ice.ns.t' already exists` |
| `DROP TABLE tpch.tiny.nation` | `DROP TABLE is not supported for catalog 'tpch' (connector tpch)` |
| `DROP TABLE nocat.s.t` | `Catalog 'nocat' is not loaded on this master` |
| a drop the catalog refuses | `DROP TABLE ice.ns.t failed: <the catalog's reason>` |

Quoted names are unquoted first (`DROP TABLE "ice"."ns"."T"` drops `ice.ns.T`). Earlier releases answered success
when the drop failed, when nothing by that name existed, for a catalog that does not exist or cannot drop tables,
and for `DROP SCHEMA IF EXISTS` of a schema that would not drop; `CREATE TABLE` of an existing table answered success
and kept the old definition.

Partition spec evolution preserves history — files written under the old spec keep their original partition values, new files use the new spec. Reads transparently span both.

## Views

Views are persisted via the Iceberg view spec (`ViewCatalog.buildView`) and remain interoperable with other Iceberg-aware engines. Internally Ontul registers each view as an `OntulViewTable` whose body is re-planned with Ontul's own SQL parser/validator at every reference, so view-body identifiers resolve under the same case rules as user queries.

```sql
CREATE VIEW ice.ns.v_recent AS
  SELECT id, total FROM ice.ns.orders WHERE total >= 100;

SELECT count(*) FROM ice.ns.v_recent;
```

### Sharing a Polaris catalog with Trino (cross-engine views)

Ontul and Trino (or Spark) can register the **same** Iceberg REST catalog — for example one Polaris warehouse — and
use each other's tables and views: DDL in one engine, reads in the other. Register the catalog in Ontul as usual
(Polaris without STS: `stsUnavailable: true`, no `roleArn`; Ontul reads S3 with its own keys). A Trino catalog on
the same warehouse looks like this (Trino 476; Trino 451 cannot send the OAuth2 scope Polaris requires):

```properties
connector.name=iceberg
iceberg.catalog.type=rest
iceberg.rest-catalog.uri=http://polaris:8181/api/catalog
iceberg.rest-catalog.warehouse=<polaris catalog>
iceberg.rest-catalog.security=OAUTH2
iceberg.rest-catalog.oauth2.credential=<client id>:<client secret>
iceberg.rest-catalog.oauth2.scope=PRINCIPAL_ROLE:ALL
iceberg.rest-catalog.vended-credentials-enabled=false
fs.native-s3.enabled=true
s3.endpoint=<S3 endpoint, e.g. ShannonStore>
s3.path-style-access=true
s3.region=us-east-1
s3.aws-access-key=...
s3.aws-secret-key=...
```

**Views another engine created.** An Iceberg view stores its SQL per *dialect* together with a `default-catalog`
and `default-namespace`. Trino writes dialect `trino` and keeps unqualified names as written (re-formatted):

| Trino statement | Stored in Polaris |
|---|---|
| `CREATE VIEW iceberg.wf.v1 AS SELECT k, v FROM iceberg.wf.t1 WHERE k = 1` (no session catalog) | `default-namespace: ["wf"]`, no `default-catalog`, SQL `SELECT k, v FROM iceberg.wf.t1 WHERE (k = 1)` |
| `USE iceberg.wf; CREATE VIEW v2 AS SELECT k FROM t1 WHERE k = 2` | `default-catalog: "iceberg"`, `default-namespace: ["wf"]`, SQL `SELECT k FROM t1 WHERE (k = 2)` |

Ontul reads both. The relations of another engine's view body are resolved as that engine would:

- `t` → `<catalog>.<default-namespace>.t`; `ns.t` → `<catalog>.ns.t` (two parts are schema.table, as in Trino);
- `<default-catalog>.ns.t` → `<catalog>.ns.t` when Ontul has no catalog of that name (Trino may call the catalog
  `iceberg` while Ontul registered the same warehouse as `ice`);
- `<catalog>` is the Ontul catalog the view was found in; without a stored default namespace the view's own
  namespace is used; a CTE's name is not a relation.

When a view has several SQL representations Ontul plans its own (`ontul`) if present, then the one closest to ANSI
SQL (`trino` before `spark`), then the first. Spark's backtick-quoted names are read as quoted names. A body Ontul
cannot read — a Trino lambda, a function Ontul does not have — fails only queries on that view, naming the view and
the dialect:

```text
View 'iceberg.wf.v_bad' uses trino SQL that ontul cannot parse: No match found for function signature transform(...)
```

Note: Trino stores the **view's own namespace** as `default-namespace`, not the session schema it resolved names
in. `USE iceberg.a; CREATE VIEW iceberg.b.v AS SELECT * FROM t` therefore stores `FROM t` with
`default-namespace: ["b"]`; Trino itself then cannot read the view (`Table 'iceberg.b.t' does not exist`), and Ontul
answers `View 'iceberg.b.v' is invalid: its base table 'iceberg.b.t' does not exist`. Create such views with
qualified names.

**Views Ontul creates** are stored with the dialect `ontul` and — so that Trino can read them (Trino reads only its
own dialect: `Cannot read unsupported dialect 'ontul'`) — the same SQL again under `trino`, with
`default-catalog` = the Ontul catalog name and `default-namespace` = the view's namespace. The body is fully
qualified (Ontul's `CREATE VIEW` requires it), so Trino reads the same tables when it names the catalog as Ontul
does. The catalog config key `view.also-dialects` (comma list, default `trino`, `none` = only `ontul`) chooses the
extra dialects.

**Replaced and dropped elsewhere.** `CREATE OR REPLACE VIEW` / `DROP VIEW` in Trino are noticed on the next query on
every master: the new body is planned, a dropped view is `Object 'v' not found within 'iceberg.wf'`.

**Listings.** `SHOW TABLES` lists tables and views, sorted by name, as Trino does. `SHOW VIEWS FROM <catalog>.<schema>`
lists the views with whether Ontul can query each (`View`, `Status` = `valid` / `invalid`, `Reason`). `DESCRIBE`
of a view shows the columns stored with it. The Catalog Browser marks views (`view`, and `view, invalid` with the
reason on hover). All listings show a user only what they may `data:Select` — a user granted only a view sees the
catalog, schema and view. Flight SQL `GetTables` types Iceberg views as `VIEW`.

**Access control.** A view is authorized like any view: the caller needs `data:Select` on the view; the base tables
it reads are reached through the view's grant (as Trino's default `SECURITY DEFINER`), and a direct query on a base
table still needs its own grant. A Deny on the view denies.

## SQL examples

Copy-pasteable recipes against an Iceberg catalog registered as `ice`. All statements run through Arrow Flight SQL, so any JDBC tool (DBeaver, DataGrip), the Ontul SDK, or the Admin UI's SQL editor can execute them.

### Namespace + table creation

```sql
CREATE SCHEMA IF NOT EXISTS ice.sales;

-- Plain v2 table
CREATE TABLE ice.sales.orders (
  id BIGINT,
  total INTEGER,
  region STRING
);

-- Hidden partitioning + sort order
CREATE TABLE ice.sales.events (
  id BIGINT,
  ts TIMESTAMP,
  region STRING,
  amount INTEGER
)
PARTITIONED BY (day(ts), bucket(8, region))
SORT BY (id ASC);

-- Inline table properties
CREATE TABLE ice.sales.props_demo (
  id BIGINT
) WITH (
  write.format.default = 'parquet',
  custom.tag = 'ml-features'
);

-- Spark-style USING + TBLPROPERTIES, ORC data files
CREATE TABLE ice.sales.orc_demo (
  id INT, name VARCHAR, amount DOUBLE
) USING iceberg
TBLPROPERTIES ('format-version'='2', 'write.format.default'='orc');
```

### Insert

```sql
INSERT INTO ice.sales.orders VALUES (1, 100, 'us'), (2, 50, 'eu'), (3, 30, 'us');

INSERT INTO ice.sales.events VALUES
  (1, TIMESTAMP '2024-03-15 12:00:00', 'us-east', 100),
  (2, TIMESTAMP '2024-03-15 13:00:00', 'us-west', 200),
  (3, TIMESTAMP '2024-03-16 12:00:00', 'us-east', 150),
  (4, TIMESTAMP '2024-03-16 14:00:00', 'eu-west', 300);
```

### CTAS with partitioning, sort, properties

```sql
CREATE TABLE ice.sales.orders_by_region
PARTITIONED BY (region)
SORT BY (id ASC)
WITH (write.format.default = 'parquet')
AS SELECT id, region, total FROM ice.sales.orders;
```

### DELETE (position-delete MOR)

```sql
DELETE FROM ice.sales.orders WHERE total < 50;
```

### UPDATE (RowDelta)

```sql
UPDATE ice.sales.orders SET total = 999 WHERE id = 1;
```

### MERGE INTO

Composite key (multiple `AND`-connected equalities) and non-equi remainder predicates are both supported. The source can be an inline `VALUES`, a `SELECT`, any subquery — including nested parentheses around a `VALUES` list — or a table name (`USING ice.staging.changes c`). Several `WHEN [NOT] MATCHED [AND <condition>]` clauses, including `THEN DELETE`, are evaluated in order; see [Write](#write) above.

```sql
MERGE INTO ice.sales.orders t
USING (SELECT * FROM (VALUES (1, 'us', 200), (3, 'us', 30)) AS v(id, region, total)) s
ON t.id = s.id AND t.region = s.region
WHEN MATCHED THEN UPDATE SET t.total = s.total
WHEN NOT MATCHED THEN INSERT (id, region, total) VALUES (s.id, s.region, s.total);
```

### Schema evolution

```sql
-- Add / drop / rename / type promote
ALTER TABLE ice.sales.orders ADD COLUMN currency STRING;
ALTER TABLE ice.sales.orders DROP COLUMN currency;
ALTER TABLE ice.sales.orders RENAME COLUMN total TO total_cents;
ALTER TABLE ice.sales.orders ALTER COLUMN total_cents SET DATA TYPE BIGINT;

-- Properties + comments
ALTER TABLE ice.sales.orders SET PROPERTIES write.format.default='parquet', custom.tag='ml-features';
COMMENT ON TABLE  ice.sales.orders          IS 'order line items';
COMMENT ON COLUMN ice.sales.orders.id        IS 'primary key';
```

After every schema-mutating DDL the Calcite-side catalog entry is re-registered automatically — subsequent SELECTs see the new column list without restart.

### Snapshots: rollback, branch, tag

```sql
-- Mark a checkpoint before a destructive change
ALTER TABLE ice.sales.orders EXECUTE create_branch(name => 'pre_change');

-- ... mutate ...
DELETE FROM ice.sales.orders WHERE id = 2;

-- Roll the main branch back if needed
ALTER TABLE ice.sales.orders EXECUTE rollback_to_snapshot(snapshot_id => 1234567890);

-- Or read the branch directly
SELECT count(*) FROM ice.sales.orders FOR VERSION AS OF 'pre_change';

-- Tag a known-good snapshot for later auditing
ALTER TABLE ice.sales.orders EXECUTE create_tag(name => 'v1', snapshot_id => 1234567890);

-- Cleanup
ALTER TABLE ice.sales.orders EXECUTE drop_branch(name => 'pre_change');
ALTER TABLE ice.sales.orders EXECUTE drop_tag(name => 'v1');
```

### Time travel

```sql
SELECT * FROM ice.sales.orders FOR VERSION AS OF 5723102914918334234;
SELECT * FROM ice.sales.orders FOR VERSION AS OF 'pre_change';
SELECT * FROM ice.sales.orders FOR TIMESTAMP AS OF '2026-04-15 12:00:00';
```

### Partition evolution

```sql
ALTER TABLE ice.sales.events EXECUTE add_partition_field(field => 'region');
ALTER TABLE ice.sales.events EXECUTE drop_partition_field(field => 'region');
```

Existing files stay under their original spec; new writes follow the new spec. Reads transparently span both.

### Views

```sql
CREATE OR REPLACE VIEW ice.sales.v_recent AS
  SELECT id, total FROM ice.sales.orders WHERE total >= 100;

SELECT count(*)    FROM ice.sales.v_recent;
SELECT max(total)  FROM ice.sales.v_recent;

DROP VIEW IF EXISTS ice.sales.v_recent;
```

**Dropped and re-created objects.** A `DROP VIEW` or `DROP TABLE` stops resolving at once on every master, not
only the one that ran it: the next query on any master answers
`Object 'v_recent' not found within 'ice.sales'`. A view or table re-created under the same name resolves to its
new definition (a view's new body, a table's new columns) everywhere. Changes made by another engine directly in
the catalog — Trino or Spark dropping a table, replacing or dropping a view — are noticed on the next query that
reads the object: a view whose body changed is re-planned with the catalog's current body, and a dropped table or
view is "not found". A catalog refresh (`POST /admin/catalogs/<name>/refresh`, or the lookup Ontul does when a
query names an unknown object) also removes what the catalog no longer lists — but never what it could not list
(a namespace that failed to list keeps its tables) and never semantic views, which live in Ontul.

Earlier releases kept every table and view they had seen: a dropped view went on answering from its stored SQL,
a dropped table failed with `cannot plan a scan ...: Table does not exist`, and both stayed in `SHOW TABLES`.

**A view whose base table is gone.** Dropping a table does not drop the views that read it (as in Trino). A query
on such a view names the view and the missing table, on every master and whichever engine dropped the table:

```text
View 'ice.sales.v_recent' is invalid: its base table 'ice.sales.orders' does not exist
```

A view whose body no longer plans for another reason (a column it reads was dropped) answers
`View '<view>' is invalid: <reason>`. Only queries on that view fail: `SHOW TABLES`, `SHOW SCHEMAS`, the Catalog
Browser and every other table and view of the schema answer as before, the view can be dropped, and re-creating the
base table makes it valid again.
Earlier releases answered `Column 'id' not found in any table`, naming neither the view nor the table.

### Cross-catalog federation

Iceberg tables can be joined directly with other registered catalogs (JDBC, file, TPC-DS, etc.) in a single query — the planner pushes filters and projection into each connector and shuffles only the bare minimum across workers.

```sql
SELECT o.id, o.total, c.name
FROM ice.sales.orders o
JOIN postgres.public.customers c ON o.customer_id = c.id
WHERE o.total > 100;
```

## Access control

Ontul IAM is the authoritative policy boundary for Iceberg data — column-level masking, row-level filters, table-level grants, plus job/source/sink scopes that Polaris's catalog-level RBAC does not express. Polaris-side table IAM is bypassed by design (no vended credentials, no delegated reads). See [IAM](../features/iam.md).
