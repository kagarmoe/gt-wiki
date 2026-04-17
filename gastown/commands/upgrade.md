---
title: gt upgrade
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/cmd/upgrade.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, diagnostics, migration, post-install, claude-md, daemon, formulas, hooks]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase5_audience: dev
phase8_audited: 2026-04-17
phase8_findings: [partial-completion]
---

# gt upgrade

Post-install migration orchestrator: runs `doctor --fix` over a
subset of workspace-structural checks (driven by the
[doctor package](../packages/doctor.md)), syncs `CLAUDE.md` from an
embedded template, ensures daemon lifecycle defaults, syncs the hook
registry into `settings.json` files, and updates formulas from
embedded copies.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupDiag` ("Diagnostics")
**Polecat-safe:** no (no `AnnotationPolecatSafe` on the cobra.Command at
`/home/kimberly/repos/gastown/internal/cmd/upgrade.go:25-49`)
**Beads-exempt:** **yes** — in `beadsExemptCommands` at
`/home/kimberly/repos/gastown/internal/cmd/root.go:75`; upgrade must
run even when `bd` is broken, because an upgrade may include the fix
for `bd`.
**Branch-check-exempt:** **yes** — in `branchCheckExemptCommands` at
`root.go:90`; upgrade is a prerequisite for fixing branch drift so
the warning would be unhelpful.

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/upgrade.go:66-504`.

### Invocation

```
gt upgrade [--dry-run] [-v|--verbose] [--no-start]
```

### Behavior

`runUpgrade` (`upgrade.go:66-104`):

1. Resolves town root via `workspace.FindFromCwdOrError`.
2. Prints a header (`Post-install migration` or `Dry run — showing
   what would change`).
3. Runs five steps in order, collecting an `upgradeResult` from each
   (`upgrade.go:80-98`):
   1. `upgradeDoctor(townRoot)` — structural checks via `doctor --fix`
   2. `upgradeCLAUDEMD(townRoot)` — sync `CLAUDE.md` from embedded template
   3. `upgradeDaemonConfig(townRoot)` — ensure `daemon.json` defaults
   4. `upgradeHooksSync(townRoot)` — regenerate `settings.json` from hooks
   5. `upgradeFormulas(townRoot)` — update formulas from embedded copies
4. Calls `printUpgradeSummary(results)` which sums `changed` counts
   and lists any per-step issues (`upgrade.go:466-504`).

`runUpgrade` always returns `nil` — per-step failures are captured
in `upgradeResult.details` and printed in the summary instead of
aborting.

### Step 1: Structural checks (`upgradeDoctor`)

Source: `upgrade.go:107-178`.

Creates a `doctor.CheckContext` with the upgrade's verbose flag and
`NoStart` flag. Creates a `doctor.NewDoctor()` and registers a
curated subset of checks — **not the full doctor suite**. Registered
checks (in order at `upgrade.go:121-157`):

- `doctor.WorkspaceChecks()` (batch)
- `NewGlobalStateCheck`
- Infrastructure prereqs: `NewStaleBinaryCheck`, `NewBeadsBinaryCheck`,
  `NewDoltBinaryCheck`, `NewDoltServerReachableCheck`
- Town root protection: `NewTownGitCheck`, `NewTownRootBranchCheck`,
  `NewPreCheckoutHookCheck`
- Runtime: `NewClaudeSettingsCheck`, `NewDaemonCheck`,
  `NewTownBeadsConfigCheck`, `NewCustomTypesCheck`,
  `NewCustomStatusesCheck`, `NewFormulaCheck`
- Prefix/database: `NewPrefixConflictCheck`, `NewPrefixMismatchCheck`,
  `NewDatabasePrefixCheck`
- Routing: `NewRoutesCheck`
- Config architecture: `NewSettingsCheck`, `NewSessionHookCheck`,
  `NewDeprecatedMergeQueueKeysCheck`
- Hooks sync: `NewStaleTaskDispatchCheck`, `NewHooksSyncCheck`
- Dolt data health: `NewStaleDoltPortCheck`,
  `NewStaleSQLServerInfoCheck`
- Migration: `NewSparseCheckoutCheck`, `NewPrimingCheck`
- Lifecycle hygiene: `NewLifecycleHygieneCheck`
- Worktree: `NewWorktreeGitdirCheck`
- **Identity bead repair** (GH#2766 comment at `upgrade.go:152-154`):
  `NewAgentBeadsCheck`, `NewRigBeadsCheck`, `NewRoleBeadsCheck`.
  These were previously omitted, which meant `gt doctor --fix` could
  repair identity gaps but `gt upgrade` could not. This subset is
  load-bearing.

In dry-run mode, runs `d.RunStreaming(ctx, os.Stdout, 0)`; otherwise
runs `d.FixStreaming` (`upgrade.go:160-164`). Returns the report's
`Fixed` count as `result.changed`.

### Step 2: CLAUDE.md sync (`upgradeCLAUDEMD`)

Source: `upgrade.go:181-236`.

1. Generates the expected content via `generateCLAUDEMD`
   (`upgrade.go:240-251`) — a small hard-coded string that uses
   `cli.Name()` for the command name so it works for any alias the
   binary is installed under.
2. Reads `<townRoot>/CLAUDE.md`. If content matches → "up-to-date"
   and returns.
3. In dry-run mode → logs "would create" or "would update".
4. Otherwise writes `CLAUDE.md` at mode `0644`.
5. Creates `AGENTS.md` as a symlink to `CLAUDE.md` if missing
   (`upgrade.go:225-232`).

**This step is the ground truth for the town root `CLAUDE.md`
content.** Any manual edit to `CLAUDE.md` will be overwritten on the
next upgrade. `generateCLAUDEMD` is a comment-pinned mirror of
`createTownRootAgentMDs` in `install.go` (`upgrade.go:239`).

### Step 3: Daemon config (`upgradeDaemonConfig`)

Source: `upgrade.go:254-296`.

1. Resolves `config.DaemonPatrolConfigPath(townRoot)`.
2. If file exists: validates it loads via
   `config.LoadDaemonPatrolConfig`. Invalid → warning; valid →
   "present and valid" no-op.
3. If missing: in dry-run mode "would create with defaults";
   otherwise calls `config.EnsureDaemonPatrolConfig(townRoot)`.

### Step 4: Hooks sync (`upgradeHooksSync`)

Source: `upgrade.go:299-386`.

1. Calls `hooks.DiscoverTargets(townRoot)` to find all target
   `settings.json` files (town + rig + per-agent).
2. For each target, calls `syncTarget(target, dryRun)` (defined
   elsewhere in the cmd package) which returns a `syncCreated /
   syncUpdated / syncUnchanged` enum and an error.
3. Tallies counts and prints a summary like
   `settings.json: 3 updated, 1 created, 12 unchanged`.
4. Errors per target are counted into `result.errors` and, if
   verbose, printed with `relPath`.

### Step 5: Formulas (`upgradeFormulas`)

Source: `upgrade.go:389-463`.

1. **Dry-run mode**: calls `formula.CheckFormulaHealth(townRoot)`
   and reports outdated/missing/new/modified counts without
   touching disk.
2. **Normal mode**: calls `formula.UpdateFormulas(townRoot)` which
   returns `(updated, skipped, reinstalled, err)`. `skipped`
   typically means locally-modified formulas that the upgrader
   refuses to overwrite. Prints a summary line.

### Summary

`printUpgradeSummary` (`upgrade.go:466-504`):

- Sums `totalChanged` across all steps.
- Collects any `details` containing "error" as issues to highlight.
- Prints:
  - Dry run with 0 changes → "Workspace is up-to-date — nothing to
    change"
  - Dry run with N changes → "Dry run complete — N change(s) would
    be applied"
  - Normal with 0 → "Workspace is up-to-date"
  - Normal with N → "Upgrade complete — N change(s) applied"
- Finally lists any issues under an "Issues:" header.

### Subcommands

None (terminal command).

### Flags

Defined in `init()` (`upgrade.go:51-56`):

| Flag              | Type | Default | Purpose                                          |
|-------------------|------|---------|--------------------------------------------------|
| `--dry-run`       | bool | `false` | Show what would change without modifying        |
| `--verbose` / `-v`| bool | `false` | Verbose output per step                          |
| `--no-start`      | bool | `false` | Suppress starting daemon/agents during doctor fix|

`SilenceUsage: true` (`upgrade.go:48`) — errors don't dump cobra
usage text.

## Related

- [doctor](doctor.md) — upgrade's Step 1 is effectively a narrower
  `doctor --fix`. The full doctor suite at `doctor.go:154-275` has
  more checks than upgrade registers, so `doctor --fix` is strictly
  more thorough.
- [stale](stale.md) — `NewStaleBinaryCheck` is the first
  infrastructure check upgrade runs; users typically rebuild then
  run `gt upgrade` to pick up migrations.
- [info](info.md) — reports current version / build; before-and-after
  pair with upgrade.
- [version](version.md) — the build-time identity the upgrade
  targets; after upgrade runs, `gt version` should match the repo
  HEAD again.
- [../files/makefile.md](../files/makefile.md) — `make install`
  copies the binary; `gt upgrade` then syncs workspace state to it.
- [../binaries/gt.md](../binaries/gt.md) — parent binary; documents
  upgrade's double exemption.
- [README.md](README.md) — command tree index.

## Failure modes

### Partial completion (what doesn't it clean up?)
- **Step N failure does not stop step N+1:** `runUpgrade` at
  `upgrade.go:66-104` runs all 5 steps sequentially but each step
  returns an `upgradeResult` — not an error. Failures within a step
  are captured in `result.details` but the function always returns
  `nil`. If step 1 (doctor --fix) fails to start the daemon, step 4
  (hooks sync) may target stale settings files. **Absent** — no
  dependency tracking between steps.
- **Error reporting is summary-only:** errors from individual steps
  are collected and printed in the summary at `upgrade.go:466-498`,
  but the exit code is always zero. Automation that relies on exit
  code to detect upgrade failures will miss them. **Absent.**

## Notes / open questions

- `doctor` registers ~80 checks (`doctor.go:154-275`); upgrade
  registers roughly half. The selection criterion is "post-install
  structural" vs. "runtime health", but the line is not explicit.
- `generateCLAUDEMD` must be kept in sync with `install.go`'s
  `createTownRootAgentMDs` (comment at `upgrade.go:239`). Drift is
  possible and silent — the template content is pinned as a Go
  string literal in two places.
- `hooks.DiscoverTargets`, `syncTarget`, `syncCreated`/`syncUpdated`/
  `syncUnchanged` all live in `internal/hooks/` and the cmd package.
  A dedicated hooks-sync page would document the target discovery
  rules.
- `formula.UpdateFormulas` decides which formulas are "modified"
  (skipped) by comparing against embedded copies. The decision
  rule deserves its own page.
- No `--step` flag to run a single stage — upgrade is all-or-nothing.
