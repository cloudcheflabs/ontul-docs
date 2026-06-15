# Iceberg Write-Audit-Publish (WAP)

Write-Audit-Publish (WAP) lets you stage writes on a **non-`main` branch**, inspect that branch
in isolation (**audit**), and then **publish** it to `main` in a single atomic fast-forward.
Readers continue to see `main` the entire time, so un-audited data is never exposed to consumers.

```
            ┌─ write ──────────────┐   ┌─ audit ──────────┐   ┌─ publish ────────────┐
main:  S0 ──┤                      │   │  main still S0    │   │  main fast-forwards   │
            └─ audit branch: S0→S1→S2   (consumers see S0)     └─ main = S2            │
```

WAP works across **all three write paths** — batch (`INSERT`/`CTAS`), interactive DML
(`UPDATE`/`DELETE`/`MERGE`), and streaming sinks — and defaults to off (writes go straight to
`main`). It builds on Ontul's existing Iceberg branch support (branches/tags, `FOR VERSION AS OF`
reads) — see [Iceberg Integration](iceberg-integration.md).

## Selecting the write branch

The target branch is selected with the property key **`ontul.iceberg.wap.branch`**. The key is
namespaced (`ontul.iceberg.*`) so it never collides with other connector or catalog options.

A blank value or the literal `main` means **write directly to `main`** (WAP off — the default).

### Interactive / batch SQL — session property

For SQL writes (`INSERT`, `CTAS`, `UPDATE`, `DELETE`, `MERGE`) the branch is a **session property**
set with `SET`. Because a session can touch several Iceberg tables, the branch is resolved
**per table** — a session can stage some tables on a branch while others go straight to `main`.

Lookup precedence (most specific first; the first key that is set wins):

| Key | Scope |
|-----|-------|
| `ontul.iceberg.wap.branch.<catalog>.<schema>.<table>` | one fully-qualified table |
| `ontul.iceberg.wap.branch.<schema>.<table>` | table by schema.table |
| `ontul.iceberg.wap.branch.<table>` | table by name |
| `ontul.iceberg.wap.branch` | session-wide default for every Iceberg write |

```sql
-- Stage writes to ice.sales.orders on the 'audit' branch; everything else still hits main.
SET ontul.iceberg.wap.branch.orders = 'audit';

INSERT INTO ice.sales.orders SELECT * FROM staging.new_orders;   -- lands on 'audit'
UPDATE ice.sales.orders SET status = 'review' WHERE amount > 1e6; -- lands on 'audit'

-- Turn WAP off again for this table (back to main):
SET ontul.iceberg.wap.branch.orders = 'main';
```

The branch is **created automatically** from `main`'s current snapshot on first write, so the
audit branch always reflects `main`'s data plus the staged changes — and a later
`fast_forward` is always valid (the branch descends from `main`).

### Streaming sink — job config

For a streaming job, set the key inside the **sink config** of the `SUBMIT STREAMING` JSON:

```json
SUBMIT STREAMING {
  "kafka": { "connectionId": "kafka", "topic": "events", "format": "json" },
  "sink": {
    "type": "table",
    "table": "ice.analytics.events",
    "ontul.iceberg.wap.branch": "audit"
  },
  "commitIntervalMs": 5000,
  "durationMs": 60000
}
```

Every streaming commit (append or upsert `RowDelta`) lands on the `audit` branch; `main` is
untouched until you publish.

## Auditing the branch

Read the staged branch in isolation with `FOR VERSION AS OF '<branch>'` while normal queries keep
seeing `main`:

```sql
-- main (frozen)
SELECT count(*) FROM ice.sales.orders;

-- staged branch (write side)
SELECT count(*) FROM ice.sales.orders FOR VERSION AS OF 'audit';
SELECT * FROM ice.sales.orders FOR VERSION AS OF 'audit' WHERE status = 'review';
```

Both row scans and `count(*)` honor the branch reference, so you can run full data-quality checks
against the branch before deciding to publish.

## Publishing

Publishing is **explicit** — there is no auto-publish, which is the whole point of an audit gate.
Two `ALTER TABLE ... EXECUTE` operations are available:

### fast_forward (the normal path)

Fast-forwards a branch (default `main`) to the head of another branch. Valid when the target is a
descendant of the branch being moved — which is always true for a WAP branch created from `main`.

```sql
ALTER TABLE ice.sales.orders EXECUTE fast_forward(branch => 'main', to => 'audit');
```

After this, `main` points at the audited snapshot and consumers see the published data. The `audit`
branch still exists; drop it (`EXECUTE drop_branch(name => 'audit')`) or reuse it for the next cycle.

### cherrypick (when main has moved on)

If `main` advanced independently while you were auditing (so a fast-forward is no longer valid),
apply a single staged snapshot onto `main`:

```sql
ALTER TABLE ice.sales.orders EXECUTE cherrypick(snapshot_id => 8123456789012345678);
```

## End-to-end example

A complete `write → audit → publish` cycle is shipped as a runnable example in both Java and Python:

| | File | Run |
|-|------|-----|
| Java   | `package/examples/java/IcebergWapExample.java`       | `examples/bin/run-iceberg-wap.sh` |
| Python | `package/examples/python/iceberg_wap_example.py`     | `examples/bin/run-iceberg-wap-python.sh` |

Both connect over Arrow Flight SQL and perform: seed baseline on `main` → `SET` the WAP branch →
`INSERT`/`UPDATE` (staged) → audit (`main` frozen vs branch staged) → `fast_forward` publish →
verify `main` reflects the published data.

### Docker-compose E2E test

`tests/test-iceberg-wap-e2e.sh` runs the full cycle against a real stack — ShannonStore (S3) +
Apache Polaris (Iceberg REST catalog) in docker, plus a local Ontul cluster:

```bash
bash tests/test-iceberg-wap-e2e.sh
# teardown: bash tests/stop-infra.sh && bash tests/stop-cluster.sh
```

Expected output (abridged):

```
--- 0. Seed baseline on main ---            main rows after seed = 2
--- 1. Write to audit branch ---            staged 2 inserts + 1 update on branch 'audit'
--- 2. Audit: main vs branch ---            main rows = 2 (frozen)   branch rows = 4 (staged)
                                            audit OK ✓
--- 3. Publish: fast-forward main -> audit  main rows after publish = 4
=== WAP cycle complete: write → audit → publish ✓ ===
```

## Notes & limitations

- WAP is **per table** for SQL (keyed by table name) and **per job** for streaming (the sink config
  belongs to one table). There is no accidental session-wide bleed across unrelated tables.
- `UPDATE`/`DELETE`/`MERGE` on a branch validate from the **branch head**, so repeated row-level
  operations on the branch merge correctly against the branch (not `main`).
- Publishing is explicit (`fast_forward` / `cherrypick`); streaming jobs keep accumulating on the
  branch until you publish — there is no auto-publish.
- The same WAP model (`<project>.iceberg.wap.branch`, branch writes, fast-forward/cherrypick publish)
  is implemented consistently in the sibling ItdaStream and NeorunBase products.
