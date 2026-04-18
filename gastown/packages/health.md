---
title: internal/health
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-18
sources:
  - /home/kimberly/repos/gastown/internal/health/health.go
tags: [package, diagnostics, health, dolt, network]
phase3_audited: 2026-04-14
phase3_findings: [drift]
phase3_severities: [wrong]
phase3_findings_post_release: false
phase8_audited: 2026-04-17
phase8_findings: [none]
detail_depth: {params: 2, data_flow: 2, errors: 1, side_effects: 1}
---

# internal/health

Reusable health-check primitives aimed at the Gas Town data plane: TCP
probes, MySQL/Dolt latency checks, database enumeration, zombie
`dolt sql-server` detection, and backup-freshness calculations.

**Go package path:** `github.com/steveyegge/gastown/internal/health`
**File count:** 1 go file, 0 test files
**Imports (notable):** `database/sql` with the `go-sql-driver/mysql` blank
import, plus [`internal/doltserver`](../inventory/go-packages.md) for
listener discovery and [`internal/util`](util.md) for
`SetDetachedProcessGroup`.
**Imported by:** only [`internal/cmd/health.go`](../commands/health.md) —
the package's own doc comment claims it is "shared between the Doctor Dog
(daemon/doctor_dog.go) and the gt health CLI command", but doctor code does
not actually import it. See "Notes / open questions" below.

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/health/health.go`.

### Public API

All functions are free functions keyed to a Dolt host/port or a filesystem
path — there is no `HealthChecker` struct or shared state.

- `TCPCheck(host string, port int, timeout time.Duration) bool`
  (`health.go:25-33`) — opens a `net.DialTimeout("tcp", ...)`, closes it
  immediately, returns `true` if the dial succeeded. Used as a cheap
  "is the port listening at all?" probe before doing the heavier MySQL
  handshake.
- `LatencyCheck(host string, port int, timeout time.Duration) (time.Duration, error)`
  (`health.go:36-53`) — opens a `root@tcp(host:port)` MySQL DSN, runs
  `SELECT 1`, and returns the round-trip duration. The DSN hard-codes
  `timeout=5s&readTimeout=10s` on top of the caller-supplied `timeout`,
  and the query itself uses `context.WithTimeout`, so there are three
  separate timers stacked on this call.
- `DatabaseCount(host string, port int) (int, []string, error)`
  (`health.go:56-86`) — runs `SHOW DATABASES` and returns the count and
  names, filtering out `information_schema` and `mysql`. Context timeout
  is fixed at 10s. Used to detect "how many beads / mail / identity
  databases does this server actually have?".
- `ZombieResult struct { Count int; PIDs []int }` (`health.go:89-92`).
- `FindZombieServers(expectedPorts []int) ZombieResult`
  (`health.go:96-118`) — delegates to `doltserver.FindAllDoltListeners()`
  to enumerate every live `dolt sql-server` listener on the box, then
  subtracts the expected-ports set. Whatever's left is a zombie. The
  comment at `health.go:95` ("ZFC fix: gt-fj87") calls out that this uses
  lsof-based discovery rather than string matching on `pgrep`/`ps`, which
  is the historically less reliable approach.
- `BackupFreshness(dir string) time.Time` (`health.go:122-138`) — walks
  `dir` and returns the `ModTime` of the newest non-directory file. Any
  error during walk is swallowed. Returns zero `time.Time` when `dir`
  doesn't exist.
- `JSONLGitFreshness(gitRepo string) (time.Time, error)`
  (`health.go:141-162`) — shells out to `git -C <gitRepo> log -1
  --format=%ci` with a 10s context timeout and parses the commit time as
  `"2006-01-02 15:04:05 -0700"`. Used for the Dolt-to-JSONL archive whose
  freshness is tracked as "when did we last commit a backup slice?".
- `DirSize(path string) (int64, error)` (`health.go:165-177`) — simple
  recursive sum of `info.Size()` for every non-directory file under
  `path`. Unlike `BackupFreshness`, this one propagates walk errors.

### Internals / Notable implementation

- All MySQL DSNs go through `root@tcp(host:port)/?timeout=5s&readTimeout=10s`
  with no password — this implies the Dolt server is expected to be bound
  to localhost only and trust-authenticated. That assumption is inherited
  from [`internal/doltserver`](../inventory/go-packages.md) and isn't
  validated here.
- The `_ = path` line at `health.go:42` is a no-op preserving the
  `exec.LookPath` return value after the error check. It is structurally
  dead but kept for symmetry.

### Usage pattern

The only in-tree caller today is
[`gt health`](../commands/health.md) (see
`/home/kimberly/repos/gastown/internal/cmd/health.go`), which composes
these primitives into the full health report: TCP probe, latency probe,
database count, zombie scan, and backup freshness. The package is
deliberately a toolbox, not a pipeline — each function is independently
callable so other diagnostic commands could mix-and-match without pulling
in a `Report` struct.

## Docs claim

### Source
- `/home/kimberly/repos/gastown/internal/health/health.go:2-3` — package doc comment

### Verbatim
> Package health provides reusable health check functions for the Gas Town data plane.
> These checks are shared between the Doctor Dog (daemon/doctor_dog.go) and the
> gt health CLI command (cmd/health.go).

## Drift

### Package doc claims Doctor Dog shares this package — only gt health imports it

- **Claim source:** package doc comment at `/home/kimberly/repos/gastown/internal/health/health.go:2-3`
- **Docs claim:** "These checks are shared between the Doctor Dog (daemon/doctor_dog.go) and the gt health CLI command (cmd/health.go)."
- **Code does:** `rg '"github.com/steveyegge/gastown/internal/health"'` finds exactly one importer: `internal/cmd/health.go`. Neither `internal/daemon/doctor_dog.go` nor any file in `internal/doctor/` imports `internal/health`. The doctor package implements its own Dolt checks via [`internal/deps`](deps.md) and [`internal/doltserver`](doltserver.md).
- **Category:** `drift`
- **Severity:** `wrong`
- **Fix tier:** `code` — the package doc comment should be corrected to remove the Doctor Dog claim
- **Release position:** `in-release`

## Related wiki pages

- [gt](../binaries/gt.md) — parent binary.
- [gt health](../commands/health.md) — the lone current caller.
- [internal/doctor](doctor.md) — sibling diagnostics package. Despite the
  package-level doc comment, doctor does not currently import `health`;
  it does its own Dolt checks via [`internal/deps`](deps.md) and internal
  Dolt-server helpers.
- [gt doctor](../commands/doctor.md) and [gt vitals](../commands/vitals.md)
  — sibling diagnostic CLI surfaces.
- [internal/util](util.md) — for `SetDetachedProcessGroup` (the
  git subprocess in `JSONLGitFreshness`).
- [go-packages inventory](../inventory/go-packages.md).

## Outgoing calls

### Subprocess invocations
| Called binary | Command | Flags | Flag source | `file:line` |
|---|---|---|---|---|
| `git` | `log` | `-C <gitRepo> -1 --format=%ci` | hardcoded | `health.go:149` |

## Notes / open questions

- **Doc-comment drift** → promoted to [## Drift](#package-doc-claims-doctor-dog-shares-this-package--only-gt-health-imports-it) above. Verified at HEAD: `internal/daemon/` was mapped in Batch 8 and does not import `internal/health`. The claim was never true in the import graph.
- `BackupFreshness` silently ignores walk errors but `DirSize` propagates
  them — minor API inconsistency for otherwise parallel helpers.
- No test file. The package has zero unit tests, which is plausible given
  that almost every function requires a live Dolt server or filesystem
  state to exercise.
