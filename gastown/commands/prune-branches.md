---
title: gt prune-branches
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/cmd/prune_branches.go
  - /home/kimberly/repos/gastown/internal/git/
tags: [command, work, git, branch, hygiene, polecat]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase5_audience: dev
phase8_audited: 2026-04-17
phase8_findings: [none]
---

# gt prune-branches

Delete stale local polecat tracking branches — ones fully merged to
the default branch (`main`) or whose remote tracking ref has been
deleted. Thin wrapper around `internal/git.PruneStaleBranches`.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupWork` ("Work Management") (`prune_branches.go:18`)
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/prune_branches.go`
(full file, 95 lines).

### Invocation

```
gt prune-branches [--dry-run] [--pattern "<glob>"]
```

### Behavior (`runPruneBranches`, `prune_branches.go:49-95`)

1. Construct a `gitpkg.Git(".")` (`:50`); bail out if not a git repo
   (`:51-53`).
2. Run `git fetch --prune origin` (`:56`). Failure is non-fatal —
   prints a warning and continues.
3. Call `g.PruneStaleBranches(pattern, dryRun)` (`:61`, implementation
   in `internal/git/`). Returns a list of pruned branches with a
   reason code.
4. If none, print `✓ No stale branches found matching <pattern>`
   (`:66-69`).
5. Otherwise print a bulleted list with reason labels (`:71-91`):
    - `merged` → "merged to main"
    - `no-remote` → "remote branch deleted"
    - `no-remote-merged` → "remote deleted, merged to main"

Safety property (from the Long text at `:32-33`): the underlying
implementation uses `git branch -d` (lowercase, merge-check) not
`-D`, so partially-merged branches are never deleted. The code also
never touches the current branch or the default branch.

### Flags

| flag | default | notes |
|---|---|---|
| `--dry-run` | `false` | Print what would be deleted without acting. |
| `--pattern <glob>` | `"polecat/*"` | Branch-name glob passed to the pruner. |

### Subcommands

None (terminal command).

## Notes / open questions

- **Thin surface, real logic in `internal/git`.** The interesting
  pruning logic — glob matching, "is-merged?" resolution against the
  default branch, `no-remote` detection, `-d` safety — lives in
  `internal/git.PruneStaleBranches`. A dedicated `packages/git.md`
  would be a good place to document the three reason codes as a
  taxonomy.
- **Related to orphan handling but distinct.** This command cleans
  up branches that are *safely disposable*. Compare with
  [orphans](orphans.md), which finds unmerged polecat branches that
  still contain work. Between the two, a typical flow is:
  1. `gt orphans` — rescue unmerged work.
  2. `gt prune-branches` — delete merged/dead tracking branches.
  3. [cleanup](cleanup.md) — broader polecat dir cleanup.
- **Fixed default branch.** The Long text (`:29`) says "fully
  merged to the default branch (main)". If `PruneStaleBranches`
  hardcodes `main` rather than reading the rig's configured
  default, this would break on non-main-default rigs.
- **Non-polecat usage.** `--pattern` is overrideable, so
  `gt prune-branches --pattern "feature/*"` cleans up regular
  feature branches too. Worth noting that the command's default
  framing is polecat-specific but the mechanism is generic.
