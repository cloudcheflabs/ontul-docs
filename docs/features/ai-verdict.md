# Typed Decisions in SQL (`ai_verdict`)

Ontul can ask a decision model a typed question about each row and return the
answer as a SQL value:

```sql
SELECT x.ticket_id, x.v.choice, x.v.confidence
FROM (
  SELECT ticket_id,
         ai_verdict_one(body, '{
           "type": "choice",
           "instructions": "Classify the support ticket",
           "criteria": {
             "refund": "asks for money back",
             "bug":    "reports a defect",
             "other":  "anything else"
           }}') AS v
  FROM support.public.tickets
) AS x
WHERE x.v.choice = 'refund' AND x.v.confidence > 0.7;
```

No model name appears in that query. Which endpoint answers it, which model,
the per-call timeout, how many requests a batch may have in flight, the retry
budget and whether the text may leave the cluster are all properties of a
`DECIDE` [connection](connection-id.md), editable in the Admin UI. Retargeting
from `jev-latest` to a pinned revision is a form submit, not a rewrite of every
query.

## What a decision model is, and is not

A decision model takes a state and a typed question and returns a decision, a
probability distribution and a confidence. It does not generate text. That is
what makes it usable inside a scan: a call is 70–500ms, which fits in a
projection's budget, where a generative model's seconds-per-call, prose output
and retry loop do not. The same reasoning keeps generative models out of a
retriever's [`rerank` block](retrievers.md), and `ontul.decide.max.timeout.ms`
is the ceiling that enforces it here.

The wire contract is [TypeSafe AI System One](https://docs.typesafe.ai/introduction),
verbatim:

```
POST {baseUrl}{path}
{ "state": "...", "model": "jev-latest",
  "questions": { "<id>": {"type":"choice","instructions":"...","criteria":{...}} } }

→ { "model": "jev-1.13.0", "answers": { "<id>": {...} }, "usage": {...} }
```

Keeping to the vendor's own contract is deliberate: it is the same contract
Databricks' `ai_decide` speaks, so a question written for one engine works
against the same endpoint from the other, and a self-hosted service that
implements the contract works from both.

## Question types

| Type | Answer fields | Shape of the question |
|------|---------------|-----------------------|
| `choice` | `choice`, `probabilities`, `confidence` | `criteria` maps each label to what it means. At most 255 labels. |
| `score` | `score`, `probabilities`, `confidence`, `legend` | `criteria` maps each rubric step to what it means. 2–10 steps. |
| `noul` | `noul` — a probability in 0–1 | `instructions` only; no criteria. |

Questions in one call are evaluated independently and in parallel by the
endpoint. The guidance from TypeSafe applies here too: split a complicated
judgement into atomic questions and combine them in SQL, rather than asking one
compound question.

## The two functions

### `ai_verdict_one(state, question [, options])` → `STRUCT`

One question, answered into a fixed struct. This is the form to reach for: it
returns typed values, so a decision works in `WHERE`, `GROUP BY`, a join key, a
`CASE`, or an `INSERT`.

| Field | Type | |
|-------|------|--|
| `type` | VARCHAR | `choice` / `score` / `noul` — which of the next three is set |
| `choice` | VARCHAR | the label, for a `choice` question |
| `score` | INTEGER | the rubric step, for a `score` question |
| `noul` | DOUBLE | the probability, for a `noul` question |
| `confidence` | DOUBLE | the model's confidence in its own answer |
| `probabilities` | VARCHAR | the distribution, as JSON (the label set is the question's, so there is no fixed column shape for it) |
| `applied` | BOOLEAN | whether the endpoint answered |
| `reason` | VARCHAR | why not, when `applied` is false |
| `model` | VARCHAR | the model that **answered** — a connection asking for `jev-latest` gets back a pinned version, and which one decided is what an audit can use |
| `latency_ms` | BIGINT | wall time for this row's request; `0` on a cache hit |

The shape is fixed rather than derived from the question. A return type that
changed with the contents of a string literal would be a type no reviewer could
read off the SQL. Exactly one of `choice` / `score` / `noul` is set, and `type`
says which.

### `ai_verdict(state, questions [, options])` → `VARCHAR`

Several questions in one request, returned as the System One response envelope
in JSON text:

```sql
SELECT ai_verdict(body, '{
         "tone":   {"type":"choice","instructions":"Pick the tone",
                    "criteria":{"angry":"hostile","calm":"polite"}},
         "urgent": {"type":"noul","instructions":"Does this need a human today?"}
       }') AS v
FROM support.public.tickets;
```

```json
{
  "model": "jev-1.13.0",
  "answers": {
    "tone":   {"choice":"angry","probabilities":{"angry":0.9,"calm":0.1},"confidence":0.91},
    "urgent": {"noul":0.95}
  },
  "decide": {"applied":true,"connection":"jev","latencyMs":84,"cached":false}
}
```

`model` and `answers` are the vendor's keys carrying the vendor's shape, so a
client that already parses a System One response finds what it expects.
`decide` is the part only Ontul can report. The value is JSON text and not a
`VARIANT` — Ontul has no `VARIANT` type, and adding one to read a classifier's
output would be a large change for a small reason — so this form is for a
client or an SDK that parses it, while `ai_verdict_one` is the form to use
inside SQL.

### `options`

A literal JSON object. Two keys:

| Key | |
|-----|--|
| `connection` | the `DECIDE` connection to use. Omitted, the one marked `default=true` is used — which is what lets Databricks SQL port over with no edit. |
| `acknowledgeEgress` | required when the connection is marked `external=true`. |

Everything else — timeout, concurrency, retries, failure mode, model — is a
property of the connection, so that an operator tuning an endpoint does not
have to find and edit every query that uses it. Putting one of those keys in
`options` is an error that names where it belongs rather than being ignored.

### `questions` must be a literal

As in Databricks' `ai_decide`. Not an implementation shortcut: a constant is
what lets a malformed question be rejected before the scan starts, what the
result cache keys on, and what keeps a per-row question — which would make cost
unpredictable and a plan unreviewable — out of the engine.

!!! note "Apostrophes in a question"
    Ontul's plan validator rejects any string literal containing a single
    quote, engine-wide. Write instructions without apostrophes ("the customer
    is at fault" rather than "it's the customer's fault") until that limit is
    lifted.

## Configuring the endpoint (Admin UI)

**Connections → New Connection → DECIDE.** Every field below is on the form.

| Property | Default | |
|----------|---------|--|
| `baseUrl` | — | required, e.g. `https://api.typesafe.ai` |
| `path` | `ontul.decide.default.path` | `/v1/systemone` as shipped |
| `model` | `ontul.decide.default.model` | `jev-latest` as shipped; what the request asks for |
| `authType` | `NONE` | `BEARER` / `HEADER` / `BASIC` — the same modes as a `REST` connection |
| `apiKey` | — | masked in the connections list, encrypted at rest by the KMS. Name it `apiKey` or `secret`; a property named `token` would be shown in clear. |
| `enabled` | `true` | per-endpoint off switch that keeps the configuration. Off, every call reports `applied=false`, `reason=connection-disabled`, and nothing is sent. |
| `default` | `false` | the endpoint a call with no `connection` option uses. Marking two is refused — resolving it by iteration order would let the same query answer differently depending on which node planned it. |
| `external` | `false` | the state text leaves the cluster. Every call must then pass `acknowledgeEgress`. |
| `onFailure` | `ontul.decide.default.on.failure` | `PASSTHROUGH` answers with `applied=false` and a reason; `FAIL` fails the query. |
| `timeoutMs` | `ontul.decide.default.timeout.ms` | per call, clamped to `ontul.decide.max.timeout.ms` |
| `maxConcurrency` | `ontul.decide.default.concurrency` | requests in flight per batch, clamped to `ontul.decide.max.concurrency` |
| `retries` | `ontul.decide.default.retries` | timeouts, `429` and `5xx` only, clamped to `ontul.decide.max.retries` |
| `cacheSize` | `ontul.decide.default.cache.size` | result cache entries; `0` disables it |
| `maxStateChars` | `ontul.decide.default.state.chars` | per-row truncation, clamped to `ontul.decide.max.state.chars` |

Every default above is a cluster property, not a literal in the code. A
deployment whose endpoint is slower, or whose corpus repeats less, moves the
starting point for *every new connection* in one place rather than editing each
one. The shipped values are listed under
[Cluster settings](#cluster-settings).

Everything operational lives on the connection rather than in
`ontul.properties` for one concrete reason: the call is evaluated **on a
worker**, and connections are replicated to workers (`CONN_SYNC_PUSH`) and
editable from the Admin UI. A property file would need a rolling restart to
raise a timeout. An edit here takes effect on the next batch.

### Probing an endpoint

The **⚡** button on a `DECIDE` connection asks the endpoint one real question —
the cheapest the contract allows — and reports whether it answered, which model
answered, and how long it took against the connection's own timeout. When it
did not answer, it reports the same named reason `ai_verdict()` would put in its
`reason` column. It is the same code path as a real call, so a probe that passes
means a real call can.

```bash
curl -s -X POST http://localhost:8080/admin/connections/test \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"connectionId":"jev"}'
# {"ok":true,"url":"https://api.typesafe.ai/v1/systemone",
#  "requestedModel":"jev-latest","model":"jev-1.13.0","latencyMs":84,...}
```

## Cost and latency

A call per row would make this unusable. Three things cut the request count
before a socket opens, in order:

1. **The whole Arrow batch goes out together**, at the connection's
   `maxConcurrency`. The endpoint evaluates each question independently anyway,
   so there is nothing to serialise.
2. **Identical states in one batch collapse to one request.** Classification
   pipelines are full of duplicate states — the same product description, the
   same policy paragraph — and each one is a request someone pays for.
3. **A bounded result cache** answers repeats across batches, keyed on the
   state, the questions *and* the endpoint identity: the same state decided by a
   different model is a different answer.

The cache also pays for something less obvious. Reading several fields of one
verdict — `x.v.choice, x.v.confidence, x.v.applied` — becomes several calls
after the optimizer inlines the function per field. With the default
`cacheSize` that is one request; with `cacheSize=0` it is three.

Only answers are cached. Caching a timeout would make one bad second look like
a permanently broken endpoint for as long as the entry lived.

**A decision nobody reads is never paid for.** The projection is pruned like any
other, so a `COUNT(*)` over a sub-select that computes a verdict makes no
request at all. Useful to know before writing a query that measures row counts
over a classified set — and the reason the e2e's cache assertions `GROUP BY` the
decided column rather than counting rows.

**The cache is per worker, and deliberately so.** Each worker holds its own
client, its own concurrency pool and its own LRU, because a scan's splits are
shared between workers and nothing about a decision needs cross-worker
coherence. The practical consequence is arithmetic: a scan of *S* distinct
states spread over *W* workers costs up to `S × W` requests on the first pass,
not `S`. For a table whose distinct states are many and whose rows are few per
state, size `cacheSize` for what one worker sees, not for the table.

## Failure is reported, never nulled

A column of NULLs from a dead endpoint is indistinguishable from a model that
decided "none of these", and the difference is the whole reason anyone is
looking at the column. So every exit path is named:

| `reason` | |
|----------|--|
| `timeout` | the call exceeded the connection's `timeoutMs` |
| `http-<code>` | the endpoint answered non-2xx |
| `malformed-response` | a 2xx body that is not JSON |
| `no-answers` | a 2xx with no `answers` object — usually a proxy serving its own health page, the most confusing misconfiguration there is because nothing errors |
| `connection-disabled` | `enabled=false` on the connection |
| `disabled-cluster-wide` | `ontul.decide.enabled=false` |
| `connection-unusable` | the connection was deleted or retyped mid-query |
| `state-null` | the state was NULL — nothing to decide, and not a failure |
| `answer-missing:q` | the endpoint answered, but not the question that was asked |

`state-null` is deliberately not a failure: treating it as one would make
`onFailure=FAIL` unusable on any nullable column.

With `onFailure=FAIL`, every one of the above except `state-null` fails the
query instead. That is for callers where a missing decision is safer than a
query that quietly produced none — anything auto-actioned.

## Governance

**`AI:DECIDE` on `connection:<id>`.** `ai_verdict()` is the one SQL function
that spends money per row and sends text to a third party, so reading the table
is not enough. The endpoint is a resource of its own:

```json
{
  "Version": "2024-01-01",
  "Statement": [
    {
      "Sid": "AllowJevEndpoint",
      "Effect": "Allow",
      "Action": "AI:DECIDE",
      "Resource": "connection:jev"
    }
  ]
}
```

That gives the two controls the feature actually needs — a per-endpoint allow
list for who may bill the expensive model, and a per-endpoint deny for "nobody
sends data to this vendor from SQL" — without touching table grants. No
matching policy is a denial, as everywhere else in [IAM](iam.md). The check runs
on the master, before any plan reaches a worker.

The Admin UI policy editor ships both shapes — **Decision Endpoint
(ai_verdict)** and **Decision Endpoint (Allow One, Deny Another)**. See
[IAM Policy Templates](iam-policy-templates.md#decision-endpoints-ai_verdict).

Note what the grant does *not* cover: it authorizes the endpoint, not the data.
An identity still needs `data:Select` on every table the query scans.

**Egress acknowledgement.** When a connection is marked `external=true`, every
call must say so:

```sql
ai_verdict_one(body, '{...}', '{"connection":"jev","acknowledgeEgress":true}')
```

Ontul cannot decide this on an operator's behalf: the state is an arbitrary SQL
expression and carries no link to the `pii` flags on an object type's
properties, so there is nothing to infer from. Recording a human decision beats
pretending to a check. Same contract as a retriever's `rerank` block.

**Truncation.** `maxStateChars` bounds how much text can leave the cluster per
row, so a column that grows does not quietly start sending more.

## Cluster settings

`ontul.properties` holds only the ceilings a connection is clamped to, plus the
switch an operator reaches for during an incident.

```properties
# ---- Master switch ----
# false = every call reports applied=false with reason "disabled-cluster-wide"
# and nothing is sent. Stops the traffic, and the bill, without editing a
# single connection.
ontul.decide.enabled=true

# ---- Ceilings a connection is clamped to ----
# Upper bound on a connection's timeoutMs. This call sits inside a scan, so it
# is also the worst-case latency per distinct state in a batch.
ontul.decide.max.timeout.ms=5000
# Lower bound. A budget below this is almost always a typo -- a "5" meant as
# seconds -- and it would make every call time out while looking like the
# endpoint is at fault.
ontul.decide.min.timeout.ms=50
# Upper bound on maxConcurrency: requests in flight per Arrow batch.
ontul.decide.max.concurrency=16
# Upper bound on maxStateChars, the per-row truncation.
ontul.decide.max.state.chars=16000
# Most questions one ai_verdict() call may carry. Each one is billed.
ontul.decide.max.questions=32
# Upper bound on retries. A high ceiling multiplies every row's worst case.
ontul.decide.max.retries=3
# Upper bound on cacheSize, in entries. The cache lives in worker heap, so this
# is a memory bound as much as a cost one.
ontul.decide.max.cache.size=100000
# TCP connect timeout, separate from the per-call budget: an unreachable host
# should fail fast rather than burn the whole timeout on every row.
ontul.decide.http.connect.timeout.ms=1000
ontul.decide.http.max.connect.timeout.ms=10000

# ---- Question-shape limits from the System One contract ----
# Here rather than in the code so a vendor raising a cap does not need an Ontul
# release before anyone can use it.
ontul.decide.max.choice.labels=255
ontul.decide.min.score.steps=2
ontul.decide.max.score.steps=10

# ---- Defaults a DECIDE connection inherits for a blank field ----
ontul.decide.default.path=/v1/systemone
ontul.decide.default.model=jev-latest
ontul.decide.default.timeout.ms=1500
ontul.decide.default.concurrency=8
ontul.decide.default.retries=1
ontul.decide.default.cache.size=2000
ontul.decide.default.state.chars=4000
ontul.decide.default.on.failure=PASSTHROUGH
```

## Where it fits

- **Batch and streaming classification** — a `choice` or `score` per row,
  written to a column other queries filter on.
- **A gate after retrieval** — a `noul` on "can this question be answered from
  these documents" before an answer is produced, using the rows a
  [retriever](retrievers.md) already returned and already filtered by IAM.
- **Before an ontology action runs** — a decision on allow / needs-approval /
  block, wired to an action type's `requiresApproval`. See
  [Ontology](ontology.md).

## Compared with Databricks `ai_decide`

| | Ontul | Databricks |
|-|-------|------------|
| Multi-question form | `ai_verdict(state, questions [, options])` → VARCHAR (JSON) | `ai_decide(state, questions [, options])` → VARIANT |
| Typed single question | `ai_verdict_one(...)` → STRUCT | — |
| Endpoint | a `DECIDE` connection, named in `options` or marked default | workspace-configured |
| Per-endpoint authorization | `AI:DECIDE` on `connection:<id>` | — |
| Failure reporting | `applied` / `reason` per row, `PASSTHROUGH` or `FAIL` | — |
| Egress acknowledgement | required for `external=true` | — |

The question JSON and the endpoint are the same, so questions port directly. The
function name differs on purpose — `ai_decide` is Databricks', and two engines
answering the same name with different return types would be worse than two
names.

## Verifying it

`tests/e2e-ai-verdict.sh` asserts the behaviour on this page — 77 assertions —
against a mock System One endpoint. No third-party API is called.

The stack is **distributed and isolated**: two masters behind nginx, two
workers, its own bridge network, its own compose project and container names,
and host ports nothing else uses, so it runs alongside a Chango-installed Ontul
without collision. Two workers because `ai_verdict` is evaluated inside a
worker's projection operator: the client, the concurrency pool, the cache and
the replicated copy of the connection are all per-worker, and a single-worker
run exercises none of that.

Distribution is measured, not assumed. The fixture is four CSV files on a volume
every node shares — four files is four splits, which the planner shares between
the workers. Each file holds the same four bodies, which makes the request count
an arithmetic prediction: **4 distinct states × 2 workers = 8 requests**. Fewer
would mean a shared cache (there is none, and must not be); more would mean a
worker failed to dedupe its own splits. A model edit then re-decides on both,
measured as 8 again, since a worker left on the stale model would have served
four from its cache.

!!! note "Why not tpch for the distributed fixture"
    The `tpch` connector returns one split per table regardless of how many are
    asked for, so a tpch scan always lands on a single worker. It proves nothing
    about a distributed run, however many workers are up.

Covered: the three answer types; a verdict in a projection, in a `WHERE`, and
nested in another expression; batching, per-worker dedup and cache request
counts; projection pruning for an unread verdict; every failure reason;
`onFailure=FAIL`; `AI:DECIDE` allowed and denied per endpoint; the egress
acknowledgement; truncation; `BEARER` auth from the connection; retries; the
per-connection and cluster kill switches; and a live connection edit reaching
every worker.
