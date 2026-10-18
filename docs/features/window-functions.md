# Window Functions

A window function computes a value for each row from a set of related rows — its *window* — without
collapsing the rows the way `GROUP BY` does: a rank within a category, a running total, a moving
average, the previous order of the same customer.

```sql
SELECT o_orderkey,
       o_orderpriority,
       ROW_NUMBER() OVER (PARTITION BY o_orderpriority ORDER BY o_totalprice DESC) AS rank_in_priority,
       SUM(o_totalprice) OVER (PARTITION BY o_custkey ORDER BY o_orderdate)        AS running_total,
       AVG(o_totalprice) OVER (ORDER BY o_orderdate
                               ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)           AS moving_avg_7,
       LAG(o_orderdate) OVER (PARTITION BY o_custkey ORDER BY o_orderdate)         AS previous_order
FROM ice.sales.orders;
```

Ontul's end-to-end suite runs these queries, and the other shapes on this page, on Ontul and on Trino
479 over the same Iceberg tables and compares the answers (`tests/e2e-window-offset.sh`).

## Functions

| Function | Result |
|---|---|
| `ROW_NUMBER()` | 1, 2, 3, … in the window's order |
| `RANK()` | rank with gaps: peers (rows with equal `ORDER BY` values) share a rank, and the next rank skips |
| `DENSE_RANK()` | rank without gaps |
| `PERCENT_RANK()` | `(rank - 1) / (rows in partition - 1)`, `0` for a one-row partition |
| `CUME_DIST()` | fraction of the partition's rows that sort at or before the row, peers included |
| `NTILE(n)` | bucket 1 … n; the first `rows % n` buckets get one extra row |
| `LAG(x [, offset [, default]])`, `LEAD(…)` | `x` from the row `offset` rows before / after (default 1) in the partition, or `default` (default `NULL`) |
| `FIRST_VALUE(x)`, `LAST_VALUE(x)`, `NTH_VALUE(x, n)` | `x` from the first / last / n-th row of the frame |
| `COUNT(*)`, `COUNT(x)`, `SUM`, `AVG`, `MIN`, `MAX` | the aggregate over the frame |

`IGNORE NULLS` is honored by `LAG`, `LEAD`, `FIRST_VALUE`, `LAST_VALUE` and `NTH_VALUE`.

Result types follow the aggregates': `SUM` over integers is an exact `BIGINT` (overflow fails the query),
over a `DECIMAL(p, s)` an exact `DECIMAL(38, s)`; `AVG` over integers is a `DOUBLE`; `MIN`, `MAX`,
`FIRST_VALUE`, `LAG`, … keep the column's type, including `DATE`, `TIMESTAMP` and text.

## Partitions, order and peers

- `PARTITION BY` splits the rows; each partition is computed on its own. Without it the whole result is
  one partition.
- `ORDER BY` orders the rows inside the partition. `NULLS FIRST` / `NULLS LAST` are honored; without
  either, NULLs sort last in both directions, as in Trino.
- Rows whose `ORDER BY` values are all equal are **peers**. `RANK`, `DENSE_RANK`, `PERCENT_RANK`,
  `CUME_DIST` and `RANGE` frames treat peers as one position.

## Frames

The aggregate functions and `FIRST_VALUE` / `LAST_VALUE` / `NTH_VALUE` work over a *frame* — a range of
rows around the current one:

```sql
SUM(x) OVER (ORDER BY d ROWS  BETWEEN 2 PRECEDING AND 2 FOLLOWING)
SUM(x) OVER (ORDER BY d RANGE BETWEEN INTERVAL '7' DAY PRECEDING AND CURRENT ROW)
SUM(x) OVER (ORDER BY n RANGE BETWEEN 10 PRECEDING AND 10 FOLLOWING)
```

| Default | Frame |
|---|---|
| no `ORDER BY` | the whole partition |
| `ORDER BY` without a frame | `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` — a running total that includes the row's peers |

- `ROWS` counts rows. `RANGE` with an offset moves by a value of the single `ORDER BY` column: a number,
  or an interval for a `DATE` / `TIMESTAMP` column. A row whose key is `NULL` has its `NULL` peers as its
  frame.
- `EXCLUDE CURRENT ROW`, `EXCLUDE GROUP` and `EXCLUDE TIES` are supported.
- Over an empty frame `COUNT` is `0` and the other aggregates are `NULL`.

Frames that start at the partition's first row, frames that end at its last row, and sliding frames are
computed incrementally, so a running total over a partition of n rows is O(n), not O(n²).

## How it runs

A window needs every row of a partition in one place. A plan with a window runs its scan on one worker,
which computes the window over all the rows — see
[Splitting a Query Across Workers](query-performance.md#splitting-a-query-across-workers). The
operator holds the window's input in memory.

A window can rank the result of a `GROUP BY` — the aggregate is computed first, then the window over
the groups:

```sql
SELECT o_orderpriority, SUM(o_totalprice) AS revenue,
       RANK() OVER (ORDER BY SUM(o_totalprice) DESC) AS revenue_rank
FROM ice.sales.orders
GROUP BY o_orderpriority;
```

Several `OVER` clauses in one query are computed by one operator, one sort per distinct window. Rows come
out in the order of the last window; add `ORDER BY` to the query when the output order matters.

## Not supported

| Construct | Error | Rewrite |
|---|---|---|
| `DISTINCT` inside a window function, e.g. `COUNT(DISTINCT x) OVER (…)` | `DISTINCT inside window function COUNT()` | `GROUP BY` first, then the window |
| Other aggregates over a window (e.g. `STDDEV_POP(x) OVER (…)`) | `Window function STDDEV_POP()` | — |
| `RANGE` with an offset over a non-numeric, non-temporal key, or several keys | `RANGE frame with an offset over this ORDER BY` | `ROWS` |
