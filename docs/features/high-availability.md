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

Ontul supports cluster state backup and restore:

- Backup creates a RocksDB checkpoint as a compressed archive
- Backups can be stored locally or on S3
- Restore loads a backup archive and rebuilds the state store
