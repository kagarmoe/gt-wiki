---
title: internal/estop
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/estop/estop.go
tags: [package, estop, safety, sentinel-file, town-wide]
---

# internal/estop

Emergency-stop primitives for Gas Town. An E-stop is a town-wide (or
per-rig) kill switch that freezes agent work: when a sentinel file exists
at the town root, all non-Mayor agents should be paused (SIGTSTP) and the
daemon should not restart them. This package owns only the file shape and
the read/write primitives — the actual freezing, scanning, and
coordination lives in the daemon/supervisor and the
[`gt estop`](../commands/estop.md) / [`gt thaw`](../commands/thaw.md)
command implementations that sit on top.

**Go package path:** `github.com/steveyegge/gastown/internal/estop`
**File count:** 1 go file, 1 test file
**Imports (notable):** stdlib only (`fmt`, `os`, `path/filepath`,
`strings`, `time`). Zero gastown-internal imports — this is a leaf
package by design so that anything (daemon, CLI, hooks) can consult the
E-stop state without pulling in larger subsystems.
**Imported by (notable):** the [`gt estop`](../commands/estop.md) and
[`gt thaw`](../commands/thaw.md) commands, the daemon/supervisor scan
loop that respects E-stop before restarting agents, and health-check
paths that surface E-stop status to the user.

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/estop/estop.go`.

### Sentinel file shape

- `FileName = "ESTOP"` (`estop.go:21`) — the town-wide sentinel at the
  town root.
- Per-rig sentinels are named `ESTOP.<rigName>` via
  `RigFileName(rigName)` (`estop.go:80-82`) and live at the same town-root
  level, not inside the rig directory.
- File body format (`estop.go:58-60`): three tab-separated fields on a
  single line — `trigger\ttimestamp\treason\n` — where `trigger` is
  `"manual"` or `"auto"`, `timestamp` is RFC3339, and `reason` is a
  free-form human-readable string.
- `parse` (`estop.go:125-148`) is tolerant: an empty file becomes
  `&Info{Trigger: TriggerManual, Timestamp: time.Now()}`, and a malformed
  timestamp is silently dropped (the `Info.Timestamp` stays zero) rather
  than failing the entire read.

### Trigger constants (`estop.go:23-27`)

- `TriggerManual = "manual"` — a human (or the Mayor) explicitly hit the
  button.
- `TriggerAuto   = "auto"` — an automated guard (e.g., Witness) tripped
  the stop.

The manual/auto split is load-bearing in `Deactivate` below.

### Town-wide API

- `FilePath(townRoot) string` (`estop.go:37-39`)
- `IsActive(townRoot) bool` (`estop.go:42-45`) — existence check on the
  sentinel, swallowing the error into a bool.
- `Read(townRoot) *Info` (`estop.go:48-54`) — returns `nil` on any read
  error, including not-found. Callers need to check both `IsActive` and
  `Read != nil` if they want to distinguish the two.
- `Activate(townRoot, trigger, reason) error` (`estop.go:57-61`) — writes
  the sentinel at mode 0644.
- `Deactivate(townRoot, onlyAuto) error` (`estop.go:65-77`) — removes the
  sentinel. When `onlyAuto` is true, it first reads the file and returns
  an explicit error (`"E-stop was manually triggered — use 'gt thaw' to
  clear"`, `estop.go:69`) if the trigger is `manual`. This is the
  protection that stops an auto-unfreeze path from silently clearing a
  manual stop.

### Per-rig API (`estop.go:79-118`)

Mirrors the town-wide API but keyed on `rigName`:

- `RigFileName`, `RigFilePath`, `IsRigActive`, `ReadRig`, `ActivateRig`,
  `DeactivateRig`.
- `DeactivateRig` (`estop.go:112-118`) does **not** have an `onlyAuto`
  guard — per-rig E-stops can be cleared unconditionally. This is an
  asymmetry with the town-wide API.

### Combined check

- `IsAnyActive(townRoot, rigName)` (`estop.go:121-123`) returns true if
  either the town-wide or the rig-specific sentinel is present. This is
  the normal check used by callers that want "should this rig be frozen
  right now?" without having to OR the two themselves.

## Docs claim

The package comment (`estop.go:1-10`) describes the E-stop as a town-wide
pause mechanism, names the ESTOP sentinel file, states that "The Mayor is
exempt from E-stop so it can coordinate recovery", and credits the
original implementation to outdoorsea (PR #3237). The Mayor-exemption is
**not enforced in this package** — the package doesn't know who the
caller is. Enforcement lives in the freeze code that consumes `IsActive`.

## Drift

- **Per-rig vs. town-wide asymmetry on `Deactivate`.** Town-wide requires
  `onlyAuto=false` to clear a manual stop, pointing users at `gt thaw`
  (`estop.go:69`). Per-rig has no equivalent guard; any caller of
  `DeactivateRig` can clear a rig-specific stop regardless of whether it
  was manual. If the human UX treats per-rig stops as first-class, this
  may be a hole worth closing.
- **Mayor-exemption is an invariant, not an implementation.** The package
  comment asserts it, but no code in this package distinguishes the
  Mayor from any other caller. Any code path that reads `IsActive` and
  acts on it (freeze/pause) is responsible for respecting the exemption
  — easy to get wrong in a new consumer.

## Notes / open questions

- `Info.Reason` is a single-line free-form string; there is no escaping
  for tabs or newlines, which would corrupt the three-field split
  (`estop.go:59`, `132`). Callers writing reasons from user input should
  strip or replace those characters.
- RFC3339 parsing is lossy: a malformed timestamp silently becomes zero,
  which `Read` callers may display as "1 Jan 0001". No warning is
  surfaced.

## Related wiki pages

- [gt](../binaries/gt.md) — top-level binary.
- [gt estop](../commands/estop.md) — user-facing freeze command (the
  primary caller of `Activate`).
- [gt thaw](../commands/thaw.md) — user-facing unfreeze command (the
  primary caller of `Deactivate(townRoot, false)`).
- [go-packages inventory](../inventory/go-packages.md).
