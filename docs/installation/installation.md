# Installation

## Community Edition Download

Ontul Community Edition is free for any use, including commercial production.

### Download

```bash
curl -L -O https://github.com/cloudcheflabs/ontul-pack/releases/download/ontul-archive/ontul-1.0.0-SNAPSHOT.tar.gz
tar xzf ontul-1.0.0-SNAPSHOT.tar.gz
cd ontul-1.0.0-SNAPSHOT
```

### Prerequisites

- **Java 17+**
- **ZooKeeper** (embedded ZooKeeper included for standalone mode)

### Quick Start

```bash
# 1. Set master key for KMS encryption (minimum 32 characters)
export ONTUL_MASTER_KEY="your-master-key-at-least-32-chars!"

# 2. Start ZooKeeper (embedded, for standalone mode)
bin/start-zk.sh

# 3. Start Master
bin/start-master.sh

# 4. Start Worker
bin/start-worker.sh
```

### Verify

Open the Admin UI at `http://localhost:8080/admin/`.

1. Login with default credentials: `admin` / `admin`
2. Change the default password on first login
3. Register catalogs through the Admin UI (Catalogs page)
4. Run SQL queries through the built-in SQL Runner

### Ports

| Service | Port | Description |
|---------|------|-------------|
| Admin UI / REST API | 8080 | Web console, REST API, query execution |
| Arrow Flight SQL | 47470 | JDBC connections (DBeaver, DataGrip, SDK) |
| Master Internal RPC | 19999 | Master-to-Worker communication |
| Worker Internal RPC | 19998 | Worker-to-Master communication |
| Worker Flight | 20998 | Worker data exchange (internal port + 1000) |
| ZooKeeper | 2181 | Cluster coordination (embedded) |

### Connect via JDBC

Use any JDBC client (DBeaver, DataGrip, or custom application):

```
URL:      jdbc:arrow-flight-sql://localhost:47470
Username: admin
Password: (your changed password)
```

### Configuration

The main configuration file is `conf/ontul.properties`. Key settings:

```properties
# Master
ontul.master.admin.port=8080
ontul.master.flight.sql.port=47470
ontul.master.internal.port=19999

# Worker
ontul.worker.internal.port=19998
ontul.worker.flight.port.offset=1000

# ZooKeeper
ontul.zk.serverList=localhost:2181

# KMS (encryption key from environment variable)
ontul.kms.master.key.env=ONTUL_MASTER_KEY

# Data directory
ontul.base.data.dir=data

# Exchange Manager (fault-tolerance)
ontul.exchange.base.dir=${ontul.base.data.dir}/exchange

# Streaming
ontul.streaming.checkpoint.interval.ms=10000
ontul.streaming.max.retries=3

# Logging
ontul.log.path=${user.dir}/logs
```

### Multi-Node Cluster

For production deployments with multiple Masters and Workers:

```bash
# On each node, set the master key
export ONTUL_MASTER_KEY="shared-master-key-across-all-nodes!"

# Master 1
bin/start-master.sh \
  -Dontul.zk.serverList=zk1:2181,zk2:2181,zk3:2181 \
  -Dontul.master.host=master-1 \
  -Dontul.master.admin.port=8080 \
  -Dontul.master.flight.sql.port=47470

# Master 2 (HA follower)
bin/start-master.sh \
  -Dontul.zk.serverList=zk1:2181,zk2:2181,zk3:2181 \
  -Dontul.master.host=master-2 \
  -Dontul.master.admin.port=8080 \
  -Dontul.master.flight.sql.port=47470

# Worker 1
bin/start-worker.sh \
  -Dontul.zk.serverList=zk1:2181,zk2:2181,zk3:2181 \
  -Dontul.worker.internal.port=29999

# Worker 2
bin/start-worker.sh \
  -Dontul.zk.serverList=zk1:2181,zk2:2181,zk3:2181 \
  -Dontul.worker.internal.port=29998
```

### Nginx Reverse Proxy

In production, place Nginx in front of the Ontul Masters to provide TLS termination, load balancing, and a single entry point for clients.

Ontul exposes two protocols that require separate Nginx configurations:

- **Admin UI + REST API** (HTTP/1.1) — Web console, catalog management, job submission
- **Arrow Flight SQL** (gRPC over HTTP/2) — JDBC connections from DBeaver, DataGrip, SDK applications

Nginx distributes requests across multiple Masters for high availability. If one Master goes down, Nginx automatically routes traffic to the remaining Masters.

```nginx
# ── Upstream: multiple Masters for HA ──
upstream ontul_admin {
    server master-1:8080;
    server master-2:8080;
}

upstream ontul_flight_sql {
    server master-1:47470;
    server master-2:47470;
}

# ── Admin UI + REST API (HTTP/1.1) ──
server {
    listen 443 ssl;
    server_name ontul.example.com;

    ssl_certificate     /etc/nginx/ssl/ontul.crt;
    ssl_certificate_key /etc/nginx/ssl/ontul.key;

    # No body size limit (large INSERT, bulk-insert, file uploads)
    client_max_body_size 0;

    location / {
        proxy_pass http://ontul_admin;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# ── Arrow Flight SQL (gRPC over HTTP/2) ──
server {
    listen 443 ssl http2;
    server_name flight.ontul.example.com;

    ssl_certificate     /etc/nginx/ssl/ontul.crt;
    ssl_certificate_key /etc/nginx/ssl/ontul.key;

    # No body size limit (large query results, streaming data)
    client_max_body_size 0;

    location / {
        grpc_pass grpc://ontul_flight_sql;
        grpc_set_header Host $host;
    }
}
```

#### JDBC Connection via Nginx

When connecting through Nginx with TLS, set `useEncryption=true`:

```
URL:      jdbc:arrow-flight-sql://flight.ontul.example.com:443
Username: admin
Password: (your password)
Properties: useEncryption=true
```

### Next Steps

- [Getting Started](../intro/intro.md) — Run your first query and example
- [Admin UI](../features/admin-ui.md) — Explore the web console
- [REST API Reference](../reference/rest-api.md) — API documentation
- [Ontul SDK](../reference/sdk-java.md) — Java and Python SDK guide
