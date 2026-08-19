# High Availability

Ontul provides fault tolerance and high availability through multi-Master leader election, Worker health monitoring, and automatic failure recovery.

## Master High Availability

Multiple Masters can run simultaneously in a leader/follower configuration:

- **Leader Election**: Apache ZooKeeper (via Curator) elects a primary Master using a leader latch. The leader owns all write operations to the state store (RocksDB).
- **State Replication**: The leader Master replicates catalog metadata, IAM policies, KMS keys, sessions, and connection credentials to follower Masters via the internal NIO protocol.
- **Automatic Failover**: If the leader Master fails, ZooKeeper elects a new leader, which reloads persisted state from RocksDB and resumes operations.

Any Master accepts both reads and writes — clients never need to know which Master is the leader:

- **Reads / queries** are served by whichever Master receives them.
- **Control-plane writes** (catalog & connector registration, IAM users/groups/policies/keys, KMS, connection credentials, driver upload, semantic-view edits) received on a **follower** are **transparently forwarded to the leader** over the internal protocol, applied once on the leader's RocksDB, and broadcast back to every follower as a snapshot. A leader-unaware proxy is therefore correct for writes as well as reads — the forwarding happens inside the engine, not in the load balancer.
- **Table data written by SQL DML** (`INSERT`, `CREATE TABLE`, `MERGE`, …) commits to the **external catalog/storage the connector points at** — Iceberg-on-Polaris + object storage, JDBC, etc. — which is the shared source of truth. Ontul is a processing engine, not a store, so no divergent per-Master copy of table data exists and nothing is lost if a follower dies mid-session.

### Leader-only background services

Request handling is leader-unaware, but a few **background** services are cluster singletons and run
on the leader alone. Followers start them and hold them idle, taking over on election.

| Service | Why it must not run twice |
| --- | --- |
| Iceberg table maintenance | Two masters compacting one table collide at commit; concurrent `remove_orphan_files` is unsafe. |
| Iceberg health collection | Table health is a property of the table, not of the master that read it — collecting on every master would multiply object-store metadata reads for identical results. |
| Streaming job reconciler | Re-dispatching a streaming job from two masters would start it twice. |
| KMS key lifecycle | Key creation and rotation are writes to the leader-owned state store. |

Two consequences worth planning for. First, per-table Iceberg gauges on `/metrics` are exported by
the collector owner only — `ontul_iceberg_collector_leader` marks it, and dashboards should aggregate
with `max by (table) (…)` so a failover does not double-count. Second, state these services hold
purely in memory (the health time series and the maintenance before/after log) restarts with the new
leader; anything persisted — maintenance schedules, job history retention, KMS material — is
unaffected, since it lives in the replicated state store.


## Load balancing (nginx)

A stock nginx in front of the Masters is all the HA "router" Ontul needs — there is no bespoke query gateway, no cluster-health routing logic, no per-query queue management, and no leader awareness in the proxy. The single rule: **the Flight SQL upstream must be sticky.**

### Why Flight SQL needs session affinity

A single Flight SQL query is **two RPCs**: `GetFlightInfo` returns a *ticket*, then `DoGet(ticket)` streams the rows. The ticket is a handle (`TicketStatementQuery`) into a **result cache that lives in the Master that planned the query** — result batches are *not* replicated across Masters (only session/auth metadata is synced). If a plain round-robin sends the `DoGet` to a different Master, that Master has no such handle and the client fails with:

```
No results for handle: <uuid>
```

Prepared statements (`CreatePreparedStatement` → execute → `ClosePreparedStatement`) use the same Master-local handle cache, so they need the same affinity.

**Consistent hashing on the client address** pins each client to one Master for the life of its connection — which is all that is required. (Contrast with a Trino-style setup, where HA depends on a *smart* gateway that tracks cluster health, routes by cluster group, and stamps resource-group tags. Ontul needs none of that — just one `hash` directive.)

```nginx
# ── Admin UI + REST API (HTTP/1.1) ──
upstream ontul_admin {
    hash $remote_addr consistent;    # keep multi-step stateful admin flows on one Master
    server master-1:8080  max_fails=2 fail_timeout=5s;
    server master-2:8080  max_fails=2 fail_timeout=5s;
    keepalive 32;
}

# ── Arrow Flight SQL (gRPC over HTTP/2) ──
upstream ontul_flight_sql {
    hash $remote_addr consistent;    # REQUIRED — the Flight ticket handle is Master-local
    server master-1:47470 max_fails=2 fail_timeout=5s;
    server master-2:47470 max_fails=2 fail_timeout=5s;
    keepalive 16;
}

server {
    listen 8080;
    location / {
        proxy_pass http://ontul_admin;
        proxy_next_upstream error timeout http_502 http_503;   # retry a healthy Master
    }
}

server {
    listen 47470 http2;
    location / {
        grpc_pass grpc://ontul_flight_sql;
        grpc_next_upstream error timeout;                      # retry a healthy Master
    }
}
```

### Choosing the hash key

`hash $remote_addr consistent` keys the affinity on the **client IP**, which is the right default for direct clients. Be aware of two cases:

- **Clients behind a shared egress (NAT, a corporate proxy, a Kubernetes egress gateway)** all present the *same* source IP, so `$remote_addr` hashing pins them **all to one Master** and defeats load spreading. When your clients funnel through a single IP, hash on a per-client attribute instead — e.g. an auth/session identifier the client always sends. Over gRPC/Flight that means a header:

    ```nginx
    # requires nginx built with the ngx_http_split_clients / map hashing you need;
    # here we hash on an Ontul auth token header the client sets on every RPC.
    upstream ontul_flight_sql {
        hash $http_x_ontul_session consistent;   # per-session affinity, NAT-safe
        server master-1:47470 max_fails=2 fail_timeout=5s;
        server master-2:47470 max_fails=2 fail_timeout=5s;
        keepalive 16;
    }
    ```

    Any stable, per-client value works (`$http_authorization`, a cookie via `$cookie_…` for the HTTP admin plane, or `$ssl_client_s_dn` with mTLS). The only requirement is that **the same client always hashes to the same key** for the life of its Flight connection.

- **X-Forwarded-For chains**: if nginx sits behind another proxy/LB, `$remote_addr` is that proxy's IP (again collapsing everyone onto one Master). Configure `set_real_ip_from` + `real_ip_header X-Forwarded-For` so `$remote_addr` reflects the true client, or hash on the forwarded header directly.

If a single upstream Master ends up hot because of one of the above, the fix is the hash *key*, not turning off affinity — round-robin is never safe for the Flight `GetFlightInfo → DoGet` handle.

### Failover behaviour & health checks

Open-source nginx performs **passive** health checks only: a dead Master is marked down after `max_fails` failed requests within `fail_timeout`, so the first request or two after a Master crashes may fail before nginx routes around it. Mitigate this by:

- Tightening `max_fails` / `fail_timeout` (shown above) so eviction is fast.
- Setting `proxy_next_upstream` / `grpc_next_upstream` so nginx **retries the surviving Master** on a connect or timeout error instead of surfacing it to the client.
- (Optional) NGINX Plus if you need **active** health probes (`health_check`) rather than passive detection.

Because a single nginx is itself a single point of failure, run it as a **redundant pair with keepalived + a virtual IP** (or use a cloud L4 load balancer) so the cluster's entry point is highly available too.

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
