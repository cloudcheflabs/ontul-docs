# The ontology — objects, links, actions

A semantic view curates *metrics*; a retriever curates *retrieval*. The ontology
curates **entities and what can be done to them**. Three places show the
difference in this demo.

1. **The agent writes no SQL.** "Tell me about HR-REG-003" is
   `{"filters":{"doc_no":"HR-REG-003"}}` against an object type. Which catalog,
   which table, which join — none of it has to be known. Referencing an undeclared
   property is rejected with a 400 **before** any SQL is built.
2. **The same relationship is followed two ways.** `has_version` is a JOIN: a
   regulation and its versions share a key and the engine resolves it in SQL.
   `derives_from` is a GRAPH: authority relations are edges in NeorunBase's
   instance graph, and the graph engine walks them.
3. **There is a write.** `request_revision` records a revision request in the
   ledger — the system of record, not the derived layer — with validation,
   authorization, idempotency and audit all applied.

![The ontology graph](../images/demo/ontul-ontology-graph.png)

**`demo/schema/ontology/README.md`**

```markdown
# The ontology — objects · links · actions

A semantic view curates *metrics*; a retriever curates *retrieval*. The ontology
curates **the entities and what can be done to them**. Three places show the
difference in this demo.

**1. The agent does not have to write SQL.** "Tell me about HR-REG-003" becomes
`object-types/reg.ontology.Regulation/query` with
`{"filters":{"doc_no":"HR-REG-003"}}`. Column names, joins and which catalog it
lives in are all unnecessary. The name is `doc_no`, not `documents.doc_no`, and
the server knows the mapping.

**2. The same relationship is followed two ways.** `has_version` is a JOIN — a
regulation and its versions share a key and the engine resolves it in SQL.
`derives_from` is a GRAPH — which regulation a guideline takes its authority from
is an edge in NeorunBase's instance graph, and the graph engine does the walking.
Not an application-side loop.

**3. There is a write.** `request_revision` records a revision request in the
ledger (Iceberg). It writes to the record of fact rather than the derived layer —
NeorunBase is a rebuildable serving copy, and a record left somewhere that
disappears on a rebuild is not a record. IAM applies unchanged, so someone who
cannot read a regulation cannot request a revision to it either.

| File | Contents |
|---|---|
| `01_object_regulation.json` | `Regulation` — reads `semantic.reg.doc_index` |
| `02_object_version.json` | `RegulationVersion` — reads `semantic.reg.version_history` |
| `03_object_employee.json` | `Employee` — reads `semantic.hr.employees` (ERP, arrived by CDC) |
| `04_object_regulation_node.json` | `RegulationNode` — the graph vertex, in NeorunBase |
| `10_link_has_version.json` | `Regulation ─has_version→ RegulationVersion` (JOIN) |
| `11_link_derives_from.json` | `RegulationNode ─derives_from→ RegulationNode` (GRAPH) |
| `12_link_vertex.json` | `Regulation ─vertex→ RegulationNode` (JOIN) |
| `20_action_request_revision.json` | The revision request — DML into Iceberg |
```


---

## Object types

![Object types](../images/demo/ontul-object-types.png)

### Regulation

**`demo/schema/ontology/01_object_regulation.json`**

```json
{
  "_comment": "One regulation. It reads a governed semantic view rather than the raw ledger, so that the policy exists in exactly one place. Reading the ledger directly would mean writing the classification condition and the masking a second time on the ontology side — and the moment there are two copies they diverge. They did: restricted regulations leaked through the object path.\n\ndoc_no is the business key and doc_id is the graph vertex id: derives_from (a GRAPH binding) traverses from a numeric vertex, so the object is found by doc_no and then traversed from its doc_id.",
  "catalog": "reg",
  "schema": "ontology",
  "name": "Regulation",
  "readSource": "semantic.reg.doc_index",
  "primaryKey": [
    "doc_no"
  ],
  "properties": [
    {
      "name": "doc_id",
      "type": "long",
      "column": "doc_id",
      "synonyms": [
        "정점 id",
        "graph vertex id"
      ]
    },
    {
      "name": "doc_no",
      "type": "string",
      "column": "doc_no",
      "synonyms": [
        "규정번호",
        "문서번호"
      ]
    },
    {
      "name": "title",
      "type": "string",
      "column": "title",
      "synonyms": [
        "제목",
        "규정명"
      ]
    },
    {
      "name": "tier",
      "type": "long",
      "column": "tier",
      "synonyms": [
        "위계",
        "단계"
      ]
    },
    {
      "name": "owner_dept",
      "type": "string",
      "column": "owner_dept",
      "synonyms": [
        "소관부서",
        "주관부서"
      ]
    },
    {
      "name": "is_official",
      "type": "boolean",
      "column": "is_official",
      "synonyms": [
        "공식",
        "정식"
      ]
    },
    {
      "name": "sensitivity",
      "type": "string",
      "column": "sensitivity",
      "synonyms": [
        "민감도",
        "등급"
      ]
    }
  ],
  "tags": [
    "regulation",
    "governed"
  ],
  "status": "CERTIFIED"
}
```


`doc_no` is the business key and `doc_id` is the graph vertex id. Note that the
object reads a **governed semantic view**, not the raw ledger. Reading the ledger
directly would mean writing the classification condition a second time, and the
two copies diverged the moment they existed: restricted regulations leaked through
the object path while search correctly refused them.

### RegulationVersion

**`demo/schema/ontology/02_object_version.json`**

```json
{
  "_comment": "One version of a regulation. effective_from is the approval date, not the date printed in the document — this whole demo rests on that distinction. It reads a semantic view, so the classification condition applies here too.",
  "catalog": "reg",
  "schema": "ontology",
  "name": "RegulationVersion",
  "readSource": "semantic.reg.version_history",
  "primaryKey": [
    "doc_no",
    "version"
  ],
  "properties": [
    {
      "name": "doc_no",
      "type": "string",
      "column": "doc_no",
      "synonyms": [
        "규정번호"
      ]
    },
    {
      "name": "title",
      "type": "string",
      "column": "title",
      "synonyms": [
        "제목"
      ]
    },
    {
      "name": "version",
      "type": "long",
      "column": "version",
      "synonyms": [
        "판",
        "버전"
      ]
    },
    {
      "name": "status",
      "type": "string",
      "column": "status",
      "synonyms": [
        "상태",
        "시행/폐지"
      ]
    },
    {
      "name": "effective_from",
      "type": "date",
      "column": "effective_from",
      "synonyms": [
        "시행일",
        "발효일",
        "승인일"
      ]
    },
    {
      "name": "effective_to",
      "type": "date",
      "column": "effective_to",
      "synonyms": [
        "종료일",
        "폐지일"
      ]
    },
    {
      "name": "stated_from",
      "type": "date",
      "column": "stated_from",
      "synonyms": [
        "부칙 시행일"
      ]
    },
    {
      "name": "date_mismatch",
      "type": "boolean",
      "column": "date_mismatch",
      "synonyms": [
        "시행일 불일치"
      ]
    },
    {
      "name": "approval_id",
      "type": "string",
      "column": "approval_id",
      "synonyms": [
        "결재번호"
      ]
    },
    {
      "name": "sensitivity",
      "type": "string",
      "column": "sensitivity",
      "synonyms": [
        "민감도"
      ]
    }
  ],
  "tags": [
    "regulation",
    "temporal"
  ],
  "status": "CERTIFIED"
}
```


### Employee — the ERP copy that arrived by CDC

**`demo/schema/ontology/03_object_employee.json`**

```json
{
  "_comment": "An employee. The ERP copy that arrived by CDC into Iceberg, read through a semantic view rather than the ledger: that view carries the employee-number row filter and the national-ID deny. Opening the ledger directly would make the ontology path a back door around those policies.",
  "catalog": "reg",
  "schema": "ontology",
  "name": "Employee",
  "readSource": "semantic.hr.employees",
  "primaryKey": [
    "사번"
  ],
  "properties": [
    {
      "name": "사번",
      "type": "string",
      "column": "사번",
      "synonyms": [
        "emp_no",
        "사원번호"
      ]
    },
    {
      "name": "성명",
      "type": "string",
      "column": "성명",
      "synonyms": [
        "emp_nm",
        "이름"
      ]
    },
    {
      "name": "부서",
      "type": "string",
      "column": "부서",
      "synonyms": [
        "dept",
        "부서코드"
      ]
    },
    {
      "name": "직급",
      "type": "string",
      "column": "직급",
      "synonyms": [
        "grade"
      ]
    },
    {
      "name": "연락처",
      "type": "string",
      "column": "연락처",
      "synonyms": [
        "mobile"
      ],
      "pii": true
    }
  ],
  "tags": [
    "hr",
    "pii"
  ],
  "status": "CERTIFIED"
}
```


### RegulationNode — the same regulation, on the graph

**`demo/schema/ontology/04_object_regulation_node.json`**

```json
{
  "_comment": "The same regulation as it appears on the graph — a serving-side vertex. It points at the same entity as Regulation but reads a different place: Regulation reads the ledger (Iceberg) and this reads NeorunBase's doc_nodes. They are separate because GRAPH traversal joins the neighbouring vertices to the target table inside NeorunBase, and an Iceberg table cannot be found there — the result is 'Table not found: ice.reg.documents'. The vertex id is the doc_id the ledger assigned, so the two objects refer to each other by the same number.",
  "catalog": "reg",
  "schema": "ontology",
  "name": "RegulationNode",
  "readSource": "nb.public.doc_nodes",
  "primaryKey": [
    "doc_id"
  ],
  "properties": [
    {
      "name": "doc_id",
      "type": "long",
      "column": "doc_id",
      "synonyms": [
        "정점 id"
      ]
    },
    {
      "name": "doc_no",
      "type": "string",
      "column": "doc_no",
      "synonyms": [
        "규정번호"
      ]
    },
    {
      "name": "title",
      "type": "string",
      "column": "title",
      "synonyms": [
        "제목"
      ]
    },
    {
      "name": "tier",
      "type": "long",
      "column": "tier",
      "synonyms": [
        "위계"
      ]
    },
    {
      "name": "owner_dept",
      "type": "string",
      "column": "owner_dept",
      "synonyms": [
        "소관부서"
      ]
    },
    {
      "name": "is_official",
      "type": "boolean",
      "column": "is_official",
      "synonyms": [
        "공식"
      ]
    },
    {
      "name": "sensitivity",
      "type": "string",
      "column": "sensitivity",
      "synonyms": [
        "민감도"
      ]
    }
  ],
  "tags": [
    "regulation",
    "graph"
  ],
  "status": "CERTIFIED"
}
```


!!! note "Why a regulation is two object types"
    GRAPH traversal joins the neighbouring vertices to the target table **inside
    NeorunBase**. An Iceberg table cannot be found there — the result is
    `Table not found: ice.reg.documents`. So the graph-side regulation reads the
    serving table, and the two objects point at each other through the same
    `doc_id`.

---

## Link types

![Link types](../images/demo/ontul-link-types.png)

### has_version — JOIN

**`demo/schema/ontology/10_link_has_version.json`**

```json
{
  "_comment": "A regulation and its versions. They share a key, so this resolves as a JOIN — the engine builds the SQL and the graph engine is not involved.",
  "catalog": "reg",
  "schema": "ontology",
  "name": "has_version",
  "fromObjectType": "reg.ontology.Regulation",
  "toObjectType": "reg.ontology.RegulationVersion",
  "cardinality": "ONE_TO_MANY",
  "binding": {
    "mode": "JOIN",
    "fromKey": "doc_no",
    "toKey": "doc_no"
  }
}
```


```bash
curl -s -X POST "$ONTUL/api/v1/link-types/reg.ontology.has_version/traverse" \
  -H "Authorization: Bearer $TOK" -H 'Content-Type: application/json' \
  -d '{"sourceKey":"HR-REG-003","select":["version","status","effective_from"],"limit":10}'
```

```json
{
  "fqn": "reg.ontology.has_version",
  "mode": "JOIN",
  "columns": ["doc_no", "version", "status", "effective_from"],
  "rows": [["HR-REG-003", 3, "EFFECTIVE", 20162], ["HR-REG-003", 2, "SUPERSEDED", 19875]],
  "sql": "SELECT t.\"doc_no\" AS \"doc_no\", … FROM semantic.reg.version_history t WHERE t.\"doc_no\" = 'HR-REG-003' LIMIT 10"
}
```

The response carries the SQL that ran. That matters: the ontology is not a layer
that hides SQL, it is one that **writes** it, and what it wrote has to be
reviewable.

### derives_from — GRAPH

**`demo/schema/ontology/11_link_derives_from.json`**

```json
{
  "_comment": "Which regulation a guideline derives its authority from. This cannot be resolved by a join: the authority relation was extracted from the body text and written as edges into NeorunBase's instance graph, and answering takes several hops. The graph engine walks it; the application only receives the result. Both endpoints are RegulationNode because traversal joins the neighbouring vertices to the target table inside NeorunBase, and an Iceberg table cannot be found there.",
  "catalog": "reg",
  "schema": "ontology",
  "name": "derives_from",
  "fromObjectType": "reg.ontology.RegulationNode",
  "toObjectType": "reg.ontology.RegulationNode",
  "cardinality": "MANY_TO_MANY",
  "binding": {
    "mode": "GRAPH",
    "graphCatalog": "nb",
    "edgeTable": "doc_edges",
    "edgeLabel": "CHILD_OF",
    "direction": "OUT"
  }
}
```


```bash
# find the object by doc_no, then traverse from its doc_id
curl -s -X POST "$ONTUL/api/v1/link-types/reg.ontology.derives_from/traverse" \
  -H "Authorization: Bearer $TOK" -H 'Content-Type: application/json' \
  -d '{"sourceKey":"22","maxDepth":2,"select":["doc_no","title","tier"],"limit":20}'
```

```json
{
  "mode": "GRAPH",
  "columns": ["doc_no", "title", "tier"],
  "rows": [["HR-GDL-003", "징계 양정기준", 0], ["HR-REG-001", "인사규정", 0]],
  "sql": "SELECT t.doc_no AS doc_no, … FROM GRAPH_NEIGHBORS(edge_table => 'doc_edges', seed => 22, max_depth => 2, edge_filter => 'CHILD_OF') n JOIN public.doc_nodes t ON t.doc_id = n.id LIMIT 20"
}
```

The traversal runs on the `GRAPH_NEIGHBORS` TVF **inside NeorunBase**. It is not
an application loop fetching neighbours and querying again.

### vertex — the JOIN that connects the two

**`demo/schema/ontology/12_link_vertex.json`**

```json
{
  "_comment": "Connects the regulation in the ledger to the same regulation on the graph. A JOIN on doc_no. This is the path an agent takes to 'follow the authority behind HR-GDL-003': find the Regulation, get its vertex, then traverse derives_from.",
  "catalog": "reg",
  "schema": "ontology",
  "name": "vertex",
  "fromObjectType": "reg.ontology.Regulation",
  "toObjectType": "reg.ontology.RegulationNode",
  "cardinality": "ONE_TO_ONE",
  "binding": {
    "mode": "JOIN",
    "fromKey": "doc_no",
    "toKey": "doc_no"
  }
}
```


---

## Actions — a governed write

![Action types](../images/demo/ontul-action-types.png)

**`demo/schema/ontology/20_action_request_revision.json`**

```json
{
  "_comment": "Requesting a revision to a regulation. Four things separate this from the agent writing its own INSERT: the parameters are validated, the caller is authorized, the idempotency key makes a retry safe, and the invocation is audited. The requester is not a parameter for the same reason — if anyone can file under someone else's name, the table is not an audit record.",
  "catalog": "reg",
  "schema": "ontology",
  "name": "request_revision",
  "objectType": "reg.ontology.Regulation",
  "mode": "DML",
  "parameters": [
    {
      "name": "doc_no",
      "type": "STRING",
      "required": true,
      "description": "The regulation to request a revision to"
    },
    {
      "name": "version",
      "type": "LONG",
      "required": true,
      "description": "The version currently in force"
    },
    {
      "name": "reason",
      "type": "STRING",
      "required": true,
      "description": "Why the revision is being requested"
    }
  ],
  "sqlTemplate": "INSERT INTO ice.reg.revision_requests (request_id, doc_no, version, reason, status) SELECT ${doc_no} || '-v' || CAST(${version} AS VARCHAR), ${doc_no}, CAST(${version} AS INTEGER), ${reason}, 'REQUESTED'",
  "allowedRoles": [
    "agent_caller_group",
    "dept_manager_group",
    "hr_staff_group"
  ],
  "requiresApproval": false,
  "tags": [
    "regulation",
    "write-back"
  ],
  "status": "CERTIFIED"
}
```


The target table.

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


### Invoking it

```bash
curl -s -X POST "$ONTUL/api/v1/action-types/reg.ontology.request_revision/invoke" \
  -H "Authorization: Bearer $TOK" -H 'Content-Type: application/json' \
  -d '{"args":{"doc_no":"HR-REG-003","version":3,"reason":"…"},
       "idempotencyKey":"demo-req-1"}'
```

```json
{"action":"reg.ontology.request_revision","ok":true,"mode":"DML","commandTag":"INSERT 1","elapsedMs":1391}
```

Called again with the same key, it **does not write again** — it returns the
earlier result.

```sql
SELECT request_id, doc_no, version, status FROM ice.reg.revision_requests;
-- HR-REG-003-v3 | HR-REG-003 | 3 | REQUESTED     ← one row, after two calls
```

Both calls are in the audit log.

```json
{"userId":"admin","action":"action:invoke",         "resource":"reg.ontology.request_revision","details":"via=REST"}
{"userId":"admin","action":"action:invoke:replayed","resource":"reg.ontology.request_revision","details":"via=REST idempotencyKey=demo-req-1"}
```

!!! note "Why the requester is not a parameter"
    An action's SQL template substitutes only declared parameters, so there is no
    slot for the session user. Taking `requested_by` as a parameter instead would
    let anyone file **under someone else's name** — a forgeable field sitting next
    to real ones is worse than no field at all. Who called and when is recorded by
    the audit log, which the caller cannot edit.

---

## From the agent

Three tools attach to the agent's tool set — two reads and a write. The full
source is on the [agent page](agent.md).

| Tool | Ontology path |
|---|---|
| `describe_regulation` | ObjectSet query + `has_version` traversal |
| `related_regulations` | `derives_from` GRAPH traversal |
| `request_regulation_revision` | the `request_revision` action |

---

## What this adds up to

Graph RAG is already in place: vectors, full-text and the graph in one engine, so
hybrid fusion happens in the engine rather than in application code. Four things
sit on top of that.

1. **Temporal correctness is structural, not a ranking weight.** A superseded
   regulation is not scored lower; it is absent from the view.
2. **Access control lives in the engine.** The same question returns different
   rows to different people.
3. **It federates.** "How much childcare leave do I have left?" is a number that
   needs both the regulation (a document) and the balance (ERP, arriving by CDC).
4. **It is not read-only.** Actions open a governed write.
---

Next: [the agent](agent.md).
