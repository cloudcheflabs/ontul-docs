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

The complete `write → audit → publish` cycle ships as a runnable example in both Java (Ontul SDK)
and Python (Arrow Flight SQL). Each one: seeds a baseline on `main` → sets the WAP branch →
stages an `INSERT` + `UPDATE` on the branch → audits `main` (frozen) against the branch (staged) →
fast-forward publishes → verifies `main` now reflects the published data.

The **audit is programmable**: the Java example reads each branch with the SDK DataFrame
(`session.source(Source.sql(...)).count()`), so the publish decision is a plain Java `if` on those
counts — no SQL result-set parsing. The writes use SQL because WAP staging appends/updates an
existing table, which the SDK's create-table sink does not express.

### Java — `package/examples/java/IcebergWapExample.java`

Run with `examples/bin/run-iceberg-wap.sh` (token via `-Dontul.user.token` / `ONTUL_USER_TOKEN`).

```java
import com.cloudcheflabs.ontul.sdk.OntulSession;
import com.cloudcheflabs.ontul.sdk.Source;

public class IcebergWapExample {

    public static void main(String[] args) throws Exception {
        String catalog = System.getProperty("ontul.catalog", "ice");
        String ns      = System.getProperty("ontul.namespace", "wap_demo");
        String table   = "orders";
        String fqn     = catalog + "." + ns + "." + table;
        String branch  = "audit";

        System.out.println("=== Iceberg Write-Audit-Publish (WAP) Example (Ontul SDK) ===\n");

        // Token is auto-loaded from -Dontul.user.token (ONTUL_USER_TOKEN).
        try (OntulSession session = OntulSession.builder()
                .master(System.getProperty("ontul.host", "localhost"),
                        Integer.getInteger("ontul.port", 47470))
                .build()) {

            // 0. SEED — baseline rows committed to main
            System.out.println("--- 0. Seed baseline on main ---");
            session.execute("DROP TABLE IF EXISTS " + fqn);
            session.execute("CREATE SCHEMA IF NOT EXISTS " + catalog + "." + ns);
            session.execute("CREATE TABLE " + fqn + " (id INT, region VARCHAR, amount DOUBLE)");
            session.execute("INSERT INTO " + fqn +
                    " SELECT * FROM (VALUES (1,'us',100.0),(2,'eu',200.0)) AS t(id, region, amount)");
            System.out.println("  main rows after seed = " + count(session, fqn, null) + "  (expect 2)");

            // 1. WRITE — stage inserts + an update on the audit branch.
            // Per-table WAP: only `orders` writes are redirected; other tables in this session
            // still go to main. Use ...wap.branch (no table suffix) for a session-wide default.
            System.out.println("\n--- 1. Write to audit branch (main stays frozen) ---");
            session.execute("SET ontul.iceberg.wap.branch." + table + " = '" + branch + "'");
            session.execute("INSERT INTO " + fqn +
                    " SELECT * FROM (VALUES (3,'us',300.0),(4,'eu',400.0)) AS t(id, region, amount)");
            session.execute("UPDATE " + fqn + " SET amount = 999.0 WHERE id = 1");
            System.out.println("  staged 2 inserts + 1 update on branch '" + branch + "'");

            // 2. AUDIT — main unchanged; branch carries the staged state. Done PROGRAMMABLY
            //    via the SDK DataFrame: count() returns a Java long used directly in the gate.
            System.out.println("\n--- 2. Audit: main vs branch ---");
            long mainCount   = count(session, fqn, null);
            long branchCount = count(session, fqn, branch);
            System.out.println("  main   rows = " + mainCount   + "  (expect 2 — frozen)");
            System.out.println("  branch rows = " + branchCount + "  (expect 4 — staged)");

            if (mainCount != 2 || branchCount != 4) {                 // gate before publishing
                throw new IllegalStateException("AUDIT FAILED: expected main=2, branch=4 but got main="
                        + mainCount + ", branch=" + branchCount + " — NOT publishing");
            }
            System.out.println("  audit OK ✓");

            // 3. PUBLISH — fast-forward main to the audited branch
            System.out.println("\n--- 3. Publish: fast-forward main -> '" + branch + "' ---");
            session.execute("ALTER TABLE " + fqn +
                    " EXECUTE fast_forward(branch => 'main', to => '" + branch + "')");
            long published = count(session, fqn, null);
            System.out.println("  main rows after publish = " + published + "  (expect 4)");

            if (published != 4) {
                throw new IllegalStateException("PUBLISH FAILED: expected main=4, got " + published);
            }
            System.out.println("\n=== WAP cycle complete: write → audit → publish ✓ ===");
        }
    }

    /** Row count of a table, optionally as of a branch/tag (null = main). */
    private static long count(OntulSession session, String fqn, String branch) {
        String sql = "SELECT * FROM " + fqn
                + (branch != null ? " FOR VERSION AS OF '" + branch + "'" : "");
        return session.source(Source.sql(sql)).count();
    }
}
```

### Python — `package/examples/python/iceberg_wap_example.py`

Run with `examples/bin/run-iceberg-wap-python.sh`.

```python
import os
import sys
import pyarrow as pa
import pyarrow.flight as flight

CATALOG = os.environ.get("ONTUL_CATALOG", "ice")
NS      = os.environ.get("ONTUL_NAMESPACE", "wap_demo")
TABLE   = "orders"
FQN     = f"{CATALOG}.{NS}.{TABLE}"
BRANCH  = "audit"


def create_client():
    host = os.environ.get("ONTUL_HOST", "localhost")
    port = int(os.environ.get("ONTUL_PORT", "47470"))
    token = os.environ.get("ONTUL_USER_TOKEN")
    if not token:
        print("ERROR: ONTUL_USER_TOKEN must be set")
        sys.exit(1)
    client = flight.FlightClient(f"grpc://{host}:{port}")
    options = flight.FlightCallOptions(
        headers=[(b"authorization", f"Token {token}".encode())])
    return client, options


def query(client, options, sql):
    """Run a query/statement and return the result as a PyArrow Table."""
    desc = flight.FlightDescriptor.for_command(sql.encode())
    info = client.get_flight_info(desc, options)
    if not info.endpoints:
        return pa.table({})
    return client.do_get(info.endpoints[0].ticket, options).read_all()


def exec_sql(client, options, sql):
    print(f"  SQL> {sql}")
    query(client, options, sql)


def count(client, options, fqn, branch=None):
    sql = f"SELECT count(*) AS c FROM {fqn}"
    if branch:
        sql += f" FOR VERSION AS OF '{branch}'"
    t = query(client, options, sql)
    return int(t.column(0)[0].as_py()) if t.num_rows else -1


def show(client, options, sql):
    t = query(client, options, sql)
    cols = t.schema.names
    print("    " + " | ".join(cols))
    data = t.to_pydict()
    for i in range(t.num_rows):
        print("    " + " | ".join(str(data[c][i]) for c in cols))


def main():
    client, options = create_client()
    print("=== Iceberg Write-Audit-Publish (WAP) Example (Python) ===\n")

    # 0. SEED — baseline on main
    print("--- 0. Seed baseline on main ---")
    exec_sql(client, options, f"DROP TABLE IF EXISTS {FQN}")
    exec_sql(client, options, f"CREATE SCHEMA IF NOT EXISTS {CATALOG}.{NS}")
    exec_sql(client, options, f"CREATE TABLE {FQN} (id INT, region VARCHAR, amount DOUBLE)")
    exec_sql(client, options,
             f"INSERT INTO {FQN} SELECT * FROM (VALUES (1,'us',100.0),(2,'eu',200.0)) AS t(id, region, amount)")
    print(f"  main rows after seed = {count(client, options, FQN)}  (expect 2)")

    # 1. WRITE — stage on the audit branch (per-table WAP key)
    print("\n--- 1. Write to audit branch (main stays frozen) ---")
    exec_sql(client, options, f"SET ontul.iceberg.wap.branch.{TABLE} = '{BRANCH}'")
    exec_sql(client, options,
             f"INSERT INTO {FQN} SELECT * FROM (VALUES (3,'us',300.0),(4,'eu',400.0)) AS t(id, region, amount)")
    exec_sql(client, options, f"UPDATE {FQN} SET amount = 999.0 WHERE id = 1")
    print(f"  staged 2 inserts + 1 update on branch '{BRANCH}'")

    # 2. AUDIT — main frozen, branch staged
    print("\n--- 2. Audit: main vs branch ---")
    main_count = count(client, options, FQN)
    branch_count = count(client, options, FQN, BRANCH)
    print(f"  main   rows = {main_count}  (expect 2 - frozen)")
    print(f"  branch rows = {branch_count}  (expect 4 - staged)")
    show(client, options,
         f"SELECT id, region, amount FROM {FQN} FOR VERSION AS OF '{BRANCH}' ORDER BY id")
    if main_count != 2 or branch_count != 4:
        print("  AUDIT FAILED: expected main=2, branch=4 - NOT publishing")
        sys.exit(1)
    print("  audit OK")

    # 3. PUBLISH — fast-forward main to the audited branch
    print(f"\n--- 3. Publish: fast-forward main -> '{BRANCH}' ---")
    exec_sql(client, options,
             f"ALTER TABLE {FQN} EXECUTE fast_forward(branch => 'main', to => '{BRANCH}')")
    published = count(client, options, FQN)
    print(f"  main rows after publish = {published}  (expect 4)")
    show(client, options, f"SELECT id, region, amount FROM {FQN} ORDER BY id")
    if published != 4:
        print(f"  PUBLISH FAILED: expected main=4, got {published}")
        sys.exit(1)
    print("\n=== WAP cycle complete: write -> audit -> publish ===")


if __name__ == "__main__":
    main()
```

Running either example prints the staged-vs-frozen audit and the published result:

```
--- 0. Seed baseline on main ---
  main rows after seed = 2  (expect 2)
--- 1. Write to audit branch (main stays frozen) ---
  staged 2 inserts + 1 update on branch 'audit'
--- 2. Audit: main vs branch ---
  main   rows = 2  (expect 2 — frozen)
  branch rows = 4  (expect 4 — staged)
  audit OK
--- 3. Publish: fast-forward main -> 'audit' ---
  main rows after publish = 4  (expect 4)
=== WAP cycle complete: write → audit → publish ===
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
