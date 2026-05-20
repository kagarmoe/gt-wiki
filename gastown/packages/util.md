---
title: internal/util
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-05-20
sources:
  - /home/kimberly/repos/gastown/internal/util/exec.go
  - /home/kimberly/repos/gastown/internal/util/exec_unix.go
  - /home/kimberly/repos/gastown/internal/util/exec_windows.go
  - /home/kimberly/repos/gastown/internal/util/path.go
  - /home/kimberly/repos/gastown/internal/util/slice.go
  - /home/kimberly/repos/gastown/internal/util/url.go
  - /home/kimberly/repos/gastown/internal/util/orphan.go
  - /home/kimberly/repos/gastown/internal/util/orphan_windows.go
  - /home/kimberly/repos/gastown/internal/util/diskspace.go
  - /home/kimberly/repos/gastown/internal/util/diskspace_unix.go
  - /home/kimberly/repos/gastown/internal/util/diskspace_windows.go
tags: [package, platform-service, util, atomic-write, exec, process-groups, orphan-cleanup, redaction, disk-space]
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

Grab-bag of low-level helpers: subprocess wrappers that set
platform-specific process groups (for Windows no-flash and Unix
kill-the-tree semantics), home-directory tilde expansion, slice helpers,
URL credential redaction, filesystem free-space queries, and — by far
the largest single member — the orphan/zombie Claude process hunter used
by `gt doctor` and daemon cleanup patrols. The atomic file-write
primitives that used to live here (`atomic.go`) moved out to the
dependency-free [internal/atomicfile](atomicfile.md) leaf package in
v1.0.1; see the [atomic.go section](#atomicgo--moved-to-internalatomicfile-v101) below.

**Go package path:** `github.com/steveyegge/gastown/internal/util`
**File count:** 11 go files (`exec.go`, `exec_unix.go`,
`exec_windows.go`, `orphan.go`, `orphan_windows.go`, `path.go`,
`slice.go`, `url.go`, `diskspace.go`, `diskspace_unix.go`,
`diskspace_windows.go`), 6 test files. `atomic.go` was removed in
v1.0.1; `diskspace*.go` were added.
**Imports (notable):** stdlib (`os`, `os/exec`, `syscall`, `unsafe`
(Windows disk-space only), `path/filepath`, `net/url`, `regexp`,
`strconv`, `time`, `fmt`), and
— inside `orphan.go` only — [`internal/tmux`](../inventory/go-packages.md)
and `internal/lock` for cross-socket pane-PID discovery.
**Imported by (notable):** nearly everywhere. The top consumers are:

- [`internal/version`](version.md) — every git subprocess goes through
  `SetDetachedProcessGroup`.
- [`internal/session`](session.md) — uses atomic writes for PID
  tracking and identity files.
- [`internal/config`](config.md) — previously used `AtomicWriteJSON`/
  `EnsureDirAndWriteJSON` for saving JSON config; as of v1.0.1 those
  calls go to [internal/atomicfile](atomicfile.md), not this package.
- [`gt doctor`](../commands/doctor.md), [`gt cleanup`](../commands/cleanup.md)
  — call the orphan/zombie cleanup functions; `gt doctor`'s
  `DiskSpaceCheck` also calls `CheckDiskSpace`.
- Generally, any command page that shells out to git, ps, or another
  CLI binary lands here for the process-group wrapper.

## What it actually does

### atomic.go — moved to internal/atomicfile (v1.0.1)

`internal/util/atomic.go` was **removed in v1.0.1**. Its atomic
file-write primitives (`AtomicWriteFile`, `AtomicWriteJSON`,
`AtomicWriteJSONWithPerm`, `EnsureDirAndWriteJSON`,
`EnsureDirAndWriteJSONWithPerm`) moved to the new dependency-free
[internal/atomicfile](atomicfile.md) leaf package and were renamed
(`AtomicWriteJSON` → `atomicfile.WriteJSON`, `AtomicWriteFile` →
`atomicfile.WriteFile`; the two `EnsureDir*` names were kept). The move
exists so that low-level packages which `internal/util` transitively
depends on can still write atomically without an import cycle — see the
[atomicfile page](atomicfile.md#relationship-to-internalutil-the-move)
for the full old→new name table and rationale. Callers that used to
reach into `util` for atomic writes (config, daemon, doctor, polecat,
quota, …) now import `atomicfile` directly.

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

### diskspace.go / diskspace_unix.go / diskspace_windows.go — filesystem free space

New in v1.0.1. A small cross-platform free-space query plus a
threshold-based level classifier, added to give
[`gt doctor`](../commands/doctor.md)'s `DiskSpaceCheck` (and any other
caller) a single place to decide "is the disk dangerously full?". Disk
exhaustion is called out elsewhere as the root cause of cascading
failures (Dolt data loss, polecat session death, lost commits).

- `DiskSpaceInfo` struct (`diskspace.go:8-20`) — `AvailableBytes`,
  `TotalBytes`, `UsedBytes`, `UsedPercent`. Methods `AvailableMB()`
  (`diskspace.go:23`), `AvailableGB()` (`diskspace.go:28`),
  `AvailableHuman()` (`diskspace.go:33`).
- `FormatBytesHuman(bytes uint64) string` (`diskspace.go:38-58`) —
  renders bytes as `B`/`KB`/`MB`/`GB`/`TB` with one decimal.
- Threshold constants (`diskspace.go:60-73`): `DiskSpaceMinimumMB = 500`
  (hard floor — block disk-heavy operations below this),
  `DiskSpaceWarningMB = 1024` (warn / shed load), and
  `DiskSpaceCriticalPercent = 95.0` (block regardless of absolute free
  bytes once usage crosses 95%).
- `DiskSpaceLevel` enum (`diskspace.go:76-85`): `DiskSpaceOK |
  DiskSpaceWarning | DiskSpaceCritical`, with a `String()` method
  (`diskspace.go:88-100`).
- `CheckDiskSpace(path) (DiskSpaceLevel, string, error)`
  (`diskspace.go:103-126`) — calls `GetDiskSpace`, then returns
  `DiskSpaceCritical` if available is below the minimum **or** usage is
  at/above the critical percent, `DiskSpaceWarning` below the warning
  threshold, else `DiskSpaceOK` with an empty message. A `GetDiskSpace`
  error is returned as `DiskSpaceOK` with the error (fail-open: a stat
  failure does not by itself block operations).
- `GetDiskSpace(path) (*DiskSpaceInfo, error)` is the platform split:
  Unix uses `syscall.Statfs` (`diskspace_unix.go:11-33`, build tag
  `!windows`); Windows calls `GetDiskFreeSpaceExW` via
  `syscall.NewLazyDLL` + `unsafe.Pointer`
  (`diskspace_windows.go:12-41`, build tag `windows`). Unlike the orphan
  cleanup below, the disk-space query has a real Windows implementation,
  not a stub.

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
   whose `ps` `comm` field is a tracked agent runtime. As of v1.0.1 the
   match set is centralised in `isAgentOrphanCommName`
   (`orphan.go:410-419`) and broadened beyond the original
   `claude`/`claude-code`/`codex`/`opencode` to also cover
   `cursor-agent`, `agent`, and `copilot` — keeping orphan detection in
   step with the `internal/config` agent presets. Both
   `FindOrphanedClaudeProcesses` and `FindZombieClaudeProcesses` call it.
   These are the primary Task-tool leak case.
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
- The atomic-write temp-file pattern (`base + ".tmp.*"` so `CreateTemp`
  generates a unique per-writer suffix) now lives in
  [internal/atomicfile](atomicfile.md), not this package — it moved out
  of `util` in v1.0.1.
- `FirstLine` is a single-purpose hack specifically for stripping
  cobra's multi-line usage-on-error output down to the real error —
  noted in the doc comment (`exec.go:13`).

## Related wiki pages

- [gt](../binaries/gt.md) — main binary; util is imported transitively
  almost everywhere.
- [gt doctor](../commands/doctor.md) — top-level consumer of the
  orphan and zombie cleanup functions.
- [gt cleanup](../commands/cleanup.md) — another orphan-cleanup surface.
- [internal/atomicfile](atomicfile.md) — new in v1.0.1; the leaf
  package that now owns the atomic file-write primitives `atomic.go`
  used to provide. The other half of the v1.0.1 split.
- [internal/doctor](doctor.md) — its `DiskSpaceCheck` wraps
  `CheckDiskSpace`, and its orphan/zombie checks call the cleanup
  functions here; the doctor page forward-links here.
- [internal/version](version.md) — uses `SetDetachedProcessGroup` for
  every git subprocess.
- [internal/session](session.md) — uses `SetDetachedProcessGroup`; its
  atomic writes for PID/identity files now go through
  [internal/atomicfile](atomicfile.md).
- [internal/config](config.md) — its JSON-config atomic writes now go
  through [internal/atomicfile](atomicfile.md), not this package
  (v1.0.1).
- [go-packages inventory](../inventory/go-packages.md).

## Failure modes

### Precondition violations (what does it assume?)
- **Atomic rename cross-device:** the `AtomicWriteFile` cross-device
  (`EXDEV`) failure mode previously documented here moved with the code
  to [internal/atomicfile](atomicfile.md) in v1.0.1 — and the move
  largely defused it, since `atomicfile.WriteFile` now creates the temp
  file in the **target's own directory** rather than a shared temp dir,
  keeping the rename within one filesystem. See that page's Failure
  modes section.
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
| orphan report | report data | Orphan cleanup report | `orphan.go:230` |

(The former `AtomicWriteFile` temp-file write moved to
[internal/atomicfile](atomicfile.md) in v1.0.1.)

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
