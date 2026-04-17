---
title: internal/testutil
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-15
sources:
  - /home/kimberly/repos/gastown/internal/testutil/cmd.go
  - /home/kimberly/repos/gastown/internal/testutil/doltserver.go
  - /home/kimberly/repos/gastown/internal/testutil/doltserver_windows.go
  - /home/kimberly/repos/gastown/internal/testutil/townenv.go
tags: [package, testing, helpers, dolt, environment]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
---

# internal/testutil

Shared test helpers for Gas Town integration tests. Three thin layers:
env-sanitized `gt`/`bd` subprocess command builders, a shared Dolt
testcontainer lifecycle for tests that need a real SQL server, and a
workspace-present guard that skips tests when they aren't running inside
a live Gas Town directory tree.

**Go package path:** `github.com/steveyegge/gastown/internal/testutil`
**File count:** 4 go files (3 unix + 1 windows stub), 2 test files
**Imports (notable):** stdlib (`os`, `os/exec`, `strings`, `testing`),
`github.com/testcontainers/testcontainers-go` + its `dolt` module,
`github.com/go-sql-driver/mysql` (blank import required by the dolt
testcontainer module), and [`internal/workspace`](workspace.md) for the
town-root discovery helper used by `RequireTownEnv`.
**Imported by (notable):** any `_test.go` file in the repo that needs
an isolated Dolt server, wants to shell out to `gt` or `bd` with a
sanitized environment, or guards itself on the presence of a real
town.

## What it actually does

### Subprocess helpers (`cmd.go`)

- `CleanGTEnv(extraEnv ...string) []string` (`cmd.go:17-31`) — returns
  `os.Environ()` stripped of all `GT_*` and `BD_*` variables **except**
  `GT_DOLT_PORT` and `GT_DOLT_HOST`, then appends any caller-supplied
  env entries. `BEADS_*` variables (note the prefix mismatch with
  `BD_*`) pass through implicitly. The intent is "subprocess that
  doesn't inherit the developer's town context but still talks to the
  test Dolt server."
- `NewBDCommand(args...)` / `NewGTCommand(args...)` (`cmd.go:38-49`) —
  plain `exec.Command` wrappers that inherit the full process
  environment. The comment explicitly notes they exist as a hint:
  "use these instead of bare `exec.Command("bd", ...)` in tests" so
  that the `GT_DOLT_PORT` propagation expectation is obvious at the
  call site.
- `NewIsolatedBDCommand` / `NewIsolatedGTCommand` (`cmd.go:55-69`) —
  pre-populate `cmd.Env` with `CleanGTEnv()` for the isolated path.

### Dolt testcontainer (`doltserver.go`, unix)

- `DoltDockerImage = "dolthub/dolt-sql-server:1.83.0"` (`doltserver.go:23`)
  — pinned test image. `DOLT_ROOT_HOST=%` is set on the container so
  `root@'%'` is created and TCP connections work (available since
  Dolt 1.46.0).
- `isDockerAvailable()` (`doltserver.go:36-41`) — caches the result of
  `docker info` so tests can skip cleanly without Docker.
- `runDoltContainerWithRetry(ctx)` (`doltserver.go:53-75`) — retries
  `dolt.Run` up to 3 times with 2s→4s→8s backoff, specifically to
  tolerate the transient testcontainers-Ryuk "reaper removing" error
  seen when a previous test run's reaper container is still being
  cleaned up.
- `startSharedDoltContainer()` (`doltserver.go:79-98`) — populates a
  `sync.Once`-protected module-level `doltCtr` and sets `GT_DOLT_PORT`
  and `BEADS_DOLT_PORT` process-wide via `os.Setenv`.
- `StartIsolatedDoltContainer(t)` (`doltserver.go:103-128`) — per-test
  variant that uses `t.Setenv` (test-scoped) and `t.Cleanup` for
  teardown. Skips on missing Docker.
- `EnsureDoltContainerForTestMain()` (`doltserver.go:133-140`) — the
  `TestMain` entry point. Uses `sync.Once` to stand up the shared
  container once per test binary. Paired with `TerminateDoltContainer`
  (`doltserver.go:168-173`) which is called after `m.Run()`.
- `RequireDoltContainer(t)` (`doltserver.go:144-154`) — per-test guard
  that ensures the shared container is running and calls `t.Skip` if
  Docker is unavailable.
- `DoltContainerAddr()` / `DoltContainerPort()` (`doltserver.go:157-164`)
  — accessors for the mapped port that test code can plug into its own
  DSN construction.

### Windows stub (`doltserver_windows.go`)

Build-tagged `//go:build windows`. Every function above becomes a
`t.Skip("Docker not available on Windows CI")` or `errors.New(...)`
(`doltserver_windows.go:10-40`). The image constant is kept in sync
with the unix file so grep hits both.

### Town-env guard (`townenv.go`)

- `RequireTownEnv(t) string` (`townenv.go:21-37`) — calls
  `workspace.FindFromCwd` and skips the test if the caller isn't inside
  a town, then further verifies `<root>/mayor/rigs.json` exists as a
  proxy for "a fully initialized town" (not just a bare marker file).
  Returns the resolved `townRoot` for the test to use. This guard is
  specifically **not** needed by tests that create their own temporary
  town structure via `t.TempDir`.

## Related wiki pages

- [gt](../binaries/gt.md) — the binary under test; the subprocess
  helpers target its CLI entry point.
- [internal/workspace](workspace.md) — the package that
  `RequireTownEnv` delegates to for town-root discovery.
- [internal/doltserver](doltserver.md) — production-side counterpart
  that `gt dolt start` manages. `testutil` never touches the
  production Dolt setup — it uses fresh testcontainers instead.
- [go-packages inventory](../inventory/go-packages.md).

## Notes / open questions

- The `BEADS_*` prefix is preserved by `CleanGTEnv` while `BD_*` is
  stripped (`cmd.go:11-14`) — a quiet gotcha for callers who think
  "bd-related env" means "anything starting with BD". Tests that set
  `BEADS_DOLT_PORT` before spawning an isolated subprocess will see
  it inherited; tests that set `BD_FOO` will not.
- `doltserver_windows.go` is a skip-only stub. Anything that actually
  needs a Dolt SQL server will silently pass on Windows CI — the
  assertion coverage on that platform is intentionally reduced.
