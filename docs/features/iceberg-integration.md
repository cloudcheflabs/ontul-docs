# Iceberg Integration

Ontul integrates with Apache Iceberg as a first-class catalog. The integration is **format-version 2** native — every write is a v2 snapshot — and includes the full operator surface that production tables need: hidden partitioning, position-delete DELETE, RowDelta UPDATE/MERGE, schema evolution with safe-type promotion, partition evolution, snapshot rollback, branches and tags, time travel, and Iceberg-spec views.

The data path is implemented in Ontul. The Apache Iceberg JAR is used only as a metadata library (catalog client, manifest schema, snapshot bookkeeping). Parquet writes, parquet reads with Iceberg field-id resolution, partition transforms, partition path encoding, predicate evaluation against position deletes, and snapshot commits are all Ontul code. This is what lets Ontul distribute every write and read across workers and bypass Polaris-side credential vending entirely.

## Catalog Type

Ontul supports the Iceberg **REST catalog** only. Storage is S3-compatible (ShannonStore, AWS S3, MinIO). Multiple Iceberg catalogs can be registered side-by-side, each with its own REST endpoint and S3 credentials.

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

!!! note "Row-level deletes are Parquet-only"
    Equality- and position-**delete** files are always written as Parquet (independent of `write.format.default`). Reading an ORC/Avro *data* file that has delete files attached is not yet supported and fails with a clear error rather than silently returning stale rows.

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
| `DELETE FROM catalog.ns.t WHERE <pred>` | Position-delete based MOR. Workers scan their assigned files, evaluate the predicate row-by-row, write per-file position-delete parquet, master commits one `RowDelta`. |
| `UPDATE catalog.ns.t SET col = expr [WHERE <pred>]` | RowDelta — original rows marked as position-deletes plus a new data file with updated values. |
| `MERGE INTO catalog.ns.t USING (<source>) ON (<eq+non-equi>) WHEN MATCHED THEN UPDATE SET ... WHEN NOT MATCHED THEN INSERT (...) VALUES (...)` | Composite-key joins (multiple `AND`-connected equalities) and non-equi remainder predicates are supported. Single-RowDelta commit. |
| `OPTIMIZE catalog.ns.t` | Delete-aware compaction: small data files are read with their position-/equality-deletes applied, written as one or more compacted files, deletes dropped. |

`MERGE INTO` parsing tolerates aliases without `AS` (`MERGE INTO t alias ...`), nested-paren source subqueries (`USING (SELECT * FROM (VALUES ...))`), and composite ON conditions like `t.id = s.id AND t.region = s.region`.

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
