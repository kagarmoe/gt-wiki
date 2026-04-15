---
title: gt refinery
type: command
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-15
sources:
  - /home/kimberly/repos/gastown/internal/cmd/refinery.go
tags: [command, agents, refinery, merge-queue, per-rig, mq, lifecycle]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
---

# gt refinery

Lifecycle and queue-inspection commands for the
[Refinery](../roles/refinery.md) — the per-rig merge queue
processor. One Refinery per [rig](../concepts/rig.md). This page
documents the `gt refinery` CLI subcommands only; see the
[Refinery role page](../roles/refinery.md) for conflict handling,
interaction with polecats/mayor, and the merge protocol.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupAgents` ("Agent Management") (`refinery.go:31`)
**Polecat-safe:** no
**Beads-exempt:** yes (listed in `beadsExemptCommands` on
`/home/kimberly/repos/gastown/internal/cmd/root.go:44-77`)
**Branch-check-exempt:** no
**Alias:** `gt ref`

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/refinery.go`
(859 lines). Registration at `refinery.go:229-270` wires flags
and attaches eleven subcommands to `refineryCmd`, then registers
the parent on `rootCmd`.

The parent `refineryCmd` (`refinery.go:28-49`) uses
`RunE: requireSubcommand`.

### Invocation

```
gt refinery start    [rig] [--agent <a>] [--foreground]
gt refinery stop     [rig]
gt refinery restart  [rig] [--agent <a>]
gt refinery attach   [rig] [--agent <a>]
gt refinery status   [rig] [--json]
gt refinery queue    [rig] [--json]
gt refinery ready    [rig] [--json] [--all]
gt refinery blocked  [rig] [--json]
gt refinery unclaimed [rig] [--json]
gt refinery claim    <mr-id>
gt refinery release  <mr-id>
```

All rig-scoped subcommands accept the rig name as a positional OR
infer it from cwd. `claim` and `release` always infer from cwd
(`refinery.go:573-580, 600-607`).

Role-shortcut note at `refinery.go:48`: `"refinery"` in
mail/nudge addresses resolves to this rig's Refinery.

### Long help (role framing)

`refinery.go:34-48` is the canonical CLI-level summary:

> The Refinery serializes all merges to main for a rig:
>   - Receives MRs submitted by polecats (via gt done)
>   - Rebases work branches onto latest main
>   - Runs validation (tests, builds, checks)
>   - Merges to main when clear
>   - If conflict: spawns FRESH polecat to re-implement
>     (original is gone)
>
> Work flows: Polecat completes → gt done → MR in queue →
> Refinery merges. The polecat is already nuked by the time the
> Refinery processes.

### Subcommands

Lifecycle:

| subcommand | var | source | run-fn |
|---|---|---|---|
| `start`  (alias `spawn`) | `refineryStartCmd`  | `refinery.go:51-68`  | `runRefineryStart`  (`refinery.go:296-329`) |
| `stop`   | `refineryStopCmd`   | `refinery.go:70-79`  | `runRefineryStop`   (`refinery.go:331-352`) |
| `restart`| `refineryRestartCmd`| `refinery.go:120-133`| `runRefineryRestart`(`refinery.go:528-558`) |
| `status` | `refineryStatusCmd` | `refinery.go:81-90`  | `runRefineryStatus` (`refinery.go:362-411`) |
| `attach` | `refineryAttachCmd` | `refinery.go:103-118`| `runRefineryAttach` (`refinery.go:494-526`) |

Queue inspection:

| subcommand | var | source | run-fn |
|---|---|---|---|
| `queue`     | `refineryQueueCmd`     | `refinery.go:92-101`  | `runRefineryQueue`     (`refinery.go:413-492`) |
| `ready`     | `refineryReadyCmd`     | `refinery.go:186-207` | `runRefineryReady`     (`refinery.go:689-762`) |
| `blocked`   | `refineryBlockedCmd`   | `refinery.go:212-225` | `runRefineryBlocked`   (`refinery.go:814-859`) |
| `unclaimed` | `refineryUnclaimedCmd` | `refinery.go:168-182` | `runRefineryUnclaimed` (`refinery.go:623-687`) |

Parallel-worker coordination:

| subcommand | var | source | run-fn |
|---|---|---|---|
| `claim`   | `refineryClaimCmd`   | `refinery.go:135-152` | `runRefineryClaim`   (`refinery.go:568-594`) |
| `release` | `refineryReleaseCmd` | `refinery.go:154-166` | `runRefineryRelease` (`refinery.go:596-621`) |

### Rig inference

`getRefineryManager(rigName)` (`refinery.go:274-294`) is the
shared helper. If `rigName` is empty, it uses
`workspace.FindFromCwdOrError` and `inferRigFromCwd` to resolve
the current rig, then calls `getRig` and
`refinery.NewManager(r)`. Returns the manager, rig pointer, and
resolved rig name.

### `start` / `stop` / `restart`

`runRefineryStart` (`refinery.go:296-329`) gets a manager,
checks the rig is not parked/docked, calls
`mgr.Start(foreground, agentOverride)`, and converts
`refinery.ErrAlreadyRunning` to a warning + nil return. If
`--foreground` was set, the function returns directly (expected
to block inside `mgr.Start`).

`runRefineryStop` (`refinery.go:331-352`) calls `mgr.Stop()`,
converts `refinery.ErrNotRunning` into a warning + nil return.

`runRefineryRestart` (`refinery.go:528-558`) rejects
parked/docked rigs, best-effort `mgr.Stop()` (ignoring
`ErrNotRunning`), then `mgr.Start(false, agentOverride)`. Always
background-starts.

### `status`

`runRefineryStatus` (`refinery.go:362-411`). ZFC check: asks
`mgr.IsRunning()` for tmux truth, `mgr.Status()` for session
info (may be nil), and `mgr.Queue()` for the queue length.

JSON shape `RefineryStatusOutput` at `refinery.go:355-360`:

```go
type RefineryStatusOutput struct {
    Running     bool   `json:"running"`
    RigName     string `json:"rig_name"`
    Session     string `json:"session,omitempty"`
    QueueLength int    `json:"queue_length"`
}
```

### `queue`

`runRefineryQueue` (`refinery.go:413-492`). Calls `mgr.Queue()`
for the full ordered queue and prints a position-numbered list.
Position 0 is the currently-processing MR and gets a `▶` prefix
instead of a number (`refinery.go:448-450`). Other positions
render a status tag:

- `MROpen` with `Error != ""` → `[needs-rework]`
- `MROpen` otherwise → `[pending]`
- `MRInProgress` → `[processing]`
- `MRClosed` → one of `[merged]`, `[rejected]`, `[conflict]`,
  `[superseded]`, or `[closed]` based on `CloseReason`.

Each line shows `worker/branch (issue-id) age`.

### `ready` / `blocked` / `unclaimed`

The three queue-filter commands all use
`refinery.NewEngineer(r)` (not the manager) for beads access.

`runRefineryReady` (`refinery.go:689-762`) has two modes:

- **Default**: `eng.ListReadyMRs()` (unclaimed AND unblocked) and
  `eng.ListQueueAnomalies(time.Now())`. Prints a `🚀 Ready MRs`
  block and, if there are any, a `⚠ Queue anomalies` block with
  per-anomaly type, branch, assignee, age, and detail
  (`refinery.go:745-759`). Used by Refinery workers to find MRs
  they can pick up.
- **`--all`**: `runRefineryReadyAll` (`refinery.go:764-812`)
  uses `eng.ListAllOpenMRs()` — all open MRs regardless of
  claim/block state. The comment at `refinery.go:196-199` says
  `--all` is "designed for agent-side queue health analysis"
  and produces raw data including timestamps, assignees, and
  branch existence. Each line can show flags like
  `blocked-by:<id>` and `no-branch` (`refinery.go:798-808`).

`runRefineryBlocked` (`refinery.go:814-859`) calls
`eng.ListBlockedMRs()` — MRs waiting for an open task (e.g.
conflict-resolution task). Lines include the blocking task ID.

`runRefineryUnclaimed` (`refinery.go:623-687`) is older and
bypasses the Engineer entirely, calling
`beads.New(r.Path).ListMergeRequests(...)` with a direct beads
query, then filtering for `issue.Assignee == ""`. Returns
`*refinery.MRInfo` built from `beads.ParseMRFields`. Comment-
free; appears to predate the Engineer API.

### `claim` / `release`

`runRefineryClaim` / `runRefineryRelease`
(`refinery.go:568-621`). Both always infer the rig from cwd
(they do NOT accept a rig positional). Worker ID comes from
`getWorkerID()` (`refinery.go:561-566`):

```go
func getWorkerID() string {
    if id := os.Getenv("GT_REFINERY_WORKER"); id != "" {
        return id
    }
    return "refinery-1"
}
```

Claim semantics (from Long help at `refinery.go:138-149`): when
running multiple refinery workers in parallel, each must claim
an MR before processing. **Claims expire after 10 minutes if not
processed** (for crash recovery — the actual expiry is enforced
inside `refinery.NewEngineer(r).ClaimMR` / `ReleaseMR`, not in
this command file).

Release is called when processing fails and the MR should be
retried by another worker.

### `attach`

`runRefineryAttach` (`refinery.go:494-526`). Uses
`getRefineryManager` (supporting cwd inference), then computes
the session name via `session.RefinerySessionName(session.PrefixFor(rigName))`.
Auto-starts the Refinery if the session is missing (prints
`"Refinery not running for <rig>, starting..."`), then calls
the shared `attachToTmuxSession` helper.

### Flags

| flag | type | default | scope | source |
|---|---|---|---|---|
| `--foreground` | bool | `false` | `start` | `refinery.go:231` |
| `--agent <alias>` | string | `""` | `start`, `attach`, `restart` | `refinery.go:232,235,238` |
| `--json` | bool | `false` | `status`, `queue`, `unclaimed`, `ready`, `blocked` | `refinery.go:241,244,247,250,254` |
| `--all`  | bool | `false` | `ready` | `refinery.go:251` |

## Related commands

- [../binaries/gt.md](../binaries/gt.md) — root.
- [README.md](README.md) — command index.
- [witness.md](witness.md) — the other per-rig agent; same
  positional rig-or-infer shape.
- [mayor.md](mayor.md) — the Refinery escalates conflicts to the
  Mayor per the "spawns FRESH polecat to re-implement" line at
  `refinery.go:41`.
- [deacon.md](deacon.md) — the Deacon patrols Refineries for
  liveness like it does Witnesses.
- [polecat.md](polecat.md) — polecats submit work via
  [done.md](done.md); the Refinery processes what they submit.
  The polecat is already nuked by the time the Refinery picks
  the MR up (`refinery.go:42-44`).
- [done.md](done.md) — the polecat-side completion command; it
  is what adds MRs to the queue this command's `queue`/`ready`
  subcommands display.
- [mq.md](mq.md) — the `gt mq` tree is the read/write surface
  for merge-request beads. Refinery reads via `beads.Engineer`
  and `ListMergeRequests`.
- [sling.md](sling.md) — work dispatch to polecats. Conflict
  recovery re-slings work to a fresh polecat.
- [handoff.md](handoff.md) — polecat-side identity handoff; not
  used by the Refinery itself.
- [synthesis.md](synthesis.md) — conflict-synthesis work that
  may block MRs from entering `ready`.
- [status.md](status.md) — town status surfaces refinery
  running state.
- [convoy.md](convoy.md) — upstream work batching; convoys
  generate work that eventually lands as MRs in this queue.
- [molecule.md](molecule.md) — Refinery runs molecule-based
  patrol logic similar to the Witness (not called out in this
  file but consistent with the pattern).

## Notes / open questions

- **Engineer vs Manager vs direct `beads.New`** — three
  different beads access patterns in one file. `status` and
  `queue` go through `refinery.Manager`; `ready`, `blocked`
  use `refinery.NewEngineer`; `unclaimed` uses `beads.New`
  directly. Likely an evolutionary artifact.
- **`unclaimed` predates the Engineer.** It reimplements filter
  logic inline rather than calling an engineer method. If a
  bug in `ParseMRFields` drifts from what Engineer knows, only
  `unclaimed` will be affected.
- **`GT_REFINERY_WORKER` is implicit.** There's no
  `gt refinery env` or equivalent to tell you which worker ID
  the current shell would use — operators need to check the
  env var by hand.
- **Claim expiry is documented but not enforced in this file.**
  The 10-minute number comes from the Long help at
  `refinery.go:141-142`; the actual enforcement is in
  `refinery.Engineer.ClaimMR` / `ReleaseMR` (not inspected
  here).
- **`--foreground` on start is still live** for the Refinery,
  unlike `gt witness start --foreground` which is vestigial.
  Presumably Refinery has real foreground work (blocking on
  the queue loop) whereas Witness's foreground path has moved
  to a molecule.
- **`getWorkerID` returns `"refinery-1"` by default** — there's
  no deduplication, so two processes with no env var set will
  both be `refinery-1` and can step on each other's claims.
- **Role/persona page pending Batch 6.** The Refinery's actual
  decision loop (claim → rebase → validate → merge or
  escalate) lives in `internal/refinery/` and the refinery
  molecule — not in this command file.
