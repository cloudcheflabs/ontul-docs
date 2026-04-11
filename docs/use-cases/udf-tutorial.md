# UDF Tutorial: TEMPORARY → USER → GLOBAL with SDK + JDBC

This tutorial walks you through Ontul's three UDF scopes — **TEMPORARY**, **USER**, and **GLOBAL** — and shows how the same Java function travels from a one-off session UDF to a persistent shared UDF callable from any BI tool over JDBC.

**What you will learn:**

- Write a Java UDF as a plain `public static` method (no `Serializable` ceremony)
- Register it as a TEMPORARY (session-only) UDF in the Ontul SDK
- Discover it via standard `SHOW FUNCTIONS` / `DESCRIBE FUNCTION`
- Promote the same function to a USER-scoped persistent UDF visible across SDK + JDBC for the same user
- Promote it to a GLOBAL UDF, gated by IAM `UDF:EXECUTE` policies
- Call the GLOBAL UDF from a JDBC BI tool as a different user
- Drop a GLOBAL UDF via standard `DROP GLOBAL FUNCTION` SQL

## Prerequisites

- Java 17
- A running Ontul cluster (see [Batch ETL](batch-etl.md) for setup), with `ONTUL_USER_TOKEN` exported

## 1. Run the bundled examples

Three end-to-end examples in the distribution exercise everything in this tutorial:

```bash
# 1) Java SQL UDFs (TEMPORARY) — method-ref + lambda + multi-arg + unregister
examples/bin/run-udf-sql-test.sh

# 2) SHOW FUNCTIONS / DESCRIBE FUNCTION discovery surface
examples/bin/run-udf-show-functions-test.sh

# 3) USER-scoped persistent UDF — visible from SDK + JDBC for the same user
examples/bin/run-udf-persistent-test.sh

# 4) GLOBAL UDF + IAM — admin creates, alice consumes via UDF:EXECUTE policy,
#    JDBC drop is denied for alice and allowed for admin
examples/bin/run-udf-global-test.sh
```

The walkthrough below mirrors what those tests do.

---

## 2. Write a UDF class

A UDF is just a `public static` method on any class. No interface to implement, no `Serializable` to inherit.

```java
public final class Helpers {
    private Helpers() {}

    /** Replaces the local part of an email with asterisks. */
    public static String maskEmail(String email) {
        if (email == null) return null;
        int at = email.indexOf('@');
        if (at <= 0) return email;
        return "*".repeat(at) + email.substring(at);
    }

    /** Bucket a string by length. */
    public static String lengthClass(String text) {
        if (text == null || text.isEmpty()) return "empty";
        int n = text.length();
        if (n <= 4) return "short";
        if (n <= 8) return "medium";
        return "long";
    }
}
```

The class can use any third-party library that is already on the worker classpath (Apache Commons Math, Guava, JTS, etc.). The Ontul SDK auto-uploads the defining class as a JAR to the master's `data/deps/` directory the first time you call `registerUdf`, and the worker classloader picks it up on the next query.

---

## 3. TEMPORARY UDF — session only

The default scope. Only the registering session can see and call it. Closing the session removes it.

```java
OntulSession session = OntulSession.builder()
        .master("localhost", 47470)
        .build();

session.registerUdf("mask_email",   Helpers.class, "maskEmail");
session.registerUdf("length_class", Helpers.class, "lengthClass");

// Call them from SQL — works in SELECT, WHERE, GROUP BY, ORDER BY
DataFrame df = session.source(Source.sql(
    "SELECT length_class(n_name) AS bucket, " +
    "       COUNT(*) AS n " +
    "FROM tpch.tiny.nation " +
    "GROUP BY length_class(n_name)"));

session.close();   // mask_email and length_class are gone
```

A second SDK session — and any JDBC client, even authenticated as the same user — will not see these UDFs. They live and die with the registering Flight session.

### Discovery

`SHOW FUNCTIONS` and `DESCRIBE FUNCTION` are standard SQL and work over both SDK and JDBC:

```sql
SHOW FUNCTIONS;
-- name         | language    | return_type | param_types | scope
-- length_class | java-method | string      | string      | TEMPORARY
-- mask_email   | java-method | string      | string      | TEMPORARY

DESCRIBE FUNCTION mask_email;
-- property      | value
-- name          | mask_email
-- language      | java-method
-- return_type   | string
-- param_types   | string
-- scope         | TEMPORARY
-- target_class  | Helpers
-- target_method | maskEmail
```

---

## 4. USER UDF — persistent for the same user, SDK + JDBC

Promote the same function to USER scope and it survives:

- Across SDK sessions of the same authenticated user
- Across JDBC connections of the same authenticated user
- Across master restarts (RocksDB metadata is cluster-replicated)

```java
OntulSession session = OntulSession.builder()
        .master("localhost", 47470)
        .build();

// Same Java method, different SDK call:
session.registerPersistentUdf("mask_email", Helpers.class, "maskEmail");

session.close();
```

Now any future session of the **same user** sees it without re-registration:

```java
// New SDK session — different sessionId, same user
try (OntulSession s2 = OntulSession.builder().master("localhost", 47470).build()) {
    DataFrame df = s2.source(Source.sql("SELECT mask_email('alice@example.com') AS m"));
    // → "*****@example.com"
}
```

And from a **JDBC BI tool** authenticated as the same user (token is the user's JWT):

```java
Properties props = new Properties();
props.put("user", "Token " + token);    // "Token <jwt>"
props.put("password", "");
props.put("useEncryption", "false");

try (Connection jdbc = DriverManager.getConnection(
         "jdbc:arrow-flight-sql://localhost:47470", props);
     Statement stmt = jdbc.createStatement();
     ResultSet rs = stmt.executeQuery(
         "SELECT mask_email(email) FROM users LIMIT 5")) {

    while (rs.next()) System.out.println(rs.getString(1));
}
```

Discovery still works the same way — `SHOW FUNCTIONS` reports `scope=USER`, and `DESCRIBE FUNCTION mask_email` shows `owner=<your userId>`.

### Drop a USER UDF

Either through the SDK:

```java
session.unregisterPersistentUdf("mask_email");
```

Or via standard SQL DDL — useful from JDBC admin tools:

```sql
DROP FUNCTION mask_email;
```

---

## 5. GLOBAL UDF — visible to everyone, gated by IAM

GLOBAL UDFs are visible to every authenticated user on the cluster, subject to the `UDF:EXECUTE` IAM policy. Authoring requires `UDF:CREATE_GLOBAL`; dropping requires `UDF:DROP_GLOBAL`. The default `AdministratorAccess` policy includes both.

```java
// Admin authors the global function
OntulSession admin = OntulSession.builder().master("localhost", 47470).build();
admin.registerGlobalUdf("mask_email", Helpers.class, "maskEmail");
admin.close();
```

### Grant non-admin users `UDF:EXECUTE`

Create a policy in the Admin UI (`Settings → IAM → Policies`) — there are pre-built templates *UDF Execute Any*, *UDF Sandbox*, *UDF Deny Sensitive* — or via the REST API:

```bash
curl -X POST http://localhost:8080/admin/iam/policies \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "UdfExecuteAll",
    "document": {
      "Version": "2024-01-01",
      "Statement": [{
        "Sid": "ExecuteAnyUdf",
        "Effect": "Allow",
        "Action": "UDF:EXECUTE",
        "Resource": "udf:*"
      }]
    }
  }'
```

Attach the policy to the group containing your non-admin users:

```bash
curl -X POST http://localhost:8080/admin/iam/attach-group-policy \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"groupName":"analysts","policyName":"UdfExecuteAll"}'
```

### Alice (different user) calls the GLOBAL UDF

From the SDK:

```java
OntulSession alice = OntulSession.builder().master("localhost", 47470).build();
DataFrame df = alice.source(Source.sql(
    "SELECT mask_email('alice@example.com') AS m"));
// → "*****@example.com"
```

From JDBC:

```java
try (Connection jdbc = DriverManager.getConnection(
         "jdbc:arrow-flight-sql://localhost:47470", aliceProps);
     Statement stmt = jdbc.createStatement();
     ResultSet rs = stmt.executeQuery(
         "SELECT name, mask_email(email) AS masked FROM users LIMIT 10")) {
    while (rs.next()) System.out.println(rs.getString(1) + " | " + rs.getString(2));
}
```

If a user lacks `UDF:EXECUTE` for that resource, the planner denies the query *before* any data is touched:

```
Access denied: user 'bob' cannot UDF:EXECUTE on 'udf:mask_email'
```

### Restricting which UDFs a user may execute

Use a more specific resource pattern:

```json
{
  "Sid": "AnalystSandbox",
  "Effect": "Allow",
  "Action": "UDF:EXECUTE",
  "Resource": ["udf:mask_*", "udf:hash_*", "udf:length_class"]
}
```

Or layer an explicit deny on top of a broad allow:

```json
{
  "Statement": [
    { "Sid": "AllowAll",     "Effect": "Allow", "Action": "UDF:EXECUTE", "Resource": "udf:*" },
    { "Sid": "DenyInternal", "Effect": "Deny",  "Action": "UDF:EXECUTE", "Resource": ["udf:internal_*", "udf:*_admin"] }
  ]
}
```

### Drop a GLOBAL UDF

From the SDK:

```java
admin.unregisterGlobalUdf("mask_email");
```

Or via SQL — JDBC-friendly:

```sql
DROP GLOBAL FUNCTION mask_email;
```

Both paths require `UDF:DROP_GLOBAL` for the resource. A user without the permission gets a clean denial instead of touching the metadata store.

---

## 6. Side-by-side comparison

```java
// All three forms, same Java function, same SQL surface:
session.registerUdf            ("mask_email", Helpers.class, "maskEmail");  // TEMPORARY
session.registerPersistentUdf  ("mask_email", Helpers.class, "maskEmail");  // USER
session.registerGlobalUdf      ("mask_email", Helpers.class, "maskEmail");  // GLOBAL
```

| Question | TEMPORARY | USER | GLOBAL |
|---|---|---|---|
| Visible in another SDK session of the same user? | ❌ | ✅ | ✅ |
| Visible to JDBC connections of the same user? | ❌ | ✅ | ✅ |
| Visible to *other* users (with `UDF:EXECUTE`)? | ❌ | ❌ | ✅ |
| Survives master restart? | ❌ | ✅ | ✅ |
| Authoring needs IAM permission? | none | `UDF:CREATE` | `UDF:CREATE_GLOBAL` |
| Drop SQL DDL | n/a | `DROP FUNCTION name` | `DROP GLOBAL FUNCTION name` |
| Best for | notebooks, ad-hoc analysis, REPL | per-user helpers, personal model wrappers | shared org-wide functions: PII masking, geo helpers, ML inference |

## 7. Where to go next

- **[UDF feature page](../features/udf.md)** — full reference: Java return-type matrix (primitive, list, struct, byte[]), Python UDFs via cloudpickle, IAM action vocabulary, architecture notes (Calcite integration, batch-mode Python execution, auto-upload, persistence).
- **[IAM feature page](../features/iam.md)** — policy syntax, wildcards, condition keys, row/column-level filters that compose with `UDF:EXECUTE`.
- **[Admin UI](../features/admin-ui.md)** — visual policy editor with pre-built UDF templates and the function catalog browser backed by `SHOW FUNCTIONS`.
