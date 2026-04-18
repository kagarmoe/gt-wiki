---
title: gt dolt
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/cmd/dolt.go
  - /home/kimberly/repos/gastown/internal/cmd/dolt_flatten.go
  - /home/kimberly/repos/gastown/internal/cmd/dolt_rebase.go
tags: [command, services, dolt, sql, database, beads-exempt]
phase3_audited: 2026-04-15
phase3_findings: [wiki-stale]
phase3_severities: [wrong]
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
phase5_audience: user
phase8_audited: 2026-04-17
phase8_findings: [none]
---

# gt dolt

Manages the per-town Dolt SQL server — the multi-client database
backend that holds beads, mail, identity, and work history for every
rig in the town. Wraps and partially replaces the upstream `dolt` CLI
with workspace-aware operations.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupServices` ("Services")
**Polecat-safe:** no
**Beads-exempt:** yes (in `beadsExemptCommands` on
`/home/kimberly/repos/gastown/internal/cmd/root.go:73` — `gt dolt`
is the bootstrap path that *creates* the bead store, so it cannot
itself require beads)
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/dolt.go:20-389`.

### Invocation

```
gt dolt                            # parent — RunE: requireSubcommand
gt dolt init
gt dolt start
gt dolt stop
gt dolt restart
gt dolt kill-imposters [--dry-run]
gt dolt status
gt dolt logs [-n N] [-f]
gt dolt dump
gt dolt sql
gt dolt init-rig <name>
gt dolt list
gt dolt migrate [--dry-run]
gt dolt fix-metadata
gt dolt recover
gt dolt cleanup [--dry-run] [--force]
gt dolt rollback [backup-dir] [--dry-run] [--list]
gt dolt sync [--db NAME] [--dry-run] [--force] [--gc]
gt dolt pull [--db NAME] [--dry-run]
gt dolt migrate-wisps [--dry-run] [--db NAME]
gt dolt flatten <database> [--yes-i-am-sure]
gt dolt rebase <database> [--yes-i-am-sure] [--keep-recent N] [--dry-run]
```

The bare `gt dolt` is a parent command — `RunE: requireSubcommand`
(`dolt.go:24`).

### Server fundamentals

From the `Long` (`dolt.go:25-35`):
- Port: **3307** (avoids MySQL's 3306)
- User: **root** (no password for localhost)
- Data dir: `.dolt-data/` (one subdir per rig database)
- Each rig (`hq`, `gastown`, `beads`, …) gets its own database

Subcommands resolve config via
`doltserver.DefaultConfig(townRoot)` and short-circuit if
`config.IsRemote()` — start/stop are managed externally for remote
servers (`dolt.go:399-401`, `:484-487`, etc.).

### Subcommands

**`init`** (`dolt.go:38-54`)
Scans every rig's `.beads/metadata.json` for Dolt server config and
ensures referenced databases actually exist in `.dolt-data/`. Migrates
from a local `.beads/dolt/` if present, otherwise creates a fresh
database. Idempotent — safe to run repeatedly.

**`start`** (`dolt.go:56-63`, run: `dolt.go:392-442`)
Refuses if `config.IsRemote()`. Refuses if no databases found
(redirects to `gt dolt init-rig <name>` — `dolt.go:407-409`). Calls
[`doltserver`](../packages/doltserver.md)`.Start(townRoot)`, prints PID/port/data-dir/connection
string, then runs `doltserver.VerifyDatabasesWithRetry(townRoot, 5)` to
confirm every on-disk DB is actually being served. If verification
detects DBs on disk but not served, it warns about a stale manifest and
suggests `dolt fsck --repair`.

**`stop`** (`dolt.go:65-70`, run: `dolt.go:478-497`)
Refuses if remote. Calls `doltserver.Stop(townRoot)` and reports the
old PID.

**`restart`** (`dolt.go:72-87`, run: `dolt.go:499-566`)
Documented as "the nuclear option for recovering from a hijacked port."
Steps: stop the tracked server, kill imposters on the configured port,
sleep 500ms, then start the correct server from `.dolt-data/`.
Verification reuses the same `VerifyDatabasesWithRetry` cascade as
`start`.

**`kill-imposters`** (`dolt.go:89-105`, run: `dolt.go:444-476`)
Finds and kills any `dolt sql-server` process holding this workspace's
configured port but serving a *different* data directory. Reports
PID, conflicting data dir, and expected data dir. Uses
`doltserver.CheckPortConflict` to identify imposters and
`doltserver.KillImposters` to kill them. The legitimate server is never
touched. `--dry-run` reports without killing.

**`status`** (`dolt.go:109-114`, run: `dolt.go:568-…`)
Reports running/PID, distinguishes local vs remote (`IsRemote`
branch), and prints `reachable` for remote.

**`logs`** (`dolt.go:116-121`)
Tails the Dolt server log file. Same `-n / -f` shape as
[daemon logs](./daemon.md) — `--lines` (default 50), `--follow`.

**`dump`** (`dolt.go:123-133`)
Sends `SIGQUIT` to the Dolt server. Per the long-form help, citing Tim
Sehn (Dolt CEO), this prints all goroutine stacks to stderr — captured
by the server log redirect. Used for diagnosing hung servers
(matches the `gt dolt status` / `kill -QUIT` workflow documented in
the project root `CLAUDE.md` "Dolt Server — Operational Awareness"
section).

**`sql`** (`dolt.go:135-143`)
Opens an interactive Dolt SQL shell. Works in both embedded and
server mode.

**`init-rig <name>`** (`dolt.go:145-159`)
Creates a new rig database in `.dolt-data/<name>`. The rig name
becomes the MySQL database name when connecting via the protocol.

**`list`** (`dolt.go:161-166`)
Lists all rig databases in `.dolt-data/`.

**`migrate`** (`dolt.go:168-184`)
Migrates legacy `.beads/dolt/` per-rig databases into the centralized
`.dolt-data/<rigname>/` layout. Detects sources, moves them, removes
old empty dirs. `--dry-run` previews source/target paths and sizes.

**`fix-metadata`** (`dolt.go:186-199`)
Idempotently rewrites every rig `.beads/metadata.json` to set
`backend: "dolt"`, `dolt_mode: "server"`, `dolt_database: "<rigname>"`.
Fixes the split-brain where bd falls back to embedded mode instead of
the centralized server. Preserves other fields.

**`recover`** (`dolt.go:201-219`)
Probes whether the server is in read-only mode (e.g., from concurrent
write contention on the storage manifest). If so: stop, restart,
write-probe to verify recovery. No-op if already writable. The daemon
runs this automatically every 30 seconds; the command exists for
immediate recovery without waiting for the daemon health-check loop.

**`sync`** (`dolt.go:221-247`)
Pushes all local Dolt databases to their configured DoltHub remotes.
**Prefers SQL** (`CALL DOLT_PUSH`) when the server is running so the
server stays up and agents are not disrupted; falls back to CLI push
(which requires stopping the server) only when offline.
Flags: `--db NAME`, `--dry-run`, `--force`, `--gc` (purges closed
ephemeral beads / wisps / convoys before pushing).

**`pull`** (`dolt.go:249-267`)
Mirror of `sync`. Pulls all local databases from configured remotes.
Same SQL-first preference (`CALL DOLT_PULL`) to avoid the exclusive
lock contention that bare `dolt pull` causes when the server is
running. Flags: `--db NAME`, `--dry-run`.

**`cleanup`** (`dolt.go:269-284`)
Removes orphaned databases from `.dolt-data/` — directories that exist
on disk but are not referenced by any rig's `metadata.json`. Typically
left over from partial setups, renamed databases, or failed
migrations. `--dry-run` previews; `--force` removes even databases that
contain user tables. (This is the command the project root `CLAUDE.md`
recommends as "`gt dolt cleanup`" instead of `rm -rf ~/.dolt-data/`.)

**`rollback [backup-dir]`** (`dolt.go:286-305`)
Restores `.beads` directories from a `migration-backup-TIMESTAMP/`
created during migration. Steps: stop server, find backup (most recent
if none specified), restore `.beads` dirs, reset `metadata.json` to
pre-migration state, validate via `bd list`. `--list` shows available
backups and exits; `--dry-run` previews.

**`migrate-wisps`** (`dolt.go:307-324`)
Creates the `wisps` table (and `wisp_labels`, `wisp_comments`,
`wisp_events`, `wisp_dependencies`) and migrates `issue_type='agent'`
beads from `issues` to `wisps`, closing the originals. Idempotent.
After this runs, `bd mol wisp list` works and the agent lifecycle
(spawn, sling, work, done, nuke, respawn) uses the wisps table.

**`flatten <database>`** (`dolt_flatten.go:23-42`, run: `dolt_flatten.go:48-…`)
Registered via `dolt_flatten.go:init()` at `:43-46`. The "NUCLEAR
OPTION" for compaction — destroys all commit history by soft-resetting
to the root commit and recommitting everything as a single commit.
Safety protocol: pre-flight row counts, `DOLT_RESET('--soft',
rootHash)`, `DOLT_COMMIT`, post-flight row count verification.
Requires `--yes-i-am-sure` flag as safety interlock. All operations
run via SQL on the running server — no downtime.

**`rebase <database>`** (`dolt_rebase.go:25-55`, run: `dolt_rebase.go:66-…`)
Registered via `dolt_rebase.go:init()` at `:56-63`. Surgical
compaction using Dolt's `DOLT_REBASE`: squashes old commits while
keeping the most recent N (`--keep-recent`, default 50) as individual
picks. Creates anchor + work branches, populates `dolt_rebase` table,
executes the plan, then swaps branches. **Not safe with concurrent
writes** — unlike `flatten`. Requires `--yes-i-am-sure` or
`--dry-run`. The Compactor Dog (daemon) has automatic retry logic for
concurrent-write failures.

### Flags

Per `dolt.go:326-388`. All scoped to their respective subcommands.

| Subcommand        | Flag             | Type   | Default |
|-------------------|------------------|--------|---------|
| `kill-imposters`  | `--dry-run`      | bool   | false   |
| `logs`            | `--lines / -n`   | int    | 50      |
| `logs`            | `--follow / -f`  | bool   | false   |
| `migrate`         | `--dry-run`      | bool   | false   |
| `cleanup`         | `--dry-run`      | bool   | false   |
| `cleanup`         | `--force`        | bool   | false   |
| `rollback`        | `--dry-run`      | bool   | false   |
| `rollback`        | `--list`         | bool   | false   |
| `sync`            | `--dry-run`      | bool   | false   |
| `sync`            | `--force`        | bool   | false   |
| `sync`            | `--db`           | string | ""      |
| `sync`            | `--gc`           | bool   | false   |
| `pull`            | `--dry-run`      | bool   | false   |
| `pull`            | `--db`           | string | ""      |
| `migrate-wisps`   | `--dry-run`      | bool   | false   |
| `migrate-wisps`   | `--db`           | string | ""      |
| `flatten`         | `--yes-i-am-sure`| bool   | false   |
| `rebase`          | `--yes-i-am-sure`| bool   | false   |
| `rebase`          | `--keep-recent`  | int    | 50      |
| `rebase`          | `--dry-run`      | bool   | false   |

## Related

- [maintain](./maintain.md) — wraps several Dolt operations (reap +
  flatten + gc) into a single supervised pipeline
- [compact](./compact.md) — separate compaction primitive
- [doctor](./doctor.md) — health checks that inspect Dolt server state
- [down](./down.md) — stops the Dolt server as part of town shutdown

## Docs claim

### Source
- `/home/kimberly/repos/gastown/internal/cmd/dolt.go:25-35` — Cobra `Long` text

### Verbatim
> Manage the Dolt SQL server for Gas Town beads.
>
> The Dolt server provides multi-client access to all rig databases,
> avoiding the single-writer limitation of embedded Dolt mode.
>
> Server configuration:
>   - Port: 3307 (avoids conflict with MySQL on 3306)
>   - User: root (default Dolt user, no password for localhost)
>   - Data directory: .dolt-data/ (contains all rig databases)
>
> Each rig (hq, gastown, beads) has its own database subdirectory.

The `doltCmd.Long` does not enumerate subcommands — it describes
server configuration only. No enumeration drift. However, two
subcommands registered from sibling files were missed by Phase 2.

**wiki-stale (inline fix):** Phase 2's invocation block listed 19
subcommands from `dolt.go` only. Two sibling files register additional
subcommands via their own `init()` blocks:
- `dolt_flatten.go:init()` at `:43-46` registers `gt dolt flatten`
- `dolt_rebase.go:init()` at `:56-63` registers `gt dolt rebase`
Total registered subcommands: **21** (not 19). The invocation block,
subcommand descriptions, and flags table have been updated inline.
**Phase 2 root cause: `phase-2-incomplete` (heuristic)** — Phase 2
read only `dolt.go` in `sources:` and missed sibling-file cobra
registrations. Both sibling files existed at Phase 2 time.

See also: [gastown/drift/README.md](../drift/README.md)

## Failure modes

No failure modes discovered. `dolt.go` wraps `doltserver` package operations (start/stop/cleanup/status/migrate). All error paths propagated. The `flatten` and `rebase` subcommands operate on Dolt databases with error checking throughout.

## Notes / open questions

- **`recover`'s 30s daemon cadence**: where in `internal/daemon` is the
  health-check loop, and what does it call? Worth grounding before the
  next dolt incident.
- **`SwapKeychainCredential` vs `dolt sync` auth**: how does Dolt push
  authenticate to DoltHub? Token? `dolt creds`?
- **`fix-metadata` field preservation**: claim is "preserves other
  fields" — verify by reading the implementation in `runDoltFixMetadata`.
- **`rollback` backup discovery**: it picks "most recent
  `migration-backup-TIMESTAMP/`" — is the search rooted at the town
  root or somewhere else?
- **`migrate-wisps` is destructive**: closes originals in the `issues`
  table. Idempotency claim deserves a test grounding.
- **The `dump` subcommand and the `CLAUDE.md` Dolt protocol** — these
  are aligned but not identical. The CLAUDE.md guidance uses a raw
  `kill -QUIT $(cat ~/gt/.dolt-data/dolt.pid)`; `gt dolt dump` should
  be the supported equivalent. Worth a note on the page.
