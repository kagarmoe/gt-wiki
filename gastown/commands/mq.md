---
title: gt mq
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/cmd/mq.go
  - /home/kimberly/repos/gastown/internal/cmd/mq_next.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, work, merge-queue, refinery, integration-branch]
phase3_audited: 2026-04-15
phase3_findings: [wiki-stale]
phase3_severities: [wrong]
phase3_findings_post_release: false
phase5_audience: agent
phase8_audited: 2026-04-17
phase8_findings: [partial-completion, silent-suppression]
---

# gt mq

Manage merge requests and the merge queue for a rig. Parent command
for seven MR-level subcommands plus a three-subcommand `integration`
branch-management group. (Phase 3 Batch 1c wiki-stale correction:
the Phase 2 page body said "six MR-level subcommands" and described
the tree as "Ten total"; `mq_next.go` wires a 7th MR subcommand
`mq next` via its own `init()` that Phase 2 missed.)

**Also known as:** `gt mr` (alias, `mq.go:64`).
**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupWork` ("Work Management") (`mq.go:65`)
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/mq.go` (585
lines). Parent command at `mq.go:62-74` is a pure `requireSubcommand`
router.

### Invocation

```
gt mq <subcommand> [args]
gt mr <subcommand> [args]        # alias
```

The alias is a semantic pun: `mq` = merge queue, `mr` = merge
request. The help text notes: `'gt mr' is equivalent to 'gt mq'
(merge request vs merge queue)` (`mq.go:70`).

### Subcommand tree

Eleven total: seven MR-level subcommands plus a three-subcommand
`integration` group (with `integration` itself as the parent).
Registered across `mq.go:306-365` (`submit`, `retry`, `list`,
`reject`, `status`, `post-merge`, and the `integration` tree) and
in the sibling file `mq_next.go` via its own `init()` (`next`).

#### MR-level subcommands

1. **`gt mq submit`** (`mqSubmitCmd`, `mq.go:76-113`)

   Submit the current branch to the merge queue. Creates a
   merge-request bead processed by the Refinery. Auto-detects branch,
   issue (parsed from branch name e.g. `polecat/Nux/gp-xyz` →
   `gt-xyz`), worker, rig, target, and priority. Target-branch
   auto-detection cascade (`mq.go:88-96`):

   1. `--epic <id>` → the integration branch for `<epic>`.
   2. Source issue's parent epic's integration branch if any.
   3. Otherwise: main.

   **Polecat auto-cleanup** (`mq.go:98-104`): when run from a polecat
   work branch (`polecat/<worker>/<issue>`), submit triggers polecat
   shutdown after submitting the MR — sends a lifecycle request to
   the Witness and waits for termination. `--no-cleanup` disables
   this. This is the mechanical counterpart to [done](done.md)'s
   MQ submission step.

2. **`gt mq retry <rig> <mr-id>`** (`mqRetryCmd`, `mq.go:115-128`)

   Reset a failed MR so the refinery can process it again. `--now`
   bypasses the refinery polling loop.

3. **`gt mq list <rig>`** (`mqListCmd`, `mq.go:130-151`)

   Show the merge queue for a rig. Output is a table of MR ID,
   status, priority, branch, worker, age (with an indented
   "waiting on <id>" line for blocked MRs per the example at
   `mq.go:137-143`).

4. **`gt mq reject <rig> <mr-id-or-branch>`** (`mqRejectCmd`,
   `mq.go:153-166`)

   Manually close an MR with a `rejected` status without merging.
   **The source issue is not closed** — work is not done
   (`mq.go:157-158`). `--notify` sends a mail notification to the
   worker.

5. **`gt mq post-merge <rig> <mr-id>`** (`mqPostMergeCmd`,
   `mq.go:171-189`)

   Post-merge cleanup: close the MR bead (status: `merged`), close
   the source issue, delete the remote polecat branch unless
   `--skip-branch-delete`. Designed for use by the refinery formula
   after a successful merge to main. The branch name is read from
   the MR bead — no manual branch argument.

6. **`gt mq status <id>`** (`mqStatusCmd`, `mq.go:191-203`)

   Show detailed MR info: all fields, current status with
   timestamps, dependencies, blockers, and processing history.

7. **`gt mq next <rig>`** (`mqNextCmd`, `mq_next.go:20-48`,
   `runMQNext :50+`)

   Show the next merge request to process for a rig, ordered by
   priority score (convoy age + issue priority + retry deprioritisation
   + MR age FIFO tiebreaker) or by `--strategy=fifo` for pure FIFO
   ordering. `--quiet` prints just the MR ID; `--json` emits the full
   MR shape. Wired via `mq_next.go:47` — `mqCmd.AddCommand(mqNextCmd)`
   — in its own `init()`, not in `mq.go`'s `init()`.

#### Integration-branch subcommands (`gt mq integration ...`)

Integration branches batch multiple MRs for an epic into a single
shared branch, then land to main atomically (`mq.go:205-219`).
`mqIntegrationCmd` is itself a `requireSubcommand` router.

1. **`gt mq integration create <epic-id>`** (`mqIntegrationCreateCmd`,
   `mq.go:221-257`)

   Create an integration branch from main, push to origin, and
   store the actual branch name in epic metadata. Branch naming
   follows `merge_queue.integration_branch_template` from rig
   settings by default (`integration/<sanitized-title>`), with
   `--branch` for one-off override. Template variables
   (`mq.go:234-238`): `{title}`, `{epic}`, `{prefix}`, `{user}`.

   Collision policy (`mq.go:240-242`): if two epics produce the
   same branch name, a numeric suffix from the epic ID is appended.

2. **`gt mq integration land <epic-id>`** (`mqIntegrationLandCmd`,
   `mq.go:259-287`)

   Merge the integration branch to main. Seven steps in the Long
   text (`mq.go:267-274`):

   1. Verify all MRs targeting `integration/<epic>` are merged.
   2. Verify integration branch exists.
   3. Merge `integration/<epic>` to main (`--no-ff`).
   4. Run tests on main.
   5. Push to origin.
   6. Delete integration branch.
   7. Update epic status.

   `--force` skips the merged-MR check; `--skip-tests` skips step
   4; `--dry-run` previews.

3. **`gt mq integration status <epic-id>`**
   (`mqIntegrationStatusCmd`, `mq.go:289-304`)

   Show integration-branch status: branch name + creation date,
   commits ahead of main, merged MRs (closed), pending MRs (open)
   targeting the branch.

### Flags

Per the `init()` at `mq.go:306-365`. Grouped by subcommand:

**`submit`** (`mq.go:307-314`):

| flag | short | type | default | description |
|---|---|---|---|---|
| `--branch` | — | string | `""` | Source branch (default: current) |
| `--issue` | — | string | `""` | Source issue (default: parse from branch) |
| `--epic` | — | string | `""` | Target epic's integration branch |
| `--priority` | `-p` | int | `-1` | Override priority 0-4 |
| `--no-cleanup` | — | bool | `false` | Disable polecat auto-cleanup |
| `--skip-deps` | — | bool | `false` | Skip molecule step dependency check |
| `--resubmit` | — | bool | `false` | Resubmit after fix (skips dep check) |

**`retry`** (`mq.go:317`): `--now` bool.

**`list`** (`mq.go:320-325`): `--ready`, `--status`, `--worker`,
`--epic`, `--json`, `--verify` (shows MISSING for deleted branches).

**`reject`** (`mq.go:328-330`): `-r/--reason`, `--notify`,
`--stdin` (read reason from stdin).

**`status`** (`mq.go:333`): `--json`.

**`post-merge`** (`mq.go:336`): `--skip-branch-delete`.

**`next`** (`mq_next.go:43-45`): `--strategy` (`priority`|`fifo`,
default `priority`), `--json`, `-q/--quiet`.

**`integration create`** (`mq.go:347-349`): `--branch` template,
`--base-branch` (create from something other than main), `--force`
(recreate existing).

**`integration land`** (`mq.go:353-355`): `--force`, `--skip-tests`,
`--dry-run`.

**`integration status`** (`mq.go:359`): `--json`.

### `findCurrentRig` helper (`mq.go:369+`)

This file owns the `findCurrentRig(townRoot)` function, used by
[done](done.md) and several other commands. Returns `(rigName, *rig.Rig,
error)`. Resolution cascade:

1. Relative path from town root — first path component is the rig
   name.
2. Fallback to `GT_RIG` env var when cwd is town root (e.g. `cd ~/gt
   && gt ...`) and relPath is `.` (`mq.go:388-392`).
3. Error if both fail.

Then loads `<townRoot>/mayor/rigs.json` via `config.LoadRigsConfig`
to return the rig object (`mq.go:398+`).

### Related commands

- [done](done.md) — polecats call `gt done`, which submits to the
  merge queue (its downstream is effectively `gt mq submit`
  followed by lifecycle transitions).
- [convoy](convoy.md) — convoys with `--merge mr` create merge
  requests via the refinery loop; convoys with `--merge direct`
  bypass MQ entirely.
- [formula](formula.md) — refinery formula lands integration
  branches post-merge via `gt mq post-merge`.
- [close](close.md) — closing a merged MR bead is what `post-merge`
  does internally, so the close → convoy-check chain still fires.
- [../binaries/gt.md](../binaries/gt.md) — root.

## Drift

See forward-link: [../drift/README.md](../drift/README.md).

### Phase 2 wiki body missed the `mq next` subcommand wired in a sibling file (wiki-stale)

- **Category:** `wiki-stale`
- **Severity:** `wrong`
- **Phase 2 root cause:** `phase-2-incomplete` (heuristic
  determination per Sweep 1 convention — the Phase 2 page read
  `mq.go` in isolation and enumerated the six subcommands registered
  by `mq.go:init()` at `:339-344`, missing the seventh subcommand
  `mq next` which is declared and registered in the sibling file
  `mq_next.go` via its own `init()` at `mq_next.go:47`. Both Phase
  2's HEAD and the current HEAD have `mq_next.go` with its
  registration line; the file was present at `v1.0.0`. Phase 2 was
  wrong at Phase 2 time, not stale against churn.)
- **Phase 2 claim (removed):** the Phase 2 page body stated
  "Ten total, registered in `init()` at `mq.go:306-365`. Six
  top-level MR subcommands plus a three-subcommand `integration`
  group." and "Registered at `mountain.go:100-103`"-style per-file
  single-source citations.
- **Current reality (2026-04-15, gastown HEAD `9f962c4a`):** seven
  top-level MR subcommands — the original six (`submit`, `retry`,
  `list`, `reject`, `status`, `post-merge`) plus `next`, which is
  declared at `mq_next.go:20-40` with its own `runMQNext`
  implementation at `:50+` and registered via
  `mqCmd.AddCommand(mqNextCmd)` at `mq_next.go:47`. Total tree:
  11 subcommands (7 MR-level + the `integration` parent + 3
  integration children). Confirmed via
  `grep -rn "mqCmd.AddCommand" internal/cmd/`. The `mq_next.go`
  file exists at `v1.0.0:internal/cmd/mq_next.go` with its
  `AddCommand` line unchanged.
- **Fix tier:** `wiki` — already fixed inline in the
  `### Subcommand tree` / `#### MR-level subcommands` tables above
  (the Phase 2 "six subcommands" count is replaced with seven; the
  new row enumerates `mq next`'s behavior, flags, and sibling-file
  registration; the frontmatter `sources:` list now includes
  `mq_next.go`).

## Failure modes

### Partial completion (what doesn't it clean up?)

- **`mq post-merge` branch delete failure is non-fatal:** `mq.go:570-573` — if `rigGit.DeleteRemoteBranch` fails, a warning is printed but the command returns nil. The MR bead and source issue are already closed. The orphaned remote branch persists indefinitely. **Present** — warning emitted, but no automated retry or cleanup bead.
- **`mq post-merge` local branch delete error fully swallowed:** `mq.go:578-579` — `_ = err` explicitly discards the error from `DeleteBranch`. **Absent** — local tracking branches accumulate without warning.

### Silent suppression (what errors are swallowed?)

- **Rigs config load failure silently creates empty config:** `mq.go:399-401` — if `config.LoadRigsConfig` fails, a fresh empty config is used. The rig lookup at line 406 may then fail with a confusing "rig not found" error rather than surfacing the root cause (config unreadable). **Absent** — root cause masked by downstream error.
- **Rig settings load error silently defaults to delete-enabled:** `mq.go:547-549` — if `config.LoadRigSettings` fails, `deleteEnabled` stays `true` (default). A rig that configured `delete_merged_branches: false` has its setting silently ignored. **Absent** — branches deleted despite config requesting preservation.

## Notes / open questions

- **`findCurrentRig` belongs in `mq.go`?** It's a general helper
  (used by done.go and probably others) but lives in `mq.go` for
  historical reasons. A refactor into `internal/rig` would be
  cleaner — neutral observation.
- **`config.LoadRigsConfig` target** — hardcoded to
  `<townRoot>/mayor/rigs.json` (`mq.go:398`). Contrast with
  [changelog](changelog.md) which uses
  `constants.DirMayor`/`constants.FileRigsJSON`. Inconsistent path
  construction across siblings.
- **Integration branch template default** — the default template
  lives in rig settings under
  `merge_queue.integration_branch_template`, but what if it's
  unset? The Long text says "`integration/<sanitized-title>`" but
  the fallback default isn't shown in the surface excerpt.
- **`mq` vs `mr` alias** — the pun is cute but also a UX hazard:
  `gt mr status` and `gt mq status` do the same thing, so docs
  referencing one may surprise readers searching for the other.
