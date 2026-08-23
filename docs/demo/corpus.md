# The corpus — deliberately messy

The real work in an indexing pipeline is not pulling text out of a PDF. It is
working out **which regulation a file is, and which revision of it**. In a real
organisation the filename reads `[최종]육아지원규정(2022.06.01).pdf` — "final" —
and the file is a superseded version.

So this corpus is not clean. On purpose.

| Defect | Count | Why it is here |
|---|---|---|
| Lying filenames | — | `[최종]…(2022.06.01).pdf` names the **superseded** version |
| Name collisions | 31 | Resolved the way a drive resolves them, ` (2)`, and recorded |
| Register defects | 19 | Unregistered · phantom rows · title mismatch · stale version |
| Scan-only PDFs | 6 | **Reported** as `image_only_pdf`, never silently dropped |
| Documents with PII | 37 | Free text, so masking swaps in a redacted twin |
| Effective-date conflicts | 27 | Approval date ≠ the date printed in the document |

Everything deliberate is recorded in `out/ground_truth.json`, so matching and
extraction can be **scored** rather than eyeballed.

!!! quote "Where it stands"
    123 of 446 files matched, **0 misidentified**. The second number matters more:
    a wrong document number makes one regulation's text serve as another's
    evidence, and it fails silently.

---

## The generator

```bash
make seed
```

**`demo/Makefile`**

```text
# Regulation lakehouse demo.
SHELL := /bin/bash
OUT   ?= ./out
PY    ?= python3

.PHONY: help seed up down logs ps clean smoke

help:
	@grep -E '^[a-z-]+:.*##' $(MAKEFILE_LIST) | sed 's/:.*## /\t/' | expand -t22

seed: ## Generate the corpus, ERP/approval SQL and the LMS payload
	cd seed/src && PYTHONPATH=. $(PY) -m regdemo_seed --out $(abspath $(OUT))

up: ## Bring up the sources (requires `make seed` first — SQL is mounted at init)
	@test -f $(OUT)/sql/erp_postgres.sql || { echo "run 'make seed' first"; exit 1; }
	docker compose up -d --build

down: ## Stop and remove containers and volumes
	docker compose down -v

ps: ## Service status
	docker compose ps

logs: ## Tail logs
	docker compose logs -f --tail=100

smoke: ## Check the embedding model standalone (loads it in-process, not the service)
	cd pipeline/src && PYTHONPATH=. $(PY) -m regdemo_pipeline.embed.smoke

clean: ## Remove generated output
	rm -rf $(OUT)
```


### Entry point

**`demo/seed/src/regdemo_seed/__main__.py`**

```python
"""Generate the corpus.

    python -m regdemo_seed --out ./out

Writes the files an organisation would have on a shared drive, the register that
describes them, and a ground-truth file recording what was deliberately made
imperfect — so a pipeline's matching and extraction can be scored rather than
eyeballed.
"""
from __future__ import annotations

import argparse
import hashlib
import json
import random
import sys
from dataclasses import asdict
from pathlib import Path

from . import erp, general, groupware, master, regulations as R, render, training
from .org import DEPT_BY_CODE, build_employees, rng as base_rng


def main(argv: list[str] | None = None) -> int:
    ap = argparse.ArgumentParser(prog="regdemo_seed")
    ap.add_argument("--out", type=Path, default=Path("out"))
    ap.add_argument("--general", type=int, default=300)
    ap.add_argument("--font", default=None, help="Korean TTF for PDF rendering")
    args = ap.parse_args(argv)

    font = render.register_font(args.font)
    out = args.out
    docs_dir = out / "documents"
    if docs_dir.exists():
        import shutil
        shutil.rmtree(docs_dir)

    rng = base_rng()
    employees = build_employees(rng)
    regs = R.build(rng)
    frng = random.Random(4242)

    manifest: list[dict] = []
    written = 0

    collisions: list[str] = []

    def write(key: str, data: bytes, meta: dict) -> None:
        """Write, disambiguating a name clash the way a shared drive does.

        Two versions of a regulation can easily land on the same filename once
        the version is only implied by words like 최종. Overwriting would lose a
        file the manifest still claims exists — and the collided name is itself
        part of what the matcher has to cope with, so it is recorded rather than
        avoided by generating unique names.
        """
        nonlocal written
        p = docs_dir / key
        if p.exists():
            collisions.append(key)
            stem, dot, ext = key.rpartition(".")
            n = 1
            while p.exists():
                n += 1
                key = f"{stem} ({n}){dot}{ext}"
                p = docs_dir / key
        p.parent.mkdir(parents=True, exist_ok=True)
        p.write_bytes(data)
        manifest.append({**meta, "s3_key": key, "bytes": len(data),
                         "sha256": hashlib.sha256(data).hexdigest()})
        written += 1

    # ── Regulations: one file per version, sometimes two ─────────────────────
    for reg in regs:
        for v in reg.versions:
            base = {"doc_no": reg.doc_no, "version": v.version, "title": reg.title,
                    "tier": reg.tier, "owner_dept": reg.owner_dept,
                    "sensitivity": reg.sensitivity, "is_official": reg.is_official,
                    "status": v.status,
                    "effective_from": v.effective_from.isoformat(),
                    "effective_to": v.effective_to.isoformat() if v.effective_to else None,
                    "stated_from": v.stated_from.isoformat(),
                    "date_mismatch": v.date_mismatch, "kind": "REGULATION"}

            # A small share of older versions survive only as scans. Their text
            # is unrecoverable, and the pipeline has to say so rather than drop
            # them quietly.
            #
            # PROTECTED documents are exempt. The scenarios turn on being able to
            # read what a superseded version actually said — an as-of question has no
            # answer if that version exists only as an image.
            # Losing the corpus's central evidence to a random 10% would make the
            # demo depend on a seed, which is a worse defect than the one being
            # injected.
            scanned = (reg.doc_no not in master.PROTECTED
                       and v.status != "EFFECTIVE"
                       and frng.random() < 0.10)
            if scanned:
                fn = render.filename(reg, v, "pdf", frng)
                write(render.s3_key(reg, fn),
                      render.render_scanned_pdf(reg, v),
                      {**base, "file_format": "pdf", "scanned": True,
                       "is_authoritative": True})
                continue

            fn = render.filename(reg, v, "pdf", frng)
            write(render.s3_key(reg, fn), render.render_pdf(reg, v),
                  {**base, "file_format": "pdf", "scanned": False,
                   "is_authoritative": True})

            # The working copy alongside the published one: same content, different
            # bytes, so a content hash will not collapse them. De-duplication has
            # to prefer the published form on other grounds.
            if frng.random() < 0.25:
                fn2 = render.filename(reg, v, "docx", frng)
                write(render.s3_key(reg, fn2), render.render_docx(reg, v),
                      {**base, "file_format": "docx", "scanned": False,
                       "is_authoritative": False, "duplicate_of": reg.doc_no})

    reg_files = written

    # ── General documents ────────────────────────────────────────────────────
    gdocs = general.build(random.Random(7), employees, args.general)
    for i, g in enumerate(gdocs):
        dept_name = DEPT_BY_CODE[g.dept].name
        fake = R.Regulation(f"GEN-{i:04d}", g.title, 3, g.dept, None, "INTERNAL", False)
        fake_v = R.Version(1, g.created, None, g.created, "EFFECTIVE",
                           [R.Article(1, "", p) for p in g.paragraphs])
        ext = g.fmt
        fn = f"{g.title}.{ext}".replace("/", "_")
        key = f"{dept_name}/{g.kind}/{fn}"
        data = (render.render_docx(fake, fake_v) if ext == "docx"
                else render.render_pdf(fake, fake_v))
        write(key, data, {"doc_no": None, "version": None, "title": g.title,
                          "kind": "GENERAL", "sub_kind": g.kind,
                          "owner_dept": g.dept, "sensitivity": "INTERNAL",
                          "is_official": False, "file_format": ext,
                          "scanned": False, "is_authoritative": True,
                          "has_pii": g.has_pii,
                          "created": g.created.isoformat()})

    # ── Register ─────────────────────────────────────────────────────────────
    defects = master.build(regs, random.Random(3), out / "regulations_master.xlsx")

    # ── The other systems ────────────────────────────────────────────────────
    # Generated here rather than independently so employee numbers, department
    # codes and approval dates are the same facts the documents describe — a
    # join across them is the whole point, and it only works if one generator
    # owns the identifiers.
    erp_sql, erp_facts = erp.build_sql(employees, random.Random(11))
    gw_sql, gw_facts = groupware.build_sql(regs, employees, random.Random(13))
    lms_payload, lms_facts = training.build(regs, employees, random.Random(17))

    (out / "sql").mkdir(parents=True, exist_ok=True)
    (out / "sql" / "erp_postgres.sql").write_text(erp_sql, encoding="utf-8")
    (out / "sql" / "groupware_mysql.sql").write_text(gw_sql, encoding="utf-8")
    (out / "lms.json").write_text(
        json.dumps(lms_payload, ensure_ascii=False, indent=2), encoding="utf-8")

    truth = {
        "font": font,
        "counts": {"regulations": len(regs),
                   "regulation_versions": sum(len(r.versions) for r in regs),
                   "regulation_files": reg_files,
                   "general_documents": len(gdocs),
                   "total_files": written},
        "register_defects": defects,
        "erp": erp_facts,
        "groupware": gw_facts,
        "training": lms_facts,
        "filename_collisions": collisions,
        "expected": {
            # What the scenarios assert against.
            "childcare_current_days": R.LEAVE_DAYS_NEW,
            "childcare_superseded_days": R.LEAVE_DAYS_OLD,
            "childcare_doc": "HR-REG-003",
            "date_mismatch_docs": sorted({r.doc_no for r in regs
                                          for v in r.versions if v.date_mismatch}),
            "restricted_docs": sorted(r.doc_no for r in regs
                                      if r.sensitivity == "RESTRICTED"),
            "scanned_files": sorted(m["s3_key"] for m in manifest if m.get("scanned")),
            "pii_files": sorted(m["s3_key"] for m in manifest if m.get("has_pii")),
        },
        "relations": [
            {"src": r.doc_no, "dst": r.parent, "rel_type": "CHILD_OF"}
            for r in regs if r.parent
        ] + [
            {"src": r.doc_no, "src_version": v.version, "dst": r.doc_no,
             "dst_version": v.version - 1, "rel_type": "SUPERSEDES"}
            for r in regs for v in r.versions if v.version > 1
        ],
    }
    (out / "manifest.json").write_text(
        json.dumps(manifest, ensure_ascii=False, indent=2), encoding="utf-8")
    (out / "ground_truth.json").write_text(
        json.dumps(truth, ensure_ascii=False, indent=2), encoding="utf-8")

    c = truth["counts"]
    print(f"regulations {c['regulations']} · versions {c['regulation_versions']} "
          f"→ files {c['regulation_files']}")
    print(f"general documents {c['general_documents']}")
    print(f"files in total {c['total_files']}  ({out})")
    print(f"name collisions {len(collisions)} (resolved with ' (n)', as a drive would)")
    print(f"register defects {sum(len(v) for v in defects.values())} · "
          f"scanned-only {len(truth['expected']['scanned_files'])} · "
          f"with PII {len(truth['expected']['pii_files'])}")
    d = erp_facts["demo_employee"]
    print(f"ERP {erp_facts['counts']['employees']} employees · "
          f"approvals {gw_facts['counts']['approvals']} "
          f"({len(gw_facts['date_mismatches'])} with date conflicts) · "
          f"training records {lms_facts['counts']['completions']}")
    print(f"the scenario: {d['name']} ({d['emp_no']}) has "
          f"{d['correct_remaining']} childcare days left — "
          f"{d['wrong_remaining_if_superseded']} if answered from the superseded version")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```


### The organisation — one owner for identifiers

Employee numbers are minted in exactly one place. ERP, the approval system and
the training records all use them. If each source invented its own scheme the
federated queries would not join, and the demo would be an exercise in data
reconciliation rather than a demo.

**`demo/seed/src/regdemo_seed/org.py`**

```python
"""Shared organisational spine for every seeded source.

Regulations, ERP rows, approval records and training completions are produced by
different generators but have to join. Department codes, employee numbers and
document numbers therefore live here — one registry imported by all of them —
rather than being invented independently and reconciled afterwards.

Deterministic: a fixed seed means two runs produce an identical corpus, so an
assertion written against the data stays valid.
"""
from __future__ import annotations

import random
from dataclasses import dataclass
from datetime import date, timedelta

SEED = 20260819
CORPUS_START = date(2019, 1, 1)   # oldest regulation enactment
TODAY = date(2026, 8, 19)         # "now" for the demo


# ── Departments ──────────────────────────────────────────────────────────────
@dataclass(frozen=True)
class Dept:
    code: str
    name: str
    parent: str | None
    headcount: int
    doc_prefix: str    # document-number prefix for regulations this dept owns


DEPTS: list[Dept] = [
    Dept("CEO", "대표이사",     None,  2,  "GEN"),
    Dept("HR",  "인사팀",       "CEO", 12, "HR"),
    Dept("FIN", "재무팀",       "CEO", 15, "FIN"),
    Dept("GA",  "총무팀",       "CEO", 10, "GA"),
    Dept("LEG", "법무팀",       "CEO", 6,  "LEG"),
    Dept("SEC", "정보보안팀",   "CEO", 8,  "SEC"),
    Dept("DEV", "개발팀",       "CEO", 95, "DEV"),
    Dept("QA",  "품질팀",       "DEV", 22, "QA"),
    Dept("SLS", "영업팀",       "CEO", 68, "SLS"),
    Dept("MKT", "마케팅팀",     "CEO", 28, "MKT"),
    Dept("PUR", "구매팀",       "FIN", 14, "PUR"),
    Dept("CS",  "고객지원팀",   "SLS", 20, "CS"),
]
DEPT_BY_CODE = {d.code: d for d in DEPTS}
TOTAL_HEADCOUNT = sum(d.headcount for d in DEPTS)   # 300


# ── Employees ────────────────────────────────────────────────────────────────
# Grade drives both the purchase-approval limit and the leave entitlement, each set
# by its own regulation — so the document corpus and the ERP rows have to agree on
# it.
GRADES = ["사원", "대리", "과장", "차장", "부장", "이사"]
GRADE_WEIGHTS = [0.34, 0.26, 0.20, 0.11, 0.07, 0.02]

SURNAMES = ["김", "이", "박", "최", "정", "강", "조", "윤", "장", "임",
            "한", "오", "서", "신", "권", "황", "안", "송", "류", "홍"]
GIVEN = ["민준", "서연", "도윤", "지우", "예준", "하은", "시우", "지민", "주원", "수아",
         "지호", "다은", "건우", "채원", "우진", "지아", "선우", "유진", "현우", "소율",
         "준서", "예린", "성민", "가은", "동현", "나윤", "재윤", "서윤", "태윤", "하린"]


@dataclass(frozen=True)
class Employee:
    emp_no: str
    name: str
    dept: str
    grade: str
    hired_on: date
    status: str        # 재직 / 휴직 / 퇴직

    @property
    def years_of_service(self) -> int:
        return (TODAY - self.hired_on).days // 365


def build_employees(rng: random.Random) -> list[Employee]:
    """300 employees, numbered by hire year so the sequence reads like a real
    ERP (20090001, 20250014 …) rather than a synthetic 1..300 range."""
    out: list[Employee] = []
    seq: dict[int, int] = {}
    for dept in DEPTS:
        for _ in range(dept.headcount):
            hired = (CORPUS_START - timedelta(days=rng.randint(0, 3650))
                     if rng.random() < 0.35 else
                     CORPUS_START + timedelta(days=rng.randint(0, 2600)))
            year = hired.year
            seq[year] = seq.get(year, 0) + 1
            emp_no = f"{year}{seq[year]:04d}"
            svc = (TODAY - hired).days // 365
            # Seniority bounded by service years: a one-year manager would break the
            # approval-limit scenario, which keys off grade.
            cap = min(len(GRADES) - 1, 1 + svc // 3)
            grade = rng.choices(GRADES[: cap + 1], weights=GRADE_WEIGHTS[: cap + 1])[0]
            status = rng.choices(["재직", "휴직", "퇴직"], weights=[0.93, 0.04, 0.03])[0]
            out.append(Employee(emp_no, rng.choice(SURNAMES) + rng.choice(GIVEN),
                                dept.code, grade, hired, status))
    return out


# ── Document numbering ───────────────────────────────────────────────────────
def doc_no(prefix: str, tier: int, serial: int) -> str:
    """HR-REG-003 / SEC-GDL-011. The tier is encoded in the number itself, which
    is how a real numbering scheme signals hierarchy — and it lets the parser
    cross-check a document's declared tier against its own identifier."""
    kind = {1: "RUL", 2: "REG", 3: "GDL"}[tier]
    return f"{prefix}-{kind}-{serial:03d}"


def rng() -> random.Random:
    return random.Random(SEED)
```


### The regulations

**`demo/seed/src/regdemo_seed/regulations.py`**

```python
"""The regulation catalogue — what exists, how it is numbered, how it changed.

Separated from PDF rendering because the *facts* are what every other seeded
source has to agree with: the approval system's effective dates, the ERP's leave
entitlements, the graph's parent links. Rendering is one consumer of this; the
groupware and ERP generators are others.

The numbers in here are load-bearing. The leave-days figure a regulation states
is the same figure the agent must answer with, and the superseded version states
a different one — that difference is how a temporal-isolation failure becomes a
wrong answer rather than merely a wrong citation.
"""
from __future__ import annotations

import random
from dataclasses import dataclass, field
from datetime import date, timedelta

from .org import DEPTS, DEPT_BY_CODE, TODAY, doc_no


@dataclass
class Version:
    version: int
    effective_from: date      # authoritative: the approval date
    effective_to: date | None
    stated_from: date         # what the document's own 부칙 claims
    status: str               # EFFECTIVE / SUPERSEDED / ABOLISHED
    articles: list[Article] = field(default_factory=list)
    approval_id: str = ""

    @property
    def date_mismatch(self) -> bool:
        return self.effective_from != self.stated_from


@dataclass
class Article:
    no: int
    title: str
    body: str
    # Cross-references parsed out of the body later; kept here so the generator
    # and the expected-relations file cannot drift apart.
    refs: list[tuple[str, int]] = field(default_factory=list)   # (doc_no, article_no)


@dataclass
class Regulation:
    doc_no: str
    title: str
    tier: int                 # 1 취업규칙 · 2 규정 · 3 지침
    owner_dept: str
    parent: str | None        # CHILD_OF target
    sensitivity: str          # PUBLIC / INTERNAL / RESTRICTED
    is_official: bool
    versions: list[Version] = field(default_factory=list)

    @property
    def current(self) -> Version | None:
        return next((v for v in self.versions if v.status == "EFFECTIVE"), None)


# ── The one regulation the demo turns on ─────────────────────────────────────
# Childcare leave: v2 says 15 days, v3 says 20. v3 was approved on 2025-03-15 but
# its supplementary provision claims 2025-01-01 — so between January and March the
# answer was still 15,
# and a pipeline that trusts the document body gets that window wrong.
LEAVE_DAYS_OLD, LEAVE_DAYS_NEW = 15, 20

CHILDCARE_V2 = [
    Article(1, "(목적)", "이 규정은 직원의 육아 지원에 관한 사항을 정함을 목적으로 한다."),
    Article(2, "(적용범위)", "이 규정은 재직 중인 모든 직원에게 적용한다. 다만 수습기간 중인 자는 제외한다."),
    Article(3, "(육아휴직의 신청)",
            "① 만 8세 이하의 자녀를 양육하는 직원은 육아휴직을 신청할 수 있다. "
            f"② 육아휴직 기간은 연간 {LEAVE_DAYS_OLD}일 이내로 한다. "
            "③ 신청은 사용 예정일 30일 전까지 인사팀에 제출하여야 한다."),
    Article(4, "(급여)", "육아휴직 기간 중의 급여는 「급여규정」 제9조에 따른다.",
            refs=[("FIN-REG-001", 9)]),
]
CHILDCARE_V3 = [
    Article(1, "(목적)", "이 규정은 직원의 육아 지원에 관한 사항을 정함을 목적으로 한다."),
    Article(2, "(적용범위)", "이 규정은 재직 중인 모든 직원에게 적용한다. 수습기간 중인 자를 포함한다."),
    Article(3, "(육아휴직의 신청)",
            "① 만 8세 이하의 자녀를 양육하는 직원은 육아휴직을 신청할 수 있다. "
            f"② 육아휴직 기간은 연간 {LEAVE_DAYS_NEW}일 이내로 한다. "
            "③ 신청은 사용 예정일 14일 전까지 인사팀에 제출하여야 한다."),
    Article(4, "(급여)", "육아휴직 기간 중의 급여는 「급여규정」 제9조에 따른다.",
            refs=[("FIN-REG-001", 9)]),
    Article(5, "(분할 사용)", "육아휴직은 연 2회에 한하여 분할하여 사용할 수 있다."),
]

# ── Catalogue ────────────────────────────────────────────────────────────────
# (doc_no, title, tier, dept, parent, sensitivity, official)
CATALOGUE: list[tuple] = [
    ("GEN-RUL-001", "취업규칙",            1, "HR",  None,          "PUBLIC",     True),
    ("HR-REG-001",  "인사규정",            2, "HR",  "GEN-RUL-001", "PUBLIC",     True),
    ("HR-REG-002",  "복무규정",            2, "HR",  "GEN-RUL-001", "PUBLIC",     True),
    ("HR-REG-003",  "육아지원규정",        2, "HR",  "HR-REG-001",  "PUBLIC",     True),
    ("HR-GDL-001",  "연차휴가 운영지침",   3, "HR",  "HR-REG-002",  "INTERNAL",   True),
    ("HR-GDL-002",  "재택근무 운영지침",   3, "HR",  "HR-REG-002",  "INTERNAL",   True),
    ("HR-GDL-003",  "징계 양정기준",       3, "HR",  "HR-REG-001",  "RESTRICTED", True),
    ("FIN-REG-001", "급여규정",            2, "FIN", "GEN-RUL-001", "RESTRICTED", True),
    ("FIN-REG-002", "여비규정",            2, "FIN", "GEN-RUL-001", "PUBLIC",     True),
    ("FIN-GDL-001", "국내출장비 지급지침", 3, "FIN", "FIN-REG-002", "INTERNAL",   True),
    ("FIN-GDL-002", "해외출장비 지급지침", 3, "FIN", "FIN-REG-002", "INTERNAL",   True),
    ("PUR-REG-001", "구매규정",            2, "PUR", "GEN-RUL-001", "PUBLIC",     True),
    ("PUR-GDL-001", "구매 승인한도 지침",  3, "PUR", "PUR-REG-001", "INTERNAL",   True),
    ("SEC-REG-001", "정보보안규정",        2, "SEC", "GEN-RUL-001", "PUBLIC",     True),
    ("SEC-GDL-001", "정보자산 반출지침",   3, "SEC", "SEC-REG-001", "INTERNAL",   True),
    ("SEC-GDL-002", "개인정보 처리지침",   3, "SEC", "SEC-REG-001", "RESTRICTED", True),
    ("GA-REG-001",  "문서관리규정",        2, "GA",  "GEN-RUL-001", "PUBLIC",     True),
    ("LEG-REG-001", "계약관리규정",        2, "LEG", "GEN-RUL-001", "INTERNAL",   True),
    ("DEV-GDL-001", "개발 보안코딩 지침",  3, "DEV", "SEC-REG-001", "INTERNAL",   True),
    ("QA-GDL-001",  "품질검사 운영지침",   3, "QA",  "GEN-RUL-001", "INTERNAL",   True),
]

# Filler regulations, generated per department. The hand-written catalogue above
# carries the scenarios; these give the graph enough depth for multi-hop
# traversal to be a real query rather than a two-row join, and give the register
# matcher a realistic number of near-miss titles to get wrong.
FILLER_TOPICS = {
    "HR":  [("복리후생규정", 2), ("교육훈련규정", 2), ("평가보상규정", 2),
            ("경조사 지원지침", 3), ("직무발명 보상지침", 3)],
    "FIN": [("회계규정", 2), ("예산관리규정", 2), ("자금운용규정", 2),
            ("법인카드 사용지침", 3), ("전표처리 지침", 3)],
    "GA":  [("자산관리규정", 2), ("차량운행규정", 2), ("사무용품 구매지침", 3),
            ("회의실 운영지침", 3)],
    "LEG": [("준법감시규정", 2), ("소송관리지침", 3), ("전자서명 운영지침", 3)],
    "SEC": [("접근통제지침", 3), ("보안사고 대응지침", 3), ("암호키 관리지침", 3)],
    "DEV": [("형상관리 지침", 3), ("배포 승인지침", 3), ("장애대응 지침", 3),
            ("코드리뷰 운영지침", 3)],
    "QA":  [("결함관리 지침", 3), ("시험성적서 작성지침", 3)],
    "SLS": [("영업활동규정", 2), ("고객정보 취급지침", 3), ("할인승인 지침", 3)],
    "MKT": [("브랜드 사용지침", 3), ("광고심의 지침", 3)],
    "PUR": [("협력사 등록지침", 3), ("입찰 운영지침", 3)],
    "CS":  [("고객응대 지침", 3), ("불만처리 지침", 3)],
}


def _filler_catalogue() -> list[tuple]:
    """Expand FILLER_TOPICS into catalogue rows, attaching each to a plausible
    parent so the hierarchy has more than two levels in most branches."""
    from .org import DEPT_BY_CODE as _D
    # Parent for a generated guideline: its department's own regulation when one
    # exists, else the top-level rule — which is how a real hierarchy looks once
    # a department has grown its own second tier.
    dept_reg = {d: n for d, n, t, dp, *_ in
                [(c[3], c[0], c[2], c[3]) + tuple(c[4:]) for c in CATALOGUE]
                if t == 2}
    rows = []
    for dept, items in FILLER_TOPICS.items():
        serial = 10
        for title, tier in items:
            prefix = _D[dept].doc_prefix
            kind = {2: "REG", 3: "GDL"}[tier]
            doc = f"{prefix}-{kind}-{serial:03d}"
            serial += 1
            parent = dept_reg.get(dept, "GEN-RUL-001")
            sens = "RESTRICTED" if "개인정보" in title or "고객정보" in title else \
                   ("INTERNAL" if tier == 3 else "PUBLIC")
            rows.append((doc, title, tier, dept, parent, sens, True))
    return rows


GENERIC_ARTICLES = [
    ("(목적)",     "이 {kind}은 관련 업무의 기준과 절차를 정함을 목적으로 한다."),
    ("(적용범위)", "이 {kind}은 전 부서에 적용한다."),
    ("(용어의 정의)", "이 {kind}에서 사용하는 용어의 뜻은 관련 법령 및 상위 규정에서 정하는 바에 따른다."),
    ("(담당부서)", "이 {kind}의 소관 부서는 {dept}으로 한다."),
    ("(절차)",     "관련 업무는 신청·검토·승인의 순으로 처리하며, 각 단계의 처리기한은 3영업일로 한다."),
    ("(기록의 보존)", "관련 기록은 처리 완료일부터 5년간 보존한다."),
    ("(예외의 승인)", "이 {kind}에서 정하지 아니한 사항은 소관 부서장의 승인을 받아 처리한다."),
]


def _articles(reg_title: str, tier: int, dept_name: str, parent: str | None,
              rng: random.Random) -> list[Article]:
    kind = {1: "규칙", 2: "규정", 3: "지침"}[tier]
    n = rng.randint(5, 9)
    arts: list[Article] = []
    for i, (title, body) in enumerate(GENERIC_ARTICLES[:n], start=1):
        arts.append(Article(i, title, body.format(kind=kind, dept=dept_name)))
    # A guideline states the article of its parent it derives from — the textual
    # form of CHILD_OF, and what the body parser has to recover.
    if parent and tier == 3:
        arts.append(Article(len(arts) + 1, "(근거)",
                            f"이 지침은 「{parent}」 제{rng.randint(3, 12)}조에 근거하여 정한다.",
                            refs=[(parent, 0)]))
    return arts


def build(rng: random.Random) -> list[Regulation]:
    regs: list[Regulation] = []
    for doc, title, tier, dept, parent, sens, official in CATALOGUE + _filler_catalogue():
        r = Regulation(doc, title, tier, dept, parent, sens, official)
        dept_name = DEPT_BY_CODE[dept].name

        if doc == "HR-REG-003":
            # The scenario regulation: an explicit v2 → v3 supersession where the
            # stated date and the approved date disagree.
            r.versions = [
                Version(2, date(2022, 6, 1), date(2025, 3, 14), date(2022, 6, 1),
                        "SUPERSEDED", CHILDCARE_V2),
                Version(3, date(2025, 3, 15), None, date(2025, 1, 1),
                        "EFFECTIVE", CHILDCARE_V3),
            ]
        else:
            n_ver = rng.choices([1, 2, 3, 4], weights=[0.35, 0.35, 0.2, 0.1])[0]
            start = date(2019, 1, 1) + timedelta(days=rng.randint(0, 900))
            versions: list[Version] = []
            for v in range(1, n_ver + 1):
                eff = start + timedelta(days=rng.randint(400, 900) * (v - 1))
                if eff > TODAY:
                    break
                # Most documents state the date they were actually approved;
                # a minority are drafted with a date the approval then misses.
                stated = eff if rng.random() > 0.25 else eff - timedelta(days=rng.randint(20, 90))
                versions.append(Version(v, eff, None, stated, "EFFECTIVE",
                                        _articles(title, tier, dept_name, parent, rng)))
            for i in range(len(versions) - 1):
                versions[i].status = "SUPERSEDED"
                versions[i].effective_to = versions[i + 1].effective_from - timedelta(days=1)
            r.versions = versions
        regs.append(r)
    return regs
```


### The register (the master spreadsheet)

Filenames cannot be trusted, so **the register is the authority**. Only the
result of matching against it reaches the ledger.

**`demo/seed/src/regdemo_seed/master.py`**

```python
"""The register — the list HR actually maintains, as a spreadsheet.

This is the master for document identity: filenames are unreliable, so doc_no is
recovered by matching a file against this list. It is also the reason the
pipeline needs a matching step at all.

It is generated *imperfect on purpose*. A register maintained by hand drifts
from the drive it describes, and the drift is where the real effort goes:
  · rows whose 문서명 no longer matches the file
  · a row for a document nobody ever uploaded
  · a file on the drive that the register never learned about
  · a version number the register did not keep up with
A generator that produces a clean register would make the matching step look
trivial, and it is not.
"""
from __future__ import annotations

import random
from pathlib import Path

from openpyxl import Workbook
from openpyxl.styles import Alignment, Font, PatternFill

from .org import DEPT_BY_CODE
from .regulations import Regulation

# Regulations the scenarios assert against. Register defects are realistic, but
# injecting one here would break the thing the corpus exists to demonstrate
# rather than exercise the matcher — the childcare regulation has to be findable
# and correctly versioned for the temporal-isolation test to mean anything.
PROTECTED = {"HR-REG-003", "FIN-GDL-001", "PUR-GDL-001", "SEC-GDL-001"}

HEADERS = ["문서번호", "문서명", "구분", "소관부서", "현행 개정차수",
           "시행일", "보안등급", "비고"]
KIND = {1: "규칙", 2: "규정", 3: "지침"}
SENS = {"PUBLIC": "공개", "INTERNAL": "사내한", "RESTRICTED": "대외비"}


def build(regs: list[Regulation], rng: random.Random, out: Path) -> dict:
    """Write the register and return what was deliberately broken, so the
    pipeline's matching accuracy can be scored against ground truth."""
    wb = Workbook()
    ws = wb.active
    ws.title = "규정목록"

    ws.append(HEADERS)
    head_fill = PatternFill("solid", fgColor="D9E1F2")
    for c in ws[1]:
        c.font = Font(bold=True)
        c.fill = head_fill
        c.alignment = Alignment(horizontal="center")

    defects = {"title_drift": [], "missing_from_register": [],
               "phantom_row": [], "stale_version": []}

    for reg in regs:
        cur = reg.current
        if cur is None:
            continue

        protected = reg.doc_no in PROTECTED

        # 1. A regulation the register never learned about. The file exists on
        #    the drive; nothing points at it.
        if not protected and rng.random() < 0.08:
            defects["missing_from_register"].append(reg.doc_no)
            continue

        title = reg.title
        # 2. The register's title drifted from the document's own.
        #    Forced for a fixed share of rows: leaving it to chance means a seed
        #    can produce zero of them, and the matcher then goes untested on the
        #    case it exists to handle.
        force_drift = len(defects["title_drift"]) < 3
        if not protected and (force_drift or rng.random() < 0.15):
            title = reg.title + rng.choice([" (개정)", "규정", " 운영"])
            defects["title_drift"].append(reg.doc_no)

        version = cur.version
        # 3. The register was not updated after the last revision.
        if not protected and version > 1 and rng.random() < 0.12:
            version = version - 1
            defects["stale_version"].append(reg.doc_no)

        ws.append([reg.doc_no, title, KIND[reg.tier],
                   DEPT_BY_CODE[reg.owner_dept].name, f"제{version}차",
                   cur.effective_from.strftime("%Y-%m-%d"),
                   SENS[reg.sensitivity], ""])

    # 4. A row for a document that was planned and never produced.
    for i in range(2):
        doc = f"GA-GDL-{90 + i:03d}"
        ws.append([doc, f"문서보안 점검지침 {i + 1}", "지침", "총무팀", "제1차",
                   "2024-01-01", "사내한", "제정 예정"])
        defects["phantom_row"].append(doc)

    widths = [14, 30, 8, 12, 13, 12, 10, 14]
    for i, w in enumerate(widths, start=1):
        ws.column_dimensions[ws.cell(row=1, column=i).column_letter].width = w

    out.parent.mkdir(parents=True, exist_ok=True)
    wb.save(out)
    return defects
```


### Rendering — PDF, DOCX, XLSX

**`demo/seed/src/regdemo_seed/render.py`**

```python
"""Render the catalogue to the files an organisation would actually have.

Not "one clean PDF per regulation". A shared drive accumulates a working copy
and a published copy of the same document, filenames that carry the version in
prose instead of metadata, and scans that no text extractor can read. The
pipeline has to survive that, so the corpus contains it.

Filenames are deliberately unreliable: doc_no is recovered by matching against
the regulation register, not by parsing the name.
"""
from __future__ import annotations

import io
import random
from datetime import date
from pathlib import Path

from docx import Document as DocxDocument
from docx.shared import Pt
from reportlab.lib import colors
from reportlab.lib.pagesizes import A4
from reportlab.lib.styles import ParagraphStyle
from reportlab.lib.units import mm
from reportlab.pdfbase import pdfmetrics
from reportlab.pdfbase.ttfonts import TTFont
from reportlab.platypus import (BaseDocTemplate, Frame, PageTemplate, Paragraph,
                                Spacer, Table, TableStyle)

from .org import DEPT_BY_CODE
from .regulations import Regulation, Version

FONT_CANDIDATES = [
    "/System/Library/Fonts/Supplemental/AppleGothic.ttf",
    "/usr/share/fonts/truetype/nanum/NanumGothic.ttf",
    "/usr/share/fonts/opentype/noto/NotoSansCJK-Regular.ttc",
]
FONT_NAME = "KoreanBody"


def register_font(path: str | None = None) -> str:
    for p in ([path] if path else []) + FONT_CANDIDATES:
        if p and Path(p).exists():
            pdfmetrics.registerFont(TTFont(FONT_NAME, p))
            return p
    raise RuntimeError("no Korean TTF found; set --font")


# ── Filenames ────────────────────────────────────────────────────────────────
# The register is the master; these are what people actually typed. A pipeline
# that trusts the filename picks the wrong version roughly a third of the time.
_MESSY = [
    "{t}.{ext}",
    "{t}_v{v}.{ext}",
    "{t}_v{v}_최종.{ext}",
    "{t}_v{v}_최종_수정.{ext}",
    "[최종]{t}({d}).{ext}",
    "{t} 사본.{ext}",
    "{t}_{d}.{ext}",
    "{doc}_{t}_v{v}.{ext}",     # the well-named minority
]


def filename(reg: Regulation, v: Version, ext: str, rng: random.Random) -> str:
    pat = rng.choice(_MESSY)
    return pat.format(t=reg.title, v=v.version, doc=reg.doc_no,
                      d=v.stated_from.strftime("%Y.%m.%d"), ext=ext)


def s3_key(reg: Regulation, fname: str) -> str:
    """Department folders — the tidy part of the starting state."""
    return f"{DEPT_BY_CODE[reg.owner_dept].name}/{'규정' if reg.tier < 3 else '지침'}/{fname}"


# ── PDF ──────────────────────────────────────────────────────────────────────
def _styles():
    return {
        "title": ParagraphStyle("t", fontName=FONT_NAME, fontSize=18, leading=26,
                                alignment=1, spaceAfter=18),
        "meta": ParagraphStyle("m", fontName=FONT_NAME, fontSize=9, leading=14,
                               textColor=colors.HexColor("#555555")),
        "art_h": ParagraphStyle("ah", fontName=FONT_NAME, fontSize=11, leading=18,
                                spaceBefore=10, spaceAfter=3),
        "body": ParagraphStyle("b", fontName=FONT_NAME, fontSize=9.5, leading=16,
                               spaceAfter=4),
    }


def render_pdf(reg: Regulation, v: Version, extra_body: list[str] | None = None) -> bytes:
    st = _styles()
    buf = io.BytesIO()

    def header(canvas, doc):
        # Document number and version live in the running header, which is where
        # a scanned copy still carries them even when the filename does not.
        canvas.saveState()
        canvas.setFont(FONT_NAME, 7.5)
        canvas.setFillColor(colors.HexColor("#777777"))
        canvas.drawString(20 * mm, A4[1] - 12 * mm,
                          f"{reg.doc_no}  v{v.version}   시행 {v.stated_from:%Y-%m-%d}")
        canvas.drawRightString(A4[0] - 20 * mm, A4[1] - 12 * mm,
                               DEPT_BY_CODE[reg.owner_dept].name)
        canvas.drawCentredString(A4[0] / 2, 12 * mm, f"- {doc.page} -")
        canvas.restoreState()

    tmpl = BaseDocTemplate(buf, pagesize=A4, title=reg.title,
                           leftMargin=20 * mm, rightMargin=20 * mm,
                           topMargin=22 * mm, bottomMargin=20 * mm)
    tmpl.addPageTemplates([PageTemplate(
        id="p", frames=[Frame(20 * mm, 20 * mm, A4[0] - 40 * mm, A4[1] - 42 * mm)],
        onPage=header)])

    flow = [Paragraph(reg.title, st["title"])]

    rows = [["문서번호", reg.doc_no], ["개정차수", f"제{v.version}차 개정"],
            ["시행일", f"{v.stated_from:%Y년 %m월 %d일}"],
            ["소관부서", DEPT_BY_CODE[reg.owner_dept].name],
            ["보안등급", {"PUBLIC": "공개", "INTERNAL": "사내한", "RESTRICTED": "대외비"}[reg.sensitivity]]]
    t = Table(rows, colWidths=[28 * mm, 90 * mm])
    t.setStyle(TableStyle([
        ("FONTNAME", (0, 0), (-1, -1), FONT_NAME), ("FONTSIZE", (0, 0), (-1, -1), 8.5),
        ("GRID", (0, 0), (-1, -1), 0.4, colors.HexColor("#cccccc")),
        ("BACKGROUND", (0, 0), (0, -1), colors.HexColor("#f2f2f2")),
        ("VALIGN", (0, 0), (-1, -1), "MIDDLE"), ("TOPPADDING", (0, 0), (-1, -1), 4),
        ("BOTTOMPADDING", (0, 0), (-1, -1), 4)]))
    flow += [t, Spacer(1, 12)]

    for a in v.articles:
        flow.append(Paragraph(f"제{a.no}조{a.title}", st["art_h"]))
        flow.append(Paragraph(a.body, st["body"]))
    for extra in (extra_body or []):
        flow.append(Paragraph(extra, st["body"]))

    # The supplementary provision — where a document states its own effective date,
    # and where a
    # supersession is written in prose rather than as a link.
    flow += [Spacer(1, 14), Paragraph("부      칙", st["art_h"])]
    flow.append(Paragraph(
        f"① 이 규정은 {v.stated_from:%Y년 %m월 %d일}부터 시행한다.", st["body"]))
    if v.version > 1:
        flow.append(Paragraph(
            f"② 종전의 「{reg.title}」(제{v.version - 1}차 개정)은 이를 폐지한다.", st["body"]))

    tmpl.build(flow)
    return buf.getvalue()


def render_docx(reg: Regulation, v: Version) -> bytes:
    """The working copy. Same content, different bytes — which is exactly why
    de-duplication cannot rely on a content hash alone."""
    d = DocxDocument()
    style = d.styles["Normal"]
    style.font.name = "맑은 고딕"
    style.font.size = Pt(10)

    d.add_heading(reg.title, level=0)
    p = d.add_paragraph()
    p.add_run(f"{reg.doc_no}  ·  제{v.version}차 개정  ·  "
              f"시행 {v.stated_from:%Y-%m-%d}  ·  "
              f"{DEPT_BY_CODE[reg.owner_dept].name}").italic = True

    for a in v.articles:
        d.add_heading(f"제{a.no}조{a.title}", level=2)
        d.add_paragraph(a.body)

    d.add_heading("부칙", level=2)
    d.add_paragraph(f"① 이 규정은 {v.stated_from:%Y년 %m월 %d일}부터 시행한다.")
    if v.version > 1:
        d.add_paragraph(f"② 종전의 「{reg.title}」(제{v.version - 1}차 개정)은 이를 폐지한다.")

    buf = io.BytesIO()
    d.save(buf)
    return buf.getvalue()


def render_scanned_pdf(reg: Regulation, v: Version) -> bytes:
    """A page image with no extractable text.

    Present so the pipeline has to *report* what it could not read. A corpus
    where every file parses hides the one behaviour that matters here: a
    document that silently never made it into the index.
    """
    from reportlab.pdfgen import canvas as pdfcanvas
    buf = io.BytesIO()
    c = pdfcanvas.Canvas(buf, pagesize=A4)
    # Draw the text as vector paths (no font glyphs to extract) plus scan noise.
    c.setFillColorRGB(0.93, 0.92, 0.90)
    c.rect(0, 0, A4[0], A4[1], stroke=0, fill=1)
    rng = random.Random(hash(reg.doc_no) & 0xFFFF)
    for _ in range(1400):
        x, y = rng.uniform(20 * mm, A4[0] - 20 * mm), rng.uniform(30 * mm, A4[1] - 30 * mm)
        c.setFillColorRGB(*(rng.uniform(0.1, 0.4),) * 3)
        c.rect(x, y, rng.uniform(1, 9), 1.5, stroke=0, fill=1)
    c.showPage()
    c.save()
    return buf.getvalue()
```


### 300 documents that are not regulations

Meeting notes, announcements, memos. They exist so that the demo can show search
*not* citing them.

**`demo/seed/src/regdemo_seed/general.py`**

```python
"""The 300 documents that are not regulations.

These are the point of the corpus, not filler. A mid-sized company holds a few
dozen governed regulations and hundreds of meeting notes, reports and decks —
and plenty of those mention 휴가, 출장비 or 보안 in passing. An index that treats
every document as an equally valid source answers "휴가 며칠?" from a meeting
note, which is the failure the 공식 문서 flag exists to prevent.

So they are written to be *plausibly relevant and wrong*: same vocabulary, no
authority.
"""
from __future__ import annotations

import random
from dataclasses import dataclass
from datetime import date, timedelta

from .org import DEPTS, DEPT_BY_CODE, Employee, TODAY

KINDS = [
    ("회의록",   "{dept} {ym} 정기회의 회의록"),
    ("주간보고", "{dept} 주간업무보고 ({ym} {w}주차)"),
    ("기획서",   "{topic} 추진 기획(안)"),
    ("보고서",   "{topic} 결과보고"),
    ("매뉴얼",   "{topic} 업무 매뉴얼"),
    ("공지",     "[안내] {topic}"),
    ("교육자료", "{topic} 교육자료"),
]

TOPICS = ["신규 채용 프로세스", "연말 정산", "재택근무 시범운영", "고객사 방문",
          "보안 점검", "품질 개선 활동", "예산 집행", "협력사 평가",
          "시스템 이관", "내부감사 대응", "복리후생 개선", "출장 절차 간소화"]

# Sentences that reuse regulation vocabulary without carrying any authority.
# This is what makes the retrieval test honest.
DECOYS = [
    "휴가 사용과 관련하여 팀별 편차가 크다는 의견이 있었다. 인사팀 확인이 필요하다.",
    "육아휴직 관련 문의가 늘고 있어 담당자가 안내 중이다. 정확한 일수는 규정을 따른다.",
    "출장비 정산이 지연되는 사례가 있어 재무팀에 확인을 요청하였다.",
    "숙박비 상한을 두고 이견이 있었으나 별도 지침에 따르기로 하였다.",
    "노트북 반출 절차가 번거롭다는 의견이 접수되었다.",
    "구매 승인 단계가 많아 리드타임이 길어진다는 지적이 있었다.",
    "연차 소진율이 낮아 권장 사용 캠페인을 검토하기로 하였다.",
]

FILLER = [
    "지난 회차 논의사항에 대한 후속 조치를 점검하였다.",
    "담당자는 다음 회의까지 세부 실행안을 준비하기로 하였다.",
    "일정은 관련 부서와 협의 후 확정하기로 한다.",
    "예산 범위 내에서 우선순위를 조정하기로 하였다.",
    "특이사항 없음.",
]


@dataclass
class GeneralDoc:
    title: str
    kind: str
    dept: str
    author_emp_no: str
    author_name: str
    created: date
    paragraphs: list[str]
    has_pii: bool
    fmt: str            # pdf | docx


def build(rng: random.Random, employees: list[Employee], n: int = 300) -> list[GeneralDoc]:
    by_dept: dict[str, list[Employee]] = {}
    for e in employees:
        by_dept.setdefault(e.dept, []).append(e)

    docs: list[GeneralDoc] = []
    # Weight by headcount so the larger departments produce more paper — which also
    # means the noise is not uniformly distributed across the search space.
    weights = [d.headcount for d in DEPTS]
    for _ in range(n):
        dept = rng.choices(DEPTS, weights=weights)[0]
        author = rng.choice(by_dept[dept.code])
        kind, pat = rng.choice(KINDS)
        created = TODAY - timedelta(days=rng.randint(1, 1500))
        topic = rng.choice(TOPICS)
        title = pat.format(dept=dept.name, topic=topic,
                           ym=f"{created:%Y년 %m월}", w=rng.randint(1, 4))

        paras = [rng.choice(FILLER) for _ in range(rng.randint(2, 4))]
        # Roughly a third mention regulation vocabulary. These are the documents
        # a naive retriever surfaces for a question about leave.
        if rng.random() < 0.35:
            paras.insert(rng.randint(0, len(paras)), rng.choice(DECOYS))

        # A minority carry personal data in free text — names with employee
        # numbers, contact details. Column masking cannot redact prose, so these
        # drive the text/text_redacted split.
        has_pii = rng.random() < 0.12
        if has_pii:
            victim = rng.choice(by_dept[dept.code])
            paras.append(
                f"담당: {victim.name}({victim.emp_no}), 연락처 010-{rng.randint(1000,9999)}-"
                f"{rng.randint(1000,9999)}, 주민등록번호 {rng.randint(70,99)}"
                f"{rng.randint(1,12):02d}{rng.randint(1,28):02d}-{rng.randint(1,2)}"
                f"{rng.randint(100000,999999)}")

        docs.append(GeneralDoc(title, kind, dept.code, author.emp_no, author.name,
                               created, paras, has_pii,
                               "docx" if rng.random() < 0.4 else "pdf"))
    return docs
```


### ERP

**`demo/seed/src/regdemo_seed/erp.py`**

```python
"""ERP rows — PostgreSQL, federated live rather than copied into the ledger.

Leave balances have to be current, so this is queried through a JDBC catalog at
question time instead of being replicated. Column names are deliberately the
kind an ERP actually has (lv_typ_cd, emp_sts_cd): a semantic view has to make
them legible before an agent can use them, which is the point of having one.

The numbers here are chosen so that reading the wrong regulation version yields
a wrong *answer*, not just a wrong citation — see DEMO_EMPLOYEE.
"""
from __future__ import annotations

import random
from datetime import date, timedelta

from .org import DEPTS, DEPT_BY_CODE, Employee, TODAY
from .regulations import LEAVE_DAYS_NEW, LEAVE_DAYS_OLD

# The employee the childcare scenario asks about. 12 days used against the
# current 20-day entitlement leaves 8; against the superseded 15-day figure it
# would leave 3. One question, two defensible-looking answers, only one correct.
DEMO_EMPLOYEE_USED = 12

LEAVE_TYPES = [("ANN", "연차"), ("CHC", "육아휴직"), ("SIC", "병가"), ("CON", "경조사")]
STATUS_CODE = {"재직": "A", "휴직": "L", "퇴직": "T"}
GRADE_CODE = {"사원": "G1", "대리": "G2", "과장": "G3", "차장": "G4", "부장": "G5", "이사": "G6"}

# Purchase approval ceilings, mirroring PUR-GDL-001. Kept here as data so the
# ERP and the regulation text cannot drift.
APPROVAL_LIMIT = {"G1": 0, "G2": 1_000_000, "G3": 5_000_000,
                  "G4": 20_000_000, "G5": 50_000_000, "G6": 300_000_000}


def _sql_str(v) -> str:
    if v is None:
        return "NULL"
    if isinstance(v, (int, float)):
        return str(v)
    if isinstance(v, date):
        return f"'{v.isoformat()}'"
    return "'" + str(v).replace("'", "''") + "'"


def _insert(table: str, cols: list[str], rows: list[tuple]) -> str:
    if not rows:
        return ""
    head = f"INSERT INTO {table} ({', '.join(cols)}) VALUES\n"
    body = ",\n".join("  (" + ", ".join(_sql_str(v) for v in r) + ")" for r in rows)
    return head + body + ";\n\n"


def build_sql(employees: list[Employee], rng: random.Random) -> tuple[str, dict]:
    """Return the init SQL and the facts the scenarios assert against."""
    out = ["-- generated by regdemo_seed; do not edit\n"]

    out.append("""CREATE TABLE hr_org (
  dept_cd     VARCHAR(8) PRIMARY KEY,
  dept_nm     VARCHAR(40) NOT NULL,
  up_dept_cd  VARCHAR(8),
  hdcnt       INT
);

CREATE TABLE hr_employee (
  emp_no      VARCHAR(12) PRIMARY KEY,
  emp_nm      VARCHAR(40) NOT NULL,
  dept_cd     VARCHAR(8)  NOT NULL,
  grd_cd      VARCHAR(4)  NOT NULL,
  hire_dt     DATE        NOT NULL,
  emp_sts_cd  CHAR(1)     NOT NULL,
  mobile_no   VARCHAR(20),
  rrn         VARCHAR(14)          -- 주민등록번호: Deny, never Mask
);

CREATE TABLE hr_leave_balance (
  emp_no      VARCHAR(12) NOT NULL,
  yr          INT         NOT NULL,
  lv_typ_cd   VARCHAR(4)  NOT NULL,
  grant_days  NUMERIC(5,1) NOT NULL,
  used_days   NUMERIC(5,1) NOT NULL,
  PRIMARY KEY (emp_no, yr, lv_typ_cd)
);

CREATE TABLE hr_leave_request (
  req_id      VARCHAR(16) PRIMARY KEY,
  emp_no      VARCHAR(12) NOT NULL,
  lv_typ_cd   VARCHAR(4)  NOT NULL,
  fr_dt       DATE NOT NULL,
  to_dt       DATE NOT NULL,
  days        NUMERIC(5,1) NOT NULL,
  sts_cd      VARCHAR(4) NOT NULL    -- APRV / RJCT / WAIT
);

CREATE TABLE fi_expense (
  exp_id      VARCHAR(16) PRIMARY KEY,
  emp_no      VARCHAR(12) NOT NULL,
  exp_dt      DATE NOT NULL,
  exp_typ_cd  VARCHAR(6) NOT NULL,   -- LODG / TRNS / MEAL
  amt         BIGINT NOT NULL,
  nights      INT,
  sts_cd      VARCHAR(4) NOT NULL
);

CREATE TABLE pu_purchase_order (
  po_id       VARCHAR(16) PRIMARY KEY,
  req_emp_no  VARCHAR(12) NOT NULL,
  apr_emp_no  VARCHAR(12),
  po_dt       DATE NOT NULL,
  amt         BIGINT NOT NULL,
  sts_cd      VARCHAR(4) NOT NULL
);
""")

    out.append(_insert("hr_org", ["dept_cd", "dept_nm", "up_dept_cd", "hdcnt"],
                       [(d.code, d.name, d.parent, d.headcount) for d in DEPTS]))

    emp_rows = []
    for e in employees:
        rrn = (f"{rng.randint(70, 99)}{rng.randint(1,12):02d}{rng.randint(1,28):02d}-"
               f"{rng.randint(1,2)}{rng.randint(100000,999999)}")
        emp_rows.append((e.emp_no, e.name, e.dept, GRADE_CODE[e.grade], e.hired_on,
                         STATUS_CODE[e.status],
                         f"010-{rng.randint(1000,9999)}-{rng.randint(1000,9999)}", rrn))
    out.append(_insert("hr_employee",
                       ["emp_no", "emp_nm", "dept_cd", "grd_cd", "hire_dt",
                        "emp_sts_cd", "mobile_no", "rrn"], emp_rows))

    # ── Leave balances ───────────────────────────────────────────────────────
    active = [e for e in employees if e.status == "재직"]
    demo = next(e for e in active if e.dept == "DEV" and e.years_of_service >= 4)

    bal_rows, req_rows = [], []
    yr = TODAY.year
    for e in active:
        ann_grant = 15 + max(0, (e.years_of_service - 1) // 2)
        bal_rows.append((e.emp_no, yr, "ANN", ann_grant,
                         min(ann_grant, rng.randint(0, ann_grant))))
        if e.emp_no == demo.emp_no:
            used = DEMO_EMPLOYEE_USED
        elif rng.random() < 0.18:
            used = rng.randint(0, LEAVE_DAYS_NEW)
        else:
            used = 0
        bal_rows.append((e.emp_no, yr, "CHC", LEAVE_DAYS_NEW, used))
        if used:
            for i in range(rng.randint(1, 2)):
                fr = date(yr, rng.randint(1, 7), rng.randint(1, 28))
                d = max(1, used // 2 if i == 0 else used - used // 2)
                req_rows.append((f"LV{yr}{len(req_rows):05d}", e.emp_no, "CHC",
                                 fr, fr + timedelta(days=d - 1), d, "APRV"))
    out.append(_insert("hr_leave_balance",
                       ["emp_no", "yr", "lv_typ_cd", "grant_days", "used_days"], bal_rows))
    out.append(_insert("hr_leave_request",
                       ["req_id", "emp_no", "lv_typ_cd", "fr_dt", "to_dt", "days", "sts_cd"],
                       req_rows))

    # ── Expenses: some deliberately over the lodging cap ─────────────────────
    # FIN-GDL-001 caps managers and above at 120,000 a night. Rows above that cap
    # exist so "does this claim comply?" has both compliant and non-compliant cases.
    exp_rows, violations = [], []
    for e in rng.sample(active, 120):
        for _ in range(rng.randint(1, 3)):
            nights = rng.randint(1, 3)
            over = rng.random() < 0.15
            per_night = rng.randint(130_000, 180_000) if over else rng.randint(60_000, 118_000)
            eid = f"EX{len(exp_rows):06d}"
            exp_rows.append((eid, e.emp_no, TODAY - timedelta(days=rng.randint(1, 400)),
                             "LODG", per_night * nights, nights, "APRV"))
            if over:
                violations.append(eid)
    out.append(_insert("fi_expense",
                       ["exp_id", "emp_no", "exp_dt", "exp_typ_cd", "amt", "nights", "sts_cd"],
                       exp_rows))

    # ── Purchase orders: some approved beyond the approver's ceiling ─────────
    po_rows, po_violations = [], []
    approvers = [e for e in active if e.grade in ("과장", "차장", "부장", "이사")]
    for e in rng.sample(active, 90):
        apr = rng.choice(approvers)
        limit = APPROVAL_LIMIT[GRADE_CODE[apr.grade]]
        breach = rng.random() < 0.12
        amt = (limit + rng.randint(1_000_000, 20_000_000) if breach
               else rng.randint(100_000, max(200_000, limit)))
        pid = f"PO{len(po_rows):06d}"
        po_rows.append((pid, e.emp_no, apr.emp_no,
                        TODAY - timedelta(days=rng.randint(1, 500)), amt, "APRV"))
        if breach:
            po_violations.append(pid)
    out.append(_insert("pu_purchase_order",
                       ["po_id", "req_emp_no", "apr_emp_no", "po_dt", "amt", "sts_cd"],
                       po_rows))

    facts = {
        "demo_employee": {"emp_no": demo.emp_no, "name": demo.name, "dept": demo.dept,
                          "grade": demo.grade,
                          "childcare_used": DEMO_EMPLOYEE_USED,
                          "childcare_grant_current": LEAVE_DAYS_NEW,
                          "correct_remaining": LEAVE_DAYS_NEW - DEMO_EMPLOYEE_USED,
                          "wrong_remaining_if_superseded": LEAVE_DAYS_OLD - DEMO_EMPLOYEE_USED},
        "lodging_violations": violations,
        "purchase_limit_breaches": po_violations,
        "approval_limits": APPROVAL_LIMIT,
        "counts": {"employees": len(emp_rows), "leave_balances": len(bal_rows),
                   "leave_requests": len(req_rows), "expenses": len(exp_rows),
                   "purchase_orders": len(po_rows)},
    }
    return "".join(out), facts
```


### The approval system — the authority on effective dates

**`demo/seed/src/regdemo_seed/groupware.py`**

```python
"""The approval system — MySQL. The authoritative source for when a regulation
took effect.

A document's 부칙 states an effective date written while it was still a draft.
If the approval then slips, that date is wrong and nothing in the document says
so. The approval record is what actually happened, so doc_versions.effective_from
comes from here and the document's own claim is kept beside it as stated_from.

This is also the event source: an approval completing is the moment a regulation
becomes effective, which is what the CDC flow reacts to. No Kafka is involved —
the trigger is a row changing in the system that already owns the fact.
"""
from __future__ import annotations

import random
from datetime import date, timedelta

from .org import DEPT_BY_CODE, Employee, TODAY
from .regulations import Regulation

STEPS = [("DRAFT", "기안"), ("REVIEW", "검토"), ("APPROVE", "승인")]


def _sql_str(v) -> str:
    if v is None:
        return "NULL"
    if isinstance(v, int):
        return str(v)
    if isinstance(v, date):
        return f"'{v.isoformat()}'"
    return "'" + str(v).replace("'", "''") + "'"


def _insert(table: str, cols: list[str], rows: list[tuple]) -> str:
    if not rows:
        return ""
    return (f"INSERT INTO {table} ({', '.join(cols)}) VALUES\n"
            + ",\n".join("  (" + ", ".join(_sql_str(v) for v in r) + ")" for r in rows)
            + ";\n\n")


def build_sql(regs: list[Regulation], employees: list[Employee],
              rng: random.Random) -> tuple[str, dict]:
    out = ["-- generated by regdemo_seed; do not edit\n"]
    out.append("""CREATE TABLE gw_approval (
  apr_id      VARCHAR(20) PRIMARY KEY,
  doc_no      VARCHAR(20) NOT NULL,
  ver         INT NOT NULL,
  subject     VARCHAR(200) NOT NULL,
  drafter_no  VARCHAR(12) NOT NULL,
  dept_cd     VARCHAR(8) NOT NULL,
  draft_dt    DATE NOT NULL,
  complete_dt DATE,                    -- authoritative effective date
  stated_dt   DATE,                    -- what the document body claims
  sts_cd      VARCHAR(8) NOT NULL,     -- CMPL / PROG / RJCT
  INDEX ix_doc (doc_no, ver)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE gw_approval_line (
  apr_id      VARCHAR(20) NOT NULL,
  seq         INT NOT NULL,
  step_cd     VARCHAR(10) NOT NULL,
  approver_no VARCHAR(12) NOT NULL,
  acted_dt    DATE,
  PRIMARY KEY (apr_id, seq)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
""")

    by_dept: dict[str, list[Employee]] = {}
    for e in employees:
        if e.status == "재직":
            by_dept.setdefault(e.dept, []).append(e)
    seniors = [e for e in employees if e.grade in ("부장", "이사") and e.status == "재직"]

    apr_rows, line_rows = [], []
    mismatches = []
    for reg in regs:
        pool = by_dept.get(reg.owner_dept) or by_dept["HR"]
        for v in reg.versions:
            apr_id = f"AP{v.effective_from:%Y}{len(apr_rows):05d}"
            drafter = rng.choice(pool)
            # Drafting starts before approval completes; the gap is what makes a
            # stated date go stale.
            draft_dt = v.effective_from - timedelta(days=rng.randint(7, 60))
            apr_rows.append((apr_id, reg.doc_no, v.version,
                             f"「{reg.title}」 제{v.version}차 개정(안)",
                             drafter.emp_no, reg.owner_dept, draft_dt,
                             v.effective_from, v.stated_from, "CMPL"))
            if v.date_mismatch:
                mismatches.append({"doc_no": reg.doc_no, "version": v.version,
                                   "approved": v.effective_from.isoformat(),
                                   "stated": v.stated_from.isoformat()})
            acted = draft_dt
            for seq, (code, _) in enumerate(STEPS, start=1):
                acted = (draft_dt if code == "DRAFT"
                         else v.effective_from if code == "APPROVE"
                         else draft_dt + timedelta(days=rng.randint(1, 10)))
                approver = drafter if code == "DRAFT" else rng.choice(seniors)
                line_rows.append((apr_id, seq, code, approver.emp_no, acted))

    # An in-flight revision: drafted, not yet approved. It must not become
    # effective, and an index that keys off the document body would make it so.
    pending = regs[3]
    pend_id = f"AP{TODAY:%Y}{len(apr_rows):05d}"
    apr_rows.append((pend_id, pending.doc_no,
                     (pending.current.version if pending.current else 1) + 1,
                     f"「{pending.title}」 개정(안)", rng.choice(seniors).emp_no,
                     pending.owner_dept, TODAY - timedelta(days=9), None,
                     TODAY + timedelta(days=21), "PROG"))

    out.append(_insert("gw_approval",
                       ["apr_id", "doc_no", "ver", "subject", "drafter_no", "dept_cd",
                        "draft_dt", "complete_dt", "stated_dt", "sts_cd"], apr_rows))
    out.append(_insert("gw_approval_line",
                       ["apr_id", "seq", "step_cd", "approver_no", "acted_dt"], line_rows))

    return "".join(out), {
        "counts": {"approvals": len(apr_rows), "approval_lines": len(line_rows)},
        "date_mismatches": mismatches,
        "pending_approval": {"apr_id": pend_id, "doc_no": pending.doc_no,
                             "note": "drafted, not approved — must not be treated as effective"},
    }
```


### Training records

**`demo/seed/src/regdemo_seed/training.py`**

```python
"""Training records — served over REST by a mock SaaS, not a database.

Plenty of HR systems are SaaS with an API and no database access, so one source
in the demo is reachable only through the rest-operation connector. It also
carries a compliance question the other sources cannot answer on their own:
"정보보안규정이 개정됐는데 신규정 교육을 안 받은 사람은?" needs the regulation
graph, the completion records, and the org chart together.
"""
from __future__ import annotations

import random

from .org import Employee
from .regulations import Regulation

# Regulations whose revisions require re-training.
TRAINING_REQUIRED = {"SEC-REG-001", "HR-REG-001", "SEC-GDL-002", "LEG-REG-001"}


def build(regs: list[Regulation], employees: list[Employee],
          rng: random.Random) -> tuple[dict, dict]:
    by_doc = {r.doc_no: r for r in regs}
    courses, completions = [], []
    outstanding: dict[str, list[str]] = {}

    for doc in sorted(TRAINING_REQUIRED):
        reg = by_doc.get(doc)
        if reg is None or reg.current is None:
            continue
        cur = reg.current
        course_id = f"CRS-{doc}-v{cur.version}"
        courses.append({"course_id": course_id, "doc_no": doc,
                        "doc_version": cur.version,
                        "title": f"{reg.title} 제{cur.version}차 개정 교육",
                        "required": True,
                        "opened_on": cur.effective_from.isoformat()})

        missing = []
        for e in employees:
            if e.status != "재직":
                continue
            # Most complete it; the rest are the compliance gap the scenario asks
            # about, and they cluster in the departments with the most headcount.
            if rng.random() < 0.82:
                completions.append({"course_id": course_id, "emp_no": e.emp_no,
                                    "completed_on":
                                        (cur.effective_from.toordinal() + rng.randint(1, 120)),
                                    "score": rng.randint(70, 100)})
            else:
                missing.append(e.emp_no)
        outstanding[course_id] = missing

    # Ordinal → ISO, kept out of the loop so the record shape stays obvious.
    from datetime import date as _d
    for c in completions:
        c["completed_on"] = _d.fromordinal(c["completed_on"]).isoformat()

    payload = {"courses": courses, "completions": completions}
    facts = {"counts": {"courses": len(courses), "completions": len(completions)},
             "outstanding_counts": {k: len(v) for k, v in outstanding.items()},
             "outstanding": outstanding}
    return payload, facts
```


---

## Putting the originals in object storage

This is not part of the pipeline; it is what happens *before* it. In a real
company the regulation documents already sit on a shared drive or in object
storage, and the pipeline discovers and reads them there. Here the seed writes
files locally, so this script creates the state of "already on the drive".

**`demo/infra/upload.sh`**

```bash
#!/usr/bin/env bash
##
## Put the original documents into object storage.
##
##   bash infra/upload.sh
##
## This is not part of the pipeline; it is what happens before it. In a real
## company the regulation documents already sit on a shared drive or in object
## storage, and the pipeline discovers and reads them there. Here the seed writes
## the files locally, so this script creates the state of "already on the drive".
##
## Keys are the path under out/documents/ appended to corpus/. The discover job
## walks that prefix, so the path convention is the pipeline's input scope.
##
set -euo pipefail
DEMO="$(cd "$(dirname "$0")/.." && pwd)"
source "$DEMO/out/stack.env"

PY="$DEMO/.venv/bin/python"
[ -x "$PY" ] || PY=python3

"$PY" - "$DEMO/out/documents" <<'PY'
import os, sys, pathlib
import boto3
from botocore.config import Config

root = pathlib.Path(sys.argv[1])
if not root.is_dir():
    raise SystemExit(f"{root} not found — run 'make seed' first")

s3 = boto3.client(
    "s3",
    endpoint_url=os.environ["S3_ENDPOINT_HOST"],
    aws_access_key_id=os.environ["S3_ACCESS_KEY"],
    aws_secret_access_key=os.environ["S3_SECRET_KEY"],
    region_name=os.environ.get("S3_REGION", "us-east-1"),
    config=Config(s3={"addressing_style": "path"}))

bucket = "iceberg-warehouse"
# Objects already uploaded are not sent again. The corpus is 446 files, and there
# is no reason to re-upload all of them every time the pipeline is re-run.
have = set()
token = None
while True:
    kw = {"Bucket": bucket, "Prefix": "corpus/"}
    if token:
        kw["ContinuationToken"] = token
    r = s3.list_objects_v2(**kw)
    have.update(o["Key"] for o in r.get("Contents", []))
    if not r.get("IsTruncated"):
        break
    token = r["NextContinuationToken"]

sent = skipped = 0
for p in sorted(x for x in root.rglob("*") if x.is_file()):
    key = "corpus/" + p.relative_to(root).as_posix()
    if key in have:
        skipped += 1
        continue
    s3.put_object(Bucket=bucket, Key=key, Body=p.read_bytes())
    sent += 1
    if sent % 50 == 0:
        print(f"  {sent} uploaded")

print(f"  uploaded {sent} · already there {skipped} · {sent + skipped} total  →  s3://{bucket}/corpus/")
PY
```


```bash
bash infra/upload.sh
```

```text
  업로드 446 · 이미 있음 0 · 총 446 개  →  s3://iceberg-warehouse/corpus/
```

---

Next: [the schema](schema.md) — where these documents land, and in what shape.
