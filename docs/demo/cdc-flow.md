# CDC and Flow — the jobs that never finish

One thing separates these from the batch pipeline: **they do not end.** Leave
them running and the tables follow their sources.

There are three groups.

| Flow | Source → sink | Why |
|---|---|---|
| ERP CDC ×5 | PostgreSQL → Iceberg | No analytical load on the operational system, and one consistent point in time |
| Graph serving ×2 | Iceberg → NeorunBase | Because serving has to be a derivative |
| Approval events | S3 (JSON) → Iceberg | The real trigger for a regulation taking effect |

---

## 1. ERP → Iceberg (CDC)

### Why not connect directly

Read-only access sounds harmless and is usually refused anyway — few organisations
welcome analytical queries against a production ERP. And connecting directly makes
the **point in time slip**: each query sees a different instant, which shows up as
a wrong number in any answer that compares records against a regulation.

So CDC collects into Iceberg and the semantic views stay as they are. Only what is
underneath them changes.

**`demo/schema/flows/erp_cdc.json`**

```json
{
  "_comment": [
    "ERP → Iceberg, 실시간 CDC.",
    "",
    "데모 초기에는 ontul 이 Postgres 를 JDBC 로 직접 읽었습니다. 시연은 되지만",
    "실제 조직이 하는 방식은 아닙니다 — 분석 질의가 OLTP 의 buffer cache 를",
    "쓸어내고, 워커마다 커넥션을 열고, 무엇보다 **과거가 없습니다**. 덮어써진",
    "값은 되돌릴 수 없으니 '2025년 2월에는 며칠이었나' 를 물을 데가 없습니다.",
    "",
    "그래서 Debezium 이 WAL 을 읽어 Iceberg 로 흘립니다. 시맨틱 뷰와 IAM 정책은",
    "그대로 두고 그 아래만 갈아끼웁니다 — 뷰가 원천을 가리고 있어서 가능한",
    "교체이고, 이 층을 둔 이유이기도 합니다.",
    "",
    "테이블마다 Flow 하나입니다. ontul 의 Flow 는 싱크 테이블이 하나라서",
    "그렇고, Postgres 쪽에는 Flow 당 복제 슬롯이 하나씩 생깁니다.",
    "",
    "upsertKeys 가 PK 이고 cdc.apply 가 켜져 있으므로, update 는 행을 교체하고",
    "delete 는 행을 지웁니다 — append 로 쌓이지 않습니다. 이게 동작하려면",
    "coordinated 커밋이 equality delete 를 같이 실어야 하는데, 그게 안 되던",
    "결함이 8e82f3f 입니다."
  ],
  "tables": [
    {
      "source": "public.hr_employee",
      "sink": "ice.erp.hr_employee",
      "keys": [
        "emp_no"
      ]
    },
    {
      "source": "public.hr_org",
      "sink": "ice.erp.hr_org",
      "keys": [
        "dept_cd"
      ]
    },
    {
      "source": "public.hr_leave_balance",
      "sink": "ice.erp.hr_leave_balance",
      "keys": [
        "emp_no",
        "yr",
        "lv_typ_cd"
      ]
    },
    {
      "source": "public.fi_expense",
      "sink": "ice.erp.fi_expense",
      "keys": [
        "exp_id"
      ]
    },
    {
      "source": "public.pu_purchase_order",
      "sink": "ice.erp.pu_purchase_order",
      "keys": [
        "po_id"
      ]
    }
  ],
  "source": {
    "type": "cdc",
    "connector": "postgres",
    "hostname": "postgres-erp",
    "port": "5432",
    "database": "erp",
    "username": "erp",
    "password": "${ERP_PASSWORD}",
    "snapshot": "initial",
    "_snapshot_note": "initial = 현재 상태를 한 번 다 읽고 그 뒤로 증분. 이게 없으면 Flow 를 켠 시점 이후의 변경만 들어와 표가 비어 보입니다."
  },
  "sink": {
    "type": "table",
    "_write_note": "write.mode=cdc 가 op 컬럼(__op)을 보고 c/u/r 은 upsert, d 는 삭제로 가릅니다. keys 는 Flow 마다 다르므로 제출할 때 채워집니다.",
    "write": {
      "mode": "cdc",
      "opColumn": "__op",
      "delete": "hard",
      "keys": []
    },
    "schemaEvolution": "add"
  },
  "ontul.streaming.exactly.once": "true",
  "commitIntervalMs": 3000,
  "numWorkers": 1,
  "durationMs": 9999999999
}
```


!!! danger "Leaving `decimal.handling.mode` at its default"
    Debezium's default encodes a NUMERIC as **base64 unscaled bytes**. A column
    that should read 18.0 arrives downstream as the string `"ALQ="` — and **the
    row counts match exactly**. Every count-based check passes while every value
    is wrong.

    That is why the verification on this page compares values, not counts.

**`demo/infra/cdc.sh`**

```bash
#!/usr/bin/env bash
# ERP → Iceberg, ontul Flow 로.
#
#   bash infra/cdc.sh            # 5개 Flow 기동 + 초기 스냅샷 확인
#   bash infra/cdc.sh verify     # 원천과 대상 행 수 비교
#   bash infra/cdc.sh change     # ERP 에 UPDATE/DELETE 를 넣고 반영되는지 확인
#   bash infra/cdc.sh stop
#
# JDBC 직결을 대체합니다. 시맨틱 뷰는 그대로 두고 아래만 바뀝니다.
set -uo pipefail
cd "$(dirname "$0")/.."
DEMO=$(pwd)
. out/stack.env 2>/dev/null || { echo "out/stack.env 없음 — infra/up.sh 먼저"; exit 1; }

ONTUL=${ONTUL_URL:-http://localhost:8080}
ADMIN_PW="${ONTUL_ADMIN_PASSWORD:-regdemo-admin-2026}"
ERP_PW="${ERP_PASSWORD:-regdemo}"
SPEC="$DEMO/schema/flows/erp_cdc.json"
MODE=${1:-start}

log(){ printf '\n\033[1;36m== %s\033[0m\n' "$*"; }
step(){ printf '\033[1;32m  ok\033[0m %s\n' "$*"; }
warn(){ printf '\033[1;33m  ..\033[0m %s\n' "$*"; }
fail(){ printf '\033[1;31m  실패\033[0m %s\n' "$*"; exit 1; }

TOK=$(curl -s -m 30 -XPOST "$ONTUL/admin/auth/login" -H 'Content-Type: application/json' \
      -d "{\"username\":\"admin\",\"password\":\"$ADMIN_PW\"}" \
      | python3 -c "import sys,json;print(json.load(sys.stdin).get('accessToken',''))" 2>/dev/null)
[ -n "$TOK" ] || fail "ontul 로그인 실패"
AH="Authorization: Bearer $TOK"

sql(){ curl -s -m 120 -XPOST "$ONTUL/admin/query/execute" -H "$AH" -H 'Content-Type: application/json' \
        -d "$(python3 -c "import json,sys;print(json.dumps({'sql':sys.argv[1]}))" "$1")"; }
count(){ sql "SELECT count(*) FROM $1" | python3 -c "
import sys,json
d=json.load(sys.stdin)
print((d.get('rows') or [['ERR']])[0][0] if d.get('status')=='ok' else 'ERR')" 2>/dev/null; }
pg(){ docker exec regdemo-erp psql -U erp -d erp -tAc "$1" 2>/dev/null | tr -d '[:space:]'; }
tables(){ python3 -c "
import json;d=json.load(open('$SPEC',encoding='utf-8'))
for t in d['tables']: print(t['source'], t['sink'], ','.join(t['keys']))"; }

# 자기 것만 죽입니다. 스트리밍 잡을 전부 죽이면 이 스크립트가 남의
# 파이프라인을 끕니다 — 실제로 그랬습니다: graph_flow 가 띄운 2개를 cdc 가
# 죽이고, cdc 가 띄운 5개를 flow 가 죽여서, 세 스크립트를 차례로 돌리면 마지막
# 하나만 살아남았습니다. 각 스크립트가 성공을 보고했기 때문에 아무도 눈치채지
# 못했습니다.
streaming_jobs(){ curl -s -m 30 "$ONTUL/v1/api/job/list" -H "$AH" \
  | python3 -c "
import sys, json
for j in json.load(sys.stdin):
    if j.get('type') != 'STREAMING':
        continue
    tag = (j.get('description') or '') + ' ' + (j.get('jobName') or '')
    if 'erp-cdc' in tag:
        print(j['jobId'])
" 2>/dev/null; }

if [ "$MODE" = "stop" ]; then
  log "Flow 중지 + 복제 슬롯 정리"
  for jid in $(streaming_jobs); do curl -s -XPOST "$ONTUL/v1/api/job/kill/$jid" -H "$AH" -o /dev/null; step "kill $jid"; done
  # 슬롯을 남기면 Postgres 가 WAL 을 계속 붙들고 디스크가 찹니다.
  for s in $(pg "SELECT slot_name FROM pg_replication_slots WHERE slot_name LIKE 'ontul_%';"); do
    pg "SELECT pg_drop_replication_slot('$s');" >/dev/null && step "슬롯 삭제 $s"
  done
  exit 0
fi

num_eq(){ python3 -c "
import sys
a=sys.argv[1].split('|'); b=sys.argv[2].split('|')
try:
    ok = len(a)==len(b) and all(abs(float(x)-float(y))<1e-9 for x,y in zip(a,b))
except ValueError:
    ok = False
print('1' if ok else '0')" "$1" "$2"; }

if [ "$MODE" = "verify" ]; then
  log "원천 대 대상 — 행 수"
  tables | while read -r src sink keys; do
    t=${src#public.}
    a=$(pg "SELECT count(*) FROM $t;"); b=$(count "$sink")
    if [ "$a" = "$b" ]; then step "$t: $a = $b"
    else printf '\033[1;31m  차이\033[0m %s: 원천 %s, Iceberg %s\n' "$t" "$a" "$b"; fi
  done

  # 행 수만 보는 검증은 이 파이프라인에서 통과해 놓고 틀린 적이 있습니다. Debezium 이
  # NUMERIC 을 base64 바이트로 보내던 동안 572 = 572 였고 값은 전부 "ALQ=" 였습니다.
  # 그래서 값도 읽습니다 — 숫자 하나, 문자 하나.
  log "값 비교"
  emp=$(pg "SELECT emp_no FROM hr_leave_balance ORDER BY emp_no, lv_typ_cd LIMIT 1;")
  typ=$(pg "SELECT lv_typ_cd FROM hr_leave_balance WHERE emp_no='$emp' ORDER BY lv_typ_cd LIMIT 1;")
  src_v=$(pg "SELECT grant_days || '|' || used_days FROM hr_leave_balance WHERE emp_no='$emp' AND lv_typ_cd='$typ';")
  ice_v=$(sql "SELECT grant_days, used_days FROM ice.erp.hr_leave_balance WHERE emp_no = '$emp' AND lv_typ_cd = '$typ'" \
          | python3 -c "
import sys,json
d=json.load(sys.stdin); r=(d.get('rows') or [[None,None]])[0]
print('|'.join('' if x is None else str(x) for x in r[:2]))" 2>/dev/null)
  if [ "$(num_eq "$src_v" "$ice_v")" = "1" ]; then
    step "hr_leave_balance($emp,$typ) 수치 일치 — 원천 $src_v, Iceberg $ice_v"
  else
    printf '\033[1;31m  불일치\033[0m hr_leave_balance(%s,%s): 원천 %s, Iceberg %s\n' "$emp" "$typ" "$src_v" "$ice_v"
  fi

  nm_src=$(pg "SELECT emp_nm FROM hr_employee WHERE emp_no='$emp';")
  nm_ice=$(sql "SELECT emp_nm FROM ice.erp.hr_employee WHERE emp_no = '$emp'" \
           | python3 -c "
import sys,json
d=json.load(sys.stdin); r=d.get('rows') or []
print(r[0][0] if r else '')" 2>/dev/null)
  if [ "$nm_src" = "$nm_ice" ]; then step "hr_employee($emp) 성명 일치 — $nm_src"
  else printf '\033[1;31m  불일치\033[0m hr_employee(%s): 원천 %s, Iceberg %s\n' "$emp" "$nm_src" "$nm_ice"; fi
  exit 0
fi

if [ "$MODE" = "change" ]; then
  # CDC 가 append 가 아니라는 것을 보이는 부분입니다. update 는 행을 교체하고
  # delete 는 지웁니다 — 행 수가 늘지 않아야 정상입니다.
  log "ERP 에 변경을 넣습니다"
  before=$(count ice.erp.hr_leave_balance)
  emp=$(pg "SELECT emp_no FROM hr_leave_balance ORDER BY emp_no LIMIT 1;")
  pg "UPDATE hr_leave_balance SET used_days = used_days + 1 WHERE emp_no = '$emp';" >/dev/null
  step "UPDATE hr_leave_balance (사번 $emp)"
  pg "DELETE FROM fi_expense WHERE exp_id = (SELECT exp_id FROM fi_expense ORDER BY exp_id LIMIT 1);" >/dev/null
  step "DELETE fi_expense 1건"
  warn "커밋 간격 + 스냅샷 반영 대기 (25초)"
  sleep 25
  after=$(count ice.erp.hr_leave_balance)
  if [ "$before" = "$after" ]; then step "hr_leave_balance 행 수 유지: $before (update 가 교체됨)"
  else printf '\033[1;31m  회귀\033[0m hr_leave_balance %s → %s — upsert 가 append 로 격하됐습니다\n' "$before" "$after"; fi
  bash "$0" verify
  exit 0
fi

log "1/3  기존 Flow 와 슬롯 정리"
for jid in $(streaming_jobs); do curl -s -XPOST "$ONTUL/v1/api/job/kill/$jid" -H "$AH" -o /dev/null; step "이전 잡 kill $jid"; done
for s in $(pg "SELECT slot_name FROM pg_replication_slots WHERE slot_name LIKE 'ontul_%';"); do
  pg "SELECT pg_drop_replication_slot('$s');" >/dev/null && step "슬롯 삭제 $s"
done
sql "CREATE SCHEMA IF NOT EXISTS ice.erp" >/dev/null; step "ice.erp 스키마"

log "2/3  테이블당 Flow 제출"
tables | while read -r src sink keys; do
  JOBID=$(TOK="$TOK" ONTUL="$ONTUL" SPEC="$SPEC" SRC="$src" SINK="$sink" KEYS="$keys" PW="$ERP_PW" python3 <<'PY'
import os, json, urllib.request, time
tok=os.environ["TOK"]; base=os.environ["ONTUL"]
cfg=json.loads(open(os.environ["SPEC"],encoding="utf-8").read())
cfg.pop("_comment",None); cfg.pop("tables",None)
src=cfg["source"]; src.pop("_snapshot_note",None)
src["password"]=os.environ["PW"]
src["table"]=os.environ["SRC"]
sink=cfg["sink"]; sink.pop("_write_note",None)
sink["table"]=os.environ["SINK"]
sink["write"]["keys"]=os.environ["KEYS"].split(",")
def api(path, body=None, method="GET"):
  data=json.dumps(body).encode() if body is not None else None
  r=urllib.request.Request(base+path, data=data, method=method)
  r.add_header("Authorization","Bearer "+tok); r.add_header("Content-Type","application/json")
  return json.load(urllib.request.urlopen(r,timeout=90))
try:
  # 나중에 이 묶음만 골라 끌 수 있도록 이름표를 답니다.
  cfg["description"] = "erp-cdc:" + str(cfg.get("source", {}).get("table") or "")
  d=api("/v1/api/sql", {"sql":"SUBMIT STREAMING "+json.dumps(cfg)}, "POST")
  if d.get("status")!="ok": print("ERR:"+str(d)[:220]); raise SystemExit
  want="streaming-"+(d.get("queryId") or "")[:8]
  for _ in range(25):
    for j in api("/v1/api/job/list"):
      if j.get("type")=="STREAMING" and j.get("jobName")==want: print(j["jobId"]); raise SystemExit
    time.sleep(1)
  print("ERR:job never appeared")
except SystemExit: pass
except Exception as e: print("ERR:"+str(e)[:220])
PY
)
  case "$JOBID" in ERR:*|"") printf '\033[1;31m  실패\033[0m %s -> %s (%s)\n' "$src" "$sink" "$JOBID";;
                   *) step "$src → $sink  [$JOBID]";; esac
done

log "3/3  초기 스냅샷 대기"
for i in $(seq 1 40); do
  n=$(count ice.erp.hr_employee)
  [ "$n" != "ERR" ] && [ "${n:-0}" -ge 300 ] 2>/dev/null && break
  sleep 6
done
bash "$0" verify
echo
echo "  변경이 반영되는지 보려면:  bash infra/cdc.sh change"
echo "  멈추려면:                  bash infra/cdc.sh stop"
```


```bash
bash infra/cdc.sh
```

```text
== 2/3  Flow per table
  ok public.hr_employee → ice.erp.hr_employee  [f5fb5c96…]
  ok public.hr_org → ice.erp.hr_org            [61a06d47…]
  ok public.hr_leave_balance → …               [db339d92…]
  ok public.fi_expense → …                     [9a3e92f2…]
  ok public.pu_purchase_order → …              [ec7017ce…]

== source vs target — row counts
  ok hr_employee: 300 = 300
  ok hr_org: 12 = 12
  ok hr_leave_balance: 572 = 572
  ok fi_expense: 240 = 240
  ok pu_purchase_order: 90 = 90

== compared by value
  ok hr_leave_balance(20090001,ANN) — source 23.0|23.0, Iceberg 23.0|23.0
  ok hr_employee(20090001) name matches
```

To watch a change propagate:

```bash
bash infra/cdc.sh change
```

---

## 2. The graph → serving

The ontology's `derives_from` link is a GRAPH binding, so it traverses
NeorunBase's instance graph. These Flows are what fill that graph.

**The batch job does not write to NeorunBase.** It used to, and that was wrong
because it made the serving layer the system of record: rebuild it and the
relations were gone, no history of what changed when existed, and the only way to
fix the graph was "run the job again".

**`demo/schema/flows/graph_serving.json`**

```json
{
  "_comment": [
    "그래프 서빙 동기화 — ice.reg.graph_edges → NeorunBase doc_edges.",
    "",
    "온톨로지의 derives_from 링크는 GRAPH 바인딩이라 NeorunBase 인스턴스",
    "그래프를 순회합니다. 그 그래프를 채우는 것이 이 Flow 입니다.",
    "",
    "배치 잡이 NeorunBase 에 직접 쓰지 않는 이유가 여기 있습니다. Iceberg 가",
    "기록의 원본이고 NeorunBase 는 다시 만들 수 있는 파생 서빙 계층입니다 —",
    "서빙을 날려도 Flow 가 스냅샷부터 다시 읽어 채웁니다. 관계가 언제 어떻게",
    "바뀌었는지는 Iceberg 스냅샷에 남고, 그건 잡을 다시 돌려서는 얻을 수",
    "없는 것입니다.",
    "",
    "source.mode=changelog: 배치 잡은 지우고 다시 넣습니다. append 로 읽으면",
    "삭제가 보이지 않아 서빙에 옛 엣지가 남습니다 — 인용이 사라진 규정이 계속",
    "근거로 따라와도 아무도 오류를 보지 못합니다.",
    "",
    "sink 가 neorunbase 가 아니라 jdbc 인 것이 핵심입니다. neorunbase 싱크는",
    "REST 대량 삽입이라 덧붙이기만 합니다 — 전량 재작성 소스와 붙이면 같은",
    "엣지가 계속 쌓입니다. jdbc 싱크의 mode=cdc 는 __op 를 보고 c/u/r 은",
    "키 기준 upsert, d 는 삭제로 적용해서, 서빙이 원본의 복제본으로 유지됩니다.",
    "NeorunBase 는 Postgres 와이어 프로토콜을 서빙하므로 그대로 붙습니다.",
    "",
    "snapshot=all: 기존 행까지 전부 읽고 그 뒤로 이어갑니다. 기본값인",
    "latest 는 Flow 가 시작한 뒤에 추가된 것만 흘려보내므로, 이미 쌓여",
    "있는 관계는 서빙에 영영 도달하지 않습니다."
  ],
  "source": {
    "type": "iceberg",
    "table": "ice.reg.graph_edges",
    "snapshot": "all",
    "mode": "changelog"
  },
  "operations": [],
  "sink": {
    "type": "jdbc",
    "jdbcUrl": "jdbc:postgresql://${NEORUNBASE_INTERNAL_HOST}:5432/neorunbase?preferQueryMode=simple",
    "tableName": "doc_edges",
    "username": "admin",
    "password": "${NEORUNBASE_PASSWORD}",
    "mode": "cdc",
    "keys": [
      "edge_pk"
    ],
    "opColumn": "__op"
  },
  "commitIntervalMs": 2000,
  "numWorkers": 1,
  "durationMs": 9999999999
}
```
**`demo/schema/flows/graph_nodes.json`**

```json
{
  "_comment": [
    "그래프 노드 동기화 — ice.reg.graph_nodes → NeorunBase doc_nodes.",
    "",
    "엣지와 같은 이유로 같은 모양입니다. 노드가 없으면 순회는 되지만 결과에",
    "제목이 없어서, 에이전트는 근거를 찾고도 그것을 부를 이름이 없습니다.",
    "",
    "snapshot=all: 기존 행까지 전부 읽고 그 뒤로 이어갑니다. 기본값인",
    "latest 는 Flow 가 시작한 뒤에 추가된 것만 흘려보내므로, 이미 쌓여",
    "있는 관계는 서빙에 영영 도달하지 않습니다."
  ],
  "source": {
    "type": "iceberg",
    "table": "ice.reg.graph_nodes",
    "snapshot": "all",
    "mode": "changelog"
  },
  "operations": [],
  "sink": {
    "type": "jdbc",
    "jdbcUrl": "jdbc:postgresql://${NEORUNBASE_INTERNAL_HOST}:5432/neorunbase?preferQueryMode=simple",
    "tableName": "doc_nodes",
    "username": "admin",
    "password": "${NEORUNBASE_PASSWORD}",
    "mode": "cdc",
    "keys": [
      "doc_id"
    ],
    "opColumn": "__op"
  },
  "commitIntervalMs": 2000,
  "numWorkers": 1,
  "durationMs": 9999999999
}
```


!!! danger "A wrong `snapshot` value delivers nothing, quietly"
    The field accepts `latest` and `all`. A plausible-looking value such as
    `earliest` was **silently treated as `latest`**, so the Flow ran healthily,
    checkpointed, and delivered none of the existing rows. Indistinguishable from
    an idle source.

    (Ontul 1.0.0 rejects unrecognised values.)

!!! note "Why the sink is `jdbc` and not `neorunbase`"
    The `neorunbase` sink is a REST bulk insert — it only **appends**. Point it at
    a source that rewrites its table in full and the same edges pile up. The
    `jdbc` sink in `mode: cdc` reads `__op` and applies c/u/r as an upsert on the
    key and d as a delete, so serving stays a replica of the source. NeorunBase
    serves the PostgreSQL wire protocol, so it attaches directly.

**`demo/infra/graph_flow.sh`**

```bash
#!/usr/bin/env bash
# 그래프 서빙 Flow — ice.reg.graph_{nodes,edges} → NeorunBase.
#
#   bash infra/graph_flow.sh          # 두 Flow 기동 + 반영 확인
#   bash infra/graph_flow.sh stop
#
# 배치 잡(build_graph_job)은 관계를 Iceberg 에 씁니다. NeorunBase 를 채우는
# 것은 이 Flow 이고, 온톨로지의 derives_from(GRAPH 바인딩)이 순회하는 것도
# 그렇게 채워진 그래프입니다. 서빙을 날려도 Flow 가 스냅샷부터 다시 읽습니다.
set -uo pipefail
cd "$(dirname "$0")/.."
DEMO=$(pwd)
. out/stack.env 2>/dev/null || { echo "out/stack.env 없음 — infra/up.sh 먼저"; exit 1; }

ONTUL=${ONTUL_URL:-http://localhost:8080}
ADMIN_PW="${ONTUL_ADMIN_PASSWORD:-regdemo-admin-2026}"
MODE=${1:-start}

log(){ printf '\n\033[1;36m== %s\033[0m\n' "$*"; }
step(){ printf '\033[1;32m  ok\033[0m %s\n' "$*"; }
fail(){ printf '\033[1;31m  실패\033[0m %s\n' "$*"; exit 1; }

TOK=$(curl -s -XPOST "$ONTUL/admin/auth/login" -H 'Content-Type: application/json' \
      -d "{\"username\":\"admin\",\"password\":\"$ADMIN_PW\"}" \
      | python3 -c "import sys,json;print(json.load(sys.stdin).get('accessToken',''))" 2>/dev/null)
[ -n "$TOK" ] || fail "ontul 로그인 실패"
AH="Authorization: Bearer $TOK"

# 이름으로 골라 죽입니다. 결재 Flow 와 CDC Flow 가 같이 돌고 있으므로,
# STREAMING 을 전부 kill 하면 이 스크립트가 남의 파이프라인을 끕니다.
kill_graph_flows(){
  curl -s "$ONTUL/v1/api/job/list" -H "$AH" | python3 -c "
import sys, json
for j in json.load(sys.stdin):
    if j.get('type') == 'STREAMING' and 'graph' in (j.get('description') or j.get('jobName') or ''):
        print(j['jobId'])
" 2>/dev/null | while read -r jid; do
    curl -s -XPOST "$ONTUL/v1/api/job/kill/$jid" -H "$AH" -o /dev/null
    step "kill $jid"
  done
}

if [ "$MODE" = "stop" ]; then
  log "그래프 Flow 중지"
  kill_graph_flows
  exit 0
fi

submit(){ # submit <spec> <label>
  TOK="$TOK" ONTUL="$ONTUL" SPEC="$1" LABEL="$2" \
  NB_HOST="${NEORUNBASE_INTERNAL_HOST:-neorun-coordinator-1}" \
  NB_PW="${NEORUNBASE_PASSWORD:-Regdemo12345}" python3 <<'PY'
import os, json, urllib.request, time
tok = os.environ["TOK"]; base = os.environ["ONTUL"]; label = os.environ["LABEL"]
raw = open(os.environ["SPEC"], encoding="utf-8").read()
# 스펙은 그대로 두고 자격증명만 주입합니다 — 파일에 비밀번호를 적어
# 커밋하지 않기 위해서입니다.
for k, v in (("NEORUNBASE_INTERNAL_HOST", os.environ["NB_HOST"]),
             ("NEORUNBASE_PASSWORD", os.environ["NB_PW"])):
    raw = raw.replace("${%s}" % k, v)
cfg = json.loads(raw)
cfg.pop("_comment", None)
# 잡 이름에 label 을 실어둡니다 — 나중에 이 Flow 만 골라 끄기 위해서입니다.
cfg["description"] = label

def api(path, body=None, method="GET"):
    data = json.dumps(body).encode() if body is not None else None
    r = urllib.request.Request(base + path, data=data, method=method)
    r.add_header("Authorization", "Bearer " + tok)
    r.add_header("Content-Type", "application/json")
    return json.load(urllib.request.urlopen(r, timeout=60))

try:
    d = api("/v1/api/sql", {"sql": "SUBMIT STREAMING " + json.dumps(cfg)}, "POST")
    if d.get("status") != "ok":
        print("ERR:" + str(d)[:300]); raise SystemExit
    want = "streaming-" + (d.get("queryId") or "")[:8]
    for _ in range(20):
        for j in api("/v1/api/job/list"):
            if j.get("type") == "STREAMING" and j.get("jobName") == want:
                print(j["jobId"]); raise SystemExit
        time.sleep(1)
    print("ERR:job never appeared")
except SystemExit:
    pass
except Exception as e:
    print("ERR:" + str(e)[:300])
PY
}

log "1/2  이전 그래프 Flow 정리"
kill_graph_flows

log "2/2  Flow 기동"
for spec_label in "graph_nodes.json:graph-nodes" "graph_serving.json:graph-edges"; do
  spec=${spec_label%%:*}; label=${spec_label##*:}
  jid=$(submit "$DEMO/schema/flows/$spec" "$label")
  case "$jid" in ERR:*|"") fail "$label 제출 실패 ($jid)";; *) step "$label 기동 $jid";; esac
done

log "반영 확인 — NeorunBase 쪽 행 수"
for i in $(seq 1 40); do
  N=$(PGPASSWORD="${NEORUNBASE_PASSWORD:-Regdemo12345}" psql -h localhost -p 5434 -U admin \
        -d neorunbase -t -A -c "SELECT count(*) FROM doc_nodes" 2>/dev/null | head -1)
  E=$(PGPASSWORD="${NEORUNBASE_PASSWORD:-Regdemo12345}" psql -h localhost -p 5434 -U admin \
        -d neorunbase -t -A -c "SELECT count(*) FROM doc_edges" 2>/dev/null | head -1)
  [ "${N:-0}" -gt 0 ] && [ "${E:-0}" -gt 0 ] && break
  sleep 3
done
step "doc_nodes=${N:-0}  doc_edges=${E:-0}"
[ "${E:-0}" -gt 0 ] || fail "엣지가 서빙에 도달하지 않았습니다 — 순회 리트리버와 온톨로지 GRAPH 링크가 빈 결과를 냅니다"

echo
echo "  Flow 는 계속 돕니다. 배치 잡이 관계를 다시 만들면 서빙이 따라옵니다."
echo "  멈추려면:  bash infra/graph_flow.sh stop"
```


```bash
bash infra/graph_flow.sh      # before the pipeline
```

!!! warning "There is an order here"
    `changelog` mode sees changes **from the moment it starts**. Run the pipeline
    first and start the Flow afterwards, and the relations already built never
    reach serving. Start the Flows, then let the pipeline fill them.

Check it:

```bash
psql -h localhost -p 5434 -U admin -d neorunbase -c "SELECT count(*) FROM doc_nodes"  # 50
psql -h localhost -p 5434 -U admin -d neorunbase -c "SELECT count(*) FROM doc_edges"  # 116
```

![The Flow page](../images/demo/ontul-flow.png)

---

## 3. The approval event stream

What actually makes a regulation take effect is an approval row changing. This
Flow keeps `ice.reg.approval_status` upserted from those events.

**`demo/schema/flows/approval_status.json`**

```json
{
  "_comment": [
    "결재 이벤트 스트림 → ice.reg.approval_status 상시 upsert.",
    "",
    "source: 결재 시스템이 단계마다 newline-JSON 한 줄을 S3 프리픽스에 떨굽니다.",
    "  실제 조직에서는 Kafka 인 경우가 많지만, 이 데모는 파일 소스로 둡니다 —",
    "  브로커를 하나 더 띄우지 않고도 Flow 의 성질(증분 소비, 체크포인트, 재기동",
    "  후 이어받기)이 그대로 드러나기 때문입니다.",
    "",
    "sink: type=table 이 Iceberg 싱크입니다. upsertKeys 가 approval_id 이므로",
    "  같은 결재건의 후속 단계는 새 행이 아니라 **기존 행의 교체**가 됩니다.",
    "",
    "ontul.streaming.exactly.once: 마스터가 단일 커미터가 되어 모든 워커의",
    "  파일을 한 커밋으로 씁니다. upsert 와 함께 쓰면 데이터 파일과 equality",
    "  delete 파일이 같은 RowDelta 로 나가므로, 새 상태가 보이는 순간 이전",
    "  상태가 사라집니다 — 둘 다 보이는 중간 상태가 없습니다."
  ],
  "source": {
    "type": "file",
    "format": "json",
    "path": "s3://iceberg-warehouse/approval-events/",
    "s3.endpoint": "${S3_ENDPOINT_INTERNAL}",
    "s3.accessKey": "${S3_ACCESS_KEY}",
    "s3.secretKey": "${S3_SECRET_KEY}",
    "s3.pathStyle": "true",
    "s3.region": "${S3_REGION}",
    "maxFilesPerPoll": "50"
  },
  "operations": [],
  "sink": {
    "type": "table",
    "table": "ice.reg.approval_status",
    "upsertKeys": ["approval_id"]
  },
  "ontul.streaming.exactly.once": "true",
  "commitIntervalMs": 2000,
  "numWorkers": 1,
  "durationMs": 9999999999
}
```


!!! note "exactly-once together with upsert"
    The master becomes the single committer and writes every worker's files in one
    commit. Combined with upsert, the data files and the equality-delete files go
    out in the **same RowDelta**, so the previous state disappears at the instant
    the new one becomes visible — there is no window where both are readable.

**`demo/infra/flow.sh`**

```bash
#!/usr/bin/env bash
# 결재 스트림 Flow — ontul Flow 를 데모 스택에 얹습니다.
#
#   bash infra/flow.sh          # 테이블 + Flow 기동 + 1차 이벤트
#   bash infra/flow.sh advance  # 후속 단계 이벤트 (upsert 가 도는 것을 보여줍니다)
#   bash infra/flow.sh stop
#
# 배치 파이프라인(index.sh)과 다른 점은 한 가지입니다: 이건 끝나지 않습니다.
# 잡을 띄워두면 새 이벤트 파일이 도착하는 대로 표가 바뀝니다.
set -uo pipefail
cd "$(dirname "$0")/.."
DEMO=$(pwd)
. out/stack.env 2>/dev/null || { echo "out/stack.env 가 없습니다 — infra/up.sh 를 먼저 실행하세요"; exit 1; }

ONTUL=${ONTUL_URL:-http://localhost:8080}
ADMIN_PW="${ONTUL_ADMIN_PASSWORD:-regdemo-admin-2026}"
PREFIX=approval-events
BUCKET=$(echo "${S3_WAREHOUSE:-s3://iceberg-warehouse/}" | sed 's#^s3://##; s#/.*##')
MODE=${1:-start}

log(){ printf '\n\033[1;36m== %s\033[0m\n' "$*"; }
step(){ printf '\033[1;32m  ok\033[0m %s\n' "$*"; }
fail(){ printf '\033[1;31m  실패\033[0m %s\n' "$*"; exit 1; }

export AWS_ACCESS_KEY_ID=$S3_ACCESS_KEY AWS_SECRET_ACCESS_KEY=$S3_SECRET_KEY
export AWS_DEFAULT_REGION=${S3_REGION:-us-east-1}
aws configure set default.s3.addressing_style path 2>/dev/null
S3="aws --endpoint-url $S3_ENDPOINT_HOST s3"

TOK=$(curl -s -XPOST "$ONTUL/admin/auth/login" -H 'Content-Type: application/json' \
      -d "{\"username\":\"admin\",\"password\":\"$ADMIN_PW\"}" \
      | python3 -c "import sys,json;print(json.load(sys.stdin).get('accessToken',''))" 2>/dev/null)
[ -n "$TOK" ] || fail "ontul 로그인 실패"
AH="Authorization: Bearer $TOK"

# 상태 줄이 아니라 본문을 봅니다 — /admin/query/execute 는 실패해도 200 입니다.
sql(){ curl -s -XPOST "$ONTUL/admin/query/execute" -H "$AH" -H 'Content-Type: application/json' \
        -d "$(python3 -c "import json,sys;print(json.dumps({'sql':sys.argv[1]}))" "$1")"; }
sql_ok(){ local out; out=$(sql "$1")
  case "$out" in *'"status":"error"'*) fail "$2: $(echo "$out" | head -c 200)";; esac; step "$2"; }

# 자기 것만 죽입니다. 스트리밍 잡을 전부 죽이면 이 스크립트가 남의
# 파이프라인을 끕니다 — 실제로 그랬습니다: graph_flow 가 띄운 2개를 cdc 가
# 죽이고, cdc 가 띄운 5개를 flow 가 죽여서, 세 스크립트를 차례로 돌리면 마지막
# 하나만 살아남았습니다. 각 스크립트가 성공을 보고했기 때문에 아무도 눈치채지
# 못했습니다.
jobs_streaming(){ curl -s "$ONTUL/v1/api/job/list" -H "$AH" \
  | python3 -c "
import sys, json
for j in json.load(sys.stdin):
    if j.get('type') != 'STREAMING':
        continue
    tag = (j.get('description') or '') + ' ' + (j.get('jobName') or '')
    if 'approval-stream' in tag:
        print(j['jobId'])
" 2>/dev/null; }

if [ "$MODE" = "stop" ]; then
  log "Flow 중지"
  for jid in $(jobs_streaming); do curl -s -XPOST "$ONTUL/v1/api/job/kill/$jid" -H "$AH" -o /dev/null; step "kill $jid"; done
  exit 0
fi

# ── 이벤트 생성 ───────────────────────────────────────────────────────────────
# 실제 결재 건을 흉내냅니다. 대상 규정은 원장에 실재하는 문서라야 에이전트가
# 현행 버전과 나란히 놓고 답할 수 있습니다.
emit(){ # emit <seq> <<'JSON' ... JSON
  local seq=$1 tmp; tmp=$(mktemp); cat > "$tmp"
  $S3 cp "$tmp" "s3://$BUCKET/$PREFIX/evt-$(printf '%05d' "$seq").json" >/dev/null 2>&1 \
    && step "이벤트 배치 $seq 업로드" || fail "업로드 실패"
  rm -f "$tmp"
}

if [ "$MODE" = "advance" ]; then
  log "후속 단계 이벤트 — 같은 결재건이 다음 단계로"
  # 같은 approval_id 입니다. append 라면 행이 늘어나고, upsert 라면 교체됩니다.
  emit 2 <<'JSON'
{"approval_id":"APV-2026-0142","doc_no":"HR-REG-003","version":4,"step":"APPROVED","step_seq":3,"drafter":"20170003","owner_dept":"HR","summary":"육아지원규정 제4차 개정 — 돌봄휴가 20일→25일 확대","expected_from":"2026-09-01","updated_at":"2026-08-21 14:20:00"}
{"approval_id":"APV-2026-0151","doc_no":"SEC-GDL-001","version":3,"step":"REJECTED","step_seq":3,"drafter":"20190004","owner_dept":"SEC","summary":"보안점검 주기 단축 개정안","expected_from":"2026-10-01","updated_at":"2026-08-21 14:25:00"}
JSON
  log "확인"
  sleep 8
  sql "SELECT approval_id, doc_no, step, step_seq FROM ice.reg.approval_status ORDER BY 1" \
    | python3 -c "
import sys,json; d=json.load(sys.stdin); rows=d.get('rows') or []
print('  현재 결재 상태:')
for r in rows: print('   ', r)
print('  행 수:', len(rows), '(3건이면 upsert, 5건이면 append 로 격하된 것입니다)')"
  exit 0
fi

log "1/4  대상 테이블"
# 줄 앞이 아니라 줄 어디에 있든 -- 뒤를 잘라냅니다. 컬럼 뒤 주석이 그대로
# 남으면 파서는 다음 컬럼까지 주석으로 삼켜 "Failed to parse CREATE TABLE" 만
# 돌려줍니다 — 어느 줄이 문제인지는 알려주지 않습니다.
# 끝의 세미콜론도 뗍니다. ontul 의 CREATE TABLE 파서는 SELECT 와 달리 그것을
# 받지 못하고 "Failed to parse CREATE TABLE statement" 만 돌려줍니다.
DDL=$(sed 's/--.*$//' "$DEMO/schema/iceberg/40_approval_stream.sql" | tr '\n' ' ' | sed 's/  */ /g; s/^ *//; s/ *$//; s/;$//')
sql_ok "$DDL" "ice.reg.approval_status 준비"

log "2/4  기존 Flow 정리 + 이벤트 프리픽스 비우기"
for jid in $(jobs_streaming); do curl -s -XPOST "$ONTUL/v1/api/job/kill/$jid" -H "$AH" -o /dev/null; step "이전 잡 kill $jid"; done
$S3 rm "s3://$BUCKET/$PREFIX/" --recursive >/dev/null 2>&1 || true
sql "DELETE FROM ice.reg.approval_status" >/dev/null 2>&1
step "s3://$BUCKET/$PREFIX/ 비움"

log "3/4  Flow 제출"
JOBID=$(TOK="$TOK" ONTUL="$ONTUL" SPEC="$DEMO/schema/flows/approval_status.json" \
        EP="$S3_ENDPOINT_INTERNAL" AK="$S3_ACCESS_KEY" SK="$S3_SECRET_KEY" RG="${S3_REGION:-us-east-1}" \
        python3 <<'PY'
import os,json,urllib.request,time,re
tok=os.environ["TOK"]; base=os.environ["ONTUL"]
raw=open(os.environ["SPEC"],encoding="utf-8").read()
# 스펙은 그대로 두고 자격증명만 주입합니다 — 파일에 키를 적어 커밋하지 않기 위해서입니다.
for k,v in (("S3_ENDPOINT_INTERNAL",os.environ["EP"]),("S3_ACCESS_KEY",os.environ["AK"]),
            ("S3_SECRET_KEY",os.environ["SK"]),("S3_REGION",os.environ["RG"])):
    raw=raw.replace("${%s}"%k, v)
cfg=json.loads(raw); cfg.pop("_comment",None)
def api(path, body=None, method="GET"):
  data=json.dumps(body).encode() if body is not None else None
  r=urllib.request.Request(base+path, data=data, method=method)
  r.add_header("Authorization","Bearer "+tok); r.add_header("Content-Type","application/json")
  return json.load(urllib.request.urlopen(r,timeout=60))
try:
  cfg["description"] = "approval-stream"
  d=api("/v1/api/sql", {"sql":"SUBMIT STREAMING "+json.dumps(cfg)}, "POST")
  if d.get("status")!="ok": print("ERR:"+str(d)[:300]); raise SystemExit
  want="streaming-"+(d.get("queryId") or "")[:8]
  for _ in range(20):
    for j in api("/v1/api/job/list"):
      if j.get("type")=="STREAMING" and j.get("jobName")==want: print(j["jobId"]); raise SystemExit
    time.sleep(1)
  print("ERR:job never appeared")
except SystemExit: pass
except Exception as e: print("ERR:"+str(e)[:300])
PY
)
case "$JOBID" in ERR:*|"") fail "Flow 제출 실패 ($JOBID)";; *) step "Flow 기동 $JOBID";; esac

log "4/4  1차 이벤트 — 결재 3건이 각각 다른 단계에 있습니다"
emit 1 <<'JSON'
{"approval_id":"APV-2026-0142","doc_no":"HR-REG-003","version":4,"step":"REVIEW","step_seq":2,"drafter":"20170003","owner_dept":"HR","summary":"육아지원규정 제4차 개정 — 돌봄휴가 20일→25일 확대","expected_from":"2026-09-01","updated_at":"2026-08-21 11:05:00"}
{"approval_id":"APV-2026-0151","doc_no":"SEC-GDL-001","version":3,"step":"REVIEW","step_seq":2,"drafter":"20190004","owner_dept":"SEC","summary":"보안점검 주기 단축 개정안","expected_from":"2026-10-01","updated_at":"2026-08-21 11:12:00"}
{"approval_id":"APV-2026-0163","doc_no":"FIN-GDL-001","version":2,"step":"DRAFT","step_seq":1,"drafter":"20210001","owner_dept":"FIN","summary":"국내출장비 일비 인상","expected_from":"2026-11-01","updated_at":"2026-08-21 11:30:00"}
JSON

for i in $(seq 1 30); do
  n=$(sql "SELECT count(*) FROM ice.reg.approval_status" \
      | python3 -c "import sys,json;r=(json.load(sys.stdin).get('rows') or [[0]]);print(r[0][0] if r else 0)" 2>/dev/null)
  [ "${n:-0}" = "3" ] && break; sleep 3
done
sql "SELECT approval_id, doc_no, version, step FROM ice.reg.approval_status ORDER BY 1" \
  | python3 -c "
import sys,json; d=json.load(sys.stdin); rows=d.get('rows') or []
print('  진행 중인 개정:')
for r in rows: print('   ', r)"

log "완료 — Flow 는 계속 돕니다"
echo "  다음 단계를 흘려보내려면:  bash infra/flow.sh advance"
echo "  멈추려면:                  bash infra/flow.sh stop"
```


```bash
bash infra/flow.sh            # start, plus a first batch of events
bash infra/flow.sh advance    # later stages — watch the upsert happen
bash infra/flow.sh stop
```

This is the table the agent's `pending_revision` tool reads. The regulations
tables hold only what is **already** in force, so an approved revision taking
effect next month is invisible to them — which is exactly where an answer is
correct today and wrong for the decision being made.

---

Next: [IAM and retrievers](iam.md) — why the same question gets different answers.
