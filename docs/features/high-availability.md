# High Availability

Ontul provides fault tolerance and high availability through multi-Master leader election, Worker health monitoring, and automatic failure recovery.

## Master High Availability

Multiple Masters can run simultaneously in a leader/follower configuration:

- **Leader Election**: Apache ZooKeeper (via Curator) elects a primary Master using a leader latch. The leader owns all write operations to the state store (RocksDB).
- **State Replication**: The leader Master replicates catalog metadata, IAM policies, KMS keys, sessions, and connection credentials to follower Masters via the internal NIO protocol.
- **Automatic Failover**: If the leader Master fails, ZooKeeper elects a new leader, which reloads persisted state from RocksDB and resumes operations.

All Masters can serve client queries — only write operations (catalog registration, IAM changes, etc.) are routed to the leader.

## Worker Health Monitoring

The Master continuously monitors Worker health:

- Scheduled health checks via the NIO protocol (configurable interval, default 10 seconds)
- Configurable timeout (default 5 seconds) and failure threshold (default 3 consecutive failures)
- Unhealthy Workers are automatically excluded from query planning
- Recovered Workers are automatically re-included

## Service Discovery

Masters and Workers register as ephemeral nodes in ZooKeeper. When a node joins or leaves the cluster, all other nodes are notified automatically. No manual configuration of cluster membership is required.

## State Store

All cluster state is stored in embedded RocksDB — no external database is needed:

- Catalog metadata and configurations
- IAM users, groups, policies, and access keys
- KMS encrypted keystore
- Sessions and connection credentials
- Job history and audit logs

## Backup & Restore

Ontul exports the cluster's stateful stores (KMS keystore, IAM, catalog metadata) to S3 and restores them with a coordinated cluster-wide handover. Three triggers share the same code path:

- **Manual** via the Admin UI or `POST /admin/backup/run`.
- **Fixed-interval** ("every N hours") via `intervalHours` on `/admin/backup/configure`.
- **Cron** via `/admin/backup/cron` for wall-clock schedules like `0 2 * * *` (daily at 02:00).

Both automatic modes coexist; the cron expression survives leader handoffs and restarts via the metadata store. Restore is a three-phase operation that blocks requests, imports the snapshot, and waits for every follower to sync before re-accepting traffic.

See **[Backup & Restore](backup-restore.md)** for the full endpoint reference and step-by-step setup.
