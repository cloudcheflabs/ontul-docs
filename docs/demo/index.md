# Regulation Lakehouse — the demo, end to end

An agent that answers questions about a company's regulations and HR records,
built so that **a superseded regulation cannot be the source of an answer**.

Everything on these pages runs. The stack is four cloudcheflabs products plus
Apache Polaris and a few ordinary source systems, installed from released
tarballs; the corpus, the pipeline, the policies and the agent are all here in
full. Nothing is elided — if a file is part of making this work, its contents are
on one of these pages, because the products are closed-source and this
documentation is the only copy you get.

!!! note "Follow it and it works"
    These pages are written to be executed, not skimmed. Read in order, paste
    what is shown, and you end with the same cluster the screenshots came from.

---

## Why this demo exists

A 300-person company holds a few dozen governed regulations buried in hundreds of
meeting notes, several versions of each, and filenames that lie about which
version they are. An index that treats all of it alike answers *"육아휴직 며칠?"*
with a number that was correct two years ago — and nothing in the answer marks it
as wrong.

That is the failure this demo is built around, and it is worth being precise
about why it is hard.

**A superseded regulation is the best match for the question it used to answer.**
It is the best lexical match and the best semantic match. No recency weighting
reliably beats a document that says exactly what was asked. Ranking is the wrong
tool.

So the retriever does not rank the old version lower. It binds to a view where
the expired text is **absent**:

```sql
WHERE v.status = 'EFFECTIVE'
  AND v.effective_from <= CURRENT_DATE
  AND (v.effective_to IS NULL OR v.effective_to > CURRENT_DATE)
```

And `effective_from` is the **approval date**, not the 부칙 the document prints.
HR-REG-003 v3 claims 2025-01-01 and was approved 2025-03-15; for those ten weeks
the old version was still binding. A pipeline that trusts the document body gets
this wrong and has no way to notice.

### The test that matters

홍가은 has used 12 of 20 childcare days.

| Source read | Answer |
|---|---|
| HR-REG-003 **v3** (current, 20 days) | **8일** ✓ |
| HR-REG-003 v2 (superseded, 15 days) | 3일 ✗ |

A temporal-isolation failure produces a wrong **number**, not a wrong citation.
So the end-to-end test asserts on arithmetic rather than on whether a document
was named.

---

## What is running

```text
                          ┌──────────────────────────────┐
    사용자 ─── 질문 ────► │  Agent (claude-opus-5)        │
                          │  9 tools, runs as the caller │
                          └──────────────┬───────────────┘
                                         │  every call carries the asker's identity
                                         ▼
    ┌────────────────────────────────────────────────────────────────────┐
    │  Ontul — federation · semantic layer · ontology · IAM · Flow        │
    │                                                                     │
    │   semantic views      retrievers        object/link/action          │
    │   (temporal filter)   (hybrid search)   (typed entities + writes)   │
    └───┬──────────────────────┬───────────────────────┬─────────────────┘
        │                      │                       │
        ▼                      ▼                       ▼
    ┌────────────┐    ┌──────────────────┐    ┌────────────────────┐
    │  Iceberg   │    │   NeorunBase     │    │  ERP (PostgreSQL)  │
    │  원장       │    │  vector · FTS ·  │    │  전자결재 (MySQL)   │
    │  (Polaris  │    │  graph · OLTP    │    │  교육이수 (REST)     │
    │   + S3)    │    │  = 서빙(파생)     │    └────────────────────┘
    └─────┬──────┘    └────────▲─────────┘
          │                    │
          │   Ontul Flow       │   CDC + changelog
          └────────────────────┘

    ShannonStore  S3 오브젝트 스토리지 (원본 문서 + Iceberg 데이터 파일)
    Polaris       Iceberg REST 카탈로그
    kiok          배치 스케줄러 — 인덱싱 DAG
    embed-svc     multilingual-e5-base, 오프라인 고정
```

Nothing is present for breadth:

- **No Kafka.** The real trigger for a regulation taking effect is an approval
  row changing, and CDC already reads that.
- **No separate search engine.** NeorunBase carries vectors, Korean full-text and
  the graph, so hybrid fusion happens *in the engine* rather than in application
  code — which is where most stacks get their ranking wrong — and the IAM row
  filter is written once instead of twice.
- **No cloud API for embeddings.** The model runs in-network with
  `HF_HUB_OFFLINE=1`, because a regulation corpus is not something you post to a
  third party to find out what it says.

---

## What it demonstrates, and where to read each part

| | |
|---|---|
| [설치](install.md) | 릴리스 tarball 네 개로 스택 전체를 세웁니다. compose 와 Dockerfile 전문 |
| [코퍼스](corpus.md) | 실제 공유 드라이브를 닮은 446개 파일 — 파일명 충돌, 스캔 PDF, 목록 결함까지 의도적으로 |
| [스키마](schema.md) | Iceberg 원장 · 시맨틱 뷰 · NeorunBase 서빙 스키마 |
| [파이프라인](pipeline.md) | kiok DAG 로 도는 분산 인덱싱 — 추출·청킹·시행일·임베딩·그래프 |
| [CDC 와 Flow](cdc-flow.md) | ERP → Iceberg, 그래프 → 서빙, 결재 이벤트 스트림 |
| [IAM 과 리트리버](iam.md) | 페르소나별로 다른 행이 돌아오는 이유 |
| [온톨로지](ontology.md) | 객체 · 링크 · 거버넌스가 붙은 쓰기 |
| [에이전트](agent.md) | 툴 9개, 시스템 프롬프트, 웹 채팅 화면 |
| [검증](verify.md) | 시나리오 스위트와 실제 측정값 |

---

## Measured, on one laptop

Docker 에 10.7GB 를 준 16GB 머신에서 나온 값입니다.

| | |
|---|---|
| 원본 문서 | 446 files → 인식 123, 미매칭 319, 스캔 전용 4 |
| 원장 | 문서 50 · 버전 104 |
| 청크 | 818 (중복 0) |
| 벡터 | 818 × 768 dim |
| 그래프 | 노드 50 · 엣지 116 |
| ERP CDC | 5 tables, 1,214 rows, 값까지 대조 |
| DAG 한 바퀴 | 약 50초 |

---

## A note on the failure mode this demo kept finding

Almost every defect found while building this — in the pipeline and in the engine
— had the same shape: **something went wrong and the caller was told nothing.**

An expired catalog token turned every query into a successful read of an
apparently empty ledger. A Flow with an unrecognised snapshot mode ran healthily
and delivered nothing. A local ingest and a distributed job both loaded the same
chunks, doubling the index while every count-based check still passed. A CDC
connector encoded every decimal as base64 and the row counts matched exactly.

That is why the checks on these pages compare *values* rather than counts, and
why several of them assert against the ledger rather than against what a
component reported about itself.
