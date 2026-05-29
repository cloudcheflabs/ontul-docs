# Backup & Restore

Ontul exports the cluster's stateful stores — **KMS keystore**, **IAM** (users, policies, access keys), and the **catalog metadata store** — to an S3 bucket of your choosing, and restores them in a coordinated way that blocks requests, broadcasts the recovered state to followers, and only marks the cluster ready once every node has caught up.

Three trigger modes share the same backup/restore implementation:

| Trigger | Where it's configured | When to use |
|---|---|---|
| **Manual** | Admin UI → *Backup & Restore* → *Backup Now*, or `POST /admin/backup/run` | Before destructive ops (schema migrations, KMS key rotation) |
| **Fixed-interval** | `intervalHours` field on `/admin/backup/configure` | Safety net: "every N hours, no matter what" |
| **Cron** | `/admin/backup/cron`, or the *Cron Schedule* card in the Admin UI | Wall-clock schedules: "every night at 02:00" |

The two automatic modes coexist — if both are set, ontul fires both. Clear one (`intervalHours: 0` or empty cron) to use only the other.

## Setting up S3 storage

Backups go to an S3 bucket (or any S3-compatible store like shannonstore) accessed through an entry in the **Connection Store**:

```bash
curl -X POST http://localhost:8080/admin/connections \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "connectionId": "ontul-backup-s3",
    "type": "S3",
    "properties": {
      "endpoint": "https://s3.us-east-1.amazonaws.com",
      "region": "us-east-1",
      "bucket": "my-ontul-backups",
      "accessKey": "AKIA…",
      "secretKey": "…",
      "pathStyle": "false"
    }
  }'
```

Then tell the backup service which connection + prefix to use:

```bash
curl -X POST http://localhost:8080/admin/backup/configure \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "connectionId": "ontul-backup-s3",
    "s3Prefix": "s3://my-ontul-backups/cluster",
    "intervalHours": 0
  }'
```

The settings are persisted in the metadata store and survive restarts — a fresh leader picks up the same target without re-running `configure`.

## Cron-driven backup

Use a 5-field UNIX cron expression (same grammar as the kiok DAG scheduler):

```bash
curl -X POST http://localhost:8080/admin/backup/cron \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"cron": "0 2 * * *"}'
```

The leader-side scheduler ticks every 30 seconds and fires `backup()` once when the cron expression has a fire time in the window `(lastChecked, now]`. The first tick after the cron is set, or after a leader handoff, arms from "now" — ontul will **not** back-fire missed schedules on startup; recovering a missed nightly backup three days late tends to surprise more than skipping it does. Clear the schedule with an empty body:

```bash
curl -X POST http://localhost:8080/admin/backup/cron \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"cron": null}'
```

The cron expression persists under `backup.cron` in the metadata store, so a restart or leader change re-arms the same schedule.

The combined-form endpoint also accepts cron, useful for setting everything in one round-trip from the Admin UI:

```bash
curl -X POST http://localhost:8080/admin/backup/configure \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "connectionId": "ontul-backup-s3",
    "s3Prefix": "s3://my-ontul-backups/cluster",
    "intervalHours": 0,
    "cron": "0 2 * * *"
  }'
```

## Layout of a backup

Each run writes to `<s3Prefix>/<timestamp>/`:

```
s3://my-ontul-backups/cluster/20260529-120000/
  ├── kms.bin            # encrypted KMS keystore
  ├── iam.json           # users, policies, access keys
  └── metadata.json      # catalog metadata snapshot
```

The history is exposed at `GET /admin/backup/history` (last 100 entries) and the available backup ids at `GET /admin/backup/list` (newest first).

## Restore

Restore is destructive — it overwrites the current stores. Ontul runs it in three phases on the leader:

1. **Block requests.** Marks the leader not-ready (`/admin/ready` returns 503) so clients fail fast instead of querying half-restored state.
2. **Download + import.** Pulls each store from S3 and imports it into KMS / IAM / Metadata in-process.
3. **Broadcast + wait.** Pushes the recovered KMS keystore, IAM snapshot, and metadata to every follower over the internal NIO protocol, then waits for the cluster-ready signal before re-accepting traffic.

```bash
curl -X POST http://localhost:8080/admin/backup/restore \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"backupId": "20260529-120000"}'
```

The Admin UI exposes this as a *Restore* button on each entry in the *Available Backups* list, behind an explicit confirmation modal that names every store about to be overwritten.

## Endpoint reference

| Method | Path | Body | Purpose |
|---|---|---|---|
| `GET` | `/admin/backup/config` | — | Current config + scheduler state (`cron`, `cronScheduled`, `cronNextFireMillis`) |
| `POST` | `/admin/backup/configure` | `{connectionId, s3Prefix, intervalHours, cron?}` | Set connection / prefix / fixed-interval / cron in one call |
| `POST` | `/admin/backup/cron` | `{cron: string \| null}` | Set or clear the cron schedule |
| `POST` | `/admin/backup/run` | — | Trigger a backup immediately |
| `POST` | `/admin/backup/restore` | `{backupId}` | Restore from a specific backup id |
| `GET` | `/admin/backup/list` | — | Backup ids in S3, newest first |
| `GET` | `/admin/backup/history` | — | Last 100 history entries (backup + restore) |

The Admin UI's *Backup & Restore* page wires the same endpoints; the underlying state survives master restarts.
