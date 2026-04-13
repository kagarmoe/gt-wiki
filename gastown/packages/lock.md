---
title: internal/lock
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/lock/lock.go
  - /home/kimberly/repos/gastown/internal/lock/flock_unix.go
  - /home/kimberly/repos/gastown/internal/lock/flock_windows.go
  - /home/kimberly/repos/gastown/internal/lock/process_unix.go
  - /home/kimberly/repos/gastown/internal/lock/process_windows.go
tags: [package, data-layer, locks, flock, agent-identity, tmux, cross-platform]
---

# internal/lock

Agent-identity locking for worker directories. When an agent (polecat,
crew, etc.) claims a worker, this package writes a JSON lock file at
`<workerDir>/.runtime/agent.lock` containing the PID, timestamp, session
ID, and hostname so a second process can't accidentally claim the same
identity. It also provides a general-purpose `flock`-based coordination
primitive (`FlockAcquire`, `FlockTryAcquire`) that other packages reuse for
their own cross-process read-modify-write serialization.

**Go package path:** `github.com/steveyegge/gastown/internal/lock`
**File count:** 5 go files, 1 test file
**Imports (notable):**

- [`internal/tmux`](../packages/cli.md) — `tmux.GetDefaultSocket()` to
  find the right tmux server when verifying active sessions
  (`lock.go:307`).
- stdlib: `encoding/json`, `errors`, `os`, `os/exec`, `path/filepath`,
  `syscall`, `time`, `golang.org/x/sys/windows` (Windows-only).

**Imported by (notable):** the spawn/attach path for polecats and crew
workers (identity acquisition), `gt doctor` stale-lock cleanup, the daemon
and witness (`CleanStaleLocks`, `DetectCollisions`) during patrol, and any
package that needs a cross-process mutex on a specific file path via
`FlockAcquire` / `FlockTryAcquire`.

## What it actually does

### Lock file location and shape

`<workerDir>/.runtime/agent.lock` — a JSON file matching `LockInfo`
(`lock.go:32-37`):

```go
type LockInfo struct {
    PID        int       `json:"pid"`
    AcquiredAt time.Time `json:"acquired_at"`
    SessionID  string    `json:"session_id,omitempty"`  // tmux session name
    Hostname   string    `json:"hostname,omitempty"`
}
```

The `.runtime/` subdirectory holds the lock file; the coordination flock
sits beside it at `agent.lock.flock`.

### Public API — agent locks (`lock.go`)

- `lock.New(workerDir string) *Lock` (`lock.go:51`) — constructs a `Lock`
  bound to a worker directory. Does not touch the filesystem.
- `(*Lock).Acquire(sessionID string) error` (`lock.go:64-102`) — the
  canonical entry point. Ensures `.runtime/` exists, acquires an advisory
  flock on `agent.lock.flock` to serialize concurrent acquisitions, then:
  - reads any existing lock file.
  - if it's stale (PID dead) removes it and proceeds.
  - if we already hold it (`info.PID == os.Getpid()`), refreshes the
    timestamp and session ID.
  - if another live process holds it, returns `ErrLocked` wrapped with
    PID/session/acquired-at context.
  - otherwise writes a fresh lock.
- `(*Lock).Release() error` (`lock.go:105-110`) — removes the lock file
  (tolerates `ENOENT`).
- `(*Lock).Read() (*LockInfo, error)` (`lock.go:113-128`) — read without
  modifying; returns `ErrNotLocked` if the file doesn't exist and
  `ErrInvalidLock` if JSON is malformed.
- `(*Lock).Check() error` (`lock.go:134-157`) — answers "is it safe for
  me to run here?". Returns `nil` if unlocked or locked by us, `ErrLocked`
  if held by another live process. Auto-cleans stale locks as a side
  effect.
- `(*Lock).Status() string` (`lock.go:160-178`) — human-readable summary:
  `"unlocked"`, `"locked (by us)"`, `"stale (dead PID N)"`, or
  `"locked by PID N (session: ...)"`.
- `(*Lock).ForceRelease() error` (`lock.go:181-183`) — alias for
  `Release`, used by `gt doctor --fix` when deliberately clearing another
  process's lock.

### Public API — error sentinels

`ErrLocked`, `ErrNotLocked`, `ErrInvalidLock` (`lock.go:26-29`). Callers
test with `errors.Is`.

### Public API — generic flock primitives

These exist so other packages (mail, beads, etc.) can reuse the same
cross-process locking mechanism without reimplementing it:

- `FlockAcquire(path string) (func(), error)` (`flock_unix.go:15-17`) —
  opens the given path, acquires an exclusive `LOCK_EX` advisory lock,
  returns a cleanup function that releases and closes. Blocks until the
  lock is available.
- `FlockTryAcquire(path string) (func(), bool, error)`
  (`flock_unix.go:44-63`) — non-blocking variant using `LOCK_EX | LOCK_NB`.
  Returns `(nil, false, nil)` when another process holds the lock
  (`EWOULDBLOCK`), `(cleanup, true, nil)` on success.
- `flockAcquire(path string) (func(), error)` (`flock_unix.go:22-38`) —
  unexported companion used internally by `Lock.Acquire`.

### Public API — batch lock operations

- `FindAllLocks(root string) (map[string]*LockInfo, error)`
  (`lock.go:227-257`) — walks a directory tree finding every
  `.runtime/agent.lock`. Skips `.dolt-data/`, `.dolt-backup/`, and
  `.git/` for speed on macOS VirtioFS / Docker bind mounts. Returns a
  map of worker directory to lock info.
- `CleanStaleLocks(root string) (int, error)` (`lock.go:264-294`) — the
  interesting one. A lock is "stale" for cleanup purposes only if **both**
  the PID is dead **and** no tmux session with the lock's `SessionID` is
  still alive. This is critical: under Claude Code, the spawning parent
  process routinely exits while the tmux-hosted child keeps running, so
  a naive PID-only check would kill live workers. To verify sessions,
  `CleanStaleLocks` calls `getActiveTmuxSessions` which shells out to
  `tmux -L <socket> list-sessions -F '#{session_name}:#{session_id}'`
  against the correct per-town socket (`lock.go:299-337`). It normalizes
  session IDs across the `$N` and legacy `%N` formats.
- `DetectCollisions(root string, activeSessions []string) []string`
  (`lock.go:398-429`) — diagnostic used by `gt doctor`. Returns
  descriptions of stale locks and *orphaned* locks (live PID but
  non-existent session).

### Internals / Notable implementation

- **Race-free acquire.** `Acquire` uses a separate `.flock` file to
  serialize the `read -> decide -> write` path across processes
  (`lock.go:73-77`). Without it, two processes could both read an empty
  lock and both claim the worker. The pattern is textbook TOCTOU
  avoidance.
- **Atomic lock-file writes.** `write` serializes to a temp file
  (`agent.lock.tmp`) and then `os.Rename`s into place so a crash can
  never leave a partially written JSON blob (`lock.go:207-218`). Works
  because POSIX rename within a single directory is atomic.
- **Platform abstraction.** The package has separate `*_unix.go` and
  `*_windows.go` files gated by build tags:
  - `flock_unix.go` uses `syscall.Flock` with `LOCK_EX` / `LOCK_UN`.
  - `flock_windows.go` is a no-op returning `(func(){}, nil)` — Gas
    Town isn't run on Windows in production, so the coordination lock
    degrades to unlocked behavior there.
  - `process_unix.go` checks liveness by `syscall.Signal(0)` via
    `os.FindProcess`.
  - `process_windows.go` uses
    `windows.OpenProcess(PROCESS_QUERY_LIMITED_INFORMATION, ...)`, treating
    `ERROR_ACCESS_DENIED` as "alive" since we were able to see it exists.
- **Detached child process groups.** `setProcessGroup` gives subprocess
  children (the tmux lookup) their own process group on Unix
  (`Setpgid = true`) or `CREATE_NEW_PROCESS_GROUP | CREATE_NO_WINDOW` on
  Windows. Prevents signal propagation issues when the parent gt is
  interrupted.
- **`execCommand` indirection.** `lock.go:355-372` wraps `exec.Command`
  behind an interface-returning function variable so tests can swap in
  a fake tmux.

### Lock types covered by this package

| Lock kind | Mechanism | What it protects |
|---|---|---|
| Agent identity lock | JSON file + advisory flock on `.flock` sidecar | Worker-dir ownership (one agent per worker) |
| Generic cross-process mutex | `FlockAcquire` / `FlockTryAcquire` on any path | Any read-modify-write a caller wants to serialize |

**Not covered here:** bd's own SQLite/Dolt lock files, `.dolt-data/` locks,
the mail layer's per-inbox locks, per-bead locks (those live in
`internal/beads/beads_agent.go` using `gofrs/flock` directly). This package
is specifically about *agent worker identity* and the *generic flock
primitive* other packages build on.

### Usage pattern

```go
l := lock.New(workerDir)
if err := l.Acquire(tmuxSessionID); err != nil {
    if errors.Is(err, lock.ErrLocked) {
        // Another live agent owns this worker
        return err
    }
}
defer l.Release()
```

Or, during `gt doctor`:

```go
cleaned, err := lock.CleanStaleLocks(townRoot)
```

## Related wiki pages

- [gt](../binaries/gt.md)
- [internal/beads](beads.md) — beads has its own per-bead flock discipline
  and also uses `FlockAcquire` via this package for its coordination
  paths.
- [internal/session](session.md) — session lifecycle; the session ID
  carried in `LockInfo` is a tmux session name.
- [internal/workspace](workspace.md) — workspace/worker directory
  resolution.
- [gt doctor](../commands/doctor.md) — uses `CleanStaleLocks` and
  `DetectCollisions` in fix mode.
- [go-packages inventory](../inventory/go-packages.md)

## Notes / open questions

- Windows flock support is a stub. The comments are explicit: "Gas Town
  doesn't run on Windows in production." A future port would need real
  `LockFileEx`/`UnlockFileEx` wiring.
- `CleanStaleLocks` shells out to tmux for every call; for patrols that
  run frequently this is cheap because tmux is already running, but a
  cached session list could be threaded through if it ever matters.
- The `FindAllLocks` skip list (`.dolt-data`, `.dolt-backup`, `.git`) was
  added specifically to survive macOS VirtioFS performance. If new
  always-huge directories appear in town trees, add them here.
- The lock file's `Hostname` field is populated but not currently used
  by any decision logic in the package — it exists for diagnostic
  readability when a shared filesystem holds locks from multiple hosts.
