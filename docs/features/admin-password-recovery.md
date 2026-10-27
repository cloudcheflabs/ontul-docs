# Admin Password Recovery

Ontul ships with a built-in recovery channel that lets an operator reset the
`admin` user's password **without stopping the master**, even when no one
remembers the current password.

Recovery uses a local Unix domain socket — there is no HTTP back-door, no
network endpoint, no recovery URL. Authentication is performed by the
operating system: only a process that already shares the master's filesystem
identity can open the socket.

## When to use

- The admin password was forgotten or rotated out of the password manager.
- An automation script needs to provision a known admin password during
  first-time setup, without going through the web UI.
- A new operator is onboarded and you need to hand them an admin credential.

For day-to-day password changes (the user remembers the old password and
wants a new one), use the Admin UI's **Change Password** screen instead —
that path requires the old password and does not flag the user for forced
rotation.

## How it works

```
┌────────────────────┐   JSON over UDS    ┌────────────────────┐
│  ontul-cli         │ ─────────────────▶ │  Master (running)  │
│  iam:reset-password│                    │   AuthManager      │
└────────────────────┘                    │   .adminResetPwd() │
         ▲                                │                    │
         │ stdout: new password           │  saveToDb()        │
         │ (one-time)                     │  cluster sync push │
                                          │  audit log append  │
                                          └────────────────────┘
```

| Property | Value |
|---|---|
| **Socket path** | `${ontul.base.data.dir}/admin.sock` by default, mode `600` — the live value is published to `bin/master.socket` |
| **Authentication** | OS file permission — same user as the master process |
| **Socket path marker** | `bin/master.socket` — written when the socket binds, removed on shutdown |
| **Network surface** | none — Unix domain socket only |
| **Downtime** | none — applied in-process on the live master |
| **Cluster sync** | the leader applies the reset and pushes it to every node before answering; a follower forwards it to the leader (see [HA clusters](#ha-clusters)) |
| **Audit log** | `data/iam-audit/reset.log` (mode `600`, append-only) |
| **Post-reset state** | `requirePasswordChange = true` (forced rotation on next login) |

## HA clusters

IAM is owned by the leader master. Running `ontul-cli iam:reset-password` on a **follower** forwards the
reset to the leader. The leader applies it, pushes the new IAM state to every master and worker, and only
then answers; the follower also confirms that its own copy has the new password (pulling it from the
leader if needed) and the CLI notes that the reset was applied on the leader. Running it on the leader
applies it directly.

If no leader is elected, or the leader does not answer within `ontul.admin.socket.forward.timeout.ms`
(default `30000`), the CLI exits with status `1`, names the leader it tried and prints
**"the password was NOT changed"**. A follower never changes the password on its own, and the leader
discards a forwarded reset that reaches it too late, so a paused leader that comes back cannot apply a
reset the CLI already reported as failed. (Before, a follower changed its own copy and broadcast its
whole IAM snapshot, so a concurrent IAM change on the leader could silently undo the reset while the CLI
reported success.)

Every reset is written to the audit log with a fingerprint of the new password hash — the first 12 hex
characters of its SHA-256, never the password or the hash — and its route: `forwardedTo=<leader>` on the
follower and `forwardedFrom=<follower>` on the leader, with the same fingerprint, so one reset can be
followed across both logs. (The fingerprint used to be the hash's first 8 characters, which is the
algorithm tag `pbkdf2-s` for every password.)

```properties
# conf/ontul.properties
# How long a follower waits for the leader to apply a forwarded CLI reset before failing with
# "the password was NOT changed". Default 30000.
ontul.admin.socket.forward.timeout.ms=30000
```

## Which Java the CLI uses

`bin/ontul-cli.sh` (and every other launcher in `bin/`) runs `$JAVA_HOME/bin/java` when `JAVA_HOME` is set,
otherwise the `java` on `PATH`. A JDK installed outside `PATH` (for example `/opt/openlogic-openjdk-17...`) works
with just `JAVA_HOME`; the launchers used to run a bare `java` and failed with `exec: java: not found` (exit 127).
If `JAVA_HOME` points at a directory without an executable `bin/java`, the launcher exits 1 and names it; with no
Java at all it exits 1 with "Java not found. Install Java 17 or later, or set JAVA_HOME to its directory."

## After a reset: change the password once

A reset (generated or given) marks the account `requirePasswordChange`. Until the password is changed, every
other API call answers `403 {"error":"Password change required: this account has an initial or reset password
that must be changed first (POST /admin/auth/change-password)."}` and Flight SQL / JDBC refuses the login. The CLI
prints the generated password alone on stdout (so `P=$(bin/ontul-cli.sh iam:reset-password)` captures just it)
and a note with the two calls on stderr:

```bash
curl -s -X POST http://<master>:8080/admin/auth/login -H 'Content-Type: application/json' \
     -d '{"username":"admin","password":"<reset password>"}'            # -> accessToken
curl -s -X POST http://<master>:8080/admin/auth/change-password -H 'Authorization: Bearer <accessToken>' \
     -H 'Content-Type: application/json' -d '{"oldPassword":"<reset password>","newPassword":"<new>"}'
```

Signing in to the Admin UI with the reset password asks for the new one the same way.

## Quick start

The simplest invocation lets the master generate a strong 20-character
password and print it to stdout. The new password must be changed on the
admin's next login (the `requirePasswordChange` flag is set automatically).

```bash
# Inside the master host or container:
bin/ontul-cli.sh iam:reset-password
```

Sample output (TTY):

```
  ┌────────────────────────────────────────────────────────────┐
  │  Password reset for user: admin                            │
  │                                                            │
  │  New temporary password: K3p9WvTx7qLm8zXa#-Bd              │
  │                                                            │
  │  Must be changed on next login (requirePasswordChange).    │
  └────────────────────────────────────────────────────────────┘
```

## Input modes

| Mode | Command | When to use |
|---|---|---|
| **Master-generated** | `ontul-cli.sh iam:reset-password` | Default. Strong random password printed once on stdout. |
| **Explicit** | `ontul-cli.sh iam:reset-password --new-password 'My!Pass'` | Automation that knows the desired value. Beware: argv may show up in `ps`. |
| **Stdin** | `echo 'My!Pass' \| ontul-cli.sh iam:reset-password --new-password -` | Automation that wants to avoid argv exposure. |
| **Interactive** | `ontul-cli.sh iam:reset-password --interactive` | Operator at a TTY. Prompts for password twice with no echo. |

Resetting a different user is also supported:

```bash
bin/ontul-cli.sh iam:reset-password --user some-user --new-password 'NewPass123'
```

## Configuration

The recovery socket is enabled by default. Every key below lives in
`conf/ontul.properties` and is read at master startup:

```properties
# conf/ontul.properties
# Set false to remove the local recovery path entirely.
ontul.admin.socket.enabled      = true
ontul.admin.socket.path         = ${ontul.base.data.dir}/admin.sock
# Name of the file under <ontul.home>/bin that receives the socket path the
# master actually bound to (see "How the CLI finds the socket" below).
ontul.admin.socket.marker.file  = master.socket
# Append-only audit trail of socket operations.
ontul.iam.audit.dir = ${ontul.base.data.dir}/iam-audit
```

The socket follows `ontul.base.data.dir`. If you launch the master with
`-Dontul.base.data.dir=/var/lib/ontul` the socket moves to
`/var/lib/ontul/admin.sock` — you do not have to restate it.

### How the CLI finds the socket

Re-deriving the socket path from `conf/ontul.properties` is not reliable on its own:
`ontul.base.data.dir` can be overridden with `-D` at launch or edited after
startup, and the file does not record which value the live process used. So the
master **publishes the path it actually bound to** into
`<install dir>/bin/master.socket` when the socket comes up, and removes that file on
shutdown. `bin/ontul-cli.sh` prefers it.

Full resolution order, highest priority first:

1. `--socket /path/to/admin.sock` — read by the Java CLI, always wins.
2. `$ONTUL_ADMIN_SOCKET` — if already exported in the caller's shell.
3. `<install dir>/bin/master.socket` — the path published by the running master.
   Used only when the file exists *and* the path in it is a live socket.
4. `ontul.admin.socket.path` from `conf/ontul.properties`, with
   `${ontul.base.data.dir}` expanded. A value that still contains a
   `${...}` placeholder is rejected rather than used literally.
5. `<install dir>/data/admin.sock`, then `/data/admin.sock`.

Step 3 is what makes a moved data dir work: with the socket at
`/data/admin.sock` and the properties file still saying `./data`, only the marker
knows where to connect.

To rename the marker, change one key — both ends read it:

```bash
# conf/ontul.properties
ontul.admin.socket.marker.file = ontul-recovery.socket
```

Restart the master; it publishes `bin/ontul-recovery.socket`, and the CLI
picks the new name up from the same properties file.

### The master key is for the master, not the CLI

`ONTUL_MASTER_KEY` must be exported for the master process. The start script does not pre-check it, but KMS is enabled by default and the master cannot unseal its keystore without the key, so startup fails during KMS initialisation.
The variable name itself is configurable — `ontul.kms.master.key.env` in
`conf/ontul.properties` names the variable the master reads:

```bash
export ONTUL_MASTER_KEY='replace-with-a-32-char-or-longer-secret'
bin/start-master.sh
```

`bin/ontul-cli.sh` does **not** need it. The CLI only opens the Unix socket and
hands the request to the running master, which already holds the unsealed key,
so this works with the variable unset:

```bash
unset ONTUL_MASTER_KEY
bin/ontul-cli.sh ping
# pong
```

If a CLI invocation complains about the key rather than the socket, you are
running a start script, not the CLI.

### Worked examples

```bash
# 1. On the host, as the same OS user that runs the master:
cd /opt/ontul
bin/ontul-cli.sh ping
bin/ontul-cli.sh iam:reset-password

# 2. The master runs as a service account and you are root:
sudo -u ontul /opt/ontul/bin/ontul-cli.sh iam:reset-password

# 3. Inside a container:
docker exec -it ontul-master-1 /app/bin/ontul-cli.sh iam:reset-password

# 4. Data dir was relocated at launch — no extra flags needed, the CLI
#    reads the published marker:
cat /opt/ontul/bin/master.socket
# /var/lib/ontul/admin.sock
bin/ontul-cli.sh ping

# 5. Socket in a non-standard place and no marker (the master is stopped,
#    or you are on a host where the marker was cleaned up):
bin/ontul-cli.sh --socket /var/lib/ontul/admin.sock iam:reset-password

# 6. Non-interactive automation, password from stdin so it never reaches argv:
echo 'S0me!Strong!Pass' | bin/ontul-cli.sh iam:reset-password --new-password -
```

## Security model

**1. The socket is OS-gated.**
At startup the master creates `data/admin.sock` with mode `600` (owner
read/write only). Even other unprivileged users on the same host cannot
connect. There is no token, no shared secret, no network listener.

**2. The audit log records every reset.**
Every successful reset appends a JSON line to `data/iam-audit/reset.log`
(mode `600`). The plaintext password is **never** logged — only the first
8 characters of its hash, the user, whether it was master-generated, and
the OS user that invoked the CLI.

```json
{"ts":"2026-05-23T16:00:46.922Z","event":"iam.reset-password","user":"admin","generated":true,"hashFp":"OYV/Ojf/","invokedAs":"root"}
```

**3. The new password is exposed exactly once.**
For master-generated passwords, the plaintext is returned only on the
single CLI invocation that triggered the reset. It is not written to the
audit log, not stored in RocksDB, and not retransmitted. Treat scrollback
and shell history accordingly — or use stdin input mode to avoid argv
exposure entirely.

**4. Forced rotation on next login.**
After reset, the user is flagged `requirePasswordChange = true`. The next
successful login forces the user through the change-password flow, so a
temporary password used by the operator is immediately replaced by a
password only the user knows.

**5. The master must be running.**
Because the recovery channel is in-process, the master must be alive for
the CLI to connect. This is intentional: RocksDB requires an exclusive
lock, so an offline edit would either conflict with a running master or
need a complex stale-lock recovery. With this design, the only way to
reset is to be on the host *and* have the master running *and* share its
filesystem identity.

## Limitations

- **Master must be running.** If the master is down (e.g. KMS unseal
  failure during boot), this CLI cannot help. The recovery path in that
  case is to inspect the boot failure, fix the underlying issue, and let
  the master come up — then run the CLI.
- **No knowledge factor.** Any process that shares the master's filesystem
  identity can invoke the CLI. In multi-tenant or shared-shell
  environments, restrict shell access to the master accordingly. A future
  enhancement may add an opt-in "recovery key" requirement for an
  additional knowledge factor.

## Related

- [IAM](iam.md) — users, groups, policies
- [High Availability](high-availability.md) — cluster sync of IAM state
- [Encryption & KMS](encryption.md) — how IAM is encrypted at rest
