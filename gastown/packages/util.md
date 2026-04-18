---
title: internal/util
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-18
sources:
  - /home/kimberly/repos/gastown/internal/util/atomic.go
  - /home/kimberly/repos/gastown/internal/util/exec.go
  - /home/kimberly/repos/gastown/internal/util/exec_unix.go
  - /home/kimberly/repos/gastown/internal/util/exec_windows.go
  - /home/kimberly/repos/gastown/internal/util/path.go
  - /home/kimberly/repos/gastown/internal/util/slice.go
  - /home/kimberly/repos/gastown/internal/util/url.go
  - /home/kimberly/repos/gastown/internal/util/orphan.go
  - /home/kimberly/repos/gastown/internal/util/orphan_windows.go
tags: [package, platform-service, util, atomic-write, exec, process-groups, orphan-cleanup, redaction]
phase3_audited: 2026-04-15
phase3_findings: [wiki-stale]
phase3_severities: [incomplete]
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
phase8_audited: 2026-04-17
phase8_findings: [precondition, cross-platform]
detail_depth: {params: 2, data_flow: 1, errors: 1, side_effects: 2}
---

# internal/util

Grab-bag of low-level helpers: atomic JSON/file writes, subprocess
wrappers that set platform-specific process groups (for Windows no-flash
and Unix kill-the-tree semantics), home-directory tilde expansion, slice
helpers, URL credential redaction, and — by far the largest single
member — the orphan/zombie Claude process hunter used by `gt doctor`
and daemon cleanup patrols.

**Go package path:** `github.com/steveyegge/gastown/internal/util`
**File count:** 9 go files (`atomic.go`, `exec.go`, `exec_unix.go`,
`exec_windows.go`, `orphan.go`, `orphan_windows.go`, `path.go`,
`slice.go`, `url.go`), 6 test files.
**Imports (notable):** stdlib (`encoding/json`, `os`, `os/exec`,
`syscall`, `path/filepath`, `net/url`, `regexp`, `strconv`, `time`), and
— inside `orphan.go` only — [`internal/tmux`](../inventory/go-packages.md)
and `internal/lock` for cross-socket pane-PID discovery.
**Imported by (notable):** nearly everywhere. The top consumers are:

- [`internal/version`](version.md) — every git subprocess goes through
  `SetDetachedProcessGroup`.
- [`internal/session`](session.md) — uses atomic writes for PID
  tracking and identity files.
- [`internal/config`](config.md) — `AtomicWriteJSON`/
  `EnsureDirAndWriteJSON` for saving all JSON config files.
- [`gt doctor`](../commands/doctor.md), [`gt cleanup`](../commands/cleanup.md)
  — call the orphan/zombie cleanup functions.
- Generally, any command page that shells out to git, ps, or another
  CLI binary lands here for the process-group wrapper.

## What it actually does

### atomic.go — atomic file writes

Source: `/home/kimberly/repos/gastown/internal/util/atomic.go`.

- `AtomicWriteFile(path, data, perm) error` (`atomic.go:58-95`) —
  writes to a `CreateTemp`-generated sibling file, `Close`s, `Chmod`s,
  then `os.Rename`s into place. The rename is atomic on POSIX. On
  error at any step, the temp file is removed. The pattern
  `base + ".tmp.*"` in the `CreateTemp` call prevents concurrent
  writers from colliding on the temp file name.
- `AtomicWriteJSON(path, v) error` (`atomic.go:14-20`) — marshal +
  `AtomicWriteFile` with 0644 perms.
- `AtomicWriteJSONWithPerm(path, v, perm) error` (`atomic.go:26-32`)
  — same with custom perms.
- `EnsureDirAndWriteJSON(path, v) error` (`atomic.go:39-44`) —
  `os.MkdirAll(filepath.Dir(path), 0755)` then `AtomicWriteJSON`. The
  most common pattern, since most callers write into a `<townRoot>/
  .runtime/...` path that may not exist yet.
- `EnsureDirAndWriteJSONWithPerm(...)` (`atomic.go:47-52`).

### exec.go / exec_unix.go / exec_windows.go — subprocess wrappers

- `FirstLine(s string) string` (`exec.go:13-21`) — returns the first
  non-empty trimmed line from `s`. Used to extract a meaningful error
  message from subprocess stderr when cobra commands helpfully tack on
  multi-line usage text after the actual failure.
- `ExecWithOutput(workDir, cmd, args...) (string, error)`
  (`exec.go:25-43`) — runs a command in `workDir`, installs
  `SetDetachedProcessGroup` (for Windows no-flash), captures stdout,
  and on failure formats stderr's content as the error. Returns
  trimmed stdout on success.
- `ExecRun(workDir, cmd, args...) error` (`exec.go:47-64`) — same
  pattern but discards stdout.
- `SetProcessGroup(cmd *exec.Cmd)` (unix: `exec_unix.go:12-20`;
  windows: `exec_windows.go:14-20`) — configures the child to run in
  its own process group. On Unix, installs a `cmd.Cancel` hook that
  `syscall.Kill(-pid, SIGKILL)`s the entire group so context
  cancellation kills the whole subtree. On Windows, sets
  `CREATE_NEW_PROCESS_GROUP | CREATE_NO_WINDOW` to detach from the
  parent console **and** suppress the transient console window
  Windows otherwise shows for console apps spawned from no-console
  parents (i.e. the daemon).
- `SetDetachedProcessGroup(cmd *exec.Cmd)` — same as `SetProcessGroup`
  on Windows; on Unix, sets `Setpgid` without installing the cancel
  hook (use when you explicitly *don't* want ctx-cancel to reach the
  child, e.g. a short-lived git query). Sources at
  `exec_unix.go:23-26` and `exec_windows.go:23-25`.

### path.go — home directory tilde expansion

- `ExpandHome(path string) string` (`path.go:25-34`) — replaces a
  leading `~/` with the user's home directory. Cached across calls via
  `sync.Once` (`path.go:10-12`). Returns the input unchanged if it
  doesn't start with `~/` or if `os.UserHomeDir()` fails.

### slice.go — two string-slice helpers

- `RemoveFromSlice(slice []string, item string) []string`
  (`slice.go:6-14`) — returns a new slice with all occurrences of
  `item` removed; original slice untouched.
- `ContainsString(slice []string, item string) bool`
  (`slice.go:17-24`).

### url.go — credential redaction

- `RedactURL(rawURL string) string` (`url.go:12-27`) — strips
  credentials from a standard URL for safe logging. Handles:
  - HTTPS with embedded credentials:
    `https://x-access-token:ghp_abc@github.com/org/repo` →
    `https://github.com/org/repo`.
  - SSH-style URLs (`git@github.com:org/repo.git`): returned as-is
    (not a standard URL).
  - Parse failures with no `@`: returned as-is (preserves debugging
    context when the URL is merely malformed).
  - Parse failures with an `@`: masked as `"<invalid URL>"` to avoid
    accidentally logging embedded secrets that `net/url` couldn't
    extract.

### orphan.go — Claude / codex / opencode cleanup

The biggest file in the package (~1000 lines; exact body starts at
`orphan.go:1`). Source comments describe the problem: Claude Code's
Task tool spawns subagent `claude` processes that sometimes survive
their tmux pane's exit, accumulating as zombies. The daemon's cleanup
patrol and `gt doctor` must find these and kill them — without
touching legitimate personal `claude` sessions or processes from
other Gas Towns on the same machine.

**Detection strategy** (two sibling paths):

1. **Orphans** — processes with TTY `?`/`??` (no controlling terminal)
   that match `claude`, `claude-code`, `codex`, or `opencode`. These
   are the primary Task-tool leak case.
2. **Zombies** — Claude processes that aren't members of any PID set
   belonging to any active tmux session on any socket. Catches cases
   where the tmux pane itself died.

**Safety guards** (explicit, multiple layers):

- `minOrphanAge = 60` seconds (`orphan.go:23`) — processes younger
  than this are never killed. Prevents race with freshly spawned
  subagents.
- `getTmuxSessionPIDs()` (`orphan.go:70-...`) — builds a protected set
  of PIDs by scanning **all** tmux sockets in `tmux.SocketDir()` plus
  descendant walking via a single shared `ps` snapshot
  (`buildChildMap()`, `orphan.go:27-48`). Protects cross-town
  processes: "We protect ALL tmux sessions on ALL sockets." The
  comment explicitly warns that a single-socket query would miss
  processes in other towns' sessions and cause cross-town kills.
- ACP session protection (`getACPSessionPIDs()`) — opencode agents
  running outside tmux have their own lifecycle and are protected
  (`orphan.go:433-438`, `543-548`).
- IDE-owned Claude processes — `isIDEClaudeProcess(pid)` excludes VS
  Code / Cursor extensions that show TTY `?` but are legitimate
  (`orphan.go:485-487`).
- **Workspace scoping** (`orphan.go:499-506`): `resolveTownRoot(pid)`
  — orphans are only killed when their cwd is under a Gas Town
  workspace root. Personal Claude instances outside `~/gt/` are
  untouched even if they have no TTY.
- **TOCTOU re-verification**: before signalling, both cleanup loops
  call `isProcessStillOrphaned(pid)` in case the process joined a
  tmux session between find and kill (`orphan.go:731-736`,
  `839-844`).

**Graceful signal escalation** — identical for orphans and zombies
(`orphan.go:760-766` and `663-666`):

1. First encounter → `SIGTERM`, record in a state file.
2. Next cycle, still alive after `sigkillGracePeriod` → `SIGKILL`,
   update state.
3. Next cycle, still alive after `SIGKILL` → mark unkillable, remove
   from tracking.

**Public API of orphan.go:**

- `util.OrphanedProcess` struct with `PID`, `Cmd`, `Age`, `TownRoot`
  (`orphan.go:408-414`).
- `util.ZombieProcess` struct with `PID`, `Cmd`, `Age`, `TTY`,
  `TownRoot` (`orphan.go:526-533`).
- `util.CleanupResult` struct wrapping `OrphanedProcess` with
  `Signal` (`"SIGTERM"`, `"SIGKILL"`, or `"UNKILLABLE"`) and `Error`
  (`orphan.go:519-524`).
- `util.FindOrphanedClaudeProcesses() ([]OrphanedProcess, error)`
  (`orphan.go:428-517`).
- `util.FindZombieClaudeProcesses() ([]ZombieProcess, error)`
  (`orphan.go:539-...`). Returns `(nil, nil)` safety early-exit when
  no tmux sessions exist (tmux probably down) rather than marking
  every Claude process as a zombie (`orphan.go:550-561`).
- `util.CleanupZombieClaudeProcesses() ([]ZombieCleanupResult, error)`
  (`orphan.go:667-758`).
- `util.CleanupOrphanedClaudeProcesses() ([]CleanupResult, error)`
  (`orphan.go:768-...`).
- State persistence helpers: `loadOrphanState`, `saveOrphanState`,
  `loadZombieState`, `saveZombieState` — used for the signal
  escalation state machine. Precise locations in-file; backing files
  live under `/tmp` or the town's runtime dir.

**orphan_windows.go** — stub implementations that return empty
results. The ps-tree/syscall.Kill approach is Unix-only.

### Internals / Notable implementation

- **`buildChildMap` is called once per cleanup cycle**
  (`orphan.go:27-48`) and shared across all tmux socket scans.
  Replaces per-PID `pgrep` calls, reducing O(N) process spawns to
  O(1) — an important optimization because the cleanup path runs
  inside a daemon patrol loop.
- Atomic-write temp file pattern (`atomic.go:65`) uses `base +
  ".tmp.*"` so `CreateTemp` generates a unique suffix per writer,
  eliminating race conditions between concurrent `AtomicWriteJSON`
  calls targeting the same destination (rare but possible under
  daemon + CLI concurrency).
- `FirstLine` is a single-purpose hack specifically for stripping
  cobra's multi-line usage-on-error output down to the real error —
  noted in the doc comment (`exec.go:13`).

## Related wiki pages

- [gt](../binaries/gt.md) — main binary; util is imported transitively
  almost everywhere.
- [gt doctor](../commands/doctor.md) — top-level consumer of the
  orphan and zombie cleanup functions.
- [gt cleanup](../commands/cleanup.md) — another orphan-cleanup surface.
- [internal/version](version.md) — uses `SetDetachedProcessGroup` for
  every git subprocess.
- [internal/session](session.md) — uses atomic writes for PID
  tracking and identity files.
- [internal/config](config.md) — uses atomic writes for every JSON
  config save.
- [go-packages inventory](../inventory/go-packages.md).

## Failure modes

### Precondition violations (what does it assume?)
- **Atomic rename cross-device:** `AtomicWriteFile` at
  `atomic.go:88-91` uses `os.Rename(tmpName, path)`. On POSIX, rename
  is atomic only within the same filesystem. If `path` and the temp
  file are on different mount points (e.g., `/tmp` vs. a network
  mount), `os.Rename` fails with `EXDEV`. **Present** — the error is
  returned and the temp file is cleaned up, but the error message is
  Go's raw `"invalid cross-device link"` with no explanation of what
  happened.
- **`getProcessCwd` requires proc or lsof:** `orphan.go:270-298` —
  on hardened Linux kernels (`kernel.yama.ptrace_scope=1`),
  `readlink(/proc/<pid>/cwd)` fails for non-descendant processes. The
  fallback is `lsof`, which may not be installed on minimal images.
  **Present** — if both methods fail, `""` is returned and the caller
  safely does not kill the process (fail-safe).

### Cross-platform concerns
- **Windows orphan/zombie cleanup is a no-op:**
  `orphan_windows.go` stubs all cleanup functions to return empty
  results. Windows users get no orphan detection or cleanup via
  `gt doctor` or the daemon patrol. **Untested** — the stubs exist
  but the feature is architecturally absent on Windows.
- **`SetProcessGroup` cancel hook is Unix-only:**
  `exec_unix.go:12-20` installs `cmd.Cancel` with
  `syscall.Kill(-pid, SIGKILL)` for process-group cleanup.
  `exec_windows.go:14-20` sets `CREATE_NEW_PROCESS_GROUP` but does
  not install a cancel hook, so context cancellation on Windows does
  not kill the process tree. **Untested** — the Windows shim exists
  but diverges in behavior.

## Outgoing calls

### Subprocess invocations
| Called binary | Command | Flags | Flag source | `file:line` |
|---|---|---|---|---|
| (dynamic) | (dynamic) | `cmd, args...` (Run) | caller | `exec.go:26` |
| (dynamic) | (dynamic) | `cmd, args...` (Output) | caller | `exec.go:48` |
| `ps` | `-eo pid,ppid` | (none) | hardcoded | `orphan.go:29` |
| `tmux` | `-S <socket> list-panes` | `-a -F #{pane_pid}` | runtime | `orphan.go:108` |
| `lsof` | `-a -p <pid>` | `-d cwd -Fn` | runtime | `orphan.go:287` |
| `ps` | `-p <pid>` | `-o args=` | runtime | `orphan.go:338` |
| `ps` | `-eo pid,tty,comm,etime` | (none) | hardcoded | `orphan.go:443` |
| `ps` | `-eo pid,tty,comm,etime` | (none) | hardcoded | `orphan.go:564` |
| `ps` | `-o tty=` | `-p <pid>` | runtime | `orphan.go:876` |

### File writes
| Target | What is written | Purpose | `file:line` |
|---|---|---|---|
| (atomic temp) | caller data | `AtomicWriteFile` helper | `atomic.go:58` |
| orphan report | report data | Orphan cleanup report | `orphan.go:230` |

## Notes / open questions

- The orphan-cleanup code lives in `util` but touches
  `internal/tmux`, `internal/lock`, and platform-specific syscalls —
  it's arguably its own package waiting to be extracted. Flagged here
  as a neutral observation, not a recommendation.
- State files for the signal-escalation ratchet persist across daemon
  restarts. Their exact path wasn't captured in this pass (they're
  read/written by `loadOrphanState`/`saveOrphanState` whose bodies
  were not read). Future pass: trace those helpers for the on-disk
  layout and lifecycle.
- `ExpandHome` only handles leading `~/`; it does not handle
  `~user/` (per-user expansion) or `~` alone. Most callers just want
  the former, but anything surfacing user-entered paths through this
  helper will silently leave `~user/...` unexpanded.
- `RedactURL`'s fallback for unparseable URLs with an `@` is
  aggressive (`"<invalid URL>"`) — intentional to avoid leaking
  embedded secrets. Worth noting for anyone who greps logs expecting
  to find original URLs.
