# 스키마 — 원장 · 서빙 · 시맨틱

세 층이 있고, 각 층이 다른 질문에 답합니다.

| 층 | 무엇 | 어디 |
|---|---|---|
| **원장** | 기록의 원본. 문서·버전·청크·관계 | Iceberg (Polaris + S3) |
| **서빙** | 다시 만들 수 있는 파생물. 벡터·FTS·그래프 | NeorunBase |
| **시맨틱** | 질문이 실제로 붙는 면. 시간 필터와 정책이 여기 | Ontul 시맨틱 뷰 |

원장과 서빙을 나누는 것이 이 데모의 설계 결정 중 가장 값어치가 큽니다.
NeorunBase 를 통째로 날리고 다시 세워도 잃는 것이 없습니다 — 파이프라인이
Iceberg 에서 다시 만들면 되고, 관계가 언제 어떻게 바뀌었는지는 Iceberg 스냅샷에
남아 있습니다.

---

## 1. Iceberg 원장

### 문서와 버전

**`demo/schema/iceberg/01_documents.sql`**

```sql
-- ============================================================================
-- 원장 (ledger) — 문서 메타와 버전
--
-- 이 파일의 테이블은 "무엇이 사실인가"를 담습니다. 벡터·인덱스·그래프 엣지는
-- 전부 여기서 다시 만들어낼 수 있는 파생물이므로 원장에 두지 않습니다.
-- ============================================================================

CREATE SCHEMA IF NOT EXISTS ice.reg;

-- ── 문서 (버전 무관한 정체성) ────────────────────────────────────────────────
-- doc_no 는 파일명이 아니라 문서 자체의 식별자입니다. 실제 조직에서 파일명은
-- "인사규정_v2_최종_수정.pdf" 처럼 신뢰할 수 없으므로, 매칭은 규정 목록
-- (엑셀 마스터)을 통해 이루어지고 그 결과만 여기에 남습니다.
CREATE TABLE IF NOT EXISTS ice.reg.documents (
    doc_id          BIGINT,     -- 그래프 정점 id. 업무 키는 doc_no 이고 이건 대리키입니다 —
                                -- 그래프 엔진의 정점은 숫자여야 하고, 그 숫자를 그래프를
                                -- 만들 때마다 새로 매기면 어제 만든 엣지를 오늘 해석할 수
                                -- 없습니다. 원장이 부여하면 한 번 정해지고 계속 같습니다.
    doc_no          VARCHAR,    -- HR-REG-003
    title           VARCHAR,
    doc_class       VARCHAR,    -- RULE(취업규칙) / REGULATION(규정) / GUIDELINE(지침)
    tier            INT,        -- 1=최상위 2=규정 3=지침. 문서번호에 인코딩된 값과 일치해야 함
    owner_dept      VARCHAR,    -- 소관 부서 코드 (org.py 의 DEPTS 와 조인)
    is_official     BOOLEAN,    -- 인사팀이 "답변 근거가 될 수 있다"고 지정한 문서
                                -- ↑ 논의의 '최소 큐레이션'. 300명 문서 전부를 믿지 않는다.
    sensitivity     VARCHAR,    -- PUBLIC / INTERNAL / RESTRICTED
                                -- ↑ RESTRICTED 는 IAM 행 필터로 후보군에서 배제
    source_system   VARCHAR,    -- onedrive / groupware / erp_attachment
    created_at      TIMESTAMP,
    updated_at      TIMESTAMP
);

-- ── 버전 (시간 정합성의 심장) ────────────────────────────────────────────────
-- effective_from 은 문서 본문이 아니라 전자결재 승인일에서 옵니다. 본문 부칙의
-- "2025년 1월 1일부터 시행"은 초안 작성 시점에 쓰인 값이고, 결재가 미뤄지면
-- 틀립니다. 그래서 두 값을 따로 보관하고 불일치를 명시적으로 표시합니다.
CREATE TABLE IF NOT EXISTS ice.reg.doc_versions (
    doc_no          VARCHAR,
    version         INT,
    status          VARCHAR,    -- DRAFT / EFFECTIVE / SUPERSEDED / ABOLISHED
    effective_from  DATE,       -- 권위: 전자결재 최종 승인일
    effective_to    DATE,       -- NULL = 현재 유효. 후속 버전 시행일 - 1일
    stated_from     DATE,       -- 참고: 문서 부칙에 적힌 시행일
    date_mismatch   BOOLEAN,    -- effective_from <> stated_from → 인사팀 확인 대상
    approval_id     VARCHAR,    -- 전자결재 문서 ID (MySQL 그룹웨어와 조인)
    s3_uri          VARCHAR,
    file_format     VARCHAR,    -- pdf / docx / hwpx / xlsx
    sha256          VARCHAR,    -- 같은 규정의 작성본(.docx)/공표본(.pdf) 중복 판별
    is_authoritative BOOLEAN,   -- 같은 sha 그룹에서 공표본 우선
    page_count      INT,
    ingested_at     TIMESTAMP
);

-- ── 관계 ─────────────────────────────────────────────────────────────────────
-- 추출이 아니라 파싱의 결과입니다. 출처(source)를 남기는 이유는 신뢰도가
-- 다르기 때문입니다: 규정목록 엑셀 > 본문 정규식 > LLM 추정.
CREATE TABLE IF NOT EXISTS ice.reg.doc_relations (
    src_doc_no      VARCHAR,
    src_version     INT,        -- NULL 이면 문서 단위 관계
    dst_doc_no      VARCHAR,
    dst_version     INT,
    rel_type        VARCHAR,    -- SUPERSEDES / REFERENCES / CHILD_OF
    src_article     VARCHAR,    -- REFERENCES 인 경우 "제12조"
    source          VARCHAR,    -- MASTER_XLSX / BODY_REGEX / APPROVAL
    confidence      DOUBLE,     -- 본문 파싱은 1.0 미만. 검증 대상 선별에 사용
    valid_from      DATE,
    created_at      TIMESTAMP
);

-- ── 인입 로그 ────────────────────────────────────────────────────────────────
-- 파이프라인 각 단계의 성패. 조용히 건너뛴 파일이 없는지 확인하는 근거입니다.
CREATE TABLE IF NOT EXISTS ice.reg.ingest_log (
    run_id          VARCHAR,
    s3_uri          VARCHAR,
    stage           VARCHAR,    -- discover / extract / chunk / embed / merge
    status          VARCHAR,    -- ok / skipped / error
    reason          VARCHAR,    -- skipped 사유: duplicate_sha / image_only_pdf / no_doc_no
    elapsed_ms      BIGINT,
    error           VARCHAR,
    ran_at          TIMESTAMP
);
```


!!! danger "`effective_from` 이 이 데모의 심장입니다"
    문서 본문의 부칙이 아니라 **전자결재 승인일**에서 옵니다. 둘이 다른 버전이
    27개 있고, `date_mismatch` 로 표시됩니다. 부칙을 믿으면 열 주 동안 틀린
    답을 하면서 아무도 모릅니다.

### 청크

**`demo/schema/iceberg/02_chunks.sql`**

```sql
-- ============================================================================
-- 청크 — 모델 무관
--
-- 여기에 임베딩 컬럼이 없는 것이 의도입니다. 조문을 어떻게 잘랐는지는 임베딩
-- 모델이 무엇이든 동일하므로, 모델을 바꿔도 이 테이블은 다시 쓰지 않습니다.
-- 벡터는 세대별 파생 테이블(NeorunBase)로 나갑니다.
-- ============================================================================

CREATE TABLE IF NOT EXISTS ice.reg.doc_chunks (
    chunk_id        VARCHAR,    -- {doc_no}#{version}#{ordinal}
    -- 숫자 키를 따로 두는 이유는 검색 엔진 쪽 제약입니다. NeorunBase 의 FTS 인덱스는
    -- 숫자 기본키를 요구하고, HYBRID_SEARCH 는 그 기본키를 결과로 돌려줍니다.
    -- chunk_id 는 사람이 읽는 이름이고, 이건 기계가 조인하는 키입니다.
    chunk_pk        BIGINT,
    doc_no          VARCHAR,
    version         INT,
    -- ordinal_no, not ordinal: ORDINAL is reserved in the planner's grammar, and
    -- a column that must be quoted in every query it appears in is a papercut
    -- paid forever to save one rename now.
    ordinal_no      INT,
    article_no      VARCHAR,    -- "제12조" — 조문 단위 청킹이므로 대개 1:1
    article_title   VARCHAR,    -- "(휴가의 종류)"
    page_from       INT,
    page_to         INT,

    -- 원문과 치환본을 나란히 둡니다. 컬럼 마스킹이 이 둘 사이의 스왑으로
    -- 동작합니다: 권한이 없으면 text 를 요청해도 text_redacted 값이 나옵니다.
    -- 텍스트 전체를 가리면 검색 자체가 무의미해지므로, 가리는 것은 PII 뿐입니다.
    text            VARCHAR,
    text_redacted   VARCHAR,
    pii_types       VARCHAR,    -- JSON 배열: ["RRN","PHONE","NAME"] — 없으면 "[]"

    token_count     INT,
    created_at      TIMESTAMP
);

-- ── 임베딩 세대 ──────────────────────────────────────────────────────────────
-- VECTOR(n) 은 컬럼당 차원이 고정이라 두 세대가 한 컬럼을 공유할 수 없습니다.
-- 그래서 세대를 레지스트리로 관리하고, NeorunBase 에는 세대별 테이블
-- (doc_vectors_<generation_id>) 을 만듭니다. 모델 교체는 옆에 새 세대를 지어
-- 검증한 뒤 리트리버가 보는 세대를 전환하는 덧붙이기 작업이 됩니다.
CREATE TABLE IF NOT EXISTS ice.reg.embedding_generations (
    generation_id   VARCHAR,    -- gen1, gen2 …
    fingerprint     VARCHAR,    -- BAAI/bge-m3@<revision>:1024:l2
    model_id        VARCHAR,
    model_revision  VARCHAR,
    dim             INT,
    normalize       BOOLEAN,
    distance        VARCHAR,    -- cosine — NeorunBase HNSW 인덱스 메트릭과 일치해야 함
    target_table    VARCHAR,    -- nb.public.doc_vectors_gen1
    status          VARCHAR,    -- BUILDING / ACTIVE / RETIRED
    chunk_count     BIGINT,
    built_at        TIMESTAMP,
    activated_at    TIMESTAMP
);
```


### 인입 대기열

**`demo/schema/iceberg/50_ingest_queue.sql`**

```sql
-- ============================================================================
-- 인입 대기열 — 파이프라인이 무엇을 처리해야 하는지의 원장
--
-- 이 테이블이 있는 이유는 규모입니다. 이전 파이프라인은 드라이버가 S3 를
-- 나열하고, 파일을 열고, 청킹하고, 결과를 밀어 넣었습니다. 442건이면 됩니다.
-- 수만 건이면 한 대가 병목이고, 수억 건이면 드라이버 메모리에 목록조차 들어가지
-- 않습니다.
--
-- 목록을 테이블로 만들면 그 다음이 전부 SQL 이 됩니다:
--
--   INSERT INTO ice.reg.doc_chunks
--   SELECT q.s3_uri, c.* FROM ice.reg.ingest_queue q,
--          UNNEST(extract_chunks(q.s3_uri)) AS c
--
-- 드라이버는 아무것도 들고 있지 않고, 워커가 각자 맡은 split 의 객체만 읽습니다.
-- 442건이든 수억 건이든 같은 문장입니다.
--
-- status 를 두는 이유는 재시도입니다. 파이프라인이 중간에 죽었을 때 처음부터
-- 다시 하지 않으려면 어디까지 했는지가 데이터에 남아야 하고, 로그에 남으면
-- 그건 조회할 수 없는 상태입니다.
-- ============================================================================

CREATE TABLE IF NOT EXISTS ice.reg.ingest_queue (
    s3_uri          VARCHAR,   -- 객체 하나 = 행 하나
    bucket          VARCHAR,
    object_key      VARCHAR,
    size_bytes      BIGINT,
    etag            VARCHAR,   -- 내용이 바뀌었는지의 판별자. 같은 etag 는 재처리 불필요
    file_format     VARCHAR,   -- pdf / docx / xlsx / hwpx
    discovered_at   TIMESTAMP,
    status          VARCHAR,   -- PENDING / EXTRACTED / FAILED / SKIPPED
    attempt         INT,
    error           VARCHAR,   -- FAILED 의 사유. 조용히 건너뛴 파일이 없어야 합니다
    processed_at    TIMESTAMP,
    run_id          VARCHAR    -- 어느 실행이 이 행을 처리했는지 — lineage 의 시작점
)
WITH (identifier_fields = ARRAY['s3_uri']);
```


### 결재 이벤트 (스트림 대상)

**`demo/schema/iceberg/40_approval_stream.sql`**

```sql
-- ============================================================================
-- 결재 진행 중인 개정 (streaming)
--
-- 원장(01_documents.sql)은 이미 **끝난** 일을 담습니다 — 승인이 떨어져 시행에
-- 들어간 버전들. 그런데 실무에서 오답이 나오는 자리는 대개 그 반대편입니다:
-- 지금 결재가 돌고 있는 개정을 모르는 채 현행 규정을 그대로 읊어버리는 경우.
-- 답 자체는 맞지만 "곧 바뀝니다"를 빠뜨린 답입니다.
--
-- 이 테이블은 배치가 아니라 ontul Flow 가 채웁니다. 결재 시스템이 단계마다
-- 이벤트를 내보내고(기안→검토→승인→시행), Flow 가 approval_id 로 upsert 해서
-- **건별 최신 상태 한 줄**만 유지합니다. 단계 이력이 아니라 현재 상태가
-- 필요하기 때문입니다 — "지금 어디까지 왔나"에 답하려면 최신 한 줄이면 됩니다.
--
-- identifier_fields 가 upsert 의 키입니다. 이게 없으면 Flow 는 equality delete
-- 를 쓸 수 없고, 단계가 바뀔 때마다 같은 건이 한 줄씩 쌓여 "검토 중"과
-- "승인됨"이 동시에 참인 표가 됩니다.
-- ============================================================================

CREATE TABLE IF NOT EXISTS ice.reg.approval_status (
    approval_id      VARCHAR,   -- 전자결재 문서 ID. 개정 1건 = 1행
    doc_no           VARCHAR,   -- 대상 규정 (ice.reg.documents 와 조인)
    version          INT,       -- 이 결재가 만들려는 차수 (현행 + 1)
    step             VARCHAR,   -- DRAFT / REVIEW / APPROVED / EFFECTIVE / REJECTED
    step_seq         INT,       -- 단계 순번. 이벤트가 뒤늦게 도착해도 순서를 알 수 있게
    drafter          VARCHAR,   -- 기안자 사번
    owner_dept       VARCHAR,
    summary          VARCHAR,   -- 개정 요지 한 줄
    expected_from    DATE,      -- 예정 시행일. 승인 전이므로 어디까지나 예정
    updated_at       TIMESTAMP
)
WITH (identifier_fields = ARRAY['approval_id']);
```


### 개정 요청 원장 — 온톨로지 액션이 쓰는 곳

**`demo/schema/iceberg/60_revision_requests.sql`**

```sql
-- 개정 요청 원장.
--
-- 온톨로지 액션 request_revision 이 쓰는 곳입니다. 파생 서빙 계층이 아니라
-- Iceberg 에 쓰는 것이 요점입니다 — NeorunBase 는 파이프라인이 언제든 다시
-- 만들 수 있는 사본이고, 다시 만들면 사라질 곳에 남긴 기록은 기록이 아닙니다.
--
-- "누가 언제" 가 이 표에 없는 이유가 중요합니다. 액션의 SQL 템플릿은 선언된
-- 파라미터만 치환하므로 세션 사용자를 넣을 자리가 없고, 그렇다고 requested_by
-- 를 파라미터로 받으면 남의 이름으로 요청할 수 있게 됩니다 — 위조 가능한 칸이
-- 진짜 기록 옆에 앉아 있는 것이 아무 칸도 없는 것보다 나쁩니다. 호출자와 시각은
-- 플랫폼의 감사 로그가 기록하고, 그건 호출자가 고칠 수 없습니다.
--
--   SELECT * FROM ontul.audit WHERE action = 'action:invoke'
--
-- request_id 는 (규정, 판) 하나당 하나입니다. 같은 판에 대한 두 번째 요청은 새
-- 요청이 아니라 같은 요청이고, 멱등 키가 그것을 그대로 돌려줍니다.
CREATE TABLE IF NOT EXISTS ice.reg.revision_requests (
    request_id    VARCHAR,
    doc_no        VARCHAR,
    version       INT,
    reason        VARCHAR,
    status        VARCHAR
) USING iceberg;
```


### 그래프의 레이크 투영

**`demo/schema/iceberg/61_graph_projection.sql`**

```sql
-- 그래프의 서빙 투영.
--
-- ice.reg.doc_relations 는 "무엇이 무엇의 근거인가" 라는 의미 관계이고, 아래
-- 두 표는 그 관계를 NeorunBase 그래프 엔진이 받는 모양 그대로 담습니다. 굳이
-- 두 벌인 이유는 온톨로지 문서가 말하는 원칙 때문입니다 — Iceberg 가 기록의
-- 원본이고 NeorunBase 는 언제든 다시 만들 수 있는 파생 서빙 계층입니다.
--
-- 그래서 배치 잡은 NeorunBase 에 직접 쓰지 않습니다. 여기에 쓰고, Flow 가
-- 이걸 보고 서빙 그래프를 따라오게 합니다. 차이는 운영에서 드러납니다:
-- NeorunBase 를 날리고 다시 세워도 잡을 다시 돌릴 필요가 없고, 관계가 언제
-- 어떻게 바뀌었는지는 Iceberg 스냅샷에 남습니다.
--
-- 대리키(doc_id, edge_pk)를 레이크 쪽에서 부여합니다. 서빙 쪽에서 매기면 다시
-- 세울 때마다 값이 달라져서, 그래프를 가리키는 무엇도 안정적으로 남지 않습니다.
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

## 2. NeorunBase 서빙 스키마

벡터 · 한국어 FTS · 그래프가 한 엔진 안에 있습니다. 하이브리드 융합이
애플리케이션 코드가 아니라 엔진에서 일어난다는 뜻이고, IAM 행 필터를 두 번
쓰지 않아도 된다는 뜻이기도 합니다.

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


!!! note "왜 시행일이 벡터 테이블에 비정규화되어 있는가"
    시간 조건이 **검색 안에서** 평가돼야 하기 때문입니다. top-k 를 뽑고 나서
    걸러내면 k 개보다 적게 남고, 때로는 하나도 남지 않습니다. 이건 성능 문제가
    아니라 정답 문제입니다.

---

## 3. 시맨틱 뷰

질문이 실제로 붙는 면입니다. 시간 필터가 여기 있고, 폐지된 본문은 **없습니다** —
점수가 낮은 것이 아니라 뷰에 존재하지 않습니다.

**`demo/schema/semantic/01_effective.sql`**

```sql
-- ============================================================================
-- 시간 정합성 — the demo's central claim, expressed as a view
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
-- The minimal curation from the discussion: 인사팀 marks the few dozen documents
-- that may serve as grounds for an answer. Without it, a meeting note that
-- mentions 휴가 competes with the regulation that defines it — and on vector
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


-- ── 문서번호 → 그래프 노드 id ────────────────────────────────────────────────
-- GRAPH_NEIGHBORS 는 숫자 seed 로 출발하는데, 질문하는 사람에게 그 숫자를 알
-- 이유가 없습니다. 그래서 도구가 문서번호로 조회해 seed 를 얻는데, 그 조회가
-- nb.public.doc_nodes 를 직접 겨누고 있었습니다 — 에이전트가 읽는 것 중 유일하게
-- 시맨틱 레이어 밖이었고, 따라서 정책이 닿지 않는 통로였습니다.
CREATE OR REPLACE VIEW semantic.reg.doc_index AS
SELECT doc_id, doc_no, title, tier, owner_dept, is_official, sensitivity
FROM nb.public.doc_nodes;
```


ERP 쪽 뷰들입니다. 연합 질의가 여기서 문서와 기록을 잇습니다.

**`demo/schema/semantic/02_erp.sql`**

```sql
-- ============================================================================
-- ERP semantic layer — making an ERP legible
--
-- 원천은 ice.erp.* 입니다 — Postgres 가 아니라 그 Postgres 를 CDC 로 흘려 받은
-- Iceberg 테이블입니다 (infra/cdc.sh). 뷰가 원천을 가리고 있어서 아래를
-- 갈아끼워도 정책과 도구는 그대로였습니다. 직접 붙이지 않는 이유는 부하보다
-- 넓습니다: 분석 질의가 OLTP 를 흔들고, 워커마다 커넥션을 열고, 무엇보다
-- 덮어써진 값은 되돌릴 수 없어 "그때는 며칠이었나" 를 물을 데가 없습니다.
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
-- The ceiling is a fact of the 구매 규정; keeping it in the view means "was this
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


-- ── 결재 진행 중인 개정 (ontul Flow 가 채우는 스트림 테이블 위) ──────────────
-- 원장 뷰들과 같은 자리에 두는 이유는 권한 때문입니다. 에이전트가 쓰는 모든
-- 읽기는 시맨틱 레이어를 지나가고, 정책도 거기에 붙습니다. 스트림 테이블을
-- 직접 읽게 하면 그 한 곳만 규칙 밖에 놓입니다.
CREATE SCHEMA IF NOT EXISTS semantic.reg;

CREATE OR REPLACE VIEW semantic.reg.pending_revisions AS
SELECT
    a.approval_id   AS "결재번호",
    a.doc_no        AS "문서번호",
    d.title         AS "제목",
    a.version       AS "개정차수",
    CASE a.step WHEN 'DRAFT' THEN '기안' WHEN 'REVIEW' THEN '검토'
                WHEN 'APPROVED' THEN '승인' WHEN 'EFFECTIVE' THEN '시행'
                WHEN 'REJECTED' THEN '반려' END AS "단계",
    a.step          AS step,
    a.summary       AS "요지",
    a.expected_from AS "예정시행일",
    a.drafter       AS "기안자사번",
    a.owner_dept    AS "소관부서",
    a.updated_at    AS "갱신시각",
    -- 정책이 semantic.reg.* 전체에 sensitivity 조건을 겁니다. 대외비 문서의
    -- 개정 건도 일반 직원에게는 보이지 않아야 하므로 조건이 걸릴 자리를
    -- 뷰가 제공해야 합니다. 아직 원장에 없는 문서의 결재는 INTERNAL 로 봅니다
    -- — NULL 이면 조건이 UNKNOWN 이 되어 행이 조용히 사라집니다.
    COALESCE(d.sensitivity, 'INTERNAL') AS sensitivity
FROM ice.reg.approval_status a
LEFT JOIN ice.reg.documents d ON d.doc_no = a.doc_no;
```


시맨틱 뷰는 엔진 레벨 VIEW 가 아니라 **정의**로 등록됩니다. 인증 상태와 필수
필터를 함께 들고 있어야 하는데, 그건 평범한 DDL 에 담을 곳이 없기 때문입니다.
`.sql` 파일을 원본으로 두고 아래 스크립트가 옮깁니다 — 검토하는 사람이 JSON 을
해독하지 않고 시간 조건을 읽을 수 있어야 합니다.

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

## 4. 커넥션

자격증명은 커넥션에 한 번 등록하고, 카탈로그와 잡은 **id 로 참조**합니다.
임베딩 커넥션이 특히 그렇습니다 — 벡터 공간의 정의가 한 곳에만 있어야 색인한
벡터와 질의 벡터가 비교 가능합니다.

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
    "_timeout_comment": "배치 하나가 64개 청크이고, CPU 로 도는 e5-base 는 그 한 배치에 수 초가 걸립니다. 여기에 CDC/그래프 Flow 가 같은 머신에서 함께 돌면 30초로는 모자랍니다 — 그 결과가 'request timed out' 이고, 실패하는 것은 임베딩 서비스가 아니라 색인 잡 전체입니다. 여유를 주는 편이 낫습니다: 모델이 실제로 죽었다면 어차피 연결 단계에서 드러납니다."
  }
}
```


!!! warning "e5 는 비대칭입니다"
    같은 문장이라도 저장 텍스트로 임베딩할 때와 검색어로 임베딩할 때가 다릅니다.
    거꾸로 해도 **오류가 나지 않습니다** — 결과가 조금 나빠질 뿐이고, 그게 설정
    필드 하나를 쓸 가치가 있는 실패 방식입니다.

**`demo/schema/connections/02_erp.json`**

```json
{
  "_comment": "ERP, read live rather than copied into the lake. A leave balance is only worth quoting if it is the balance right now — an overnight snapshot would let the agent state a number that was true yesterday with the same confidence as one that is true today.",
  "connectionId": "erp",
  "type": "JDBC",
  "description": "ERP (PostgreSQL) — 인사·근태·경비, federated",
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
  "_comment": "전자결재. Authoritative for when a regulation actually took effect: the approval record is what makes a rule binding, and it routinely disagrees with the date printed in the document's 부칙. HR-REG-003 v3 states 2025-01-01 and was approved 2025-03-15 — ten weeks in which citing the document would have been wrong.",
  "connectionId": "groupware",
  "type": "JDBC",
  "description": "전자결재 (MySQL) — 결재 이력, CDC 소스",
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
  "description": "교육 이수 (SaaS LMS) — rest-operation 경유",
  "properties": {
    "baseUrl": "http://regdemo-lms:8000",
    "authType": "none",
    "timeoutMs": "10000"
  }
}
```


---

## 5. 등록 스크립트

카탈로그 · 커넥션 · 스키마 · IAM · 시맨틱 뷰 · 리트리버 · 온톨로지를 전부
등록합니다. 멱등합니다 — 다시 돌리면 같은 정의를 다시 적용하지 중복을 만들지
않습니다.

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

# NeorunBase 카탈로그는 여기서 등록하지 않습니다. 커넥터는 등록하는 순간의 테이블
# 목록을 잡는데, 첫 설치에서는 NeorunBase 에 아직 테이블이 하나도 없습니다. 스키마를
# 만든 뒤(3단계 끝)에 등록해야 테이블이 보입니다.

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
  # 이미 있으면 지우고 다시 만듭니다. post 는 "already exists" 를 성공으로
  # 취급하는데, 그건 재실행을 허용하려는 것이지 **바뀐 정의를 무시하려는**
  # 것이 아닙니다. 파일에서 타임아웃이나 모델 리비전을 고치고 이 스크립트를
  # 다시 돌렸을 때 아무 일도 일어나지 않으면, 고친 사람은 반영됐다고 믿고
  # 반영되지 않은 채로 계속 갑니다.
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
  "gw (mysql — 결재, the authority on effective dates)"

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

# 이제 테이블이 있으니 카탈로그를 등록합니다.
#
# jdbcUrl 이지 host/port/database 가 아닙니다. 커넥터는 'jdbcUrl' 또는
# 'endpoint' 만 읽고 그 밖의 키는 보지 않습니다 — 둘 다 없으면 등록은 성공하고
# 테이블이 0개가 되어, 이후 모든 질의가 설정 이야기가 아니라 "Object 'nb' not
# found" 로 실패합니다. preferQueryMode=simple 은 NeorunBase 가 simple query
# 프로토콜을 서빙하기 때문입니다.
post /admin/catalogs "$(cat <<JSON
{"name":"nb","config":{"connector":"neorunbase",
   "jdbcUrl":"jdbc:postgresql://$NEORUNBASE_INTERNAL_HOST:5432/neorunbase?preferQueryMode=simple",
   "endpoint":"http://$NEORUNBASE_INTERNAL_HOST:8080",
   "username":"admin","password":"$NEORUNBASE_PASSWORD","schema":"public"}}
JSON
)" "nb (neorunbase — vectors + korean fts)"

# 등록됐는데 테이블이 0개면 등록된 것이 아닙니다. 여기서 멈춰야 잘못된 설정이
# 한참 뒤의 엉뚱한 오류가 아니라 그 자리에서 드러납니다.
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
# A real DEV 부장. The department attribute drives the row filter, so a persona
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

# ── 온톨로지. 객체 → 링크 → 액션 순서가 강제입니다: 링크는 양쪽 객체 타입이
#    이미 있어야 하고, 액션은 자기가 다루는 객체 타입이 있어야 합니다.
#    파일 이름의 숫자가 그 순서입니다.
log "5/6  온톨로지 (객체 · 링크 · 액션)"
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

### 이 스크립트가 순서를 지키는 이유

- **NeorunBase 카탈로그는 스키마를 만든 뒤에 등록합니다.** 커넥터가 등록 시점의
  테이블 목록을 잡기 때문입니다. 첫 설치에서 먼저 등록하면 테이블 0개로 잡히고,
  이후 모든 질의가 설정 이야기가 아니라 `Object 'nb' not found` 로 실패합니다.
- **커넥션은 지우고 다시 만듭니다.** "이미 있음" 을 성공으로 취급하는 것은
  재실행을 허용하려는 것이지 **바뀐 정의를 무시하려는** 것이 아닙니다. 파일에서
  타임아웃이나 모델 리비전을 고치고 다시 돌렸을 때 아무 일도 일어나지 않으면,
  고친 사람은 반영됐다고 믿고 반영되지 않은 채로 계속 갑니다.
- **온톨로지는 객체 → 링크 → 액션 순서**입니다. 링크는 양쪽 객체 타입이 이미
  있어야 하고, 액션은 자기가 다루는 객체 타입이 있어야 합니다.

---

다음: [파이프라인](pipeline.md) — 이 스키마를 채우는 분산 잡들.
