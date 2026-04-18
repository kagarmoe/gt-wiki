---
title: gt orphans
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/cmd/orphans.go
  - /home/kimberly/repos/gastown/internal/cmd/prune_branches.go
tags: [command, work, git, recovery, polecat, worktree, process]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
phase5_audience: dev
phase8_audited: 2026-04-17
phase8_findings: [none]
---

# gt orphans

Find lost polecat work — orphaned git commits (unreachable via
`git fsck`) plus unmerged polecat worktree branches — and optionally
clean them up. Also has a `procs` subtree for finding and killing
orphaned Claude CLI processes (PPID=1 or not-in-any-tmux-session).

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupWork` ("Work Management") (`orphans.go:26`)
**Polecat-safe:** no (no `AnnotationPolecatSafe` annotation)
**Beads-exempt:** no
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/orphans.go` (1027
lines). This page documents the command surface; implementation
helpers for `findOrphanCommits`, `findOrphanPolecatBranches`,
process-scanning logic, and the `git gc --prune=now` path live in
the same file but aren't walked here.

### Invocation

```
gt orphans [--days N] [--all] [--rig NAME]
gt orphans kill [flags]
gt orphans procs [list|kill] [flags]
```

### Default scan: `runOrphans` (`orphans.go:192-296`)

1. **Find town root + rig.** `workspace.FindFromCwdOrError`, then
   resolve rig via `--rig` flag or `findCurrentRig(townRoot)`
   (`:199-213`). Error out if neither.
2. **Scan orphaned commits.** Runs against `mayorPath = r.Path +
   "/mayor/rig"` — the mayor's canonical clone, not the polecat
   worktree. Uses `findOrphanCommits(mayorPath)` which shells to
   `git fsck --unreachable` and parses commit hashes, dates, authors
   (`:219-247`). Filtered by `--days` cutoff unless `--all`.
3. **Scan unmerged polecat branches.** `findOrphanPolecatBranches`
   (`:337+`, summary here): walks `rig.Path/polecats/<name>/<rig>`
   (new structure) or `rig.Path/polecats/<name>` (old backward-compat),
   resolves each worktree, computes commits ahead of the default
   branch, and reports `OrphanBranch{Polecat, Branch, AheadCount,
   LatestSubject, HasUncommitted, WorktreePath}` (`:298-306`).
4. **Report.** Orphan commits print with `git show/cherry-pick/branch
   rescue` recovery hints (`:244-247`). Polecat branches print with
   `cd <path>; git log <default>..HEAD` hints (`:273-275`). Skipped
   polecats (due to scan errors) are listed separately (`:278-284`).

The `--days`/`--all` flags only apply to commits — polecat branches
are always shown (`:39-40`).

### Subcommands

Registered at `orphans.go:174-181`:

| subcommand | source | one-liner |
|---|---|---|
| `kill` | `:66-95`, `runOrphansKill` | Destructive: `git gc --prune=now` to drop orphaned commits, plus kill orphaned Claude processes. Requires confirmation unless `--force`. |
| `procs` | `:98-117` (default = list) | Manage orphaned Claude CLI processes. `RunE = runOrphansListProcesses`. |
| `procs list` | `:119-139` | List Claude processes with `PPID=1`. With `--aggressive`, cross-references against active tmux sessions and flags any Claude process not in a `gt-*` or `hq-*` session. |
| `procs kill` | `:141-155` | Kill listed processes. `-f/--force` skips confirmation. Excludes tmux server and `Claude.app` desktop binaries. |

### Flags

On `orphansCmd` (`:157-160`):

| flag | default | notes |
|---|---|---|
| `--days <n>` | `7` | Orphaned-commit date filter. |
| `--all` | `false` | No date filter for commits. |
| `--rig <name>` | `""` | **Persistent** — applies to subcommands too. Required when not in a rig dir. |

On `orphansKillCmd` (`:163-166`): `--dry-run`, `--days`, `--all`,
`--force`.

On `orphansProcsCmd` (persistent, `:172`): `--aggressive`.

On `orphansProcsKillCmd` (`:169`): `-f/--force`.

## Notes / open questions

- **Destructive `kill` is not reversible.** The inline warning
  (`:73-75`) and 1027-line file both emphasize: once commits are
  pruned via `git gc --prune=now`, they cannot be recovered. Users
  should prefer `git branch rescue <sha>` before running `kill`.
- **"Orphan" is overloaded in Gas Town.** This command is about
  git/process orphans. There is a distinct concept of orphan Dolt
  databases (testdb_*, etc.) handled by `gt dolt cleanup` per the
  project CLAUDE.md — not related.
- **Two-axis cleanup.** Between this command and
  [prune-branches](prune-branches.md), the branch/worktree hygiene
  story is split: `orphans` targets unmerged-but-still-alive polecat
  work, while `prune-branches` targets merged or remote-deleted
  tracking branches. See [cleanup](cleanup.md) for broader polecat
  directory cleanup.
- **Aggressive tmux correlation** (`--aggressive`) is a heuristic
  assuming all legitimate Claude processes live in a `gt-*` or
  `hq-*` tmux session. Users running `claude` manually outside a
  tmux session will be flagged as orphans.
- **Mayor-path assumption** (`:216`) — `mayorPath := r.Path +
  "/mayor/rig"` hardcodes the mayor layout. If the mayor rig
  structure changes, this break silently.
