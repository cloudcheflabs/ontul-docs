# Backup & Restore

Ontul exports the cluster's stateful stores — **KMS keystore**, **IAM** (users, policies, access keys), the **catalog metadata store** and the **Connection Store** — to an S3 bucket of your choosing, and restores them in a coordinated way that blocks requests, broadcasts the recovered state to followers, and only marks the cluster ready once every node has caught up.

Three trigger modes share the same backup/restore implementation, and an optional [retention policy](#retention)
deletes old backups after each successful one (off by default: every backup is kept):

| Trigger | Where it's configured | When to use |
|---|---|---|
| **Manual** | Admin UI → *Backup & Restore* → *Backup Now*, or `POST /admin/backup/run` | Before destructive ops (schema migrations, KMS key rotation) |
| **Fixed-interval** | *Auto Backup Interval* in the Admin UI, or `intervalHours` on `/admin/backup/configure` | Safety net: "every N hours, no matter what" |
| **Cron** | *Cron Expression* in the Admin UI, or `cron` on `/admin/backup/configure` (also `/admin/backup/cron`) | Wall-clock schedules: "every night at 02:00" |

The two automatic modes are independent:

| Interval | Cron | What runs |
|---|---|---|
| `0` | empty | no automatic backup |
| `N` | empty | every N hours |
| `0` | `0 2 * * *` | the cron only |
| `N` | `0 2 * * *` | both (each fires on its own schedule) |

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

`configure` checks everything before it saves anything, and saves all of it or nothing:

- `connectionId` must name an existing **S3** connection and `s3Prefix` must be an `s3://bucket[/path]`
  location (a valid bucket name); `intervalHours` is `0`–`8784`; `cron`, when given, is a 5-field UNIX
  cron expression (the same grammar as table maintenance).
- The target is then checked with the connection's own keys — the bucket is read (HEAD + a one-key
  listing under the prefix) and a probe object is written under the prefix and deleted again — so a
  missing bucket, a read-only key or a wrong endpoint is reported when you save, not by the first
  scheduled backup. Turning every schedule off on the target already saved skips this check, so you can
  always stop backups to a bucket that has gone away.
- A refused save answers `400` with a message naming the field or the S3 reason, for example
  `s3Prefix is required: an S3 location such as s3://bucket/path` or
  `S3 check failed for s3://no-such-bucket/b with connection 'objstore': bucket 'no-such-bucket' is not readable: not found (no such bucket) [NoSuchBucket] — nothing was saved`.
- A successful save answers `{"status":"ok","config":{...}}` with the schedule now in force, including
  `intervalNextFireMillis` and `cronNextFireMillis` (0 when that schedule is off) and `cronTimeZone`.

`cron` is optional in the body: **absent** leaves the current cron schedule as it is (callers written
before cron existed), **`""` or `null`** clears it. The retention fields `retentionCount` and `retentionDays`
(see [Retention](#retention)) work the same way: absent leaves the stored policy unchanged; `retentionCount` is
`0`–`10000` and `retentionDays` is `0`–`3660`.

The settings are persisted in the metadata store and survive restarts. Every master re-reads them
(on each 30-second scheduler tick and whenever the configuration is read), so a master that becomes
leader runs — and reports — the schedule saved last, not the one it booted with.

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

The combined-form endpoint takes the cron too — this is what the Admin UI's **Save Config** sends:

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

## Retention

Without a retention policy every backup stays in the bucket forever. Two rules, both **off by default (`0`)**,
limit what is kept:

| Field | Admin UI | Meaning |
|---|---|---|
| `retentionCount` | *Keep Newest Backups* | Keep only the newest N backups. `0` = keep all. |
| `retentionDays` | *Delete Backups Older Than* (days) | Delete backups older than N days, by the time in their id. `0` = no age limit. |

With both at `0` (the default) nothing is ever deleted — the behaviour of earlier releases. When both are set, a
backup is deleted if **either** rule drops it (for example `5` and `30`: at most the newest five, and none older
than 30 days).

```bash
curl -X POST http://localhost:8080/admin/backup/configure \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "connectionId": "ontul-backup-s3",
    "s3Prefix": "s3://my-ontul-backups/cluster",
    "intervalHours": 0,
    "cron": "0 2 * * *",
    "retentionCount": 14,
    "retentionDays": 0
  }'
```

How it is applied:

- **After each successful backup** — manual, interval or cron — on the leader, with the backup's own S3
  connection. Saving a policy deletes nothing by itself; the next successful backup applies it. A failed backup
  deletes nothing.
- The backup **just written is never deleted**, so at least one backup always remains.
- Only folders whose name is a backup id (`yyyyMMdd-HHmmss`) are considered. Anything else under the prefix —
  a manual copy, a README — is never touched.
- Every object under a deleted backup's folder (`kms.bin`, `iam.bin`, `metadata.bin`, `connections.bin`, older
  `*.json` parts) is removed.
- Each run that deletes something adds a **`prune`** entry to the history, for example
  `Retention (keep newest 5): deleted 8 (20261030-120000, …)`. If deleting fails part-way, the entry is recorded
  as an error with what was already deleted; the backup itself stays successful.

The policy is stored in the metadata store (`backup.retentionCount`, `backup.retentionDays`), so it survives
restarts and leader changes like the schedule does. `GET /admin/backup/config` reports it as `retentionCount`
and `retentionDays`, and the Admin UI's status panel states it in words ("Retention: keeps the newest 5 backups;
older ones are deleted after each successful backup", or "Retention: every backup is kept").

## Layout of a backup

Each run writes to `<s3Prefix>/<timestamp>/`. **Every part is encrypted** — nothing in the bucket is readable
without the cluster's KMS master key:

```
s3://my-ontul-backups/cluster/20260529-120000/
  ├── kms.bin            # the KMS keystore as stored: wrapped with the cluster master key
  ├── iam.bin            # users, password hashes, groups, policies, access keys + secrets + tokens,
  │                      #   STS and federated sessions -- sealed with the `backup` KMS key
  ├── metadata.bin       # catalog configs (with their credentials), SSO settings, flows, every other
  │                      #   config key -- sealed with the `backup` KMS key
  └── connections.bin    # every connection, sealed with the `connections` KMS key
```

`iam.bin` and `metadata.bin` are KMS envelopes (a random data key per part, wrapped by the `backup` KMS key).
The `backup` key is created on the first backup and travels inside the same backup's `kms.bin`, which is itself
wrapped with the master key named by `ontul.kms.master.key.env` (`ONTUL_MASTER_KEY` in the examples). There is
no backup without KMS: a cluster with KMS disabled refuses to back up rather than write credentials in the
clear.

### Which key restores a backup

A backup restores on a cluster started with **the same KMS master key** as the cluster that wrote it — the
original cluster, or a rebuilt one given the same master key. Keep the master key with your disaster-recovery
material; without it a backup cannot be opened (that is the point). A restore loads the backup's `kms.bin`
first and decrypts **every** part before importing anything. A backup written under a different master key is
refused with `backup <id> cannot be decrypted with the keys in its kms.bin (...). It was written by a cluster
with a different KMS master key; restore it on a cluster started with that master key. Nothing was restored`
and the cluster keeps its current keystore and state.

!!! warning "Backups taken before this release are plaintext"
    Earlier releases wrote `iam.json` and `metadata.json` as **plaintext JSON**: every user's access key id,
    secret and token, password hashes, STS / federated sessions, and the catalog / SSO / flow configs with their
    S3 keys, OIDC client secret, LDAP bind password, SAML private key and inline JDBC passwords. Anyone who could
    read the backup bucket could read them. Those backups **still restore** (a backup without `iam.bin` /
    `metadata.bin` is read from the `.json` parts, with a warning in the master log), but you should:

    1. take a new backup (it is encrypted), then
    2. **delete** every old `<s3Prefix>/<id>/iam.json` and `metadata.json` object — they keep revoked keys and
       old secrets readable forever, and
    3. **rotate** what they exposed: users' access keys and tokens (delete and re-create them), user passwords,
       the SSO OIDC client secret, LDAP bind password and SAML signing key, and the credentials inside catalog,
       connection and flow configs, starting with the most privileged.

Backup ids are unique per master: a second backup started in the same second as the previous one takes the next free second, so two backups never write the same files.

The history is exposed at `GET /admin/backup/history`: the last 100 backup, restore and prune entries, newest
first. It is kept **in memory on the master that ran the operation** — the leader, since every backup endpoint
is forwarded to it — so it starts empty after that master restarts or another master becomes leader; the
backups themselves are in S3 and still listed.

The available backup ids are at `GET /admin/backup/list`, newest first. Without parameters it answers every id
as a JSON array. With `limit` (1–1000) and/or `offset` (default `limit` 10, `offset` 0) it answers one page:

```bash
curl "http://localhost:8080/admin/backup/list?limit=10&offset=10" -H "Authorization: Bearer $TOKEN"
# {"backups":["20261029-020000", ...],"total":23,"offset":10,"limit":10}
```

A `limit` outside 1–1000 or a negative `offset` answers `400`. The Admin UI reads the list this way: *Available
Backups* shows the newest 10 with *Previous* / *Next* and "Showing 11–20 of 23 backups", and *Job History* pages
its entries 10 at a time the same way.

`/admin/backup/list` checks that the bucket exists before listing it and pages through every backup under the prefix. When the target cannot be read — a missing bucket, wrong credentials, a deleted connection — it answers `502` with the storage error, and the Admin UI shows that error instead of "No backups found". With no backup configured it answers `[]`. *Backup Now* in the UI is enabled only once a configuration has been saved. Saving a configuration always re-arms the interval schedule (an interval of `0` turns it off). Since a configuration can only be saved after its bucket passed the check above, a `502` from the list now means the target broke after it was saved (bucket deleted, key revoked).

Scheduled and cron backups run on the **leader only**: every master keeps the schedule, but a follower's tick does nothing, so a three-master cluster takes one backup per run rather than three. All backup endpoints, including the reads, need administrator access.

## Restore

Restore is destructive — it overwrites the current stores. Ontul runs it in three phases on the leader:

1. **Block requests.** Marks the leader not-ready (`/admin/ready` returns 503) so clients fail fast instead of querying half-restored state.
2. **Download + import.** Pulls each store from S3, loads the backup's KMS keystore, decrypts every part (refusing the whole restore, with the previous keystore put back, if one does not decrypt), then imports IAM / Metadata / Connections in-process. The leader then reloads its catalogs, semantic views and storage settings from the restored metadata, so a catalog restored from the backup can be queried at once. The restored connections replace the current set: one created after the backup is removed, one deleted after it comes back with its credentials. A backup taken before connections were included has no `connections.bin`; restoring it keeps the current connections.
3. **Broadcast + wait.** Pushes the recovered KMS keystore, IAM snapshot, metadata and connections to every follower over the internal NIO protocol, then waits for the cluster-ready signal before re-accepting traffic.

If a step fails (the bucket cannot be read, a file times out), the restore answers with the error and the cluster is **returned to service** with the state the leader holds at that point — the stores imported before the failure plus the untouched rest, pushed to every node so they agree. It no longer stays "not ready" until a restart. Check the restore history entry and run the restore again once the storage problem is fixed.

```bash
curl -X POST http://localhost:8080/admin/backup/restore \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"backupId": "20260529-120000"}'
```

The Admin UI exposes this as a *Restore* button on each entry in the *Available Backups* list, behind an explicit confirmation modal that names every store about to be overwritten.

!!! note "Restore blocks requests on its own — maintenance mode is for the work around it"
    Phase 1 already marks the cluster not-ready for the duration, so you do **not** need to open a
    [maintenance window](cluster-maintenance.md) just to run a restore.

    Where a window earns its keep is the time on either side: deciding *which* backup to restore,
    checking what the current state looks like, and confirming the result afterwards. Those are the
    minutes when a job or a client can still commit a write that the restore is about to discard —
    or, worse, one that lands after it. Maintenance mode closes the write path across that whole
    span while leaving reads and settings available, so you can keep inspecting.

    The restore itself will overwrite `cluster.maintenance.mode` along with every other config key,
    so the window ends up however the backup had it. Each Master re-reads the value after importing
    the snapshot, so they agree on whatever that turns out to be — check the banner afterwards
    rather than assuming the window you opened is still open.

## Endpoint reference

| Method | Path | Body | Purpose |
|---|---|---|---|
| `GET` | `/admin/backup/config` | — | Current config + scheduler state (`scheduled`, `intervalNextFireMillis`, `cron`, `cronScheduled`, `cronNextFireMillis`, `cronTimeZone`, `retentionCount`, `retentionDays`, last backup) |
| `POST` | `/admin/backup/configure` | `{connectionId, s3Prefix, intervalHours, cron?, retentionCount?, retentionDays?}` | Validate, check the bucket (read + probe write/delete) and save connection / prefix / fixed-interval / cron / retention in one call; `400` with the reason otherwise |
| `POST` | `/admin/backup/cron` | `{cron: string \| null}` | Set or clear the cron schedule |
| `POST` | `/admin/backup/run` | — | Trigger a backup immediately |
| `POST` | `/admin/backup/restore` | `{backupId}` | Restore from a specific backup id |
| `GET` | `/admin/backup/list` | — (`?limit=&offset=` for one page) | Backup ids in S3, newest first: an array, or `{backups, total, offset, limit}` with paging |
| `GET` | `/admin/backup/history` | — | Last 100 history entries (backup, restore, prune), in memory on the leader |

The Admin UI's *Backup & Restore* page wires the same endpoints; the underlying state survives master restarts.
