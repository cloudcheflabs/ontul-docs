# IAM Policy Templates

Ontul's Admin UI policy editor ships with a set of ready-to-use **policy templates**. Each is a complete policy document you can attach to a user or group as-is, or use as a starting point. This page shows every template, explains when to use it, and documents the action/resource namespaces the policies are written against.

For the underlying concepts (users, groups, attachment, column masking, row filters, evaluation order), see [Identity and Access Management](iam.md).

## Policy document shape

Every policy is a JSON document with a `Version` and a list of `Statement`s. Each statement has an `Effect` (`Allow` / `Deny`), one or more `Action`s, and one or more `Resource`s; optional `Columns` and `Condition` add column- and row-level control.

```json
{
  "Version": "2024-01-01",
  "Statement": [
    {
      "Sid": "ReadOnlyAccess",
      "Effect": "Allow",
      "Action": "data:Select",
      "Resource": "data:table:*.*.*"
    }
  ]
}
```

**Evaluation order is `Deny > Mask > Allow`** — an explicit `Deny` always wins, then column masks / row filters apply, then `Allow` grants. The same policy store governs every access path (Arrow Flight SQL, REST, MCP), so one statement covers all clients.

### Action namespace

| Action | Meaning |
| --- | --- |
| `data:Select` | Read rows |
| `data:Insert` / `data:Update` / `data:Delete` / `data:Merge` | Write / row-level DML |
| `data:CreateTable` / `data:DropTable` / `data:AlterTable` | DDL |
| `data:KillJob` / `data:CancelQuery` | Job / query control |
| `UDF:EXECUTE` / `UDF:CREATE` / `UDF:DROP` | User-defined function use / authoring |
| `UDF:CREATE_GLOBAL` / `UDF:DROP_GLOBAL` | Global (cluster-wide) UDF authoring |
| `*` | Everything (administrator) |

### Resource namespace

| Resource | Pattern | Example |
| --- | --- | --- |
| Table | `data:table:<catalog>.<schema>.<table>` | `data:table:ice.*.*` |
| Schema | `data:schema:<catalog>.<schema>` | `data:schema:ice.team_a` |
| Job / query | `data:job:<id>` / `data:query:<id>` | `data:job:*` |
| UDF | `udf:<name>` | `udf:mask_*` |
| Everything | `*` | `*` |

Wildcards (`*`) may appear in any segment. These strings line up 1:1 with what the query engine emits at runtime, so a template's `Resource` matches exactly what an authorization check produces.

---

## Broad access

### AdministratorAccess

Full access to everything. Reserved — it cannot be deleted.

```json
{
  "Version": "2024-01-01",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "*",
      "Resource": "*"
    }
  ]
}
```

### Superuser

Same effect as `AdministratorAccess`, but a normal (deletable) policy — use it when you want an admin-equivalent grant you can later revoke.

```json
{
  "Version": "2024-01-01",
  "Statement": [
    {
      "Sid": "SuperuserAccess",
      "Effect": "Allow",
      "Action": "*",
      "Resource": "*"
    }
  ]
}
```

### Read-Only Analyst

Read any table in any catalog — the typical BI / analyst grant.

```json
{
  "Version": "2024-01-01",
  "Statement": [
    {
      "Sid": "ReadOnlyAccess",
      "Effect": "Allow",
      "Action": "data:Select",
      "Resource": "data:table:*.*.*"
    }
  ]
}
```

### App Read-Write

Full DML (no DDL) across all tables — for application service accounts that read and write but never change schema.

```json
{
  "Version": "2024-01-01",
  "Statement": [
    {
      "Sid": "AppReadWrite",
      "Effect": "Allow",
      "Action": ["data:Select", "data:Insert", "data:Update", "data:Delete"],
      "Resource": "data:table:*.*.*"
    }
  ]
}
```

### Schema Admin (DDL + DML)

DML plus DDL (`CreateTable` / `DropTable` / `AlterTable` / `Merge`) across all tables — for data engineers who own table lifecycle.

```json
{
  "Version": "2024-01-01",
  "Statement": [
    {
      "Sid": "SchemaAdmin",
      "Effect": "Allow",
      "Action": [
        "data:Select",
        "data:Insert",
        "data:Update",
        "data:Delete",
        "data:CreateTable",
        "data:DropTable",
        "data:AlterTable",
        "data:Merge"
      ],
      "Resource": "data:table:*.*.*"
    }
  ]
}
```

---

## Catalog-scoped access

Scope grants to a single connector by using the catalog segment of the resource (`ice.*.*`, `pg.*.*`, `kafka.*.*`, …).

### Iceberg Read-Only

```json
{
  "Version": "2024-01-01",
  "Statement": [
    {
      "Sid": "IcebergReadOnly",
      "Effect": "Allow",
      "Action": "data:Select",
      "Resource": "data:table:ice.*.*"
    }
  ]
}
```

### Iceberg Read-Write

```json
{
  "Version": "2024-01-01",
  "Statement": [
    {
      "Sid": "IcebergDML",
      "Effect": "Allow",
      "Action": ["data:Select", "data:Insert", "data:Update", "data:Delete", "data:Merge"],
      "Resource": "data:table:ice.*.*"
    }
  ]
}
```

### JDBC Catalog Read-Only

```json
{
  "Version": "2024-01-01",
  "Statement": [
    {
      "Sid": "JdbcReadOnly",
      "Effect": "Allow",
      "Action": "data:Select",
      "Resource": "data:table:pg.*.*"
    }
  ]
}
```

### JDBC Catalog Read-Write

```json
{
  "Version": "2024-01-01",
  "Statement": [
    {
      "Sid": "JdbcDML",
      "Effect": "Allow",
      "Action": ["data:Select", "data:Insert", "data:Update", "data:Delete"],
      "Resource": "data:table:pg.*.*"
    }
  ]
}
```

### Cross-Catalog (Iceberg + JDBC)

Different permissions per catalog in one policy — e.g. full write to Iceberg, insert-only to JDBC.

```json
{
  "Version": "2024-01-01",
  "Statement": [
    {
      "Sid": "IcebergAccess",
      "Effect": "Allow",
      "Action": ["data:Select", "data:Insert", "data:Merge"],
      "Resource": "data:table:ice.*.*"
    },
    {
      "Sid": "JdbcAccess",
      "Effect": "Allow",
      "Action": ["data:Select", "data:Insert"],
      "Resource": "data:table:pg.*.*"
    }
  ]
}
```

### Kafka Stream Consumer

Read-only on a Kafka catalog — for streaming source consumers.

```json
{
  "Version": "2024-01-01",
  "Statement": [
    {
      "Sid": "KafkaConsume",
      "Effect": "Allow",
      "Action": "data:Select",
      "Resource": "data:table:kafka.*.*"
    }
  ]
}
```

### Namespace Isolation

Confine a team to a single schema (namespace) — everything within `ice.team_a`, nothing outside it.

```json
{
  "Version": "2024-01-01",
  "Statement": [
    {
      "Sid": "NamespaceAccess",
      "Effect": "Allow",
      "Action": "*",
      "Resource": ["data:table:ice.team_a.*", "data:schema:ice.team_a"]
    }
  ]
}
```

---

## Guardrails (Deny)

`Deny` statements always win, so they enforce guardrails on top of broader `Allow`s.

### Audit-Safe (Deny Modify Logs)

Allow DML everywhere, but forbid updating/deleting audit and log tables (matched by name pattern).

```json
{
  "Version": "2024-01-01",
  "Statement": [
    {
      "Sid": "AllowDML",
      "Effect": "Allow",
      "Action": ["data:Select", "data:Insert", "data:Update", "data:Delete"],
      "Resource": "data:table:*.*.*"
    },
    {
      "Sid": "DenyModifyAudit",
      "Effect": "Deny",
      "Action": ["data:Update", "data:Delete"],
      "Resource": ["data:table:*.*.audit_*", "data:table:*.*.*_logs"]
    }
  ]
}
```

---

## Column- and row-level control

These express Ontul's column deny, column masking, and row filtering. The Admin UI ships the allow-list and row-filter forms as one-click templates; the **Column Deny** and **Column Mask** forms below are the same policy shapes, written by hand (or edited from a template). All apply server-side inside the query plan. See [Column-Level Security](iam.md#column-level-security).

### Column-Restricted Read (allow-list)

Grant `Select` on a table but expose only specific columns via `Columns` on an **Allow** — all other columns are hidden.

```json
{
  "Version": "2024-01-01",
  "Statement": [
    {
      "Sid": "LimitedColumns",
      "Effect": "Allow",
      "Action": "data:Select",
      "Resource": "data:table:pg.public.employees",
      "Columns": ["id", "name", "department"]
    }
  ]
}
```

### Column Deny (hide specific columns)

The inverse of the allow-list: `Columns` on a **Deny** drops just those columns from the scan output, so a `SELECT *` returns every column except the denied ones.

```json
{
  "Version": "2024-01-01",
  "Statement": [
    {
      "Sid": "HideSsnFromAnalysts",
      "Effect": "Deny",
      "Action": "data:Select",
      "Resource": "data:table:hr.core.employees",
      "Columns": ["ssn"]
    }
  ]
}
```

### Column Mask

Replace a column's value with a SQL expression via `MaskedColumns` (`Effect: "Mask"`). The output column keeps its original name and type — joins, `WHERE`, and aggregations still work, just on the masked value. The expression is evaluated at the worker and may reference the original column and `${user.attr.*}` attributes.

```json
{
  "Version": "2024-01-01",
  "Statement": [
    {
      "Sid": "MaskHrPii",
      "Effect": "Mask",
      "Action": "data:Select",
      "Resource": "data:table:hr.core.employees",
      "MaskedColumns": {
        "ssn": "'***-**-' || substr(ssn, -4)",
        "email": "regexp_replace(email, '(^.).*(@.*$)', '$1***$2')",
        "salary": "0"
      }
    }
  ]
}
```

Masks can be conditional — e.g. unmask for one role and mask everyone else:

```json
{
  "Version": "2024-01-01",
  "Statement": [
    {
      "Sid": "ConditionalUnmaskSsn",
      "Effect": "Mask",
      "Action": "data:Select",
      "Resource": "data:table:hr.core.employees",
      "MaskedColumns": {
        "ssn": "CASE WHEN '${user.attr.role}' = 'compliance' THEN ssn ELSE '***-**-XXXX' END"
      }
    }
  ]
}
```

!!! note
    Precedence is **Deny > Mask > Allow**. A column named in both a Deny and a Mask statement disappears entirely — the mask never fires.

### Row-Level Filter

Attach a `Condition` so the user only sees matching rows — the predicate is injected into every query against the table. Templated conditions like `department = '${user.attr.department}'` resolve per user. See [Row-Level Security](iam.md#row-level-security).

```json
{
  "Version": "2024-01-01",
  "Statement": [
    {
      "Sid": "DeptFilter",
      "Effect": "Allow",
      "Action": "data:Select",
      "Resource": "data:table:pg.public.employees",
      "Condition": "department = 'Engineering'"
    }
  ]
}
```

---

## Job control

### Job Submitter

Read everything, and write/create tables in Iceberg — the grant a batch/streaming job's identity needs.

```json
{
  "Version": "2024-01-01",
  "Statement": [
    {
      "Sid": "ReadAllCatalogs",
      "Effect": "Allow",
      "Action": "data:Select",
      "Resource": "data:table:*.*.*"
    },
    {
      "Sid": "WriteIceberg",
      "Effect": "Allow",
      "Action": ["data:Insert", "data:Merge", "data:CreateTable"],
      "Resource": "data:table:ice.*.*"
    }
  ]
}
```

### Streaming Flow (Ontul Flow)

Starting an [Ontul Flow](../streaming/flow.md) enforces the same table-level RBAC as SQL DML: the
identity needs `data:Select` on any Ontul-catalog **source** table and write access on the Ontul-catalog
**sink** table — `data:Insert` for an append flow, plus `data:Update` / `data:Delete` for a CDC / upsert
write mode. External systems (Kafka, CDC databases, JDBC, Elasticsearch, webhooks, NeorunBase) are
reached through their registered connection credentials, not IAM. This template grants a Flow that reads
any table and maintains an Iceberg CDC replica.

```json
{
  "Version": "2024-01-01",
  "Statement": [
    {
      "Sid": "FlowReadSources",
      "Effect": "Allow",
      "Action": "data:Select",
      "Resource": "data:table:*.*.*"
    },
    {
      "Sid": "FlowWriteIcebergSink",
      "Effect": "Allow",
      "Action": ["data:Insert", "data:Update", "data:Delete"],
      "Resource": "data:table:ice.*.*"
    }
  ]
}
```

For an **append-only** flow (no CDC / upsert), the sink statement needs only `data:Insert`. Scope the
`Resource` segments down (`ice.sales.*`, a single table) to confine which sinks a Flow identity may
write.

### Job Manager

Kill jobs / cancel queries plus read access — for operators managing running work.

```json
{
  "Version": "2024-01-01",
  "Statement": [
    {
      "Sid": "JobManagement",
      "Effect": "Allow",
      "Action": ["data:KillJob", "data:CancelQuery"],
      "Resource": ["data:job:*", "data:query:*"]
    },
    {
      "Sid": "ReadAllCatalogs",
      "Effect": "Allow",
      "Action": "data:Select",
      "Resource": "data:table:*.*.*"
    }
  ]
}
```

---

## User-Defined Functions

UDF resources are `udf:<name>`, so `udf:*` covers all functions and a prefix like `udf:mask_*` scopes to a family. See the [UDF feature page](udf.md).

### UDF Execute Any

```json
{
  "Version": "2024-01-01",
  "Statement": [
    {
      "Sid": "UdfExecAny",
      "Effect": "Allow",
      "Action": "UDF:EXECUTE",
      "Resource": "udf:*"
    }
  ]
}
```

### UDF Author (Create + Drop + Execute)

```json
{
  "Version": "2024-01-01",
  "Statement": [
    {
      "Sid": "UdfAuthor",
      "Effect": "Allow",
      "Action": ["UDF:CREATE", "UDF:DROP", "UDF:EXECUTE"],
      "Resource": "udf:*"
    }
  ]
}
```

### UDF Sandbox (Execute Specific Functions)

Execute only an allow-listed set of functions — nothing else.

```json
{
  "Version": "2024-01-01",
  "Statement": [
    {
      "Sid": "UdfSandbox",
      "Effect": "Allow",
      "Action": "UDF:EXECUTE",
      "Resource": ["udf:mask_*", "udf:hash_*", "udf:length_class"]
    }
  ]
}
```

### UDF Deny Sensitive

Execute any function except a denied family (admin/internal), enforced by a `Deny` overlay.

```json
{
  "Version": "2024-01-01",
  "Statement": [
    {
      "Sid": "UdfBaseExec",
      "Effect": "Allow",
      "Action": "UDF:EXECUTE",
      "Resource": "udf:*"
    },
    {
      "Sid": "UdfDenyAdmin",
      "Effect": "Deny",
      "Action": "UDF:EXECUTE",
      "Resource": ["udf:*_admin", "udf:internal_*"]
    }
  ]
}
```

### UDF Global Admin (Create + Drop GLOBAL)

Authoring rights for **global** (cluster-wide) UDFs, plus execute.

```json
{
  "Version": "2024-01-01",
  "Statement": [
    {
      "Sid": "UdfGlobalAdmin",
      "Effect": "Allow",
      "Action": ["UDF:CREATE_GLOBAL", "UDF:DROP_GLOBAL", "UDF:EXECUTE"],
      "Resource": "udf:*"
    }
  ]
}
```

### UDF Global Read-Only (Execute Only)

Execute-only — for consumers who use global UDFs but cannot author them.

```json
{
  "Version": "2024-01-01",
  "Statement": [
    {
      "Sid": "UdfGlobalExec",
      "Effect": "Allow",
      "Action": "UDF:EXECUTE",
      "Resource": "udf:*"
    }
  ]
}
```

---

## Applying a template

Templates are starting points — attach one directly, or edit it in the Admin UI policy editor and save under a new `name`. Policies attach to a user or a group (a group policy applies to all members). See [Creating and updating policies](iam.md#policy-based-access-control) for the REST/CLI flow.
