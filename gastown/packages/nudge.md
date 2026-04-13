---
title: internal/nudge
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/nudge/queue.go
  - /home/kimberly/repos/gastown/internal/nudge/poller.go
  - /home/kimberly/repos/gastown/internal/nudge/poller_process_unix.go
  - /home/kimberly/repos/gastown/internal/nudge/poller_process_windows.go
  - /home/kimberly/repos/gastown/internal/nudge/procattr_unix.go
  - /home/kimberly/repos/gastown/internal/nudge/procattr_windows.go
tags: [package, data-layer, nudge, ephemeral-messaging, filesystem-queue, poller, fsnotify]
---

# internal/nudge

Filesystem-backed, non-destructive nudge queue. "Nudge" is Gas Town's
**ephemeral** agent-to-agent messaging mechanism: a short note dropped into
a per-session queue directory, picked up at the next turn boundary, and
injected into the agent's context as a `<system-reminder>`. Unlike
[`internal/mail`](mail.md), nudges create **no persistent record** — they
never become beads, never hit Dolt, and vanish once delivered or expired.

**Go package path:** `github.com/steveyegge/gastown/internal/nudge`
**File count:** 6 non-test go files (queue.go 358, poller.go 276, two
platform-specific process shims, two procattr shims), 2 test files.
**Imports (notable):** `github.com/fsnotify/fsnotify` (poller.go's
Watcher), [`internal/config`](config.md) (operational thresholds),
`internal/constants` (`DirRuntime` path), [`internal/util`](util.md)
(`SetDetachedProcessGroup`). Stdlib: `crypto/rand`, `encoding/json`,
`os`, `os/exec`, `path/filepath`, `sort`, `strings`, `sync`, `syscall`,
`time`.
**Imported by (notable):** [`gt nudge`](../commands/nudge.md) (enqueues),
[`gt nudge-poller`](../commands/nudge-poller.md) (runs the drain loop
for non-Claude agents), `internal/cmd/hook.go` (UserPromptSubmit hook
that drains at Claude-Code turn boundaries), `internal/cmd/mail_check.go`
(mail delivery also drops nudges for notification), the mail
[`Router.notifyRecipient`](mail.md) which converts mail arrivals to
nudge drops, `internal/acp/propulsion.go` (ACP Propeller uses the
`Watcher`), `internal/crew/manager.go` and the witness/refinery managers
that start pollers when workers come online, and `internal/cmd/sling.go`
/ `unsling.go` which manage the poller around worker lifecycle.

## What it actually does

### The queue (queue.go)

The core data structure is a **directory per session**, one JSON file per
pending nudge, filename-ordered for FIFO delivery. Queue path format:
`<townRoot>/.runtime/nudge_queue/<session>/` (`queue.go:76-79`), where
`<session>` is the tmux session name with `/` → `_` sanitisation.

**Types:**

- `QueuedNudge` struct (`queue.go:59-71`): `Sender`, `Message`, `Priority`,
  `Kind` (optional kind tag like `mail`, `escalation`), `ThreadID`,
  `Severity`, `Timestamp`, `ExpiresAt`, `DeliverAfter` (deferred delivery
  — Drain skips-but-keeps until the deadline passes).

**Priority constants** (`queue.go:28-33`): `PriorityNormal` (next
turn-boundary), `PriorityUrgent` (delivered with a stronger
system-reminder wrapper, TTL 2h).

**TTL constants** (`queue.go:38-51`): `DefaultNormalTTL = 30*time.Minute`,
`DefaultUrgentTTL = 2*time.Hour`, `MaxQueueDepth = 50`,
`staleClaimThreshold = 5*time.Minute`. All three TTLs are overridable via
`operational.nudge` in `settings/config.json` (the `nudgeConfig` helper
at `queue.go:54-56` pulls from `config.LoadOperationalConfig`).

**Public API:**

- `Enqueue(townRoot, session string, nudge QueuedNudge) error`
  (`queue.go:92-138`) — writes one `<nanotimestamp>-<randsuffix>.json`
  file after checking queue depth against `MaxQueueDepthV()` from
  operational config. Fills in defaults for `Timestamp`, `Priority`, and
  `ExpiresAt` if not set.
- `Requeue(townRoot, session string, nudges []QueuedNudge) error`
  (`queue.go:143-153`) — re-enqueues previously drained nudges that
  weren't actually delivered (e.g., ACP watcher saw them but the agent
  was busy). Skips any whose `ExpiresAt` has passed.
- `Drain(townRoot, session string) ([]QueuedNudge, error)`
  (`queue.go:164-279`) — the hook-side consumer. Reads the queue dir,
  sorts by filename (FIFO), and **atomically claims** each `.json` file
  by renaming it to `<name>.claimed.<randsuffix>`. The unique suffix is
  load-bearing on Windows (`queue.go:218-225`): `os.Rename` to a shared
  destination is not atomic on Windows, so two goroutines could both
  "succeed" via `MOVEFILE_REPLACE_EXISTING`, causing loss. Orphan claims
  older than `staleClaimThreshold` are renamed back to `.json` for
  retry — **not** deleted, because deletion would permanently drop a
  nudge whose drainer merely crashed.
- `Pending(townRoot, session string) (int, error)` /
  `QueueLen(townRoot, session string) int` (`queue.go:283-314`) —
  approximate count of `.json` files; `QueueLen` silences missing-dir
  errors since that's the common "no queue yet" case.
- `FormatForInjection(nudges []QueuedNudge) string` (`queue.go:318-358`)
  — formats a batch as a `<system-reminder>` block for injection into
  Claude Code's hook output. Splits urgent from normal, emits different
  trailing directives ("Handle urgent nudges before continuing current
  work." vs. "This is a background notification. Continue current work
  unless the nudge is higher priority.").

### The poller (poller.go)

Not every agent runtime has a hook that fires at turn boundaries. Claude
Code has `UserPromptSubmit`, which calls `Drain` inline (`cmd/hook.go`).
Other runtimes (Gemini, Codex, Cursor, ACP-driven agents) do not. For
them, a **background process polls the queue** and injects nudges via
`tmux send-keys` wrapped in the `NudgeSession` helper.

**Poller tuning** (`poller.go:32-38`):
`DefaultPollInterval = "10s"`, `DefaultIdleTimeout = "3s"`.

**Public API:**

- `StartPoller(townRoot, session string) (int, error)`
  (`poller.go:54-90`) — spawns a detached `gt nudge-poller <session>`
  child via `os/exec`, using `util.SetDetachedProcessGroup(cmd)` so the
  poller survives the caller's exit. Writes
  `<townRoot>/.runtime/nudge_poller/<session>.pid`. If a poller is
  already running for the session (checked via `pollerAlive`), returns
  the existing PID instead of starting a duplicate.
- `StopPoller(townRoot, session string) error` (`poller.go:102-139`) —
  reads the PID file, SIGTERMs the process, removes the PID file.
  Tolerates missing / corrupt PID files and dead processes.
- `pollerAlive(townRoot, session string) (int, bool)`
  (`poller.go:143-163`) — checks PID file + signal(0) liveness, cleans
  up stale PID files.

**The Watcher (fsnotify-based, ACP-friendly):**

- `Watcher` struct + `NewWatcher(townRoot, session string) (*Watcher,
  error)` (`poller.go:168-197`) — uses `fsnotify` to watch the queue
  directory for `Create`/`Write` events. Coalesces bursts with a
  100ms timer (`poller.go:236-267`) and surfaces a single signal on the
  `Events()` channel so subscribers can re-drain without polling.
  This is the preferred path for long-running watchers like ACP
  Propeller.
- `Events() <-chan struct{}`, `Close() error`, `WatcherForSession`
  (`poller.go:201-276`).

### Platform shims

- `poller_process_unix.go` / `poller_process_windows.go`
  (`pollerProcessAlive(pid int) bool`) — isolates the Unix
  `signal(0)` check vs. the Windows handle-based check so the rest of
  the package is cross-platform.
- `procattr_unix.go` / `procattr_windows.go` — provides
  `detachedProcAttr()`, `isProcessAlive(*os.Process) bool`, and
  `terminateProcess(*os.Process) error`. On Unix these map to
  `Setpgid: true`, `Signal(0)`, and `SIGTERM` (`procattr_unix.go:11-26`).

### Delivery modes — package-level view

The `gt nudge` command exposes "immediate", "wait-idle", and "queue"
modes at the CLI, but from this package's perspective:

- **Queue mode** = the normal `Enqueue` path: write a JSON file and
  return. Drain happens later, by hook or by poller, at a turn
  boundary when the agent is idle.
- **Wait-idle mode** is implemented at the CLI level by polling
  `NudgeSession` tmux idleness before calling `Enqueue` — the
  `nudge` package doesn't itself do the wait.
- **Immediate mode** bypasses the queue and uses tmux `send-keys`
  directly; it is not a `nudge` package operation.

### Usage pattern

```go
// Sender:
nudge.Enqueue(townRoot, "gastown-Toast", nudge.QueuedNudge{
    Sender:   "gastown/max",
    Message:  "Can you review PR #42?",
    Priority: nudge.PriorityNormal,
})

// Receiver (Claude Code UserPromptSubmit hook):
nudges, _ := nudge.Drain(townRoot, session)
if len(nudges) > 0 {
    fmt.Println(nudge.FormatForInjection(nudges))
}
```

## Related wiki pages

- [gt](../binaries/gt.md)
- [gt nudge](../commands/nudge.md) — producer CLI.
- [gt nudge-poller](../commands/nudge-poller.md) — the long-running
  process spawned by `StartPoller`.
- [internal/mail](mail.md) — the **durable** messaging counterpart.
  Nudge is fire-and-forget; mail is commit-log backed. `Router.notifyRecipient`
  in mail drops a nudge as the arrival notification.
- [internal/config](config.md) — `operational.nudge` thresholds
  (`DefaultNormalTTL`, `DefaultUrgentTTL`, `MaxQueueDepth`,
  `staleClaimThreshold`) resolve through here.
- [internal/util](util.md) — `SetDetachedProcessGroup` used by
  `StartPoller`.
- [internal/events](events.md) — `TypeNudge` audit events are emitted
  by callers of this package, not by the package itself.
- [internal/lock](lock.md) — nudge does **not** use the internal lock
  helper; its own claim mechanism uses rename-based locks. Noted here
  so future readers don't expect cross-use.
- [go.mod](../files/go-mod.md) — `github.com/fsnotify/fsnotify`.
- [go-packages inventory](../inventory/go-packages.md)

## Notes / open questions

- **The rename-then-process claim protocol is the subtle heart of the
  package.** Each `Drain` caller atomically claims a `.json` file by
  renaming it to a unique `.claimed.<suffix>` name, then processes it.
  If the drainer crashes mid-process, the orphan-claim sweep at
  `queue.go:182-202` restores the file to `.json` after
  `staleClaimThreshold` so it's retried rather than lost. This is
  implemented without flock because only one process per session is
  expected to call `Drain` at a time — the unique-suffix trick is the
  safety net against Windows non-atomicity.
- The queue is **not** a transactional durable store. Crashes between
  `Enqueue` writing the file and the agent draining it are fine
  (file survives); crashes mid-drain are handled by the orphan sweep.
  But the filesystem itself is trusted — there's no journaling or
  checksumming.
- `FormatForInjection` hardcodes English text directives. If Gas Town
  ever becomes multilingual, the strings at `queue.go:336-354` would
  need to move to a resource bundle.
- The `DeliverAfter` field is newer than `ExpiresAt` and enables
  scheduled-for-later nudges (e.g., "nudge me in 30 minutes if I
  haven't responded"). `Drain` quietly unclaims them instead of
  draining so they stay visible to future calls.
- There is no cross-session "broadcast nudge" in this package — one
  session at a time. Broadcast is a mail-level concept
  ([`internal/mail`](mail.md) channels and announce).
