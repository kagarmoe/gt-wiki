---
title: gt done
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/cmd/done.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
  - /home/kimberly/repos/gastown/docs/CLEANUP.md
tags: [command, work, polecat-safe, polecat, merge-queue, handoff]
phase3_audited: 2026-04-15
phase3_findings: [drift]
phase3_severities: [wrong]
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
phase5_audience: agent
phase8_audited: 2026-04-17
phase8_findings: [partial-completion, silent-suppression]
---

# gt done

Signal that a polecat's work is complete and ready for the merge
queue. The canonical way a polecat ends a work cycle — submits the
branch to MQ, notifies the Witness, and transitions the polecat to
IDLE with sandbox preserved.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupWork` ("Work Management") (`done.go:33`)
**Polecat-safe:** yes (`AnnotationPolecatSafe: "true"` on
`done.go:34`)
**Beads-exempt:** no
**Branch-check-exempt:** no
**`SilenceUsage: true`** (`done.go:59`) — operational errors don't
print the usage blurb, which "confuses agents" per the inline
comment.

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/done.go:31-89` for
the cobra surface; the full `runDone` implementation is 1896 lines
and encodes the entire polecat end-of-work dance.

**Critical role guard** at `done.go:91-100`: `gt done` is for
polecats only. If `BD_ACTOR` is set and `!isPolecatActor(actor)`, the
command refuses with a message explaining the asymmetry — polecats
end sessions with `gt done`, other roles persist across tasks and
don't use it. See the related guard in [prime](prime.md) that warns
**agents not to run `gt done` at all** unless they are a polecat.

### Invocation

```
gt done [--issue <id>] [--status <STATUS>] [--pre-verified] [--target <branch>] [--priority <N>] [--cleanup-status <state>] [--resume]
```

### Exit statuses (`done.go:73-77`)

```go
ExitCompleted = "COMPLETED"
ExitEscalated = "ESCALATED"
ExitDeferred  = "DEFERRED"
```

From the `Long` help (`done.go:45-49`):

- `COMPLETED` — work done, MR submitted (default).
- `ESCALATED` — hit a blocker, needs human intervention; skips MR.
- `DEFERRED` — work paused, issue still open; skips MR.

### Behavior outline

Based on the first ~200 lines of `runDone` (`done.go:91-200`) and
the top-level help:

1. **Record telemetry on exit** via
   `telemetry.RecordDone(context.Background(), status, retErr)` in a
   defer (`done.go:92`).
2. **Validate role** — block non-polecats as described above.
3. **Validate exit status** — must be one of the three constants
   (`done.go:102-106`).
4. **Find workspace with fallback** — `workspace.FindFromCwdWithFallback`
   (`done.go:115-118`). The fallback exists because the Witness may
   have deleted the polecat's worktree before `gt done` finishes
   (referenced bead `hq-3xaxy`). If cwd is unavailable, fall back to
   `GT_POLECAT_PATH`.
5. **Determine current rig** — walk the rel-path from town root to
   cwd for the first component, then override with `GT_RIG` env var
   if set (`done.go:132-151`). The env var is more reliable than
   cwd-derived rig name because Claude Code's Bash tool may have
   reset cwd to `mayor/rig`.
6. **Normalize cwd to polecat worktree** — if cwd is not inside
   `polecats/`, reconstruct the expected polecat clone path using
   `GT_POLECAT` or `GT_CREW` env vars (`done.go:161-179`). Then walk
   up from cwd to the git repo root stopping if we leave the
   polecats area (`done.go:185-198`).
7. **Git init + further work** — initializes
   `internal/git`, `internal/polecat`, session, mail, events,
   telemetry, templates, townlog, rig, beads, config. The full flow
   (branch detection, MR submission, worktree sync to main, Witness
   notification, IDLE transition) spans the remaining 1700 lines.

The surface-level help text (`done.go:36-57`) summarizes the
downstream steps as:

1. Submits the current branch to the merge queue.
2. Auto-detects issue ID from branch name.
3. Notifies the Witness with the exit outcome.
4. Syncs worktree to main and transitions polecat to IDLE (sandbox
   preserved, session stays alive for reuse).

### Subcommands

None. `grep doneCmd.AddCommand ... done.go` returns no hits.

### Flags

Defined in `init()` at `done.go:79-89`:

| flag | short | type | default | description |
|------|-------|------|---------|-------------|
| `--issue` | — | string | `""` | Source issue ID (default: parse from branch name) |
| `--priority` | `-p` | int | `-1` | Override priority (0-4, default: inherit from issue) |
| `--status` | — | string | `COMPLETED` | Exit status: COMPLETED, ESCALATED, or DEFERRED |
| `--cleanup-status` | — | string | `""` | Git cleanup status: clean, uncommitted, unpushed, stash, unknown (ZFC: agent-observed) |
| `--resume` | — | bool | `false` | Resume from last checkpoint (auto-detected, for Witness recovery) |
| `--pre-verified` | — | bool | `false` | Mark MR as pre-verified (polecat ran gates after rebasing onto target) |
| `--target` | — | string | `""` | Explicit MR target branch (overrides formula_vars and auto-detection) |

### Related commands

- [handoff](handoff.md) — the sister command. Polecats running
  `gt handoff` are silently redirected to `gt done --status
  DEFERRED` (`handoff.go:152-161`). Non-polecats use `gt handoff`
  directly.
- [hook](hook.md) — refuses polecat access entirely
  (`hook.go:224-231`) — polecats cannot hook work because their
  session lifecycle is driven by `gt done`.
- [mq](mq.md) — the merge queue that `gt done` submits into (via
  the underlying refinery / `bd mq` path).
- [convoy](convoy.md) — convoys that tracked the closed issue get
  checked via [close](close.md)'s `checkConvoyCompletion`.
- [../binaries/gt.md](../binaries/gt.md) — root.

## Docs claim

### Source

- `/home/kimberly/repos/gastown/docs/CLEANUP.md:28` — the `gt done`
  row of the "Polecat (Agent Sandbox) Cleanup" table.

### Verbatim

> `| `gt done` | Polecat self-cleaning: pushes branch, submits MR (by default), self-nukes worktree, kills own session. MR skipped for `--status ESCALATED\|DEFERRED` or `no_merge` paths |`

## Drift

See forward-link: [../drift/README.md](../drift/README.md).

### `docs/CLEANUP.md` claims `gt done` self-nukes the worktree and kills its session, but code transitions to IDLE with sandbox preserved

- **Claim source:**
  `/home/kimberly/repos/gastown/docs/CLEANUP.md:28`.
- **Docs claim:** the CLEANUP.md row for `gt done` states that the
  command "self-nukes worktree, kills own session." This implies a
  hard teardown of the polecat's sandbox as the default completion
  path.
- **Code does:** `doneCmd.Long` at
  `/home/kimberly/repos/gastown/internal/cmd/done.go:36-57` describes
  the completion path as "Syncs worktree to main and transitions
  polecat to IDLE (sandbox preserved, session stays alive for
  reuse)." The matching implementation-side comment at
  `done.go:108-110` is explicit: *"Persistent polecat model
  (gt-hdf8): sessions stay alive after gt done. No deferred session
  kill — the polecat transitions to IDLE with sandbox preserved.
  The Witness handles any cleanup if the polecat gets stuck."* The
  DONE→IDLE transition is handled at `done.go:1209-1263`, which
  syncs the worktree to main, deletes the old branch, and prints
  `"✓ Polecat transitioned to IDLE — ready for new work"`
  (`done.go:1263`). The function `selfNukePolecat` at
  `done.go:1673-1680` still exists but has **no call sites** in
  `done.go` at HEAD `9f962c4a` (verified via
  `grep -n "selfNukePolecat\b" internal/cmd/done.go` — only the
  declaration and a reference in a doc comment at `:1735` that says
  it is "Kept for explicit kill scenarios"). Nuke is now only
  triggered explicitly via `gt polecat nuke`, not as a side effect
  of `gt done`. The Cobra `Long` text and the code agree with each
  other; only `docs/CLEANUP.md` carries the stale claim.
- **Category:** `drift`
- **Severity:** `wrong`
- **Fix tier:** `docs` — edit `docs/CLEANUP.md:28` to describe the
  actual behavior: "Polecat self-cleaning: pushes branch, submits
  MR (by default), syncs worktree to main, transitions session to
  IDLE (sandbox preserved for reuse). MR skipped for `--status
  ESCALATED|DEFERRED` or `no_merge` paths." The Cobra `Long` and
  the inline comments at `done.go:36-57`, `:108-110`, `:1209-1263`
  are already correct and do not need changes. The bead references
  `gt-4ac` and `gt-hdf8` track the original "persistent polecat
  model" redesign and should be mentioned in the docs commit
  message.
- **Release position:** `in-release` — the DONE→IDLE transition
  code at `done.go:1209-1263` is byte-equivalent at `v1.0.0` (the
  persistent polecat model shipped in the v1.0.0 release), and the
  stale `docs/CLEANUP.md:28` row is also byte-identical at
  `v1.0.0:docs/CLEANUP.md`. The drift was already present at the
  release tag.

**Note on provenance:** this finding was originally filed by the
pre-plan 2026-04-14 `docs/CLEANUP.md` drift-found entry (see
[log.md](../../log.md) `[2026-04-14] drift-found |
docs/CLEANUP.md vs code`) and initially drafted as a `## Drift`
section on this page in commit `f143813`. Phase 3 Batch 1c
reformatted the section to the v1.2 schema (verbatim docs quote,
fresh `file:line` refs, Category/Severity/Fix tier/Release position
fields, forward link to the drift index). The substance of the
finding is unchanged; only the shape is. Per the plan, Sweep 2
Batch 9 (`docs/CLEANUP.md` formal re-audit) will supersede the
pre-plan informal entry and this section will be left as-is.

## Failure modes

### Partial completion (what doesn't it clean up?)

- **Auto-commit can fail silently while continuing:** `done.go:292-316` — if `g.Add("-A")` fails, a warning is printed but gt done continues. The uncommitted work remains at risk. If `g.Commit(autoMsg)` fails, same pattern — warning printed, doneCleanupStatus NOT updated from "uncommitted", and the flow continues with stale status. **Present** — warnings emitted, but work may still be lost.
- **Push failure leaves branch in limbo:** `done.go:673-712` — the code has three fallback push paths (worktree, bare repo, mayor/rig). If all three fail, the branch is "committed locally but failed to push" — the commits exist only in the worktree's git objects. Since the polecat's worktree may be nuked after gt done completes, the commits could be lost. The Witness is notified but can't recover the data. **Present** — witness notification exists but data recovery is not guaranteed.
- **Overlay strip adds commit but fails silently if git operations fail:** `done.go:564` — `stripOverlayCLAUDEmd` internally runs git operations that could fail, leaving the overlay in the committed history. **Absent** — if the strip commit fails, CLAUDE.md overlay pollutes the PR diff.

### Silent suppression (what errors are swallowed?)

- **Issue close retry failures after no-MR path:** `done.go:533-536` — after 3 failed close attempts, a warning is printed but gt done continues. The issue remains in HOOKED status with the assignee pointing to a polecat that's about to be cleaned up. **Present** — warning emitted but issue is orphaned.
- **Heartbeat state write failure ignored:** `done.go:387-388` — `polecat.TouchSessionHeartbeatWithState` errors are not checked. If heartbeat write fails, the Witness doesn't know the polecat is in the gt done flow. **Absent** — Witness falls back to timer-based inference which may be wrong.
- **Done-intent label write failure ignored:** `done.go:376` — `setDoneIntentLabel` errors are not propagated. If this fails, the Witness crash-recovery mechanism doesn't know gt done was attempted. **Absent** — crash recovery gap.

## Notes / open questions

- **1896-line command.** The bulk of the logic is a linear script
  handling edge cases: deleted worktrees, cwd resets by Claude Code's
  Bash tool, polecat-not-polecat role drift, and Witness-mediated
  cleanup. Each edge-case patch has a bead reference in comments
  (e.g. `hq-3xaxy`, `gt-hdf8`). Worth a dedicated concept page on
  "polecat lifecycle" rather than forcing it into a command page.
- **Telemetry hook** uses `strings.ToUpper(doneStatus)` for the
  status label (`done.go:92`). So `--status completed` works just
  as well as `COMPLETED`.
- **Persistent polecat model** — comment at `done.go:108-110`
  references bead `gt-hdf8`: "sessions stay alive after gt done. No
  deferred session kill — the polecat transitions to IDLE with
  sandbox preserved." The sandbox-preservation design is worth a
  concept page.
- **`BD_ACTOR` vs `GT_ROLE`** — the role guard checks `BD_ACTOR` not
  `GT_ROLE`. Both are polecat identity indicators but come from
  different layers; confirm whether they're kept in sync.
