---
title: gt polecat
type: command
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/cmd/polecat.go
  - /home/kimberly/repos/gastown/internal/cmd/polecat_identity.go
  - /home/kimberly/repos/gastown/internal/cmd/polecat_spawn.go
  - /home/kimberly/repos/gastown/internal/cmd/polecat_cycle.go
  - /home/kimberly/repos/gastown/internal/cmd/polecat_helpers.go
tags: [command, agents, polecat, worktree, nuke, lifecycle, git, identity]
---

# gt polecat

[Polecat](../roles/polecat.md) lifecycle, identity, and cleanup
commands. A polecat has **persistent identity** (an agent bead
tracking CV/work history) and **ephemeral sessions/sandboxes**
(worktrees that come and go with work). This page documents the
`gt polecat` CLI subcommand tree only; see the
[Polecat role page](../roles/polecat.md) for startup flow,
self-cleaning model, and interaction with refinery/witness/deacon.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupAgents` ("Agent Management") (`polecat.go:34`)
**Polecat-safe:** no
**Beads-exempt:** yes (listed in `beadsExemptCommands` on
`/home/kimberly/repos/gastown/internal/cmd/root.go:44-77`)
**Branch-check-exempt:** no
**Alias:** `gt polecats`

## What it actually does

The parent `polecatCmd` is defined in
`/home/kimberly/repos/gastown/internal/cmd/polecat.go:31-60`
(1786 lines). Subcommands are split across several sibling
files in the same package:

- `polecat.go` — parent cmd plus `list`, `add` (deprecated),
  `remove`, `status`, `git-state`, `check-recovery`, `gc`,
  `nuke`, `stale`, `prune`, `pool-init` (11 subcommands).
- `polecat_identity.go` (1079 lines) — the entire
  `gt polecat identity` sub-tree with 5 subcommands (`add`,
  `list`, `show`, `rename`, `remove`).
- `polecat_spawn.go` (506 lines) — additional spawn-related
  subcommands (inspection outside this wiki page's scope —
  flag this as a follow-up).
- `polecat_cycle.go` (85 lines) — additional cycle-related
  subcommands (same — follow-up).
- `polecat_helpers.go` (270 lines) — shared helpers
  (`resolvePolecatTargets`, `checkPolecatSafety`,
  `polecatBeadIDForRig`, `purgeClosedEphemeralBeads`, etc.).

This page documents the user-facing subcommands grounded in
`polecat.go` + `polecat_identity.go`. Commands defined in
`polecat_spawn.go` / `polecat_cycle.go` are listed but not
enumerated in depth here.

The parent `polecatCmd` uses `RunE: requireSubcommand` — bare
`gt polecat` prints the subcommand list.

### Long help (polecat identity model)

`polecat.go:37-59` is the canonical CLI-level framing of what a
polecat IS:

> Polecats have PERSISTENT IDENTITY but EPHEMERAL SESSIONS.
> Each polecat has a permanent agent bead and CV chain that
> accumulates work history across assignments. Sessions and
> sandboxes are ephemeral — spawned for specific tasks, cleaned
> up on completion — but the identity persists.
>
> A polecat is either:
>   - Working: Actively doing assigned work
>   - Stalled: Session crashed mid-work (needs Witness
>     intervention)
>   - Zombie: Finished but gt done failed (needs cleanup)
>   - Nuked: Session ended, identity persists (ready for next
>     assignment)
>
> Self-cleaning model: When work completes, the polecat runs
> 'gt done', which pushes the branch, submits to the merge
> queue, and exits. The Witness then nukes the sandbox. The
> polecat's identity (agent bead) persists with
> agent_state=nuked, preserving work history.
>
> Session vs sandbox: The Claude session cycles frequently
> (handoffs, compaction). The git worktree (sandbox) persists
> until nuke. Work survives session restarts.
>
> Cats build features. Dogs clean up messes.

### Invocation

```
gt polecat list      [rig]                    [--all] [--json]
gt polecat add       <rig> <name>             (DEPRECATED → use identity add)
gt polecat remove    <rig>/<name>... | <rig>  [-f|--force] [--all]
gt polecat status    <rig>/<name>             [--json]
gt polecat git-state <rig>/<name>             [--json]
gt polecat check-recovery <rig>/<name>        [--json]
gt polecat gc        <rig>                    [--dry-run]
gt polecat nuke      <rig>/<name>... | <rig>  [--all] [--dry-run] [-f|--force]
gt polecat stale     <rig>                    [--threshold N] [--json] [--cleanup] [--dry-run]
gt polecat prune     <rig>                    [--dry-run] [--remote]
gt polecat pool-init <rig>                    [--dry-run] [--size N]

gt polecat identity add    <rig> [name]
gt polecat identity list   <rig>              [--json]
gt polecat identity show   <rig> <name>       [--json]
gt polecat identity rename <rig> <old> <new>
gt polecat identity remove <rig> <name>       [-f|--force]
```

(Additional subcommands exist in `polecat_spawn.go` and
`polecat_cycle.go`; see "Follow-up" in Notes.)

### Top-level subcommands

| subcommand | var | source | run-fn |
|---|---|---|---|
| `list`           | `polecatListCmd`          | `polecat.go:62-78`    | `runPolecatList`          (`polecat.go:423-545`) |
| `add` (DEPRECATED) | `polecatAddCmd`         | `polecat.go:80-96`    | `runPolecatAdd`           (`polecat.go:547-573`) |
| `remove`         | `polecatRemoveCmd`        | `polecat.go:98-114`   | `runPolecatRemove`        (`polecat.go:575-634`) |
| `status`         | `polecatStatusCmd`        | `polecat.go:116-136`  | `runPolecatStatus`        (`polecat.go:652-768`) |
| `git-state`      | `polecatGitStateCmd`      | `polecat.go:199-217`  | `runPolecatGitState`      (`polecat.go:793-860`) |
| `check-recovery` | `polecatCheckRecoveryCmd` | `polecat.go:219-237`  | `runPolecatCheckRecovery` (`polecat.go:958-1088`) |
| `gc`             | `polecatGCCmd`            | `polecat.go:150-168`  | `runPolecatGC`            (`polecat.go:1090-1154`) |
| `nuke`           | `polecatNukeCmd`          | `polecat.go:170-197`  | `runPolecatNuke`          (`polecat.go:1167-1253`) |
| `stale`          | `polecatStaleCmd`         | `polecat.go:248-271`  | `runPolecatStale`         (`polecat.go:1465-1578`) |
| `prune`          | `polecatPruneCmd`         | `polecat.go:273-295`  | `runPolecatPrune`         (`polecat.go:1580-1670`) |
| `pool-init`      | `polecatPoolInitCmd`      | `polecat.go:297-323`  | `runPolecatPoolInit`      (`polecat.go:1675-1777`) |

### `identity` sub-tree

Defined in
`/home/kimberly/repos/gastown/internal/cmd/polecat_identity.go`
and attached to `polecatCmd` at
`polecat_identity.go:158-159`.

| subcommand | var | source |
|---|---|---|
| `identity` (parent, alias `id`) | `polecatIdentityCmd` | `polecat_identity.go:31-40` |
| `identity add`    | `polecatIdentityAddCmd`    | `polecat_identity.go:42-61` |
| `identity list`   | `polecatIdentityListCmd`   | `polecat_identity.go:63-79` |
| `identity show`   | `polecatIdentityShowCmd`   | `polecat_identity.go:81-99` |
| `identity rename` | `polecatIdentityRenameCmd` | `polecat_identity.go:101-120` |
| `identity remove` | `polecatIdentityRemoveCmd` | `polecat_identity.go:122-139` |

`gt polecat identity add` replaces the deprecated
`gt polecat add` (`polecat.go:82-83`:
`Deprecated: "use 'gt polecat identity add' instead. This
command will be removed in v1.0."`). `runPolecatAdd` at
`polecat.go:547-551` emits a stderr deprecation warning before
doing anything else.

`gt polecat identity show` prints CV summary: session count,
issues completed/failed/abandoned, language breakdown from
file extensions, work type breakdown, recent work list
(per the Long help at `polecat_identity.go:84-96`).

`gt polecat identity rename` creates a new identity bead,
copies CV history links, closes the old bead with a reference
to the new one (`polecat_identity.go:104-110`). Refuses if the
session is running.

### `list`

`runPolecatList` (`polecat.go:423-545`). Lists polecats in one
rig or (with `--all`) all rigs. For each rig:

1. Create `polecat.Manager` and call `.List()` for filesystem-
   known polecats.
2. Track `knownNames` for zombie detection.
3. Discover **zombie tmux sessions** — tmux sessions matching
   the polecat naming convention that have no matching worktree
   on disk (`polecat.go:474-493`):

   > Zombie tmux sessions: sessions without matching worktree
   > directories. These occur when a worktree is deleted but
   > the tmux session persists (incomplete nuke or session
   > naming mismatch).

   Zombies are added as synthetic `PolecatListItem`s with
   `State: StateZombie, Zombie: true`.
4. Apply `effectivePolecatState` (`polecat.go:395-407`) to
   reconcile observed state with reported state:
   - Running session overrides `StateDone` and `StateIdle` →
     `StateWorking`.
   - No running session + `StateWorking` → `StateDone`.
   - Zombies are never rewritten.

JSON output is a slice of `PolecatListItem`:

```go
type PolecatListItem struct {
    Rig            string
    Name           string
    State          polecat.State
    Issue          string
    SessionRunning bool
    Zombie         bool
    SessionName    string
}
```

### `remove`

`runPolecatRemove` (`polecat.go:575-634`). Targets resolved via
`resolvePolecatTargets(args, polecatRemoveAll)` — shared
helper in `polecat_helpers.go`. For each target:

- Without `--force`: refuses if the session is running (tells
  the user to stop first or use `--force`).
- Calls `mgr.Remove(polecatName, force)`.
- Special-cases `polecat.ErrHasChanges` into a "has
  uncommitted changes (use --force)" error message.

Reports per-polecat errors in a summary block, returns an
error if any removal failed.

### `status`

`runPolecatStatus` (`polecat.go:652-768`). Shows lifecycle
state, issue, clone path, branch, and session info (running,
session ID, attached, windows, created, last activity). JSON
shape is `PolecatStatus` at `polecat.go:637-650`. Session info
comes from `polecat.SessionManager.Status` — if that fails,
falls back to a `SessionInfo{Running: false}` stub rather
than erroring (`polecat.go:674-679`).

`formatActivityTime` at `polecat.go:771-783` produces
`"N seconds ago"` / `"N minutes ago"` / `"N hours ago"` /
`"N days ago"` for the last-activity display.

### `git-state`

`runPolecatGitState` (`polecat.go:793-860`). Used by the
Witness for pre-kill verification. Inspects the worktree
directly via `exec.Command("git", ...)` at
`polecat.go:863-944`:

1. `git status --porcelain` — uncommitted files.
2. `git log origin/main..HEAD --oneline` (fallback to
   `origin/master` if `origin/main` is missing) — unpushed
   commits.
3. **Content-diff check** at `polecat.go:917-929`: if commits
   exist on HEAD that aren't on `origin/main`, run
   `git diff origin/main HEAD --quiet` — if exit code is 0
   (no diff), the work was **squash-merged** and is not
   really unpushed. The commit count is zeroed out. Otherwise,
   record the count as `UnpushedCommits`.
4. Stash count comes from `git.NewGit(worktreePath).StashCount`
   which filters by current branch (comment at
   `polecat.go:932-934`: without branch filtering, worktrees
   see repo-wide stashes and produce false NEEDS_RECOVERY
   verdicts).

Output shape is `GitState` at `polecat.go:786-791`. Verdict
line: `CLEAN (safe to kill)` if clean, else
`DIRTY (needs cleanup)`.

### `check-recovery`

`runPolecatCheckRecovery` (`polecat.go:958-1088`). Asks whether
a polecat is safe to nuke. Used by the Witness to decide
between nuke / recover / escalate. Produces `RecoveryStatus`
at `polecat.go:947-956` with one of three verdicts:

- `SAFE_TO_NUKE` — cleanup_status is clean AND work was
  submitted to the merge queue.
- `NEEDS_MQ_SUBMIT` — git is clean but no MR exists for the
  branch. The Long help at `polecat.go:226-227` says this
  occurs when a polecat crashed between push and MQ submission.
  Without this check, branches would be orphaned on remote.
  See #1035 (cross-ref at `polecat.go:1029-1031`).
- `NEEDS_RECOVERY` — cleanup_status indicates unpushed or
  uncommitted work. Should be escalated to the Mayor per the
  Long help at `polecat.go:229`.

The flow:

1. Fetch the agent bead via `beads.New(rigPath).GetAgentBead`.
2. **If no agent bead / no cleanup_status**, fall back to
   live git inspection via `getGitState(p.ClonePath)` and map
   git state to a verdict (`polecat.go:989-1013`).
3. **If agent bead has cleanup_status**, use it directly
   (`polecat.go:1014-1026`). Clean + no active MR →
   SAFE_TO_NUKE; anything else → NEEDS_RECOVERY.
4. **MQ check** at `polecat.go:1032-1046`: if the verdict is
   SAFE_TO_NUKE and the polecat has a branch, query
   `beads.New(r.Path).FindMRForBranchAny(branch)`. If no MR
   exists, downgrade to `NEEDS_MQ_SUBMIT`.

### `gc` — branch garbage collection

`runPolecatGC` (`polecat.go:1090-1154`). Deletes orphaned
`polecat/*` branches — branches for polecats that no longer
exist, plus old timestamped branches. With `--dry-run`, lists
what would be deleted vs. kept. Otherwise calls
`mgr.CleanupStaleBranches()`.

### `nuke` — the nuclear option

`runPolecatNuke` (`polecat.go:1167-1253`). The most complex
subcommand in this file. "Completely destroy a polecat and all
its artifacts" per the Long help at `polecat.go:173`.

Four steps per polecat, executed by `nukePolecatFull`
(`polecat.go:1261-1369`):

1. **Kill tmux session** unconditionally
   (`polecat.go:1264-1273`): "unconditionally to prevent ghost
   sessions when IsRunning fails to detect the session." Uses
   `sessMgr.Stop(polecatName, true)`. Tolerates
   `ErrSessionNotFound`.

2. **Burn attached molecules on the work bead** via
   `nukeCleanupMolecules` (`polecat.go:1282-1288`,
   implementation `1374-1430`): force-close descendants,
   detach with audit trail, remove the dependency bond, and
   force-close the molecule root. The comment at
   `polecat.go:1283-1285` explains the failure mode this
   prevents: "stale `attached_molecule` in the work bead's
   description causes sling to fail with 'bead already has N
   attached molecule(s)' on re-dispatch (gt-npzy)."

3. **Best-effort `git push`** of the branch before deleting
   (`polecat.go:1291-1317`): the "gt-4vr guardrail". Attempts
   to preserve unpushed commits. If push fails, proceeds —
   `--force` means "I accept data loss". Tries the worktree
   first, falls back to the bare repo at `<rig>/.repo.git`.

4. **Delete the worktree** via
   `mgr.RemoveWithOptions(name, nuclear=true, ...)`
   (`polecat.go:1319-1328`). Tolerates `ErrPolecatNotFound`.

5. **Delete local branch** — only local. The comment at
   `polecat.go:1331-1335` spells out the race:

   > Remote branch is never deleted during nuke — the refinery
   > owns remote branch cleanup after successful merge (gt mq
   > post-merge). This prevents the race where nuke deletes the
   > branch before the refinery has a chance to merge it.
   > (gt-v5ku)

6. **Reset agent bead for reuse** via
   `bd.ResetAgentBeadForReuse(agentBeadID, "nuked")`
   (`polecat.go:1346-1361`). Comment block explains why a
   plain `bd close` would not work: agent beads are ephemeral
   (wisps table), `bd close` hits the issues table, the row
   would persist and block re-sling with duplicate-key errors
   (gt--irj).

7. **Purge closed ephemeral beads** via
   `purgeClosedEphemeralBeads(bd)` (`polecat.go:1366`).
   "Without this, closed wisps from mol-polecat-work steps,
   mol-witness-patrol cycles, etc. accumulate across sessions
   and pollute bd ready/list (hq-6161m)."

Before running the four-step sequence, `runPolecatNuke`
applies **safety checks** via `checkPolecatSafety` unless
`--force` or `--dry-run` is set (`polecat.go:1179-1192`). The
checks refuse to nuke polecats with active work (uncommitted
changes, open merge requests, hooked work). Blocked polecats
produce a collective error after displaying the block table.

After all nukes, `cleanupOrphanedProcesses` is called
(`polecat.go:1244-1246`, implementation `1434-1463`) — this
kills any Claude processes that escaped session termination
via `util.CleanupZombieClaudeProcesses()`.

### `stale`

`runPolecatStale` (`polecat.go:1465-1578`). Detects stale
polecats and optionally nukes them. "Stale" means:

- No active tmux session
- Behind main by > `--threshold` commits (default 20) OR has
  no agent bead
- No uncommitted work that could be lost

Implementation delegates detection to
`mgr.DetectStalePolecats(threshold)` which returns
`StaleInfo` records. Display shows session status, commits
behind, agent state, uncommitted yes/no, and a reason string.

With `--cleanup`, calls `nukePolecatFull` on each stale
polecat, honoring `--dry-run`. Notably, `--cleanup` does NOT
go through `runPolecatNuke`, so it does NOT apply
`checkPolecatSafety` — the stale detection itself is
responsible for filtering out polecats with at-risk work.

### `prune`

`runPolecatPrune` (`polecat.go:1580-1670`). Prunes stale
`polecat/*` branches locally (always) and optionally remote
(with `--remote`). Implementation:

1. Pick the repo for branch operations: prefer
   `<rig>/.repo.git` bare repo if it exists, otherwise
   `<rig>/mayor/rig` worktree (`polecat.go:1588-1595`).
2. `repoGit.FetchPrune("origin")` — prune stale remote-
   tracking refs first, non-fatal on failure.
3. `repoGit.PruneStaleBranches("polecat/*", dryRun)` — local
   pruning of merged/orphan branches.
4. With `--remote`: list remote refs under
   `refs/heads/polecat/`, check each for ancestry with
   `origin/<defaultBranch>`, delete via
   `repoGit.DeleteRemoteBranch("origin", branch)` if merged.

### `pool-init`

`runPolecatPoolInit` (`polecat.go:1675-1777`). Initializes a
persistent polecat pool — N polecats created in IDLE state,
ready for immediate work assignment via `gt sling`.

Pool size priority (`polecat.go:1683-1691`):
1. `--size` flag.
2. `polecat_pool_size` in rig `config.json`.
3. Default 4.

Names priority (`polecat.go:1693-1697, 1716-1739`):
1. `polecat_names` in rig `config.json`, skipping existing.
2. Allocated from the rig's name pool (theme-based, e.g.
   mad-max) via `mgr.GetNamePool().Allocate()`.

Existing polecats are preserved — only new ones are created
to reach the target pool size.

Post-creation, each new polecat has its agent state explicitly
set to `idle` via `mgr.SetAgentState(name, "idle")`
(`polecat.go:1764-1770`).

### Flags

| flag | type | default | scope | source |
|---|---|---|---|---|
| `--json`             | bool | `false` | `list`, `status`, `git-state`, `check-recovery`, `stale`, plus identity list/show | `polecat.go:327,335,338,349,352`, `polecat_identity.go:143,146` |
| `--all`              | bool | `false` | `list`, `remove`, `nuke` | `polecat.go:328,332,344` |
| `--force` / `-f`     | bool | `false` | `remove`, `nuke`, identity remove | `polecat.go:331,346`, `polecat_identity.go:149` |
| `--dry-run`          | bool | `false` | `gc`, `nuke`, `stale` (+ `--cleanup`), `prune`, `pool-init` | `polecat.go:341,345,355,358,362` |
| `--threshold <n>`    | int  | `20`    | `stale` | `polecat.go:353` |
| `--cleanup`          | bool | `false` | `stale` | `polecat.go:354` |
| `--remote`           | bool | `false` | `prune` | `polecat.go:359` |
| `--size <n>`         | int  | `0`     | `pool-init` | `polecat.go:363` |

### Shared helpers and types

- `effectivePolecatState` — `polecat.go:395-407`. Running
  session overrides reported state.
- `getPolecatManager(rigName)` — `polecat.go:410-421`.
- `nukePolecatFull` — `polecat.go:1261-1369`. The canonical
  cleanup path, shared by `nuke` and `stale --cleanup`.
- `nukeCleanupMolecules` — `polecat.go:1374-1430`. Molecule
  burn during nuke.
- `cleanupOrphanedProcesses` — `polecat.go:1434-1463`. Post-
  nuke zombie Claude process kill.
- `getGitState` — `polecat.go:863-944`. Shared between
  `git-state` and `check-recovery`.
- `polecatBeadIDForRig`, `resolvePolecatTargets`,
  `checkPolecatSafety`, `displaySafetyCheckBlocked`,
  `displayDryRunSafetyCheck`, `findRigPolecatSessions`,
  `parsePolecatSessionName`, `forceCloseDescendants`,
  `purgeClosedEphemeralBeads`, `getRepoGitForRig` — defined in
  `polecat_helpers.go` and friends, not enumerated here.
- `PolecatListItem` — `polecat.go:382-390`.
- `PolecatStatus` — `polecat.go:637-650`.
- `GitState` — `polecat.go:786-791`.
- `RecoveryStatus` — `polecat.go:947-956`.

## Related commands

- [../binaries/gt.md](../binaries/gt.md) — root.
- [README.md](README.md) — command index.
- [session.md](session.md) — the underlying tmux session
  surface. `gt polecat list` shows both polecat identities
  and their session state via `polecat.SessionManager`.
- [witness.md](witness.md) — the per-rig Witness is the
  primary consumer of `git-state`, `check-recovery`, and
  `nuke` during its patrol cycle.
- [refinery.md](refinery.md) — the Refinery owns the
  post-merge remote-branch cleanup that `nuke` explicitly
  leaves alone (`polecat.go:1331-1335`).
- [mayor.md](mayor.md) — escalation target for NEEDS_RECOVERY
  verdicts from `check-recovery`.
- [deacon.md](deacon.md) — the Deacon orchestrates the
  rig-level agents that call these commands. `gt deacon
  force-kill` is a higher-level "nuke this agent" path.
- [done.md](done.md) — `gt done` is the polecat-side completion
  path. It pushes the branch and submits to the MQ; the
  Witness then runs `gt polecat nuke` (or the Refinery owns the
  remote branch cleanup).
- [sling.md](sling.md) — `gt sling` dispatches work to
  polecats. Stale attached molecules block re-sling — the
  `nukeCleanupMolecules` path exists to unblock that.
- [hook.md](hook.md) — slung work lands on the polecat's hook.
  `check-recovery` references the hook bead via the agent
  bead.
- [mq.md](mq.md) — merge queue; `check-recovery` queries it
  via `FindMRForBranchAny`.
- [molecule.md](molecule.md) — `mol-polecat-work`,
  `mol-witness-patrol`, and other molecules are what create
  the ephemeral wisps `purgeClosedEphemeralBeads` cleans up.
- [convoy.md](convoy.md) — work batching upstream of sling.
  Stranded convoys feed polecats.
- [handoff.md](handoff.md) — polecat-side session handoff.
- [prune-branches.md](prune-branches.md) — town-level
  branch pruning. `gt polecat prune` is the polecat-scoped
  equivalent.
- [stale.md](stale.md) — town-level stale detection. `gt
  polecat stale` is the polecat-scoped equivalent.
- [cleanup.md](cleanup.md) — general cleanup umbrella command.
- [callbacks.md](callbacks.md) — run during Deacon patrol;
  indirect sibling.
- [heartbeat.md](heartbeat.md), [seance.md](seance.md),
  [resume.md](resume.md), [prime.md](prime.md) — session-adjacent
  commands polecats run during their lifecycle.
- [agents.md](agents.md) — `gt agents list` enumerates
  polecats using the same identity beads this command
  manages.

## Notes / open questions

- **This file is 1786 lines.** The largest CLI file in the
  Agents group. Six command files in the package share the
  `polecat_*` prefix (`polecat.go`, `polecat_identity.go`,
  `polecat_spawn.go`, `polecat_cycle.go`, `polecat_helpers.go`,
  plus tests). A follow-up read of `polecat_spawn.go` and
  `polecat_cycle.go` is warranted — they register additional
  subcommands that this page does not enumerate yet.
- **`gt polecat add` is deprecated but still there.** The
  deprecation warning prints to stderr (`polecat.go:547-551`),
  and the Long help explicitly says "will be removed in v1.0"
  (`polecat.go:83`).
- **Nuke is surprisingly gentle to the remote.** The remote
  branch is never touched — the comment at
  `polecat.go:1331-1335` is explicit about the race with the
  Refinery. Operators conditioned on `git branch -D` actually
  deleting the remote branch will be surprised.
- **`stale --cleanup` bypasses `checkPolecatSafety`.** It
  calls `nukePolecatFull` directly (`polecat.go:1564`). The
  safety of this relies on `mgr.DetectStalePolecats` having
  already filtered out polecats with at-risk work. If
  detection is buggy, stale-cleanup can silently lose work.
- **The MQ check in `check-recovery` is a post-hoc
  override.** The bead's cleanup_status can say "clean" and
  the polecat can still need recovery because its branch was
  never submitted — see the gt-1035 cross-ref at
  `polecat.go:1029-1031`. Anyone grepping for "SAFE_TO_NUKE"
  in logs must understand the verdict can still be
  overridden by the MQ check.
- **Zombie session detection** in `list` is filesystem-vs-
  tmux. A polecat that exists on disk but has no tmux session
  is NOT a zombie; a tmux session with no on-disk worktree
  IS. The asymmetry is intentional — the former is the
  post-nuke steady state.
- **`pool-init` sets agent state to "idle"** — but the Long
  help for the parent polecat command at `polecat.go:44-48`
  does not list "idle" as one of the four states. This is
  the pool-initialization state, distinct from the working/
  stalled/zombie/nuked lifecycle. Worth a note on a future
  polecat role/concept page.
- **Orphan process cleanup is silent on success.** If zero
  orphaned processes were killed, `cleanupOrphanedProcesses`
  prints nothing (`polecat.go:1442-1444`). The Witness
  relies on this to avoid noise in routine nukes.
- **Role/persona page pending Batch 6.** The polecat as a
  persona (its prompts, its self-cleaning model, its
  interaction with sling/done/handoff, what "Cats build
  features. Dogs clean up messes." actually means) lives in
  the Long help and molecules — not in this command file.
