---
title: internal/doltserver
type: package
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-18
sources:
  - /home/kimberly/repos/gastown/internal/doltserver/doltserver.go
  - /home/kimberly/repos/gastown/internal/doltserver/dolthub.go
  - /home/kimberly/repos/gastown/internal/doltserver/sync.go
  - /home/kimberly/repos/gastown/internal/doltserver/rollback.go
  - /home/kimberly/repos/gastown/internal/doltserver/wisps_migrate.go
  - /home/kimberly/repos/gastown/internal/doltserver/wl_commons.go  # full read Phase 6
  - /home/kimberly/repos/gastown/internal/doltserver/wl_charsheet.go  # partial read Phase 6
  - /home/kimberly/repos/gastown/internal/doltserver/sysproc_unix.go
  - /home/kimberly/repos/gastown/internal/doltserver/sysproc_windows.go
tags: [package, data-layer, dolt, mysql-server, sql-server, lifecycle, imposter-killing, dolthub, migration, wl-commons]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [incomplete]
phase8_audited: 2026-04-17
phase8_findings: [partial-completion, silent-suppression]
detail_depth: {params: 2, data_flow: 2, errors: 1, side_effects: 1}
---

# internal/doltserver

Per-town Dolt MySQL server lifecycle manager. Every Gas Town town runs
one Dolt SQL server on port 3307 (by default) serving all the databases
in `<townRoot>/.dolt-data/`. This package owns every aspect of that
server's life: start/stop, imposter detection and killing, port
management, config.yaml authoring, health metrics, database discovery
and repair, DoltHub remote sync, migration backups, and — unexpectedly
— a "wanted list commons" (WL Commons) schema and character-sheet
projection layered on top of the same server.

**Go package path:** `github.com/steveyegge/gastown/internal/doltserver`
**File count:** 9 non-test go files; `doltserver.go` alone is ~139 KB
(~3990 lines) and is by far the largest file in the package. `sync.go`
(~21 KB), `wisps_migrate.go` (~17 KB), `wl_commons.go` (~21 KB), and
`wl_charsheet.go` (~15 KB) are the other substantial ones. 12 test
files including a large `doltserver_test.go` and dedicated suites for
rollback, sync, wisps migration, WL Commons conformance, and the fake
store.
**Imports (notable):** `github.com/go-sql-driver/mysql` (direct MySQL
protocol driver), `github.com/gofrs/flock` (start lock),
`gopkg.in/yaml.v3` (reading/writing `config.yaml`),
[`internal/beads`](beads.md) (beadsDir resolution for
per-[rig](../concepts/rig.md) databases),
[`internal/config`](config.md) (town config for rig enum),
`internal/style` (user-facing output), [`internal/util`](util.md)
(process-group detach). Standard library uses
`database/sql`, `encoding/json`, `net`, `net/http` (for DoltHub API),
`os/exec`, `strconv`, `strings`, `sync`, `syscall`, `time`.
**Imported by (notable):** [`gt dolt`](../commands/dolt.md) (start,
stop, status, logs, sql, init-rig, cleanup, repair subcommands —
nearly every function in this package is a direct `gt dolt` subcommand
backend), [`gt install`](../commands/install.md) (initial server
bring-up), [`gt init`](../commands/init.md) (per-rig database creation
via `InitRig`), the beads daemon (health-checking and recovery),
[`internal/beads`](beads.md) (connection string derivation and stale-
PID cleanup), and any CLI path that needs to resolve the connection
string to the town's Dolt server.

## What it actually does

The package doc comment at `doltserver.go:1-27` is the single best
user-facing summary:

> Package doltserver manages the Dolt SQL server for Gas Town.
>
> The Dolt server provides multi-client access to beads databases,
> avoiding the single-writer limitation of embedded Dolt mode.
>
> Server configuration:
>
> - Port: 3307 (avoids conflict with MySQL on 3306)
> - User: root (default Dolt user, no password for localhost)
> - Data directory: ~/gt/.dolt-data/ (contains all rig databases)

So: one long-running `dolt sql-server` subprocess per town, a flat
directory of rig subdirectories under it, and every `bd` subprocess
plus the in-process `beadsdk` store in the town connect to it via
MySQL protocol. The motivation is that **embedded Dolt mode is
single-writer**; running a server avoids lock contention when multiple
agents are hitting beads simultaneously.

### Config and state

- `DefaultPort = 3307`, `DefaultUser = "root"`,
  `DefaultMaxConnections = 1000`, `DefaultReadTimeoutMs` and
  `DefaultWriteTimeoutMs` both = 5 minutes (`doltserver.go:137-156`).
  The read/write timeouts are load-bearing: they prevent CLOSE_WAIT
  accumulation from abandoned connections during compactor GC, and
  they must be at least as long as the longest query expected to
  run. Comment at `doltserver.go:145-155` explains both in detail.
- `Config` struct (`doltserver.go:194-246`) — every knob the server
  supports: `TownRoot`, `Host`, `Port`, `User`, `Password`, `DataDir`,
  `LogFile`, `PidFile`, `MaxConnections`, `ReadTimeoutMs`,
  `WriteTimeoutMs`, `LogLevel`.
- `DefaultConfig(townRoot) *Config` (`doltserver.go:263-339`) — the
  environment-aware constructor. Priority order for port:
  `config.yaml > GT_DOLT_PORT env > daemon/daemon.env file > hard
  default 3307`. The daemon.env fallback (`readDaemonEnvVar` at
  `doltserver.go:343-359`) exists because `gt dolt status` and kin
  are usually invoked without the daemon-exported env, so without it
  those commands would compute a wrong default port. Host can come
  from `GT_DOLT_HOST`; user/password from `GT_DOLT_USER` /
  `GT_DOLT_PASSWORD`. A secondary fallback reads `mayor/daemon.json`
  to catch custom ports set by the daemon but not exported.
- `doltConfigYAML` + `readPortFromConfigYAML`
  (`doltserver.go:159-180`) — parses `.dolt-data/config.yaml` to learn
  the port; highest priority so that a running server with a custom
  port wins over whatever env vars happen to be set.
- `Config.IsRemote()` / `EffectiveHost()` / `HostPort()` / `SQLArgs()`
  / `userDSN()` / `displayDSN()` (`doltserver.go:363-411`, 1854-1859)
  — host utilities. Remote mode means `Host` is non-empty and non-
  localhost, in which case the package skips PID-file work, port-
  conflict checking, and local starts.
- `State` struct + `LoadState` / `SaveState` / `StateFile`
  (`doltserver.go:442-499`) — persists `Running`, `PID`, `Port`,
  `StartedAt`, `DataDir`, `Databases` to `<townRoot>/daemon/dolt.state.json`.
- `RigDatabaseDir(townRoot, rigName)` (`doltserver.go:436-440`) —
  `<townRoot>/.dolt-data/<rigName>`.

### Identity

- `EnsureDoltIdentity()` (`doltserver.go:60-104`) — ensures
  `dolt config --global` has `user.name` and `user.email` set.
  Populates them from `git config --global user.name/user.email` as a
  sensible default if unset. `InitRig` and `Start` both call this
  because `dolt init` refuses to run without identity.
- `doltConfigMissing(key)` (`doltserver.go:109-123`) — probes a dolt
  global config key and distinguishes "not set" (expected) from
  "dolt crashed" (propagate).
- `setDoltGlobalConfig(key, value)` (`doltserver.go:127-135`) —
  idempotent unset-then-add to avoid duplicate entries.

### Imposter detection and killing — the heart of the package

This is a huge concern because on every bd subprocess invocation,
bd **will spawn its own Dolt server** if it can't reach one — and
that rogue server may come up on 3307 with a **different data
directory**, serving the wrong set of databases. The package treats
these as "imposters" and kills them on sight.

- `IsRunning(townRoot) (bool, int, error)` (`doltserver.go:526-586`)
  — checks PID file, checks the process is alive, verifies ownership.
- `CheckServerReachable(townRoot) error` (`doltserver.go:587-609`) —
  opens a MySQL connection, pings, returns an error on failure.
- `WaitForReady(townRoot, timeout)` (`doltserver.go:610-658`) — poll
  loop around `CheckServerReachable`.
- `HasServerModeMetadata(townRoot) []string` (`doltserver.go:660-691`)
  — lists rigs whose beads dir is in server-mode.
- `CheckPortConflict(townRoot) (int, string)` /
  `findDoltServerOnPort(port) int` (`doltserver.go:711-766`) — finds
  any dolt process currently bound to the configured port.
- `DoltListener` struct + `FindAllDoltListeners()`
  (`doltserver.go:768-832`) — enumerates every dolt sql-server
  listener on the system, across ports. Used by `gt dolt status` to
  surface "why is port 3307 held by a different town".
- `isDoltServerOnPort(port) bool` (`doltserver.go:833-845`).
- `getServerDataDir`, `getDoltFlagFromArgs`, `getProcessArgs`,
  `getProcessCWD`, `resolveProcessPath`, `GetDoltDataDirFromProcess`,
  `getDoltConfigPathFromProcess` (`doltserver.go:846-937`) — forensic
  helpers that reach into `/proc/<pid>/` on Linux (and analogous
  routes on macOS) to determine which data directory a running dolt
  was started with.
- `doltProcessMatchesTownPaths` / `doltProcessMatchesTown`
  (`doltserver.go:938-968`) — ownership check: does this running dolt
  belong to this town, or is it a squatter?
- `doltProcessOwnerPathFromEvidence` / `doltProcessOwnerPath`
  (`doltserver.go:970-994`) — reverse lookup: which town owns a given
  dolt PID?
- `VerifyServerDataDir(townRoot) (bool, error)` (`doltserver.go:996-1047`)
  — the canonical imposter test. Returns `true` if the running server's
  data dir matches the town's expected `.dolt-data/`.
- **`KillImposters(townRoot) error`** (`doltserver.go:1048-1094`) —
  kills any dolt server on 3307 whose data dir isn't the town's.
  Invoked from `Start` (`doltserver.go:1441-1457`) and exposed via
  [`gt dolt`](../commands/dolt.md)'s CLI surface.
- `StopIdleMonitors(townRoot) int` (`doltserver.go:1121-1203`) — a
  prerequisite to imposter killing. Idle-monitor background processes
  auto-spawn rogue Dolt servers; if you kill the server without first
  stopping the monitors, they immediately respawn an imposter. The
  "gt-restart-race fix" comment at `doltserver.go:1385-1392` records
  the incident.

### Port management

- `CheckPortAvailable(port) error`, `PortHolder(port)`,
  `FindFreePort(startFrom)`, `checkPortAvailable(port)`,
  `waitForPortRelease(port, timeout)` (`doltserver.go:1204-1265`).

### Start and stop

- **`Start(townRoot) error`** (`doltserver.go:1320-1673`). The single
  most complex function in the package. In order:
  1. Acquire `<daemon>/dolt.lock` (flock) with exponential-ish
     retry scaled by database count (15s min, 120s max — gt-nkn
     "thundering herd" fix at `doltserver.go:1344-1363`).
  2. `StopIdleMonitors` to prevent respawn.
  3. `IsRunning` check → if running, orphan-detect (data dir missing
     → stop it) or verify legitimacy (imposter → `KillImposters` →
     fall through to start; legitimate → idempotent success with PID
     file fix-up).
  4. Evict any squatter holding the port (`doltserver.go:1406-1419`)
     via `findDoltServerOnPort` + `os.Process.Kill()`.
  5. Create data dir, **quarantine** any phantom rig dirs whose
     `.dolt/noms/manifest` is missing (moves to `.quarantine/` so the
     next `dolt sql-server` launch doesn't crash on the whole server;
     large directories are refused to prevent data destruction) —
     `doltserver.go:1489-1539`, tagged "gt-hs1i2" and "gt-xvh".
  6. `cleanupStaleDoltLock` on each database dir
     (`doltserver.go:1679-1707`) — checks `lsof` for holders before
     removing `.dolt/noms/LOCK`; safe because if bd has the lock open
     we leave it alone.
  7. Open the log file, `cleanStaleDoltSocket` (GH#2687), re-check
     port availability, **write `config.yaml` from `Config`** via
     `writeServerConfig` (`doltserver.go:1266-1319`) — CLI flags are
     ignored by dolt when `--config` is set, so all tuning has to
     land in this file.
  8. `exec.Command("dolt", "sql-server", "--config", configPath)`
     with `setProcessGroup` so `SIGHUP` from the parent tmux doesn't
     kill the server.
  9. Write PID file, save `State`, then **wait for the server to
     accept connections**. The wait loop scales with database count
     (`doltserver.go:1650-1672`): `maxAttempts = dbCount * 10`, each
     attempt sleeping 500ms. A process that dies during startup is
     detected via `processIsAlive` on each iteration; a process that
     never binds is reported with a "check gt dolt logs" hint.
- **`Stop(townRoot) error`** (`doltserver.go:1782-1838`). Drain
  connections first (see `drainConnectionsBeforeStop` at
  `doltserver.go:1746-1781`) to reduce the NomsBlockStore.Close race
  window, then `gracefulTerminate` (SIGTERM on Unix, Kill on Windows
  via the `sysproc_*.go` shims) with up to 5s of polite wait, then
  `Kill()` as a force quit. Cleans up PID file and state.

### Databases and health

- `ListDatabases(townRoot)` / `listDatabasesLocal` /
  `listDatabasesCached` / `listDatabasesRemote`
  (`doltserver.go:1891-2029`) — queries `SHOW DATABASES` and filters
  system databases. The cached variant uses the process-level
  `dbCache` struct (`doltserver.go:1864-1871`) with a 30s TTL and
  an in-flight channel to coalesce concurrent callers into one dolt
  sql subprocess (GH#2180 thundering herd fix).
- `InvalidateDBCache()` (`doltserver.go:1877-1890`) — callers of
  schema-changing SQL call this after mutation.
- `VerifyDatabases` / `VerifyDatabasesWithRetry` /
  `verifyDatabasesWithRetry` (`doltserver.go:2030-2137`) — compares
  served databases vs. the on-disk directory listing; reports
  `missing` for filesystem dirs the server didn't pick up.
- `IsSystemDatabase` (`doltserver.go:2138-2145`).
- `parseShowDatabases` / `findMissingDatabases` / `jsonKeys`
  (`doltserver.go:2146-2225`).
- `InitRig(townRoot, rigName)` (`doltserver.go:2226-2315`) — create a
  new rig database. Returns `(serverWasRunning, created, err)` so
  callers know whether they need to restart the server to pick up
  the new directory.
- `DatabaseExists` (`doltserver.go:2465-2473`).
- `RemoveDatabase(townRoot, dbName, force bool)`
  (`doltserver.go:2736-2795`) — guarded by `databaseHasUserTables`
  (`doltserver.go:2796-2826`) unless `force` is passed.
- Health: `GetActiveConnectionCount` (`doltserver.go:3312-3364`),
  `HasConnectionCapacity` (`doltserver.go:3365-3387`),
  `HealthMetrics` struct + `GetHealthMetrics`
  (`doltserver.go:3388-3496`).
- Read-only recovery: `CheckReadOnly`, `IsReadOnlyError`,
  `RecoverReadOnly`, `doltSQLWithRecovery`
  (`doltserver.go:3497-3632`) — if a dolt branch is stuck on a
  read-only working set, replay it into a recovery branch.
- `MeasureQueryLatency` (`doltserver.go:3633-3665`) — sampled health
  signal.
- `GetLastCommitAge` (`doltserver.go:3666-3721`).

### Workspace discovery and repair

- `Migration` struct + `findLocalDoltDB` / `FindMigratableDatabases` /
  `MigrateRigFromBeads` (`doltserver.go:2316-2464`) — discovery and
  migration of legacy per-rig embedded dolt DBs into the server.
- `BrokenWorkspace` struct, `FindBrokenWorkspaces`, `checkWorkspace`,
  `RepairWorkspace` (`doltserver.go:2474-2976`) — the repair side of
  `gt dolt repair`. A workspace is broken if its `.beads` dir points
  at a database the server doesn't know about, or if the schema is
  wrong, or if the rig prefix doesn't match.
- `OrphanedDatabase` struct + `FindOrphanedDatabases`
  (`doltserver.go:2499-2545`), `readExistingDoltDatabase`,
  `collectReferencedDatabases`, `CollectDatabaseOwners`
  (`doltserver.go:2546-2735`) — the bookkeeping that backs
  `gt dolt cleanup`. An orphaned DB is one sitting under
  `.dolt-data/` that no rig's `.beads` points to — these accumulate
  from tests that use `testdb_*` / `beads_t*` / `beads_pt*` /
  `doctest_*` patterns and never clean up after themselves.
- `EnsureMetadata` / `buildRigPrefixMap` / `EnsureAllMetadata` /
  `pickDBForRig` / `buildDatabaseToRigMap` / `FindRigBeadsDir` /
  `FindOrCreateRigBeadsDir` (`doltserver.go:2977-3311`) — rig/beadsDir
  lookup and repair utilities used by `gt dolt init-rig`, `gt
  dolt repair`, and callers from [`internal/beads`](beads.md).

### Server-side SQL helpers

- `serverExecSQL(townRoot, query string) error`
  (`doltserver.go:3791-3809`),
- `waitForCatalog(townRoot, dbName string) error`
  (`doltserver.go:3810-3845`) — busy-wait until a newly-created DB
  appears in the server's catalog.
- `doltSQL`, `doltSQLWithRetry`, `isDoltRetryableError`,
  `CommitServerWorkingSet`, `doltSQLScript`, `doltSQLScriptWithRetry`
  (`doltserver.go:3846-3991`) — thin wrappers that add retry on
  transient errors (`retryable` = known restart/lock patterns).

### Subprocess invocation helpers

- `buildDoltSQLCmd(ctx, config, args...) *exec.Cmd`
  (`doltserver.go:413-434`) — shared "run `dolt sql …` as a
  subprocess" helper so every SQL-over-subprocess path uses the same
  detached process group.
- `setProcessGroup(cmd)` / `processIsAlive(pid)` /
  `gracefulTerminate(p)` in `sysproc_unix.go` (`Setpgid: true`,
  `Signal(0)`, `SIGTERM`) and the Windows shim `sysproc_windows.go`.

### DoltHub integration — dolthub.go

Source: `/home/kimberly/repos/gastown/internal/doltserver/dolthub.go`.

The DoltHub remote API client. Enables pushing per-rig Dolt databases
to DoltHub repositories for off-site backup and collaboration.

- **Endpoints** (`dolthub.go:14-22`): `dolthubAPIBase` (var, test-
  overridable) for the REST API, `dolthubRemoteBase` (const) for
  push/pull via the Dolt remote protocol.
- **Credentials** — `DoltHubToken()` / `DoltHubOrg()`
  (`dolthub.go:24-34`) read `DOLTHUB_TOKEN` and `DOLTHUB_ORG` from
  the environment. No fallback; callers must set these for remote
  sync to work.
- **Repo name mapping** — `DoltHubRepoName(dbName)`
  (`dolthub.go:36-44`) converts local DB names to DoltHub repo
  names: underscores become hyphens; the one special case is `hq`
  to `gt-hq` (avoiding a bare two-letter repo name).
- **`CreateDoltHubRepo(org, repo, token)`** (`dolthub.go:52-96`) —
  POSTs to `<apiBase>/database` with private visibility and owner set
  to the org. Treats HTTP 200-299 and "already exists" as success;
  any other response is an error. Uses `net/http` directly (no SDK).
- **`AddRemote(dbDir, org, repo)`** (`dolthub.go:98-124`) — runs
  `dolt remote add origin <url>` as a subprocess; idempotent on
  "already exists" errors.
- **`SetupDoltHubRemote(dbDir, org, dbName, token)`**
  (`dolthub.go:130-149`) — the full setup flow: create repo on
  DoltHub, add remote to local DB, initial push. Fail-fast at each
  step.

### Sync — sync.go

Per-rig push/pull to DoltHub remotes, two flavors:

- **Subprocess path:** `CommitWorkingSet`, `PushDatabase`,
  `PullDatabase`, `PullDatabases`, `SyncDatabases`, `FindRemote`,
  `HasRemote` (`sync.go:52-324`, 406-500).
- **SQL path (in-server):** `PullDatabaseSQL`, `PullDatabasesSQL`,
  `PushDatabaseSQL`, `FindRemoteSQL`, `SyncDatabasesSQL`
  (`sync.go:153-258`, 325-405, 501-587). These issue `dolt_pull()` /
  `dolt_push()` stored functions over the server connection instead
  of shelling out, which is faster and respects the server's
  connection pool.
- `SyncOptions` struct with per-call flags (`sync.go:18-29`),
  `SyncResult` (`sync.go:30-51`).
- `PurgeClosedEphemerals(townRoot, dbName, dryRun)`
  (`sync.go:588-672`) — drops rows for closed ephemeral (wisp) issues
  during patrol squash, so that git sync doesn't carry ephemeral
  data.

### Rollback — rollback.go

`Backup` struct and `FindBackups(townRoot) ([]Backup, error)`
(`rollback.go:13-...`) — enumerates `migration-backup-YYYYMMDD-HHMMSS`
directories produced by earlier `MigrateRigFromBeads` runs, sorted
newest-first. Backs the `gt dolt rollback` subcommand.

### Wisps migration — wisps_migrate.go

`MigrateAgentBeadsToWisps` and a family of helpers
(`bdSQL`, `bdSQLCSV`, `bdExec`, `bdSQLCount`, `bdTableExists`,
`ensureWispsTable`, `ensureWispAuxTables`, `copyAgentBeadsToWisps`,
`copyAuxiliaryData`, `deriveDBName`, `ensureWispsOnGTServer`,
`gtTableExists`) — one-shot migration that copies agent bead records
into the wisp (ephemeral) table on the GT server. Related to the mail
wisp storage path in [`internal/mail`](mail.md)'s
`queryWispMessagesByAssignee`.

### WL Commons — wl_commons.go + wl_charsheet.go

Layered on top of the Dolt server, the "Wanted List Commons" is a
**separate schema** (not beads) that implements a
wanted-list / bounty / stamps / badges / leaderboard system. This is
a meaningful surprise for a package named "doltserver": it hosts not
just the server but also a second data model. The database name is
`wl_commons` (`WLCommonsDB` const, `wl_commons.go:21`). Phase 1
("wild-west mode"): direct writes to main branch via the local Dolt
server (`wl_commons.go:5`).

**Abstraction layer:**

- `WLCommonsStore` interface (`wl_commons.go:24-38`) — 12 methods:
  `EnsureDB`, `DatabaseExists`, `InsertWanted`, `ClaimWanted`,
  `SubmitCompletion`, `QueryWanted`, `QueryWantedFull`,
  `InsertStamp`, `QueryLastStampForSubject`, `QueryStampsForSubject`,
  `QueryBadges`, `QueryAllSubjects`, `UpsertLeaderboard`.
- `WLCommons` struct (`wl_commons.go:41`) — concrete implementation
  wrapping `townRoot`. Each method delegates to the package-level
  function (e.g., `InsertWanted(w.townRoot, item)`). The interface
  exists for testability — a fake store is used in test suites.

**Domain types:**

- `WantedItem` (`wl_commons.go:81-97`) — `ID`, `Title`,
  `Description`, `Project`, `Type`, `Priority`, `Tags` ([]string),
  `PostedBy`, `ClaimedBy`, `Status`, `EffortLevel`, `EvidenceURL`,
  `SandboxRequired`, `CreatedAt`, `UpdatedAt`.
- `StampRecord`, `BadgeRecord` (across `wl_commons.go` and
  `wl_charsheet.go`) — provenance tokens for work quality.
- `CharacterSheet`, `GeometryBar`, `SkillEntry`, `StampHighlight`,
  `Warning`, `Neighbor`, `BridgeConnection`, `CrossClusterStamp`,
  `LeaderboardEntry`, `TierThreshold` (`wl_charsheet.go:15-127`) —
  the full character-sheet projection model.

**Schema management:**

- `EnsureWLCommons(townRoot)` and `initWLCommonsSchema(townRoot)`
  (`wl_commons.go:127-271`) create the `wl_commons` database and
  its tables (wanted, stamps, badges, leaderboard) if they don't
  exist. All schema DDL is executed over the local Dolt server
  connection.

**CRUD operations:**

- `InsertWanted` — inserts a wanted item and commits.
- `ClaimWanted(townRoot, wantedID, rigHandle)` — sets `claimed_by`
  and `status=claimed`.
- `SubmitCompletion(townRoot, completionID, wantedID, rigHandle,
  evidence)` — records a completion submission.
- `QueryWanted` / `QueryWantedFull` — single-item lookup; Full
  variant includes related stamps and completions.
- `InsertStamp` / `QueryLastStampForSubject` /
  `QueryStampsForSubject` — stamp CRUD.
- `QueryBadges(handle)` — badges earned by a handle.
- `QueryAllSubjects` — distinct subject list for leaderboard.
- `UpsertLeaderboard(entry)` — insert-or-update leaderboard row.

**Character sheet projection** (`wl_charsheet.go:128-332`):

- `AssembleCharacterSheet` aggregates stamps, skills, warnings,
  neighbors, and tier into a single `CharacterSheet` projection.
- `computeStampGeometry` — lays out stamps visually as geometry bars.
- `computeSkillCoverage` — derives skill entries from stamp history.
- `computeTopStamps` — picks highlight stamps.
- `computeWarnings` — generates warnings for missing coverage.
- `computeTier` — calculates tier from total stamp count against
  `TierThreshold` thresholds.
- `RunScorekeeper(store WLCommonsStore) ([]*LeaderboardEntry, error)`
  (`wl_charsheet.go:474+`) — aggregates all subjects' stamps into
  leaderboard entries with computed tiers.

**SQL helpers:** `doltSQLQuery`, `parseSimpleCSV`, `parseCSVLine`,
`EscapeSQL`, `GenerateWantedID` (SHA-256 of random bytes,
`wl_commons.go:9-10`), `backtickKey`, `isNothingToCommit`
(`wl_commons.go:99+`).

### Internals / Notable implementation

- **Authentication is by convention, not enforcement.** User is
  `root`, password is empty, server listens on localhost only by
  default. Any process on the machine with the port can connect.
  Multi-town isolation is by port, not by credentials. Non-local
  deployments (`IsRemote() == true`) expect the operator to set
  `GT_DOLT_HOST`/`GT_DOLT_USER`/`GT_DOLT_PASSWORD` and handle real
  auth externally.
- **`--config` owns the server tuning.** `Start` always writes
  `config.yaml` from the `Config` struct and passes `--config <path>`
  to `dolt sql-server`. CLI flags would be ignored. This means
  changes to server tuning must flow through `writeServerConfig`.
- **Metadata mutex map.** `metadataMu sync.Map` + `getMetadataMu`
  (`doltserver.go:185-192`) provides per-path goroutine mutexes for
  `EnsureMetadata` because flock only synchronises across processes,
  not within a single process — the same process can acquire its own
  flock twice without blocking.
- **The PID file + port + ownership triple check.** `IsRunning` is
  intentionally paranoid: PID file alone is not enough (stale), port
  alone is not enough (imposter), data dir alone is not enough (old
  state). Only when all three agree does the package consider a
  server "ours". Any mismatch triggers `KillImposters` or recovery.
- **Thundering-herd protection.** Two independent mechanisms:
  (1) `dolt.lock` file lock on `Start` with retry scaled to the
  database count (see gt-nkn tagged comments); (2) `dbCache` with
  `inflight` channel so concurrent `ListDatabases` callers collapse
  into one subprocess per TTL window.
- **Stale-state cleanup is eager.** `cleanStaleDoltSocket`, stale
  `.dolt/noms/LOCK` removal, stale `dolt.pid`, stale PID-file
  rewriting after imposter kills, `.quarantine` for phantom DBs.
  Every one of these is an incident preserved in code.

### Usage pattern

Typical install / daily operation:

```go
// gt install
_ = doltserver.EnsureDoltIdentity()
_, _, _ = doltserver.InitRig(townRoot, "gastown")
_ = doltserver.Start(townRoot)

// gt dolt status
running, pid, _ := doltserver.IsRunning(townRoot)
metrics := doltserver.GetHealthMetrics(townRoot)

// gt dolt cleanup
orphans, _ := doltserver.FindOrphanedDatabases(townRoot)
for _, o := range orphans {
    _ = doltserver.RemoveDatabase(townRoot, o.Name, false)
}

// gt dolt stop
_ = doltserver.Stop(townRoot)
```

## Related wiki pages

- [gt](../binaries/gt.md)
- [gt dolt](../commands/dolt.md) — the subcommand surface. Most
  functions in this page have a `gt dolt <verb>` front door.
- [gt install](../commands/install.md) — bootstraps the server on
  first run.
- [gt init](../commands/init.md) — calls `InitRig` for new rigs.
- [internal/beads](beads.md) — the *client* side. Every `bd`
  subprocess and every in-process beadsdk store connects to the
  server managed by this package. `beads.ResolveBeadsDir`,
  `beads.CleanStaleDoltServerPID`, and `beads.DatabaseEnv` cooperate
  with server state.
- [internal/mail](mail.md) — downstream consumer via beads. Mail
  wisps live in the same Dolt server, accessed through
  `wisps_migrate.go`'s schema and the mail package's raw-SQL wisp
  queries.
- [internal/config](config.md) — town config enumerates rigs, which
  `InitRig` uses.
- [internal/util](util.md) — process-group detach (though many of
  the Cmd helpers in doltserver use the package-local
  `setProcessGroup` from `sysproc_unix.go`).
- [internal/lock](lock.md) — the package uses `github.com/gofrs/flock`
  directly for `dolt.lock` rather than the `internal/lock` helper.
- [go.mod](../files/go-mod.md) — `github.com/go-sql-driver/mysql`,
  `github.com/gofrs/flock`, `gopkg.in/yaml.v3`.
- [go-packages inventory](../inventory/go-packages.md)

## Failure modes

### Partial completion (what doesn't it clean up?)
- **Stale PID file after crash:** `Start` writes a PID file at
  `doltserver.go:~1450`. If the Dolt server crashes without
  `Stop` being called, the PID file persists. **Present** —
  `IsRunning` at `doltserver.go:540-558` detects stale PID files
  (process dead) and removes them with `_ = os.Remove(config.PidFile)`.
- **Lock file on Start failure:** If `Start` fails after acquiring
  the flock at `doltserver.go:1334-1383`, the lock is released in
  the defer. However, the lock file itself is never removed on
  success. **Present** — POSIX flocks auto-release on process exit;
  the file is a coordination artifact, not a leak.

### Silent suppression (what errors are swallowed?)
- **Connection close errors:** Multiple `_ = conn.Close()` calls
  at `doltserver.go:535,576,598,633` discard TCP connection close
  errors during health checks. **Present** — intentional for probe
  connections that are immediately discarded.
- **PID file removal on stale detection:** `doltserver.go:558` —
  `_ = os.Remove(config.PidFile)` discards the error when removing
  a stale PID file. **Absent** — if the PID file cannot be removed
  (permissions, read-only mount), the stale file persists and
  `IsRunning` will repeatedly try and fail to remove it on every call,
  without logging the failure.
- **Lock file removal on stale lock:** `doltserver.go:1337,1375` —
  `_ = os.Remove(lockFile)` for corrupt or stale lock files.
  **Absent** — same pattern as stale PID file: silent failure on
  permission errors.

### Cross-platform concerns
- **sysproc_unix.go / sysproc_windows.go:** Platform shims for
  `setProcessGroup` and `processIsAlive`. The Windows shim uses
  `CREATE_NEW_PROCESS_GROUP` similar to `util.SetProcessGroup`.
  **Untested** — no indication in the codebase that the Windows
  Dolt server lifecycle has been tested.

## Troubleshooting

For diagnostic workflows involving this entity, see
[Investigating: data-plane failures](../workflows/investigations/data-plane.md).

## Detail tables

### Server configuration defaults

| Key | Value | Source |
|---|---|---|
| `DefaultPort` | 3307 | `doltserver.go:137` |
| `DefaultUser` | `"root"` | `doltserver.go:139` |
| `DefaultMaxConnections` | 1000 | `doltserver.go:141` |
| `DefaultReadTimeoutMs` | 300000 (5 min) | `doltserver.go:143-146` |
| `DefaultWriteTimeoutMs` | 300000 (5 min) | `doltserver.go:149-156` |

### Port resolution priority

| Priority | Source | Code |
|---|---|---|
| 1 (highest) | `.dolt-data/config.yaml` | `readPortFromConfigYAML` at `doltserver.go:159-180` |
| 2 | `GT_DOLT_PORT` env var | `DefaultConfig` at `doltserver.go:263-339` |
| 3 | `daemon/daemon.env` file | `readDaemonEnvVar` at `doltserver.go:343-359` |
| 4 | `mayor/daemon.json` | `DefaultConfig` secondary fallback |
| 5 (lowest) | Hard default `3307` | `DefaultConfig` |

### Health and liveness functions

| Function | Purpose | Timeout/Threshold | Source |
|---|---|---|---|
| `CheckServerReachable` | TCP connection + MySQL ping | 2s TCP timeout | `doltserver.go:587-600` |
| `WaitForReady` | Poll loop around `CheckServerReachable` | `maxAttempts = dbCount * 10`, 500ms sleep | `doltserver.go:610-658` |
| `IsRunning` | PID file read + signal(0) probe | none | `doltserver.go:526-586` |
| `CheckPortConflict` | Find dolt process on configured port | none | `doltserver.go:711-766` |
| `VerifyServerDataDir` | Compare running server data dir vs expected | none | `doltserver.go:996-1047` |
| `GetHealthMetrics` | Connection count, latency, capacity | none | `doltserver.go:3388-3496` |
| `GetActiveConnectionCount` | Current connection count vs `DefaultMaxConnections` | none | `doltserver.go:3312-3364` |
| `MeasureQueryLatency` | Sampled `SELECT 1` latency | > 5s = degraded | `doltserver.go:3633-3665` |
| `CheckReadOnly` | Detect read-only working set after crash | none | `doltserver.go:3497-3540` |
| `RecoverReadOnly` | Replay into recovery branch | none | `doltserver.go:3563-3632` |

### Start sequence critical steps

| Step | Action | Source |
|---|---|---|
| 1 | Acquire `dolt.lock` flock (retry: 15s-120s scaled by DB count) | `doltserver.go:1329-1383` |
| 2 | `StopIdleMonitors` to prevent imposter respawn | `doltserver.go:1385-1392` |
| 3 | Check `IsRunning` — imposter detect + orphan evict | `doltserver.go:1394-1470` |
| 4 | Quarantine phantom rig dirs (missing `.dolt/noms/manifest`) | `doltserver.go:1489-1539` |
| 5 | `cleanupStaleDoltLock` per DB dir (checks `lsof` first) | `doltserver.go:1679-1707` |
| 6 | Write `config.yaml` via `writeServerConfig` | `doltserver.go:1266-1319` |
| 7 | `exec.Command("dolt", "sql-server", "--config", ...)` with `setProcessGroup` | `doltserver.go:1596+` |
| 8 | Write PID file + save State | `doltserver.go:1640+` |
| 9 | Wait for connections (maxAttempts = dbCount * 10, 500ms each) | `doltserver.go:1650-1672` |

### Stop sequence

| Step | Action | Source |
|---|---|---|
| 1 | `drainConnectionsBeforeStop` — reduce NomsBlockStore.Close race window | `doltserver.go:1746-1781` |
| 2 | `gracefulTerminate` (SIGTERM, 5s wait) | `doltserver.go:1782-1838` |
| 3 | `Kill()` force quit if still alive | `doltserver.go:1782-1838` |
| 4 | Clean up PID file + state | `doltserver.go:1782-1838` |

## Outgoing calls

### Subprocess invocations
| Called binary | Command | Flags | Flag source | `file:line` |
|---|---|---|---|---|
| `dolt` | `remote add` | `origin <url>` | runtime | `dolthub.go:111` |
| `git` | `config --global` | `user.name` | hardcoded | `doltserver.go:80` |
| `git` | `config --global` | `user.email` | hardcoded | `doltserver.go:92` |
| `dolt` | `config --global` | `--get <key>` | runtime | `doltserver.go:110` |
| `dolt` | `config --global` | `--unset <key>` | runtime | `doltserver.go:129` |
| `dolt` | `config --global` | `--add <key> <value>` | runtime | `doltserver.go:132` |
| `dolt` | (dynamic) | `fullArgs...` | caller | `doltserver.go:420` |
| `lsof` | `-i :<port>` | `-sTCP:LISTEN -t` | hardcoded | `doltserver.go:736` |
| `ss` | `-tlnp` | `sport = :<port>` | hardcoded | `doltserver.go:749` |
| `lsof` | `-a -c dolt` | `-sTCP:LISTEN -i TCP -n -P -F pn` | hardcoded | `doltserver.go:782` |
| `ps` | `-p <pid>` | `-o args=` | runtime | `doltserver.go:876` |
| `lsof` | `-a -p <pid>` | `-d cwd -Fn` | runtime | `doltserver.go:893` |
| `ps` | `-eo pid,args` | (none) | hardcoded | `doltserver.go:1127` |
| `dolt` | `sql-server` | startup args | config | `doltserver.go:1594` |
| `lsof` | `<lockPath>` | (none) | runtime | `doltserver.go:1688` |
| `lsof` | `<socketPath>` | (none) | runtime | `doltserver.go:1728` |
| `dolt` | `init` | (none) | hardcoded | `doltserver.go:2295` |
| `dolt` | (dynamic) | `fullArgs...` | caller | `doltserver.go:3334` |
| `robocopy` | (Windows) | `/E /MOVE /R:1 /W:1` | hardcoded | `doltserver.go:3767` |
| `cp` | `-a` | `<src> <dest>` | runtime | `doltserver.go:3778` |
| `dolt` | `remote -v` | (none) | hardcoded | `sync.go:53` |
| `dolt` | `add .` | (none) | hardcoded | `sync.go:91` |
| `dolt` | `commit` | `-m <msg>` | hardcoded | `sync.go:99` |
| `dolt` | (push/pull) | `args...` | runtime | `sync.go:124` |
| `dolt` | `pull` | `<remote> main` | runtime | `sync.go:181` |
| `bd` | (dynamic) | `args...` | caller | `sync.go:627` |
| `bd` | `sql` | `<query>` | runtime | `wisps_migrate.go:120` |
| `bd` | `sql` | `--csv <query>` | runtime | `wisps_migrate.go:132` |
| `bd` | (dynamic) | `args...` | caller | `wisps_migrate.go:144` |

### SQL / config mutations
| Target | Statement | Value | Purpose | `file:line` |
|---|---|---|---|---|
| Dolt | `CREATE DATABASE` | `<rigName>` | Register new rig database | `doltserver.go:2276` |
| Dolt | `DROP DATABASE IF EXISTS` | `<dbName>` | Remove rig database | `doltserver.go:2773` |
| Dolt | `DELETE FROM dolt_branch_control` | `database = <dbName>` | Clean branch control | `doltserver.go:2781` |
| Dolt | health probe cycle | `CREATE TABLE; REPLACE INTO; DROP TABLE` | Health check | `doltserver.go:3513` |
| Dolt | `CREATE TABLE wisps` | DDL | Wisps migration | `wisps_migrate.go:351` |
| Dolt | `CREATE TABLE wisp_labels` | DDL | Wisps migration | `wisps_migrate.go:424` |
| Dolt | `CREATE TABLE wisp_comments` | DDL | Wisps migration | `wisps_migrate.go:433` |
| Dolt | `CREATE TABLE wisp_events` | DDL | Wisps migration | `wisps_migrate.go:445` |
| Dolt | `CREATE TABLE wisp_dependencies` | DDL | Wisps migration | `wisps_migrate.go:460` |
| Dolt | `UPDATE issues SET status = 'closed'` | agent wisps | Close agent issues on migration | `wisps_migrate.go:479` |
| Dolt | `CREATE TABLE IF NOT EXISTS _meta` | DDL | Wasteland schema | `wl_commons.go:154` |
| Dolt | `CREATE TABLE IF NOT EXISTS rigs` | DDL | Rig registry | `wl_commons.go:162` |
| Dolt | `CREATE TABLE IF NOT EXISTS wanted` | DDL | Bounty board | `wl_commons.go:176` |
| Dolt | `CREATE TABLE IF NOT EXISTS completions` | DDL | Completions | `wl_commons.go:196` |
| Dolt | `CREATE TABLE IF NOT EXISTS stamps` | DDL | Stamps | `wl_commons.go:210` |
| Dolt | `CREATE TABLE IF NOT EXISTS badges` | DDL | Badges | `wl_commons.go:235` |
| Dolt | `CREATE TABLE IF NOT EXISTS leaderboard` | DDL | Leaderboard | `wl_commons.go:243` |
| Dolt | `CREATE TABLE IF NOT EXISTS chain_meta` | DDL | Chain tracking | `wl_commons.go:255` |
| Dolt | `INSERT INTO wanted` | bounty data | Post bounty | `wl_commons.go:326` |
| Dolt | `UPDATE wanted SET claimed_by, status='claimed'` | claim data | Claim bounty | `wl_commons.go:351` |
| Dolt | `UPDATE wanted SET status='in_review'` | review data | Submit for review | `wl_commons.go:378` |
| Dolt | `INSERT INTO stamps` | stamp data | Record stamp | `wl_commons.go:611` |

### File writes
| Target | What is written | Purpose | `file:line` |
|---|---|---|---|
| Dolt config file | YAML content | Server configuration | `doltserver.go:1316` |
| PID file | PID string | Process identity | `doltserver.go:1468` |
| Dolt log file | append | Server log output | `doltserver.go:1552` |
| PID file | PID string | Process identity (post-start) | `doltserver.go:1618` |
| metadata JSON | atomic write | Server metadata | `doltserver.go:3076` |
| SQL script temp file | SQL content | Batch SQL execution | `doltserver.go:3936` |
| rollback dest file | file data | Rollback file copy | `rollback.go:254` |

## Notes / open questions

- **doltserver.go is ~3990 lines** — by far the largest single file
  in the package and one of the largest in the repo. Splitting by
  concern (start/stop, imposter detection, health, databases,
  workspace repair) would aid navigation but is a substantial
  refactor. Test coverage is high (~4460 lines of tests) so the
  refactor is safer than it sounds.
- **WL Commons is a surprise inhabitant** — now fully grounded in
  the expanded WL Commons section above. The `WLCommonsStore`
  interface enables test fakes. Finding wanted-list /
  character-sheet / leaderboard code inside `internal/doltserver`
  suggests either (a) a historical accident — WL Commons reused the
  per-town Dolt server and ended up co-located, or (b) an intentional
  "this is where town-scoped schemas live" convention. It would be
  more discoverable as `internal/wlcommons`.
- **Imposter killing assumes root control over the host.** If bd is
  running under a different UID, `os.Process.Kill()` will fail
  silently. The package does not surface this case — it just logs
  warnings and moves on.
- **Authentication is effectively none** for local mode. If the host
  is shared (desktop with untrusted users), any local user can write
  to another user's beads DB via port 3307. The package doc at
  `doltserver.go:8-9` says "root user, no password for localhost" as
  if it were a feature; in a hardened deployment this is a caveat to
  flag.
- `Start`'s 5-stage wait loop (`flock` → kill idle monitors →
  imposter check → quarantine → write config → start) is tightly
  sequenced. Any future parallelization or optimization should
  respect the gt-nkn thundering-herd fix and the gt-hs1i2 phantom-DB
  quarantine — both of those exist because reordering broke things
  in the past.
- The DoltHub API base is a `var` explicitly so tests can override
  it (`dolthub.go:14-16`); future callers adding DoltHub endpoints
  should keep that pattern for testability.
- `CollectDatabaseOwners` and related "who owns which DB" helpers
  hint at a feature that may warrant its own concept page once the
  ownership model is fleshed out further.
