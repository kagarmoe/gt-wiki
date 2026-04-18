---
title: gt convoy
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/cmd/convoy.go
  - /home/kimberly/repos/gastown/internal/cmd/convoy_stage.go
  - /home/kimberly/repos/gastown/internal/cmd/convoy_launch.go
  - /home/kimberly/repos/gastown/internal/cmd/convoy_watch.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, work, convoy, polecat-safe, tracking, batch, merge-queue]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
phase5_audience: agent
phase8_audited: 2026-04-17
phase8_findings: [silent-suppression]
---

# gt convoy

Manage [convoys](../concepts/convoy.md) — persistent tracking units
for batched work that span issues, [rigs](../concepts/rig.md), and
merge strategies. The top-level parent for 12 subcommands
implementing the [convoy lifecycle](../workflows/convoy-launch.md),
plus an interactive TUI view.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupWork` ("Work Management") (`convoy.go:167`)
**Polecat-safe:** yes (`AnnotationPolecatSafe: "true"` on
`convoy.go:168`)
**Beads-exempt:** no
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/convoy.go` (2739
lines) plus three sibling files: `convoy_stage.go`, `convoy_launch.go`,
`convoy_watch.go`. Registration happens in `init()` at
`convoy.go:375-427`.

### Invocation

```
gt convoy [-i|--interactive]
gt convoy <subcommand> [args...]
```

With `-i`, `runConvoyTUI()` launches a bubbletea TUI
(`convoy.go:170-175`). Otherwise `requireSubcommand` is the fallback.

### Concept (from the Long help at `convoy.go:176-206`)

- A **convoy** is a persistent tracking unit with an `hq-*` ID that
  monitors related issues across rigs.
- A **swarm** is ephemeral — "the workers currently assigned to a
  convoy's issues" — reusing the convoy ID, no separate identity.
- Tracking semantics: `tracks` is a non-blocking relation; convoys
  can span prefixes (`hq-*` tracking `gt-*`, `bd-*`, etc.). Landing
  happens when all tracked issues close, and subscribers get
  notifications.

### Subcommands

Twelve total, registered on `convoy.go:415-424` and in the stage/
launch/watch sibling files:

| subcommand | var | source | one-line summary |
|---|---|---|---|
| `create <name> [issues...]` | `convoyCreateCmd` | `convoy.go:209-241` | Create a new convoy tracking specified issues. |
| `status [convoy-id]` | `convoyStatusCmd` | `convoy.go:243-253` | Show detailed status for a convoy; no ID → all active. |
| `list` | `convoyListCmd` | `convoy.go:255-268` | List convoys (open by default; `--all`, `--tree`, `--json`). |
| `add <convoy-id> <issue-id>...` | `convoyAddCmd` | `convoy.go:270-283` | Add issues; re-opens a closed convoy. |
| `check [convoy-id]` | `convoyCheckCmd` | `convoy.go:285-304` | Auto-close completed convoys — bridges cross-rig gap where `bd close` alone doesn't reach town beads. |
| `stranded` | `convoyStrandedCmd` | `convoy.go:306-326` | Find convoys with ready-but-unassigned work, stuck convoys, or empty convoys — consumed by Deacon patrol. |
| `close <convoy-id>` | `convoyCloseCmd` | `convoy.go:328-346` | Close a convoy; verifies tracked issues done unless `--force`. |
| `land <convoy-id>` | `convoyLandCmd` | `convoy.go:348-373` | Caller-managed landing for `gt:owned` convoys — closes + cleans polecat worktrees. |
| `stage <epic\|issues>` | `convoyStageCmd` | `convoy_stage.go:204` | Stage a convoy (pre-launch); see `convoy_stage.go`. |
| `launch <epic\|issues>` | `convoyLaunchCmd` | `convoy_launch.go:39` | Stage + launch in one step; delegates to `runConvoyStage` with `--launch` (`convoy_launch.go:328`). |
| `watch <convoy-id>` | `convoyWatchCmd` | `convoy_watch.go:34` | Subscribe to convoy completion notifications. |
| `unwatch <convoy-id>` | `convoyUnwatchCmd` | `convoy_watch.go:55` | Remove notification subscription. |

### Status state machine

Defined by the four constants at `convoy.go:97-102`:

```
convoyStatusOpen           = "open"
convoyStatusClosed         = "closed"
convoyStatusStagedReady    = "staged_ready"
convoyStatusStagedWarnings = "staged_warnings"
```

Transition validation in `validateConvoyStatusTransition`
(`convoy.go:129-163`):

- `open ↔ closed` is allowed either direction.
- `staged_*` → `open` (launch) and `staged_*` → `closed` (cancel)
  are allowed.
- `staged_*` ↔ `staged_*` is allowed (re-stage with a different
  result).
- **Rejected:** `open → staged_*` and `closed → staged_*`. Staging is
  a pre-launch state only.

`isStagedStatus(s) == strings.HasPrefix(s, "staged_")`
(`convoy.go:125-127`).

### Create flags (on `convoyCreateCmd`, `convoy.go:377-384`)

| flag | type | default | notes |
|---|---|---|---|
| `--molecule <id>` | string | `""` | Associated molecule ID. |
| `--owner <addr>` | string | `""` | Owner (gets completion notification). |
| `--notify <addr>` | string | `""` | Extra subscriber; `NoOptDefVal = "mayor/"` so `--notify` with no value defaults to `mayor/`. |
| `--owned` | bool | `false` | Caller-managed lifecycle; no automatic witness/refinery registration. |
| `--merge <strategy>` | string | `""` | `direct`, `mr`, or `local` (see Long text `convoy.go:226-232`). |
| `--base-branch <name>` | string | `""` | Target branch for polecats (e.g. `feat/extraction-review`). |
| `--from-epic <epic-id>` | string | `""` | Auto-discover tracked issues from an epic's slingable children. |

### Helpers of note

- **`generateShortID`** (`convoy.go:30-46`): 5-char base36 suffix for
  convoy IDs, ~60M possible values, ~1% birthday-paradox collision
  risk at ~1,100 IDs. Entropy source is swappable via the
  package-level `convoyIDEntropy io.Reader = rand.Reader` for tests.
- **`looksLikeIssueID`** (`convoy.go:50-70`): checks
  `session.HasKnownPrefix` first, then accepts any 2–3 letter
  lowercase-prefix hyphenated ID as a fallback for unregistered rig
  prefixes. Stricter than
  [cat](cat.md)'s `isBeadID` which has no length cap.
- **`runBdJSON`** (`convoy.go:444-471`): internal helper to run `bd`
  with `BEADS_DIR` stripped and stderr captured into the returned
  error. Filters `--allow-stale` if the local `bd` doesn't support it
  (`beads.BdSupportsAllowStale()`).
- **`bdDepListRawIDs`** (`convoy.go:483+`): queries raw
  `dependencies` rows via `bd sql` to work around cross-database
  dependency lookups — `bd dep list` joins with the `issues` table,
  which fails for cross-rig dependencies. References `GH #2624`.
- **`getTownBeadsDir`** (`convoy.go:432-438`): returns the town root
  for bd commands — convoy calls `bd` from town root rather than
  `.beads/` so bd discovers the correct database via its own
  workspace detection.

### Flags on the parent `convoyCmd`

Just one: `-i, --interactive` (`convoy.go:396`). All other flags are
attached to individual subcommands.

### Related commands

- [formula](formula.md) — convoy-type formulas spawn polecats via
  `runFormulaRun` (`formula.go:217-...`). Convoys and formulas are
  tightly coupled in the "kick off a batch" flow.
- [close](close.md) — when any tracked issue closes, `gt close` calls
  `convoy.CheckConvoysForIssue` (`close.go:273`) to propagate the
  completion event. This is the dual of `gt convoy check`.
- [mq](mq.md) — merge queue; the `--merge mr` strategy on convoy
  create defers work into refinery-managed merge requests.
- [compact](compact.md), [changelog](changelog.md) — related
  observability tools.
- [../binaries/gt.md](../binaries/gt.md) — root.

## Failure modes

### Silent suppression (what errors are swallowed?)

- **Convoy status errors during watch suppressed:** `convoy.go:1526-1527` — when iterating convoys in `convoy watch`, errors reading individual convoy metadata are printed to stderr and skipped via `continue`. A broken convoy is invisible in the watch display. **Present** — warning emitted but convoy is omitted.
- **Random ID generation failure undetectable:** `convoy.go:41` — `io.ReadFull(r, b)` error is discarded with `_, _ = ...`. If the entropy source fails, the ID will be derived from zero bytes, producing predictable/colliding IDs. **Absent** — convoy ID collision predicted bug surface.

## Notes / open questions

- **Polecat-safe** (`convoy.go:168`): the only command in this batch
  with the annotation *that isn't* a session-end or durability
  primitive. Convoy operations apparently need to be safe to run
  from inside a polecat session, presumably so polecats can query
  their own convoy status or participate in staging. Confirm what
  polecat-safe actually gates at runtime — still an open question
  tracked in [version.md](version.md).
- **Separate files for `stage`, `launch`, `watch`, `unwatch`.** The
  convoy command is split across four files
  (`convoy.go`, `convoy_stage.go`, `convoy_launch.go`,
  `convoy_watch.go`). The parent `init()` adds the `stage` and
  `launch` subcommands (`convoy.go:423-424`), while `convoy_watch.go`
  adds `watch`/`unwatch` in its own `init()` (`convoy_watch.go:30-31`).
  This distributed registration is worth noting — someone searching
  for the full command tree won't find it in one place.
- **`convoy_launch.go:328` delegates to stage with `--launch`** — so
  `gt convoy launch` is strictly `gt convoy stage --launch` plus a
  different help string. Both share `runConvoyStage`.
- **`convoy.go` is 2739 lines** — the largest command file in this
  batch. Many helpers (state transitions, dep-list, JSON handling)
  live inline rather than in `internal/convoy`. A refactor to push
  logic into the `internal/convoy` package is plausible future work.
