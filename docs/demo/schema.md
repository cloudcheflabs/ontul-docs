# Schema — ledger, serving, semantic

Three layers, each answering a different question.

| Layer | What | Where |
|---|---|---|
| **Ledger** | The record of fact: documents, versions, chunks, relations | Iceberg (Polaris + S3) |
| **Serving** | A rebuildable derivative: vectors, full-text, graph | NeorunBase |
| **Semantic** | The surface questions actually bind to — the temporal filter and the policies live here | Ontul semantic views |

Separating the ledger from the serving layer is the most valuable design decision
in this demo. NeorunBase can be destroyed and rebuilt with nothing lost: the
pipeline remakes it from Iceberg, and the history of what changed when is in the
Iceberg snapshots.

---

## 1. The Iceberg ledger

### Documents and versions

**`demo/schema/iceberg/01_documents.sql`**

```sql
-- ============================================================================
-- The ledger — document identity and versions
--
-- The tables in this file hold what is true. Vectors, indexes and graph edges are
-- all derivatives that can be rebuilt from here, so none of them live in the
-- ledger.
-- ============================================================================

CREATE SCHEMA IF NOT EXISTS ice.reg;

-- ── Documents: identity, independent of version ─────────────────────────────
-- doc_no identifies the document itself, not the file. In a real organisation a
-- filename like "인사규정_v2_최종_수정.pdf" cannot be trusted, so matching goes
-- through the register (the master spreadsheet) and only its result lands here.
CREATE TABLE IF NOT EXISTS ice.reg.documents (
    doc_id          BIGINT,     -- Graph vertex id. doc_no is the business key; this is a
                                -- surrogate. A graph engine's vertices have to be numbers,
                                -- and renumbering them on every rebuild means an edge
                                -- written yesterday cannot be read today. Assigned by the
                                -- ledger, it is decided once and stays.
    doc_no          VARCHAR,    -- HR-REG-003
    title           VARCHAR,
    doc_class       VARCHAR,    -- RULE / REGULATION / GUIDELINE
    tier            INT,        -- 1=top level, 2=regulation, 3=guideline; must agree with
                                -- what the document number encodes
    owner_dept      VARCHAR,    -- Owning department code (joins org.py's DEPTS)
    is_official     BOOLEAN,    -- Marked by HR as citable as the basis for an answer.
                                -- The minimum curation: not every document a 300-person
                                -- company produces is trusted.
    sensitivity     VARCHAR,    -- The classification the register carries.
                                -- Restricted documents are excluded from the candidate
                                -- set by an IAM row filter, not by the pipeline.
    source_system   VARCHAR,    -- onedrive / groupware / erp_attachment
    created_at      TIMESTAMP,
    updated_at      TIMESTAMP
);

-- ── Versions: the heart of temporal correctness ─────────────────────────────
-- effective_from comes from the approval system, not from the document body. The
-- "in force from 1 January 2025" printed in the 부칙 was written when the draft
-- was, and becomes false the moment approval slips. So both values are kept and
-- the disagreement is flagged explicitly.
CREATE TABLE IF NOT EXISTS ice.reg.doc_versions (
    doc_no          VARCHAR,
    version         INT,
    status          VARCHAR,    -- DRAFT / EFFECTIVE / SUPERSEDED / ABOLISHED
    effective_from  DATE,       -- Authoritative: the approval system's completion date
    effective_to    DATE,       -- NULL means still current; otherwise the next version's date
    stated_from     DATE,       -- For reference: the date printed in the document
    date_mismatch   BOOLEAN,    -- effective_from <> stated_from — a queue for HR to review
    approval_id     VARCHAR,    -- The approval record's id (joins the MySQL system)
    s3_uri          VARCHAR,
    file_format     VARCHAR,    -- pdf / docx / hwpx / xlsx
    sha256          VARCHAR,    -- Distinguishes the drafted .docx from the published .pdf
    is_authoritative BOOLEAN,   -- Within a sha group, the published form wins
    page_count      INT,
    ingested_at     TIMESTAMP
);

-- ── Relations ───────────────────────────────────────────────────────────────
-- The result of parsing rather than extraction. `source` is recorded because the
-- confidence differs: the register spreadsheet beats a body regex, which beats an
-- LLM's guess.
CREATE TABLE IF NOT EXISTS ice.reg.doc_relations (
    src_doc_no      VARCHAR,
    src_version     INT,        -- NULL means the relation is at document level
    dst_doc_no      VARCHAR,
    dst_version     INT,
    rel_type        VARCHAR,    -- SUPERSEDES / REFERENCES / CHILD_OF
    src_article     VARCHAR,    -- For REFERENCES, the citing article
    source          VARCHAR,    -- MASTER_XLSX / BODY_REGEX / APPROVAL
    confidence      DOUBLE,     -- Below 1.0 for body parsing; used to pick what to verify
    valid_from      DATE,
    created_at      TIMESTAMP
);

-- ── Ingest log ──────────────────────────────────────────────────────────────
-- Success or failure of each pipeline stage. The evidence for checking that no
-- file was skipped quietly.
CREATE TABLE IF NOT EXISTS ice.reg.ingest_log (
    run_id          VARCHAR,
    s3_uri          VARCHAR,
    stage           VARCHAR,    -- discover / extract / chunk / embed / merge
    status          VARCHAR,    -- ok / skipped / error
    reason          VARCHAR,    -- Why skipped: duplicate_sha / image_only_pdf / no_doc_no
    elapsed_ms      BIGINT,
    error           VARCHAR,
    ran_at          TIMESTAMP
);
```


!!! danger "`effective_from` is the heart of this demo"
    It comes from the **approval system**, not from the date printed in the
    document. 27 versions disagree, flagged as `date_mismatch`. Trust the printed
    date and you answer wrongly for ten weeks with no way to notice.

### Chunks

**`demo/schema/iceberg/02_chunks.sql`**

```sql
-- ============================================================================
-- Chunks — independent of any embedding model
--
-- The absence of an embedding column here is deliberate. How an article was split
-- is the same whatever model embeds it, so changing models does not rewrite this
-- table. Vectors go into per-generation derived tables in NeorunBase.
-- ============================================================================

CREATE TABLE IF NOT EXISTS ice.reg.doc_chunks (
    chunk_id        VARCHAR,    -- {doc_no}#{version}#{ordinal}
    -- A separate numeric key exists because of the search engine. NeorunBase's FTS
    -- index requires a numeric primary key, and HYBRID_SEARCH returns that key.
    -- chunk_id is the name a human reads; this is what the machine joins on.
    chunk_pk        BIGINT,
    doc_no          VARCHAR,
    version         INT,
    -- ordinal_no, not ordinal: ORDINAL is reserved in the planner's grammar, and
    -- a column that must be quoted in every query it appears in is a papercut
    -- paid forever to save one rename now.
    ordinal_no      INT,
    article_no      VARCHAR,    -- The article; chunking is per article, so usually 1:1
    article_title   VARCHAR,    -- The article's heading
    page_from       INT,
    page_to         INT,

    -- The original and a redacted twin, side by side. Column masking works as a
    -- swap between them: without clearance, asking for `text` returns
    -- `text_redacted`. Masking the whole body would make search pointless, so only
    -- the PII is hidden.
    text            VARCHAR,
    text_redacted   VARCHAR,
    pii_types       VARCHAR,    -- JSON array: ["RRN","PHONE","NAME"], or "[]" when none

    token_count     INT,
    created_at      TIMESTAMP
);

-- ── Embedding generations ───────────────────────────────────────────────────
-- VECTOR(n) fixes the dimension per column, so two generations cannot share one.
-- Generations are therefore kept in a registry and NeorunBase gets a table per
-- generation (doc_vectors_<generation_id>). Swapping models becomes additive:
-- build the new generation alongside, verify it, then point the retriever at it.
CREATE TABLE IF NOT EXISTS ice.reg.embedding_generations (
    generation_id   VARCHAR,    -- gen1, gen2 …
    fingerprint     VARCHAR,    -- BAAI/bge-m3@<revision>:1024:l2
    model_id        VARCHAR,
    model_revision  VARCHAR,
    dim             INT,
    normalize       BOOLEAN,
    distance        VARCHAR,    -- cosine; must match the NeorunBase HNSW index metric
    target_table    VARCHAR,    -- nb.public.doc_vectors_gen1
    status          VARCHAR,    -- BUILDING / ACTIVE / RETIRED
    chunk_count     BIGINT,
    built_at        TIMESTAMP,
    activated_at    TIMESTAMP
);
```


### The ingest queue

**`demo/schema/iceberg/50_ingest_queue.sql`**

```sql
-- ============================================================================
-- The ingest queue — the record of what the pipeline still has to process
--
-- This table exists because of scale. The earlier pipeline had the driver list
-- S3, open each file, chunk it and push the results. That works for 442 files. At
-- tens of thousands one machine is the bottleneck, and at hundreds of millions the
-- listing alone does not fit in the driver's memory.
--
-- Turn the listing into a table and everything after it becomes SQL:
--
--   INSERT INTO ice.reg.doc_chunks
--   SELECT q.s3_uri, c.* FROM ice.reg.ingest_queue q,
--          UNNEST(extract_chunks(q.s3_uri)) AS c
--
-- The driver holds nothing; each worker reads only the objects in its own split.
-- The statement is the same at 442 files and at hundreds of millions.
--
-- `status` exists for retries. Resuming after a pipeline dies halfway means how
-- far it got has to be in the data — in a log it is a state that cannot be
-- queried.
-- ============================================================================

CREATE TABLE IF NOT EXISTS ice.reg.ingest_queue (
    s3_uri          VARCHAR,   -- One object, one row
    bucket          VARCHAR,
    object_key      VARCHAR,
    size_bytes      BIGINT,
    etag            VARCHAR,   -- Whether the content changed; the same etag needs no rework
    file_format     VARCHAR,   -- pdf / docx / xlsx / hwpx
    discovered_at   TIMESTAMP,
    status          VARCHAR,   -- PENDING / EXTRACTED / FAILED / SKIPPED
    attempt         INT,
    error           VARCHAR,   -- Why it FAILED. No file may be skipped quietly
    processed_at    TIMESTAMP,
    run_id          VARCHAR    -- Which run processed this row — where lineage starts
)
WITH (identifier_fields = ARRAY['s3_uri']);
```


### Approval events (the streaming target)

**`demo/schema/iceberg/40_approval_stream.sql`**

```sql
-- ============================================================================
-- Revisions currently in approval (streaming)
--
-- The ledger (01_documents.sql) holds what has already **happened** — versions
-- that were approved and took effect. But in practice the wrong answers tend to
-- come from the other side: reciting the current regulation without knowing that a
-- revision is in approval right now. The answer is correct and omits "this is
-- about to change".
--
-- This table is filled by an Ontul Flow, not a batch. The approval system emits an
-- event per stage (drafted → reviewed → approved → in force) and the Flow upserts
-- on approval_id, keeping **one row per approval, at its latest state**. What is
-- needed is the current state, not the stage history: answering "where has this
-- got to" takes one row.
--
-- identifier_fields is the upsert key. Without it the Flow cannot write equality
-- deletes, and each stage change appends another row for the same approval — a
-- table in which "under review" and "approved" are simultaneously true.
-- ============================================================================

CREATE TABLE IF NOT EXISTS ice.reg.approval_status (
    approval_id      VARCHAR,   -- The approval record's id. One revision, one row
    doc_no           VARCHAR,   -- The regulation it targets (joins ice.reg.documents)
    version          INT,       -- The revision this approval would produce (current + 1)
    step             VARCHAR,   -- DRAFT / REVIEW / APPROVED / EFFECTIVE / REJECTED
    step_seq         INT,       -- Stage number, so a late-arriving event is still ordered
    drafter          VARCHAR,   -- The drafter's employee number
    owner_dept       VARCHAR,
    summary          VARCHAR,   -- One line on what the revision changes
    expected_from    DATE,      -- Expected effective date; before approval it is only that
    updated_at       TIMESTAMP
)
WITH (identifier_fields = ARRAY['approval_id']);
```


### The revision-request ledger — what the ontology action writes to

**`demo/schema/iceberg/60_revision_requests.sql`**

```sql
-- The revision-request ledger.
--
-- This is what the ontology action request_revision writes to. Writing into
-- Iceberg rather than into the derived serving layer is the point: NeorunBase is
-- a copy the pipeline can rebuild at any time, and a record left somewhere that
-- disappears on a rebuild is not a record.
--
-- Why "who and when" is not in this table matters. An action's SQL template
-- substitutes only its declared parameters, so there is no slot for the session
-- user — and taking requested_by as a parameter would let anyone file under
-- someone else's name. A forgeable column sitting beside real ones is worse than
-- no column at all. The caller and the time are recorded by the platform's audit
-- log, which the caller cannot edit:
--
--   SELECT * FROM ontul.audit WHERE action = 'action:invoke'
--
-- request_id is one per (regulation, version). A second request against the same
-- version is not a new request but the same one, and the idempotency key returns
-- it unchanged.
CREATE TABLE IF NOT EXISTS ice.reg.revision_requests (
    request_id    VARCHAR,
    doc_no        VARCHAR,
    version       INT,
    reason        VARCHAR,
    status        VARCHAR
) USING iceberg;
```


### The graph, projected into the lake

**`demo/schema/iceberg/61_graph_projection.sql`**

```sql
-- The graph, projected for serving.
--
-- ice.reg.doc_relations holds the semantic relation — what is the basis for what.
-- The two tables below hold that same relation in the exact shape NeorunBase's
-- graph engine consumes. Keeping two copies is deliberate, and it follows the
-- principle the ontology documentation states: Iceberg is the system of record
-- and NeorunBase is a derived serving layer that can be rebuilt at any time.
--
-- So the batch job does not write to NeorunBase. It writes here, and a Flow keeps
-- serving in step with it. The difference shows up in operation: NeorunBase can
-- be destroyed and rebuilt without re-running the job, and when and how the
-- relations changed is in the Iceberg snapshots.
--
-- The surrogate keys (doc_id, edge_pk) are assigned on the lake side. Assign them
-- in serving and they change on every rebuild, leaving nothing that can point at
-- the graph and still mean the same thing tomorrow.
CREATE TABLE IF NOT EXISTS ice.reg.graph_nodes (
    doc_id       BIGINT,
    doc_no       VARCHAR,
    title        VARCHAR,
    tier         INT,
    owner_dept   VARCHAR,
    is_official  BOOLEAN,
    sensitivity  VARCHAR
) USING iceberg;

CREATE TABLE IF NOT EXISTS ice.reg.graph_edges (
    edge_pk      BIGINT,
    src_id       BIGINT,
    dst_id       BIGINT,
    rel_type     VARCHAR,
    src_doc_no   VARCHAR,
    dst_doc_no   VARCHAR,
    src_version  INT,
    dst_version  INT,
    src_article  VARCHAR,
    source       VARCHAR,
    confidence   DOUBLE
) USING iceberg;
```


---

## 2. The NeorunBase serving schema

Vectors, Korean full-text and the graph live in one engine. Hybrid fusion happens
*in the engine* rather than in application code, and the IAM row filter is
written once rather than twice.

**`demo/schema/neorunbase/01_vectors.sql`**

```sql
-- ============================================================================
-- NeorunBase — vectors, Korean FTS and the relation graph in one engine
--
-- One engine matters here beyond tidiness. With lexical search in one system and
-- vectors in another, hybrid fusion happens in application code, which is where
-- most stacks get their ranking wrong — and the IAM row filter has to be
-- reimplemented in a second query language. Here both paths cross the same scan,
-- so the filter is written once.
-- ============================================================================

-- ── Vectors, per embedding generation ────────────────────────────────────────
-- VECTOR(n) fixes the width per column, so two generations cannot share one
-- table. That is the whole constraint — not "changing models is hard", but "two
-- vector spaces cannot occupy one column". Building the next generation beside
-- the current one turns a migration into an additive step: index, verify, switch
-- the retriever, drop the old.
-- chunk_pk is the declared primary key, and it has to be: HYBRID_SEARCH returns
-- (id, score) where id is the table's own key, so a table with no declared key
-- yields ids that join to nothing. Relying on the implicit _rowid looks like it
-- works — the FTS index accepts it — right up until the search returns internal
-- locators from a different number space and every join comes back empty.
CREATE TABLE IF NOT EXISTS doc_vectors_gen1 (
    chunk_pk     BIGINT PRIMARY KEY,
    chunk_id     TEXT NOT NULL,
    doc_no       TEXT NOT NULL,
    version      INT  NOT NULL,
    article_no   TEXT,
    -- Denormalised from the ledger on purpose. The temporal predicate has to be
    -- evaluable inside the search, not applied afterwards: filtering a top-k
    -- result set post hoc returns fewer than k rows, and sometimes none.
    effective_from DATE NOT NULL,
    effective_to   DATE,
    is_official  BOOLEAN NOT NULL,
    sensitivity  TEXT NOT NULL,
    owner_dept   TEXT NOT NULL,
    body         TEXT NOT NULL,        -- Nori FTS reads this
    embedding    VECTOR(768) NOT NULL
);

-- Korean morphology. The same Lucene Nori analyzer an Elasticsearch cluster
-- would use, so "육아휴직을" matches the noun "육아휴직" — without it, a
-- particle-inflected query misses the article that answers it.
CREATE INDEX IF NOT EXISTS ix_gen1_fts
    ON doc_vectors_gen1 USING FTS (body) WITH (lang = 'korean');

-- Metric must match the connection's declared distance; a cosine index queried
-- with L2 returns plausible neighbours that are simply wrong.
CREATE INDEX IF NOT EXISTS ix_gen1_ann
    ON doc_vectors_gen1 USING HNSW (embedding) WITH (metric = 'cosine', m = 16, ef_construction = 200);

-- The temporal predicate runs on every search, so it gets its own index rather
-- than riding along as a filter over the whole table. Single-column: NeorunBase
-- indexes one column, and effective_from is the selective half — effective_to is
-- NULL for every currently-valid row, which is most of the table.
CREATE INDEX IF NOT EXISTS ix_gen1_effective
    ON doc_vectors_gen1 (effective_from);


-- ── Relation graph ───────────────────────────────────────────────────────────
-- Parsed, not inferred. Depth varies per document, so "every regulation this one
-- derives its authority from" is a traversal and cannot be written as a join
-- with a fixed number of levels.
--
-- The columns are the traversal's, not ours: GRAPH_NEIGHBORS expands a frontier
-- with `SELECT DISTINCT dst_id FROM <edges> WHERE src_id IN (…) AND rel_type = ?`,
-- and its seed is a number. So documents carry a numeric id here even though
-- doc_no is what a person would name — the readable identifier rides along for
-- the join back.
CREATE TABLE IF NOT EXISTS doc_edges (
    edge_pk     BIGINT PRIMARY KEY,
    src_id      BIGINT NOT NULL,
    dst_id      BIGINT NOT NULL,
    rel_type    TEXT NOT NULL,
    src_doc_no  TEXT NOT NULL,
    dst_doc_no  TEXT NOT NULL,
    src_version INT,
    dst_version INT,
    src_article TEXT,
    source      TEXT NOT NULL,
    confidence  REAL NOT NULL
);

CREATE INDEX IF NOT EXISTS ix_edges_src ON doc_edges (src_id);
CREATE INDEX IF NOT EXISTS ix_edges_dst ON doc_edges (dst_id);

-- Documents as graph nodes, so a traversal can return something readable
-- without a second round trip to the ledger.
CREATE TABLE IF NOT EXISTS doc_nodes (
    doc_id      BIGINT PRIMARY KEY,
    doc_no      TEXT NOT NULL,
    title       TEXT NOT NULL,
    tier        INT  NOT NULL,
    owner_dept  TEXT NOT NULL,
    is_official BOOLEAN NOT NULL,
    sensitivity TEXT NOT NULL
);
```


!!! note "Why the effective dates are denormalised into the vector table"
    Because the temporal predicate has to be evaluable **inside** the search.
    Filtering a top-k result set afterwards returns fewer than k rows, and
    sometimes none. That is a correctness problem, not a performance one.

---

## 3. The semantic views

This is the surface questions bind to. The temporal filter lives here, and
superseded text is **absent** — not ranked lower, not present at all.

**`demo/schema/semantic/01_effective.sql`**

```sql
-- ============================================================================
-- Temporal correctness — the demo's central claim, expressed as a view
--
-- Ranking cannot make this safe. A superseded regulation that is a better
-- lexical and semantic match for "육아휴직 며칠?" than anything else will win a
-- ranked contest, and no amount of recency weighting reliably beats a document
-- that says exactly what was asked. The only durable answer is that the expired
-- version is not a candidate.
--
-- So the retriever is bound to this view rather than to doc_chunks. Expired text
-- is not demoted; it is absent.
-- ============================================================================

CREATE SCHEMA IF NOT EXISTS semantic.reg;

-- ── Currently effective chunks ───────────────────────────────────────────────
CREATE OR REPLACE VIEW semantic.reg.effective_chunks AS
SELECT
    c.chunk_id,
    c.doc_no,
    c.version,
    c.ordinal_no,
    c.article_no,
    c.article_title,
    c.text,
    c.text_redacted,
    c.pii_types,
    c.page_from,
    d.title           AS doc_title,
    d.doc_class,
    d.tier,
    d.owner_dept,
    d.is_official,
    d.sensitivity,
    v.effective_from,
    v.effective_to,
    v.date_mismatch
FROM ice.reg.doc_chunks c
JOIN ice.reg.doc_versions v
       ON c.doc_no = v.doc_no AND c.version = v.version
JOIN ice.reg.documents d
       ON d.doc_no = c.doc_no
WHERE v.status = 'EFFECTIVE'
  -- effective_from is the approval date, not the 부칙 the document prints. A
  -- version approved on 2025-03-15 whose text claims 2025-01-01 was not in
  -- force in January, and reading the document body gets that ten-week window
  -- wrong in a way nothing downstream can detect.
  AND v.effective_from <= CURRENT_DATE
  AND (v.effective_to IS NULL OR v.effective_to > CURRENT_DATE);


-- ── Point-in-time ────────────────────────────────────────────────────────────
-- "2025년 2월 기준으로는 며칠이었나?" is a real question — disputes and audits are
-- always about a past date. A view cannot take a parameter, so the as-of form
-- lives in the retriever template, which binds :as_of. Kept here as the
-- canonical predicate so the template and this view cannot drift apart.
--
--   WHERE v.effective_from <= :as_of
--     AND (v.effective_to IS NULL OR v.effective_to > :as_of)


-- ── What an agent may cite ───────────────────────────────────────────────────
-- The minimal curation: HR marks the few dozen documents
-- that may serve as grounds for an answer. Without it, a meeting note that
-- mentions leave competes with the regulation that defines it — and on vector
-- similarity alone it sometimes wins.
CREATE OR REPLACE VIEW semantic.reg.citable_chunks AS
SELECT * FROM semantic.reg.effective_chunks
WHERE is_official = TRUE;


-- ── Version history, for tracing rather than answering ───────────────────────
-- Superseded text still has to be reachable when someone asks what changed. It
-- is exposed separately and deliberately not what the retriever reads, so
-- reaching it is an explicit act rather than an accident of ranking.
CREATE OR REPLACE VIEW semantic.reg.version_history AS
SELECT
    d.doc_no,
    d.title,
    v.version,
    v.status,
    v.effective_from,
    v.effective_to,
    v.stated_from,
    v.date_mismatch,
    v.approval_id,
    -- The policy on semantic.reg.* filters on sensitivity, and a row filter is
    -- applied to whichever view the caller named. A view in this namespace that
    -- cannot answer "is this restricted?" makes every query against it fail with
    -- "Column 'sensitivity' not found" — which points at the view rather than at
    -- the policy that asked for it. Every view here carries it.
    d.sensitivity
FROM ice.reg.doc_versions v
JOIN ice.reg.documents d ON d.doc_no = v.doc_no;


-- ── Rows a human has to look at ──────────────────────────────────────────────
-- Where the approval record and the document body disagree. Surfaced as a view
-- rather than a log line because it is a standing queue, not an event.
CREATE OR REPLACE VIEW semantic.reg.date_conflicts AS
SELECT doc_no, title, version, effective_from AS approved_on, stated_from AS document_claims,
       effective_from - stated_from AS drift_days, sensitivity
FROM semantic.reg.version_history
WHERE date_mismatch = TRUE
ORDER BY ABS(effective_from - stated_from) DESC;


-- ── Document number → graph vertex id ──────────────────────────────────────
-- GRAPH_NEIGHBORS starts from a numeric seed, and the person asking has no reason
-- to know that number. So the tool looks it up by document number — but that
-- lookup pointed straight at nb.public.doc_nodes, the one thing the agent read
-- from outside the semantic layer, and therefore a path no policy reached.
CREATE OR REPLACE VIEW semantic.reg.doc_index AS
SELECT doc_id, doc_no, title, tier, owner_dept, is_official, sensitivity
FROM nb.public.doc_nodes;
```


The ERP-side views. This is where federated queries join documents to records.

**`demo/schema/semantic/02_erp.sql`**

```sql
-- ============================================================================
-- ERP semantic layer — making an ERP legible
--
-- The source is ice.erp.* — not Postgres itself but the Iceberg tables that
-- Postgres feeds by CDC (infra/cdc.sh). The views hide the source, so swapping
-- what is underneath left the policies and the tools untouched. The reasons not
-- to connect directly go wider than load: analytical queries disturb the OLTP
-- system, every worker opens a connection, and above all an overwritten value
-- cannot be recovered — there is nowhere left to ask "how many was it then".
--
-- The source columns are lv_typ_cd, emp_sts_cd, grd_cd. An agent cannot read
-- those, and asking an LLM to guess what G4 means is how a plausible wrong
-- answer gets produced. Translating them is the semantic layer's job, and it is
-- also where the join to the regulations happens: entitlement comes from the
-- regulation, usage comes from the ERP, and neither alone answers the question.
-- ============================================================================

-- Korean column aliases are double-quoted throughout. Ontul's SQL lexer rejects
-- a bare non-ASCII identifier with a lexical error at the character position,
-- which points at the column and says nothing about quoting — so this is worth
-- stating once here rather than rediscovering per view.

CREATE SCHEMA IF NOT EXISTS semantic.hr;

CREATE OR REPLACE VIEW semantic.hr.employees AS
SELECT
    e.emp_no                         AS "사번",
    e.emp_nm                         AS "성명",
    o.dept_nm                        AS "부서",
    e.dept_cd                        AS "부서코드",
    CASE e.grd_cd WHEN 'G1' THEN '사원' WHEN 'G2' THEN '대리' WHEN 'G3' THEN '과장'
                  WHEN 'G4' THEN '차장' WHEN 'G5' THEN '부장' WHEN 'G6' THEN '이사'
    END                              AS "직급",
    e.grd_cd,
    e.hire_dt                        AS "입사일",
    CASE e.emp_sts_cd WHEN 'A' THEN '재직' WHEN 'L' THEN '휴직' WHEN 'T' THEN '퇴직'
    END                              AS "재직상태",
    e.mobile_no                      AS "연락처",
    e.rrn                            AS "주민등록번호"
FROM ice.erp.hr_employee e
JOIN ice.erp.hr_org o ON o.dept_cd = e.dept_cd;


-- ── Leave balance, joined to the regulation that grants it ───────────────────
-- The entitlement column comes from the *effective* regulation, not from the
-- ERP's own grant_days. When a regulation is revised the ERP is updated on its
-- own schedule, and until it catches up the two disagree — in which case the
-- regulation is what is true and the ERP row is stale.
CREATE OR REPLACE VIEW semantic.hr.leave_balance AS
SELECT
    b.emp_no          AS "사번",
    e.emp_nm          AS "성명",
    o.dept_nm         AS "부서",
    b.yr              AS "연도",
    CASE b.lv_typ_cd WHEN 'ANN' THEN '연차' WHEN 'CHC' THEN '육아휴직'
                     WHEN 'SIC' THEN '병가' WHEN 'CON' THEN '경조사'
    END               AS "휴가종류",
    b.lv_typ_cd,
    b.grant_days      AS "부여일수_ERP",
    b.used_days       AS "사용일수",
    b.grant_days - b.used_days AS "잔여일수_ERP"
FROM ice.erp.hr_leave_balance b
JOIN ice.erp.hr_employee e ON e.emp_no = b.emp_no
JOIN ice.erp.hr_org o      ON o.dept_cd = e.dept_cd;


-- ── Expenses against the cap the guideline sets ──────────────────────────────
CREATE OR REPLACE VIEW semantic.hr.expenses AS
SELECT
    x.exp_id      AS "전표번호",
    x.emp_no      AS "사번",
    e.emp_nm      AS "성명",
    o.dept_nm     AS "부서",
    x.exp_dt      AS "사용일",
    CASE x.exp_typ_cd WHEN 'LODG' THEN '숙박비' WHEN 'TRNS' THEN '교통비'
                      WHEN 'MEAL' THEN '식비' END AS "비목",
    x.amt         AS "금액",
    x.nights      AS "숙박일수",
    CASE WHEN x.nights > 0 THEN x.amt / x.nights END AS "1박당금액"
FROM ice.erp.fi_expense x
JOIN ice.erp.hr_employee e ON e.emp_no = x.emp_no
JOIN ice.erp.hr_org o      ON o.dept_cd = e.dept_cd;


-- ── Purchase orders with the approver's ceiling attached ─────────────────────
-- The ceiling is a fact of the purchasing regulation; keeping it in the view means "was this
-- approved by someone entitled to?" is a comparison rather than an inference.
CREATE OR REPLACE VIEW semantic.hr.purchase_orders AS
SELECT
    p.po_id        AS "발주번호",
    p.req_emp_no   AS "요청자사번",
    r.emp_nm       AS "요청자",
    p.apr_emp_no   AS "승인자사번",
    a.emp_nm       AS "승인자",
    CASE a.grd_cd WHEN 'G1' THEN '사원' WHEN 'G2' THEN '대리' WHEN 'G3' THEN '과장'
                  WHEN 'G4' THEN '차장' WHEN 'G5' THEN '부장' WHEN 'G6' THEN '이사'
    END            AS "승인자직급",
    p.po_dt        AS "발주일",
    p.amt          AS "금액",
    CASE a.grd_cd WHEN 'G3' THEN 5000000 WHEN 'G4' THEN 20000000
                  WHEN 'G5' THEN 50000000 WHEN 'G6' THEN 300000000 ELSE 1000000
    END            AS "승인한도"
FROM ice.erp.pu_purchase_order p
JOIN ice.erp.hr_employee r ON r.emp_no = p.req_emp_no
JOIN ice.erp.hr_employee a ON a.emp_no = p.apr_emp_no;


-- ── Revisions in approval, over the stream table an Ontul Flow fills ────────
-- It sits alongside the ledger views for one reason: authorization. Every read the
-- agent makes goes through the semantic layer, and that is where the policies
-- attach. Reading the stream table directly would leave exactly one place outside
-- the rules.
CREATE SCHEMA IF NOT EXISTS semantic.reg;

CREATE OR REPLACE VIEW semantic.reg.pending_revisions AS
SELECT
    a.approval_id   AS "결재번호",
    a.doc_no        AS "문서번호",
    d.title         AS "제목",
    -- The CAST is explained in
    -- tests/known-issues/row-filter-changes-a-column-type.md. In short: this column
    -- is declared INTEGER and re-derived as BIGINT through the join, and normally
    -- only one of those two paths runs so the disagreement never shows. The moment
    -- an IAM row filter wraps the view in a derived table, both run and the planner
    -- rejects the query for not preserving datatypes — a 400 that only the caller
    -- with **less** access sees, whose message points at the view rather than at
    -- the policy.
    CAST(a.version AS INTEGER) AS "개정차수",
    CASE a.step WHEN 'DRAFT' THEN '기안' WHEN 'REVIEW' THEN '검토'
                WHEN 'APPROVED' THEN '승인' WHEN 'EFFECTIVE' THEN '시행'
                WHEN 'REJECTED' THEN '반려' END AS "단계",
    a.step          AS step,
    a.summary       AS "요지",
    a.expected_from AS "예정시행일",
    a.drafter       AS "기안자사번",
    a.owner_dept    AS "소관부서",
    a.updated_at    AS "갱신시각",
    -- The policy puts a sensitivity condition on all of semantic.reg.*. A revision
    -- of a restricted document must not be visible to an ordinary employee either,
    -- so the view has to provide somewhere for that condition to attach. An
    -- approval for a document not yet in the ledger is treated as INTERNAL: with
    -- NULL the condition evaluates UNKNOWN and the row disappears silently.
    COALESCE(d.sensitivity, 'INTERNAL') AS sensitivity
FROM ice.reg.approval_status a
LEFT JOIN ice.reg.documents d ON d.doc_no = a.doc_no;
```


Semantic views are registered as **definitions** rather than created as
engine-level views: they carry certification status and mandatory filters, and
plain DDL has nowhere to put those. The `.sql` files stay the source — a reviewer
should be able to read the temporal predicate without decoding JSON — and the
script below does the translation.

**`demo/infra/semantic_views.py`**

```python
"""
Turn the semantic .sql files into semantic-view registrations.

The views are written as SQL because that is what they are, and because a
reviewer should be able to read the temporal predicate without decoding JSON.
Ontul stores them as definitions rather than as engine-level views — a semantic
view carries a certification status, mandatory filters and metric definitions
that plain DDL has nowhere to put — so the file is the source and this is the
translation.

    python3 semantic_views.py schema/semantic/01_effective.sql
"""
from __future__ import annotations

import json
import re
import sys
from pathlib import Path

# CREATE [OR REPLACE] VIEW <catalog>.<schema>.<name> AS <body> ;
VIEW = re.compile(
    r"CREATE\s+(?:OR\s+REPLACE\s+)?VIEW\s+([A-Za-z_][\w]*)\.([A-Za-z_][\w]*)\.([A-Za-z_][\w]*)\s+AS\s+(.*?);",
    re.S | re.I,
)


def strip_comments(sql: str) -> str:
    # Line comments only; nothing here uses /* */. Keeping them would put a
    # leading "--" comment inside baseSql, where it comments out the SELECT that
    # follows it on the same stored line.
    return re.sub(r"--[^\n]*", "", sql)


def parse(path: Path) -> list[dict]:
    src = strip_comments(path.read_text(encoding="utf-8"))
    out = []
    for catalog, schema, name, body in VIEW.findall(src):
        out.append({
            "catalog": catalog,
            "schema": schema,
            "name": name,
            "baseSql": " ".join(body.split()),
            "description": f"{path.name} 에서 정의됨",
        })
    return out


def main(argv: list[str]) -> int:
    if len(argv) < 2:
        print(__doc__, file=sys.stderr)
        return 2
    views = []
    for arg in argv[1:]:
        views.extend(parse(Path(arg)))
    if not views:
        print("no CREATE VIEW statements found", file=sys.stderr)
        return 1
    json.dump(views, sys.stdout, ensure_ascii=False)
    return 0


if __name__ == "__main__":
    sys.exit(main(sys.argv))
```


---

## 4. Connections

Credentials are registered once as a connection; catalogs and jobs then reference
them **by id**. The embedding connection matters most: the vector space is
defined in exactly one place, which is what makes a stored vector and a query
vector comparable.

**`demo/schema/connections/01_embedding.json`**

```json
{
  "_comment": "The vector space, defined once. Both sides of retrieval — the batch job that embeds 6,000 chunks and the query that embeds one question — reach the model only through this connection, so neither can pick a different one. That is the whole point: a stored vector and a query vector are comparable because the same connection produced them, not because someone remembered to configure two jobs identically.",
  "connectionId": "emb_main",
  "type": "EMBEDDING",
  "description": "multilingual-e5-base, pinned, served in-network",
  "properties": {
    "endpoint": "http://embed-svc:8000",
    "modality": "text",
    "_model_comment": "The revision is the commit, not the tag. 'intfloat/multilingual-e5-base' is a moving target; d1287505… is the weights that produced every vector in generation 1. verify=strict makes the endpoint prove it loaded exactly this before it is allowed to serve.",
    "model": "intfloat/multilingual-e5-base",
    "revision": "d128750597153bb5987e10b1c3493a34e5a4502a",
    "verify": "strict",
    "dim": "768",
    "normalize": "true",
    "distance": "cosine",
    "_asymmetry_comment": "e5 is asymmetric: the same sentence embeds differently as stored text than as a search string. Getting this backwards does not error — it quietly returns worse results, which is the failure mode worth spending a config field on.",
    "asymmetry": "prefix",
    "prefix.passage": "passage: ",
    "prefix.query": "query: ",
    "prefixScheme": "e5v1",
    "dialect": "ontul",
    "batchSize": "64",
    "timeoutMs": "180000",
    "_timeout_comment": "A batch is 64 chunks, and e5-base on CPU takes several seconds for one of them. With the CDC and graph Flows sharing the machine, 30 seconds is not enough — the result is 'request timed out', and what fails is not the embedding service but the whole indexing job. Better to leave headroom: if the model is genuinely dead, that shows up at connection time anyway."
  }
}
```


!!! warning "e5 is asymmetric"
    The same sentence embeds differently as stored text than as a search string.
    Getting it backwards does **not** error — it quietly returns worse results,
    which is the failure mode worth spending a config field on.

**`demo/schema/connections/02_erp.json`**

```json
{
  "_comment": "ERP, read live rather than copied into the lake. A leave balance is only worth quoting if it is the balance right now — an overnight snapshot would let the agent state a number that was true yesterday with the same confidence as one that is true today.",
  "connectionId": "erp",
  "type": "JDBC",
  "description": "ERP (PostgreSQL) — HR, attendance and expenses, federated",
  "properties": {
    "url": "jdbc:postgresql://regdemo-erp:5432/erp",
    "driver": "org.postgresql.Driver",
    "user": "erp",
    "password": "${ERP_PASSWORD}",
    "pool.maxSize": "8"
  }
}
```
**`demo/schema/connections/03_groupware.json`**

```json
{
  "_comment": "The approval system. Authoritative for when a regulation actually took effect: the approval record is what makes a rule binding, and it routinely disagrees with the date printed in the document's 부칙. HR-REG-003 v3 states 2025-01-01 and was approved 2025-03-15 — ten weeks in which citing the document would have been wrong.",
  "connectionId": "groupware",
  "type": "JDBC",
  "description": "The approval system (MySQL) — approval history, and the CDC source",
  "properties": {
    "url": "jdbc:mysql://regdemo-groupware:3306/groupware?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=Asia/Seoul",
    "driver": "com.mysql.cj.jdbc.Driver",
    "user": "gw",
    "password": "${GW_PASSWORD}",
    "pool.maxSize": "8"
  }
}
```
**`demo/schema/connections/04_lms.json`**

```json
{
  "_comment": "The training system has an API and no database. Not every source worth joining will hand you a JDBC URL, and the ones that will not are usually the ones holding the compliance evidence.",
  "connectionId": "lms",
  "type": "REST",
  "description": "Training records (a SaaS LMS) — reached through a rest-operation",
  "properties": {
    "baseUrl": "http://regdemo-lms:8000",
    "authType": "none",
    "timeoutMs": "10000"
  }
}
```


---

## 5. The registration script

Catalogs, connections, schema, IAM, semantic views, retrievers and the ontology.
Idempotent: running it again re-applies the same definitions rather than
duplicating them.

**`demo/infra/register.sh`**

```bash
#!/usr/bin/env bash
##
## Register everything Ontul needs to answer a question about a regulation:
## catalogs, connections, Iceberg and NeorunBase schema, semantic views, IAM,
## and the retrievers the agent is allowed to call.
##
## Runs after infra/up.sh, which wrote out/stack.env with the credentials this
## bring-up minted. Idempotent — re-running it re-applies the same definitions
## rather than duplicating them, so a failed stage can be fixed and repeated.
##
set -euo pipefail

DEMO="$(cd "$(dirname "$0")/.." && pwd)"
source "$DEMO/out/stack.env"

ONTUL="${ONTUL_URL:-http://localhost:8080}"
ADMIN_USER=admin
ADMIN_PW_INITIAL=admin
ADMIN_PW="${ONTUL_ADMIN_PASSWORD:-regdemo-admin-2026}"

log()  { printf '\n\033[1m=== %s ===\033[0m\n' "$*"; }
step() { printf '  %s\n' "$*"; }
fail() { printf '\033[31mFAIL: %s\033[0m\n' "$*" >&2; exit 1; }

# ── Authenticate. A fresh master demands a password change before it will do
#    anything else, which is the correct behaviour and an easy thing to trip on.
# The login response names the JWT 'accessToken'. Reading 'token' returns an
# empty string, and an empty Bearer header fails as "Unauthorized" — which reads
# like a password problem and is not one.
tok() { python3 -c "import sys,json;d=json.load(sys.stdin);print(d.get('accessToken') or d.get('token') or '')" 2>/dev/null; }

login() {
  local pw=$1
  curl -sf -X POST "$ONTUL/admin/auth/login" -H 'Content-Type: application/json' \
    -d "{\"username\":\"$ADMIN_USER\",\"password\":\"$pw\"}" 2>/dev/null
}
log "Authenticating"
RES=$(login "$ADMIN_PW" || true)
TOKEN=$(printf '%s' "$RES" | tok || true)
if [ -z "$TOKEN" ]; then
  RES=$(login "$ADMIN_PW_INITIAL") || fail "cannot log in with either the initial or the demo password"
  TOKEN=$(printf '%s' "$RES" | tok)
  NEED=$(printf '%s' "$RES" | python3 -c "import sys,json;print(json.load(sys.stdin).get('requirePasswordChange',False))")
  if [ "$NEED" = "True" ] || [ "$NEED" = "true" ]; then
    step "rotating the default password"
    curl -sf -X POST "$ONTUL/admin/auth/change-password" \
      -H 'Content-Type: application/json' -H "Authorization: Bearer $TOKEN" \
      -d "{\"oldPassword\":\"$ADMIN_PW_INITIAL\",\"newPassword\":\"$ADMIN_PW\"}" >/dev/null
    TOKEN=$(login "$ADMIN_PW" | tok)
  fi
fi
[ -n "$TOKEN" ] || fail "no token"
AUTH=(-H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json')
step "authenticated"

post() {  # post <path> <json> <label>
  local out code
  out=$(curl -s -o /tmp/regdemo-post.out -w '%{http_code}' -X POST "$ONTUL$1" "${AUTH[@]}" -d "$2")
  code="$out"
  # "already exists" arrives as 400 rather than 409, so the status code alone
  # cannot distinguish a re-run from a real failure. This script promises to be
  # idempotent; treating that message as success is what keeps the promise.
  case "$code" in
    2*)  step "$3" ;;
    409) step "$3 (already present)" ;;
    *)   if grep -qi 'already exists' /tmp/regdemo-post.out; then
           step "$3 (already present)"
         else
           printf '\033[31m  %s -> HTTP %s: %s\033[0m\n' "$3" "$code" "$(head -c 300 /tmp/regdemo-post.out)" >&2
           return 1
         fi ;;
  esac
}

# /admin/query/execute answers 200 for a failed statement and puts the failure in
# the body as status:"error". Checking only the HTTP code makes a schema stage
# report every statement as applied while none of them were — which is worse
# than failing, because the next stage then fails somewhere unrelated.
sql() {  # sql <statement> — prints "ok" or "error"; body left in /tmp/regdemo-sql.out
  local body
  body=$(python3 -c "import json,sys;print(json.dumps({'sql':sys.stdin.read()}))" <<<"$1")
  curl -s -o /tmp/regdemo-sql.out -w '%{http_code}' -X POST "$ONTUL/admin/query/execute" "${AUTH[@]}" -d "$body" >/tmp/regdemo-sql.code
  python3 -c "
import json,sys
code = open('/tmp/regdemo-sql.code').read().strip()
if not code.startswith('2'):
    print('error'); sys.exit()
try:
    d = json.load(open('/tmp/regdemo-sql.out', encoding='utf-8'))
except Exception:
    print('ok'); sys.exit()
print('error' if d.get('status') == 'error' or d.get('error') else 'ok')
"
}

sqlerr() { python3 -c "
import json
try:
    d = json.load(open('/tmp/regdemo-sql.out', encoding='utf-8'))
    print((d.get('error') or d.get('message') or '')[:240])
except Exception:
    print(open('/tmp/regdemo-sql.out', encoding='utf-8').read()[:240])
"; }

run_sql_file() {  # run_sql_file <path> — statements split on ';' at line ends
  local f=$1 stmt code n=0
  step "$(basename "$f")"
  while IFS= read -r stmt; do
    [ -z "${stmt// }" ] && continue
    if [ "$(sql "$stmt")" = "ok" ]; then
      n=$((n+1))
    else
      printf '\033[31m    failed: %s\n      %s\033[0m\n' "$(sqlerr)" "$(head -c 140 <<<"$stmt")" >&2
      return 1
    fi
  done < <(python3 - "$f" <<'PY'
import re,sys
src = open(sys.argv[1], encoding='utf-8').read()
# Strip line comments, then split on semicolons. Statements are written one per
# ';' in these files; nothing here embeds a semicolon in a string literal.
src = re.sub(r'--[^\n]*', '', src)
for s in src.split(';'):
    s = ' '.join(s.split())
    if s: print(s)
PY
)
  step "  $n statements"
}

# ── 1. Catalogs. The Iceberg warehouse, and NeorunBase as the serving engine.
#
# The connector goes inside "config" as the "connector" key, not beside the name
# as a "type" field. Getting that wrong is silent: the catalog registers, and
# every table in it resolves through the default file connector, which then
# rejects CREATE SCHEMA with a message about FileConnector that has nothing to
# do with what was actually misconfigured.
log "1/6  Catalogs"
# Property names are the connector's, not Iceberg's own and not NeorunBase's.
# IcebergCatalogProps reads catalog.rest.client_id / client_secret and builds the
# OAuth2 credential from them; passing client-id / client-secret instead leaves
# the REST client anonymous, and Polaris rejects the very first call — GET
# /v1/config — with a NotAuthorizedException whose message is empty. The failure
# names neither the missing property nor the catalog.
#
# 'warehouse' also has to be the Polaris catalog name, because the connector
# derives the REST 'prefix' from it. A warehouse that is an S3 URI produces a
# prefix that no catalog answers to.
post /admin/catalogs "$(cat <<JSON
{"name":"ice","config":{
   "connector":"iceberg",
   "catalog.rest.uri":"$POLARIS_URI_INTERNAL",
   "warehouse":"$POLARIS_CATALOG",
   "catalog.rest.client_id":"$POLARIS_CLIENT_ID",
   "catalog.rest.client_secret":"$POLARIS_CLIENT_SECRET",
   "catalog.rest.scope":"PRINCIPAL_ROLE:ALL",
   "catalog.rest.flavor":"polaris",
   "header.Polaris-Realm":"POLARIS",
   "s3.endpoint":"$S3_ENDPOINT_INTERNAL",
   "s3.accessKey":"$S3_ACCESS_KEY",
   "s3.secretKey":"$S3_SECRET_KEY",
   "s3.region":"$S3_REGION",
   "s3.pathStyle":"true"}}
JSON
)" "ice (iceberg via polaris)"

# The NeorunBase catalog is not registered here. The connector captures the table
# list at the moment of registration, and on a first install NeorunBase has no
# tables yet. Registering after the schema exists (end of step 3) is what makes
# them visible.

# ── 2. Connections. The embedding connection is the one that matters most:
#    it is the single definition of the vector space, and both indexing and
#    search reach the model only through it.
log "2/6  Connections"
for f in "$DEMO"/schema/connections/*.json; do
  [ -f "$f" ] || continue
  body=$(python3 - "$f" <<'PY'
import json,sys,os,re
d = json.load(open(sys.argv[1], encoding='utf-8'))
d = {k: v for k, v in d.items() if not k.startswith('_')}
missing = []
def _one(m):
    v = os.environ.get(m.group(1))
    if v is None:
        # Leaving the placeholder in place stores a literal "${ERP_PASSWORD}" as
        # the password. The connection then registers cleanly and fails later at
        # "Failed to connect", which points at the network rather than at the
        # variable that was never set.
        missing.append(m.group(1))
        return m.group(0)
    return v
def sub(x):
    if isinstance(x, str):
        return re.sub(r'\$\{(\w+)\}', _one, x)
    if isinstance(x, dict):  return {k: sub(v) for k, v in x.items()}
    if isinstance(x, list):  return [sub(v) for v in x]
    return x
out = sub(d)
if missing:
    sys.stderr.write('unset variable(s) referenced by %s: %s\n' % (sys.argv[1], ', '.join(sorted(set(missing)))))
    sys.exit(3)
print(json.dumps(out))
PY
)
  [ -n "$body" ] || fail "could not build a body for $(basename "$f") — see the error above"
  cid=$(python3 -c "import json,sys;print(json.loads(sys.argv[1])['connectionId'])" "$body")
  # Deleted and recreated if it already exists. `post` treats "already exists" as
  # success, and that exists to make re-runs safe — not to **ignore a changed
  # definition**. Edit a timeout or a model revision in the file, run this again,
  # and if nothing happens the person who edited it believes it took effect when
  # it did not.
  curl -s -X DELETE "$ONTUL/admin/connections/$cid" "${AUTH[@]}" -o /dev/null
  post /admin/connections "$body" "$(basename "$f" .json)"
done

# Now the federated catalogs, which had to wait: each names a connectionId
# and has nothing to resolve until that connection exists.
# Each one resolves its credentials each one resolving its credentials
# through the connection rather than repeating them here. ERP is federated on
# purpose: a leave balance copied into the lake overnight is a number that was
# true yesterday, quoted with the confidence of one that is true now.
post /admin/catalogs '{"name":"erp","config":{"connector":"jdbc","connectionId":"erp"}}' \
  "erp (postgres, federated)"
post /admin/catalogs '{"name":"gw","config":{"connector":"jdbc","connectionId":"groupware"}}' \
  "gw (mysql — the approval system, the authority on effective dates)"

# ── 3. Schema. Iceberg first — the semantic views read from it, and a view over
#    a table that does not exist yet fails at definition time, not at query time.
log "3/6  Schema"
for f in "$DEMO"/schema/iceberg/*.sql; do
  [ -f "$f" ] || continue
  run_sql_file "$f"
done

# NeorunBase DDL goes straight to NeorunBase, over its PostgreSQL wire.
#
# Not a routing preference — VECTOR(768), USING HNSW and USING FTS are
# NeorunBase's own dialect, and Ontul's planner has no reason to know them. Sent
# through Ontul they fail on the table name before the parser ever reaches the
# interesting part. Ontul reads these tables through the 'nb' catalog; it does
# not define them.
NB_PG_PORT="${NB_PG_PORT:-$(printf '%s' "${NEORUNBASE_PG:-}" | sed -n 's/.*:\([0-9][0-9]*\)\/.*/\1/p')}"
NB_PG_PORT="${NB_PG_PORT:-5434}"
step "01_vectors.sql → neorunbase :$NB_PG_PORT"
if ! command -v psql >/dev/null 2>&1; then
  fail "psql not found — needed to apply the NeorunBase schema (brew install libpq)"
fi
# Comments are stripped on the way out, not removed from the file. NeorunBase's
# parser reads a `--` comment inside a CREATE TABLE body as part of the column
# definition and reports "Unknown type: DENORMALISED" — a word that appears only
# in a comment. The explanation of why a column is denormalised is worth keeping;
# it just cannot travel with the statement.
python3 - "$DEMO/schema/neorunbase/01_vectors.sql" <<'STRIP' > /tmp/regdemo-nb-schema.sql
import re, sys
src = open(sys.argv[1], encoding='utf-8').read()
sys.stdout.write(re.sub(r'--[^\n]*', '', src))
STRIP
PGPASSWORD="${NEORUNBASE_PASSWORD:-Regdemo12345}" psql -v ON_ERROR_STOP=1 -q \
  -h localhost -p "$NB_PG_PORT" -U admin -d neorunbase \
  -f /tmp/regdemo-nb-schema.sql || fail "neorunbase schema failed"
step "  applied"

# The tables exist now, so the catalog can be registered.
#
# jdbcUrl, not host/port/database. The connector reads 'jdbcUrl' or 'endpoint'
# and nothing else — given neither it registers happily and then discovers zero
# tables, so every query against it fails with "Object 'nb' not found" rather than
# with anything about configuration. preferQueryMode=simple because NeorunBase
# serves the simple query protocol.
post /admin/catalogs "$(cat <<JSON
{"name":"nb","config":{"connector":"neorunbase",
   "jdbcUrl":"jdbc:postgresql://$NEORUNBASE_INTERNAL_HOST:5432/neorunbase?preferQueryMode=simple",
   "endpoint":"http://$NEORUNBASE_INTERNAL_HOST:8080",
   "username":"admin","password":"$NEORUNBASE_PASSWORD","schema":"public"}}
JSON
)" "nb (neorunbase — vectors + korean fts)"

# A catalog that registered but resolved no tables is not registered in any sense
# that matters. Stopping here turns a silent misconfiguration into a failure at
# the point it happened, rather than an unrelated error much later.
NBT=$(curl -sf --max-time 20 "$ONTUL/admin/catalogs" "${AUTH[@]}" | python3 -c "
import sys, json
print(next((c.get('tableCount', 0) for c in json.load(sys.stdin) if c.get('name') == 'nb'), 0))
" 2>/dev/null || echo 0)
[ "${NBT:-0}" -gt 0 ] || fail "the 'nb' catalog resolved $NBT tables — check jdbcUrl and the NeorunBase password"
step "  nb sees $NBT tables"


# ── 4. IAM. Policies, then the groups that carry them, then the demo users.
log "4/6  IAM"
for f in "$DEMO"/schema/iam/*.json; do
  [ -f "$f" ] || continue
  name=$(basename "$f" .json | sed 's/^[0-9]*_//')
  # 'document' is the policy object itself, not a string containing it — the
  # handler calls .toString() on the node, so a quoted string would be stored
  # with its own escapes baked in.
  doc=$(python3 -c "
import json,sys
d=json.load(open(sys.argv[1],encoding='utf-8'))
d={k:v for k,v in d.items() if not k.startswith('_')}
if 'Statement' in d:
    d['Statement']=[{k:v for k,v in s.items() if not k.startswith('_')} for s in d['Statement']]
print(json.dumps({'name':sys.argv[2],'document':d}, ensure_ascii=False))
" "$f" "$name")
  post /admin/iam/policies "$doc" "policy $name"
done

# ── 4b. The people the policies are about.
#
# A policy attached to nobody is not access control. Every query was running as
# admin — which holds AdministratorAccess — so the row filters and column rules
# were registered, correct, and never consulted. Nothing failed; the answers were
# simply complete.
#
# Attributes matter as much as membership: a filter of
# "사번" = '${user.attr.emp_no}' resolves against the *user record*, so without
# emp_no on the user it expands to an empty string and matches nothing — which
# reads as "this employee has no records" rather than as "this is misconfigured".
log "4b/6  IAM principals"
DEMO_EMP=$(python3 -c "
import json; print(json.load(open('$DEMO/out/ground_truth.json', encoding='utf-8'))['erp']['demo_employee']['emp_no'])")

iam() { curl -s -o /tmp/regdemo-iam.out -w '%{http_code}' -X POST "$ONTUL$1" "${AUTH[@]}" -d "$2"; }
iam_ok() {  # iam_ok <path> <json> <label>
  code=$(iam "$1" "$2")
  case "$code" in
    2*) step "$3" ;;
    *)  if grep -qiE 'already exists' /tmp/regdemo-iam.out; then step "$3 (already present)";
        else fail "$3 -> HTTP $code: $(head -c 160 /tmp/regdemo-iam.out)"; fi ;;
  esac
}

# group per policy, so membership is the only thing that varies per person
for pol in agent_caller dept_manager hr_staff indexer; do
  iam_ok /admin/iam/groups "{\"groupName\":\"${pol}_group\"}" "group ${pol}_group"
  iam_ok /admin/iam/attach-group-policy \
    "{\"groupName\":\"${pol}_group\",\"policyName\":\"${pol}\"}" "  ← policy $pol"
done

# The demo personas. Passwords are fixed and worthless — this is a demo cluster
# whose whole point is showing what each identity may see.
add_person() {  # add_person <username> <group> <emp_no> <dept> <clearance>
  iam_ok /admin/iam/users \
    "{\"username\":\"$1\",\"password\":\"regdemo-2026\",\"attributes\":{\"emp_no\":\"$3\",\"dept\":\"$4\",\"clearance\":\"$5\"}}" \
    "user $1 (emp_no=$3, dept=$4, clearance=$5)"
  # Attributes again for the already-present case: create is a no-op then, and a
  # persona carrying last run's attributes is worse than one carrying none.
  curl -s -o /dev/null -X PUT "$ONTUL/admin/iam/users/$1/attributes" "${AUTH[@]}" \
    -d "{\"attributes\":{\"emp_no\":\"$3\",\"dept\":\"$4\",\"clearance\":\"$5\"}}"
  iam_ok /admin/iam/add-user-to-group "{\"username\":\"$1\",\"groupName\":\"$2\"}" "  ← $2"
}

add_person hong  agent_caller_group  "$DEMO_EMP" DEV none
# A real DEV manager. The department attribute drives the row filter, so a persona
# whose attribute disagrees with their own ERP record would still "work" while
# demonstrating the wrong thing.
add_person park  dept_manager_group  20150010    DEV manager
add_person cho   hr_staff_group      20090001    HR  hr

# ── 5. Semantic views, then the retrievers that bind to them.
#
# The views are stored as definitions rather than created as engine-level views.
# That is not a workaround: a semantic view carries certification status and
# mandatory filters, and those have nowhere to live in plain DDL. The .sql files
# stay the source — a reviewer should be able to read the temporal predicate
# without decoding JSON — and infra/semantic_views.py does the translation.
log "5/6  Semantic views & retrievers"
python3 "$DEMO/infra/semantic_views.py" "$DEMO"/schema/semantic/*.sql > /tmp/regdemo-views.json \
  || fail "could not parse the semantic view definitions"
COUNT=$(python3 -c "import json;print(len(json.load(open('/tmp/regdemo-views.json'))))")
for i in $(seq 0 $((COUNT-1))); do
  V=$(python3 -c "import json,sys;print(json.dumps(json.load(open('/tmp/regdemo-views.json'))[int(sys.argv[1])], ensure_ascii=False))" "$i")
  N=$(python3 -c "import json,sys;d=json.loads(sys.argv[1]);print(d['catalog']+'.'+d['schema']+'.'+d['name'])" "$V")
  post /api/v1/semantic-views "$V" "$N"
done

for f in "$DEMO"/schema/retrievers/*.json; do
  [ -f "$f" ] || continue
  body=$(python3 -c "
import json,sys
d=json.load(open(sys.argv[1],encoding='utf-8'))
print(json.dumps({k:v for k,v in d.items() if not k.startswith('_')}))
" "$f")
  post /api/v1/retrievers "$body" "$(basename "$f" .json)"
done

# ── The ontology. Objects → links → actions is mandatory: a link needs both
#    endpoint object types to exist, and an action needs the object type it
#    operates on. The numbers in the filenames are that order.
log "5/6  Ontology (objects · links · actions)"
for f in "$DEMO"/schema/ontology/*.json; do
  [ -f "$f" ] || continue
  body=$(python3 -c "
import json,sys
d=json.load(open(sys.argv[1],encoding='utf-8'))
print(json.dumps({k:v for k,v in d.items() if not k.startswith('_')}, ensure_ascii=False))
" "$f")
  case "$(basename "$f")" in
    0*) ep=/api/v1/object-types ;;
    1*) ep=/api/v1/link-types ;;
    2*) ep=/api/v1/action-types ;;
    *)  continue ;;
  esac
  post "$ep" "$body" "$(basename "$f" .json)"
done

# ── 6. Prove the embedding connection round-trips before anything depends on it.
log "6/6  Verifying the vector space"
if [ "$(sql "SELECT embed_query('emb_main', '연차 며칠이야') AS v")" = "ok" ]; then
  # The vector arrives as a rendered string, not a JSON array, so len() on the
  # cell counts characters. Parse it — a check that prints "?" and passes is not
  # a check, and dimension is exactly the thing worth asserting here.
  DIM=$(python3 -c "
import json
d = json.load(open('/tmp/regdemo-sql.out', encoding='utf-8'))
cell = (d.get('rows') or [[None]])[0][0]
if isinstance(cell, list):
    print(len(cell))
elif isinstance(cell, str) and cell.startswith('['):
    print(len([x for x in cell.strip('[]').split(',') if x.strip()]))
else:
    print(0)
")
  [ "$DIM" = "768" ] || fail "embed_query returned a ${DIM}-dim vector; the connection declares 768"
  step "embed_query('emb_main', …) returned a $DIM-dim vector"
  step "generation: $EMBED_FINGERPRINT"
else
  fail "embed_query failed: $(sqlerr)"
fi

log "Registered"
cat <<SUMEOF
  Next:  bash infra/index.sh    (chunk, embed and load the corpus)
         bash tests/e2e.sh      (the 16 scenario cases)
  Admin: $ONTUL   ($ADMIN_USER / $ADMIN_PW)
SUMEOF
```


```bash
bash infra/register.sh
```

### Why this script insists on an order

- **The NeorunBase catalog is registered after its schema exists.** The connector
  captures the table list at registration time. Register first on a fresh install
  and it resolves zero tables, after which every query fails with
  `Object 'nb' not found` — which says nothing about configuration.
- **Connections are deleted and recreated.** Treating "already exists" as success
  exists to make re-runs safe, not to **ignore a changed definition**. Edit a
  timeout or a model revision in the file, re-run, and if nothing happens the
  person who edited it believes it took effect when it did not.
- **The ontology goes objects → links → actions.** A link needs both endpoint
  object types to exist; an action needs the object type it operates on.

---

Next: [the pipeline](pipeline.md) — the distributed jobs that fill this schema.
