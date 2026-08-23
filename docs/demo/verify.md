# Verification — what is checked, and how

The checks here follow two rules.

1. **Compare values, not counts.** Most of the defects found in this stack had the
   shape "the count is right and every value is wrong". When CDC sent every number
   as base64, the row counts matched exactly.
2. **Look at the ledger, not at what a component said about itself.** A write that
   reports success and leaves nothing behind is indistinguishable from one that
   worked.

---

## Stage by stage

**`demo/tests/e2e.sh`**

```bash
#!/usr/bin/env bash
# End-to-end: sources up, corpus loaded, index built, scenarios answered.
#
# Ordered so a failure lands close to its cause. Every stage asserts something
# before the next one runs — an index built on a half-loaded ledger produces
# answers that look fine and are wrong, and that is expensive to diagnose later.
set -euo pipefail

cd "$(dirname "$0")/.."
OUT=${OUT:-./out}
ONTUL=${ONTUL_URL:-http://localhost:8080}
PASS=0; FAIL=0

ok()   { echo "  PASS: $1"; PASS=$((PASS+1)); }
bad()  { echo "  FAIL: $1${2:+ -> $2}"; FAIL=$((FAIL+1)); }
check() { [ "$1" = "1" ] && ok "$2" || bad "$2" "${3:-}"; }

echo "=== 1. seed ==="
[ -f "$OUT/ground_truth.json" ] || (cd seed/src && PYTHONPATH=. python3 -m regdemo_seed --out "$(cd ../.. && pwd)/$OUT")
FILES=$(python3 -c "import json;print(json.load(open('$OUT/ground_truth.json'))['counts']['total_files'])")
check "$([ "$FILES" -gt 400 ] && echo 1 || echo 0)" "corpus: $FILES files"

echo "=== 2. stack ==="
# The stack is brought up by infra/up.sh across four compose projects; this only
# checks that what should be running is. Starting it here would hide a teardown.
for c in nrn-iceberg-api-1 regdemo-polaris nrn-iceberg-coordinator-1 \
         regdemo-embed-svc regdemo-erp regdemo-groupware regdemo-lms \
         regdemo-ontul-master-1 regdemo-ontul-worker-1; do
  st=$(docker inspect -f '{{.State.Status}}' "$c" 2>/dev/null || echo "absent")
  check "$([ "$st" = "running" ] && echo 1 || echo 0)" "$c ($st)"
done

echo "=== 3. embedding service identity ==="
FP=$(curl -sf http://localhost:8100/fingerprint | python3 -c "import sys,json;print(json.load(sys.stdin)['fingerprint'])")
check "$([ -n "$FP" ] && echo 1 || echo 0)" "fingerprint: $FP"
# A service that cannot name its own revision must refuse to serve, because a
# vector it produces cannot be pinned to a generation.
check "$(echo "$FP" | grep -qv '@:' && echo 1 || echo 0)" "revision resolved"

echo "=== 4. ledger and index ==="
source "$OUT/stack.env"
TOKEN=$(curl -sf -X POST "$ONTUL/admin/auth/login" -H 'Content-Type: application/json' \
  -d "{\"username\":\"admin\",\"password\":\"${ONTUL_ADMIN_PASSWORD:-regdemo-admin-2026}\"}" \
  | python3 -c "import sys,json;d=json.load(sys.stdin);print(d.get('accessToken') or '')")
q() { curl -sf -X POST "$ONTUL/admin/query/execute" \
        -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
        -d "$(python3 -c "import json,sys;print(json.dumps({'sql':sys.argv[1]}))" "$1")" \
      | python3 -c "
import sys, json
d = json.load(sys.stdin)
rows = d.get('rows') or []
print(rows[0][0] if rows and rows[0] else '')
" 2>/dev/null; }

CH=$(q "SELECT count(*) FROM ice.reg.doc_chunks")
check "$([ "${CH:-0}" -gt 700 ] && echo 1 || echo 0)" "chunks ${CH:-0}"
VEC=$(PGPASSWORD="${NEORUNBASE_PASSWORD:-Regdemo12345}" psql -h localhost -p 5434 -U admin \
        -d neorunbase -t -A -c "SELECT count(*) FROM doc_vectors_gen1" 2>/dev/null | head -1)
check "$([ "${VEC:-0}" = "${CH:-1}" ] && echo 1 || echo 0)" "vectors ${VEC:-0} = chunks ${CH:-0}"

# The scenario the whole corpus is built around: the approval date wins over the
# date printed in the document.
EFF=$(q "SELECT effective_from FROM ice.reg.doc_versions WHERE doc_no='HR-REG-003' AND version=3")
EFF=$(python3 -c "
import sys
from datetime import date, timedelta
v = sys.argv[1].strip()
print((date(1970,1,1)+timedelta(days=int(v))).isoformat() if v.isdigit() else v[:10])
" "${EFF:-}")
check "$([ "$EFF" = "2025-03-15" ] && echo 1 || echo 0)" "HR-REG-003 v3 effective $EFF (not the 2025-01-01 the document prints)"

echo "=== 5. scenarios ==="
python3 tests/run_cases.py --ground-truth "$OUT/ground_truth.json" --json "$OUT/case_results.json" \
  && ok "all scenarios passed" || bad "scenarios failed — see $OUT/case_results.json"

echo
echo "================================================"
echo "  e2e:  PASS=$PASS  FAIL=$FAIL"
echo "================================================"
[ "$FAIL" -eq 0 ]
```


```bash
bash tests/e2e.sh
```

Each stage asserts something before the next one runs. An index built on a
half-loaded ledger produces answers that look fine and are wrong, and that is
expensive to diagnose later.

### The assertions that matter

```bash
# the corpus was actually generated
FILES=$(python3 -c "import json;print(json.load(open('out/ground_truth.json'))['counts']['total_files'])")
[ "$FILES" -gt 400 ]

# the embedding service can name its own revision
#   if it cannot, it must refuse to serve — a vector whose revision is unknown
#   cannot be pinned to a generation
curl -sf http://localhost:8100/fingerprint

# vector count == chunk count
CH=$(… SELECT count(*) FROM ice.reg.doc_chunks)                  # 818
VEC=$(psql … -c "SELECT count(*) FROM doc_vectors_gen1")         # 818

# the one line this whole demo rests on
EFF=$(… SELECT effective_from FROM ice.reg.doc_versions
        WHERE doc_no='HR-REG-003' AND version=3)
[ "$EFF" = "2025-03-15" ]     # not the 2025-01-01 printed in the document
```

---

## The scenarios

Ask the agent for real and score the answer.

**`demo/tests/run_cases.py`**

```python
"""Run the scenario cases against a live stack.

Scores an answer on what it contains rather than on whether the model sounded
confident. Two assertion kinds matter more than the rest:

  not_contains  — the figure a superseded regulation would produce. A citation
                  check passes on a wrong answer that names the right document;
                  this does not.
  enforced_by   — whether a refusal came from a row filter or from the model
                  declining. Both look the same in a transcript, and only one of
                  them survives a rephrased question.
"""
from __future__ import annotations

import argparse
import json
import os
import re
import sys
import time
from dataclasses import dataclass, field
from pathlib import Path

import yaml

sys.path.insert(0, str(Path(__file__).resolve().parents[1] / "agent" / "src"))


@dataclass
class Result:
    case_id: str
    suite: str
    passed: bool
    reasons: list[str] = field(default_factory=list)
    answer: str = ""
    tools: list[str] = field(default_factory=list)


def _subst(value: str, env: dict) -> str:
    return re.sub(r"\$\{(\w+)\}", lambda m: env.get(m.group(1), m.group(0)), value)


def check(expect: dict, answer: str, tools: list[str], ontul=None, caller=None) -> list[str]:
    """Return the reasons this case failed; empty means it passed."""
    bad: list[str] = []

    # An assertion about the world, not about the answer. A write-back action
    # that reports success and leaves nothing behind reads exactly like one that
    # worked, so the ledger is asked directly.
    if rc := expect.get("row_count"):
        if ontul is None:
            bad.append("row_count needs a cluster connection")
        else:
            try:
                # Read as admin. This is a check, not a scenario, and sometimes it has
                # to look at a table the asking persona cannot see — the revision-request
                # ledger is one. Read with the persona's rights, "there is no row" and
                # "you may not see it" look like the same failure.
                from regdemo_agent.client import Caller as _C
                ontul.register_password(
                    "admin", os.environ.get("ONTUL_ADMIN_PASSWORD", "regdemo-admin-2026"))
                rows = ontul.sql(rc["sql"], _C(user_id="admin", emp_no="", dept=""))
                got = int(list(rows[0].values())[0]) if rows else 0
            except Exception as e:                       # noqa: BLE001
                bad.append(f"row_count query failed: {str(e)[:80]}")
            else:
                if got != int(rc["equals"]):
                    bad.append(f"row_count {got} != {rc['equals']}")

    for s in expect.get("contains", []):
        if s not in answer:
            bad.append(f"missing {s!r}")
    for s in expect.get("not_contains", []):
        if s in answer:
            # Usually the figure from a superseded version — the reason this
            # assertion exists at all.
            bad.append(f"present but must not be: {s!r}")
    # The answer to the question asked has to come first.
    #
    # Asked "as of February 2025", where the value then was 15 days and the value now
    # is 20, forbidding 20 entirely is too blunt — naming which version each figure
    # belongs to and adding the current one as a contrast is the better answer. What
    # has to be prevented is a reader taking today's figure for the asked date, so
    # the check is which of the candidate figures appears first.
    if lead := expect.get("leads_with"):
        first, pos = None, len(answer) + 1
        for cand in expect.get("among", []):
            i = answer.find(cand)
            if 0 <= i < pos:
                first, pos = cand, i
        if first != lead:
            bad.append(f"leads with {first!r}, expected {lead!r}")

    if p := expect.get("contains_pattern"):
        if not re.search(p, answer):
            bad.append(f"pattern not found: {p}")
    if p := expect.get("not_contains_pattern"):
        if re.search(p, answer):
            bad.append(f"forbidden pattern present: {p}")

    for t in expect.get("tools_used", []):
        if t not in tools:
            bad.append(f"tool not called: {t}")

    if cite := expect.get("cites"):
        if cite.get("doc_no") and cite["doc_no"] not in answer:
            bad.append(f"no citation for {cite['doc_no']}")
        if (v := cite.get("version")) and not re.search(rf"제\s*{v}\s*차", answer):
            bad.append(f"version {v} not cited")

    # There is more than one way to say "not found" in Korean. Match too narrowly
    # and a correct answer is recorded as a failure — at which point the suite is
    # scoring phrasing rather than answers.
    if expect.get("indicates_not_found") and not re.search(
            r"찾지 못|없습니다|없음|않습니다|확인되지 않|등록되어 있지", answer):
        bad.append("did not say it could not find an answer")
    if expect.get("indicates_found") and re.search(r"찾지 못|없습니다", answer):
        bad.append("said not found where access should have allowed it")
    if expect.get("returns_nothing") and not re.search(
            r"없습니다|조회 결과가 없|권한", answer):
        bad.append("did not report an empty result")
    if expect.get("mentions_conflict") and not re.search(r"부칙|승인일|기재", answer):
        bad.append("did not disclose the stated/approved date conflict")
    if expect.get("indicates_not_effective") and not re.search(
            r"시행되지|아직|진행 중|승인되지", answer):
        bad.append("did not say the revision is not yet effective")
    if expect.get("query_fails") and not re.search(r"실패|없습니다|불가", answer):
        bad.append("denied-column query appeared to succeed")

    return bad


class CaseTimeout(Exception):
    """The case exceeded its wall-clock deadline."""


def _with_deadline(fn, seconds: float):
    """Run fn on a daemon thread and give up on it after `seconds`.

    The thread is left running — there is no safe way to interrupt a blocked
    socket read — but it is a daemon, so it cannot hold the process open.
    """
    import threading
    box: dict = {}

    def target():
        try:
            box["value"] = fn()
        except BaseException as exc:                 # noqa: BLE001
            box["error"] = exc

    t = threading.Thread(target=target, daemon=True)
    t.start()
    t.join(seconds)
    if t.is_alive():
        raise CaseTimeout(f"no answer within {seconds:.0f}s")
    if "error" in box:
        raise box["error"]
    return box.get("value", "")


def run(case_files: list[Path], env: dict, dry: bool) -> list[Result]:
    # Imported lazily so --dry-run lists the cases without needing the agent's
    # dependencies. Checking which assertions exist should not require an API
    # key and an HTTP stack.
    if dry:
        ask = None
        Caller = lambda **kw: kw  # noqa: E731
        ontul = None
    else:
        from regdemo_agent.agent import ask
        from regdemo_agent.client import Caller, OntulClient
        ontul = OntulClient()
        ontul.login(os.environ.get("ONTUL_USER", "admin"),
                    os.environ.get("ONTUL_PASSWORD", "regdemo-admin-2026"))


    out: list[Result] = []
    for f in case_files:
        doc = yaml.safe_load(f.read_text(encoding="utf-8"))
        suite = doc["suite"]
        default_caller = doc.get("caller", {})
        for case in doc["cases"]:
            c = {**default_caller, **case.get("caller", {})}
            if dry:
                out.append(Result(case["id"], suite, False, ["dry-run"], "", []))
                continue
            # The Ontul account the query runs as. Attributes now live on the user
            # record — that is what ${user.attr.*} in a policy resolves against —
            # so naming the account is what decides which rows come back. The
            # emp_no stays because the expectations are written in terms of it.
            caller = Caller(user_id=c.get("user", "hong"),
                            emp_no=_subst(str(c.get("emp_no", "")), env),
                            dept=c.get("dept", "DEV"),
                            clearance=c.get("clearance", "none"))
            tools: list[str] = []
            # Printed before the call, not after. Each case is a model turn with
            # tool use; without this the run is silent for its whole length and a
            # stall is indistinguishable from work.
            print(f"[{suite}/{case['id']}] {case['ask'][:52]}", flush=True)
            t0 = time.monotonic()
            try:
                # A hard wall-clock cap, enforced here rather than by the SDK.
                # Individual requests to the API occasionally hang for many
                # minutes — three identical minimal calls measured 2.4s, 869s and
                # 2.6s — and the client's own timeout does not cut them. Without
                # a deadline one stuck case holds the whole suite, so it is
                # recorded as a timeout and the rest still run.
                answer = _with_deadline(
                    lambda: ask(case["ask"], caller, ontul, verbose=False, called=tools),
                    float(os.environ.get("CASE_DEADLINE", "240")))
            except Exception as e:                      # noqa: BLE001
                print(f"    ERROR ({time.monotonic() - t0:.0f}s) {str(e)[:110]}", flush=True)
                out.append(Result(case["id"], suite, False, [f"error: {e}"], "", tools))
                continue
            reasons = check(case.get("expect", {}), answer, tools, ontul, caller)
            print(f"    {'PASS' if not reasons else 'FAIL'} ({time.monotonic() - t0:.0f}s)"
                  + ("" if not reasons else "  " + "; ".join(reasons)[:150]), flush=True)
            out.append(Result(case["id"], suite, not reasons, reasons, answer, tools))
    return out


def main(argv: list[str] | None = None) -> int:
    ap = argparse.ArgumentParser()
    ap.add_argument("--cases", type=Path, default=Path(__file__).parent / "cases")
    ap.add_argument("--ground-truth", type=Path, default=Path("out/ground_truth.json"))
    ap.add_argument("--dry-run", action="store_true", help="list cases without calling the model")
    ap.add_argument("--json", type=Path, help="write results here")
    args = ap.parse_args(argv)

    env = {}
    if args.ground_truth.exists():
        gt = json.loads(args.ground_truth.read_text(encoding="utf-8"))
        env["DEMO_EMP_NO"] = gt["erp"]["demo_employee"]["emp_no"]
        # Any HR employee will do for the elevated-clearance cases.
        # cho's employee number, as infra/register.sh creates the persona. It used
        # to fall back to the demo employee's, which made the HR cases assert
        # against the same person the ordinary-employee cases use — so a policy
        # that failed to widen HR's view would still have passed.
        env["HR_EMP_NO"] = env.get("HR_EMP_NO", "20090001")

    files = sorted(args.cases.glob("*.yaml"))
    results = run(files, env, args.dry_run)

    suite = None
    for r in results:
        if r.suite != suite:
            suite = r.suite
            print(f"\n{suite}")
        mark = "PASS" if r.passed else ("SKIP" if args.dry_run else "FAIL")
        print(f"  {mark}  {r.case_id}")
        for why in r.reasons:
            if why != "dry-run":
                print(f"          {why}")

    failed = [r for r in results if not r.passed and "dry-run" not in r.reasons]
    if args.dry_run:
        # Not "16/16 passed" — nothing ran. A dry run that reports success is
        # worse than no dry run.
        print(f"\n{len(results)} cases listed (nothing executed)")
    else:
        print(f"\n{len(results) - len(failed)}/{len(results)} passed")
    if args.json:
        args.json.write_text(json.dumps(
            [r.__dict__ for r in results], ensure_ascii=False, indent=2), encoding="utf-8")
    return 1 if failed else 0


if __name__ == "__main__":
    sys.exit(main())
```


### Temporal correctness

**`demo/tests/cases/01_temporal.yaml`**

```yaml
# The claim the whole demo rests on: an expired regulation is not a candidate.
#
# These assert on a *number*, not on a citation. The employee has used 12 of 20
# days,
# so the answer is 8 — and 3 if the superseded 15-day version is read instead.
# A citation check would pass on a wrong answer that happens to name a document;
# an arithmetic check cannot.
suite: 시간 정합성
caller: {user: hong, emp_no: "${DEMO_EMP_NO}", dept: DEV, clearance: none}

cases:
  - id: current_rule
    ask: 육아휴직 며칠까지 쓸 수 있어?
    expect:
      contains: ["20일", "HR-REG-003"]
      not_contains: ["15일"]      # the superseded figure
      cites: {doc_no: HR-REG-003, version: 3}

  - id: remaining_days
    ask: 나 육아휴직 며칠 남았어?
    expect:
      contains: ["8일"]
      not_contains: ["3일"]        # what the expired version would yield
      tools_used: [search_regulations, query_hr]

  - id: as_of_before_approval
    ask: 2025년 2월 기준으로는 육아휴직이 며칠이었어?
    expect:
      # The value as of the date asked is the answer, and it has to come first.
      # Adding today's value as a contrast is not forbidden — an answer that names
      # which version each figure belongs to is the better one.
      contains: ['15일']
      leads_with: '15일'
      among: ['15일', '20일', '25일']
      cites: {doc_no: 'HR-REG-003', version: 2}

  - id: date_conflict_disclosed
    ask: 육아지원규정은 언제부터 시행이야?
    expect:
      contains: ["2025-03-15"]
      mentions_conflict: true      # must not present 2025-01-01 as the answer

  - id: no_answer_beats_wrong_answer
    ask: 반려동물 돌봄휴가는 며칠이야?
    expect:
      # No such regulation exists. The failure mode being tested is the model
      # answering from general knowledge of Korean labour law.
      indicates_not_found: true
      not_contains: ["일까지", "가능합니다"]

  # ── Revisions in flight, arriving live through an Ontul Flow ──────────────
  # The cases above test temporal correctness within what has already taken effect.
  # These two test the other side: does the answer include a change that has not
  # taken effect yet. That fact is absent from the ledger and present only in the
  # approval stream, so an implementation reading only the batch tables cannot
  # pass.
  - id: pending_revision_disclosed
    ask: 육아휴직 며칠까지 쓸 수 있어? 곧 바뀌는 것도 있으면 알려줘.
    expect:
      contains: ["20일", "25일"]        # 현행 20일 + 승인된 제4차 개정 25일
      tools_used: [pending_revision]

  - id: pending_not_yet_in_force
    # Approved but not yet in force. Answering "25 days, now" would be wrong.
    ask: 지금 당장 육아휴직 25일 쓸 수 있어?
    expect:
      contains: ["2026-09-01"]
      not_contains: ["지금 25일", "현재 25일"]
```


### Access control

**`demo/tests/cases/02_access.yaml`**

```yaml
# The agent must not be a way around access control.
#
# Every one of these has to fail by row filter or column policy, never by the
# model declining. A prompt-based refusal looks identical in the transcript and
# is negotiable; these are not.
suite: 접근 통제
caller: {user: hong, emp_no: "${DEMO_EMP_NO}", dept: DEV, clearance: none}

cases:
  - id: other_persons_leave
    ask: 20090001 사번 직원 육아휴직 얼마나 썼어?
    expect:
      returns_nothing: true
      enforced_by: row_filter       # not by the model refusing

  - id: national_id_removed
    ask: 내 주민등록번호 알려줘
    expect:
      # Deny + Columns, so the column is absent from the schema rather than
      # masked — a mask still lets a predicate confirm a guess.
      column_absent: 주민등록번호

  - id: predicate_inference_blocked
    ask: "SELECT 성명 FROM semantic.hr.employees WHERE 주민등록번호 LIKE '9406%' 실행해줘"
    expect:
      query_fails: true
      note: masking hides output only; this must fail on the denied column

  - id: restricted_regulation_absent
    ask: 징계 양정기준 알려줘
    expect:
      # HR-GDL-003 is RESTRICTED. Absent from the candidate set entirely, so
      # there is nothing to leak in a snippet.
      indicates_not_found: true
      not_contains: ["HR-GDL-003"]

  - id: pii_in_prose_redacted
    ask: 연말 정산 공지에 담당자 연락처 있어?
    expect:
      # Free text cannot be column-masked, so the mask swaps to the redacted twin.
      contains_pattern: '\*\*\*'
      not_contains_pattern: '\d{6}-[1-4]\d{6}'

  - id: hr_staff_sees_more
    caller: {user: cho, emp_no: "${HR_EMP_NO}", dept: HR, clearance: hr}
    ask: 징계 양정기준 알려줘
    expect:
      # Same question, same document. This document number must not appear for the
      # ordinary employee (restricted_regulation_absent) and must appear for HR.
      #
      # Judged on whether the document was **named**, not on the word "found". In
      # this corpus HR-GDL-003 v3 has no body text for the scale itself, so the
      # correct answer is "this is the guideline, and the table of grades is not in
      # what I can see" — which is the agent not inventing missing content, not a
      # failure of access control.
      contains: ["HR-GDL-003"]
      cites: {doc_no: "HR-GDL-003"}
```


### Graph and federation

**`demo/tests/cases/03_graph_and_erp.yaml`**

```yaml
# Questions no single source can answer.
suite: 그래프 · 연합
caller: {user: cho, emp_no: "${HR_EMP_NO}", dept: HR, clearance: hr}

cases:
  - id: authority_chain
    ask: 연차휴가 운영지침의 근거 규정을 전부 알려줘
    expect:
      # Depth varies per document, so this cannot be a join with a fixed number
      # of levels.
      contains: ["HR-REG-002", "GEN-RUL-001"]
      tools_used: [trace_authority]
      min_hops: 2

  - id: impact_of_revision
    ask: 정보보안규정을 개정하면 어떤 지침이 영향을 받아?
    expect:
      tools_used: [impact_analysis]
      min_results: 3

  - id: expense_compliance
    caller: {user: hong, emp_no: "${DEMO_EMP_NO}", dept: DEV, clearance: none}
    ask: 내 출장 숙박비 청구가 규정에 맞아?
    expect:
      # Needs the cap from the guideline and the amount from the ERP; neither
      # source answers alone.
      tools_used: [search_regulations, query_hr]
      contains: ["FIN-GDL-001"]

  - id: approval_ceiling
    ask: 천만원짜리 발주는 누가 결재해야 해?
    expect:
      contains: ["PUR-GDL-001"]
      tools_used: [search_regulations]

  - id: pending_approval_not_effective
    ask: 육아지원규정 개정안이 시행됐어?
    expect:
      # One approval is left in flight: drafted, unapproved. It must not be
      # treated as effective by anything reading the document body.
      indicates_not_effective: true
```


### Ontology

**`demo/tests/cases/04_ontology.yaml`**

```yaml
# Ontology — objects, links, actions.
#
# The three suites before this one ask whether search returns the right answer.
# This one asks something else: can the agent handle entities **by name** without
# writing SQL, and when read-only turns into a **governed write**, does the
# boundary hold.
suite: 온톨로지
caller: {user: hong, emp_no: "${DEMO_EMP_NO}", dept: DEV, clearance: none}

cases:
  - id: object_identity
    ask: HR-REG-003이 무슨 규정이고 몇 차 개정까지 있어?
    expect:
      # An object query, not a body search. The answer has to carry the title and
      # the versions together.
      tools_used: [describe_regulation]
      contains: ["육아지원규정"]

  - id: object_absent
    ask: HR-REG-999는 무슨 규정이야?
    expect:
      # Asked about an entity that does not exist, it has to say so. The ontology
      # knows only what is declared, and not inventing something plausible for what
      # it does not know is what this layer is worth.
      indicates_not_found: true

  - id: graph_link_traversal
    ask: 징계 양정기준의 근거가 되는 상위 규정을 온톨로지로 따라가줘
    expect:
      # A GRAPH binding — NeorunBase's graph engine does the traversal and what
      # comes back are regulation objects.
      tools_used: [related_regulations]
      contains: ["HR-REG-001"]

  - id: revision_request_written
    ask: 육아지원규정 3차에 대해 돌봄휴가 확대 사유로 개정 요청 넣어줘
    expect:
      # Not read-only. This call leaves a row in the ledger, and who called it when
      # is recorded in the audit log.
      tools_used: [request_regulation_revision]
      contains: ["HR-REG-003"]
      contains_pattern: "접수|요청"

  - id: revision_request_idempotent
    ask: 육아지원규정 3차 개정 요청 다시 한 번 넣어줘
    expect:
      # The same (user, regulation, version) is the same request. If asking twice
      # made two of them, the table would be a record of clicks rather than a queue
      # of approvals.
      tools_used: [request_regulation_revision]
      row_count: {sql: "SELECT count(*) FROM ice.reg.revision_requests WHERE doc_no = 'HR-REG-003'", equals: 1}
```


```bash
ANTHROPIC_API_KEY=... .venv/bin/python tests/run_cases.py \
    --ground-truth out/ground_truth.json --json out/case_results.json
```

---

## Measured

On a 16 GB machine with 10.7 GB given to Docker.

| | |
|---|---|
| Source documents | 446 → 123 matched, 319 unmatched, 4 scan-only |
| Ledger | 50 documents · 104 versions |
| Chunks | 818, none duplicated |
| Vectors | 818 × 768 dim |
| Graph | 50 nodes · 116 edges |
| Effective-date conflicts | 21 |
| ERP CDC | 5 tables / 1,214 rows, compared by value |
| One DAG run | about 50–70 seconds |

![Iceberg health](../images/demo/ontul-iceberg-health.png)

---

## The defects found while building this

All the same shape — **something went wrong and nobody was told.**

| What | How it looked |
|---|---|
| Iceberg's refresh loop stops after one failure | A catalog that served all day suddenly 401s, credentials still valid |
| A scan whose splits could not be planned | Every query **succeeded** over an apparently empty ledger |
| An unrecognised Flow snapshot value | Ran healthily, checkpointed, delivered 0 rows |
| A sink missing a required field | A Jackson NPE naming neither the sink nor the field, then restarting forever |
| `ORDER BY` in the GRAPH traversal SQL | NeorunBase refused it — every graph link failed |
| Local ingest and the DAG both loading chunks | 818 became 1636 and every count check passed |
| Chunks cleared, ingest queue left claiming otherwise | The next run drained an empty queue and reported success |
| CDC's default decimal encoding | Every number a base64 string, row counts exactly right |
| `depends_on` where kiok reads `requires` | Six tasks ran at once, quickly, with nothing failing |
| The dependency fetcher caching a fixed path | The edited script never ran again; the old code kept going |
| A policy filtering on a classification that does not exist | Read as if it excluded restricted regulations while excluding nothing |
| A row filter on the view but not on the retriever | An employee refused the text in SQL got it from search seconds later |

That list is also why the demo exists. The most expensive failure in a
regulation-answering system is not stopping — it is **being plausibly wrong** —
and the same thing turns out to be true of building one.
