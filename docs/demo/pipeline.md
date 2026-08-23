# The pipeline — distributed indexing on a kiok DAG

For a handful of documents a shell script is enough. For tens of thousands it is
not. Here a **kiok DAG** holds the order and the heavy work happens on **Ontul
workers**.

```text
    discover ──► chunk ──► effective_dates ──► … ──► vectors ──┐
                    └────► graph ────────────────────────────────┴──► generation
```

| Task | Kind | What it does |
|---|---|---|
| `discover` | Ontul PYTHON | Walks S3 and fills the ingest queue |
| `chunk` | Ontul BATCH SQL | `UNNEST(extract_chunks(uri))` — the driver holds no data |
| `effective_dates` | Ontul BATCH SQL | **Federated join** against the approval system to settle the date |
| `close_versions` | Ontul PYTHON | Closes each version at the next one's effective date |
| `vectors_clear` | Ontul PYTHON | Empties the generation's table |
| `vectors` | Ontul BATCH SQL | Evaluates `embed_passage()` per Arrow batch |
| `graph` | Ontul PYTHON | Extracts authority relations from citations in the text |
| `generation` | Ontul BATCH SQL | Records this generation's identity |

!!! quote "Why a scheduler"
    A shell script's line order cannot be queried. Which step stopped, what needs
    re-running, how yesterday's run differed from today's — none of it is
    recorded anywhere. Moving to a DAG turns all of that into data.

![The kiok DAG](../images/demo/kiok-dag.png)

---

## The DAG definition

**`demo/schema/dags/regdemo_index.yaml`**

```yaml
# The regulation indexing pipeline.
#
# Before this file existed, the dependency order between stages lived only inside
# a shell script. That is enough to run once but hard to call a pipeline: which
# stage stopped and where, what needs re-running, how yesterday's run differed
# from today's — none of it is recorded anywhere.
#
#   discover ─→ chunk ─→ effective_dates ─→ … ─→ vectors ─→ generation
#                   └──→ graph ────────────────────────────────┘
#
# Every heavy thing happens on an Ontul worker. kiok keeps the order and records
# the results, so it holds no data.
#
# Two things are deliberately absent from this file:
#
#   Tokens. Only ${conn.regdemoOntul.*} references appear. kiok resolves them on
#   the worker from the KMS-encrypted connection store just before the task runs,
#   so no value survives in the stored DagSpec or in the admin UI's Source tab.
#   That is why this file is committed as it is. The value is also a non-expiring
#   user token (OTOK…) rather than a login JWT — a JWT lasts fifteen minutes and
#   would break the next scheduled run.
#
#   The dependency key is `requires`. Written as `depends_on`, kiok silently
#   dropped a key it did not recognise: the file looked ordered while all six
#   tasks ran at once.
dag:
  id: regdemo_index
  default_timeout: 30m
tasks:
- id: discover
  type: ontul
  config:
    ontul.url: ${conn.regdemoOntul.url}
    ontul.token: ${conn.regdemoOntul.token}
    ontul.jobType: PYTHON
    ontul.scriptPath: ${conn.regdemoJobs.discover_job}
    ontul.args:
      bucket: iceberg-warehouse
      prefix: corpus/
      run_id: '#{ nowFormatted(''yyyyMMdd-HHmmss'') }'
      s3_endpoint: ${conn.regdemoS3.endpoint}
      s3_access_key: ${conn.regdemoS3.accessKey}
      s3_secret_key: ${conn.regdemoS3.secretKey}
    ontul.jobConfig:
      ontul.job.driver.mode: worker
      ontul.deps.s3.connectionId: regdemoS3
    ontul.pollIntervalMs: '2000'
  timeout: 10m
- id: chunk
  type: ontul
  requires:
  - discover
  timeout: 20m
  config:
    ontul.url: ${conn.regdemoOntul.url}
    ontul.token: ${conn.regdemoOntul.token}
    ontul.jobType: BATCH
    ontul.pollIntervalMs: '3000'
  script: "INSERT INTO ice.reg.doc_chunks\n  (chunk_pk, chunk_id, doc_no, version, ordinal_no, article_no,\
    \ article_title,\n   page_from, page_to, text, token_count)\nSELECT\n  chunk_pk(q.s3_uri || '#' ||\
    \ jfield(c, 'ordinal_no')),\n  q.s3_uri || '#' || jfield(c, 'ordinal_no'),\n  v.doc_no,\n  v.version,\n\
    \  CAST(jfield(c, 'ordinal_no') AS INTEGER),\n  jfield(c, 'article_no'),\n  jfield(c, 'article_title'),\n\
    \  CAST(jfield(c, 'page_from') AS INTEGER),\n  CAST(jfield(c, 'page_to') AS INTEGER),\n  jfield(c,\
    \ 'text'),\n  CAST(jfield(c, 'token_count') AS INTEGER)\nFROM ice.reg.ingest_queue q\nJOIN ice.reg.doc_versions\
    \ v ON v.s3_uri = q.s3_uri,\n     UNNEST(extract_chunks(q.s3_uri)) AS t(c)\nWHERE q.status = 'PENDING'\n"
- id: effective_dates
  type: ontul
  requires:
  - chunk
  config:
    ontul.url: ${conn.regdemoOntul.url}
    ontul.token: ${conn.regdemoOntul.token}
    ontul.jobType: BATCH
    ontul.pollIntervalMs: '2000'
  script: "MERGE INTO ice.reg.doc_versions v\nUSING (\n    SELECT doc_no,\n           ver            \
    \  AS version,\n           apr_id,\n           complete_dt      AS approved_on,\n           stated_dt\
    \        AS stated_on,\n           \n           \n           \n           \n           (complete_dt\
    \ <> stated_dt) AS mismatch\n    FROM gw.groupware.gw_approval\n    WHERE sts_cd = 'CMPL'\n) a\nON\
    \ v.doc_no = a.doc_no AND v.version = a.version\nWHEN MATCHED THEN UPDATE SET\n    effective_from\
    \ = a.approved_on,\n    stated_from    = a.stated_on,\n    date_mismatch  = a.mismatch,\n    approval_id\
    \    = a.apr_id,\n    status         = 'EFFECTIVE'\n"
- id: drafts_pending
  type: ontul
  requires:
  - effective_dates
  config:
    ontul.url: ${conn.regdemoOntul.url}
    ontul.token: ${conn.regdemoOntul.token}
    ontul.jobType: BATCH
    ontul.pollIntervalMs: '2000'
  script: "MERGE INTO ice.reg.doc_versions v\nUSING (\n    SELECT doc_no, ver AS version, apr_id\n   \
    \ FROM gw.groupware.gw_approval\n    WHERE sts_cd <> 'CMPL'\n) p\nON v.doc_no = p.doc_no AND v.version\
    \ = p.version\nWHEN MATCHED THEN UPDATE SET\n    effective_from = NULL,\n    approval_id    = p.apr_id,\n\
    \    status         = 'DRAFT'\n"
- id: queue_done
  type: ontul
  requires:
  - drafts_pending
  config:
    ontul.url: ${conn.regdemoOntul.url}
    ontul.token: ${conn.regdemoOntul.token}
    ontul.jobType: BATCH
    ontul.pollIntervalMs: '2000'
  script: 'UPDATE ice.reg.ingest_queue SET status = ''EXTRACTED'' WHERE status = ''PENDING''

    '
- id: close_versions
  type: ontul
  requires:
  - queue_done
  timeout: 10m
  config:
    ontul.url: ${conn.regdemoOntul.url}
    ontul.token: ${conn.regdemoOntul.token}
    ontul.jobType: PYTHON
    ontul.scriptPath: ${conn.regdemoJobs.close_versions_job}
    ontul.jobConfig:
      ontul.job.driver.mode: worker
      ontul.deps.s3.connectionId: regdemoS3
    ontul.pollIntervalMs: '2000'
- id: vectors_clear
  type: ontul
  requires:
  - close_versions
  timeout: 10m
  config:
    ontul.url: ${conn.regdemoOntul.url}
    ontul.token: ${conn.regdemoOntul.token}
    ontul.jobType: PYTHON
    ontul.scriptPath: ${conn.regdemoJobs.clear_vectors_job}
    ontul.args:
      host: ${conn.regdemoNeorun.host}
      port: ${conn.regdemoNeorun.port}
      database: ${conn.regdemoNeorun.database}
      user: ${conn.regdemoNeorun.username}
      password: ${conn.regdemoNeorun.password}
    ontul.jobConfig:
      ontul.job.driver.mode: worker
      ontul.deps.s3.connectionId: regdemoS3
    ontul.pollIntervalMs: '2000'
- id: vectors
  type: ontul
  requires:
  - vectors_clear
  timeout: 30m
  config:
    ontul.url: ${conn.regdemoOntul.url}
    ontul.token: ${conn.regdemoOntul.token}
    ontul.jobType: BATCH
    ontul.pollIntervalMs: '5000'
  script: "INSERT INTO nb.public.doc_vectors_gen1\n    (chunk_pk, chunk_id, doc_no, version, article_no,\
    \ effective_from, effective_to,\n     is_official, sensitivity, owner_dept, body, embedding)\nSELECT\n\
    \    c.chunk_pk,\n    c.chunk_id,\n    c.doc_no,\n    c.version,\n    c.article_no,\n    v.effective_from,\n\
    \    v.effective_to,\n    d.is_official,\n    d.sensitivity,\n    d.owner_dept,\n    c.text,\n   \
    \ embed_passage('emb_main', c.text)\nFROM ice.reg.doc_chunks c\nJOIN ice.reg.doc_versions v\n  ON\
    \ c.doc_no || '#' || CAST(c.version AS VARCHAR)\n   = v.doc_no || '#' || CAST(v.version AS VARCHAR)\n\
    JOIN ice.reg.documents d ON d.doc_no = c.doc_no\n"
- id: graph
  type: ontul
  requires:
  - chunk
  config:
    ontul.url: ${conn.regdemoOntul.url}
    ontul.token: ${conn.regdemoOntul.token}
    ontul.jobType: PYTHON
    ontul.scriptPath: ${conn.regdemoJobs.build_graph_job}
    ontul.jobConfig:
      ontul.job.driver.mode: worker
      ontul.deps.s3.connectionId: regdemoS3
    ontul.pollIntervalMs: '3000'
  timeout: 15m
- id: generation
  type: ontul
  requires:
  - vectors
  - graph
  config:
    ontul.url: ${conn.regdemoOntul.url}
    ontul.token: ${conn.regdemoOntul.token}
    ontul.jobType: BATCH
    ontul.pollIntervalMs: '2000'
  script: 'SELECT count(*) AS docs FROM ice.reg.documents

    '
```


!!! danger "It is `requires`, not `depends_on`"
    kiok **silently drops** keys it does not know. Written as `depends_on`, the
    file looked ordered while all six tasks ran at once; the run was fast and
    nothing failed. The only clue was that the graph view had no edges in it.

!!! note "The token is not in the file"
    Only a `${conn.regdemoOntul.token}` reference is. kiok resolves it on the
    worker from the KMS-encrypted connection store just before the task runs, so
    the value appears neither in the stored DagSpec nor in the admin UI — which
    is why this file is committed as-is. The value is a non-expiring user token
    (`OTOK…`), not a login JWT: a JWT lasts fifteen minutes and would break the
    next scheduled run.

---

## 1. Loading the ledger (local Python)

This is the one stage that runs locally. Reading PDF/DOCX/XLSX and matching
against the register needs a filesystem and format libraries, and the result is a
few tens of thousands of ledger rows.

```bash
python3 -m venv .venv && .venv/bin/pip install -e pipeline
( cd pipeline/src && PYTHONPATH=. ../../.venv/bin/python -m regdemo_pipeline.jobs.ingest \
    --out ../../out --no-upload )
```

**`demo/pipeline/src/regdemo_pipeline/jobs/ingest.py`**

```python
"""
Ingest: files on disk become rows in the ledger.

What this job does and, more importantly, what it deliberately does not do.

It does not embed anything. Embedding is a separate job (jobs/10_index_vectors.sql)
that runs inside Ontul, distributed across workers, because that is where the
data already is. Doing it here would mean pulling every chunk to one machine and
pushing 6,000 vectors back — and it would put the model behind a Python import
instead of behind the connection that the retriever also names, which is the one
thing that guarantees an index and a query are comparable.

It does not decide when a regulation took effect. That comes from the approval
system and is applied by jobs/20_effective_dates.sql. A document's supplementary
provision states an
intended date; the approval is what makes it binding, and the two disagree often
enough that trusting the document is a real source of wrong answers.

What it does own is the part that genuinely needs a filesystem: reading formats,
finding article boundaries, and deciding what each file actually is. Everything
downstream is SQL.

    python -m regdemo_pipeline.jobs.ingest --out ./out --ontul http://localhost:8080
"""
from __future__ import annotations

import argparse
import json
import sys
import time
import uuid
from dataclasses import dataclass, field
from datetime import datetime, timezone
from pathlib import Path

from ..extract import extract
from ..extract.base import Route
from ..chunk.split import chunk as chunk_sections
from ..relations.register import load_register, match, unregistered_rows

BATCH_ROWS = 200          # rows per INSERT — large enough to matter, small enough to read in an error
BUCKET = "iceberg-warehouse"
PREFIX = "corpus/"


# ── SQL literals ────────────────────────────────────────────────────────────
# Hand-built because these statements go through Ontul's REST endpoint, not a
# driver with bind parameters. Everything that reaches here is machine-generated
# corpus text, but it still contains quotes and newlines, and getting this
# wrong produces a syntax error hundreds of rows into a batch.
class Raw(str):
    """A value that is already SQL and must not be quoted."""


def lit(v) -> str:
    if isinstance(v, Raw):
        return str(v)
    if v is None:
        return "NULL"
    if isinstance(v, bool):
        return "TRUE" if v else "FALSE"
    if isinstance(v, (int, float)):
        return str(v)
    s = str(v).replace("\\", "\\\\").replace("'", "''")
    return "'" + s + "'"


def row(*vals) -> str:
    return "(" + ",".join(lit(v) for v in vals) + ")"


@dataclass
class Stats:
    seen: int = 0
    extracted: int = 0
    chunks: int = 0
    uploaded: int = 0
    skipped: dict[str, int] = field(default_factory=dict)
    unmatched: list[str] = field(default_factory=list)

    def skip(self, reason: str) -> None:
        self.skipped[reason] = self.skipped.get(reason, 0) + 1


class Ontul:
    """The master, reached over its admin API.

    Statements go through the master rather than straight to a worker so that
    IAM, lineage and the query log see them. An ingest that bypasses those is an
    ingest nobody can audit afterwards.
    """

    def __init__(self, url: str, user: str, password: str):
        import requests
        self.s = requests.Session()
        self.url = url.rstrip("/")
        r = self.s.post(f"{self.url}/admin/auth/login",
                        json={"username": user, "password": password}, timeout=30)
        r.raise_for_status()
        body = r.json()
        # The response names it accessToken; "token" is empty and produces a
        # Bearer header that fails as Unauthorized rather than as a bad login.
        self.token = body.get("accessToken") or body.get("token")
        if not self.token:
            raise SystemExit(f"login returned no token: {body}")
        self.s.headers["Authorization"] = f"Bearer {self.token}"

    def sql(self, statement: str) -> dict:
        """Run one statement, and treat a reported failure as a failure.

        /admin/query/execute answers 200 for a statement that did not run and puts
        the reason in the body. Checking only the status code makes an ingest
        report every row as loaded while the table stays empty — which is worse
        than crashing, because the next stage then fails somewhere unrelated and
        the count that was wrong is three steps back.
        """
        r = self.s.post(f"{self.url}/admin/query/execute",
                        json={"sql": statement}, timeout=300)
        head = statement[:180].replace("\n", " ")
        if r.status_code >= 300:
            raise SystemExit(f"SQL failed ({r.status_code}): {r.text[:400]}\n  -> {head}")
        body = r.json() if r.content else {}
        if body.get("status") == "error" or body.get("error"):
            raise SystemExit(f"SQL failed: {body.get('error') or body}\n  -> {head}")
        return body

    def count(self, table: str) -> int:
        body = self.sql(f"SELECT count(*) FROM {table}")
        rows = body.get("rows") or []
        return int(rows[0][0]) if rows and rows[0] else 0

    def insert(self, table: str, cols: list[str], rows: list[str], exact: bool = True) -> int:
        """Insert, then check the table actually holds what was inserted.

        Reporting the number of rows sent is not the same as reporting the number
        of rows there. A DELETE that silently did nothing leaves the previous load
        in place, the insert succeeds on top of it, and the job prints a number
        that looks right while the table holds a multiple of it. That stays
        invisible until something downstream counts — which, if it is a retriever,
        means duplicated evidence in an answer rather than an error.
        """
        for i in range(0, len(rows), BATCH_ROWS):
            batch = rows[i:i + BATCH_ROWS]
            self.sql(f"INSERT INTO {table} ({','.join(cols)}) VALUES {','.join(batch)}")
        if not exact:
            # An append-only history: it is supposed to grow across runs, so the
            # count is not a statement about this run.
            return len(rows)
        actual = self.count(table)
        if actual != len(rows):
            raise SystemExit(
                f"{table}: inserted {len(rows)} rows but the table holds {actual}. "
                f"A preceding DELETE did not take effect, or another writer is active.")
        return len(rows)


def s3_client(env: dict):
    import boto3
    from botocore.config import Config
    return boto3.client(
        "s3",
        endpoint_url=env["S3_ENDPOINT_HOST"],
        aws_access_key_id=env["S3_ACCESS_KEY"],
        aws_secret_access_key=env["S3_SECRET_KEY"],
        region_name=env.get("S3_REGION", "us-east-1"),
        config=Config(s3={"addressing_style": "path"}, retries={"max_attempts": 3}),
    )


def read_stack_env(path: Path) -> dict:
    """out/stack.env, written by infra/up.sh. Credentials are minted per
    bring-up, so nothing downstream should hard-code them."""
    env = {}
    if not path.is_file():
        raise SystemExit(f"{path} not found — run infra/up.sh first")
    for line in path.read_text(encoding="utf-8").splitlines():
        line = line.strip()
        if not line.startswith("export "):
            continue
        k, _, v = line[len("export "):].partition("=")
        env[k.strip()] = v.strip()
    return env


def main(argv=None) -> int:
    ap = argparse.ArgumentParser(description=__doc__)
    ap.add_argument("--out", default="./out", type=Path)
    ap.add_argument("--ontul", default=None, help="defaults to ONTUL_URL from stack.env")
    ap.add_argument("--password", default="regdemo-admin-2026")
    ap.add_argument("--user", default="admin")
    ap.add_argument("--limit", type=int, default=0, help="stop after N files (smoke runs)")
    ap.add_argument("--no-upload", action="store_true",
                    help="skip putting the originals in ShannonStore; s3_uri still recorded")
    ap.add_argument("--with-chunks", action="store_true",
                    help="also load doc_chunks from this machine (legacy single-node path). "
                         "The kiok DAG chunks in the cluster instead; loading here as well "
                         "puts every article in the index twice.")
    a = ap.parse_args(argv)

    out: Path = a.out
    env = read_stack_env(out / "stack.env")
    ontul_url = a.ontul or env.get("ONTUL_URL", "http://localhost:8080")

    docs_dir = out / "documents"
    if not docs_dir.is_dir():
        raise SystemExit(f"{docs_dir} not found — run 'make seed' first")

    register = load_register(out / "regulations_master.xlsx")
    files = sorted(p for p in docs_dir.rglob("*") if p.is_file())
    if a.limit:
        files = files[:a.limit]

    run_id = uuid.uuid4().hex[:12]
    # The ingest stamps its own rows. CURRENT_TIMESTAMP is not evaluable in this
    # position — the planner rejects it and the writer, given the word as a
    # string, fails parsing at index 0 — and the moment worth recording is when
    # the stage ran here, not when the commit landed.
    ran_at = Raw("TIMESTAMP '" + datetime.now(timezone.utc).strftime("%Y-%m-%d %H:%M:%S") + "'")
    st = Stats()
    s3 = None if a.no_upload else s3_client(env)

    doc_rows: dict[str, tuple] = {}       # doc_no -> documents row, deduped
    ver_rows: dict[tuple, tuple] = {}     # (doc_no, version) -> doc_versions row
    # Chunks are held per (doc_no, version) rather than appended per file. The
    # same version usually arrives twice — the .docx that was drafted and the
    # .pdf that was published — and appending both puts the same article in the
    # index twice. It survives every count check (799 chunks is 799 chunks) and
    # shows up only at the far end, as an answer citing the same clause twice.
    chunks_by_version: dict[tuple, tuple] = {}   # (doc,ver) -> (file_format, [chunk tuples])
    log_rows: list[str] = []
    seen_sha: dict[str, str] = {}         # sha256 -> first s3_key that carried it
    matched_keys: set[str] = set()

    print(f"=== ingest {run_id} — {len(files)} files ===")
    for n, path in enumerate(files, 1):
        st.seen += 1
        rel = path.relative_to(docs_dir).as_posix()
        s3_key = f"{PREFIX}{rel}"
        s3_uri = f"s3://{BUCKET}/{s3_key}"
        t0 = time.time()

        try:
            ex = extract(path, s3_key)
        except Exception as e:                                  # noqa: BLE001
            st.skip("extract_error")
            log_rows.append(row(run_id, s3_uri, "extract", "error", None,
                                int((time.time() - t0) * 1000), str(e)[:400], ran_at))
            continue

        # A scanned page is not a failure of the pipeline; it is a fact about
        # the file. Recording it as 'skipped/image_only_pdf' is what makes the
        # difference between "we have 411 documents" and "we can answer from 389
        # of them" visible to whoever has to trust this.
        if ex.route is Route.UNREADABLE:
            st.skip(ex.skip_reason or "unreadable")
            log_rows.append(row(run_id, s3_uri, "extract", "skipped", ex.skip_reason,
                                int((time.time() - t0) * 1000), None, ran_at))
            continue

        # The same regulation is routinely present twice: the .docx someone
        # drafted and the .pdf that was published. Both are real; only one is
        # authoritative, and it is the published one.
        first = seen_sha.get(ex.sha256)
        is_authoritative = True
        if first is not None:
            is_authoritative = False
            st.skip("duplicate_sha")
            log_rows.append(row(run_id, s3_uri, "extract", "skipped", "duplicate_sha",
                                int((time.time() - t0) * 1000), f"same bytes as {first}",
                                ran_at))
        else:
            seen_sha[ex.sha256] = s3_key

        m = match(s3_key, ex.header_doc_no, ex.header_version, register)
        if not m.doc_no or m.version is None:
            st.unmatched.append(s3_key)
            log_rows.append(row(run_id, s3_uri, "discover", "skipped", m.reason or "no_doc_no",
                                int((time.time() - t0) * 1000), None, ran_at))
            continue
        matched_keys.add(m.doc_no)

        reg = next((r for r in register if r.doc_no == m.doc_no), None)
        if reg is None:
            st.unmatched.append(s3_key)
            continue

        st.extracted += 1
        if s3 is not None:
            s3.put_object(Bucket=BUCKET, Key=s3_key, Body=path.read_bytes())
            st.uploaded += 1

        doc_rows.setdefault(m.doc_no, (
            m.doc_no, reg.title, reg.kind,
            int(m.doc_no.split("-")[-1][0]) if m.doc_no.split("-")[-1][:1].isdigit() else 2,
            reg.dept,
            # is_official — is this a document the register marks as citable as the
            # basis for an answer?
            #
            # A different axis from the classification. This flag says whether the
            # document is a citable regulation; who may see it is decided by IAM.
            # It once doubled as the classification — restricted meant
            # is_official=False — and that stopped HR from citing those regulations
            # too. Excluding a document from the index and showing it to different
            # people differently are different jobs, and only policy can do the
            # second.
            #
            # Membership of the register is the curation. Files that are not in it —
            # meeting notes, announcements, memos — never become a documents row at
            # all.
            True,
            reg.sensitivity, "onedrive",
        ))

        # effective_from is left NULL on purpose. stated_from is what the
        # document claims; only the approval system can say what is binding, and
        # 20_effective_dates.sql applies it.
        # One version routinely arrives as two files: the .docx someone drafted and
        # the .pdf that was published. Both are real. Overwriting by arrival order
        # would let the draft win whenever it sorts later, so the published form
        # takes the row and the draft is recorded as non-authoritative.
        key = (m.doc_no, m.version)
        row_v = (
            m.doc_no, m.version, "DRAFT", None, None,
            ex.header_stated_date, None, None,
            s3_uri, ex.file_format, ex.sha256, is_authoritative, ex.page_count,
        )
        prev = ver_rows.get(key)
        if prev is None:
            ver_rows[key] = row_v
        else:
            prev_fmt, new_fmt = prev[9], ex.file_format
            if new_fmt == "pdf" and prev_fmt != "pdf":
                ver_rows[key] = row_v            # published form supersedes the draft
            # else: keep what is already there — an equally-authoritative duplicate
            # or a draft arriving after the published form.

        chunks = chunk_sections(ex.sections)
        prev_chunks = chunks_by_version.get(key)
        if prev_chunks is None or (ex.file_format == "pdf" and prev_chunks[0] != "pdf"):
            chunks_by_version[key] = (ex.file_format, chunks)
        log_rows.append(row(run_id, s3_uri, "chunk", "ok", None,
                            int((time.time() - t0) * 1000), None, ran_at))

        if n % 50 == 0:
            print(f"  {n}/{len(files)}  versions={len(chunks_by_version)}")

    # Numbering happens after the winners are known, so the key is dense and
    # stable for a given ledger rather than an artefact of walk order.
    chunk_rows: list[str] = []
    next_chunk_pk = 1
    for (doc_no, version), (_fmt, chunks) in sorted(chunks_by_version.items()):
        for c in chunks:
            chunk_rows.append(row(
                next_chunk_pk,
                f"{doc_no}#{version}#{c.ordinal}", doc_no, version, c.ordinal,
                c.article_no, c.article_title, c.page_from, c.page_to,
                c.text, c.text_redacted, json.dumps(c.pii_types, ensure_ascii=False),
                len(c.text), None,
            ))
            next_chunk_pk += 1
    st.chunks = len(chunk_rows)

    # ── Load. Ledger first: a chunk whose document is absent is unciteable.
    o = Ontul(ontul_url, a.user, a.password)
    print("\n=== loading ===")

    o.sql("DELETE FROM ice.reg.doc_chunks")
    o.sql("DELETE FROM ice.reg.doc_versions")
    o.sql("DELETE FROM ice.reg.documents")
    # doc_chunks is cleared either way. This stage owns the ledger, and the
    # chunks belonging to a ledger that is about to be replaced are stale
    # whether or not this run is the one that refills them.
    #
    # Clearing them means the ingest queue is now lying: it says EXTRACTED for
    # objects whose chunks no longer exist, and the DAG's chunk task only reads
    # rows still marked PENDING. Left alone, the next pipeline run reports
    # success and produces nothing — the queue is drained, the ledger is fresh,
    # and doc_chunks stays empty, which downstream reads as "this corpus has no
    # text in it". Whoever invalidates the chunks has to invalidate the claim
    # that they were made.
    o.sql("UPDATE ice.reg.ingest_queue SET status = 'PENDING' WHERE status = 'EXTRACTED'")

    # The graph vertex ids are assigned here. Numbered in doc_no order, so the same
    # set of regulations always produces the same values and an edge written
    # yesterday still points at the same document after the graph is rebuilt.
    doc_ids = {d: i + 1 for i, d in enumerate(sorted(doc_rows))}
    n = o.insert("ice.reg.documents",
                 ["doc_id", "doc_no", "title", "doc_class", "tier", "owner_dept",
                  "is_official", "sensitivity", "source_system"],
                 [row(doc_ids[k], *v) for k, v in doc_rows.items()])
    print(f"  documents      {n}")

    n = o.insert("ice.reg.doc_versions",
                 ["doc_no", "version", "status", "effective_from", "effective_to",
                  "stated_from", "date_mismatch", "approval_id",
                  "s3_uri", "file_format", "sha256", "is_authoritative", "page_count"],
                 [row(*v) for v in ver_rows.values()])
    print(f"  doc_versions   {n}")

    # Chunking belongs to the cluster now. The DAG's `chunk` task calls
    # extract_chunks() per row of the ingest queue, so the work spreads with the
    # scan instead of running here on one machine — which is the only shape that
    # survives a corpus that does not fit on one.
    #
    # Loading them here as well is not a smaller version of the same thing: the
    # DAG would then insert the same articles again, the count would double, and
    # nothing would report it. Every check that counts rows still passes, and the
    # only visible symptom is an answer citing the same clause twice.
    if a.with_chunks:
        n = o.insert("ice.reg.doc_chunks",
                     ["chunk_pk", "chunk_id", "doc_no", "version", "ordinal_no", "article_no", "article_title",
                      "page_from", "page_to", "text", "text_redacted", "pii_types",
                      "token_count", "created_at"],
                     chunk_rows)
        print(f"  doc_chunks     {n}   (--with-chunks; the DAG normally does this)")
    else:
        print(f"  doc_chunks     0     (the DAG's chunk task fills these; the {st.chunks} extracted here are discarded)")

    n = o.insert("ice.reg.ingest_log",
                 ["run_id", "s3_uri", "stage", "status", "reason", "elapsed_ms", "error", "ran_at"],
                 log_rows, exact=False)
    print(f"  ingest_log     {n}")

    missing = unregistered_rows(register, matched_keys)
    print(f"""
=== ingest {run_id} ===
  files seen         {st.seen}
  documents ingested {st.extracted}   (uploaded to S3: {st.uploaded})
  chunks             {st.chunks}
  skipped            {sum(st.skipped.values())}  {st.skipped or ''}
  unmatched files    {len(st.unmatched)}
  register rows with no file  {len(missing)}  {missing[:5]}

  Nothing above is an error on its own. A corpus with no skipped files and no
  unmatched rows would mean the generator built something tidier than a real
  shared drive, which would make every downstream number look better than it
  should.

  Next: bash infra/pipeline.sh   (kiok DAG: chunk, dates, vectors, graph)
""")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```


!!! danger "Chunks are not loaded here"
    They used to be. The DAG's `chunk` task loads the same chunks, so 818 became
    1636 — and **every count-based check still passed**. The only visible symptom
    was an answer citing the same clause twice.

    And clearing those chunks means the ingest queue has to be reset too. Left
    saying `EXTRACTED`, the next run drains an empty queue and reports success:
    the ledger is fresh, the index is empty, and nothing says anything failed.

### The extractors

**`demo/pipeline/src/regdemo_pipeline/extract/__init__.py`**

```python
"""Dispatch on format. PDF only where PDF is the original."""
from __future__ import annotations

from pathlib import Path

from .base import Extracted, Route, Section

_BY_SUFFIX = {".pdf": "pdf", ".docx": "docx", ".xlsx": "xlsx"}


def extract(path: Path, s3_key: str) -> Extracted:
    kind = _BY_SUFFIX.get(path.suffix.lower())
    if kind == "pdf":
        from . import pdf
        return pdf.extract(path, s3_key)
    if kind == "docx":
        from . import docx as docx_mod
        return docx_mod.extract(path, s3_key)
    if kind == "xlsx":
        from . import xlsx
        return xlsx.extract(path, s3_key)
    return Extracted(s3_key=s3_key, file_format=path.suffix.lstrip("."),
                     route=Route.UNREADABLE, sha256="",
                     skip_reason="unsupported_format")
```
**`demo/pipeline/src/regdemo_pipeline/extract/base.py`**

```python
"""Format-aware extraction.

DOCX and XLSX are structured formats: heading levels, table cells and styles are
already metadata. Converting them to PDF first throws that away and forces the
structure to be guessed back from a visual layout, so each format is read
natively and PDF is used only where PDF is the original.

Prose and tables take different routes. A spreadsheet is not prose — chunking a
register into text produces context-free cells, so tables land in Iceberg as
rows and only prose is embedded.
"""
from __future__ import annotations

import hashlib
import re
from dataclasses import dataclass, field
from enum import Enum
from pathlib import Path


class Route(Enum):
    PROSE = "prose"      # chunk and embed
    TABLE = "table"      # load as rows; never embedded
    UNREADABLE = "unreadable"


@dataclass
class Section:
    """An article where the document has them, otherwise a paragraph block."""
    article_no: str | None      # the article number, as printed in the document
    title: str | None           # the article heading, in parentheses
    text: str
    page_from: int
    page_to: int


@dataclass
class Extracted:
    s3_key: str
    file_format: str
    route: Route
    sha256: str
    sections: list[Section] = field(default_factory=list)
    tables: list[list[list[str]]] = field(default_factory=list)
    # Recovered from the running header, which survives a filename that does not
    # say which version it is.
    header_doc_no: str | None = None
    header_version: int | None = None
    header_stated_date: str | None = None
    page_count: int = 0
    skip_reason: str | None = None


ARTICLE_RE = re.compile(r"^제\s*(\d+)\s*조\s*(\([^)]*\))?")
HEADER_RE = re.compile(
    r"([A-Z]{2,4}-(?:RUL|REG|GDL)-\d{3})\s+v(\d+)\s+시행\s+(\d{4}-\d{2}-\d{2})")


def sha256_of(data: bytes) -> str:
    return hashlib.sha256(data).hexdigest()


def split_articles(text: str, page_from: int = 1, page_to: int = 1) -> list[Section]:
    """Split on article boundaries.

    Article-level chunks are the right unit here: an article is what a regulation
    citation points at, so a retrieved chunk maps onto something a person can
    verify. Text before the first article (cover page, metadata table) is dropped —
    it is layout, not content.
    """
    lines = [ln.strip() for ln in text.splitlines()]
    sections: list[Section] = []
    cur_no: str | None = None
    cur_title: str | None = None
    buf: list[str] = []

    def flush() -> None:
        if cur_no and buf:
            body = " ".join(x for x in buf if x).strip()
            if body:
                sections.append(Section(cur_no, cur_title, body, page_from, page_to))

    for ln in lines:
        m = ARTICLE_RE.match(ln)
        if m:
            flush()
            cur_no = f"제{m.group(1)}조"
            cur_title = m.group(2)
            rest = ln[m.end():].strip()
            buf = [rest] if rest else []
        elif cur_no is not None:
            buf.append(ln)
    flush()
    return sections


def parse_header(text: str) -> tuple[str | None, int | None, str | None]:
    """Recover doc_no / version / stated date from the running header.

    Filenames are unreliable — one marked "final" with a date in it names a superseded
    version — so identity comes from the page header and, authoritatively, from
    matching against the register.
    """
    m = HEADER_RE.search(text)
    return (m.group(1), int(m.group(2)), m.group(3)) if m else (None, None, None)
```
**`demo/pipeline/src/regdemo_pipeline/extract/pdf.py`**

```python
"""PDF extraction. Used where PDF is the original — scans and published copies."""
from __future__ import annotations

from pathlib import Path

import pdfplumber

from .base import Extracted, Route, parse_header, sha256_of, split_articles

# Below this, a PDF is a scan: glyphs were never embedded, so no extractor will
# ever read it. Reported rather than dropped — a document that silently never
# entered the index is the failure this corpus exists to expose.
MIN_CHARS_PER_PAGE = 40


def extract(path: Path, s3_key: str) -> Extracted:
    data = path.read_bytes()
    out = Extracted(s3_key=s3_key, file_format="pdf", route=Route.PROSE,
                    sha256=sha256_of(data))
    texts: list[str] = []
    with pdfplumber.open(path) as doc:
        out.page_count = len(doc.pages)
        for page in doc.pages:
            texts.append(page.extract_text() or "")

    full = "\n".join(texts)
    if out.page_count and len(full.strip()) < MIN_CHARS_PER_PAGE * out.page_count:
        out.route = Route.UNREADABLE
        out.skip_reason = "image_only_pdf"
        return out

    out.header_doc_no, out.header_version, out.header_stated_date = parse_header(full)
    out.sections = split_articles(full, 1, out.page_count)
    if not out.sections:
        # No article structure — a general document. Keep it as one block per page so
        # it is still searchable, but it carries no article citation.
        out.sections = [
            __import__("regdemo_pipeline.extract.base", fromlist=["Section"]).Section(
                None, None, t.strip(), i + 1, i + 1)
            for i, t in enumerate(texts) if t.strip()]
    return out
```
**`demo/pipeline/src/regdemo_pipeline/extract/docx.py`**

```python
"""DOCX extraction — read natively, never via a PDF conversion.

Heading levels and table cells are already metadata here. A conversion to PDF
flattens both into a visual layout that then has to be reconstructed by
guesswork, which is strictly worse than reading what is already there.
"""
from __future__ import annotations

from pathlib import Path

from docx import Document

from .base import Extracted, Route, Section, parse_header, sha256_of, split_articles


def extract(path: Path, s3_key: str) -> Extracted:
    data = path.read_bytes()
    out = Extracted(s3_key=s3_key, file_format="docx", route=Route.PROSE,
                    sha256=sha256_of(data))
    doc = Document(path)

    lines: list[str] = []
    for p in doc.paragraphs:
        t = p.text.strip()
        if not t:
            continue
        # A heading styled as such already tells us it starts a section; keep the
        # text as-is so the shared article splitter sees the same shape it sees in PDF.
        lines.append(t)

    # Tables travel with the prose that surrounds them rather than being embedded
    # separately: a cell on its own has no context to retrieve on.
    for t in doc.tables:
        rows = [[c.text.strip() for c in r.cells] for r in t.rows]
        if rows:
            out.tables.append(rows)

    full = "\n".join(lines)
    out.page_count = 1
    out.header_doc_no, out.header_version, out.header_stated_date = parse_header(full)
    out.sections = split_articles(full)
    if not out.sections and full.strip():
        out.sections = [Section(None, None, full.strip(), 1, 1)]
    return out
```
**`demo/pipeline/src/regdemo_pipeline/extract/xlsx.py`**

```python
"""XLSX extraction — rows, not chunks.

A spreadsheet is not prose. Chunking a register into text produces cells with no
context to retrieve on, and embedding them pollutes the index with fragments
that match everything weakly. Sheets become Iceberg rows instead; the register
in particular is the master for document identity, not a search target.
"""
from __future__ import annotations

from pathlib import Path

from openpyxl import load_workbook

from .base import Extracted, Route, sha256_of


def extract(path: Path, s3_key: str) -> Extracted:
    out = Extracted(s3_key=s3_key, file_format="xlsx", route=Route.TABLE,
                    sha256=sha256_of(path.read_bytes()))
    wb = load_workbook(path, read_only=True, data_only=True)
    for ws in wb.worksheets:
        rows = [["" if c is None else str(c) for c in row]
                for row in ws.iter_rows(values_only=True)]
        rows = [r for r in rows if any(x.strip() for x in r)]
        if rows:
            out.tables.append(rows)
    wb.close()
    return out
```


### Chunking, by article

**`demo/pipeline/src/regdemo_pipeline/chunk/split.py`**

```python
"""Chunking.

One article is one chunk wherever a document has them: it is the unit a citation
points at, so a retrieved chunk maps onto something a person can open and check.
General documents have no articles, so they fall back to paragraph blocks bounded by
token budget.

Long articles are split, but on sentence boundaries and with the article number
carried onto every part — a fragment that cannot say which article it came from is
useless as an answer's evidence no matter how well it matches.
"""
from __future__ import annotations

import re
from dataclasses import dataclass

from ..extract.base import Section
from ..pii.detect import redact

# The embedding model's window is 512 tokens. Korean runs roughly 1.5 chars per
# token for this model, so this is a conservative character budget that leaves
# room for the "passage: " prefix.
MAX_CHARS = 600
SENT_END = re.compile(r"(?<=[.。」』])\s+|(?<=다\.)\s+")


@dataclass
class Chunk:
    ordinal: int
    article_no: str | None
    article_title: str | None
    text: str
    text_redacted: str
    pii_types: list[str]
    page_from: int
    page_to: int

    @property
    def has_pii(self) -> bool:
        return bool(self.pii_types)


def _split_long(text: str) -> list[str]:
    if len(text) <= MAX_CHARS:
        return [text]
    parts, cur = [], ""
    for sent in SENT_END.split(text):
        if not sent:
            continue
        if len(cur) + len(sent) + 1 > MAX_CHARS and cur:
            parts.append(cur.strip())
            cur = sent
        else:
            cur = f"{cur} {sent}".strip()
    if cur.strip():
        parts.append(cur.strip())
    return parts


def chunk(sections: list[Section]) -> list[Chunk]:
    out: list[Chunk] = []
    for sec in sections:
        pieces = _split_long(sec.text)
        for i, piece in enumerate(pieces):
            # The article number and title ride on the text itself, not only in
            # metadata: the embedding sees them, so an article number and its heading
            # together match a
            # query naming the article as well as one naming the topic.
            head = ""
            if sec.article_no:
                head = f"{sec.article_no}{sec.title or ''} "
                if len(pieces) > 1:
                    head = f"{sec.article_no}{sec.title or ''} ({i + 1}/{len(pieces)}) "
            body = head + piece
            r = redact(body)
            out.append(Chunk(len(out), sec.article_no, sec.title, body,
                             r.text, r.types, sec.page_from, sec.page_to))
    return out
```


### PII detection

**`demo/pipeline/src/regdemo_pipeline/pii/detect.py`**

```python
"""PII detection and redaction for free text.

Column masking cannot help here. A regulation's prose is one VARCHAR, so masking
it wholesale would return an empty search result rather than a protected one —
the value is in the sentence, not in a field. So the pipeline produces a
redacted twin of every chunk and lets masking swap between the two columns:
text → CASE WHEN clearance THEN text ELSE text_redacted END.

Rule-based on purpose. These patterns are well-formed identifiers with check
digits and fixed shapes, and a rule that misses is visible in the ground-truth
score; a model that misses is not.
"""
from __future__ import annotations

import re
from dataclasses import dataclass

# National ID numbers. Denied rather than masked on the ERP side, but in prose
# they have to be
# redacted in place — there is no column to remove.
RRN = re.compile(r"\b(\d{6})-([1-4]\d{6})\b")
PHONE = re.compile(r"\b(01[016-9])-?(\d{3,4})-?(\d{4})\b")
EMP_NO = re.compile(r"\b(20[0-2]\d)(\d{4})\b")
EMAIL = re.compile(r"\b[\w.+-]+@[\w-]+\.[\w.]+\b")
# Deliberately does not start with a mobile prefix: without that guard this
# pattern swallows 010-7850-3702 before the phone rule sees it, and the number
# comes back fully masked instead of keeping its last four digits.
ACCOUNT = re.compile(r"\b(?!01[016-9]-)\d{3,6}-\d{2,6}-\d{4,8}\b")


@dataclass
class Redaction:
    text: str
    types: list[str]


def redact(text: str) -> Redaction:
    """Return the text with identifiers replaced, and which kinds were found.

    Shape is preserved (last four digits of a phone, the leading date part of an
    RRN) because a reader has to be able to tell that something was removed and
    roughly what — a blanket ██ makes a redacted document unreviewable.
    """
    found: list[str] = []

    def mark(kind: str):
        def sub(m: re.Match) -> str:
            if kind not in found:
                found.append(kind)
            if kind == "RRN":
                return f"{m.group(1)}-*******"
            if kind == "PHONE":
                return f"{m.group(1)}-****-{m.group(3)}"
            if kind == "EMP_NO":
                return f"{m.group(1)}****"
            if kind == "EMAIL":
                return "***@***"
            return "***-**-****"
        return sub

    out = RRN.sub(mark("RRN"), text)
    out = PHONE.sub(mark("PHONE"), out)
    out = ACCOUNT.sub(mark("ACCOUNT"), out)
    out = EMAIL.sub(mark("EMAIL"), out)
    out = EMP_NO.sub(mark("EMP_NO"), out)
    return Redaction(out, found)
```


---

## 2. Registering the UDFs

The `chunk` task calls `extract_chunks(uri)`. It has to be registered at **GLOBAL
scope**: a session-scoped UDF is visible only to the connection that registered
it, and the scheduler's task opens its own — which ends as
`No match found for function signature`.

**`demo/pipeline/src/regdemo_pipeline/jobs/register_udfs.py`**

```python
"""Register the UDFs the pipeline uses with the cluster.

Registered at session scope, a UDF is visible only to the connection that
registered it. That is right while exploring and wrong for a pipeline: the
scheduler's task opens its own connection and is told the function does not
exist. GLOBAL persists on the server and survives both sessions and restarts.

    python -m regdemo_pipeline.jobs.register_udfs --out ./out

cloudpickle serialises the function by value, so this module does not have to be
installed on the worker. What does have to match is the Python version on both
sides: a code object's layout is version-specific, and a mismatch produces
"code expected at most 16 arguments" — a message that mentions neither Python nor
a version.
"""
from __future__ import annotations

import argparse
import os
import sys
from pathlib import Path


def main(argv=None) -> int:
    ap = argparse.ArgumentParser(prog="register_udfs")
    ap.add_argument("--out", default="./out")
    ap.add_argument("--ontul", default=None)
    ap.add_argument("--password", default=os.environ.get("ONTUL_ADMIN_PASSWORD", "regdemo-admin-2026"))
    ap.add_argument("--scope", default="GLOBAL", choices=["GLOBAL", "USER", "TEMPORARY"])
    a = ap.parse_args(argv)

    from .ingest import read_stack_env, Ontul
    env = read_stack_env(Path(a.out) / "stack.env")
    url = a.ontul or env.get("ONTUL_URL", "http://localhost:8080")
    ontul = Ontul(url, "admin", a.password)

    import cloudpickle
    from ontul.session import OntulSession
    from ..udf import extract_chunks as mod

    # By value, not by reference. Serialising by reference would require the same
    # module to be installed on the worker, which means rebuilding the image every
    # time the extractor changes.
    cloudpickle.register_pickle_by_value(mod)

    s3cfg = {
        "endpoint": env["S3_ENDPOINT_INTERNAL"],
        "access_key": env["S3_ACCESS_KEY"],
        "secret_key": env["S3_SECRET_KEY"],
        "region": env.get("S3_REGION", "us-east-1"),
    }

    def extract_chunks(s3_uri):
        # The worker has no stack.env, so the credentials travel in the closure.
        os.environ.setdefault("REGDEMO_S3_ENDPOINT", s3cfg["endpoint"])
        os.environ.setdefault("REGDEMO_S3_ACCESS_KEY", s3cfg["access_key"])
        os.environ.setdefault("REGDEMO_S3_SECRET_KEY", s3cfg["secret_key"])
        os.environ.setdefault("REGDEMO_S3_REGION", s3cfg["region"])
        return mod.extract_chunks(s3_uri)

    def jfield(doc, key):
        # Chunks travel as JSON because the engine does not carry the element type of
        # a struct array in the schema, and JSON_VALUE is not implemented either.
        # Not needing DDL when a field is added is a bonus.
        import json as _json
        if not doc:
            return None
        try:
            v = _json.loads(doc).get(key)
        except Exception:                                     # noqa: BLE001
            return None
        return None if v is None else str(v)

    # ── Matching the document number ────────────────────────────────────────
    # Filenames cannot be trusted — one marked "final" carries no document number,
    # and one marked "copy" does not say what it is a copy of. The register is the
    # authority, and only the result of matching against it reaches the ledger.
    #
    # The register travels in the closure. That is possible because it is 50 rows,
    # and it also means the function is fixed to the register as it stood at
    # registration — add a regulation and the UDF has to be registered again. That
    # is better than the worker querying the ledger for every row.
    body = ontul.sql("SELECT doc_no, title FROM ice.reg.documents")
    register = [(r[0], r[1]) for r in (body.get("rows") or []) if r[0] and r[1]]
    if not register:
        raise SystemExit("the register is empty — there are no documents in the ledger")
    # Longer titles are tried first. A shorter title matching before the longer one
    # it is contained in would send the match to the wrong document.
    register.sort(key=lambda t: -len(t[1]))

    def doc_no_for(object_key):
        """The document number from a file path: the number if the name carries one,
        otherwise a title match."""
        import re as _re
        if not object_key:
            return None
        m = _re.search(r"([A-Z]{2,4}-[A-Z]{3}-\d{3})", object_key)
        if m:
            return m.group(1)
        # Matched on a stem with the extension and the usual decorations stripped off.
        # The ledger title is often longer than what the filename carries, so testing
        # only whether the title appears in the filename would never match.
        stem = object_key.rsplit("/", 1)[-1]
        stem = _re.sub(r"\.[A-Za-z0-9]+$", "", stem)
        stem = _re.sub(r"[\[(].*?[\])]", " ", stem)
        stem = _re.sub(r"(사본|최종|수정|복사본|v\d+|\d{4}[.\-]\d{2}[.\-]\d{2})", " ", stem)
        stem = _re.sub(r"[_\s]+", " ", stem).strip()
        for no, title in register:
            if not title:
                continue
            if title in stem or stem in title:
                return no
        return None

    def chunk_pk(chunk_id):
        """A stable 64-bit integer derived from chunk_id.

        NeorunBase's FTS index requires a numeric primary key, and HYBRID_SEARCH
        returns that key as its id. Numbering the rows would need a window function,
        which the engine does not have, so the key is derived from the value instead:
        the same chunk gets the same key however many times it is re-indexed, and a
        re-index replaces the existing row rather than colliding with it.
        """
        import hashlib
        if not chunk_id:
            return None
        h = hashlib.blake2b(str(chunk_id).encode("utf-8"), digest_size=8).digest()
        # Kept inside a signed 64-bit range. A negative primary key is fine for the
        # index and confusing to read, so the top bit is dropped.
        return int.from_bytes(h, "big") & 0x7FFF_FFFF_FFFF_FFFF

    host = url.split("//", 1)[-1].split(":")[0]
    session = OntulSession(host=host, token=ontul.token)
    session.register_udf("extract_chunks", extract_chunks, return_type="list",
                         param_types=["string"], element_type="string", scope=a.scope)
    session.register_udf("jfield", jfield, return_type="string",
                         param_types=["string", "string"], scope=a.scope)
    session.register_udf("doc_no_for", doc_no_for, return_type="string",
                         param_types=["string"], scope=a.scope)
    session.register_udf("chunk_pk", chunk_pk, return_type="long",
                         param_types=["string"], scope=a.scope)
    print(f"registered extract_chunks, jfield, doc_no_for, chunk_pk "
          f"(scope={a.scope}, register={len(register)} documents)")

    # Checked from a separate connection. A response saying it registered and the
    # function actually being usable are different claims; unchecked here, the
    # difference surfaces inside the DAG.
    probe = ontul.sql("SELECT jfield('{\"k\":\"v\"}', 'k') AS v")
    got = (probe.get("rows") or [[None]])[0][0]
    if got != "v":
        raise SystemExit(f"UDF registered but not usable from a new connection: got {got!r}")
    print("verified from a separate connection")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```


The UDF body. cloudpickle serialises the function **by value**, so the module does
not have to be installed on the worker.

**`demo/pipeline/src/regdemo_pipeline/udf/extract_chunks.py`**

```python
"""A UDF that runs extraction and chunking on the workers.

This module exists because of where it runs. The same code is already in
``regdemo_pipeline.extract`` / ``.chunk``, but that version runs on the driver:
it opens the files and pushes the results. As the file count grows that one
machine is the bottleneck, and further along the listing alone does not fit in
memory.

The function here executes inside an Ontul worker's Python process. Its input is
one S3 URI and its output is the chunks of that document, so the pipeline becomes::

    INSERT INTO ice.reg.doc_chunks
    SELECT q.s3_uri, c.ordinal_no, c.article_no, c.text
    FROM ice.reg.ingest_queue q,
         UNNEST(extract_chunks(q.s3_uri)) AS c

The driver holds nothing. Each worker sees only the rows in its own split and
reads only the objects those rows point at.

Returning an array is the crux: expressing 1→N in SQL — one document becoming
many chunks — needs UNNEST, and without it the work ends up back on the driver.
"""
from __future__ import annotations

import json
import os

# The worker process is not restarted for each UDF call. Keeping the client and
# the extractors at module level avoids rebuilding them per document — invisible at
# 442 files and the whole difference at hundreds of thousands.
_S3 = None


def _s3():
    global _S3
    if _S3 is None:
        import boto3
        from botocore.config import Config
        _S3 = boto3.client(
            "s3",
            endpoint_url=os.environ.get("REGDEMO_S3_ENDPOINT", "http://api-server-1:8080"),
            aws_access_key_id=os.environ.get("REGDEMO_S3_ACCESS_KEY", ""),
            aws_secret_access_key=os.environ.get("REGDEMO_S3_SECRET_KEY", ""),
            region_name=os.environ.get("REGDEMO_S3_REGION", "us-east-1"),
            config=Config(s3={"addressing_style": "path"}, retries={"max_attempts": 3}),
        )
    return _S3


def extract_chunks(s3_uri: str) -> list:
    """One document, as an array of chunk JSON strings.

    Each element is a JSON object. ARRAY<VARCHAR> rather than ARRAY<ROW> because
    the engine does not yet carry the element type of a struct array in the schema;
    a single layer of JSON works around that and also means adding a field needs no
    schema change.

    Failures come back as values. Raising would kill the whole batch and leave no
    record of which document caused it, so instead a single element carrying an
    error is emitted and the pipeline records it as FAILED.
    """
    if not s3_uri:
        return []
    try:
        without = s3_uri[len("s3://"):] if s3_uri.startswith("s3://") else s3_uri
        bucket, _, key = without.partition("/")
        body = _s3().get_object(Bucket=bucket, Key=key)["Body"].read()
    except Exception as e:                                   # noqa: BLE001
        return [json.dumps({"error": f"read failed: {type(e).__name__}: {e}",
                            "ordinal_no": 0}, ensure_ascii=False)]

    try:
        sections = _sections(key, body)
    except Exception as e:                                   # noqa: BLE001
        return [json.dumps({"error": f"extract failed: {type(e).__name__}: {e}",
                            "ordinal_no": 0}, ensure_ascii=False)]

    out = []
    for i, sec in enumerate(sections):
        out.append(json.dumps({
            "ordinal_no": i,
            "article_no": sec.get("article_no"),
            "article_title": sec.get("article_title"),
            "page_from": sec.get("page_from"),
            "page_to": sec.get("page_to"),
            "text": sec.get("text", ""),
            "token_count": len(sec.get("text", "")) // 2,
        }, ensure_ascii=False))
    return out


def _sections(key: str, body: bytes) -> list:
    """Extraction per format. Chosen by extension rather than by sniffing content
    because, unreliable as the filenames in this corpus are, the extension is the
    one part the originating system put there."""
    import io
    lower = key.lower()
    if lower.endswith(".pdf"):
        return _pdf(io.BytesIO(body))
    if lower.endswith(".docx"):
        return _docx(io.BytesIO(body))
    if lower.endswith(".xlsx"):
        return _xlsx(io.BytesIO(body))
    return [{"text": body.decode("utf-8", errors="replace")}]


def _pdf(buf) -> list:
    import pdfplumber
    pages = []
    with pdfplumber.open(buf) as pdf:
        for n, page in enumerate(pdf.pages, start=1):
            pages.append((n, page.extract_text() or ""))
    return _split_articles(pages)


def _docx(buf) -> list:
    import docx
    d = docx.Document(buf)
    text = "\n".join(p.text for p in d.paragraphs)
    return _split_articles([(1, text)])


def _xlsx(buf) -> list:
    import openpyxl
    wb = openpyxl.load_workbook(buf, read_only=True, data_only=True)
    rows = []
    for ws in wb.worksheets:
        for row in ws.iter_rows(values_only=True):
            cells = [str(c) for c in row if c is not None]
            if cells:
                rows.append(" | ".join(cells))
    return [{"text": "\n".join(rows)}]


def _split_articles(pages: list) -> list:
    """Split on article boundaries. Not fixed-length, because an answer has to cite
    an article and a fixed-length chunk cuts through the middle of one, leaving
    nothing whole to cite."""
    import re
    article = re.compile(r"^\s*(제\s*\d+\s*조(?:의\s*\d+)?)\s*[\(（]?([^\)）\n]*)[\)）]?")
    out, cur = [], None
    for page_no, text in pages:
        for line in (text or "").splitlines():
            m = article.match(line)
            if m:
                if cur:
                    out.append(cur)
                cur = {"article_no": m.group(1).replace(" ", ""),
                       "article_title": (m.group(2) or "").strip() or None,
                       "page_from": page_no, "page_to": page_no, "text": line}
            elif cur is not None:
                cur["text"] += "\n" + line
                cur["page_to"] = page_no
            else:
                cur = {"article_no": None, "article_title": None,
                       "page_from": page_no, "page_to": page_no, "text": line}
    if cur:
        out.append(cur)
    # An empty chunk in the index takes up a search result and answers nothing.
    return [s for s in out if s.get("text", "").strip()]
```


---

## 3. Publishing the job sources

A PYTHON job's script is uploaded to S3 and the DAG refers to it through a
connection.

**`demo/pipeline/jobs/publish.sh`**

```bash
#!/usr/bin/env bash
# Publish the job sources to S3 and record their URIs in a kiok connection.
#
#   bash pipeline/jobs/publish.sh
#
# The content hash goes into the key. Ontul's dependency fetcher caches on the
# worker assuming that the same key means the same bytes, so overwriting a fixed
# path means an edited script never runs again — with no error at all, the old code
# keeps going. With the hash in the key, a changed file gets a new key and an
# unchanged one keeps its valid cache.
#
# And the DAG does not name these paths. The regdemoJobs connection holds the
# current URIs and the DAG refers to ${conn.regdemoJobs.<name>} — so the DAG does
# not have to be edited every time a file changes, and which version ran can be
# read off the connection.
set -uo pipefail
cd "$(dirname "$0")/../.."
DEMO=$(pwd)
. out/stack.env 2>/dev/null || { echo "no out/stack.env"; exit 1; }

KIOK=${KIOK_URL:-http://localhost:18081}
KIOK_PW=${KIOK_PASSWORD:-Regdemo-kiok-2026}

export AWS_ACCESS_KEY_ID=$S3_ACCESS_KEY AWS_SECRET_ACCESS_KEY=$S3_SECRET_KEY
export AWS_DEFAULT_REGION=${S3_REGION:-us-east-1}
aws configure set default.s3.addressing_style path 2>/dev/null
BUCKET=$(echo "${S3_WAREHOUSE:-s3://iceberg-warehouse/}" | sed 's#^s3://##; s#/.*##')
PREFIX="pipeline/jobs"

declare -a NAMES URIS
for f in pipeline/jobs/*.py; do
  base=$(basename "$f" .py)
  sha=$(shasum -a 256 "$f" | cut -c1-12)
  key="$PREFIX/$sha-$base.py"
  aws --endpoint-url "$S3_ENDPOINT_HOST" s3 cp "$f" "s3://$BUCKET/$key" >/dev/null \
    || { echo "publish failed: $f"; exit 1; }
  NAMES+=("$base"); URIS+=("s3://$BUCKET/$key")
  echo "  $base -> $sha"
done

# Record the current URIs in the kiok connection.
KTOK=$(curl -s -m 30 -X POST "$KIOK/api/v1/auth/login" -H 'Content-Type: application/json' \
       -d "{\"user\":\"admin\",\"password\":\"$KIOK_PW\"}" \
     | python3 -c "import sys,json;print(json.load(sys.stdin).get('accessToken',''))" 2>/dev/null)
if [ -z "$KTOK" ]; then
  echo "  (kiok login failed — pipeline.sh will update the connection)"
  exit 0
fi
BODY=$(python3 - "${NAMES[@]}" -- "${URIS[@]}" <<'PY'
import json, sys
argv = sys.argv[1:]
sep = argv.index("--")
names, uris = argv[:sep], argv[sep + 1:]
print(json.dumps({
    "connectionId": "regdemoJobs",
    "type": "generic",
    "description": "published pipeline job sources, content-addressed",
    "properties": dict(zip(names, uris)),
}))
PY
)
code=$(curl -s -o /dev/null -w '%{http_code}' -X POST "$KIOK/api/v1/connections" \
        -H "Authorization: Bearer $KTOK" -H 'Content-Type: application/json' -d "$BODY")
case "$code" in 2*) echo "  kiok connection regdemoJobs updated";;
  *) echo "  connection update failed HTTP $code";; esac
```


!!! danger "Why the key is content-addressed"
    Ontul's dependency fetcher caches on the worker assuming that the same key
    means the same bytes. Overwrite a fixed path and the fixed script **never
    runs again** — with no error at all, the old code keeps running.

---

## 4. The distributed jobs

### discover — walk S3, fill the queue

**`demo/pipeline/jobs/discover_job.py`**

```python
"""An Ontul PYTHON job that walks S3 and fills the ingest queue.

The kiok DAG points at this script's s3:// URI with ``ontul.jobType: PYTHON``, and
an Ontul worker downloads and runs it. Arguments arrive in argv as ``key=value``:

    prefix=corpus/  bucket=iceberg-warehouse  run_id=...

This is the only stage in the pipeline that deals with a listing. Reading, parsing
and splitting all move into SQL, which the workers distribute. At a scale where
even the listing does not fit on one machine, what divides is the prefix rather
than the files — which is why prefix is an argument.
"""
import os
import sys
from datetime import datetime, timezone

FORMATS = {".pdf": "pdf", ".docx": "docx", ".xlsx": "xlsx", ".hwpx": "hwpx"}


def args():
    return dict(a.split("=", 1) for a in sys.argv[1:] if "=" in a)


def main():
    p = args()
    bucket = p.get("bucket", "iceberg-warehouse")
    prefix = p.get("prefix", "")
    run_id = p.get("run_id", "discover")
    batch = int(p.get("batch", "500"))

    import boto3
    from botocore.config import Config
    from ontul.session import OntulSession

    s3 = boto3.client(
        "s3",
        endpoint_url=p.get("s3_endpoint", os.environ.get("REGDEMO_S3_ENDPOINT", "http://api-server-1:8080")),
        aws_access_key_id=p.get("s3_access_key", os.environ.get("REGDEMO_S3_ACCESS_KEY", "")),
        aws_secret_access_key=p.get("s3_secret_key", os.environ.get("REGDEMO_S3_SECRET_KEY", "")),
        region_name=p.get("s3_region", "us-east-1"),
        config=Config(s3={"addressing_style": "path"}))

    # The job runs inside a worker, so the master is reachable by name.
    session = OntulSession(host=p.get("ontul_host", "ontul-master-1"),
                           port=int(p.get("ontul_port", "47470")))

    # Objects already in the queue are left alone. The queue's identifier is s3_uri,
    # so re-inserting one upserts it, resets status to PENDING, and the next run
    # chunks the same document again — doubling the chunks and colliding on the
    # primary key. For re-running the pipeline to be safe, this stage has to look
    # only at what newly arrived.
    seen = set()
    try:
        rows = session.source("SELECT s3_uri FROM ice.reg.ingest_queue").to_pylist()
        seen = {r["s3_uri"] for r in rows if r.get("s3_uri")}
    except Exception as e:                                   # noqa: BLE001
        # If the queue does not exist yet, everything is new. Any other failure would
        # create duplicates if passed over silently, so it is at least reported.
        print(f"queue not readable yet ({type(e).__name__}) — treating everything as new")

    now = datetime.now(timezone.utc).strftime("%Y-%m-%d %H:%M:%S")
    pending, total, skipped, already = [], 0, 0, 0

    for page in s3.get_paginator("list_objects_v2").paginate(Bucket=bucket, Prefix=prefix):
        for obj in page.get("Contents", []):
            key = obj["Key"]
            ext = key[key.rfind("."):].lower() if "." in key else ""
            fmt = FORMATS.get(ext)
            if fmt is None:
                skipped += 1
                continue
            uri = f"s3://{bucket}/{key}"
            if uri in seen:
                already += 1
                continue
            total += 1
            pending.append((uri, bucket, key, obj["Size"], obj["ETag"].strip('"'), fmt))
            if len(pending) >= batch:
                insert(session, pending, now, run_id)
                pending = []
    if pending:
        insert(session, pending, now, run_id)

    print(f"discovered {total} new file(s) under s3://{bucket}/{prefix} "
          f"({already} already queued, {skipped} skipped: unsupported format), "
          f"run_id={run_id}")


def insert(session, rows, now, run_id):
    def lit(v):
        return "NULL" if v is None else "'" + str(v).replace("'", "''") + "'"
    values = ", ".join(
        "(" + ", ".join([lit(r[0]), lit(r[1]), lit(r[2]), str(r[3]), lit(r[4]), lit(r[5]),
                         f"TIMESTAMP '{now}'", "'PENDING'", "0", "NULL", "NULL", lit(run_id)]) + ")"
        for r in rows)
    res = session.execute(
        "INSERT INTO ice.reg.ingest_queue "
        "(s3_uri, bucket, object_key, size_bytes, etag, file_format, discovered_at, "
        " status, attempt, error, processed_at, run_id) VALUES " + values)
    # execute() reports failure as a status rather than an exception. Unchecked, the
    # queue stays empty and the next stage succeeds with "nothing to process".
    if res.get("status") != "ok":
        raise SystemExit(f"queue insert failed: {res.get('message')}")


if __name__ == "__main__":
    main()
```


### Distributed chunking (the Python form of the SQL path)

**`demo/pipeline/src/regdemo_pipeline/jobs/chunk_distributed.py`**

```python
"""The stage that runs extraction and chunking on the workers.

The earlier implementation had the driver open 442 files, build the chunks and
INSERT them. Here the driver sends two SQL statements and is done — opening the
files, splitting them and writing the results all happen on the workers.

    python -m regdemo_pipeline.jobs.chunk_distributed --run-id <id>

How it works:

1. ``extract_chunks`` is registered as a Python UDF. The function is serialised
   with cloudpickle and travels to the workers with the query plan. The module is
   serialised by value on purpose: the default, by reference, would require the
   same module to be installed on the worker — which means rebuilding the image
   every time the code changes.

2. ``UNNEST`` expands one document into many chunks and inserts them into
   doc_chunks. Doing the 1→N inside SQL is the point: coming back to the driver
   and going out again makes that round trip the whole cost as the corpus grows.

3. The queue's status is updated. A failed document is left as FAILED with a
   reason — the numbers in the next stage are only trustworthy if no file was
   skipped quietly.
"""
from __future__ import annotations

import argparse
import os
import sys
import uuid
from pathlib import Path


def main(argv=None) -> int:
    ap = argparse.ArgumentParser(prog="chunk_distributed")
    ap.add_argument("--out", default="./out")
    ap.add_argument("--ontul", default=None)
    ap.add_argument("--password", default=os.environ.get("ONTUL_ADMIN_PASSWORD", "regdemo-admin-2026"))
    ap.add_argument("--run-id", default=None)
    ap.add_argument("--limit", type=int, default=0,
                    help="process at most N queued files (0 = all); a way to try the "
                         "pipeline on a slice before committing the whole corpus")
    a = ap.parse_args(argv)

    from .ingest import read_stack_env, Ontul
    env = read_stack_env(Path(a.out) / "stack.env")
    run_id = a.run_id or f"chunk-{uuid.uuid4().hex[:8]}"
    url = a.ontul or env.get("ONTUL_URL", "http://localhost:8080")
    ontul = Ontul(url, "admin", a.password)

    pending = ontul.count("ice.reg.ingest_queue")
    if pending == 0:
        raise SystemExit("ingest_queue is empty — run discover first")

    # The UDF is scoped to the Flight session that registered it, so the INSERT
    # that calls it has to travel the same connection. Sending it over the admin
    # HTTP endpoint instead would reach a master that has never heard of the
    # function.
    session = _register_udf(env, url, ontul.token)

    # The queue is named directly rather than wrapped in a subquery: a derived
    # table's alias is not visible inside UNNEST's argument, so q.s3_uri there
    # resolves to nothing and the statement fails with "Table 'q' not found".
    where = "q.status = 'PENDING'"
    if a.limit > 0:
        # A slice to try the pipeline on. Chosen here rather than with LIMIT so the
        # restriction is a predicate on the base table, which UNNEST can correlate to.
        body = ontul.sql(f"SELECT s3_uri FROM ice.reg.ingest_queue "
                         f"WHERE status = 'PENDING' ORDER BY s3_uri LIMIT {a.limit}")
        uris = [r[0] for r in (body.get("rows") or [])]
        if not uris:
            raise SystemExit("nothing PENDING in the queue")
        quoted = ", ".join("'" + u.replace("'", "''") + "'" for u in uris)
        where += f" AND q.s3_uri IN ({quoted})"

    before = ontul.count("ice.reg.doc_chunks")
    # One statement. The driver holds nothing: each worker reads the objects its
    # own split points at, and writes the chunks it produced.
    insert_sql = f"""
        INSERT INTO ice.reg.doc_chunks
          (chunk_id, doc_no, version, ordinal_no, article_no, article_title,
           page_from, page_to, text, token_count)
        SELECT
          q.s3_uri || '#' || jfield(c, 'ordinal_no'),
          NULL, NULL,
          CAST(jfield(c, 'ordinal_no') AS INTEGER),
          jfield(c, 'article_no'),
          jfield(c, 'article_title'),
          CAST(jfield(c, 'page_from') AS INTEGER),
          CAST(jfield(c, 'page_to') AS INTEGER),
          jfield(c, 'text'),
          CAST(jfield(c, 'token_count') AS INTEGER)
        FROM ice.reg.ingest_queue q, UNNEST(extract_chunks(q.s3_uri)) AS t(c)
        WHERE {where}
    """
    res = session.execute(insert_sql)
    if res.get("status") != "ok":
        raise SystemExit(f"chunking failed: {res.get('message')}")
    after = ontul.count("ice.reg.doc_chunks")
    print(f"chunks: {before} -> {after}  (+{after - before})")

    # The queue is the record of what was done, so it is updated from what landed
    # rather than from what was attempted.
    ontul.sql(f"UPDATE ice.reg.ingest_queue SET status = 'EXTRACTED', run_id = '{run_id}' "
              f"WHERE {where.replace('q.', '')}")
    print(f"run_id={run_id}")
    return 0


def _register_udf(env: dict, url: str, token: str):
    """Ship extract_chunks to the workers, and return the session it lives in."""
    import cloudpickle
    from ontul.session import OntulSession                 # ontul-python-sdk
    from ..udf import extract_chunks as mod

    # By value, not by reference: the default would require this module to be
    # installed in the worker image, so every edit to the extractor would mean
    # rebuilding it. The S3 credentials the UDF needs ride along in the closure
    # for the same reason — the worker has no stack.env.
    cloudpickle.register_pickle_by_value(mod)

    s3cfg = {
        "endpoint": env["S3_ENDPOINT_INTERNAL"],
        "access_key": env["S3_ACCESS_KEY"],
        "secret_key": env["S3_SECRET_KEY"],
        "region": env.get("S3_REGION", "us-east-1"),
    }

    def extract_chunks(s3_uri):
        os.environ.setdefault("REGDEMO_S3_ENDPOINT", s3cfg["endpoint"])
        os.environ.setdefault("REGDEMO_S3_ACCESS_KEY", s3cfg["access_key"])
        os.environ.setdefault("REGDEMO_S3_SECRET_KEY", s3cfg["secret_key"])
        os.environ.setdefault("REGDEMO_S3_REGION", s3cfg["region"])
        return mod.extract_chunks(s3_uri)

    host = url.split("//", 1)[-1].split(":")[0]
    session = OntulSession(host=host, token=token)
    # The chunk travels as JSON because the engine does not carry the element type
    # of an array of structs, and it is read back with a UDF because JSON_VALUE is
    # not implemented either. One function rather than a schema change: a new field
    # in a chunk needs no DDL, only a new jfield() in the SELECT.
    def jfield(doc, key):
        import json as _json
        if not doc:
            return None
        try:
            v = _json.loads(doc).get(key)
        except Exception:                                    # noqa: BLE001
            return None
        return None if v is None else str(v)

    session.register_udf("extract_chunks", extract_chunks,
                         return_type="list", param_types=["string"],
                         element_type="string")
    session.register_udf("jfield", jfield,
                         return_type="string", param_types=["string", "string"])
    print("registered extract_chunks (list<string>) and jfield (string)")
    return session


if __name__ == "__main__":
    sys.exit(main())
```


### Settling the effective dates — a federated join

**`demo/jobs/20_effective_dates.sql`**

```sql
-- ============================================================================
-- Effective dates — decided by the approval, not by the document
--
-- The "in force from 1 January 2025" printed in a document's 부칙 was written when
-- the draft was. If approval slips, that sentence stays where it is and stops
-- being true. HR-REG-003 v3 is exactly that case: the document says 2025-01-01 and
-- approval completed 2025-03-15. For those ten weeks, citing the document means
-- answering with a regulation that had not taken effect.
--
-- So both dates are kept and the authority is the approval. The disagreement is
-- not erased but flagged as date_mismatch, leaving a queue for HR to review.
--
-- That this join is federated is the point. Copy the approval history into the
-- lake and approvals after the copy are simply absent — a fact that surfaces only
-- as a quietly wrong answer.
-- ============================================================================

-- ── 1. Completed approvals → effective date and status ─────────────────────
-- The mismatch is computed in the USING query and passed through as a column.
-- MERGE's SET takes only literals and column references, so the expression cannot
-- go there — the place to compute it is the source side. There is a second reason
-- to precompute it: leave every query to compare the two dates and one of them
-- will forget.
MERGE INTO ice.reg.doc_versions v
USING (
    SELECT doc_no,
           ver              AS version,
           apr_id,
           complete_dt      AS approved_on,
           stated_dt        AS stated_on,
           -- The planner rewrites CASE ... THEN TRUE into an IS TRUE call, which the
           -- execution engine does not have. Left as a comparison, no such rewrite
           -- happens. With stated_dt NULL the result is NULL — "unknown" rather than
           -- "different", which is the correct value.
           (complete_dt <> stated_dt) AS mismatch
    FROM gw.groupware.gw_approval
    WHERE sts_cd = 'CMPL'
) a
ON v.doc_no = a.doc_no AND v.version = a.version
WHEN MATCHED THEN UPDATE SET
    effective_from = a.approved_on,
    stated_from    = a.stated_on,
    date_mismatch  = a.mismatch,
    approval_id    = a.apr_id,
    status         = 'EFFECTIVE';

-- ── 2. Approvals still in flight → not yet in force ────────────────────────
-- A revision that was submitted but not completed must not be answered as if it
-- were current. The fact that a draft exists is kept; the effective date is left
-- empty.
MERGE INTO ice.reg.doc_versions v
USING (
    SELECT doc_no, ver AS version, apr_id
    FROM gw.groupware.gw_approval
    WHERE sts_cd <> 'CMPL'
) p
ON v.doc_no = p.doc_no AND v.version = p.version
WHEN MATCHED THEN UPDATE SET
    effective_from = NULL,
    approval_id    = p.apr_id,
    status         = 'DRAFT';

-- ── 3. The end date is not computed here ───────────────────────────────────
-- "The next version" is nxt.version > cur.version — a self-join on an inequality.
-- The execution engine handles one equality conjunct per join, so the moment the
-- planner pushes this predicate into the join condition it is refused. Window
-- functions (LEAD) are unsupported too, and a composite key built by concatenating
-- version numbers is only correct while they are contiguous, which they are not:
-- 121 approvals produced 102 versions that have a file. A gap leaves effective_to
-- NULL, and NULL means "current" — answering with a superseded regulation as
-- though it were current is precisely the failure this demo exists to prevent, so
-- this is not a place to be clever.
--
-- Instead close_versions sorts those 102 rows and closes them. The place where
-- distribution earns its keep is 10_index_vectors.sql, embedding thousands of
-- chunks.
```


Once the approval dates land, the previous versions are closed.

**`demo/pipeline/jobs/close_versions_job.py`**

```python
"""An Ontul PYTHON job that closes each version — effective_to becomes the next
version's effective date.

The kiok DAG points at this script's s3:// URI with ``ontul.jobType: PYTHON`` and
an Ontul worker downloads and runs it.

Not doing this in SQL on the cluster is an engine constraint rather than a
preference. "The next version" is ``nxt.version > cur.version`` — a self-join on
an inequality — and a SELECT-level join executes a single equality conjunct. The
escapes are closed too: window functions (LEAD) are unsupported, and a composite
key built by concatenating version numbers is only correct while they are
contiguous, which they are not. 121 approvals produced 102 versions that have a
file. A gap leaves effective_to NULL, and NULL means "current" — so a superseded
regulation would answer as the current one, which is the exact failure this demo
exists to prevent.

What is left is sorting 102 rows per document. Claiming that needs a cluster would
be the dishonest part; the place distribution earns its keep is the vectors task,
embedding thousands of chunks.
"""
import sys
from collections import defaultdict
from datetime import date, timedelta

EPOCH = date(1970, 1, 1)


def args():
    return dict(a.split("=", 1) for a in sys.argv[1:] if "=" in a)


def as_date(value):
    """Normalise whatever the engine returns for a DATE column.

    CAST(d AS VARCHAR) on a DATE returns the stored value — days since the epoch —
    rather than a formatted date, so '19112' arrives where '2022-05-15' was
    expected and the literal built from it is rejected. Both forms are accepted
    rather than depending on which one a given path produces.
    """
    if value is None:
        return None
    if isinstance(value, int):
        return (EPOCH + timedelta(days=value)).isoformat()
    text = str(value).strip()
    if text.isdigit():
        return (EPOCH + timedelta(days=int(text))).isoformat()
    return text[:10] or None


def main():
    p = args()
    from ontul.session import OntulSession

    s = OntulSession(host=p.get("ontul_host", "ontul-master-1"),
                     port=int(p.get("ontul_port", "47470")))

    rows = s.source("SELECT doc_no, version, effective_from FROM ice.reg.doc_versions "
                    "WHERE effective_from IS NOT NULL").to_pylist()
    if not rows:
        # Succeeding quietly would leave every version's effective_to at NULL, and
        # NULL means "current" — every superseded version would answer as current.
        raise SystemExit("no version has an effective date — effective_dates has to run first")

    by_doc = defaultdict(list)
    for r in rows:
        d = as_date(r.get("effective_from"))
        if d:
            by_doc[r["doc_no"]].append((int(r["version"]), d))

    updates = []
    for doc_no, vs in by_doc.items():
        # Sorted by effective date, not by version number. A higher-numbered version
        # taking effect first does happen (rejected, then resubmitted), and sorting by
        # number then closes an earlier version at a date that has not arrived.
        vs.sort(key=lambda x: (x[1], x[0]))
        for (ver, _), (_, nxt_from) in zip(vs, vs[1:]):
            updates.append((doc_no, ver, nxt_from))

    if not updates:
        print("closed 0 versions (every document has only one)")
        return

    def lit(v):
        return "NULL" if v is None else "'" + str(v).replace("'", "''") + "'"

    values = ", ".join(
        f"({lit(d)}, {v}, DATE {lit(t)})" for d, v, t in updates)
    res = s.execute(
        "MERGE INTO ice.reg.doc_versions v USING ("
        f"  SELECT * FROM (VALUES {values}) AS t(doc_no, version, closes_on)"
        ") c ON v.doc_no = c.doc_no AND v.version = c.version "
        "WHEN MATCHED THEN UPDATE SET effective_to = c.closes_on, status = 'SUPERSEDED'")
    if res.get("status") != "ok":
        raise SystemExit(f"close failed: {res.get('message')}")

    print(f"closed {len(updates)} superseded version(s) across {len(by_doc)} document(s)")


if __name__ == "__main__":
    main()
```


The local equivalent, kept for the single-node path.

**`demo/pipeline/src/regdemo_pipeline/jobs/close_versions.py`**

```python
"""
Close each regulation version: set effective_to to when the next one starts.

This is the one step that does not run in the cluster, and the reason is a
specific engine limit rather than a preference. "The next version" is
`nxt.version > cur.version` — a self-join on an inequality — and a SELECT-level
join executes a single equality conjunct. The usual escapes are closed too:
window functions (LEAD) are unsupported, and a concatenated composite key would
only be correct if version numbers were contiguous, which they are not. 121
approved versions produced 102 rows, because some versions have no file. A gap
would leave effective_to NULL, and a NULL effective_to means "still current" —
so a superseded regulation would answer as the current one, which is the exact
failure this demo exists to show being prevented. Not a place to be clever.

The write-back is a MERGE, keyed on (doc_no, version) with the computed dates
supplied as literals. What is left for this process is 102 rows to sort per
document; claiming that needs a cluster would be the dishonest part. The
embedding job is where distribution earns its keep, over 6,000 chunks.

    python -m regdemo_pipeline.jobs.close_versions --out ./out
"""
from __future__ import annotations

import argparse
import sys
from collections import defaultdict
from datetime import date, timedelta
from pathlib import Path

EPOCH = date(1970, 1, 1)


def as_date(value) -> str | None:
    """Normalise whatever the engine hands back for a DATE column.

    CAST(d AS VARCHAR) on a DATE returns the stored value — days since the epoch —
    rather than a formatted date, so '19112' arrives where '2022-05-15' was
    expected and the literal built from it is rejected. Accept both forms rather
    than depending on which one a given path produces.
    """
    if value is None:
        return None
    if isinstance(value, int):
        return (EPOCH + timedelta(days=value)).isoformat()
    text = str(value).strip()
    if text.isdigit():
        return (EPOCH + timedelta(days=int(text))).isoformat()
    return text[:10] or None

from .ingest import Ontul, Raw, read_stack_env, row


def main(argv=None) -> int:
    ap = argparse.ArgumentParser(description=__doc__)
    ap.add_argument("--out", default="./out", type=Path)
    ap.add_argument("--ontul", default=None)
    ap.add_argument("--user", default="admin")
    ap.add_argument("--password", default="regdemo-admin-2026")
    a = ap.parse_args(argv)

    env = read_stack_env(a.out / "stack.env")
    o = Ontul(a.ontul or env.get("ONTUL_URL", "http://localhost:8080"), a.user, a.password)

    body = o.sql(
        "SELECT doc_no, version, effective_from "
        "FROM ice.reg.doc_versions WHERE effective_from IS NOT NULL"
    )
    rows = body.get("rows") or []
    if not rows:
        print("no versions carry an effective_from — has 20_effective_dates.sql run?")
        return 1

    by_doc: dict[str, list[tuple[int, str]]] = defaultdict(list)
    for doc_no, version, eff in rows:
        d = as_date(eff)
        if d:
            by_doc[doc_no].append((int(version), d))

    # effective_to is the next version's start, not the day before it. The
    # retriever compares [from, to), so the boundary belongs to exactly one
    # version; subtracting a day is how two versions end up both valid, or
    # neither, on the day of the change.
    closes: list[str] = []
    for doc_no, versions in by_doc.items():
        versions.sort()
        for i, (version, _) in enumerate(versions):
            if i + 1 < len(versions):
                closes.append(row(doc_no, version, Raw(f"DATE '{versions[i + 1][1]}'"), "SUPERSEDED"))

    if not closes:
        print("every document has a single effective version — nothing to close")
        return 0

    # One MERGE, source supplied as literals. The target keeps its other columns.
    for i in range(0, len(closes), 100):
        chunk = closes[i:i + 100]
        o.sql(
            "MERGE INTO ice.reg.doc_versions v "
            "USING (VALUES " + ",".join(chunk) + ") w (doc_no, version, ends_on, new_status) "
            "ON v.doc_no = w.doc_no AND v.version = w.version "
            "WHEN MATCHED THEN UPDATE SET effective_to = w.ends_on, status = w.new_status"
        )

    superseded = o.sql("SELECT count(*) FROM ice.reg.doc_versions WHERE effective_to IS NOT NULL")
    n = (superseded.get("rows") or [[0]])[0][0]
    print(f"closed {len(closes)} versions; {n} rows now carry an effective_to")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```


### Clearing the vector generation

**`demo/pipeline/jobs/clear_vectors_job.py`**

```python
"""An Ontul PYTHON job that empties the vector table.

This is the one step that does not go through Ontul. NeorunBase is not a JDBC
catalog, so ``DELETE FROM nb.public.doc_vectors_gen1`` is refused with "Not a JDBC
catalog: nb". The rows are removed over NeorunBase's own PostgreSQL wire instead.

The clearing step exists because indexing is a full pass. Incremental would be the
natural thing, but knowing which chunks are already embedded means reading the
vector table, and reading a table with a VECTOR column is still broken (see
tests/known-issues). Clearing and rebuilding never leaves the ambiguous state of
"partly indexed".

    host=... port=5434 database=neorunbase user=admin password=...
"""
import sys


def args():
    return dict(a.split("=", 1) for a in sys.argv[1:] if "=" in a)


def main():
    p = args()
    import pg8000.native

    conn = pg8000.native.Connection(
        user=p.get("user", "admin"),
        password=p.get("password", ""),
        host=p.get("host", "neorun-coordinator-1"),
        port=int(p.get("port", "5434")),
        database=p.get("database", "neorunbase"),
    )
    try:
        for table in ("doc_vectors_gen1",):
            conn.run(f"DELETE FROM {table}")
            n = conn.run(f"SELECT count(*) FROM {table}")[0][0]
            # A response saying it was cleared and the table actually being empty are
            # different claims. Anything left behind means the next stage indexes on
            # top of duplicates.
            if n != 0:
                raise SystemExit(f"{table} still holds {n} row(s) after DELETE")
            print(f"cleared {table}")
    finally:
        conn.close()


if __name__ == "__main__":
    main()
```


!!! note "The one step that does not go through Ontul"
    NeorunBase is not a JDBC catalog, so a `DELETE` through Ontul is refused
    (`Not a JDBC catalog: nb`). The vector table is emptied over NeorunBase's own
    protocol.

### Embedding — as BATCH SQL

Not as a PYTHON job. Ontul dispatches PYTHON jobs to a **single** worker, so
writing this in Python would parallelise nothing while looking like it should.
BATCH SQL spreads with the scan and evaluates `embed_passage()` per Arrow batch —
where the data already is.

**`demo/jobs/10_index_vectors.sql`**

```sql
-- ============================================================================
-- Indexing — distributed by the scan, embedded by the connection
--
-- Not a Python job. An Ontul PYTHON job is dispatched to a single worker
-- (JobManager picks one by hash), so it parallelises nothing. A BATCH SQL job
-- does: the scan spreads across workers and embed_passage() evaluates per Arrow
-- batch where the data already is.
--
-- Which model produced these vectors is not stated here. It belongs to the
-- 'emb_main' connection, and the retriever names the same connection — so the
-- index and the queries against it cannot come from different weights.
-- ============================================================================

-- A full re-index. "Only the chunks not yet embedded" would be the natural thing,
-- but it requires Ontul to read the vector table and that read is broken: a table
-- with a VECTOR column cannot be read over JDBC (see tests/known-issues). So
-- re-run safety comes from clearing the table before indexing instead.
--
-- 799 chunks take 87 seconds, so a full pass is simpler than an incremental one
-- and — more importantly — it never leaves the ambiguous state of "partly
-- indexed".
INSERT INTO nb.public.doc_vectors_gen1
    (chunk_pk, chunk_id, doc_no, version, article_no, effective_from, effective_to,
     is_official, sensitivity, owner_dept, body, embedding)
SELECT
    c.chunk_pk,
    c.chunk_id,
    c.doc_no,
    c.version,
    c.article_no,
    v.effective_from,
    v.effective_to,
    d.is_official,
    d.sensitivity,
    d.owner_dept,
    c.text,
    embed_passage('emb_main', c.text)
FROM ice.reg.doc_chunks c
-- Only one equality conjunct executes per join, so the (doc_no, version) pair is
-- concatenated into a single key. This is exact pair equality, not an
-- approximation — it does not depend on version numbers being contiguous.
JOIN ice.reg.doc_versions v
  ON c.doc_no || '#' || CAST(c.version AS VARCHAR)
   = v.doc_no || '#' || CAST(v.version AS VARCHAR)
JOIN ice.reg.documents d ON d.doc_no = c.doc_no;

-- Register what was built.
--
-- The identity below is substituted by infra/index.sh from out/stack.env, which
-- read it from the live endpoint's /fingerprint at bring-up. That is the only
-- honest source: the connection declares an identity, but what is recorded here
-- should be what actually answered, and verify=strict is what makes the two the
-- same thing rather than a hope.
--
-- Without this row a re-indexed table is indistinguishable from one built by a
-- different model, and the failure mode is not an error — it is slightly worse
-- retrieval that nobody can explain.
MERGE INTO ice.reg.embedding_generations g
-- The chunk count is computed in the USING query and passed through as a column.
-- SET accepts only literals and column references, so a subquery cannot sit there
-- — the place to count is the source side.
USING (
    SELECT '${EMBED_GENERATION}' AS generation_id,
           count(*)              AS n
    FROM nb.public.doc_vectors_gen1
) s
ON g.generation_id = s.generation_id
WHEN MATCHED THEN UPDATE SET
    fingerprint    = '${EMBED_FINGERPRINT}',
    model_id       = '${EMBED_MODEL_ID}',
    model_revision = '${EMBED_MODEL_REVISION}',
    dim            = ${EMBED_DIM},
    normalize      = TRUE,
    distance       = 'cosine',
    target_table   = 'nb.public.doc_vectors_gen1',
    chunk_count    = s.n,
    status         = 'ACTIVE'
WHEN NOT MATCHED THEN INSERT
    (generation_id, fingerprint, model_id, model_revision, dim, normalize, distance,
     target_table, status, chunk_count)
    VALUES (s.generation_id, '${EMBED_FINGERPRINT}', '${EMBED_MODEL_ID}',
            '${EMBED_MODEL_REVISION}', ${EMBED_DIM}, TRUE, 'cosine',
            'nb.public.doc_vectors_gen1', 'ACTIVE', s.n);
```


### The graph — authority relations

**`demo/pipeline/jobs/build_graph_job.py`**

```python
"""An Ontul PYTHON job that builds the relation graph.

Edges are read from the clauses in which one document cites another as its basis.
Those citations are in the body text, so this is only possible once chunking has
finished — which is why it comes after `chunk` in the DAG.

    python build_graph_job.py   (run by an Ontul worker)

Arguments:
    ontul_host / ontul_port   where the master is (default ontul-master-1:47470)

Edges are stored in both directions for one reason. GRAPH_NEIGHBORS expands from
src → dst only, while "what is this based on" and "what depends on this" are the
same edge read the other way round. Since only one direction can be walked, the
other is stored — cheaper than writing a second traversal engine.
"""
import re
import sys
from collections import defaultdict

CITATION = re.compile(r"[「『]\s*([A-Z]{2,4}-[A-Z]{3}-\d{3})\s*[」』]")


def args():
    return dict(a.split("=", 1) for a in sys.argv[1:] if "=" in a)


def lit(v):
    if v is None:
        return "NULL"
    if isinstance(v, bool):
        return "TRUE" if v else "FALSE"
    if isinstance(v, (int, float)):
        return str(v)
    return "'" + str(v).replace("'", "''") + "'"


def main():
    p = args()
    from ontul.session import OntulSession
    s = OntulSession(host=p.get("ontul_host", "ontul-master-1"),
                     port=int(p.get("ontul_port", "47470")))

    docs = s.source("SELECT doc_id, doc_no, title, tier, owner_dept, is_official, sensitivity "
                    "FROM ice.reg.documents").to_pylist()
    if not docs:
        raise SystemExit("no documents in the ledger — run the ingest first")

    # Traversal starts from a number; doc_no stays the name people use.
    # The vertex id is whatever the ledger holds. Renumbering here would make the
    # same regulation a different vertex on every run, and then nothing that points
    # at the graph — a stored traversal result, an ontology call — could be read
    # tomorrow with yesterday's meaning.
    doc_id = {d["doc_no"]: int(d["doc_id"]) for d in docs if d.get("doc_id") is not None}
    known = set(doc_id)

    chunks = s.source("SELECT doc_no, version, article_no, text FROM ice.reg.doc_chunks").to_pylist()

    edges = {}
    for c in chunks:
        src = c.get("doc_no")
        if src not in known:
            continue
        for target in CITATION.findall(c.get("text") or ""):
            if target == src or target not in known:
                # Citing an unregistered document is a finding in itself, but not an
                # edge — there is nowhere for it to lead.
                continue
            key = (src, target, "CHILD_OF")
            edges.setdefault(key, (doc_id[src], doc_id[target], "CHILD_OF", src, target,
                                   c.get("version"), None, c.get("article_no"),
                                   "BODY_REGEX", 0.95))

    for (src, dst, rel), e in list(edges.items()):
        if rel == "CHILD_OF":
            edges[(dst, src, "DEPENDED_ON_BY")] = (
                e[1], e[0], "DEPENDED_ON_BY", dst, src, e[6], e[5], e[7], e[8], e[9])

    versions = s.source("SELECT doc_no, version FROM ice.reg.doc_versions").to_pylist()
    by_doc = defaultdict(list)
    for v in versions:
        if v["doc_no"] in known:
            by_doc[v["doc_no"]].append(int(v["version"]))
    for doc_no, vs in by_doc.items():
        vs.sort()
        for older, newer in zip(vs, vs[1:]):
            edges[(doc_no, doc_no, f"SUPERSEDES:{newer}")] = (
                doc_id[doc_no], doc_id[doc_no], "SUPERSEDES", doc_no, doc_no,
                newer, older, None, "APPROVAL", 1.0)

    # The relations are written to the lake, not to NeorunBase.
    #
    # This job used to delete from and insert into nb.public.doc_edges directly.
    # That is bad because it makes the serving layer the system of record: rebuild
    # NeorunBase and the relations are gone, when and how they changed is recorded
    # nowhere, and the only way to fix the graph is "run the job again".
    #
    # Now the job writes into Iceberg and a Flow
    # (schema/flows/graph_serving.json) keeps NeorunBase in step with it. The graph
    # the ontology's GRAPH link traverses is the one that Flow filled.
    #
    # The graph is a derivative, so it is rebuilt whole. A partial update leaves
    # behind citations that no longer exist.
    s.execute("DELETE FROM ice.reg.graph_edges")
    s.execute("DELETE FROM ice.reg.graph_nodes")

    nodes = ", ".join(
        "(" + ", ".join(lit(x) for x in (
            doc_id[d["doc_no"]], d["doc_no"], d["title"],
            int(d["tier"]) if d.get("tier") is not None else 2,
            d.get("owner_dept"), bool(d.get("is_official")), d.get("sensitivity"))) + ")"
        for d in docs)
    res = s.execute("INSERT INTO ice.reg.graph_nodes "
                    "(doc_id, doc_no, title, tier, owner_dept, is_official, sensitivity) "
                    "VALUES " + nodes)
    if res.get("status") != "ok":
        raise SystemExit(f"node insert failed: {res.get('message')}")

    edge_vals = ", ".join(
        "(" + ", ".join(lit(x) for x in (i + 1,) + e) + ")"
        for i, e in enumerate(edges.values()))
    res = s.execute("INSERT INTO ice.reg.graph_edges "
                    "(edge_pk, src_id, dst_id, rel_type, src_doc_no, dst_doc_no, "
                    " src_version, dst_version, src_article, source, confidence) "
                    "VALUES " + edge_vals)
    if res.get("status") != "ok":
        raise SystemExit(f"edge insert failed: {res.get('message')}")

    child = sum(1 for e in edges.values() if e[2] == "CHILD_OF")
    rev = sum(1 for e in edges.values() if e[2] == "DEPENDED_ON_BY")
    sup = sum(1 for e in edges.values() if e[2] == "SUPERSEDES")
    print(f"graph: {len(docs)} nodes, {len(edges)} edges "
          f"({child} CHILD_OF from authority clauses, {rev} reversed, {sup} SUPERSEDES)")
    if child == 0:
        # Succeeding quietly would leave the traversal retrievers returning nothing,
        # which reads as "there is no authority for this".
        raise SystemExit("no authority edges were found — the traversal retrievers "
                         "would return nothing")


if __name__ == "__main__":
    main()
```


The same logic in local form.

**`demo/pipeline/src/regdemo_pipeline/jobs/build_graph.py`**

```python
"""
Build the relation graph the traversal retrievers walk.

Edges are parsed, not inferred. Every regulation states its own basis in a
authority clause — "this guideline is established on the basis of «HR-REG-002»
Article 12" — and that
sentence is the edge. Reading it out of the text is why doc_relations carries a
`source` column: an edge from the register is a different kind of claim from one
read out of a body, and a reviewer checking a chain of authority needs to know
which is which.

Two edge types come out of this:

  CHILD_OF     the authority clause: this document derives its authority from that one
  SUPERSEDES   consecutive versions of the same document

The graph lives in NeorunBase because that is where the traversal runs
(GRAPH_NEIGHBORS). The ledger stays in Iceberg; this is a derivative and is
rebuilt whole.

    python -m regdemo_pipeline.jobs.build_graph --out ./out
"""
from __future__ import annotations

import argparse
import re
import sys
from collections import defaultdict
from pathlib import Path

from .ingest import Ontul, read_stack_env, row

# «HR-REG-002» — the bracket form every authority clause uses. Matching the brackets
# rather than the bare pattern avoids picking up a document number that merely
# appears in prose.
CITATION = re.compile(r"[「『]\s*([A-Z]{2,4}-[A-Z]{3}-\d{3})\s*[」』]")


def main(argv=None) -> int:
    ap = argparse.ArgumentParser(description=__doc__)
    ap.add_argument("--out", default="./out", type=Path)
    ap.add_argument("--ontul", default=None)
    ap.add_argument("--user", default="admin")
    ap.add_argument("--password", default="regdemo-admin-2026")
    a = ap.parse_args(argv)

    env = read_stack_env(a.out / "stack.env")
    o = Ontul(a.ontul or env.get("ONTUL_URL", "http://localhost:8080"), a.user, a.password)

    docs = o.sql("SELECT doc_no, title, tier, owner_dept, is_official, sensitivity "
                 "FROM ice.reg.documents").get("rows") or []
    if not docs:
        print("no documents in the ledger — run the ingest first")
        return 1

    # Numeric ids, because the traversal seeds on a number. doc_no stays as the
    # name a person uses; this is the one the BFS follows.
    doc_id = {no: i + 1 for i, no in enumerate(sorted(d[0] for d in docs))}
    known = set(doc_id)

    chunks = o.sql("SELECT doc_no, version, article_no, text FROM ice.reg.doc_chunks").get("rows") or []

    # ── CHILD_OF, read out of the authority clauses ─────────────────────────
    edges: dict[tuple, tuple] = {}
    for doc_no, version, article_no, text in chunks:
        for target in CITATION.findall(text or ""):
            if target == doc_no or target not in known:
                # A citation to a document that is not in the register is a real
                # finding, but it is not an edge — there is nothing to traverse to.
                continue
            key = (doc_no, target, "CHILD_OF")
            # Keep the first article that states it; later mentions are repeats.
            edges.setdefault(key, (doc_id[doc_no], doc_id[target], "CHILD_OF",
                                   doc_no, target, version, None, article_no,
                                   "BODY_REGEX", 0.95))

    # ── The same authority edges, reversed ──────────────────────────────────
    # The traversal only ever expands src → dst. "What does this derive from"
    # and "what depends on this" are the same edges read in opposite directions,
    # and only one of those directions is walkable — so the other is stored.
    # Materialising the reverse is cheaper than the alternative, which is a
    # second traversal engine.
    for (src, dst, rel), e in list(edges.items()):
        if rel != "CHILD_OF":
            continue
        edges[(dst, src, "DEPENDED_ON_BY")] = (
            e[1], e[0], "DEPENDED_ON_BY", dst, src, e[6], e[5], e[7], e[8], e[9])

    # ── SUPERSEDES, from consecutive versions ───────────────────────────────
    versions = o.sql("SELECT doc_no, version FROM ice.reg.doc_versions").get("rows") or []
    by_doc: dict[str, list[int]] = defaultdict(list)
    for doc_no, v in versions:
        if doc_no in known:
            by_doc[doc_no].append(int(v))
    for doc_no, vs in by_doc.items():
        vs.sort()
        for older, newer in zip(vs, vs[1:]):
            edges[(doc_no, doc_no, f"SUPERSEDES:{newer}")] = (
                doc_id[doc_no], doc_id[doc_no], "SUPERSEDES",
                doc_no, doc_no, newer, older, None, "APPROVAL", 1.0)

    # The tables are cleared by infra/index.sh, over NeorunBase's own wire:
    # DELETE through Ontul handles Iceberg and JDBC catalogs and answers "Not a
    # JDBC catalog: nb" for this one. The graph is a derivative and is rebuilt
    # whole, so clearing belongs with the other reset steps rather than here.
    node_rows = [row(doc_id[d[0]], d[0], d[1],
                     int(d[2]) if d[2] is not None else 2, d[3], bool(d[4]), d[5])
                 for d in docs]
    o.insert("nb.public.doc_nodes",
             ["doc_id", "doc_no", "title", "tier", "owner_dept", "is_official", "sensitivity"],
             node_rows, exact=False)

    edge_rows = [row(i + 1, *e) for i, e in enumerate(edges.values())]
    o.insert("nb.public.doc_edges",
             ["edge_pk", "src_id", "dst_id", "rel_type", "src_doc_no", "dst_doc_no",
              "src_version", "dst_version", "src_article", "source", "confidence"],
             edge_rows, exact=False)

    child = sum(1 for e in edges.values() if e[2] == "CHILD_OF")
    reverse = sum(1 for e in edges.values() if e[2] == "DEPENDED_ON_BY")
    supersedes = sum(1 for e in edges.values() if e[2] == "SUPERSEDES")
    print(f"graph: {len(node_rows)} nodes, {len(edge_rows)} edges "
          f"({child} CHILD_OF from authority clauses, {reverse} reversed, {supersedes} SUPERSEDES)")
    if child == 0:
        print("  no authority edges were found — the traversal retrievers will return nothing")
        return 1
    return 0


if __name__ == "__main__":
    sys.exit(main())
```


The register helper.

**`demo/pipeline/src/regdemo_pipeline/relations/register.py`**

```python
"""Match a file to a document number.

The register is the master for identity, but it has drifted from the drive it
describes: titles edited, versions not kept up, rows for documents nobody
uploaded, files nobody registered. So matching is a ranked decision over three
independent signals rather than a lookup, and it reports what it could not
resolve instead of guessing.

Signal order is by trustworthiness, not convenience:
  1. the running header      — printed by whoever published the document
  2. the register            — maintained, but stale in known ways
  3. the filename            — the least reliable, used only to break ties
The last one is deliberately weak: a file marked "final" with a date in its name
   names a
superseded version, and a pipeline that trusts filenames answers with it.
"""
from __future__ import annotations

import re
import unicodedata
from dataclasses import dataclass, field
from pathlib import Path

from openpyxl import load_workbook


@dataclass
class RegisterRow:
    doc_no: str
    title: str
    kind: str
    dept: str
    version: int | None
    effective_from: str
    sensitivity: str
    note: str


@dataclass
class Match:
    s3_key: str
    doc_no: str | None
    version: int | None
    confidence: float
    signals: list[str] = field(default_factory=list)
    reason: str | None = None       # why it could not be resolved


DOC_NO_RE = re.compile(r"[A-Z]{2,4}-(?:RUL|REG|GDL)-\d{3}")
VER_IN_NAME = re.compile(r"[_\s(]v(\d+)", re.IGNORECASE)
VER_KO = re.compile(r"제\s*(\d+)\s*차")
NOISE = re.compile(r"\[[^\]]*\]|\([^)]*\)|사본|최종|수정|본|_|\s|\.(pdf|docx|xlsx)$", re.I)


def load_register(path: Path) -> list[RegisterRow]:
    ws = load_workbook(path, read_only=True, data_only=True).active
    rows: list[RegisterRow] = []
    for r in ws.iter_rows(min_row=2, values_only=True):
        if not r or not r[0]:
            continue
        m = VER_KO.search(str(r[4] or ""))
        rows.append(RegisterRow(str(r[0]).strip(), str(r[1] or "").strip(),
                                str(r[2] or ""), str(r[3] or ""),
                                int(m.group(1)) if m else None,
                                str(r[5] or ""), str(r[6] or ""), str(r[7] or "")))
    return rows


def _norm(s: str) -> str:
    """Strip the decoration people add to filenames so two spellings of the same
    title compare equal — "final", "copy", bracketed dates, spacing, extension."""
    s = unicodedata.normalize("NFKC", s)
    return NOISE.sub("", s).lower()


def match(s3_key: str, header_doc_no: str | None, header_version: int | None,
          register: list[RegisterRow]) -> Match:
    by_no = {r.doc_no: r for r in register}
    name = Path(s3_key).name
    signals: list[str] = []

    # 1. The header. Printed at publication time, so when it is present it is
    #    the strongest evidence — and it survives a filename that says nothing.
    if header_doc_no:
        signals.append("header")
        row = by_no.get(header_doc_no)
        if row is None:
            # On the drive, absent from the register. Not an error: the register
            # is incomplete in exactly this way, and dropping the file would hide
            # a document that genuinely exists.
            return Match(s3_key, header_doc_no, header_version, 0.85,
                         signals, reason="not_in_register")
        return Match(s3_key, header_doc_no, header_version, 0.99, signals + ["register"])

    # 2. A document number written into the filename — the well-named minority.
    m = DOC_NO_RE.search(name)
    if m and m.group(0) in by_no:
        signals.append("filename_doc_no")
        v = VER_IN_NAME.search(name)
        return Match(s3_key, m.group(0), int(v.group(1)) if v else None, 0.90,
                     signals + ["register"])

    # 3. Title match against the register. The fallback for working copies,
    #    which carry no header.
    n = _norm(name)
    hits = [r for r in register if _norm(r.title) and _norm(r.title) in n]
    if len(hits) == 1:
        signals.append("title")
        v = VER_IN_NAME.search(name)
        version = int(v.group(1)) if v else hits[0].version
        # The register's version is known to lag, so a title-only match is not
        # authoritative about which version this file is.
        return Match(s3_key, hits[0].doc_no, version,
                     0.70 if v else 0.55, signals + ["register"])
    if len(hits) > 1:
        return Match(s3_key, None, None, 0.0, signals + ["title"],
                     reason=f"ambiguous_title:{len(hits)}")

    return Match(s3_key, None, None, 0.0, signals, reason="unmatched")


def unregistered_rows(register: list[RegisterRow], matched: set[str]) -> list[str]:
    """Register rows that no file ever matched — planned documents that were
    never produced. Reported so the register can be corrected, rather than
    silently treated as missing content."""
    return sorted(r.doc_no for r in register if r.doc_no not in matched)
```


---

## 5. Running it

**`demo/infra/pipeline.sh`**

```bash
#!/usr/bin/env bash
# Register the indexing pipeline as a kiok DAG and run it.
#
#   bash infra/pipeline.sh            # register, run, wait for completion
#   bash infra/pipeline.sh register   # register only
#   bash infra/pipeline.sh status     # status of the latest run
#
# Replaces index.sh. Same stages, same order; what changes is where that order is
# written down — a DAG that can be queried rather than the line order of a shell
# script, so a failed task can be re-run on its own and yesterday's run can be put
# beside today's.
set -uo pipefail
cd "$(dirname "$0")/.."
DEMO=$(pwd)
. out/stack.env 2>/dev/null || { echo "no out/stack.env — run infra/up.sh first"; exit 1; }

KIOK=${KIOK_URL:-http://localhost:18081}
ONTUL=${ONTUL_URL:-http://localhost:8080}
ONTUL_INTERNAL=${ONTUL_INTERNAL_URL:-http://ontul-master-1:8080}
ONTUL_PW="${ONTUL_ADMIN_PASSWORD:-regdemo-admin-2026}"
KIOK_INITIAL_PW=${KIOK_INITIAL_PW:-admin}
KIOK_PW=${KIOK_PASSWORD:-Regdemo-kiok-2026}
DAG_FILE="$DEMO/schema/dags/regdemo_index.yaml"
MODE=${1:-run}

log(){ printf '\n\033[1;36m== %s\033[0m\n' "$*"; }
step(){ printf '\033[1;32m  ok\033[0m %s\n' "$*"; }
fail(){ printf '\033[1;31m  failed\033[0m %s\n' "$*"; exit 1; }

# ── kiok login. The initial password works exactly once, so rotation is handled
#    here too. ────────────────────────────────────────────────────────────────
tok(){ curl -s -m 30 -X POST "$KIOK/api/v1/auth/login" -H 'Content-Type: application/json' \
        -d "{\"user\":\"admin\",\"password\":\"$1\"}" \
      | python3 -c "import sys,json;print(json.load(sys.stdin).get('accessToken',''))" 2>/dev/null; }
KTOK=$(tok "$KIOK_PW")
if [ -z "$KTOK" ]; then
  KTOK=$(tok "$KIOK_INITIAL_PW")
  [ -n "$KTOK" ] || fail "kiok login failed"
  curl -sf -m 30 -X POST "$KIOK/api/v1/auth/change-password" -H "Authorization: Bearer $KTOK" \
      -H 'Content-Type: application/json' \
      -d "{\"oldPassword\":\"$KIOK_INITIAL_PW\",\"newPassword\":\"$KIOK_PW\"}" >/dev/null
  KTOK=$(tok "$KIOK_PW")
  [ -n "$KTOK" ] || fail "login failed after rotating the password"
  step "rotated kiok's default password"
fi
KAH="Authorization: Bearer $KTOK"
step "kiok authenticated"

# ── Ontul credentials go into a kiok connection ─────────────────────────────
#
# A token written into the DAG stays in the stored DagSpec and in the admin UI's
# Source tab. kiok resolves ${conn.<id>.<key>} on the worker just before the task
# runs, so only the reference is in the DAG and the value lives solely in the
# KMS-encrypted store.
#
# And the value is a user token (OTOK…) issued alongside an access key, not a
# login JWT. A JWT expires in fifteen minutes, which breaks authentication on the
# next run of a scheduled DAG. An OTOK does not expire and is sent as
# `Authorization: Token`.
OJWT=$(curl -s -m 30 -X POST "$ONTUL/admin/auth/login" -H 'Content-Type: application/json' \
       -d "{\"username\":\"admin\",\"password\":\"$ONTUL_PW\"}" \
     | python3 -c "import sys,json;print(json.load(sys.stdin).get('accessToken',''))" 2>/dev/null)
[ -n "$OJWT" ] || fail "ontul login failed"
OTOK=$(curl -s -m 30 -X POST "$ONTUL/admin/iam/keys" -H "Authorization: Bearer $OJWT" \
       -H 'Content-Type: application/json' -d '{"username":"admin"}' \
     | python3 -c "import sys,json;print(json.load(sys.stdin).get('token',''))" 2>/dev/null)
[ -n "$OTOK" ] || fail "could not mint an ontul user token"
step "minted an ontul user token (${OTOK:0:6}…)"

CONN=$(python3 - "$ONTUL_INTERNAL" "$OTOK" <<'PY'
import json, sys
print(json.dumps({
    "connectionId": "regdemoOntul",
    "type": "generic",
    "description": "ontul master for the regulation indexing DAG",
    "properties": {"url": sys.argv[1], "token": sys.argv[2]},
}))
PY
)
code=$(curl -s -o /tmp/kiok-conn.out -w '%{http_code}' -X POST "$KIOK/api/v1/connections" \
        -H "$KAH" -H 'Content-Type: application/json' -d "$CONN")
case "$code" in 2*) step "kiok connection regdemoOntul registered";;
  *) fail "connection registration failed HTTP $code: $(head -c 200 /tmp/kiok-conn.out)";; esac

# The S3 credentials live in a connection for the same reason. The discover job
# needs them to list the bucket from the worker, and writing them into the DAG
# would put them straight in the UI.
S3CONN=$(python3 - "$S3_ENDPOINT_INTERNAL" "$S3_ACCESS_KEY" "$S3_SECRET_KEY" "${S3_REGION:-us-east-1}" <<'PY'
import json, sys
print(json.dumps({
    "connectionId": "regdemoS3",
    "type": "s3",
    "description": "shannonstore, as the pipeline's jobs see it from inside the network",
    "properties": {"endpoint": sys.argv[1], "accessKey": sys.argv[2],
                   "secretKey": sys.argv[3], "region": sys.argv[4], "pathStyle": "true"},
}))
PY
)
code=$(curl -s -o /tmp/kiok-conn-s3.out -w '%{http_code}' -X POST "$KIOK/api/v1/connections" \
        -H "$KAH" -H 'Content-Type: application/json' -d "$S3CONN")
case "$code" in 2*) step "kiok connection regdemoS3 registered";;
  *) fail "S3 connection registration failed HTTP $code: $(head -c 200 /tmp/kiok-conn-s3.out)";; esac

# Registered in Ontul under the same id. kiok uploads and Ontul downloads, so
# both stores have to know the same credentials — Ontul is what fetches a PYTHON
# job's s3:// scriptPath, and without this it ends at
# "ontul.deps.s3.connectionId is required to fetch s3:// dep paths".
OCONN=$(python3 - "$S3_ENDPOINT_INTERNAL" "$S3_ACCESS_KEY" "$S3_SECRET_KEY" "${S3_REGION:-us-east-1}" <<'PY'
import json, sys
print(json.dumps({
    "connectionId": "regdemoS3",
    "type": "S3",
    "properties": {"endpoint": sys.argv[1], "accessKey": sys.argv[2],
                   "secretKey": sys.argv[3], "region": sys.argv[4], "pathStyle": "true"},
}))
PY
)
code=$(curl -s -o /tmp/ontul-conn-s3.out -w '%{http_code}' -X POST "$ONTUL/admin/connections" \
        -H "Authorization: Bearer $OJWT" -H 'Content-Type: application/json' -d "$OCONN")
# NeorunBase's own wire credentials. Clearing the vectors is the one step that
# does not go through Ontul, so it needs a connection of its own.
NBCONN=$(python3 - "${NEORUNBASE_INTERNAL_HOST:-neorun-coordinator-1}" "${NEORUNBASE_PASSWORD:-Regdemo12345}" <<'PY'
import json, sys
print(json.dumps({
    "connectionId": "regdemoNeorun",
    "type": "jdbc",
    "description": "NeorunBase postgres wire — for the one step Ontul cannot route",
    # 5432, not 5434. The demo publishes NeorunBase's wire on the host as 5434 to
    # avoid a bind clash, but a job running inside the network reaches the
    # container's own port — and connecting to the host mapping from in there
    # gets "Connection refused" with nothing to say why.
    "properties": {"host": sys.argv[1], "port": "5432", "database": "neorunbase",
                   "username": "admin", "password": sys.argv[2]},
}))
PY
)
curl -s -o /tmp/kiok-conn-nb.out -w '%{http_code}' -X POST "$KIOK/api/v1/connections" \
      -H "$KAH" -H 'Content-Type: application/json' -d "$NBCONN" >/dev/null
step "kiok connection regdemoNeorun registered"

case "$code" in 2*) step "ontul connection regdemoS3 registered";;
  *) printf '\033[1;33m  ..\033[0m ontul S3 connection HTTP %s (ignored if it already exists)\n' "$code";; esac

if [ "$MODE" = "status" ]; then
  log "recent runs"
  curl -s -m 30 "$KIOK/api/v1/dags/regdemo_index/runs" -H "$KAH" | python3 -c "
import sys,json
try: runs=json.load(sys.stdin)
except Exception: print('  (none)'); raise SystemExit
runs = runs if isinstance(runs,list) else runs.get('runs',[])
for r in runs[:5]:
    print('  ', r.get('runId') or r.get('id'), r.get('state'), r.get('startedAt',''))"
  exit 0
fi

# ── Register the UDFs ───────────────────────────────────────────────────────
# The chunk task calls extract_chunks(uri). It has to be registered at GLOBAL
# scope: a session-scoped UDF is visible only to the connection that registered
# it, and the scheduler's task opens its own — which ends as
# "No match found for function signature".
log "0/2  registering UDFs"
PYBIN="$DEMO/.venv/bin/python"
[ -x "$PYBIN" ] || fail "no virtualenv — python3 -m venv .venv && .venv/bin/pip install -e pipeline"
( cd "$DEMO/pipeline/src" && PYTHONPATH=. "$PYBIN" -m regdemo_pipeline.jobs.register_udfs \
    --out "$DEMO/out" --password "$ONTUL_PW" ) || fail "UDF registration failed"

# ── Publish the job sources ─────────────────────────────────────────────────
# The DAG refers to the scripts as ${conn.regdemoJobs.<name>}, and publish.sh
# fills that connection. Without publishing, discover dies with exitCode -1 the
# moment it starts and the log does not say why — the reference simply did not
# resolve.
log "0/2  publishing job sources"
bash "$DEMO/pipeline/jobs/publish.sh" || fail "publishing the jobs failed"

log "1/2  registering the DAG"
# Nothing to substitute — the DAG carries only ${conn.regdemoOntul.*} references
# and the values live in the connections registered above. That is why the file
# can be committed as it is.
code=$(curl -s -o /tmp/kiok-dag.out -w '%{http_code}' -X POST "$KIOK/api/v1/dags" \
        -H "$KAH" -H 'Content-Type: application/yaml' --data-binary @"$DAG_FILE")
case "$code" in 2*) step "regdemo_index registered";; *) fail "registration failed HTTP $code: $(head -c 200 /tmp/kiok-dag.out)";; esac

[ "$MODE" = "register" ] && exit 0

log "2/2  running it"
RUN=$(curl -s -m 60 -X POST "$KIOK/api/v1/dags/regdemo_index/runs" -H "$KAH" \
      -H 'Content-Type: application/json' -d '{}' \
    | python3 -c "import sys,json;d=json.load(sys.stdin);print(d.get('runId') or d.get('id') or '')" 2>/dev/null)
[ -n "$RUN" ] || fail "could not create a run"
step "runId=$RUN"

STATE=""
for i in $(seq 1 120); do
  STATE=$(curl -s -m 30 "$KIOK/api/v1/runs/$RUN" -H "$KAH" \
        | python3 -c "import sys,json;print(json.load(sys.stdin).get('state',''))" 2>/dev/null)
  printf '\r  [%03ds] %s        ' $((i*5)) "${STATE:-...}"
  case "$STATE" in SUCCESS|FAILED|CANCELLED|ERROR) break;; esac
  sleep 5
done
echo

# Per-task results. Knowing which stage stopped is the point of running this as
# a pipeline at all.
curl -s -m 30 "$KIOK/api/v1/runs/$RUN/tasks" -H "$KAH" | python3 -c "
import sys,json
try: ts=json.load(sys.stdin)
except Exception: raise SystemExit
ts = ts if isinstance(ts,list) else ts.get('tasks',[])
for t in ts:
    mark = 'ok ' if t.get('state')=='SUCCESS' else '   '
    print(f\"  {mark}{t.get('taskId') or t.get('id'):<18} {t.get('state','')}\")"

[ "$STATE" = "SUCCESS" ] && { printf '\033[1;32mpipeline SUCCESS\033[0m\n'; exit 0; } \
                         || { printf '\033[1;31mpipeline %s\033[0m\n' "$STATE"; exit 1; }
```


```bash
bash infra/pipeline.sh
```

```text
== 0/2  UDF 등록
registered extract_chunks, jfield, doc_no_for, chunk_pk (scope=GLOBAL, register=50 documents)
verified from a separate connection

== 0/2  잡 소스 게시
  build_graph_job -> cacca440c425
  clear_vectors_job -> 8092e80a5531
  discover_job -> b572e5d46118

== 2/2  실행
  ok runId=regdemo_index-1787451103180-9
  [005s] PENDING  [010s] RUNNING  …  [050s] SUCCESS
파이프라인 SUCCESS
```

Check it:

```sql
SELECT count(*) FROM ice.reg.doc_chunks;                                   -- 818
SELECT count(*) FROM (SELECT chunk_id FROM ice.reg.doc_chunks
                      GROUP BY chunk_id HAVING count(*) > 1) t;            -- 0  (idempotent)
```

```bash
psql -h localhost -p 5434 -U admin -d neorunbase \
  -c "SELECT count(*) FROM doc_vectors_gen1"   # 818
```

![kiok run history](../images/demo/kiok-executions.png)

---

## The embedding model

**`demo/pipeline/src/regdemo_pipeline/embed/model.py`**

```python
"""Single source of truth for the embedding model.

Indexing runs on Ontul workers (Python UDF); querying runs in the agent process.
Two processes, one vector space — and a mismatch between them does not raise, it
silently returns wrong neighbours. Every temporal-isolation guarantee downstream
is worthless if the vectors being compared came from different models.

Three things are pinned here, and all three fail silently if they drift:
  1. the model + revision       (different weights → different space)
  2. dim + normalisation        (VECTOR(n) storage, cosine == inner product)
  3. the e5 prefix convention   (see below)
"""
from __future__ import annotations

import os

# ── The pin ──────────────────────────────────────────────────────────────────
# Multilingual, 768-dim. Chosen over bge-m3 because article-level chunks run
# 100–300 tokens, so bge-m3's 8192 window buys nothing while costing ~3x the
# resident memory — and the model is resident on every worker, not just once.
MODEL_ID = "intfloat/multilingual-e5-base"

# Pinned at deploy. A bare model name resolves to "whatever the hub serves
# today", which is exactly the silent drift this module exists to prevent.
MODEL_REVISION = os.environ.get("EMBED_MODEL_REVISION", "").strip()

DIM = 768
NORMALIZE = True        # L2-normalise, so inner product == cosine
DISTANCE = "cosine"     # must match the NeorunBase HNSW index metric
MAX_TOKENS = 512

# ── e5 prefix convention ─────────────────────────────────────────────────────
# e5 is trained asymmetrically: stored text is prefixed "passage: ", the search
# string "query: ". Getting this wrong costs real retrieval quality and raises
# nothing — so the prefixes are part of the pinned contract, applied by the
# encoder rather than left to each caller to remember.
PASSAGE_PREFIX = "passage: "
QUERY_PREFIX = "query: "


def fingerprint(revision: str | None = None) -> str:
    """Identity stored on an embedding generation and matched at query time.

    Includes dim, normalisation and the prefix scheme, because two runs of the
    same weights that differ in any of them are as incomparable as two models.
    """
    rev = (revision if revision is not None else MODEL_REVISION).strip()
    if not rev:
        raise RuntimeError(
            "EMBED_MODEL_REVISION is unset. Pin the exact revision used to build "
            "the index — an unpinned model silently changes the vector space."
        )
    return f"{MODEL_ID}@{rev}:{DIM}:{'l2' if NORMALIZE else 'raw'}:e5v1"


def vector_column_ddl(name: str = "embedding") -> str:
    """NeorunBase column type. The dimension is fixed per column, which is why
    a model change means a new generation table rather than an ALTER."""
    return f"{name} VECTOR({DIM})"
```
**`demo/pipeline/src/regdemo_pipeline/embed/encoder.py`**

```python
"""The only place the model is actually loaded.

Held for the process lifetime. That matters most on an Ontul worker, where the
alternative is reloading the model on every UDF batch.
"""
from __future__ import annotations

import logging, os, threading, time
from . import model

log = logging.getLogger(__name__)
_instance, _lock = None, threading.Lock()


class Encoder:
    def __init__(self, device: str | None = None) -> None:
        from sentence_transformers import SentenceTransformer
        started = time.time()
        kwargs = {"revision": model.MODEL_REVISION} if model.MODEL_REVISION else {}
        self.st = SentenceTransformer(model.MODEL_ID, device=device, **kwargs)
        self.device = str(self.st.device)
        self.load_seconds = time.time() - started

        dim = self.st.get_sentence_embedding_dimension()
        if dim != model.DIM:
            # The dimension is baked into VECTOR(n); a mismatch would produce
            # rows the target table cannot hold.
            raise RuntimeError(f"model dim={dim} but pin declares {model.DIM}")

    def resolved_revision(self) -> str:
        """The commit actually loaded — the value to pin. Reading it back turns
        'whatever the hub served' into a recorded fact before any index exists."""
        for m in self.st._modules.values():
            p = getattr(getattr(m, "auto_model", None), "name_or_path", "") or ""
            if "/snapshots/" in p:
                return p.split("/snapshots/")[1].split("/")[0]
        return ""

    def _encode(self, texts, prefix, batch_size):
        return self.st.encode([prefix + t for t in texts], batch_size=batch_size,
                              normalize_embeddings=model.NORMALIZE,
                              show_progress_bar=False, convert_to_numpy=True)

    def encode_passages(self, texts: list[str], batch_size: int = 16):
        """Index side. Prefix applied here so no caller can forget it."""
        return self._encode(texts, model.PASSAGE_PREFIX, batch_size)

    def encode_query(self, text: str):
        """Query side — a different prefix, deliberately not interchangeable."""
        return self._encode([text], model.QUERY_PREFIX, 1)[0]


def get(device: str | None = None) -> Encoder:
    global _instance
    if _instance is None:
        with _lock:
            if _instance is None:
                _instance = Encoder(device=device or os.environ.get("EMBED_DEVICE"))
    return _instance
```
**`demo/pipeline/src/regdemo_pipeline/embed/smoke.py`**

```python
"""Standalone check: does the embedding model work, and what does it cost?

Run before wiring anything into a UDF. Answers four questions:
  1. does it load, and how long / how much resident memory (x every worker)
  2. what revision actually loaded (the value to pin in EMBED_MODEL_REVISION)
  3. throughput on article-length Korean text (the real corpus shape)
  4. does Korean similarity behave — a regulation query must rank its own
     article above an unrelated one, or nothing downstream can work

Usage:  python -m regdemo_pipeline.embed.smoke
"""
from __future__ import annotations

import resource, sys, time
import numpy as np

from . import model
from .encoder import get

# Article-length Korean text, shaped like the corpus the pipeline will index.
PASSAGES = [
    "제12조(육아휴직) ① 만 8세 이하 자녀를 양육하는 직원은 육아휴직을 신청할 수 있다. "
    "② 육아휴직 기간은 연간 20일 이내로 한다. ③ 신청은 사용 예정일 30일 전까지 인사팀에 제출한다.",
    "제13조(연차유급휴가) ① 1년간 80퍼센트 이상 출근한 직원에게 15일의 유급휴가를 부여한다. "
    "② 3년 이상 계속 근로한 직원에게는 매 2년마다 1일을 가산한다.",
    "제7조(출장비의 지급) ① 국내출장은 일비·숙박비·교통비를 실비로 지급한다. "
    "② 부장 이상은 숙박비 상한을 1박 12만원으로 한다.",
    "제5조(정보자산의 반출) ① 사내 정보자산을 외부로 반출할 때에는 정보보안팀장의 승인을 받아야 한다.",
    "제3조(구매 승인 한도) ① 500만원 미만은 팀장, 500만원 이상 5천만원 미만은 본부장이 승인한다.",
]
QUERIES = [
    ("육아휴직 며칠까지 쓸 수 있어?", 0),
    ("연차는 몇 일 나와?", 1),
    ("출장 숙박비 얼마까지 되나", 2),
    ("노트북 외부로 가져가려면", 3),
    ("천만원짜리 발주 누가 결재해?", 4),
]


def rss_gb() -> float:
    # macOS reports maxrss in bytes; Linux in kilobytes.
    m = resource.getrusage(resource.RUSAGE_SELF).ru_maxrss
    return m / 1024**3 if sys.platform == "darwin" else m / 1024**2


def main() -> int:
    base = rss_gb()
    print(f"model      : {model.MODEL_ID}  (declared dim {model.DIM})")

    enc = get()
    rev = enc.resolved_revision()
    print(f"load       : {enc.load_seconds:.1f}s  device={enc.device}")
    print(f"memory     : +{rss_gb() - base:.2f} GB  (total {rss_gb():.2f} GB) — per worker")
    print(f"revision   : {rev or '(could not determine)'}")
    if rev:
        print(f"fingerprint: {model.fingerprint(rev)}")

    # throughput on corpus-shaped text
    batch = PASSAGES * 40  # 200 chunks
    t0 = time.time()
    vecs = enc.encode_passages(batch, batch_size=16)
    dt = time.time() - t0
    print(f"\nthroughput : {len(batch)} chunks / {dt:.1f}s = {len(batch)/dt:.0f} chunks/s")
    print(f"             → 6,000 chunks in an estimated {6000/(len(batch)/dt):.0f}s (single process)")
    print(f"vectors    : shape={np.array(vecs).shape}  "
          f"norm={np.linalg.norm(vecs[0]):.4f} (L2 normalisation confirmed)")

    # the check that actually matters: does Korean retrieval rank correctly
    P = np.array(enc.encode_passages(PASSAGES))
    print("\nKorean retrieval sanity (query → top article)")
    ok = 0
    for q, expect in QUERIES:
        sims = P @ enc.encode_query(q)
        top = int(np.argmax(sims))
        hit = top == expect
        ok += hit
        print(f"  {'PASS' if hit else 'FAIL'}  {q:<24} → article #{top} "
              f"(sim {sims[top]:.3f}, expected #{expect} {sims[expect]:.3f})")
    print(f"\nresult: {ok}/{len(QUERIES)} passed")
    return 0 if ok == len(QUERIES) else 1


if __name__ == "__main__":
    raise SystemExit(main())
```


---

Next: [CDC and Flow](cdc-flow.md) — the jobs that never finish.
