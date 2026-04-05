# Identity and Access Management

Ontul includes a built-in IAM system that provides authentication, authorization, and fine-grained access control down to the column and row level.

## Authentication

Ontul supports multiple authentication methods:

- **Username / Password**: Basic authentication with PBKDF2-hashed passwords
- **Access Keys**: Long-lived credentials (AKIA prefix) with access key ID and secret key for programmatic access
- **STS Temporary Credentials**: Short-lived credentials (ASIA prefix) with configurable expiration for temporary access
- **JWT Tokens**: Bearer token authentication for Arrow Flight SQL and REST API

## Users and Groups

- Create and manage database users with passwords and metadata
- Organize users into IAM groups
- Policies attached to a group apply to all members

## Policy-Based Access Control

Ontul uses AWS-style JSON policies to manage permissions:

```json
{
  "statements": [
    {
      "effect": "ALLOW",
      "actions": ["SELECT", "INSERT"],
      "resources": ["iceberg_catalog.sales.*"]
    },
    {
      "effect": "DENY",
      "actions": ["SELECT"],
      "resources": ["iceberg_catalog.hr.salaries"],
      "columns": ["salary", "ssn"]
    }
  ]
}
```

- Policies define allowed or denied actions on specific resources (catalogs, schemas, tables)
- Policies can be attached to users or groups
- Deny rules take precedence over allow rules
- Deny-by-default: no matching policies means access is denied

## Column-Level Security

Restrict access to specific columns within a table. Denied columns are automatically removed from query results — no changes to the query required.

## Row-Level Security

Apply row filter conditions to restrict which rows a user can see:

```json
{
  "effect": "ALLOW",
  "actions": ["SELECT"],
  "resources": ["iceberg_catalog.sales.orders"],
  "rowFilter": "region = 'APAC'"
}
```

Row filters are injected into the query plan automatically, ensuring users only see data they are authorized to access.

## Management

Users, groups, and policies are managed through the Admin UI (with a visual policy editor) or the REST API.
