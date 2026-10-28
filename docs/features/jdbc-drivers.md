# JDBC Drivers

A **JDBC catalog** or job needs a database's JDBC driver on the classpath. Rather
than baking every driver into the distribution, Ontul lets you **upload driver
JARs at runtime** from the admin UI (**Drivers** page) or the admin API; each is
loaded dynamically on **every master and every worker** with no restart, then made
available on the job classpath.

## One JAR per driver class — versions do not coexist

The store is keyed by the **driver class** a JAR provides (discovered from the
JAR's own `java.sql.Driver` service), not by file name. The rule is deliberate:

- **A given driver class exists exactly once.** Two JARs cannot both provide, say,
  `org.postgresql.Driver` — that would make the active version ambiguous on the
  classpath the workers load.
- **Uploading a newer version replaces the old one.** Upload `postgresql-42.7.4.jar`
  when `postgresql-42.6.0.jar` is already installed (same `org.postgresql.Driver`,
  different file name) and the older JAR is **superseded** — deleted and replaced —
  so there is always a single, unambiguous driver per class. No manual cleanup, and
  no version coexistence to reason about.

Driver classes are read from the JAR's own `META-INF/services/java.sql.Driver`
file, so a driver already bundled on the system classpath is never mis-attributed
to an uploaded JAR.

## An uploaded driver wins over a bundled one

Each uploaded JAR is loaded **child-first**: its own classes are used before the
classes Ontul ships with (only `java.*`, `javax.*`, `jdk.*`, `sun.*` and
`org.slf4j.*` always come from the platform). When a catalog or job asks for a
driver class, uploaded JARs are searched before the bundled classpath. So
uploading `postgresql-42.7.4.jar` really makes PostgreSQL connections use 42.7.4,
even though a PostgreSQL driver ships with the distribution. Delete the uploaded
JAR to go back to the bundled version.

## Deleting a driver

`DELETE /admin/drivers/{file}` removes the JAR on the master that received it and tells
every other master and worker to remove its copy. A name that does not exist answers `404`. Upload and delete
need administrator access, and a file name must end in `.jar` and may not contain
`/`, `\` or `..`.

Deleting (or replacing) a JAR also takes its drivers out of the JVM's driver registry on every node, so the next
connection uses the bundled driver again. Earlier releases closed the JAR but left its driver registered: the next
connection on that node went to a driver whose classes could no longer load — for PostgreSQL
`NoClassDefFoundError: org/postgresql/jre7/sasl/ScramAuthenticator` on a SCRAM login — and every JDBC catalog on
that database failed there until a restart.

## Streamed to every master and worker

On upload the master validates and loads the JAR, then **streams it to each
worker and to the other masters in bounded chunks** over the internal NIO channel (rather than encoding a
large JAR into a single message). Big drivers are fine — some vendor JDBC drivers
(Snowflake, BigQuery, Databricks) bundle dependencies and reach tens of megabytes.
The admin HTTP body limit and the chunk size are configurable:

```properties
# ontul.properties
ontul.admin.http.max.content.size.bytes=134217728   # 128 MiB — accepts large driver JARs
ontul.driver.sync.chunk.size.bytes=4194304           # 4 MiB per streamed chunk
```

Every master needs the JAR, not only workers: a query on a JDBC catalog is planned on whichever
master the client reached, so a driver held by one master alone failed with "No suitable driver"
on the others and after a failover.

### Nodes that were down catch up on start

A master or worker that was down during an upload (or joined later) compares its drivers with the
leader's when it starts — by file name and SHA-256 — and fetches every JAR that is missing or
different. A follower master does the same whenever it re-reads the leader's state. JARs the leader
does not have are left in place: a leader that lost its drivers directory must not remove the
cluster's drivers, so delete a driver while its nodes are up or delete it again after they return.

The PostgreSQL driver ships with the distribution; upload others (MySQL, Oracle,
SQL Server, ClickHouse, …) as needed. A JDBC catalog then names the driver class
in its `driver` field, or references it through a registered
[Connection ID](connection-id.md).

## Admin API

| Method + path | Purpose |
| --- | --- |
| `GET /admin/drivers` | List installed JARs with the driver classes each provides. |
| `POST /admin/drivers` | Upload a JAR (`X-File-Name` header, raw body). Returns the detected classes and any replaced JAR. |
| `DELETE /admin/drivers/{file}` | Remove a JAR from every master and worker. |

## Related

- [Connection ID (Source/Sink by Reference)](connection-id.md) — register JDBC
  credentials once and reference them from catalogs and jobs.
- [Connector Architecture](connector-architecture.md) — how catalogs and
  connectors are registered.
