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

INSERT INTO ice.sales.orders SELECT * FROM ice.staging.new_orders;  -- lands on 'audit'
UPDATE ice.sales.orders SET status = 'review' WHERE amount > 1e6;   -- lands on 'audit'

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

The complete `write → audit → publish` cycle ships as a runnable example. The Java example uses the
**Ontul SDK DataFrame API** end to end: read a source table, `filter`, `withColumn`, then
`sink(Sink.table(...))` — so the enriched table is *built directly on the `audit` branch*, never on
`main`, until you publish. The audit is programmatic too: `DataFrame.count()` returns a Java `long`
used straight in the publish gate.

This is a realistic WAP use case — building a derived/enriched table that consumers must not see
until it has been validated.

### Java — `package/examples/java/IcebergWapExample.java`

Run with `examples/bin/run-iceberg-wap.sh` (token via `-Dontul.user.token` / `ONTUL_USER_TOKEN`).

```java
import com.cloudcheflabs.ontul.sdk.OntulSession;
import com.cloudcheflabs.ontul.sdk.Sink;
import com.cloudcheflabs.ontul.sdk.Source;

public class IcebergWapExample {

    public static void main(String[] args) throws Exception {
        String catalog = System.getProperty("ontul.catalog", "ice");
        String ns      = System.getProperty("ontul.namespace", "wap_demo");
        String raw     = catalog + "." + ns + ".raw_sales";
        String table   = "enriched_sales";
        String fqn     = catalog + "." + ns + "." + table;
        String branch  = "audit";

        try (OntulSession session = OntulSession.builder()
                .master(System.getProperty("ontul.host", "localhost"),
                        Integer.getInteger("ontul.port", 47470))
                .build()) {

            // 0. SEED — a raw_sales source table (3 valid, 2 invalid rows). A bare VALUES
            //    literal like 10.0 is inferred as DECIMAL; use an explicit DOUBLE column.
            session.execute("DROP TABLE IF EXISTS " + fqn);
            session.execute("DROP TABLE IF EXISTS " + raw);
            session.execute("CREATE SCHEMA IF NOT EXISTS " + catalog + "." + ns);
            session.execute("CREATE TABLE " + raw +
                    " (sale_id INT, product_id VARCHAR, quantity INT, unit_price DOUBLE, region VARCHAR)");
            session.execute("INSERT INTO " + raw + " SELECT * FROM (VALUES " +
                    "(1,'p1',2,10.0,'us'), (2,'p2',3,20.0,'eu'), (3,'p3',0,15.0,'us'), " +
                    "(4,'p4',1,0.0,'eu'), (5,'p5',4,25.0,'us')) " +
                    "AS t(sale_id, product_id, quantity, unit_price, region)");

            // 1. SET — the enriched table will be built on the 'audit' branch, not main.
            session.execute("SET ontul.iceberg.wap.branch." + table + " = '" + branch + "'");

            // 2. WRITE — programmatic DataFrame ETL: read source → filter invalid → enrich → sink.
            //    Because WAP is set above, the resulting table is built on the 'audit' branch.
            session.source(Source.sql("SELECT * FROM " + raw))
                    .filter("quantity > 0 AND unit_price > 0")           // drop the 2 invalid rows
                    .withColumn("total_amount", "quantity * unit_price")  // enrich
                    .sink(Sink.table(fqn));                               // → 'audit' branch

            // 3. AUDIT — programmatic via the SDK DataFrame.count(): branch has the rows, main empty.
            long mainCount   = count(session, fqn, null);
            long branchCount = count(session, fqn, branch);
            System.out.println("  main rows = " + mainCount + " (expect 0)   branch rows = "
                    + branchCount + " (expect 3)");
            if (mainCount != 0 || branchCount != 3) {
                throw new IllegalStateException("AUDIT FAILED: main=" + mainCount
                        + " branch=" + branchCount + " — NOT publishing");
            }

            // 4. PUBLISH — fast-forward main to the audited branch.
            session.execute("ALTER TABLE " + fqn +
                    " EXECUTE fast_forward(branch => 'main', to => '" + branch + "')");
            long published = count(session, fqn, null);
            System.out.println("  main rows after publish = " + published + " (expect 3)");
            if (published != 3) throw new IllegalStateException("PUBLISH FAILED: main=" + published);
            System.out.println("=== WAP cycle complete: write → audit → publish ✓ ===");
        }
    }

    /** Row count of a table, optionally as of a branch/tag (null = main) — SDK DataFrame.count(). */
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
TABLE   = "enriched_sales"
FQN     = f"{CATALOG}.{NS}.{TABLE}"
RAW     = f"{CATALOG}.{NS}.raw_sales"
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


def main():
    client, options = create_client()
    print("=== Iceberg Write-Audit-Publish (WAP) Example (Python) ===\n")

    # 0. SEED — raw_sales source table (DOUBLE price; a bare 10.0 would be DECIMAL)
    print("--- 0. Seed source table raw_sales ---")
    exec_sql(client, options, f"DROP TABLE IF EXISTS {FQN}")
    exec_sql(client, options, f"DROP TABLE IF EXISTS {RAW}")
    exec_sql(client, options, f"CREATE SCHEMA IF NOT EXISTS {CATALOG}.{NS}")
    exec_sql(client, options,
             f"CREATE TABLE {RAW} (sale_id INT, product_id VARCHAR, quantity INT, unit_price DOUBLE, region VARCHAR)")
    exec_sql(client, options,
             f"INSERT INTO {RAW} SELECT * FROM (VALUES "
             "(1,'p1',2,10.0,'us'), (2,'p2',3,20.0,'eu'), (3,'p3',0,15.0,'us'), "
             "(4,'p4',1,0.0,'eu'), (5,'p5',4,25.0,'us')) "
             "AS t(sale_id, product_id, quantity, unit_price, region)")
    print("  raw_sales seeded (5 rows: 3 valid, 2 invalid)")

    # 1. SET — the enriched table will be built on the 'audit' branch, not main
    print(f"\n--- 1. SET WAP branch '{BRANCH}' for {TABLE} ---")
    exec_sql(client, options, f"SET ontul.iceberg.wap.branch.{TABLE} = '{BRANCH}'")

    # 2. WRITE — build the enriched table (filter invalid + enrich) → lands on 'audit'
    print(f"\n--- 2. Build enriched table (→ branch '{BRANCH}') ---")
    exec_sql(client, options,
             f"CREATE TABLE {FQN} AS SELECT *, quantity * unit_price AS total_amount "
             f"FROM {RAW} WHERE quantity > 0 AND unit_price > 0")
    print(f"  enriched_sales built on branch '{BRANCH}'")

    # 3. AUDIT — main empty, branch has the rows
    print("\n--- 3. Audit: main vs branch ---")
    main_count = count(client, options, FQN)
    branch_count = count(client, options, FQN, BRANCH)
    print(f"  main   rows = {main_count}  (expect 0 - not published)")
    print(f"  branch rows = {branch_count}  (expect 3 - valid enriched rows)")
    if main_count != 0 or branch_count != 3:
        print("  AUDIT FAILED: expected main=0, branch=3 - NOT publishing")
        sys.exit(1)
    print("  audit OK")

    # 4. PUBLISH — fast-forward main to the audited branch
    print(f"\n--- 4. Publish: fast-forward main -> '{BRANCH}' ---")
    exec_sql(client, options,
             f"ALTER TABLE {FQN} EXECUTE fast_forward(branch => 'main', to => '{BRANCH}')")
    published = count(client, options, FQN)
    print(f"  main rows after publish = {published}  (expect 3)")
    if published != 3:
        print(f"  PUBLISH FAILED: expected main=3, got {published}")
        sys.exit(1)
    print("\n=== WAP cycle complete: write -> audit -> publish ===")


if __name__ == "__main__":
    main()
```

Running either example prints the staged-vs-frozen audit and the published result:

```
--- 0. Seed source table raw_sales ---
  raw_sales seeded (5 rows: 3 valid, 2 invalid)
--- 1. SET WAP branch 'audit' for enriched_sales ---
--- 2. Build enriched table (→ branch 'audit') ---
  enriched_sales built on branch 'audit'
--- 3. Audit: main vs branch ---
  main   rows = 0  (expect 0 — not published)
  branch rows = 3  (expect 3 — valid enriched rows)
  audit OK
--- 4. Publish: fast-forward main -> 'audit' ---
  main rows after publish = 3  (expect 3)
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
