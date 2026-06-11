# Iceberg Integration

Ontul integrates with Apache Iceberg as a first-class catalog. The integration is **format-version 2** native — every write is a v2 snapshot — and includes the full operator surface that production tables need: hidden partitioning, position-delete DELETE, RowDelta UPDATE/MERGE, schema evolution with safe-type promotion, partition evolution, snapshot rollback, branches and tags, time travel, and Iceberg-spec views.

The data path is implemented in Ontul. The Apache Iceberg JAR is used only as a metadata library (catalog client, manifest schema, snapshot bookkeeping). Parquet writes, parquet reads with Iceberg field-id resolution, partition transforms, partition path encoding, predicate evaluation against position deletes, and snapshot commits are all Ontul code. This is what lets Ontul distribute every write and read across workers and bypass Polaris-side credential vending entirely.

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

The S3 credential keys accept either the `s3.`-prefixed spelling (`s3.accessKey`, `s3.secretKey`, `s3.endpoint`, `s3.region`, `s3.pathStyle`) or the bare spelling (`accessKey`, `secretKey`, …) used by [Connection IDs](connection-id.md), so a catalog can be backed by a stored S3 connection (`"connectionId": "..."`) without renaming keys.

### Lookup-on-miss for externally created tables

A table created in a shared REST catalog by **another engine** (for example Trino) is picked up automatically: when a query references `catalog.ns.table` and the planner does not yet have it cached, Ontul refreshes that catalog and re-plans once before failing. No manual `POST /admin/catalogs/<name>/refresh` is needed for the new table to become queryable. (The refresh endpoint still exists for forcing a full re-list on demand.)

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
| `CREATE SCHEMA [IF NOT EXISTS] catalog.namespace` | Creates an Iceberg namespace. |
| `CREATE TABLE [IF NOT EXISTS] catalog.ns.t (cols) [USING iceberg] [PARTITIONED BY (...)] [SORT BY (...)] [WITH (props) \| TBLPROPERTIES (props)]` | Format v2, with optional hidden partitioning and sort order. `USING iceberg` (Spark style) is accepted and ignored; `WITH` and `TBLPROPERTIES` are interchangeable property clauses. |
| `CREATE TABLE catalog.ns.t [PARTITIONED BY (...)] AS SELECT ...` | CTAS, distributed across workers. |
| `DROP TABLE [IF EXISTS] catalog.ns.t` | Honors Polaris `polaris.config.drop-with-purge.enabled` for actual S3 cleanup. |
| `DROP SCHEMA [IF EXISTS] catalog.namespace` | Namespace must be empty. |
| `CREATE [OR REPLACE] VIEW catalog.ns.v AS <SELECT>` | Persisted via Iceberg's view spec; see [Views](#views). |
| `DROP VIEW [IF EXISTS] catalog.ns.v` | |
| `COMMENT ON TABLE catalog.ns.t IS '...'` | |
| `COMMENT ON COLUMN catalog.ns.t.col IS '...'` | |

### Hidden partitioning

`PARTITIONED BY (...)` accepts the standard Iceberg transforms — `identity`, `year(col)`, `month(col)`, `day(col)`, `hour(col)`, `bucket(N, col)`, `truncate(L, col)`, `void`. The transforms and partition-path encoding are implemented in Ontul (Murmur3_32 for `bucket`, ISO date math for the temporal transforms), so writes do not need a Spark/Trino runtime and the same logic runs identically on master and worker.

```sql
CREATE TABLE ice.events.events (
  id BIGINT, ts TIMESTAMP, region STRING, amount INTEGER
)
PARTITIONED BY (day(ts), bucket(8, region));
```

Predicates against the source columns are pushed into the Iceberg scan and used for partition pruning before the worker even opens a parquet file.

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
- Distributed scan: each worker reads its assigned splits; the master combines per-worker partial aggregates for `COUNT/SUM/MIN/MAX` so scalar aggregates are correct across multiple files.

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
| `MERGE INTO catalog.ns.t USING (<source>) ON (<eq+non-equi>) WHEN MATCHED THEN UPDATE SET ... [WHEN NOT MATCHED THEN INSERT (...) VALUES (...)]` | Composite-key joins (multiple `AND`-connected equalities) and non-equi remainder predicates are supported. The `WHEN NOT MATCHED` branch is optional — a `MATCHED`-only MERGE is valid. Single-RowDelta commit. |
| `OPTIMIZE catalog.ns.t` | Delete-aware compaction: small data files are read with their position-/equality-deletes applied, written as one or more compacted files, deletes dropped. |

`MERGE INTO` parsing tolerates aliases without `AS` (`MERGE INTO t alias ...`), nested-paren source subqueries (`USING (SELECT * FROM (VALUES ...))`), and composite ON conditions like `t.id = s.id AND t.region = s.region`.

The source relation may carry a **Trino-style column-alias list** so a bare `VALUES` block can be used directly as the MERGE source without an inner `SELECT ... AS`:

```sql
MERGE INTO ice.ns.events t
USING (VALUES (1, 'US', 100), (2, 'EU', 200)) AS s (id, region, amount)
ON t.id = s.id
WHEN MATCHED THEN UPDATE SET amount = s.amount
WHEN NOT MATCHED THEN INSERT (id, region, amount) VALUES (s.id, s.region, s.amount);
```

The alias column list (`s (id, region, amount)`) is applied positionally to the source columns, so `ON` / `SET` / `INSERT` can reference them by name. Both `AS s (...)` and `s (...)` spellings are accepted.

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

Partition spec evolution preserves history — files written under the old spec keep their original partition values, new files use the new spec. Reads transparently span both.

## Views

Views are persisted via the Iceberg view spec (`ViewCatalog.buildView`) and remain interoperable with other Iceberg-aware engines. Internally Ontul registers each view as an `OntulViewTable` whose body is re-planned with Ontul's own SQL parser/validator at every reference, so view-body identifiers resolve under the same case rules as user queries.

```sql
CREATE VIEW ice.ns.v_recent AS
  SELECT id, total FROM ice.ns.orders WHERE total >= 100;

SELECT count(*) FROM ice.ns.v_recent;
```

## Maintenance

A built-in maintenance service runs periodic tasks per registered Iceberg table:

- **Expire snapshots** beyond a retention window (default 7 days)
- **Rewrite data files** — small-file compaction, delete-aware
- **Rewrite manifests** for faster planning
- **Remove orphan files** that no live snapshot references

The same operations are available as on-demand SQL:

```sql
ALTER TABLE ice.ns.t EXECUTE optimize(file_size_threshold_mb => 128);
EXPIRE SNAPSHOTS ice.ns.t RETAIN LAST 5;
REMOVE ORPHAN FILES ice.ns.t;
```

For tests and quick teardown, set `polaris.config.drop-with-purge.enabled=true` on the Polaris catalog so `DROP TABLE` actually deletes the underlying S3 data.

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

Composite key (multiple `AND`-connected equalities) and non-equi remainder predicates are both supported. The source can be an inline `VALUES`, a `SELECT`, or any subquery — including nested parentheses around a `VALUES` list.

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

### Table maintenance — supported SQL

All maintenance ops are on-demand SQL; the maintenance service runs the same operations on a schedule.

```sql
-- Compact small files; drops applied position/equality deletes
ALTER TABLE ice.sales.orders EXECUTE optimize(file_size_threshold_mb => 128);
OPTIMIZE ice.sales.orders;                                           -- shorthand

-- Expire snapshots older than a window
ALTER TABLE ice.sales.orders EXECUTE expire_snapshots(retention_hours => 168);
EXPIRE SNAPSHOTS ice.sales.orders RETAIN LAST 5;                     -- keep N most recent

-- Rewrite manifests for faster planning
ALTER TABLE ice.sales.orders EXECUTE rewrite_manifests();

-- Delete S3 files no live snapshot references
REMOVE ORPHAN FILES ice.sales.orders;
```

For tests and quick teardown, set `polaris.config.drop-with-purge.enabled=true` on the Polaris catalog so `DROP TABLE` actually deletes the underlying S3 data.

### Cross-catalog federation

Iceberg tables can be joined directly with other registered catalogs (JDBC, file, TPC-DS, etc.) in a single query — the planner pushes filters and projection into each connector and shuffles only the bare minimum across workers.

```sql
SELECT o.id, o.total, c.name
FROM ice.sales.orders o
JOIN postgres.public.customers c ON o.customer_id = c.id
WHERE o.total > 100;
```

## Access control

Ontul IAM is the authoritative policy boundary for Iceberg data — column-level masking, row-level filters, table-level grants, plus job/source/sink scopes that Polaris's catalog-level RBAC does not express. Polaris-side table IAM is bypassed by design (no vended credentials, no delegated reads). See [IAM](iam.md).
