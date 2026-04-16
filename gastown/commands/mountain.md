---
title: gt mountain
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/cmd/mountain.go
  - /home/kimberly/repos/gastown/internal/cmd/convoy.go
tags: [command, work, mountain, convoy, polecat-safe, epic, grinding]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
---

# gt mountain

Activate the "Mountain-Eater" on an epic — stage a convoy from the
epic's slingable children, add the `mountain` label (opting into
enhanced stall detection), and launch Wave 1 in one shot. Mountains
are convoys marked for autonomous grinding by the Deacon and Witness.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupWork` ("Work Management") (`mountain.go:23`)
**Polecat-safe:** yes (`AnnotationPolecatSafe: "true"` on
`mountain.go:24`)
**Beads-exempt:** no
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/mountain.go` (710
lines). Heavy use of staging/DAG helpers from `convoy.go`: `bdShow`,
`collectBeads`, `buildConvoyDAG`, `detectErrors`, `detectWarnings`,
`computeWaves`, `createStagedConvoy`, `transitionConvoyToOpen`,
`dispatchWave1`, `checkBlockedRigsForLaunch`.

### Invocation

```
gt mountain <epic-id> [--force] [--json]
gt mountain <subcommand> [args]
```

### `gt mountain <epic-id>` pipeline (`runMountain`, `mountain.go:110-270`)

1. **Validate** — `bdShow(epicID)` (`:114`); refuse if
   `IssueType != "epic"` (`:118-120`).
2. **Collect** — `collectBeads` with `StageInputEpic` (`:126-130`):
   walks the epic tree and returns beads + dependencies.
3. **Build DAG** — `buildConvoyDAG` (`:132`).
4. **Findings** — `detectErrors` + `detectWarnings`; any errors abort
   (`:133-141`).
5. **Compute waves** — `computeWaves` from blocking dependencies
   (`:143`). Gated tasks produce extra warnings (`:149-157`).
6. **Status** — `chooseStatus(errs, warns)` returns `staged_ready` or
   `staged_warnings`. If `staged_warnings` and `--force` not set,
   abort (`:198-200`).
7. **Create staged convoy** — `createStagedConvoy(dag, waves, status,
   "Mountain: "+result.Title)` (`:203-207`).
8. **Add `mountain` label** — `bdAddLabelTown(convoyID, "mountain")`
   (`:214`, helper at `:272-284`).
9. **Launch** — `transitionConvoyToOpen(convoyID, true)` (`:219`) and
   `checkBlockedRigsForLaunch(dag, townRoot, mountainForce)` (`:228`).
10. **Dispatch Wave 1** — `dispatchWave1(convoyID, dag, waves,
    townRoot)` (`:232`), then render results sorted by bead ID.

Output tells the user: "ConvoyManager will feed subsequent waves,
Deacon will audit progress every ~10 minutes, check status with
`gt mountain status <convoyID>`" (`:265-267`).

### Subcommands

Registered at `mountain.go:100-103`:

| subcommand | source | one-liner |
|---|---|---|
| `status [epic-id\|convoy-id]` | `:50-61` | No args → list active mountains with progress bars; with arg → detailed status (completed/active/ready/skipped/blocked). |
| `pause <id>` | `:63-72`, `:643-663` | Add `mountain:paused` label; ConvoyManager skips dispatch. Active polecats finish current work. |
| `resume <id>` | `:74-81`, `:666-684` | Remove `mountain:paused` label. |
| `cancel <id>` | `:83-92`, `:687-710` | Remove `mountain` (and `mountain:paused`) labels but keep convoy. |

`runMountainStatus` (`:309-320`) dispatches between
`showAllMountainStatus` and `showMountainDetail`. The detail view
categorizes tracked beads into `completed/active/ready/skipped/blocked`
via `hasBeadLabel(…, "mountain:skipped")` and DAG blocker analysis
(`:418-457`).

### Flags

| flag | attached to | default | source |
|---|---|---|---|
| `-f`, `--force` | `mountainCmd` | `false` | `:95` |
| `--json` | `mountainCmd`, `mountainStatusCmd` | `false` | `:96-98` |

### Helpers

- `bdAddLabelTown` / `bdRemoveLabelTown` (`:272-298`): run
  `bd update <id> --add-label=<label>` / `--remove-label=<label>` in
  the town beads dir.
- `findMountainConvoys` (`:565-577`): queries `bd list --type=convoy
  --status=open --label=mountain --json`.
- `resolveMountainID` (`:582-606`): accepts either epic-id or
  convoy-id; if epic, searches for a mountain convoy whose title
  contains the epic title.
- `renderProgressBar` (`:626-640`): Unicode bar `███░░░` helper.

## Notes / open questions

- **Mountain-Eater metaphor.** A "mountain" is strictly a convoy
  with the `mountain` label. No separate data type. Other convoys
  keep working as normal; the label opts in to enhanced stall
  detection and skip-after-N-failures monitoring.
- **Pause semantics are soft** — `pause` only adds a label the
  ConvoyManager reads; existing polecats finish their current step.
  There is no forced kill path in this file.
- **Epic → convoy title mapping is fuzzy.** `resolveMountainID` does
  a substring match on convoy title (`:600`), so if two mountains
  track epics with similar titles, the first one wins.
- **Cross-reference** — see [convoy](convoy.md) for the underlying
  tracking unit, [ready](ready.md) for how wave-computed readiness
  surfaces in the town-wide view, and [done](done.md) for the
  polecat-side completion path that drives waves forward.
