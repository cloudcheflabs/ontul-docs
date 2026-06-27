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
doesn't conflict with concurrent index builds on continuously-ingested tables. A manual
**Lance Maintenance** page is also available in the Admin UI.

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

## Cross-engine interoperability

Because Ontul uses the `org.lance` lineage, the Lance datasets it writes are readable by Spark
(`lance-spark`) and Trino (`lance-trino`), and vice versa — over both directory and Polaris
namespaces.

## Notes and current limits

- Writes are master-side single-node today (distributed worker-side fragment writes are planned).
- Streaming is at-least-once.
- `UPDATE` is non-atomic (delete + re-append) and assumes the SET preserves column types; use
  `MERGE` for atomic new-value upserts.
- Publish is tag-gated, not branch-isolated (the current Lance Java API has no branch-write/merge).
