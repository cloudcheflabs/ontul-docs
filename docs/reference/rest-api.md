# REST API Reference

All REST API endpoints require authentication via JWT Bearer token unless noted otherwise.

## Authentication

### Login

```
POST /admin/auth/login
```

```json
{"username": "admin", "password": "your-password"}
```

Response:

```json
{
  "accessToken": "eyJ...",
  "refreshToken": "uuid-string",
  "username": "admin",
  "isAdmin": true,
  "requirePasswordChange": false
}
```

### Change Password

```
POST /admin/auth/change-password
Authorization: Bearer <accessToken>
```

```json
{"oldPassword": "admin", "newPassword": "new-secure-password"}
```

### Refresh Token

```
POST /admin/auth/refresh
```

```json
{"refreshToken": "uuid-string"}
```

### Session Info

```
GET /admin/auth/session
Authorization: Bearer <accessToken>
```

---

## SQL Query

### Execute SQL

```
POST /admin/query/execute
Authorization: Bearer <accessToken>
```

```json
{"sql": "SELECT * FROM tpch.tiny.customer LIMIT 5"}
```

Response:

```json
{
  "queryId": "uuid",
  "status": "ok",
  "commandTag": "SELECT",
  "elapsedMs": 245,
  "columns": [
    {"name": "c_custkey", "type": "Int(64, true)"},
    {"name": "c_name", "type": "Utf8"}
  ],
  "rows": [[1, "Customer#000000001"], [2, "Customer#000000002"]],
  "rowCount": 2
}
```

### Execute SQL (SDK Endpoint)

```
POST /v1/api/sql
Authorization: Bearer <accessToken>
```

Same request/response format as `/admin/query/execute`.

### Cancel Query

```
POST /admin/query/cancel
Authorization: Bearer <accessToken>
```

```json
{"queryId": "uuid"}
```

---

## Catalog Management

### List Catalogs

```
GET /admin/catalogs
Authorization: Bearer <accessToken>
```

Response:

```json
[
  {
    "name": "tpch",
    "connectorType": "tpch",
    "config": {"connector": "tpch", "scale.factor": "0.01"},
    "tableCount": 16
  }
]
```

### Register Catalog

```
POST /admin/catalogs
Authorization: Bearer <accessToken>
```

```json
{
  "name": "iceberg",
  "config": "{\"connector\":\"iceberg\",\"catalog-type\":\"rest\",\"uri\":\"http://polaris:8181/api/catalog\",\"warehouse\":\"my_catalog\",\"credential\":\"root:secret\",\"scope\":\"PRINCIPAL_ROLE:ALL\",\"io-impl\":\"org.apache.iceberg.aws.s3.S3FileIO\",\"s3.endpoint\":\"http://s3:9000\",\"s3.path-style-access\":\"true\",\"s3.accessKey\":\"ACCESS_KEY\",\"s3.secretKey\":\"SECRET_KEY\"}"
}
```

Note: The `config` field is a **JSON string**, not a nested object.

### Update Catalog

```
PUT /admin/catalogs/{name}
Authorization: Bearer <accessToken>
```

Same body as Register.

### Delete Catalog

```
DELETE /admin/catalogs/{name}
Authorization: Bearer <accessToken>
```

### Refresh Catalog

Re-discover tables without removing existing ones.

```
POST /admin/catalogs/{name}/refresh
Authorization: Bearer <accessToken>
```

---

## Connection Management

### List Connections

```
GET /admin/connections
Authorization: Bearer <accessToken>
```

### Create Connection

```
POST /admin/connections
Authorization: Bearer <accessToken>
```

```json
{
  "id": "my-s3-conn",
  "type": "s3",
  "properties": {
    "endpoint": "http://s3:9000",
    "accessKey": "ACCESS_KEY",
    "secretKey": "SECRET_KEY",
    "region": "us-east-1",
    "pathStyle": "true"
  }
}
```

### Test Connection

```
POST /admin/connections/test
Authorization: Bearer <accessToken>
```

---

## Job Management

### Submit Job

```
POST /v1/api/job/submit
Authorization: Bearer <accessToken>
```

**Batch job:**

```json
{
  "name": "my-batch-job",
  "type": "BATCH",
  "sql": "INSERT INTO iceberg.db.target SELECT * FROM tpch.tiny.customer"
}
```

**Streaming job:**

```json
{
  "name": "kafka-to-iceberg",
  "type": "STREAMING",
  "config": {
    "source.type": "kafka",
    "source.kafka.bootstrap.servers": "kafka:9092",
    "source.kafka.topic": "events",
    "source.kafka.group.id": "ontul-group",
    "source.kafka.auto.offset.reset": "earliest",
    "sink.type": "table",
    "sink.table": "iceberg.db.events"
  },
  "operations": [
    {"type": "FILTER", "value": "amount > 100"},
    {"type": "WINDOW", "value": "TUMBLING(SIZE 10 SECONDS)"},
    {"type": "GROUP_BY", "value": "category"},
    {"type": "AGG", "value": "SUM(amount) as total, COUNT(*) as cnt"}
  ]
}
```

### Job Status

```
GET /v1/api/job/status/{jobId}
Authorization: Bearer <accessToken>
```

### Job Log

```
GET /v1/api/job/{jobId}/log
Authorization: Bearer <accessToken>
```

Supports `?from=N` for incremental log fetch. For streaming jobs running on multiple Workers, logs from all Workers are merged with `[nodeId]` prefix.

### Kill Job

```
POST /v1/api/job/kill/{jobId}
Authorization: Bearer <accessToken>
```

### List Jobs

```
GET /v1/api/job/list
Authorization: Bearer <accessToken>
```

---

## IAM

### Users

```
GET    /admin/iam/users                              # List users
POST   /admin/iam/users                              # Create user
DELETE /admin/iam/users/{username}                    # Delete user
```

Create user:

```json
{"username": "analyst", "password": "secure-password"}
```

### Access Keys

```
POST   /admin/iam/keys                               # Generate access key
DELETE /admin/iam/users/{username}/keys/{accessKeyId}  # Delete key
```

### Groups

```
GET    /admin/iam/groups                              # List groups
POST   /admin/iam/groups                              # Create group
DELETE /admin/iam/groups/{name}                        # Delete group
POST   /admin/iam/add-user-to-group                   # Add user to group
POST   /admin/iam/remove-user-from-group              # Remove from group
```

### Policies

```
GET    /admin/iam/policies                            # List policies
POST   /admin/iam/policies                            # Create policy
DELETE /admin/iam/policies/{name}                      # Delete policy
POST   /admin/iam/attach-group-policy                  # Attach to group
POST   /admin/iam/detach-group-policy                  # Detach from group
```

Policy document example:

```json
{
  "name": "ReadOnlyIceberg",
  "document": "{\"Version\":\"2024-01-01\",\"Statement\":[{\"Effect\":\"Allow\",\"Action\":\"SELECT\",\"Resource\":\"iceberg.*\"}]}"
}
```

---

## KMS

```
GET    /admin/kms/list                                # List encryption keys
GET    /admin/kms/status/{keyId}                      # Key status
POST   /admin/kms/create                              # Create key
POST   /admin/kms/rotate/{keyId}                      # Rotate key
```

---

## Cluster

### Health Check

```
GET /admin/health          # Liveness (no auth required)
GET /admin/ready           # Readiness (cluster-wide)
```

### Nodes

```
GET /admin/nodes/masters   # List master nodes
GET /admin/nodes/workers   # List worker nodes
```

### Metrics

```
GET /metrics               # Prometheus format (no auth required)
```

---

## Drivers

Upload external JDBC driver JARs for plugin connectors.

```
GET    /admin/drivers                                 # List drivers
POST   /admin/drivers                                 # Upload driver JAR
DELETE /admin/drivers/{fileName}                       # Delete driver
```

---

## Maintenance

Iceberg table maintenance (snapshot expiry, compaction, orphan cleanup).

```
GET    /admin/maintenance/config                      # View config
PUT    /admin/maintenance/config                      # Update config
POST   /admin/maintenance/run                         # Trigger maintenance
GET    /admin/maintenance/history                     # Job history
```

---

## Backup & Restore

```
POST   /admin/backup/configure                        # Configure backup
POST   /admin/backup/run                              # Run backup
POST   /admin/backup/restore                          # Restore from backup
GET    /admin/backup/list                             # List backups
GET    /admin/backup/config                           # View config
GET    /admin/backup/history                          # Backup history
```
