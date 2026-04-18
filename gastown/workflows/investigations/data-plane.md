---
title: "Investigating: data-plane failures"
type: investigation
status: verified
topic: gastown
created: 2026-04-17
updated: 2026-04-18
sources:
  - gastown/packages/doltserver.md
  - gastown/packages/beads.md
  - gastown/packages/daemon.md
  - gastown/packages/mail.md
  - /home/kimberly/repos/gastown/internal/doltserver/doltserver.go
  - /home/kimberly/repos/gastown/internal/beads/beads.go
  - /home/kimberly/repos/gastown/internal/beads/stale_pid.go
  - /home/kimberly/repos/gastown/internal/daemon/dolt.go
  - /home/kimberly/repos/gastown/internal/daemon/daemon.go
  - /home/kimberly/repos/gastown/internal/mail/bd.go
  - /home/kimberly/repos/gastown/internal/mail/router.go
tags: [investigation, data-plane, diagnostic, dolt, beads]
---

# Investigating: data-plane failures

**Symptom:** `bd` commands hang or timeout, "connection refused" errors,
"database not found" responses, query latency > 5s, unexpected empty
results from beads queries, or mail send/receive failures that trace
back to beads storage.

The data plane has three layers: the Dolt SQL server (TCP listener on
port 3307), the beads library (subprocess + SDK hybrid that talks to
Dolt), and consumers like mail that build on beads. Failures cascade
upward: a dead Dolt server breaks beads, which breaks mail, which
breaks nudge notifications for mail arrivals.

## Decision tree

### 1. Is the Dolt server reachable?

**Check:** `gt dolt status` or manually:

```bash
nc -z localhost 3307 && echo "reachable" || echo "unreachable"
```

Programmatically, `doltserver.CheckServerReachable(townRoot)` at
`doltserver.go:587-600` opens a TCP connection with a 2-second timeout.

- **Unreachable** -> go to [Step 2: Is the Dolt process alive?](#2-is-the-dolt-process-alive)
- **Reachable** -> go to [Step 5: Is the right server responding?](#5-is-the-right-server-responding)

### 2. Is the Dolt process alive?

**Check:** `doltserver.IsRunning(townRoot)` at `doltserver.go:526-586`
reads the PID file at `<townRoot>/daemon/dolt.pid` and probes with
`signal(0)`.

```bash
cat ~/gt/daemon/dolt.pid 2>/dev/null && kill -0 $(cat ~/gt/daemon/dolt.pid) 2>/dev/null && echo "alive" || echo "dead"
```

- **PID file missing or process dead** -> the Dolt server crashed.
  Go to [Step 3: Why did Dolt crash?](#3-why-did-dolt-crash)
- **Process alive but port not listening** -> the server is still
  starting up. `doltserver.WaitForReady(townRoot, timeout)` at
  `doltserver.go:610-658` polls with a retry loop scaled by database
  count. Check `gt dolt logs` for startup progress. If startup takes
  > 2 minutes, check for [phantom database directories](#4-are-there-phantom-or-orphan-databases)
  that slow initialization.

### 3. Why did Dolt crash?

**Check:** `gt dolt logs` — the Dolt server log is at
`<townRoot>/daemon/dolt-server.log`. See
[doltserver](../../packages/doltserver.md) ## Failure modes ->
Partial completion for the stale PID file behavior.

Common crash causes:

- **Port conflict:** another Dolt server (or MySQL) holds port 3307.
  `doltserver.CheckPortConflict(townRoot)` at `doltserver.go:711-766`
  identifies the holder. See
  [doltserver](../../packages/doltserver.md) ## What it actually does
  -> Imposter detection.
- **Corrupt database directory:** a phantom rig directory whose
  `.dolt/noms/manifest` is missing causes `dolt sql-server` to crash
  on startup. `Start` quarantines these at `doltserver.go:1489-1539`.
- **CLOSE_WAIT accumulation:** abandoned connections from crashed
  agents exhaust connection capacity. The read/write timeouts at
  `doltserver.go:137-156` (5 minutes each) are the mitigation.

**Fix:** `gt dolt start` — it handles imposter killing, quarantining
phantom directories, and stale lock cleanup automatically. If the
daemon is running, it will also restart Dolt within 30 seconds via the
`doltHealth` ticker (see [daemon](../../packages/daemon.md) ## The
maintenance tickers).

### 4. Are there phantom or orphan databases?

**Check:** `gt dolt status` shows orphan database count.
`gt dolt cleanup` removes orphan databases (test remnants like
`testdb_*`, `beads_t*`, `beads_pt*`, `doctest_*`).

See [doltserver](../../packages/doltserver.md) ## What it actually
does -> Workspace discovery and repair for `FindOrphanedDatabases`
at `doltserver.go:2499-2545`.

Phantom directories (missing `.dolt/noms/manifest`) are quarantined
by `Start` at `doltserver.go:1489-1539`. If quarantine fails
(directory too large), the start aborts with an error.

### 5. Is the right server responding?

Even when port 3307 is reachable, the server might be an **imposter**
— a rogue Dolt instance spawned by a `bd` subprocess serving a
different data directory.

**Check:** `doltserver.VerifyServerDataDir(townRoot)` at
`doltserver.go:996-1047` compares the running server's data directory
against the expected `<townRoot>/.dolt-data/`.

```bash
gt dolt status  # Shows "Server data directory: ..." — compare to expected
```

- **Wrong data directory** -> imposter. `gt dolt start` calls
  `KillImposters(townRoot)` at `doltserver.go:1048-1094`. The imposter
  was likely spawned by a `bd` subprocess that couldn't reach the real
  server and started its own. See
  [doltserver](../../packages/doltserver.md) ## What it actually does
  -> Imposter detection.
- **Correct data directory** -> go to [Step 6](#6-is-the-database-healthy).

### 6. Is the database healthy?

**Check:** `doltserver.GetHealthMetrics(townRoot)` at
`doltserver.go:3388-3496` returns connection count, latency, and
capacity.

```bash
gt dolt sql "SELECT 1"  # Basic query test
gt dolt status           # Health metrics summary
```

Sub-checks:

- **Read-only working set:** `doltserver.CheckReadOnly` at
  `doltserver.go:3497-3540`. A Dolt branch can get stuck in a
  read-only state after a crash. `RecoverReadOnly` at
  `doltserver.go:3563-3632` replays into a recovery branch.
  See [doltserver](../../packages/doltserver.md) ## What it actually
  does -> Databases and health.
- **High connection count:** `GetActiveConnectionCount` at
  `doltserver.go:3312-3364`. If near `DefaultMaxConnections = 1000`,
  agents may get "too many connections" errors. The daemon's
  `drainConnectionsBeforeStop` at `doltserver.go:1746-1781` reduces
  the connection storm during restart.
- **High query latency:** `MeasureQueryLatency` at
  `doltserver.go:3633-3665`. Latency > 5s indicates either a hot
  table (compaction needed) or disk I/O pressure. The daemon's
  `compactorDog` ticker runs every 24h by default.

### 7. Is beads resolving the correct database?

When Dolt is healthy but beads queries return wrong or empty results,
the problem is often in the routing layer.

**Check:** Verify the `.beads/` directory is configured correctly.

```bash
cat <workDir>/.beads/metadata.json  # Shows dolt_database, dolt_server_port
cat <workDir>/.beads/redirect       # If present, shows where beads is redirected
```

See [beads](../../packages/beads.md) ## What it actually does ->
`beads_redirect.go` for the redirect chain (up to depth 3, cycle
detection). `routes.jsonl` handles cross-rig prefix routing at
`routes.go:360`.

- **Stale PID file in `.beads/dolt/`:** `beads.CleanStaleDoltServerPID`
  at `stale_pid.go:20-46` removes PID files that point to dead
  processes. A stale PID causes `bd` to connect to port 3307 with the
  wrong `config.yaml`, potentially talking to the wrong server. The
  mail package's `runBdCommand` calls this at `bd.go:62-64` before
  every subprocess invocation.
  See [beads](../../packages/beads.md) ## Failure modes ->
  Precondition violations.
- **Wrong `BEADS_DIR` environment:** `beads.go:426-430` explicitly
  sets `BEADS_DIR` on every subprocess to prevent inherited values
  from causing prefix mismatches (fixed GH#803).

### 8. Is the `bd` subprocess timing out?

**Check:** Look for "signal: killed" in stderr output from `bd`
commands.

The mail package sets `bdReadTimeout = 60s` and `bdWriteTimeout = 60s`
at `bd.go:15-22`. The comment at `bd.go:18-20` records a real
incident: 30s caused `signal: killed` under concurrent agent load.
See [mail](../../packages/mail.md) ## Failure modes -> Precondition
violations.

- **Multiple agents hitting beads simultaneously:** The beads
  subprocess path (`beads.go:411-525`) creates one `bd` process per
  call. Under concurrent load, multiple `bd` processes compete for
  Dolt locks. The SDK path (`store.go`) avoids this by using an
  in-process connection, but only when a `beadsdk.Storage` is attached.
  See [beads](../../packages/beads.md) ## What it actually does ->
  Subprocess vs native SDK.

### 9. Is the daemon managing Dolt correctly?

If Dolt keeps crashing and restarting, check the daemon's Dolt
supervision.

**Check:** The daemon's `DoltServerManager` runs a health check every
30 seconds (`DefaultDoltHealthCheckInterval` at `dolt.go:27`). The
`RestartTracker` at `restart_tracker.go:14-45` applies exponential
backoff (30s initial, 10m max, 5 crashes before crash-loop state).

```bash
gt daemon status          # Shows daemon state including Dolt health
cat ~/gt/daemon/restart_state.json  # Per-agent backoff counters
```

- **Crash loop detected:** The daemon will stop restarting Dolt after
  5 crashes within 10 minutes. `gt daemon clear-backoff dolt` resets
  the counter. See [daemon](../../packages/daemon.md) ## Supervision
  -> Agent supervision.
- **Daemon not running:** Dolt gets no automatic supervision.
  `gt up` starts both the daemon and Dolt. See
  [daemon](../../packages/daemon.md) ## Lifecycle.

## Connection management comparison: beads, doltserver, mail

Cross-entity comparison of how the three data-plane packages acquire
connections, handle errors, and manage timeouts. Each cell verified
against source code.

### Connection acquisition

| Behavior | beads (internal/beads) | doltserver (internal/doltserver) | mail (internal/mail) |
|---|---|---|---|
| **Connection mode** | Hybrid: subprocess (`bd` via `exec.Command`) or in-process SDK (`beadsdk.Storage`) | Direct MySQL driver (`github.com/go-sql-driver/mysql`) for SQL helpers; subprocess (`dolt sql-server`) for lifecycle | Delegates to beads: subprocess (`bd` via `runBdCommand`) or in-process SDK |
| **Connection target** | Dolt SQL server on port 3307 (resolved from `.beads/metadata.json`) | Same Dolt SQL server (it IS the server) | Same Dolt SQL server (via beads layer) |
| **Per-call vs pooled** | Subprocess: one `bd` process per call (~600ms overhead). SDK: connection pooled inside `beadsdk.Storage` | Direct SQL: `database/sql` connection pool. Subprocess: one `dolt` process per call | Subprocess: one `bd` process per call. SDK: reuses `beadsdk.Storage` pool |
| **Stale PID cleanup** | Yes: `CleanStaleDoltServerPID` at `stale_pid.go:20-46` before resolve | n/a (owns the server process) | Yes: `beads.CleanStaleDoltServerPID` called at `bd.go:62-64` before every subprocess |
| **BEADS_DIR explicitly set** | Yes: prevents inherited env from causing prefix mismatches (`beads.go:426-430`, GH#803) | n/a | Yes: inherited from beads layer |

**Asymmetry: connection overhead.** The beads package spawns a new
`bd` subprocess per call (~600ms) unless a `beadsdk.Storage` is
attached. The doltserver package uses `database/sql` connection
pooling for its direct SQL helpers. Mail delegates to beads and
inherits whichever path is available. **By design:** the subprocess
path provides CLI compatibility and isolation; the SDK path is opt-in
for callers that can hold a long-lived store (daemon, long-running
services). The comment at `store.go:1-6` in the mail package records
the ~600ms motivation for the in-process path.

### Error handling

| Behavior | beads | doltserver | mail |
|---|---|---|---|
| **Timeout mechanism** | Context deadline from caller; no package-level default | 2s TCP timeout for `CheckServerReachable`; read/write timeouts of 5 min in `config.yaml` | `bdReadTimeout = 60s`, `bdWriteTimeout = 60s` (context deadlines at `bd.go:15-22`) |
| **Retry on transient error** | `--allow-stale` flag; `isSubprocessCrash` detection for SIGSEGV retries (`beads.go:555-565`) | `doltSQLWithRetry` retries on `isDoltRetryableError` patterns | `InjectFlatForListJSON` auto-retries without `--flat` if bd rejects the flag (`beads.go:451-468`) |
| **Error wrapping** | Maps stderr substrings to `ErrNotFound`, `ErrNotInstalled` (`beads.go:531-550`) | Returns raw MySQL/exec errors | Wraps in `bdError` struct with inner error + stderr capture (`bd.go:26-50`) |
| **Connection refused handling** | Propagates as `bd` subprocess exit code + stderr | `CheckServerReachable` returns error; `WaitForReady` retries | Propagates as `signal: killed` after 60s timeout |

**Asymmetry: timeout values.** Beads has no package-level timeout
default (relies on caller context). Doltserver uses 2s for reachability
checks but 5 minutes for read/write (to prevent CLOSE_WAIT
accumulation). Mail uses 60s for both read and write. **By design:**
the comment at `bd.go:18-20` records a real incident where 30s caused
`signal: killed` under concurrent load. The 60s is a deliberate
overcorrection. Doltserver's 5-minute timeouts are for the MySQL
`wait_timeout` equivalent, not for individual queries.

### Timeout and capacity thresholds

| Threshold | Value | Package | Source |
|---|---|---|---|
| TCP connection timeout | 2s | doltserver | `doltserver.go:587-600` |
| Read/write timeout (MySQL session) | 5 min each | doltserver | `doltserver.go:137-156` |
| Max connections | 1000 | doltserver | `doltserver.go:141` |
| `bd` subprocess read timeout | 60s | mail | `bd.go:15-22` |
| `bd` subprocess write timeout | 60s | mail | `bd.go:15-22` |
| In-process store context timeout | 30s | beads | `store.go` (`storeCtx()`) |
| Dolt health check interval (daemon) | 30s | daemon | `dolt.go:27` |
| DB list cache TTL | 30s | doltserver | `doltserver.go:1864-1871` |

**Asymmetry: in-process store timeout.** The beads SDK store uses a
30s context timeout (`storeCtx()`) while the subprocess path uses
whatever the caller provides (or no timeout). Mail's subprocess path
uses 60s. **Undocumented:** the 30s vs 60s discrepancy between the
SDK path and the subprocess path means the same logical operation can
time out at different thresholds depending on which code path executes.
The SDK path is faster (no subprocess overhead) so 30s is likely
sufficient, but no comment explains the choice.

## Related investigation workflows

- [Investigating: message delivery](message-delivery.md) -- mail
  delivery failures often trace back to Dolt reachability (Step 1
  above). That workflow links here for the shared diagnostic prefix.
