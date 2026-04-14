---
title: gt scheduler
type: command
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/cmd/scheduler.go
  - /home/kimberly/repos/gastown/internal/scheduler/capacity/
tags: [command, work, scheduler, dispatch, capacity, polecat, daemon]
---

# gt scheduler

Manage the capacity-controlled dispatch scheduler — the daemon-facing
machinery that defers sling requests when running too many polecats.
Queries scheduler state (active polecats, paused flag, queue), lists
scheduled beads, pauses/resumes dispatch town-wide, clears queued
work, and manually triggers dispatch.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupWork` ("Work Management") (`scheduler.go:29`)
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/scheduler.go`
(558 lines). Delegates state persistence to
`internal/scheduler/capacity` and shares helpers with [sling](sling.md)
(which writes "sling context" beads that the scheduler consumes).

### Invocation

```
gt scheduler <subcommand> [flags]
```

`schedulerCmd.RunE = requireSubcommand` (`scheduler.go:44`) — bare
`gt scheduler` errors.

### Subcommands

Registered at `scheduler.go:110-115`:

| subcommand | source | one-liner |
|---|---|---|
| `status` | `:47-51`, `runSchedulerStatus :129-194` | Show scheduler state: paused, queued total/ready, active polecats, last dispatch timestamp. |
| `list` | `:53-57`, `runSchedulerList :196-238` | List scheduled beads grouped by target rig, with blocked indicator. |
| `pause` | `:59-63`, `runSchedulerPause :240-264` | Set the `paused` flag town-wide (writes to `capacity.State`). |
| `resume` | `:65-69`, `runSchedulerResume :266-289` | Clear the `paused` flag. |
| `clear` | `:71-79`, `runSchedulerClear :291-352` | Remove beads from scheduler by closing their sling context beads. With `--bead`, scoped to one bead; without, clears ALL. |
| `run` | `:81-93`, `runSchedulerRun :354-362` | Manually trigger dispatch using the same logic as the daemon heartbeat. `--dry-run` previews; `--batch` overrides batch size. |

### Architecture

**Scheduler state** lives in `capacity.State` (`scheduler.go:135,
246, 272`), loaded and saved via `capacity.LoadState(townRoot)` /
`capacity.SaveState(townRoot, state)`. Fields referenced in this
file: `Paused`, `PausedBy`, `LastDispatchAt`, `LastDispatchCount`.

**Scheduled beads** are not stored in a scheduler table — they are
sling context beads (created by [sling](sling.md)) living in each
rig's beads database. `listScheduledBeads` (`:367-431`) reconciles
them by:

1. `listAllSlingContexts(townRoot)` — scan all rig beads dirs for
   sling context beads.
2. `beads.ParseSlingContextFields(ctx.Description)` — extract
   `WorkBeadID`, `TargetRig`, `DispatchFailures`.
3. Skip circuit-broken contexts where `DispatchFailures >=
   maxDispatchFailures` (`:399`).
4. Dedup by `WorkBeadID` (`:403-407`).
5. `batchFetchBeadInfoByIDs` (`:388`) — resolve current
   title/status for each work bead.
6. Skip hooked/closed/tombstoned work beads (`:416-418`).
7. Mark `Blocked: !readyWorkIDs[fields.WorkBeadID]` (`:426`).

**`beadsSearchDirs`** (`:458-483`) enumerates dirs to scan: the
town root plus any first-level subdir with a `.beads/` (excluding
dotfiles, `mayor`, `settings`), plus `rig/mayor/rig/.beads` if
present.

### Active-polecat counting

Two helpers count active polecats — subtly different:

- `countActivePolecats` (`:489-510`): counts *all* polecat tmux
  sessions, including idle ones. Parses `tmux list-sessions` output,
  extracts identities via `session.ParseSessionName`, keeps only
  `session.RolePolecat`.
- `countWorkingPolecats` (`:516-558`): counts polecats that have
  non-null `hook_bead` on their agent bead (i.e. actively working).
  For **capacity gating**, this is the right count — idle polecats
  are available for re-sling under the persistent polecat model.

### Flags

All attached to specific subcommands, not the parent:

| flag | on | default | notes |
|---|---|---|---|
| `--json` | `status`, `list` | `false` | Indented JSON output. |
| `--bead <id>` | `clear` | `""` | Narrow `clear` to one work bead. |
| `--batch <n>` | `run` | `0` | Override dispatch batch size. `0` = use config. |
| `--dry-run` | `run` | `false` | Preview what would dispatch. |

### Config knobs (referenced in Long text)

From `scheduler.go:41-43`:

- `scheduler.max_polecats` — via `gt config set`. `-1` (default)
  means direct dispatch, no deferral. Positive N enables capacity-
  gated deferred dispatch.

See [config](config.md) for the broader config surface.

### Data shape

```go
// scheduler.go:121-128
type scheduledBeadInfo struct {
    ID        string
    Title     string
    Status    string
    TargetRig string
    Blocked   bool
}
```

## Notes / open questions

- **Scheduler ≠ cron.** Despite the name, this is a *dispatch*
  scheduler — gating how many polecats spawn concurrently. It is
  not a time-based cron. Work items don't have a "run at" time;
  they have a "can we afford another polecat right now?" gate.
- **Sling contexts are the queue.** There is no dedicated table:
  the queue is synthesized at read time by scanning all rig beads
  dirs for sling context beads and reconciling against current
  work-bead status. This is robust (no drift between queue and
  reality) but expensive at scale.
- **Cross-rig dispatch coupling.** `gt-u6n6a` is referenced in
  [sling](sling.md) as the rationale for disabling
  `BD_DOLT_AUTO_COMMIT` during batch slinging. Any scheduler
  run via the daemon inherits that concurrency risk; `gt scheduler
  run` too.
- **`scheduler.go:299-326`** lives in `runSchedulerClear` and
  deals with GH#3468 — duplicate contexts from concurrent
  `scheduleBead` calls racing past idempotency. Scanning all rig
  dirs is how the cleanup path finds them.
- **Related commands.** [sling](sling.md) (producer),
  [convoy](convoy.md) (the other dispatch surface — wave-based),
  [mountain](mountain.md) (convoy + enhanced monitoring),
  [ready](ready.md) (town-wide readiness view used by the
  dispatcher).
- **Related package.** [../packages/scheduler.md](../packages/scheduler.md)
  — pure scheduling functions live in the `capacity/` subpackage
  (the `internal/scheduler` namespace itself has 0 Go files).
