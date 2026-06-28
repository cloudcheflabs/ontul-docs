# Lance Integration

Ontul integrates [Lance](https://lance.org) / LanceDB as a first-class catalog connector — a
columnar format purpose-built for vector search and multimodal data. Ontul reads, searches
(vector ANN + full-text), writes (`INSERT`/`CTAS`/`DELETE`/`UPDATE`/`MERGE`), streams from Kafka,
maintains, and exposes Lance tables to retrievers, all through standard SQL.

The connector is built on the **vendor-neutral `org.lance` lineage** (Apache-2.0) — the same
lineage the Spark (`lance-spark`) and Trino (`lance-trino`) connectors use — so datasets Ontul
writes are format-compatible and cross-engine readable. The Lance native engine (Rust, via JNI)
is bundled in the connector; nothing extra to install.

## Register a catalog

Two namespace modes via `impl` (default `dir`).

**Directory layout** (`impl=dir`) — `<root>/<schema>/<table>.lance` on local FS or S3:

```json
{
  "connector": "lance", "impl": "dir",
  "root": "s3://my-bucket/lancewh", "default-namespace": "public",
  "s3.endpoint": "https://s3.example.com",
  "s3.accessKey": "…", "s3.secretKey": "…", "s3.region": "us-east-1", "s3.pathStyle": "true"
}
```

**Apache Polaris Generic Tables** (`impl=polaris`, format=lance) — Polaris holds only the
metadata pointer; data is read/written with the connector's own S3 credentials (STS disabled,
mirroring Ontul's Iceberg credential-vending bypass):

```json
{
  "connector": "lance", "impl": "polaris",
  "polaris.uri": "http://polaris:8181", "polaris.catalog": "ontul_cat",
  "polaris.client-id": "…", "polaris.client-secret": "…",
  "lance.warehouse": "s3://my-bucket/lancewh",
  "s3.endpoint": "…", "s3.accessKey": "…", "s3.secretKey": "…", "s3.region": "us-east-1"
}
```

## Query and search

Standard SQL scans flow straight into the Arrow engine; IAM column masking and row-level
filters apply at the scan output (engine-side, connector-agnostic).

```sql
SELECT id, title FROM lance.public.docs WHERE category = 'news';
```

Index-served search via two table functions that compose inside a larger `SELECT`:

```sql
-- full-text (BM25)
SELECT * FROM match('lance.public.docs', 'body', 'distributed systems', 10);
-- vector ANN (query vector is a quoted comma list; optional k)
SELECT id, title FROM vector_search('lance.public.docs', 'embedding', '0.1,0.2,0.3', 5);
```

### Indexes

```sql
CREATE INDEX idx_body ON lance.public.docs (body)      WITH (type='inverted', base_tokenizer='simple');
CREATE INDEX idx_emb  ON lance.public.docs (embedding) WITH (type='ivf_flat', metric='cosine');
CREATE INDEX idx_id   ON lance.public.docs (id)        WITH (type='btree');
DROP INDEX idx_id ON lance.public.docs;
```

Types: `btree` / `bitmap` (scalar secondary), `ivf_flat` / `ivf_pq` (vector ANN), `inverted` /
`fts` (full-text).

### Korean / CJK full-text

CJK full-text (`base_tokenizer='lindera/ko-dic'`, `jieba/default`) is served through a bundled
**pylance sidecar** for both index build and query tokenization. In a packaged distribution this
resolves automatically — set `ontul.python.path` to a Python with `pylance` installed and (optionally)
a cluster default `ontul.lance.fts.base-tokenizer=lindera/ko-dic`. The ko-dic dictionary ships with
the distribution.

## Write

```sql
CREATE TABLE lance.public.events AS SELECT id, name FROM source;   -- CTAS
INSERT INTO lance.public.events SELECT id, name FROM more_rows;     -- append
DELETE FROM lance.public.events WHERE id < 100;                     -- deletion vectors
UPDATE lance.public.events SET name = upper(name) WHERE id = 1;     -- engine rewrite
MERGE INTO lance.public.events USING (SELECT …) s ON lance.public.events.id = s.id
  WHEN MATCHED THEN UPDATE SET name = s.name
  WHEN NOT MATCHED THEN INSERT (id, name) VALUES (s.id, s.name);    -- native mergeInsert upsert
```

`DELETE` and `MERGE` use Lance natives (deletion vectors / `mergeInsert`); `UPDATE` is an
engine-side rewrite (project the SET, delete, re-append).

## Streaming (Kafka → Lance)

A streaming job whose sink table is a Lance catalog appends micro-batches (commit on a row-count
threshold or time interval, flush on stop):

```json
{ "sink": { "type": "table", "table": "lance.public.events" } }
```

## Maintenance

Lance compaction and storage reclaim are two **separate** steps — `OPTIMIZE` compacts fragments
for read performance but does not free disk (old versions still reference old files); `VACUUM`
reclaims:

```sql
OPTIMIZE lance.public.events WITH (target_rows_per_fragment='1048576', defer_index_remap='true');
VACUUM   lance.public.events WITH (older_than_days='7');
EXPIRE SNAPSHOTS lance.public.events;   -- alias of VACUUM for Lance
```

`defer_index_remap` (Fragment Reuse Index) lets compaction skip the vector-index remap so it
doesn't conflict with concurrent index builds on continuously-ingested tables.

**Streaming — compact only recent commits, not the whole table.** Lance has no hidden partitions,
so a time window is resolved from version timestamps and the plan-based compaction reads the manifest
(not the data), rewriting only the matching fragments:

```sql
OPTIMIZE lance.public.events WITH (
  window_hours='2',              -- only fragments committed in the last N hours
  window_cooldown_minutes='5',   -- exclude the freshest fragments (the hot zone a streaming
                                 --   writer is appending to) → never races in-flight commits
  min_input_files='2',           -- skip a task with fewer than N fragments
  dry_run='true');               -- plan + report only, no rewrite
VACUUM lance.public.events WITH (retain_last='1');   -- keep the last N versions
```

A manual **Lance Maintenance** page is also available in the Admin UI. Cross-engine verified:
ontul `OPTIMIZE`/`VACUUM` produces a dataset Trino re-reads with identical data (no loss).

## Versioning and tag-based publish

Lance keeps every write as an immutable version. Ontul exposes the audit→publish primitives:
list versions, audit a specific version's row count, and point a tag (e.g. `published`) at an
audited version so consumers reading that tag only advance when you move it.

## Retrievers

A retriever's `targetCatalog` may be a Lance catalog; its template uses `match()` /
`vector_search()` and runs in-engine:

```json
{
  "targetCatalog": "lance", "kind": "VECTOR",
  "sqlTemplate": "SELECT id, title FROM vector_search('lance.public.docs','embedding', ${qvec}, ${k})"
}
```

See **[Retrievers](retrievers.md)**.

## Python SDK

Everything above is plain SQL, so it runs through the Python SDK (`OntulSession`) unchanged —
`execute()` for DDL/DML (returns a status dict), `sql()` for reads (returns a PyArrow table).

```python
from ontul.session import OntulSession

session = OntulSession(host="localhost", port=47470, token="your-jwt-token")

# CTAS — create a Lance table from a query (distributed write: workers write fragments,
# master single-commits)
session.execute("""
    CREATE TABLE lance.public.events AS
    SELECT id, name, embedding FROM staging.raw_events
""")

# INSERT — append more rows
session.execute("INSERT INTO lance.public.events SELECT id, name, embedding FROM staging.more")

# MERGE INTO — native upsert on the ON key(s)
session.execute("""
    MERGE INTO lance.public.events USING (SELECT * FROM staging.updates) s
      ON lance.public.events.id = s.id
      WHEN MATCHED THEN UPDATE SET name = s.name
      WHEN NOT MATCHED THEN INSERT (id, name, embedding) VALUES (s.id, s.name, s.embedding)
""")

# DELETE / UPDATE
session.execute("DELETE FROM lance.public.events WHERE id < 100")
session.execute("UPDATE lance.public.events SET name = upper(name) WHERE id = 1")

# Search — read back as a PyArrow table
hits = session.sql(
    "SELECT id, name FROM vector_search('lance.public.events', 'embedding', '0.1,0.2,0.3', 5)")
print(hits.to_pandas())

fts = session.sql("SELECT * FROM match('lance.public.events', 'name', 'distributed systems', 10)")

# Maintenance (streaming-friendly: only recent commits)
session.execute("OPTIMIZE lance.public.events WITH (window_hours='2', defer_index_remap='true')")
session.execute("VACUUM lance.public.events WITH (retain_last='5')")
```

Per-session WAP staging (the SDK session carries a session id, so `SET` scopes to it):

```python
session.execute("SET ontul.lance.wap.branch=stage_2024_06")     # stage writes on a branch
session.execute("INSERT INTO lance.public.events SELECT * FROM staging.batch")
audit = session.sql("SELECT count(*) FROM lance.public.events")  # main is unchanged until publish
session.execute("PUBLISH WAP lance.public.events BRANCH 'stage_2024_06'")
```

## Cross-engine interoperability

Because Ontul uses the `org.lance` lineage, the Lance datasets it writes are readable by Spark
(`lance-spark`) and Trino (`lance-trino`), and vice versa — over both directory and Polaris
namespaces.

## Notes and current limits

- Writes are distributed (workers write Lance fragments, the master makes one commit), with an
  automatic fallback to a master-side write if a distributed write fails.
- Streaming is at-least-once (a coordinated exactly-once drain primitive exists; the full
  checkpoint-barrier integration is in progress).
- `UPDATE` is non-atomic (delete + re-append) and assumes the SET preserves column types; use
  `MERGE` for atomic new-value upserts.
- Branch-isolated WAP works on local/dir datasets; on S3/Polaris, branch operations are currently
  blocked by a Lance Java binding (they don't thread storage credentials), so WAP there errors
  clearly rather than writing to `main` — use tag-based publish on S3 until that is fixed.
- Blob V2 read is supported (serve large binary); a SQL surface to declare/insert blob columns is planned.
