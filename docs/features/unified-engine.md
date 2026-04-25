# Unified Data Engine

Ontul is a distributed unified data engine that combines batch processing, stream processing, and interactive SQL queries into a single system. One Ontul cluster replaces what traditionally requires separate clusters for each workload type.

## Why Unified?

Running separate systems for different workloads creates operational overhead — separate clusters to deploy, monitor, and maintain, separate configurations, separate security policies, and data movement between systems. Ontul eliminates this by providing all three capabilities in a single engine with shared infrastructure.

## What Ontul Unifies

### Processing Engine (Batch & Streaming)

Submit distributed batch ETL jobs or run continuous streaming pipelines using the Ontul SDK (Java, Python). Data transformations, aggregations, joins, and writes to sinks (Iceberg, Kafka, S3) are all executed across the cluster.

### Query Engine (Interactive SQL)

Run interactive SQL queries with federation across multiple data sources. Connect via JDBC (DBeaver, DataGrip) or Arrow Flight SQL and query Iceberg tables, JDBC databases, files on S3, and more through standard SQL.

### Shared Foundation

Both modes share:

- The same **Master/Worker** cluster infrastructure
- The same **Arrow-native operator pipeline** (Scan, Filter, Project, Join, Aggregate, Sort, Window, Exchange)
- The same **Exchange Manager** — unified fault-tolerance for batch spill and streaming checkpoint (KMS encrypted)
- The same **connector architecture** for accessing external data sources
- The same **IAM policies** for security and access control
- The same **Admin UI** for monitoring and management

## Exchange Manager

The Exchange Manager is Ontul's unified fault-tolerance infrastructure, providing two capabilities through a single backend:

| Capability | Used By | Purpose |
|-----------|---------|---------|
| **Data Spill** | Query / Batch / Streaming | When operators (Sort, HashJoin, HashAggregate) exceed memory, intermediate Arrow batches are spilled to local disk. Interactive SQL queries, batch ETL, streaming — 모두 동일한 경로 |
| **State Snapshot** | Streaming | Kafka offsets + window state are checkpointed for failure recovery via barrier checkpoint |

All spill and snapshot data is KMS envelope-encrypted (AES-256-GCM). Storage is two-tier: local disk (always, low-latency) plus optional S3 backup (async, for cross-worker recovery).

## Execution Modes

### Query Mode

Clients connect via Arrow Flight SQL and submit SQL queries interactively. The Master plans the query and distributes execution to Workers. After planning, Workers communicate directly with each other for shuffles — the Master is not a bottleneck during execution.

### Job Mode

Clients submit batch or streaming jobs programmatically via the SDK:

- **Client mode**: Interactive execution from applications or notebooks. No dependency upload needed.
- **Server mode**: Upload JARs with custom dependencies. Jobs run asynchronously — the client can disconnect after submission. Suitable for scheduled batch ETL and long-running streaming pipelines.

SQL and code can be mixed freely in both modes. Execution is lazy — the full plan is optimized at `.execute()` time.
