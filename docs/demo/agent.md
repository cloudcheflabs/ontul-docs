# 에이전트 — 툴 9개, 그리고 질문한 사람으로 실행되는 것

에이전트는 이 스택을 쓰는 **하나의 클라이언트**입니다. 특권이 없습니다. 인증도
평범한 페르소나 계정으로 하고, 그 계정이 볼 수 없는 행은 에이전트도 못 봅니다.

```text
질문 ──► Agent (claude-opus-5)
           │  tool_runner 루프
           ▼
        9 tools ──► OntulClient ──► Ontul (as the caller)
                                      ├─ 리트리버 (하이브리드 검색)
                                      ├─ 시맨틱 뷰 (SQL)
                                      └─ 온톨로지 (객체 · 링크 · 액션)
```

---

## 클라이언트

모든 호출이 질문한 사람의 신원을 싣습니다.

**`demo/agent/src/regdemo_agent/client.py`**

```python
"""Thin Ontul client.

Every call carries the asking user's identity. The agent is not a service
account with broad access that filters afterwards — it runs as the caller, so a
row the caller cannot read is a row the agent cannot read, and that stays true
however the question is phrased.
"""
from __future__ import annotations

import os
from dataclasses import dataclass
from typing import Any

import requests


@dataclass
class Caller:
    """Who is asking. Bound to the Ontul session, not passed as prompt text."""
    user_id: str
    emp_no: str
    dept: str
    clearance: str = "none"   # none | hr | security


def _rows_as_dicts(body: Any) -> list[dict]:
    """Zip the response's columns onto its rows.

    The API answers with a column list and rows as arrays. Handing those arrays
    to a tool that indexes by name fails with "list indices must be integers",
    which reads like a bug in the tool rather than a shape it was never given.
    """
    if isinstance(body, list):
        return [r for r in body if isinstance(r, dict)]
    rows = body.get("rows") or []
    if rows and isinstance(rows[0], dict):
        return rows
    cols = [c if isinstance(c, str) else c.get("name") for c in (body.get("columns") or [])]
    if not cols:
        return []
    return [dict(zip(cols, r)) for r in rows]


class OntulClient:
    def __init__(self, base_url: str | None = None, timeout: int = 30) -> None:
        self.base = (base_url or os.environ.get("ONTUL_URL", "http://localhost:8080")).rstrip("/")
        self.timeout = timeout
        self._token_admin: str | None = None
        self._credentials: tuple[str, str] | None = None
        # One token per persona. Answers differ by who is asking, which is the
        # point, so the identity cannot be shared across callers.
        self._tokens: dict[str, str] = {}
        self._passwords: dict[str, str] = {}
        self._persona_password = os.environ.get("DEMO_PERSONA_PASSWORD", "regdemo-2026")

    def login(self, user: str, password: str) -> None:
        # Credentials are kept so the token can be renewed. Ontul's JWT lives
        # about fifteen minutes; a run longer than that dies on a 401 partway
        # through, which reads like an access-control result rather than an
        # expired session.
        self._credentials = (user, password)
        self._authenticate()

    def _authenticate(self) -> None:
        user, password = self._credentials
        r = requests.post(f"{self.base}/admin/auth/login",
                          json={"username": user, "password": password},
                          timeout=self.timeout)
        r.raise_for_status()
        self._token_admin = r.json()["accessToken"]

    def _post(self, path: str, payload: dict, caller: Caller):
        """POST with one re-authentication if the token has expired.

        Retried once only: a second 401 is a real authorisation answer, and
        retrying that would turn "you may not see this" into a loop.
        """
        r = requests.post(f"{self.base}{path}", json=payload,
                          headers=self._headers(caller), timeout=self.timeout)
        if r.status_code == 401:
            # The persona's token expired. Ontul's JWT lives about fifteen
            # minutes and a 401 reaching a tool becomes an empty result, which
            # the agent reports as "I could not find it" — a session that lapsed
            # looks exactly like a corpus with nothing in it.
            self._tokens.pop(caller.user_id, None)
            r = requests.post(f"{self.base}{path}", json=payload,
                              headers=self._headers(caller), timeout=self.timeout)
        return r

    def _headers(self, caller: Caller) -> dict:
        # The identity is the token, not a header. ${user.attr.*} resolves against
        # the stored user record, so the caller's attributes have to be on the
        # user the query authenticates as — sending them alongside an admin token
        # changes nothing, and admin holds AdministratorAccess, so every filter
        # and column rule is simply not consulted.
        return {"Authorization": f"Bearer {self._token(caller)}",
                "Content-Type": "application/json"}

    def register_password(self, username: str, password: str) -> None:
        """Password for a persona, so its first query can authenticate."""
        self._passwords[username] = password

    def _token(self, caller: Caller) -> str:
        """The token for this caller, logging in on first use."""
        tok = self._tokens.get(caller.user_id)
        if tok is None:
            tok = self._login_as(caller.user_id)
            self._tokens[caller.user_id] = tok
        return tok

    def _login_as(self, username: str) -> str:
        password = self._passwords.get(username, self._persona_password)
        r = requests.post(f"{self.base}/admin/auth/login",
                          json={"username": username, "password": password},
                          timeout=self.timeout)
        if r.status_code >= 300:
            raise RuntimeError(
                f"cannot authenticate as '{username}' — the demo personas are created by "
                f"infra/register.sh; running as admin would bypass every policy "
                f"({r.status_code})")
        return r.json()["accessToken"]

    # NeorunBase fails to parse a hybrid search whose lexical side matched
    # nothing, instead of returning no rows (tests/known-issues). The signature is
    # specific enough to recognise: a parse error naming a WHERE that the sent
    # statement does not contain, because the WHERE is in the SQL the rewriter
    # built. Asking about something the regulations do not cover is a case this
    # demo has to answer, and the answer is "nothing matched" — not a stack trace
    # the agent will spend three more turns rephrasing around.
    _EMPTY_MATCH = "SQL parse error: Encountered \"WHERE\""

    def invoke_retriever(self, fqn: str, caller: Caller, **args: Any) -> list[dict]:
        r = self._post(f"/api/v1/retrievers/{fqn}/invoke", {"args": args}, caller)
        body = r.json() if r.content else {}
        err = str(body.get("error") or "")
        if self._EMPTY_MATCH in err:
            return []
        r.raise_for_status()
        if err:
            raise RuntimeError(err)
        return _rows_as_dicts(body)

    def sql(self, statement: str, caller: Caller) -> list[dict]:
        r = self._post("/admin/query/execute", {"sql": statement}, caller)
        r.raise_for_status()
        body = r.json()
        if body.get("status") != "ok":
            # The field is "error"; reading "errorMessage" produced the bare
            # string "query failed" for every failure, which is the least useful
            # thing an error can say.
            raise RuntimeError(body.get("error") or body.get("errorMessage") or "query failed")
        cols = [c["name"] for c in body.get("columns", [])]
        return [dict(zip(cols, row)) for row in body.get("rows", [])]

    # ── 온톨로지 ────────────────────────────────────────────────────────────
    #
    # 읽기 두 개와 쓰기 하나. SQL 이 아니라 객체 이름으로 부르는 것이 요점입니다 —
    # 호출하는 쪽은 어느 카탈로그의 어느 테이블인지, 조인을 어떻게 하는지 몰라도
    # 되고, 서버가 속성 이름을 컬럼으로 옮깁니다. 선언되지 않은 속성을 부르면 SQL
    # 이 만들어지기 전에 400 으로 거절됩니다.

    def object_query(self, fqn: str, caller: Caller, filters: dict | None = None,
                     select: list[str] | None = None, limit: int = 20) -> list[dict]:
        """객체 타입의 인스턴스를 속성 이름으로 조회합니다."""
        payload: dict[str, Any] = {"limit": limit}
        if filters:
            payload["filters"] = filters
        if select:
            payload["select"] = select
        r = self._post(f"/api/v1/object-types/{fqn}/query", payload, caller)
        body = r.json() if r.content else {}
        if err := body.get("error"):
            raise RuntimeError(err)
        r.raise_for_status()
        return _rows_as_dicts(body)

    def link_traverse(self, fqn: str, caller: Caller, source_key: Any,
                      select: list[str] | None = None, max_depth: int = 1,
                      limit: int = 20) -> list[dict]:
        """링크를 따라 이어진 객체들을 가져옵니다 (JOIN 또는 GRAPH)."""
        payload: dict[str, Any] = {"sourceKey": str(source_key), "limit": limit,
                                   "maxDepth": max_depth}
        if select:
            payload["select"] = select
        r = self._post(f"/api/v1/link-types/{fqn}/traverse", payload, caller)
        body = r.json() if r.content else {}
        if err := body.get("error"):
            raise RuntimeError(err)
        r.raise_for_status()
        return _rows_as_dicts(body)

    def action_invoke(self, fqn: str, caller: Caller, args: dict,
                      idempotency_key: str | None = None) -> dict:
        """액션을 호출합니다 — 검증 · 인가 · 멱등 · 감사가 붙은 쓰기.

        멱등 키를 주면 같은 키의 재시도는 다시 쓰지 않고 이전 결과를 돌려줍니다.
        에이전트는 재시도를 하기 마련이고, 두 번 쓰이는 요청은 두 건의 요청으로
        보입니다.
        """
        payload: dict[str, Any] = {"args": args}
        if idempotency_key:
            payload["idempotencyKey"] = idempotency_key
        r = self._post(f"/api/v1/action-types/{fqn}/invoke", payload, caller)
        body = r.json() if r.content else {}
        if not body.get("ok") and body.get("error"):
            raise RuntimeError(body["error"])
        r.raise_for_status()
        return body
```


!!! note "왜 `caller` 를 툴 인자로 두지 않는가"
    모델이 다른 사번을 지어내면 **다른 사람으로 물어보게** 됩니다. 클로저로
    닫아 두면 질문이 누가 묻는지를 바꿀 수 없습니다.

---

## 툴

**`demo/agent/src/regdemo_agent/tools/registry.py`**

```python

"""The agent's tools.

Five, and the split is deliberate: retrieval, point lookup, two graph
directions, and ERP. A single "run SQL" tool would be more flexible and much
worse — the harness could not tell a regulation lookup from an HR query, and the
model would spend its turns writing SQL against schemas it half remembers
instead of asking questions.

Every tool takes the caller and passes it down. None of them can widen access.
"""
from __future__ import annotations
from datetime import date, timedelta

from ..client import Caller, OntulClient


EPOCH = date(1970, 1, 1)


def _epoch_days(iso: str) -> int:
    """Days since 1970-01-01, which is how the engine stores a DATE."""
    y, m, d = (int(x) for x in str(iso)[:10].split("-"))
    return (date(y, m, d) - EPOCH).days


def _as_iso(value) -> str:
    """Render a DATE the engine handed back. It arrives as epoch days."""
    if value is None:
        return ""
    text = str(value).strip()
    if text.isdigit():
        return (EPOCH + timedelta(days=int(text))).isoformat()
    return text[:10]


def build_tools(client: OntulClient, caller: Caller) -> list:
    """Return @beta_tool-decorated callables bound to this caller.

    Bound rather than parameterised: if the caller were a tool argument, a model
    that hallucinated a different employee number would be asking as someone
    else. It is closed over instead, so the question cannot change who is asking.
    """
    from anthropic import beta_tool

    @beta_tool
    def search_regulations(question: str, as_of: str = "", dept: str = "") -> str:
        """Search company regulations for the rule that answers a question.

        Returns only regulations in force on the given date (today if omitted),
        and only documents 인사팀 has marked as citable — so a meeting note that
        mentions the topic will not come back.

        Args:
            question: what to look up, in the user's own words
            as_of: YYYY-MM-DD to ask what the rule was on a past date
            dept: department code to narrow to, e.g. HR, FIN, SEC
        """
        rows = client.invoke_retriever(
            "semantic.rag.regulation_search", caller, q=question, k=8,
            as_of=as_of or None, dept=dept or None)
        if not rows:
            return "해당하는 규정을 찾지 못했습니다."
        return "\n\n".join(
            f"[{r['doc_no']} 제{r['version']}차 {r.get('article_no') or ''} "
            f"시행 {_as_iso(r['effective_from'])}]\n{r['body']}" for r in rows[:5])

    @beta_tool
    def get_effective_version(doc_no: str, as_of: str = "") -> str:
        """Which version of a regulation is (or was) in force, and since when.

        Use this when the answer depends on timing, or to confirm the version
        behind a rule you are about to quote.

        Args:
            doc_no: document number, e.g. HR-REG-003
            as_of: YYYY-MM-DD; today if omitted
        """
        d = as_of or date.today().isoformat()
        rows = client.sql(
            "SELECT doc_no, title, version, effective_from, effective_to, "
            "stated_from, date_mismatch FROM semantic.reg.version_history "
            # Compared as the epoch rendered to text, which is the only form
            # that filters correctly today. A DATE literal is refused by the
            # executor, an integer is refused by the planner, and
            # CAST('yyyy-mm-dd' AS DATE) — the one both accept — silently
            # returns the wrong rows, because the column is an Int32 holding the
            # epoch day and never carries its type into the comparison
            # (tests/known-issues/date-comparisons-silently-return-wrong-rows.md).
            #
            # Text ordering is correct here only because every value has the same
            # digit count. That holds from 1997-05-19 to 2243-10-16 and nowhere
            # else, so this goes away when the column is mapped as a date.
            f"WHERE doc_no = '{doc_no}' "
            f"AND CAST(effective_from AS VARCHAR) <= '{_epoch_days(d)}' "
            f"AND (effective_to IS NULL OR CAST(effective_to AS VARCHAR) > '{_epoch_days(d)}')",
            caller)
        if not rows:
            return f"{doc_no}: {d} 시점에 유효한 버전이 없습니다."
        r = rows[0]
        # Rendered back to a date. The engine returns epoch days, and an answer
        # that says "시행 20162" is worse than no answer — it looks like a fact.
        out = (f"{r['doc_no']} {r['title']} 제{r['version']}차 개정, "
               f"시행 {_as_iso(r['effective_from'])}")
        if r.get("date_mismatch"):
            # Surfaced rather than smoothed over: the document prints a date the
            # approval did not meet, and a reader comparing the two needs to know
            # which one governs.
            out += (f" (문서 부칙에는 {_as_iso(r['stated_from'])}로 기재되어 있으나 "
                    f"결재 승인일 {_as_iso(r['effective_from'])}이 기준입니다)")
        return out

    def _doc_id(doc_no: str) -> int | None:
        """The traversal's numeric id for a document number.

        GRAPH_NEIGHBORS seeds on a number, and a caller asking about
        "HR-GDL-002" has no reason to know one. Resolved here so the document
        number stays the only identifier that leaves this module.
        """
        # Through the semantic layer, like every other read. Querying the graph's
        # node table directly put one lookup outside the policies — and once
        # authorisation started requiring a grant on what a query names, it was
        # the one call an ordinary caller had no grant for.
        rows = client.sql(
            f"SELECT doc_id FROM semantic.reg.doc_index WHERE doc_no = '{doc_no}'", caller)
        return int(rows[0]["doc_id"]) if rows else None

    @beta_tool
    def trace_authority(doc_no: str) -> str:
        """The chain of regulations a given one derives its authority from.

        Depth varies per document, so this walks the hierarchy rather than
        looking up a single parent.

        Args:
            doc_no: document number to trace upward from
        """
        seed = _doc_id(doc_no)
        if seed is None:
            return f"{doc_no}는 규정 목록에 없습니다."
        rows = client.invoke_retriever("semantic.rag.authority_trace", caller, doc_id=seed)
        # depth 0 is the document itself; the chain is what lies above it.
        rows = [r for r in rows if int(r.get("depth", 0)) > 0]
        if not rows:
            return f"{doc_no}의 상위 규정을 찾지 못했습니다."
        return "\n".join(f"{'  ' * r['depth']}└ {r['doc_no']} {r['title']}"
                         for r in rows)

    @beta_tool
    def impact_of_change(doc_no: str) -> str:
        """Which guidelines and regulations would be affected if this one changed.

        Follows both derivation and citation inbound — a guideline that merely
        cites an article is affected by a change to it just as much as one
        derived from it.

        Args:
            doc_no: document number to assess
        """
        seed = _doc_id(doc_no)
        if seed is None:
            return f"{doc_no}는 규정 목록에 없습니다."
        rows = client.invoke_retriever("semantic.rag.impact_analysis", caller, doc_id=seed)
        rows = [r for r in rows if int(r.get("depth", 0)) > 0]
        if not rows:
            return f"{doc_no}에 의존하는 하위 문서가 없습니다."
        return (f"{len(rows)}건이 영향을 받습니다:\n"
                + "\n".join(f"  {r['doc_no']} {r['title']} ({r['owner_dept']})"
                            for r in rows))

    @beta_tool
    def query_hr(question_sql: str) -> str:
        """Read HR records — leave balances, expenses, purchase orders.

        Runs as the person asking, so rows they cannot see do not come back. An
        empty result means the data is not theirs to read, not that it is absent.

        Views: semantic.hr.employees, semantic.hr.leave_balance,
        semantic.hr.expenses, semantic.hr.purchase_orders.
        Columns are Korean: 사번, 성명, 부서, 휴가종류, 부여일수_ERP, 사용일수, 금액.

        Args:
            question_sql: a SELECT against one of those views
        """
        stmt = question_sql.strip().rstrip(";")
        if not stmt.lower().startswith("select"):
            # The tool is read-only by contract. Enforced here as well as by IAM
            # because a refusal the model can read is worth more than a 403 it
            # has to interpret.
            return "조회(SELECT)만 가능합니다."
        try:
            rows = client.sql(stmt, caller)
        except RuntimeError as e:
            return f"조회 실패: {e}"
        if not rows:
            return "조회 결과가 없습니다. (권한 범위 밖이거나 해당 데이터가 없습니다)"
        head = list(rows[0].keys())
        lines = [" | ".join(head)]
        lines += [" | ".join(str(r.get(c, "")) for c in head) for r in rows[:20]]
        if len(rows) > 20:
            lines.append(f"… 외 {len(rows) - 20}건")
        return "\n".join(lines)

    @beta_tool
    def pending_revision(doc_no: str = "") -> str:
        """Revisions currently moving through approval — drafted, under review or
        approved but not yet in force.

        Use this whenever the answer states a current rule. A rule can be correct
        and still be about to change, and an answer that omits an approved
        revision taking effect next month is technically right and practically
        wrong. The regulations tables hold only what is already in force; this is
        the only place an in-flight change is visible.

        Args:
            doc_no: limit to one regulation, e.g. HR-REG-003. Omit for all.
        """
        where = ""
        if doc_no.strip():
            safe = doc_no.strip().replace("'", "''")
            where = ' WHERE "문서번호" = \'' + safe + "'"
        try:
            rows = client.sql(
                'SELECT "문서번호", "제목", "개정차수", "단계", "요지", "예정시행일", "기안자사번" '
                f'FROM semantic.reg.pending_revisions{where} ORDER BY 1', caller)
        except RuntimeError as e:
            return f"결재 현황 조회 실패: {e}"
        # REJECTED is deliberately kept rather than filtered out. "제출됐지만
        # 반려됨"은 "그런 개정은 없음"과 다른 사실이고, 물어본 사람이 알아야
        # 하는 쪽은 대개 전자입니다.
        if not rows:
            return ("진행 중인 개정 결재가 없습니다."
                    if doc_no.strip() else "진행 중인 개정 결재가 없습니다.")
        out = []
        for r in rows:
            out.append(
                f"[{r['문서번호']} 제{r['개정차수']}차 개정 — {r['단계']}] "
                f"{r.get('요지') or ''} (예정 시행일 {_as_iso(r.get('예정시행일'))}, "
                f"기안 {r.get('기안자사번')})")
        return "\n".join(out)

    # ── 온톨로지 ────────────────────────────────────────────────────────────
    #
    # 위의 툴들은 SQL 이나 리트리버로 갑니다. 아래 셋은 온톨로지로 갑니다 —
    # 객체 이름과 속성 이름만 쓰고, 어느 테이블인지도 어떻게 조인하는지도
    # 모델이 알 필요가 없습니다. 그리고 마지막 하나는 읽기가 아니라 쓰기입니다.

    @beta_tool
    def describe_regulation(doc_no: str) -> str:
        """규정 한 건의 신원과 그 판들.

        본문을 찾는 것이 아니라 "이 규정이 무엇이고 몇 차까지 있는가" 를 봅니다.
        근거를 따라가기 전에 대상이 실재하는지 확인하는 데도 씁니다.

        Args:
            doc_no: 문서번호, 예: HR-REG-003
        """
        try:
            regs = client.object_query(
                "reg.ontology.Regulation", caller,
                filters={"doc_no": doc_no},
                select=["doc_no", "title", "doc_class", "tier", "owner_dept"], limit=1)
        except RuntimeError as e:
            return f"조회 실패: {e}"
        if not regs:
            return f"{doc_no}는 규정 목록에 없습니다."
        r = regs[0]
        out = [f"{r['doc_no']} {r['title']} (구분 {r.get('doc_class')}, "
               f"위계 {r.get('tier')}, 소관 {r.get('owner_dept')})"]
        try:
            vers = client.link_traverse(
                "reg.ontology.has_version", caller, r["doc_no"],
                select=["version", "status", "effective_from"], limit=20)
        except RuntimeError as e:
            return "\n".join(out + [f"판 조회 실패: {e}"])
        for v in sorted(vers, key=lambda x: x.get("version") or 0):
            out.append(f"  제{v['version']}차 [{v.get('status')}] "
                       f"시행 {_as_iso(v.get('effective_from'))}")
        return "\n".join(out)

    @beta_tool
    def related_regulations(doc_no: str, depth: int = 2) -> str:
        """근거 관계를 따라 이어진 규정들 — 온톨로지 그래프 링크로.

        trace_authority 와 같은 그래프를 보지만 부르는 방법이 다릅니다: 리트리버
        가 아니라 객체와 링크로 가고, 결과는 규정 객체입니다.

        Args:
            doc_no: 출발 문서번호
            depth: 몇 단계까지 따라갈지 (기본 2)
        """
        seed = _doc_id(doc_no)
        if seed is None:
            return f"{doc_no}는 규정 목록에 없습니다."
        try:
            rows = client.link_traverse(
                "reg.ontology.derives_from", caller, seed,
                select=["doc_no", "title", "tier"], max_depth=max(1, depth), limit=20)
        except RuntimeError as e:
            return f"순회 실패: {e}"
        rows = [r for r in rows if r.get("doc_no") != doc_no]
        if not rows:
            return f"{doc_no}에 연결된 근거 규정이 없습니다."
        return "\n".join(f"{r['doc_no']} {r['title']} (위계 {r.get('tier')})" for r in rows)

    @beta_tool
    def request_regulation_revision(doc_no: str, version: int, reason: str) -> str:
        """규정 개정을 요청합니다 — 원장에 기록되는 쓰기.

        읽기만 하는 다른 툴들과 다릅니다. 이건 실제로 기록을 남기므로, 사용자가
        개정을 요청해 달라고 명시했을 때만 부르십시오. 같은 규정의 같은 판에
        대한 두 번째 요청은 새 요청이 아니라 같은 요청으로 처리됩니다.

        Args:
            doc_no: 개정할 규정 번호
            version: 현재 시행 중인 판
            reason: 요청 사유
        """
        try:
            res = client.action_invoke(
                "reg.ontology.request_revision", caller,
                {"doc_no": doc_no, "version": int(version), "reason": reason},
                idempotency_key=f"{caller.user_id}:{doc_no}:v{version}")
        except RuntimeError as e:
            # 권한이 없으면 여기로 옵니다. 모델이 다시 시도하지 않도록 분명히
            # 말해 줍니다 — 표현을 바꿔서 통과할 수 있는 종류가 아닙니다.
            return f"개정 요청이 거부되었습니다: {e}"
        return (f"{doc_no} 제{version}차에 대한 개정 요청을 접수했습니다. "
                f"사유: {reason} (처리 {res.get('commandTag') or 'OK'})")

    return [search_regulations, get_effective_version, trace_authority,
            impact_of_change, query_hr, pending_revision,
            describe_regulation, related_regulations, request_regulation_revision]
```


| 툴 | 경로 |
|---|---|
| `search_regulations` | 리트리버 `semantic.rag.regulation_search` — 하이브리드 |
| `get_effective_version` | 시맨틱 뷰 `semantic.reg.version_history` |
| `trace_authority` | 리트리버 — 그래프 순회 |
| `impact_of_change` | 리트리버 — 역방향 순회 |
| `query_hr` | 시맨틱 뷰 `semantic.hr.*` — 행 필터가 걸린 채 |
| `pending_revision` | 시맨틱 뷰 `semantic.reg.pending_revisions` — CDC 로 들어온 결재 |
| `describe_regulation` | **온톨로지** ObjectSet + `has_version` |
| `related_regulations` | **온톨로지** `derives_from` GRAPH 순회 |
| `request_regulation_revision` | **온톨로지** 액션 — 유일한 쓰기 |

---

## 시스템 프롬프트

**`demo/agent/src/regdemo_agent/prompts/system.md`**

```markdown
You answer questions about company regulations and HR records.

## Your tools — use them

You have no knowledge of this company. Everything you say about its regulations,
its people or its records comes from one of these:

| tool | what it answers |
|---|---|
| `search_regulations(question, as_of, dept)` | which rule applies; returns the article text with its document number, version and effective date |
| `get_effective_version(doc_no, as_of)` | which version of a named regulation was in force, and whether its stated date differs from the approved one |
| `trace_authority(doc_no)` | the chain of regulations a given one derives its authority from |
| `impact_of_change(doc_no)` | which documents depend on a given one |
| `query_hr(question_sql)` | ERP records — leave balances, expenses, approvals — as the person asking |
| `pending_revision(doc_no)` | revisions still in approval — drafted, under review, approved but not yet in force |
| `describe_regulation(doc_no)` | what a named regulation *is* — title, class, owner, and every version it has |
| `related_regulations(doc_no, depth)` | regulations reached by following the authority graph from one document |
| `request_regulation_revision(doc_no, version, reason)` | **writes**: files a revision request against the ledger |

Call one before answering. Every question here is about this company, so there
is no question you can answer without them, including the ones that look like
general knowledge. If you find yourself composing an answer without a tool
result in front of you, that is the mistake this document exists to prevent.

When you state a current rule, check `pending_revision` for that document as
well. The regulations tables hold what is already in force, so an approved
revision taking effect next month is invisible to them — an answer that omits it
is correct today and wrong for the decision the person is about to make. Say
what applies now, then what is about to change and when.

## Which tool for which shape of question

`search_regulations` finds *text* — the article that answers a question. The
next two answer questions about the regulation as a thing rather than about what
it says:

- "HR-REG-003이 무슨 규정이야", "몇 차까지 있어", "어느 부서 소관이야" →
  `describe_regulation`. Searching the body for this returns article text and
  makes you infer the identity from it, which is how a wrong title gets stated
  confidently.
- "무엇을 근거로 하나", "상위 규정이 뭐야", "온톨로지로 따라가줘" →
  `related_regulations` when the question asks for the regulations themselves,
  `trace_authority` when it asks for the chain as a narrative. Both walk the same
  graph; neither is a body search, and neither can be answered by reading text.

## Writing

`request_regulation_revision` is the only tool that changes anything. Call it
when the person asks for a revision to be requested, filed, or submitted — and
only then. Do not call it to "check" whether it would work, and do not call it
because a rule looks outdated to you; noticing that a regulation should change is
not the same as being asked to file for it.

Filing the same request twice is not an error and does not create two requests —
the platform recognises the repeat and returns the original. So if someone asks
again, file again and say it is the same request, rather than refusing.

If it comes back refused, say so plainly. That is an authorisation decision, not
a phrasing problem, and rewording the request will not change it.

## Grounding

Every factual claim comes from a tool result. If the tools return nothing that
supports an answer, say you could not find it — do not fill the gap from general
knowledge about Korean labour law or from what a regulation of this kind usually
says. A confident answer with no source is worse than no answer, because nobody
can tell it apart from a correct one.

## Citations are mandatory

State the document number, the version, and the effective date for every rule
you quote:

> 육아휴직은 연간 20일까지 사용할 수 있습니다. (HR-REG-003 제3조, 제3차 개정, 시행 2025-03-15)

An answer without a citation is treated as a failure, not a style problem. The
citation is what lets someone open the document and check you.

## Dates

`search_regulations` returns only rules in force today unless you pass `as_of`.
When the question is about a past date — a dispute, an audit, "작년에는" — pass
that date. Do not reason about which version applied; ask for the date and let
the retrieval answer it.

If a document's stated effective date differs from its approval date, the
approval date governs. Say so when it matters to the answer.

## Numbers that combine two sources

Entitlement comes from the regulation; usage comes from the ERP. Fetch both and
show the arithmetic:

> 부여 20일 − 사용 12일 = 잔여 8일

Never carry an entitlement figure over from memory of an earlier answer. A
revision changes it, and a stale number is indistinguishable from a current one.

## Access

You run as the person asking. If a query returns nothing, that is the answer —
they cannot see those rows. Do not try another phrasing to get around it, do not
speculate about what the hidden values might be, and do not tell them what they
would see with more access. Say the information is not available to them.

## Style

Answer in Korean, in the register the question used. Lead with the answer, then
the citation, then any caveat. Keep it short — this is usually read on a phone
between meetings.
```


!!! quote "프롬프트가 하지 않는 일"
    접근 통제는 여기 없습니다. "박부장 기록은 보지 마" 같은 문장은 없고, 있어도
    소용이 없습니다 — 행 필터가 애초에 그 행을 돌려주지 않기 때문입니다.
    프롬프트가 하는 일은 **어느 툴을 언제 부를지**와 **인용을 반드시 달 것** 뿐입니다.

---

## 루프

**`demo/agent/src/regdemo_agent/agent.py`**

```python
"""The agent loop.

Claude Opus 5 with five tools. The SDK's tool runner drives the loop, so the
only thing written here is what the tools do and who is asking.
"""
from __future__ import annotations

import os
import sys
from pathlib import Path

from .client import Caller, OntulClient
from .tools.registry import build_tools

MODEL = os.environ.get("AGENT_MODEL", "claude-opus-5")
SYSTEM = (Path(__file__).parent / "prompts" / "system.md").read_text(encoding="utf-8")


def _client() -> "object":
    from anthropic import Anthropic
    # Reads ANTHROPIC_API_KEY, or an `ant auth login` profile if no key is set.
    #
    # A timeout, because the default is none and a hung connection then looks
    # exactly like a long turn — this model thinks before it answers, so "no
    # output for two minutes" is normal and "no output ever" is not
    # distinguishable from it without one.
    # The timeout has to sit INSIDE the caller's per-case deadline, or a slow
    # call is abandoned rather than reported: the deadline gives up on the
    # thread, the socket read keeps going with retries behind it, and the run
    # goes quiet for minutes with nothing saying which case is stuck. Retries
    # multiply it — 300s x 3 attempts against a 240s deadline meant the first
    # attempt could never finish in time.
    return Anthropic(timeout=float(os.environ.get("AGENT_TIMEOUT", "90")),
                     max_retries=int(os.environ.get("AGENT_RETRIES", "1")))


def ask(question: str, caller: Caller, ontul: OntulClient,
        verbose: bool = False, called: list[str] | None = None,
        on_event=None) -> str:
    """Answer one question and return the text.

    `called`, when given, is filled with the tool names used, in order. Which
    tools an answer went through is the difference between a refusal that came
    from a row filter and one the model decided on its own, and the transcript
    does not distinguish them.

    `on_event(kind, payload)`, when given, is called as things happen rather
    than after — a turn here runs for tens of seconds across several tool calls,
    and a caller that only gets the final string has nothing to show meanwhile.
    Kinds: "tool" with {name, input}, "text" with {text} for prose the model
    emitted before a tool call.

    max_tokens covers thinking as well as the reply. Opus 5 thinks by default,
    so a budget sized around the answer alone truncates mid-sentence.

    No temperature: it is rejected outright on this model rather than deprecated,
    and sending it costs a 400 and a retry on every call.
    """
    anthropic = _client()
    tools = build_tools(ontul, caller)

    runner = anthropic.beta.messages.tool_runner(
        model=MODEL,
        max_tokens=16000,
        thinking={"type": "adaptive"},
        output_config={"effort": "high"},
        system=SYSTEM,
        tools=tools,
        messages=[{"role": "user", "content": question}],
    )

    final = None
    for message in runner:
        if message.stop_reason == "refusal":
            # A 200 with no usable content. Reading content[0] here would raise
            # something unrelated and hide what actually happened.
            return "요청이 거부되었습니다."
        # Prose in a message that also calls a tool is commentary on the way to
        # the answer; prose in a message that calls none IS the answer, and
        # emitting it here as well would show it twice.
        step = any(b.type == "tool_use" for b in message.content)
        for block in message.content:
            if block.type == "tool_use":
                if called is not None:
                    called.append(block.name)
                if verbose:
                    print(f"  → {block.name}({block.input})", file=sys.stderr)
                if on_event is not None:
                    on_event("tool", {"name": block.name, "input": block.input})
            elif block.type == "text" and step and on_event is not None and block.text.strip():
                # Shown as it arrives so a long turn reads as work, not as a hang.
                on_event("text", {"text": block.text})
        final = message

    if final is None:
        return "응답을 받지 못했습니다."
    return "\n".join(b.text for b in final.content if b.type == "text").strip()


def main(argv: list[str] | None = None) -> int:
    import argparse

    ap = argparse.ArgumentParser(prog="regdemo-agent")
    ap.add_argument("question", nargs="*", help="question; omit for a REPL")
    ap.add_argument("--emp-no", default=os.environ.get("DEMO_EMP_NO", ""))
    ap.add_argument("--dept", default=os.environ.get("DEMO_DEPT", "DEV"))
    ap.add_argument("--clearance", default=os.environ.get("DEMO_CLEARANCE", "none"),
                    choices=["none", "hr", "security"])
    # A persona, not admin. Admin holds AdministratorAccess, so asking as admin
    # answers every question in full and demonstrates nothing about access.
    ap.add_argument("--user", default=os.environ.get("ONTUL_USER", "hong"),
                    help="the Ontul account to ask as (hong / park / cho)")
    ap.add_argument("--password",
                    default=os.environ.get("DEMO_PERSONA_PASSWORD", "regdemo-2026"))
    ap.add_argument("-v", "--verbose", action="store_true", help="show tool calls")
    args = ap.parse_args(argv)

    if not args.emp_no:
        # The employee number is context for the question, not the identity —
        # that comes from the account. Fall back to the seed's demo employee so
        # the REPL starts without having to look one up.
        try:
            import json
            gt = json.loads((Path.cwd() / "out" / "ground_truth.json").read_text(encoding="utf-8"))
            args.emp_no = gt["erp"]["demo_employee"]["emp_no"]
        except Exception:
            print("--emp-no required (out/ground_truth.json not readable from here)",
                  file=sys.stderr)
            return 2

    ontul = OntulClient()
    # Each caller authenticates as itself inside the client; the passwords are
    # registered here so a REPL session can switch personas without re-logging.
    ontul.register_password(args.user, args.password)
    caller = Caller(user_id=args.user, emp_no=args.emp_no,
                    dept=args.dept, clearance=args.clearance)

    if args.question:
        print(ask(" ".join(args.question), caller, ontul, args.verbose))
        return 0

    print(f"질문을 입력하세요 (사번 {args.emp_no}, {args.dept}). 종료: Ctrl-D")
    while True:
        try:
            q = input("\n> ").strip()
        except EOFError:
            return 0
        if q:
            print(ask(q, caller, ontul, args.verbose))


if __name__ == "__main__":
    sys.exit(main())
```


---

## 웹 채팅 화면

터미널로는 두 가지를 보여주기 어렵습니다: 같은 질문에 사람마다 다른 답이 나오는
것, 그리고 한 번의 답이 툴 세 번을 거치며 40초를 쓰는 것. 그래서 채팅 페이지가
하나 있고, 에이전트가 하는 일을 **하는 동안** 흘려보냅니다.

**`demo/web/server.py`**

```python
"""A chat window onto the regulation agent.

The point of this demo is that the same question gets different answers
depending on who asks, and that the answer names the version it came from. Both
are hard to show from a terminal: you cannot put two identities side by side,
and a turn that spends forty seconds in three tool calls looks like a hang.

So this serves a chat page and streams what the agent is doing while it does it
— each tool call as it is made, then the answer. Switching persona re-runs as a
different Ontul account, not as the same account with a different prompt: the
rows that come back are decided by the policy, so the difference on screen is
the access control working, not the model being agreeable.

    python3 -m web.server              # http://127.0.0.1:8900
    PORT=9000 HOST=0.0.0.0 python3 -m web.server

Standard library only. The demo already asks a lot of a machine, and a web
framework would be one more thing to install before anyone can look at it.
"""
from __future__ import annotations

import json
import os
import queue
import sys
import threading
import traceback
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
from pathlib import Path

HERE = Path(__file__).resolve().parent
DEMO = HERE.parent
sys.path.insert(0, str(DEMO / "agent" / "src"))

from regdemo_agent.agent import ask                      # noqa: E402
from regdemo_agent.client import Caller, OntulClient     # noqa: E402

PERSONA_PASSWORD = os.environ.get("DEMO_PERSONA_PASSWORD", "regdemo-2026")

# Kept in step with infra/register.sh, which creates these accounts and puts the
# attributes on them. The attributes here are for the label on screen — the ones
# that decide what comes back live on the Ontul user record.
PERSONAS = [
    {"id": "hong", "name": "홍가은", "role": "개발팀 사원",
     "emp_no": "20170003", "dept": "DEV", "clearance": "none",
     "note": "자기 기록만 봅니다"},
    {"id": "park", "name": "홍지아", "role": "개발팀 부장",
     "emp_no": "20150010", "dept": "DEV", "clearance": "manager",
     "note": "부서원 기록까지 봅니다"},
    {"id": "cho", "name": "조태윤", "role": "인사팀",
     "emp_no": "20090001", "dept": "HR", "clearance": "hr",
     "note": "전사 기록을 봅니다 (주민번호는 제외)"},
]
BY_ID = {p["id"]: p for p in PERSONAS}

SUGGESTIONS = [
    "육아휴직 며칠까지 쓸 수 있어? 곧 바뀌는 것도 있으면 알려줘.",
    "2025년 2월 기준으로는 육아휴직이 며칠이었어?",
    "나 육아휴직 며칠 남았어?",
    "사번 20090001 직원 휴가 현황 알려줘.",
    "육아지원규정은 어떤 규정에 근거해?",
]


def _sse(kind: str, payload: dict) -> bytes:
    return f"event: {kind}\ndata: {json.dumps(payload, ensure_ascii=False)}\n\n".encode()


class Handler(BaseHTTPRequestHandler):
    protocol_version = "HTTP/1.1"

    def log_message(self, fmt, *args):        # quieter than the default
        if os.environ.get("WEB_VERBOSE"):
            super().log_message(fmt, *args)

    # ── routes ───────────────────────────────────────────────────────────────

    def do_GET(self):
        if self.path in ("/", "/index.html"):
            self._send(200, "text/html; charset=utf-8",
                       (HERE / "chat.html").read_bytes())
        elif self.path == "/personas":
            self._send(200, "application/json; charset=utf-8",
                       json.dumps({"personas": PERSONAS, "suggestions": SUGGESTIONS},
                                  ensure_ascii=False).encode())
        else:
            self._send(404, "text/plain; charset=utf-8", b"not found")

    def do_POST(self):
        if self.path != "/ask":
            self._send(404, "text/plain; charset=utf-8", b"not found")
            return
        try:
            body = json.loads(self.rfile.read(int(self.headers.get("Content-Length", 0))) or b"{}")
        except Exception:
            self._send(400, "text/plain; charset=utf-8", b"bad request")
            return

        question = (body.get("question") or "").strip()
        persona = BY_ID.get(body.get("persona") or "hong")
        if not question or persona is None:
            self._send(400, "text/plain; charset=utf-8", b"question and persona required")
            return

        self.send_response(200)
        self.send_header("Content-Type", "text/event-stream; charset=utf-8")
        self.send_header("Cache-Control", "no-cache")
        self.send_header("Connection", "close")
        self.end_headers()
        self._stream(question, persona)

    # ── the turn ─────────────────────────────────────────────────────────────

    def _stream(self, question: str, persona: dict) -> None:
        """Run one turn, forwarding events to the browser as they happen.

        The agent call is synchronous and blocks for as long as the turn takes,
        so it runs on its own thread and posts to a queue this one drains. A
        turn that produced nothing to show for forty seconds would be
        indistinguishable from a dead server.
        """
        events: queue.Queue = queue.Queue()

        def run():
            try:
                ontul = OntulClient()
                ontul.register_password(persona["id"], PERSONA_PASSWORD)
                caller = Caller(user_id=persona["id"], emp_no=persona["emp_no"],
                                dept=persona["dept"], clearance=persona["clearance"])
                answer = ask(question, caller, ontul,
                             on_event=lambda kind, payload: events.put((kind, payload)))
                events.put(("answer", {"text": answer}))
            except Exception as e:                       # noqa: BLE001
                # Shown rather than swallowed. Most failures here are a policy
                # refusing something, and a blank reply would read as the agent
                # having nothing to say about it.
                events.put(("error", {"text": f"{type(e).__name__}: {e}",
                                      "detail": traceback.format_exc()[-1200:]}))
            finally:
                events.put((None, None))

        threading.Thread(target=run, daemon=True).start()

        while True:
            kind, payload = events.get()
            if kind is None:
                break
            try:
                self.wfile.write(_sse(kind, payload))
                self.wfile.flush()
            except (BrokenPipeError, ConnectionResetError):
                return          # reader navigated away; the thread is a daemon
        try:
            self.wfile.write(_sse("done", {}))
            self.wfile.flush()
        except (BrokenPipeError, ConnectionResetError):
            pass

    # ── plumbing ─────────────────────────────────────────────────────────────

    def _send(self, code: int, ctype: str, body: bytes) -> None:
        self.send_response(code)
        self.send_header("Content-Type", ctype)
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)


def main() -> int:
    host = os.environ.get("HOST", "127.0.0.1")
    port = int(os.environ.get("PORT", "8900"))
    if not os.environ.get("ANTHROPIC_API_KEY"):
        print("ANTHROPIC_API_KEY is not set — the page will load and every "
              "question will fail.", file=sys.stderr)
    srv = ThreadingHTTPServer((host, port), Handler)
    print(f"규정 에이전트 채팅: http://{host}:{port}")
    print(f"페르소나: {', '.join(p['id'] + '(' + p['name'] + ')' for p in PERSONAS)}")
    try:
        srv.serve_forever()
    except KeyboardInterrupt:
        print("\n종료")
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```


**`demo/web/chat.html`**

```html
<!doctype html>
<html lang="ko">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>규정 에이전트</title>
<style>
  :root{
    --bg:#0f1115; --panel:#161a21; --line:#252b36; --ink:#e6e9ef; --dim:#9aa4b5;
    --me:#2b3a55; --tool:#1d2530; --accent:#6ea8fe; --warn:#ffb86b; --bad:#ff8087;
  }
  @media (prefers-color-scheme: light){
    :root{ --bg:#f6f7f9; --panel:#fff; --line:#e3e6ec; --ink:#161a21; --dim:#61697a;
           --me:#dbe7ff; --tool:#f0f2f6; --accent:#2a6fdb; }
  }
  *{box-sizing:border-box}
  body{margin:0;background:var(--bg);color:var(--ink);
       font:15px/1.65 -apple-system,BlinkMacSystemFont,"Apple SD Gothic Neo","Noto Sans KR",sans-serif;
       display:flex;flex-direction:column;height:100vh}
  header{padding:14px 18px;border-bottom:1px solid var(--line);background:var(--panel);
         display:flex;gap:16px;align-items:center;flex-wrap:wrap}
  h1{font-size:15px;margin:0;font-weight:650;letter-spacing:-.01em}
  .who{display:flex;gap:6px;flex-wrap:wrap}
  .who button{background:transparent;color:var(--dim);border:1px solid var(--line);
              border-radius:999px;padding:5px 12px;font:inherit;font-size:13px;cursor:pointer}
  .who button.on{background:var(--accent);border-color:var(--accent);color:#fff}
  .who small{display:block;font-size:11px;opacity:.8}
  #log{flex:1;overflow-y:auto;padding:20px;display:flex;flex-direction:column;gap:14px}
  .row{display:flex}
  .row.me{justify-content:flex-end}
  .bubble{max-width:min(760px,86%);padding:11px 14px;border-radius:14px;
          background:var(--panel);border:1px solid var(--line);white-space:pre-wrap;word-break:break-word}
  .me .bubble{background:var(--me);border-color:transparent}
  .who-tag{font-size:11px;color:var(--dim);margin:0 4px 4px}
  .tools{display:flex;flex-direction:column;gap:5px;margin:2px 0 0}
  .tool{font:12px/1.5 ui-monospace,SFMono-Regular,Menlo,monospace;color:var(--dim);
        background:var(--tool);border:1px solid var(--line);border-radius:9px;padding:5px 9px;
        max-width:min(760px,86%);word-break:break-all}
  .tool b{color:var(--accent);font-weight:600}
  .note{font-size:12px;color:var(--dim);font-style:italic}
  .err{color:var(--bad)}
  .think{color:var(--dim)}
  footer{border-top:1px solid var(--line);background:var(--panel);padding:12px 18px}
  .sugg{display:flex;gap:6px;flex-wrap:wrap;margin-bottom:9px}
  .sugg button{background:transparent;border:1px dashed var(--line);color:var(--dim);
               border-radius:999px;padding:4px 11px;font:inherit;font-size:12px;cursor:pointer}
  .sugg button:hover{color:var(--ink);border-style:solid}
  form{display:flex;gap:9px}
  textarea{flex:1;resize:none;background:var(--bg);color:var(--ink);border:1px solid var(--line);
           border-radius:11px;padding:10px 12px;font:inherit;min-height:44px;max-height:150px}
  button.send{background:var(--accent);color:#fff;border:0;border-radius:11px;
              padding:0 20px;font:inherit;font-weight:600;cursor:pointer}
  button.send:disabled{opacity:.45;cursor:default}
  .spin{display:inline-block;width:9px;height:9px;border:2px solid var(--dim);
        border-top-color:transparent;border-radius:50%;animation:s .8s linear infinite;
        vertical-align:-1px;margin-right:6px}
  @keyframes s{to{transform:rotate(360deg)}}
</style>
</head>
<body>
<header>
  <h1>규정 에이전트</h1>
  <div class="who" id="who"></div>
</header>

<div id="log"></div>

<footer>
  <div class="sugg" id="sugg"></div>
  <form id="f">
    <textarea id="q" placeholder="규정이나 내 기록에 대해 물어보세요.  ⏎ 전송 · ⇧⏎ 줄바꿈" autofocus></textarea>
    <button class="send" id="send" type="submit">보내기</button>
  </form>
</footer>

<script>
const log = document.getElementById('log');
const whoBar = document.getElementById('who');
const suggBar = document.getElementById('sugg');
const form = document.getElementById('f');
const box = document.getElementById('q');
const send = document.getElementById('send');
let personas = [], current = 'hong', busy = false;

function el(cls, text){ const d=document.createElement('div'); d.className=cls; if(text!=null) d.textContent=text; return d; }
function atBottom(){ return log.scrollHeight - log.scrollTop - log.clientHeight < 80; }
function scroll(force){ if(force || atBottom()) log.scrollTop = log.scrollHeight; }

function bubble(side, text, tag){
  const row = el('row ' + side);
  const wrap = el('');
  if(tag) wrap.appendChild(el('who-tag', tag));
  const b = el('bubble', text);
  wrap.appendChild(b);
  row.appendChild(wrap);
  log.appendChild(row);
  scroll(true);
  return b;
}

fetch('/personas').then(r=>r.json()).then(d=>{
  personas = d.personas;
  personas.forEach(p=>{
    const b = document.createElement('button');
    b.innerHTML = `${p.name} <small>${p.role} · ${p.note}</small>`;
    b.onclick = ()=>{ current = p.id; render(); 
      log.appendChild(el('note', `— 이제 ${p.name}(${p.role}) 으로 묻습니다. 같은 질문도 답이 달라집니다.`));
      scroll(true); };
    b.dataset.id = p.id;
    whoBar.appendChild(b);
  });
  d.suggestions.forEach(s=>{
    const b = document.createElement('button');
    b.type='button'; b.textContent = s;
    b.onclick = ()=>{ box.value = s; box.focus(); };
    suggBar.appendChild(b);
  });
  render();
});

function render(){
  [...whoBar.children].forEach(b=> b.classList.toggle('on', b.dataset.id===current));
}

function persona(){ return personas.find(p=>p.id===current) || {name:current}; }

form.onsubmit = async (e)=>{
  e.preventDefault();
  const question = box.value.trim();
  if(!question || busy) return;
  box.value=''; busy=true; send.disabled=true;

  bubble('me', question, `${persona().name} · ${persona().role}`);

  const tools = el('tools');
  log.appendChild(tools);
  const status = el('tool'); status.innerHTML = '<span class="spin"></span>생각하는 중…';
  tools.appendChild(status); scroll(true);

  try{
    const res = await fetch('/ask', {
      method:'POST', headers:{'Content-Type':'application/json'},
      body: JSON.stringify({question, persona: current})
    });
    const reader = res.body.getReader();
    const dec = new TextDecoder();
    let buf = '';
    for(;;){
      const {value, done} = await reader.read();
      if(done) break;
      buf += dec.decode(value, {stream:true});
      // SSE frames are separated by a blank line.
      let i;
      while((i = buf.indexOf('\n\n')) >= 0){
        const frame = buf.slice(0, i); buf = buf.slice(i+2);
        const kind = (frame.match(/^event: (.*)$/m)||[])[1];
        const data = (frame.match(/^data: ([\s\S]*)$/m)||[])[1];
        if(!kind) continue;
        let payload = {};
        try{ payload = JSON.parse(data); }catch(_){}
        handle(kind, payload, status, tools);
      }
    }
  }catch(err){
    status.remove();
    bubble('them', '연결이 끊겼습니다: ' + err.message).classList.add('err');
  }
  status.remove();
  busy=false; send.disabled=false; box.focus();
};

function handle(kind, p, status, tools){
  if(kind === 'tool'){
    const t = el('tool');
    const args = JSON.stringify(p.input||{}, null, 0);
    t.innerHTML = `<b>${p.name}</b> ${args==='{}'?'':args}`;
    tools.insertBefore(t, status);
    status.innerHTML = '<span class="spin"></span>조회하는 중…';
    scroll();
  } else if(kind === 'text'){
    const t = el('tool think', p.text.trim());
    tools.insertBefore(t, status);
    scroll();
  } else if(kind === 'answer'){
    status.remove();
    bubble('them', p.text, `${persona().name} 에게 답함`);
  } else if(kind === 'error'){
    status.remove();
    bubble('them', p.text, '오류').classList.add('err');
  }
}

box.addEventListener('keydown', e=>{
  if(e.key==='Enter' && !e.shiftKey){ e.preventDefault(); form.requestSubmit(); }
});
</script>
</body>
</html>
```


```bash
python3 -m web.server            # http://127.0.0.1:8900
PORT=9000 HOST=0.0.0.0 python3 -m web.server
```

!!! note "페르소나를 바꾸면 계정이 바뀝니다"
    프롬프트가 바뀌는 것이 아닙니다. **다른 Ontul 계정으로 다시 실행**됩니다.
    화면에서 보이는 차이는 모델이 말을 잘 들어서가 아니라 정책이 다른 행을
    돌려주기 때문입니다.

---

## 에이전트 패키지

**`demo/agent/pyproject.toml`**

```toml
[project]
name = "regdemo-agent"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = ["anthropic>=0.60", "requests>=2"]

[project.scripts]
regdemo-agent = "regdemo_agent.agent:main"

[build-system]
requires = ["setuptools>=68"]
build-backend = "setuptools.build_meta"

[tool.setuptools.packages.find]
where = ["src"]
```


---

다음: [검증](verify.md).
