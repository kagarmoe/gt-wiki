---
title: gt sling
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/cmd/sling.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, work, sling, polecat-safe, dispatch, formula, polecat, crew, mayor, hook]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
---

# gt sling

The unified work-dispatch command for Gas Town. Attaches work to an
agent's hook and starts execution immediately. Handles existing
agents (mayor, crew, witness, refinery), auto-spawning polecats when
the target is a rig, dog-dispatch (Deacon's helper workers), formula
instantiation, wisp creation, and auto-convoy creation for dashboard
visibility.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupWork` ("Work Management") (`sling.go:27`)
**Polecat-safe:** yes (`AnnotationPolecatSafe: "true"` on
`sling.go:28`)
**Beads-exempt:** no
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/sling.go` (1192
lines) — one of the two largest command files in the repo. This
page documents the command surface and high-level control flow;
the dispatch machinery proper (resolveTarget, batchDispatch,
pourFormula, autoConvoyCreate, etc.) lives in siblings and is not
walked here.

### Invocation

```
gt sling <bead-or-formula> [target] [flags]
gt sling <bead> <bead> <bead> <rig> [flags]          # batch
gt sling <formula-id> --on <bead> [target] [flags]   # formula-on-bead
gt sling <bead> --stdin <<< "extra instructions"     # stdin mode
```

`Args: cobra.MinimumNArgs(1)` (`sling.go:108`).

### Subcommands

Just one (`sling.go:167`): `respawn-reset <bead-id>`
(`slingRespawnResetCmd`, `:171-194`). Resets the per-bead respawn
counter after the witness-enforced 3-attempt circuit breaker trips,
allowing re-dispatch after root-cause investigation. Implementation
calls `witness.ResetBeadRespawnCount(townRoot, beadID)`.

### Early guards in `runSling` (`sling.go:196-290`)

1. **Telemetry defer** (`:201-210`): `telemetry.RecordSling` captures
   bead, target, error.
2. **Polecat guard** (`:216-223`): refuses if `GT_ROLE` parses to
   `RolePolecat`, or if `GT_ROLE` is unset but legacy `GT_POLECAT`
   is set. Message: `"polecats cannot sling (use gt done for
   handoff)"`. This is the propulsion asymmetry — polecats end
   sessions via [done](done.md), everyone else slings.
3. **Merge-strategy validation** (`:226-233`): `--merge` must be
   `direct`, `mr`, or `local`.
4. **Disable Dolt auto-commit** (`:239-247`): sets
   `BD_DOLT_AUTO_COMMIT=off` for the duration of the sling and
   restores on return. Rationale (`gt-u6n6a`): under concurrent
   batch load, individual auto-commits cause manifest contention
   and "database is read only" errors.
5. **Stdin mode** (`:250-266`): `--stdin` reads stdin into `--args`
   (primary channel) or `--message` (if `--args` already on CLI).
6. **Town root** (`:270-274`) via `workspace.FindFromCwd`; computes
   `townBeadsDir` for hq-bead access from polecat worktrees.
7. **Normalize targets** (`:280-282`): strip trailing slashes from
   all args to forgive `greenplace/` tab-completion artifacts.
8. **`--crew` expansion** (`:286-292`): rewrites the last arg from
   `<rig>` to `<rig>/crew/<name>`.
9. **`ValidateTarget`** (`:296-300`) on the last arg before any
   dispatch path can trigger side-effects.

### Target resolution (from Long help, `sling.go:53-67`)

- `gt sling gt-abc` → self (current agent)
- `gt sling gt-abc crew` → crew worker in current rig
- `gt sling gp-abc greenplace` → auto-spawn polecat in that rig
- `gt sling gt-abc greenplace/Toast` → specific polecat
- `gt sling gt-abc gastown --crew mel` → `gastown/crew/mel`
- `gt sling gt-abc mayor` → mayor
- `gt sling gt-abc deacon/dogs` → auto-dispatch to idle dog
- `gt sling gt-abc deacon/dogs/alpha` → specific dog

### Auto-convoy

Slinging a single issue automatically creates a convoy titled
`"Work: <issue-title>"` to make it visible in `gt convoy list`
(even "swarm of one"). Disabled with `--no-convoy`. The convoy
stores the `--merge` strategy so completed work lands the way the
sender intended. See [convoy](convoy.md) for the state machine.

### Merge strategy (`--merge`)

From `sling.go:47-51` Long text:

- `direct` — push branch directly to main
- `mr` — merge queue (default)
- `local` — keep on feature branch

Stored on the auto-convoy, so `gt done` (polecat-side) and the
refinery/merge-queue know how to land the work.

### Formula slinging

`gt sling <formula-id>` (where the first arg matches a formula
name, not a bead ID) pours the formula to create a molecule +
attach + nudge. `--on <bead-id>` applies a formula to an existing
work bead instead of creating a fresh one ("formula-on-bead"
mode). See [formula](formula.md) for formula mechanics and
[molecule](molecule.md) for the runtime attachment.

### Flags (`sling.go:140-166`)

| flag | default | notes |
|---|---|---|
| `-s`, `--subject` | `""` | Context subject. |
| `-m`, `--message` | `""` | Context message. |
| `-n`, `--dry-run` | `false` | Show what would happen. |
| `--on <bead-id>` | `""` | Apply formula to an existing bead. |
| `--var key=value` | (array) | Formula variable; repeatable. |
| `-a`, `--args <text>` | `""` | Natural-language instructions for the executor. |
| `--stdin` | `false` | Read `--args` and/or `--message` from stdin. |
| `--create` | `false` | Create polecat if missing. |
| `--force` | `false` | Spawn even if polecat has unread mail. |
| `--account <handle>` | `""` | Claude Code account override. |
| `--agent <runtime>` | `""` | Override runtime agent (claude/gemini/codex/…). |
| `--no-convoy` | `false` | Skip auto-convoy creation. |
| `--owned` | `false` | Caller-managed convoy lifecycle. |
| `--hook-raw-bead` | `false` | Expert: hook bead without default formula. |
| `--no-merge` | `false` | Skip merge queue on completion. |
| `--merge <strategy>` | `""` | `direct` / `mr` / `local`. |
| `--no-boot` | `false` | Skip rig boot after polecat spawn. |
| `--max-concurrent <n>` | `0` | Throttle batch spawn rate. |
| `--base-branch <name>` | `""` | Override base branch for polecat worktree. |
| `--ralph` | `false` | Enable Ralph Wiggum loop mode (fresh context per step). |
| `--formula <name>` | `""` | Override formula (default: `mol-polecat-work`). |
| `--crew <name>` | `""` | Target a crew member in the rig. |
| `--review-only` | `false` | Mark work as review-only: assignee reports back, must not merge/commit/push. |

### Batch dispatch

Multiple bead args before a rig target each get their own polecat.
`--max-concurrent` throttles spawn rate to prevent Dolt server
overload (`sling.go:101-107` Long text).

## Notes / open questions

- **THE unified dispatch command.** `sling.go:29` calls itself
  "THE command for assigning work in Gas Town". In practice it
  absorbs: hook attachment, polecat spawning, formula pouring,
  crew/dog dispatch, merge-queue strategy selection, and convoy
  auto-creation. This is the single largest cross-cutting command
  in the codebase.
- **Polecat-safe annotation.** Polecats cannot themselves sling
  (guard at `:216-223`) — so the polecat-safe annotation must
  gate a *read path* that even a polecat can traverse without
  tripping the dispatch machinery. Worth confirming what the
  annotation actually unlocks at runtime.
- **Propulsion principle** (`sling.go:99`): "if it's on your
  hook, YOU RUN IT." The sling/done asymmetry enforces this.
- **Compare:**
  - [hook](hook.md) — attach without starting.
  - `gt sling` — attach + start now (keep context).
  - [handoff](handoff.md) — attach + restart (fresh context).
- **Related commands.** [unsling](unsling.md) (the inverse),
  [convoy](convoy.md) (auto-tracking), [formula](formula.md) /
  [molecule](molecule.md) (what gets poured), [done](done.md)
  (polecat-side counterpart), [scheduler](scheduler.md) (batch
  gating), [mq](mq.md) (merge queue for `--merge=mr`),
  [assign](assign.md) (lighter sibling: create bead + hook).
- **`respawn-reset` subcommand** is easy to miss — it's the
  break-glass for the 3-attempt witness circuit breaker.
