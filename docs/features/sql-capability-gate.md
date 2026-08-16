# SQL Capability Gate

Ontul validates every planned query against what the execution engine can actually evaluate, and
**rejects the rest with an explicit error** instead of running it. A query either produces a correct
answer or fails loudly — it never quietly returns wrong rows.

```
ERROR: Window functions (OVER) is not supported by the Ontul execution engine
ERROR: INTERSECT is not supported by the Ontul execution engine: rewrite as an inner join on the shared columns
ERROR: This JOIN condition is not supported by the Ontul execution engine: only a single equality is
       executable; this join has 2 conjuncts
```

## Why a gate exists

Ontul's master plans a query with Apache Calcite and ships the resulting expressions to workers as
**Calcite `RexNode` text**. Each worker re-parses that text into its own expression tree.

That design is what makes the plan small and language-neutral, but it has a sharp edge: a construct
the worker's parser does not recognise does not raise — it degrades. An unknown function evaluates to
`NULL`; an unrecognised literal becomes a string. Without a gate, an unsupported feature surfaces as a
**silently wrong answer**:

| Without the gate | What the user saw |
|------------------|-------------------|
| `SUM(x) OVER (PARTITION BY k)` | a column of `NULL` |
| `WHERE id = (SELECT MAX(id) FROM t)` | zero rows |
| `a EXCEPT b` | all of `a` |
| `a UNION b` | duplicates (behaved as `UNION ALL`) |
| `LEFT JOIN` | unmatched rows dropped (behaved as `INNER`) |
| `ON a.x = b.x AND a.y = b.y` | the second equality ignored → extra rows |
| `TRIM(name)` | `NULL` |

Wrong numbers that look plausible are far more expensive than a failed query, so the engine is
**fail-closed**: if it cannot prove it will evaluate a construct correctly, it refuses to run it.

## Where it runs

The gate runs on the master, immediately after planning and **before the plan is converted to text or
dispatched**. It inspects the `RelNode` tree, so it sees real column types, join conditions and
aggregate flags rather than re-parsed strings.

```
SQL → Calcite parse/validate → RelNode → [ capability gate ] → PhysicalPlan → workers
                                              │
                                              └─ UnsupportedSqlException → client error
```

Two consequences worth knowing:

- **Cached plans are already validated.** A plan-cache hit skips the gate because the miss that built
  it passed the gate.
- **`EXPLAIN` is gated too.** Explaining a query that cannot run reports the same error, so `EXPLAIN`
  never shows a plan the engine would not execute.

## What is rejected

| Construct | Rewrite |
|-----------|---------|
| Window functions (`OVER`) | Pre-aggregate, or rank client-side |
| Scalar subquery, `IN (SELECT …)`, `EXISTS`, correlated subqueries | A join, or materialise the inner query into a table first |
| `INTERSECT` | Inner join on the shared columns |
| `EXCEPT` / `MINUS` | Anti-join |
| `UNION` (de-duplicating) | `UNION ALL`, wrapped in `SELECT DISTINCT` if needed |
| `SEMI` / `ANTI` join | — |
| Join `ON` that is not one equality — composite (`AND`), non-equi (`>`), or a cross join | Keep one equality in `ON`, move the rest to `WHERE` |
| `COUNT(DISTINCT x)`, `SUM(DISTINCT x)` | `GROUP BY x`, then count the groups |
| `FILTER (WHERE …)`, `WITHIN GROUP`, `GROUPING SETS` / `CUBE` / `ROLLUP` | Separate queries, or `CASE WHEN` inside the aggregate's input |
| Aggregates other than `COUNT`/`SUM`/`AVG`/`MIN`/`MAX` (e.g. `STDDEV_POP`) | A [UDF](udf.md) |
| `SUM`/`AVG` over a non-numeric column | `CAST` first |
| `OFFSET` | `LIMIT` only; page with a `WHERE` range over a sorted key |
| Scalar functions with no implementation — `TRIM`, `EXTRACT`, `POWER`, `SQRT`, `CHAR_LENGTH`, `POSITION`, `CURRENT_DATE`, … | A [UDF](udf.md) |
| Unary minus (`-x`) | `(0 - x)` |
| `IN (...)` / `BETWEEN` in the **SELECT list** (they survive as `SEARCH(Sarg[…])`) | Explicit `OR` / `AND` comparisons. Both are fine in `WHERE` |
| Comparing a column against a `DATE` / `TIME` / `TIMESTAMP` / `INTERVAL` literal | Compare on the epoch value — these types travel through the engine as epoch numbers. A datetime literal as **data** (`INSERT … VALUES (DATE '2026-01-01')`) is fine |
| String literal containing `'` | — |
| `LIKE … ESCAPE` | Plain `LIKE`; `%` and `_` are the only wildcards, everything else is literal |
| Dynamic parameters (`?`) | Inline the value — the plan is shipped as text |

## What is *not* gated

- **`DELETE`, `UPDATE`, `MERGE INTO` predicates.** These never reach the operator pipeline: they are
  evaluated row-by-row by a separate evaluator that handles composite and non-equi conditions, typed
  literals and n-ary `AND`/`OR`. A `DELETE FROM t WHERE a = 1 AND b = 2 AND c = 3` or a `MERGE … ON
  t.id = s.id AND t.region = s.region` works exactly as before.
- **`CREATE VIEW`.** Only the view's output schema is derived at creation time. An Iceberg view is a
  catalog object other engines (Trino, Spark) also read, so creating one does not require the body to
  be Ontul-executable. The gate applies when the view is **queried** through Ontul.
- **Data lineage extraction**, which only reads scan nodes out of the plan.

## Configuration

```properties
# conf/ontul.properties
ontul.sql.strict=true
```

Resolved in the standard order — `-Dontul.sql.strict=…`, then the `ONTUL_SQL_STRICT` environment
variable, then `conf/ontul.properties`. Default `true`.

Setting it to `false` restores the previous behaviour, in which the queries listed above **run and
return wrong results**. It exists to triage a regression (to confirm a query is blocked by the gate
rather than by something else), never for production use. The master logs a warning on every
validation while it is off.

## Reading an error

The message always names the construct, and adds the rewrite when there is a mechanical one:

```
Comparing against a DATE / TIME / TIMESTAMP / INTERVAL literal is not supported by the Ontul
execution engine: the engine stores these as epoch numbers; compare on the epoch value instead
```

If a query you expect to work is rejected, that is a bug in the gate — the rule list is meant to be
exactly the set of constructs the engine mis-evaluates, no wider. Report the SQL and the message.

## Roadmap

The gate is deliberately a **capability contract, not a permanent limitation**. Each rule is removed
in the same change that adds real support — outer joins, non-numeric `MIN`/`MAX`, n-ary `AND`/`OR`
and regex-unsafe `LIKE` patterns were all gated at one point and are now executed. The next
candidates are `OFFSET`, `COUNT(DISTINCT)`, set operations and window functions.
