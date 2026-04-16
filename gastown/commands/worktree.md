---
title: gt worktree
type: command
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-15
sources:
  - /home/kimberly/repos/gastown/internal/cmd/worktree.go
tags: [command, workspace, git, worktree, crew, cross-rig]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
---

# gt worktree

Create, list, and remove git worktrees **for cross-rig work by a
crew member**. Lets `gastown/crew/joe` open a working copy of the
`beads` rig's main branch without losing their source identity.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupWorkspace` ("Workspace") (`worktree.go:25`)
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/worktree.go`
(348 lines). Registration: `worktree.go:87-95` attaches two
subcommands (`list`, `remove`) to `worktreeCmd` and registers the
parent on `rootCmd`.

The parent `worktreeCmd` takes exactly one positional arg (the
target rig name) and runs `runWorktree` — so `gt worktree <rig>`
is itself a functional command, not just a grouping. The two
subcommands (`list`, `remove`) live under it.

### Invocation

```
gt worktree <rig> [--no-cd]
gt worktree list
gt worktree remove <rig> [--force|-f]
```

### Subcommands

| subcommand | var | source | run-fn |
|---|---|---|---|
| (bare)   | `worktreeCmd`       | `worktree.go:23-48`  | `runWorktree`       (`worktree.go:97-202`) |
| `list`   | `worktreeListCmd`   | `worktree.go:50-65`  | `runWorktreeList`   (`worktree.go:210-274`) |
| `remove` | `worktreeRemoveCmd` | `worktree.go:72-85`  | `runWorktreeRemove` (`worktree.go:296-348`) |

### `gt worktree <rig>` — create

`runWorktree` (`worktree.go:97-202`):

1. **Detect crew identity from cwd** via `detectCrewFromCwd()`
   (defined in the crew sibling files; referenced at
   `worktree.go:101`). Must be inside a crew workspace, else
   `"must be in a crew workspace to use this command"`.
2. **Reject self-targeting** (`worktree.go:110-112`). Can't make a
   cross-rig worktree to your own rig.
3. **Verify target rig** via `getRig(targetRig)` — from the rig
   sibling files; aborts with a "not found — run 'gt rig list'"
   hint if absent.
4. **Compute worktree path:**
   `<target-rig>/crew/<source-rig>-<crew-name>/`
   (`worktree.go:120-122`), using `constants.RigCrewPath`. If the
   worktree already exists, prints its location (and a `cd`
   command unless `--no-cd`) and exits successfully.
5. **Create from the target rig's mayor clone.**
   `targetMayorRig := constants.RigMayorPath(targetRigInfo.Path)`
   (`worktree.go:139`) — the Mayor's working clone is the bare-ish
   repo that backs all worktrees for that rig. `os.MkdirAll`
   ensures `crew/` exists.
6. **Fetch from origin** (`worktree.go:149-152`). Non-fatal; prints
   a warning and continues if it fails.
7. **`WorktreeAddExistingForce(worktreePath, "main")`**
   (`worktree.go:157`). Uses the `-force` variant because `main`
   may already be checked out in other worktrees (e.g. the mayor
   rig clone). Comment: "This is safe for cross-rig work."
8. **Set local git user.name** in the new worktree via
   `setGitConfig` → `git -C <path> config user.name
   "<source-rig>/crew/<crew-name>"` (`worktree.go:161-168,
   205-208`). Non-fatal on failure.
9. **Pull latest main** in the new worktree (`worktree.go:177-179`)
   — also non-fatal.
10. **Print identity-preserving env vars.**
    `runWorktree` emits `export BD_ACTOR=<source-rig>/crew/<name>`,
    `GT_ROLE=crew`, `GT_RIG=<source-rig>`, `GT_CREW=<name>` —
    with a PowerShell variant (`$env:...`) when `runtime.GOOS ==
    "windows"` (`worktree.go:188-198`). This is a rare explicit
    Windows-path in the gt CLI. **`gt worktree` does not set
    these variables automatically** — it prints them for the
    user to paste.

### `gt worktree list`

`runWorktreeList` (`worktree.go:210-274`):

1. Detects crew identity from cwd (same precondition).
2. `townRoot := workspace.FindFromCwdOrError()`.
3. Loads `mayor/rigs.json` via `config.LoadRigsConfig`
   (`worktree.go:228-232`).
4. For each rig other than the source rig, checks whether
   `<rig>/crew/<source-rig>-<crew>/` exists. If yes, prints its
   path (abbreviated with `~/` when possible) and a git status
   summary via `getGitStatusSummary` (`worktree.go:277-294`).
5. Status summary is one of `clean`, `N uncommitted`, or `error`.

If no worktrees exist, prints `"(none)"` and a creation hint.

### `gt worktree remove <rig>`

`runWorktreeRemove` (`worktree.go:296-348`):

1. Detects crew identity, verifies target rig exists, recomputes
   the path (same rules).
2. Aborts if the worktree does not exist.
3. Unless `--force`, checks git status summary and refuses if
   uncommitted work is present (`worktree.go:328-333`).
4. Calls `g.WorktreeRemove(worktreePath, worktreeRemoveForce)`
   against the target rig's mayor clone (`worktree.go:337-343`).

### Flags

| flag | type | default | scope | source |
|---|---|---|---|---|
| `--no-cd` | bool | `false` | `worktree` (bare) | `worktree.go:88` |
| `--force` / `-f` | bool | `false` | `remove` | `worktree.go:91` |

## Related commands

- [crew.md](crew.md) — the noun this command serves. A crew
  workspace is the only valid cwd for any of these subcommands.
- [rig.md](rig.md) — target-rig lookup via `getRig`; `gt rig list`
  is the suggested remediation in error messages.
- [prune-branches.md](prune-branches.md) — general
  branch/worktree hygiene.
- [orphans.md](orphans.md) — cleanup of stale worktrees.

## Notes / open questions

- **Source rig == target rig's mayor clone.** The comment at
  `worktree.go:137-139` ("For cross-rig work, we need to use the
  target rig's repository") is the key insight: the **target**
  rig's main git repo is where the worktree is added. This means
  worktrees share refs with the target rig's Mayor and anything
  else working against that clone — if someone on the Mayor side
  forces a branch, crew worktrees tracking it get affected.
- **Env vars are printed, not set.** `gt worktree` does not
  actually set `BD_ACTOR` in the new shell — it assumes the user
  will paste the exported variables. For non-interactive use
  (scripts, other agents) this is a footgun: the worktree exists
  but no shell has the identity environment. Worth a dedicated
  workflow page once one lands.
- **No "move crew" subcommand.** The worktree is tied to a
  specific crew name at creation time; there is no rename or
  transfer operation. A rename via [crew.md](crew.md) `rename`
  would leave worktree paths stale.
- **`--force` on `remove` gates two things.** It skips the
  uncommitted-work check AND passes `force=true` to the
  underlying `git worktree remove` (the second `bool` parameter
  on `WorktreeRemove`, `worktree.go:341`). No way to skip one
  without the other.
- **Only `main` is supported.** The worktree is always created
  on `main` (`worktree.go:157`). There is no `--branch` flag.
  Cross-rig branch work requires switching manually after creation.
