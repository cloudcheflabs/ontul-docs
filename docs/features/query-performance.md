# Query Performance

Ontul is a distributed, Arrow-native query engine for Apache Iceberg. Its performance strategy follows the
same principles modern lakehouse engines rely on: **skip data you don't need to read, decode the data you do
need columnar, and keep per-query overhead low.** This page describes the concrete mechanisms and how to
control them. The Iceberg scan path reads **Parquet** data files (Ontul's Iceberg read/write is Parquet-based).

## Data Skipping

The single biggest lever for selective queries is reading fewer bytes. Ontul prunes at two levels.

### File pruning (Iceberg column metrics)

Every Parquet data file Ontul writes carries Iceberg **column metrics** — per-column lower/upper bounds, null
counts, and value counts — recorded in the table's manifests at commit time
(`ParquetUtil.fileMetrics(...)`). At scan-planning time Iceberg evaluates the query predicate against these
bounds and prunes whole files whose range cannot contain a matching row, before any data is read.

For a table clustered on a column, a selective predicate (`WHERE id > 980000`) plans only the handful of
files whose bounds overlap the range, skipping the rest entirely. On a 1,000,000-row / 25-file table this
prunes 24 of 25 files and yields a **5–13× speedup** over a full scan, with exact results.

### Row-group skipping (Parquet statistics)

Within a file, Ontul reads each Parquet row group's footer statistics (min/max per column) and skips an
entire row group when its statistics prove no row can match the pushed-down predicate — no column data is
decoded for skipped groups. This is correctness-safe: the `FILTER` operator re-applies the exact predicate,
so a missed skip only costs I/O, never a wrong answer. Disabled automatically when delete files are present.

Toggles: `-Dontul.scan.pushdown.enabled` (file/predicate pushdown), `-Dontul.scan.rowgroup.skip.enabled`
(both default on).

### Random / range reads

Ontul reads Parquet over the object-store `InputFile` directly — only the footer, the non-skipped row groups,
and (when projecting) the requested column chunks are fetched, rather than pre-fetching the whole file. This
is what realizes the I/O savings from pruning and projection. Set
`-Dontul.scan.prefetch.whole.file=true` to revert to whole-file pre-fetch.

## Columnar Parquet Decode

For scan- and compute-heavy queries (full-table aggregation, `GROUP BY`, sort), the dominant cost is decoding
Parquet into the engine's in-memory format. Ontul decodes **column-at-a-time, directly into Arrow vectors**
via Parquet's low-level `ColumnReader` API — no intermediate per-row object materialization and no value
boxing. Type dispatch is hoisted out of the per-value loop so each column is decoded by a tight, JIT-friendly
typed loop.

The columnar decode path is used for the common analytical case — no delete files and a flat, all-primitive
projection (integers, floating point, boolean, string/binary). Tables with nested types, decimals/temporals,
or attached delete files fall back to the proven row-at-a-time reader, so results are always correct. It is
Arrow-version-independent (built on `parquet-column`).

Toggle: `-Dontul.scan.columnar.enabled`.

## Columnar Aggregation & Bounded Top-N

The execution operators are tuned for the same column-oriented access:

- **Scalar aggregation** (`SUM`/`AVG`/`MIN`/`MAX`/`COUNT` with no `GROUP BY`) reduces each input column with
  typed Arrow vector access — no per-row group key, no hash map, no boxing. `COUNT` is computed in O(1) per
  batch from the Arrow null count.
- **`ORDER BY ... LIMIT k`** uses a bounded max-heap of the `k` best rows while scanning — **O(n·log k)** time
  and **O(k)** memory — instead of materializing and fully sorting all `n` rows before taking `k`. Each worker
  emits only its local top-`k`; the coordinator merges the small per-worker results into the global top-`k`.

Together these turn what were the slowest query shapes into ones competitive with, or faster than, mature
columnar engines. On a 600,000-row table (data cache off), the internal before/after is representative:
full-table aggregation `938 → 169 ms`, `GROUP BY` `834 → 285 ms`, `ORDER BY ... LIMIT 10` `1063 → 328 ms`.

## Low Per-Query Overhead

Ontul is designed for a low fixed cost per query, which matters for interactive and high-concurrency
workloads:

- **Manifest content cache** — Iceberg manifest files are immutable, so their content is cached by path
  (`io.manifest.cache-enabled`, on by default); repeated planning over the same snapshot skips re-reading
  manifests from object storage.
- **Persistent worker connections** — the coordinator keeps warm, multiplexed connections to workers instead
  of opening a fresh socket per query, removing handshake and thread-spawn churn under high QPS.
- **Distributed aggregate combine** — single-row and grouped aggregates run a partial pass on each worker and
  a final combine on the coordinator (correct `GROUP BY`, `AVG`, and `ORDER BY`/`LIMIT` across workers).

A trivial cached query (e.g. `COUNT(*)` served from metadata) completes in single-digit milliseconds, giving
Ontul a much lower latency floor than batch-oriented analytical engines.

## Tuning Reference

| Property | Default | Effect |
| --- | --- | --- |
| `ontul.scan.pushdown.enabled` | `true` | Predicate/file pruning at scan planning |
| `ontul.scan.rowgroup.skip.enabled` | `true` | Skip Parquet row groups by footer stats |
| `ontul.scan.columnar.enabled` | `false` | Columnar `ColumnReader` decode (flat-primitive, no deletes) |
| `ontul.scan.prefetch.whole.file` | `false` | Pre-fetch whole file instead of random/range reads |
| `ontul.scan.projection.enabled` | `false` | Decode only referenced columns |
| `io.manifest.cache-enabled` | `true` | Cache immutable Iceberg manifests |
| `ontul.cache.data.enabled` | `false` | In-memory data cache for repeated scans |

All data-skipping and decode toggles are correctness-safe: when a fast path cannot apply, Ontul falls back to
the reference path, so results are identical regardless of configuration.
