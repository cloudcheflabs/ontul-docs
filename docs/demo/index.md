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

And `effective_from` is the **approval date**, not the date the document's own 부칙 (supplementary provision) prints.
HR-REG-003 v3 claims 2025-01-01 and was approved 2025-03-15; for those ten weeks
the old version was still binding. A pipeline that trusts the document body gets
this wrong and has no way to notice.

### The test that matters

홍가은 has used 12 of 20 childcare days.

| Source read | Answer |
|---|---|
| HR-REG-003 **v3** (current, 20 days) | **8 days left** ✓ |
| HR-REG-003 v2 (superseded, 15 days) | 3 days left ✗ |

A temporal-isolation failure produces a wrong **number**, not a wrong citation.
So the end-to-end test asserts on arithmetic rather than on whether a document
was named.

---

## What is running

```text
                          ┌──────────────────────────────┐
    user ──── question ──► │  Agent (claude-opus-5)       │
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
    │  the       │    │  vector · FTS ·  │    │  Approvals (MySQL) │
    │  ledger    │    │  graph · OLTP    │    │  Training (REST)   │
    │ (Polaris+S3)│   │  = derived serve │    └────────────────────┘
    └─────┬──────┘    └────────▲─────────┘
          │                    │
          │   Ontul Flow       │   CDC + changelog
          └────────────────────┘

    ShannonStore  S3 object storage — the original documents and the Iceberg data files
    Polaris       the Iceberg REST catalog
    kiok          the batch scheduler that runs the indexing DAG
    embed-svc     multilingual-e5-base, pinned and offline
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
| [Install](install.md) | The whole stack from four release tarballs — every compose file and Dockerfile in full |
| [Corpus](corpus.md) | 446 files shaped like a real shared drive — name collisions, scanned PDFs and register defects, all deliberate |
| [Schema](schema.md) | The Iceberg ledger, the semantic views, and the NeorunBase serving layer |
| [Pipeline](pipeline.md) | Distributed indexing on a kiok DAG — extract, chunk, effective dates, embed, graph |
| [CDC and Flow](cdc-flow.md) | ERP → Iceberg, graph → serving, and the approval event stream |
| [IAM and retrievers](iam.md) | Why the same question returns different rows to different people |
| [Ontology](ontology.md) | Objects, links, and a governed write |
| [Agent](agent.md) | Nine tools, the system prompt, and the chat window |
| [Verification](verify.md) | The scenario suite and what was actually measured |

---

## Measured, on one laptop

Taken on a 16 GB laptop with 10.7 GB given to Docker.

| | |
|---|---|
| Source documents | 446 files → 123 matched, 319 unmatched, 4 scan-only |
| Ledger | 50 documents · 104 versions |
| Chunks | 818, none duplicated |
| Vectors | 818 × 768 dim |
| Graph | 50 nodes · 116 edges |
| ERP CDC | 5 tables, 1,214 rows, compared by value |
| One DAG run | about 50 seconds |

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
