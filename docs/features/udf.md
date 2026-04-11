# User-Defined Functions (UDF)

Ontul lets you register Java and Python functions as first-class SQL UDFs and call them from any client — Ontul SDK, Arrow Flight SQL JDBC, or REST. UDFs run on Workers in the same Arrow-native pipeline as built-in operators, so there is no per-row JVM ↔ Python boundary cost for primitives, and POJO/dict return values are materialised directly into Arrow `StructVector` / `ListVector` (no JSON detour).

UDFs are real Calcite scalar functions — usable in `SELECT`, `WHERE`, `GROUP BY`, ordering, projection, and inside other SQL expressions:

```sql
SELECT name, mask_email(email) AS masked, length_class(name) AS bucket
FROM users
WHERE is_likely_bot(user_agent) = FALSE
GROUP BY length_class(name);
```

## Three scopes

Ontul supports three UDF scopes that map to common organisational needs:

| Scope | Visibility | Persistence | Author with | Drop with | Lifetime |
|---|---|---|---|---|---|
| **TEMPORARY** | Current SDK session only — JDBC connections do *not* see it | None | `session.registerUdf(...)` | `session.unregisterUdf(...)` | Until session closes |
| **USER** | All sessions of the same authenticated user (SDK + JDBC alike) | RocksDB metadata, replicated to follower Masters | `session.registerPersistentUdf(...)` | `session.unregisterPersistentUdf(...)` or `DROP FUNCTION name` | Until explicitly dropped, survives master restarts |
| **GLOBAL** | All authenticated users (subject to IAM `UDF:EXECUTE`) | RocksDB metadata, replicated | `session.registerGlobalUdf(...)` | `session.unregisterGlobalUdf(...)` or `DROP GLOBAL FUNCTION name` | Until explicitly dropped, survives master restarts |

When a name collision exists, lookup precedence is **TEMPORARY > USER > GLOBAL**. A session can shadow a USER UDF, and a USER UDF can shadow a GLOBAL one without affecting other clients.

## Authoring is SDK-only

UDFs are authored exclusively through the SDK because authoring inherently requires shipping code (a Java class or Python function) to the Workers. The SDK auto-uploads the defining class as a JAR to `data/deps/` on the master, and the worker classloader picks it up on the next query — no manual jar drops, no `CREATE FUNCTION ... USING JAR 'hdfs://...'` ceremony. JDBC clients (BI tools, dashboards, notebooks) consume UDFs but do not author them, mirroring how Snowflake/BigQuery treat function objects.

```java
public final class Helpers {
    public static String maskEmail(String email) {
        if (email == null) return email;
        int at = email.indexOf('@');
        return at <= 0 ? email : "*".repeat(at) + email.substring(at);
    }
}

OntulSession session = OntulSession.builder().master("localhost", 47470).build();
session.registerPersistentUdf("mask_email", Helpers.class, "maskEmail");
```

Subsequent SDK sessions of the same user — and any JDBC connection authenticated as the same user — can immediately call `mask_email(...)` without re-registration.

## Discovery from anywhere — `SHOW FUNCTIONS`

Both SDK and JDBC clients can discover the UDFs visible to them through standard SQL:

```sql
SHOW FUNCTIONS;
-- name        | language    | return_type | param_types | scope
-- mask_email  | java-method | string      | string      | USER
-- length_class| java-method | string      | string      | TEMPORARY
-- geohash     | java-method | string      | double, double | GLOBAL
```

`DESCRIBE FUNCTION mask_email` returns full metadata: language, parameter types, return type, scope, owner, target class, target method, struct fields (for POJO returns), etc.

## Java UDF features

- **Method-reference style** — preferred. Pass `(class, methodName)`; the SDK uploads the class and registers metadata only. The function does not need to implement `Serializable`.
- **Lambda style** — `session.registerUdf("name", (Udf1<String,String>) s -> ...)` for one-off inline functions; the lambda is `ObjectOutputStream`-serialized.
- **Multi-argument** — up to 4 arguments out of the box; arbitrary arity through method references.
- **Primitive returns** — `String`, `long`, `double`, `boolean`, `byte[]` (for ML embeddings or binary blobs), `BigDecimal`.
- **`List<X>` returns** — materialised as Arrow `ListVector`. Element type can be primitive or struct.
- **POJO struct returns** — public-field POJOs become Arrow `StructVector`. Field access via dot notation works in SQL: `SELECT ocr(image).text, ocr(image).confidence FROM images`. The POJO is never serialised across the wire — workers populate the struct child vectors via reflection.
- **External libraries** — UDFs can use any third-party JAR available on the worker classpath (Apache Commons Math, Guava, JTS, etc.).
- **Static state** — the worker deps classloader is cached and fingerprinted, so static caches inside UDF classes (e.g. a loaded ML model) survive across queries.

## Python UDF features

Python UDFs are executed in a long-lived Python subprocess per worker, with batch-level Arrow IPC between the JVM worker and the subprocess (no per-row JSON, no per-call process spawn):

```python
from ontul.session import OntulSession

session = OntulSession()

def normalise(name: str) -> str:
    return name.strip().lower() if name else None

session.register_udf("normalise", normalise, return_type="string", param_types=["string"])
```

- **cloudpickle-based** — register any callable, including closures and methods.
- **Returns**: primitives, `dict` (→ Arrow struct), `list` (→ Arrow list), `bytes` (→ Arrow binary).
- **Notebook-friendly** — interactive Jupyter / IPython workflows work without restarting the cluster.
- **Same SQL surface** as Java UDFs — `SELECT my_py_udf(col) FROM ...` works identically from JDBC and the SDK.

## IAM-controlled execution

Every UDF call is authorised through the IAM policy engine. Five action verbs apply:

| Action | Required for |
|---|---|
| `UDF:EXECUTE` | Calling a UDF in any query (TEMPORARY UDFs in your own session are implicitly allowed) |
| `UDF:CREATE` | Registering a TEMPORARY or USER UDF |
| `UDF:DROP` | Dropping a USER UDF |
| `UDF:CREATE_GLOBAL` | Registering a GLOBAL UDF |
| `UDF:DROP_GLOBAL` | Dropping a GLOBAL UDF |

Resources are addressed as `udf:<name>` so policies can use exact match or wildcards. Examples:

```json
{
  "Sid": "ExecuteAnyUdf",
  "Effect": "Allow",
  "Action": "UDF:EXECUTE",
  "Resource": "udf:*"
}
```

```json
{
  "Sid": "ExecuteOnlySafeFamily",
  "Effect": "Allow",
  "Action": "UDF:EXECUTE",
  "Resource": ["udf:mask_*", "udf:hash_*", "udf:geohash"]
}
```

```json
{
  "Sid": "DenySensitiveUdfs",
  "Effect": "Deny",
  "Action": "UDF:EXECUTE",
  "Resource": ["udf:internal_*", "udf:*_admin"]
}
```

The Admin UI policy editor ships with preset templates for the most common shapes: *UDF Execute Any*, *UDF Author*, *UDF Sandbox*, *UDF Deny Sensitive*, *UDF Global Admin*, *UDF Global Read-Only*. The planner enforces `UDF:EXECUTE` against the actual functions referenced in each query — execution is denied early, before any data is touched.

## Architecture notes

- **Calcite integration** — UDFs are installed onto a per-query `SchemaPlus` clone wrapped in a chained operator table (`SqlStdOperatorTable` + the per-session UDF schema reader). Per-query isolation means concurrent queries with different UDF names never collide.
- **Plan attachment** — the planner attaches the resolved `Map<String, UdfDescriptor>` directly onto `PhysicalPlan.udfs`. The distributed planner copies the map onto every worker plan, so workers receive only the UDFs the query actually uses, with no out-of-band registration step.
- **Row-wise vs batch evaluation** — Java method-ref UDFs are wrapped in a per-row `EvalExpr`. Python UDFs short-circuit through `PyUdfBatchExpr`, which packs all argument rows into Arrow IPC and ships a single batch to the Python subprocess, so the per-call cost amortises across thousands of rows.
- **Auto-upload + cached classloader** — `WorkerServer.buildDepsClassLoader()` caches the URLClassLoader for `data/deps/` and invalidates on directory change (file size + mtime fingerprint). UDFs that hold static models (caches, embeddings, ML weights) are loaded once per worker and reused across queries.
- **Persistence** — USER and GLOBAL descriptors are JSON-serialised into the cluster `RocksDbMetadataStore` under `udf.<userId>.<name>` and `udf._global.<name>` respectively. The store is the same KMS-encrypted, leader-replicated metadata store used for catalogs and connections, so UDFs participate in HA and survive master restarts. JARs are persisted independently in `data/deps/` by the auto-upload path.
- **Strict Arrow data plane** — control messages (descriptor metadata, registration acks) travel as JSON over Flight Actions. All data flowing in or out of a UDF travels as Arrow IPC. There is never a JSON or base64 hop on the data path, even for `byte[]`, list, or struct return types.
