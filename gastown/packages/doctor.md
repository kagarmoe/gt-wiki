---
title: internal/doctor
type: package
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-14
sources:
  - /home/kimberly/repos/gastown/internal/doctor/doctor.go
  - /home/kimberly/repos/gastown/internal/doctor/types.go
  - /home/kimberly/repos/gastown/internal/doctor/errors.go
  - /home/kimberly/repos/gastown/internal/doctor/workspace_check.go
  - /home/kimberly/repos/gastown/internal/doctor/rig_check.go
  - /home/kimberly/repos/gastown/internal/doctor/claude_settings_check.go
  - /home/kimberly/repos/gastown/internal/doctor/stale_binary_check.go
  - /home/kimberly/repos/gastown/internal/doctor/daemon_check.go
  - /home/kimberly/repos/gastown/internal/doctor/zombie_check.go
  - /home/kimberly/repos/gastown/internal/doctor/orphan_check.go
  - /home/kimberly/repos/gastown/internal/doctor/beads_check.go
  - /home/kimberly/repos/gastown/internal/cmd/doctor.go
tags: [package, diagnostics, health-checks, fix-mode]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
---

# internal/doctor

Gas Town's health-check registry and fix engine. Provides the `Check`
interface, the `Doctor` runner, streaming output, and ~80 concrete check
implementations spanning workspace layout, infrastructure, rig state,
config, Dolt, beads, patrol, hooks, and cleanup. Powers
[`gt doctor`](../commands/doctor.md), [`gt repair`](../commands/repair.md),
and chunks of [`gt upgrade`](../commands/upgrade.md).

**Go package path:** `github.com/steveyegge/gastown/internal/doctor`
**File count:** **70 non-test `.go` files + ~50 `_test.go` files** — the
largest single package in the codebase by file count and roughly 36k
lines of Go total (`wc -l internal/doctor/*.go`).
**CLI surface:** [`gt doctor`](../commands/doctor.md),
[`gt repair`](../commands/repair.md), [`gt upgrade`](../commands/upgrade.md),
[`gt vitals`](../commands/vitals.md)
**Parent binary:** [gt](../binaries/gt.md)
**Role that runs it:** [deacon](../roles/deacon.md) — Deacon runs doctor
patrols in the background; the [rig](../concepts/rig.md) concept drives
the `--rig` / `RigChecks` branch.

## What it actually does

### The `Check` interface

The core type is defined at
`/home/kimberly/repos/gastown/internal/doctor/types.go:94-110`:

```go
type Check interface {
    Name() string
    Description() string
    Run(ctx *CheckContext) *CheckResult
    Fix(ctx *CheckContext) error
    CanFix() bool
}
```

A check has a stable identifier, a human-readable description, a pure
inspection method (`Run`), and an optional auto-fix path (`Fix` + `CanFix`).

`Run` returns a `*CheckResult`
(`types.go:82-91`) with:

- `Status` — `StatusOK` / `StatusWarning` / `StatusError` (`types.go:37-44`)
- `Message` — one-line result
- `Details` — bullet list shown in verbose mode
- `FixHint` — human-oriented remediation text shown in the FAILURES/WARNINGS
  summary section even if the check isn't auto-fixable
- `Category` — grouping label (optional; filled from the check via the
  `categoryGetter` interface at `doctor.go:40-42`)
- `Elapsed` — runtime, filled in by the runner (`doctor.go:61-63`)
- `Fixed` — set to true after a successful `--fix` re-run

A separate optional interface, `categoryGetter`, lets a check supply its
category without it being part of the main `Check` contract
(`doctor.go:39-42`). `BaseCheck` satisfies this automatically.

### `BaseCheck` and `FixableCheck`

Two embeddable base types at `doctor.go:238-278`:

- **`BaseCheck`** — holds `CheckName`, `CheckDescription`, `CheckCategory`.
  Returns `false` from `CanFix()` and `ErrCannotFix` from `Fix()`
  (`doctor.go:260-267`). Used for read-only checks.
- **`FixableCheck`** — embeds `BaseCheck`, overrides `CanFix()` to return
  true (`doctor.go:271-278`). Checks that want auto-fix embed this and
  supply their own `Fix` method. A few checks (notably
  `StaleBinaryCheck`) embed `FixableCheck` but then override `CanFix()`
  back to `false` to opt out with a polite manual-fix hint
  (`stale_binary_check.go:82-84`).

The `ErrCannotFix` and `ErrSkippedNoStart` sentinel errors live in
`errors.go:6-12`. `ErrSkippedNoStart` is raised by fixes that would
otherwise start a daemon/agent when `--no-start` is in effect.

### The `Doctor` runner

Defined at `doctor.go:13-37`:

```go
type Doctor struct {
    checks []Check
}
```

A `Doctor` is just an ordered slice of checks. Construction is
`NewDoctor()`; registration is `Register(check)` or
`RegisterAll(checks...)`. There is no dependency graph, no topological
sort, no priority field — **ordering is defined purely by call order at
registration time.** See [ordering constraints](#ordering-constraints)
below.

### `CheckContext`

`CheckContext` (`types.go:61-76`) is the parameter passed to every
`Run`/`Fix` call:

```go
type CheckContext struct {
    TownRoot        string
    RigName         string   // empty for town-level checks
    Verbose         bool
    RestartSessions bool     // explicit --restart-sessions
    NoStart         bool     // --no-start
}
```

`RigPath()` is a convenience accessor that returns
`TownRoot + "/" + RigName` when a rig is selected. Built once in
`runDoctor` at `/home/kimberly/repos/gastown/internal/cmd/doctor.go:141-148`
and passed to every check — no per-check setup or teardown.

### Execution model

Two entry points, each with a plain and a streaming variant:

- `Doctor.Run(ctx)` / `Doctor.RunStreaming(ctx, w, slowThreshold)`
  (`doctor.go:45-106`)
- `Doctor.Fix(ctx)` / `Doctor.FixStreaming(ctx, w, slowThreshold)`
  (`doctor.go:110-234`)

The non-streaming variants call the streaming variant with `w == nil`.
`gt doctor` always uses the streaming variant with `w == os.Stdout`
(`internal/cmd/doctor.go:293-298`).

**Execution is strictly sequential, single-threaded, in registration order.**
There is no parallelism and no short-circuit: every registered check runs
every time, even if a prerequisite check failed. A `Dolt server not
reachable` error will not stop downstream Dolt-dependent checks from
trying (they just fail individually) — consistent with the "doctor
reports the whole state of the world" design.

The streaming output loop (`doctor.go:52-106`):

1. Print a spinner line `○  <check-name>...` to `w`.
2. Time the `check.Run(ctx)` call (`start := time.Now(); result := ...;
   result.Elapsed = time.Since(start)`).
3. If the check didn't fill `Name` / `Category`, fill them from the
   `Check` instance (via `Name()` and the `categoryGetter` interface).
4. Overwrite the line with a status icon (`✓` / `⚠` / `✗`), optional
   slow indicator (`⏳` when `result.Elapsed >= slowThreshold`), the
   check name, and the muted message.
5. Append the `CheckResult` to the `Report`.

After all checks run, `runDoctor` calls `report.PrintSummaryOnly`
(`types.go:184-210`) which prints a separator, then a one-line summary
(`N passed  N warnings  N failed [N fixed] [N slow]`), then grouped
FAILURES / WARNINGS / FIXED sections (`types.go:356-424`) with each
entry's `FixHint` on a trailing tree connector.

### Fix mode

`FixStreaming` (`doctor.go:129-234`) follows a different protocol:

1. Run the check. Record the initial result.
2. If `result.Status != StatusOK && check.CanFix()`:
   a. Print a `(fixing)...` indicator on the same line.
   b. Call `safeFixCheck(check, ctx)` — a panic-recovery wrapper at
      `doctor.go:117-124` that catches Dolt nil-pointer panics (cited as
      GH#1769) and returns them as errors instead of crashing
      `gt doctor`.
   c. If fix returned nil: **re-run the check** to verify. If the re-run
      is now `StatusOK`, append `(fixed)` to the message and set
      `result.Fixed = true`.
   d. If fix returned `ErrSkippedNoStart`: append a
      `Skipped: --no-start suppresses startup` detail and leave the
      original result alone.
   e. If fix returned any other error: append `Fix failed: <err>` to
      details and leave the original result alone.
3. Record total elapsed time (including fix + re-run).
4. Print the final icon: `🔧` if fixed, else the status icon.

Because every check is re-run after a successful fix, a fix that doesn't
actually resolve the underlying issue will show up as still-failing in
the final report — `--fix` cannot lie by construction.

The `Report.printWarningsSection` distinguishes three buckets in the
summary: FAILURES (errors that didn't get fixed), WARNINGS (warnings
that didn't get fixed), and FIXED (anything where `result.Fixed == true`,
regardless of the underlying status). See `types.go:356-424`.

### Ordering constraints

**There is no ordering mechanism in the package itself.** `Doctor.checks`
is a plain ordered slice. All ordering is encoded in the call order
inside `runDoctor` at `/home/kimberly/repos/gastown/internal/cmd/doctor.go:154-275`.

The one documented constraint, for the `gt-99u` race fix, lives as an
inline comment block at `doctor.go:173-177` (in `internal/cmd/`):

> `NewClaudeSettingsCheck()` must run **before** `NewDaemonCheck()` so
> that if claude settings are stale, we fix them first (deleting stale
> files) before the daemon check kicks the daemon — otherwise the daemon
> would pick up the stale settings.

The enforcement mechanism is literally "call `d.Register` in the right
order." If someone reorders the calls, nothing in the package will flag
it — the race is only caught by integration tests and by the
`ClaudeSettingsCheck` fix re-running the check after removing the stale
file.

Other implicit ordering exists but is not commented:

- Binary-freshness checks (`StaleBinaryCheck`, `BeadsBinaryCheck`,
  `DoltBinaryCheck`) run before `DoltServerReachableCheck` so that a
  missing `dolt` binary surfaces as "binary missing" rather than "server
  not reachable."
- `NewTownGitCheck` runs before `NewTownRootBranchCheck` so that the
  branch check can assume it's operating on a real git repo.
- Workspace-level checks (`WorkspaceChecks()`) are registered first so a
  missing `mayor/town.json` short-circuits the rest of the report with
  early errors.

### Check categories

`types.go:12-32` defines a fixed category enum used for grouping in
non-streaming output:

```go
const (
    CategoryCore           = "Core"
    CategoryInfrastructure = "Infrastructure"
    CategoryRig            = "Rig"
    CategoryPatrol         = "Patrol"
    CategoryConfig         = "Configuration"
    CategoryCleanup        = "Cleanup"
    CategoryHooks          = "Hooks"
)

var CategoryOrder = []string{
    CategoryCore, CategoryInfrastructure, CategoryRig, CategoryPatrol,
    CategoryConfig, CategoryCleanup, CategoryHooks,
}
```

`CategoryOrder` is **only used by `Report.Print`** (the full, non-
streaming printer at `types.go:215-270`) to group checks under headers.
It does **not** affect execution order, and `PrintSummaryOnly` (the one
`gt doctor` actually uses) ignores it entirely — the streaming output is
in registration order, and the summary section is bucketed by
failure/warning/fix status, not category.

### File inventory — categorical breakdown

Grouping the 70 non-test files by filename convention:

**Framework (3 files):**

- `doctor.go` — `Doctor`, `BaseCheck`, `FixableCheck`, streaming runners
- `types.go` — `Check` interface, `CheckContext`, `CheckResult`,
  `Report`, `ReportSummary`, `CategoryOrder`, printing helpers
- `errors.go` — `ErrCannotFix`, `ErrSkippedNoStart`

**Workspace / town root (6 files):**

- `workspace_check.go` — 5 checks: town config exists/valid, rigs
  registry exists/valid, mayor dir exists (`WorkspaceChecks()` batch)
- `global_state_check.go` — Gas Town global state files
- `town_git_check.go` — town root is a git repo
- `town_root_branch_check.go` — town root is on the expected branch
- `town_beads_config_check.go` — town-level beads config
- `town_claude_md_check.go` — town-level CLAUDE.md identity anchor

**Infrastructure / binaries / daemon (8 files):**

- `stale_binary_check.go` — gt build freshness vs repo HEAD
- `beads_binary_check.go` — `bd` presence and min version
- `dolt_binary_check.go` — `dolt` presence and min version
- `daemon_check.go` — Gas Town daemon running
- `boot_check.go` — boot-health markers
- `tmux_check.go` — linked-pane layout integrity
- `tmux_global_env_check.go` — tmux global env vars
- `tmux_socket_check.go` — split-brain socket detection

**Rig-level (8 files):**

- `rig_check.go` — **13 checks** via `RigChecks()` batch: rig is git
  repo, git/exclude config, hooks path, bare repo exists/refspec,
  default branch exists, witness/refinery/mayor/polecat worktree
  existence, beads config, beads redirect, testutil symlink
- `rig_name_check.go` — config `rigs.name` matches directory name
- `rig_config_sync_check.go` — `mayor/rigs.json` in sync with
  `<rig>/gastown/config.json`
- `rigs_json_check.go` — rigs registry structure
- `rig_beads_check.go` — per-rig beads DB integrity
- `rig_routes_jsonl_check.go` — per-rig prefix routes
- `hooks_path_all_rigs_check.go` — `core.hooksPath` set for all rigs
- `testutil_symlink_check.go` — shared testutil symlink

**Config & templates (5 files):**

- `config_check.go` — 6 checks: Settings, RuntimeGitignore,
  LegacyGastown, SessionHook, CustomTypes, CustomStatuses
- `claude_settings_check.go` — Claude `settings.json` template
  conformance + stale-file detection (the biggest single check)
- `deprecated_config_check.go` — deprecated merge-queue keys
- `env_check.go` — environment variables
- `theme_check.go` — UI theme config

**Dolt & beads (10 files):**

- `beads_check.go` — 3 checks: prefix conflict, prefix mismatch,
  database prefix
- `beads_redirect_target_check.go` — beads redirect target validity
- `stale_beads_redirect_check.go` — stale beads redirect files
- `unregistered_beads_check.go` — unregistered `.beads` dirs
- `migration_check.go` — 3 checks: `DoltMetadata`,
  `DoltServerReachable`, `DoltOrphanedDatabase` (despite the file name,
  these are live-state Dolt checks, not migration runners)
- `stale_dolt_port_check.go` — stale Dolt port files
- `stale_sql_server_info_check.go` — stale `sql-server.info` files
  (GH#2770)
- `null_assignee_check.go` — orphan beads with null assignees
- `idle_timeout_check.go` — `dolt.idle-timeout: "0"` for all rigs
- `routing_mode_check.go` — beads routing mode

**Patrol (3 files):**

- `patrol_check.go` — 5 checks: patrol molecules exist, hooks wired, not
  stuck, plugins accessible, plugin drift
- `agent_beads_check.go` — per-agent beads assignment
- `stale_agent_beads_check.go` — stale agent beads entries
- `role_beads_check.go` — role beads config validation

**Hooks & sessions (5 files):**

- `hook_check.go` — 3 checks: attachment valid, singleton, orphaned
  attachments
- `hooks_sync_check.go` — hook files in sync
- `stale_task_dispatch_check.go` — stale task-dispatch state
- `lifecycle_check.go` — lifecycle hygiene
- `lifecycle_defaults_check.go` — lifecycle defaults

**Cleanup / orphans / stale state (9 files):**

- `orphan_check.go` — 2 checks: orphan tmux sessions, orphan processes
- `zombie_check.go` — tmux session alive but Claude process dead
- `wisp_check.go` — wisp garbage collection
- `misclassified_wisp_check.go` — wisp misclassification
- `jsonl_bloat_check.go` — JSONL file bloat
- `crash_report_check.go` — crash report detection
- `stale_runtime_files_check.go` — stale runtime files
- `session_name_check.go` — malformed session names
- `identity_check.go` — agent identity collisions

**Git & branches (6 files):**

- `branch_check.go` — 2 checks: branch check, clone divergence
- `foreign_remote_check.go` — unexpected remotes
- `precheckout_hook_check.go` — pre-checkout hook (branch protection);
  note `NewPreCheckoutHookCheck()` is an alias for
  `NewBranchProtectionCheck()`
- `worktree_gitdir_check.go` — worktree `.git` validity
- `sparse_checkout_check.go` — sparse checkout config
- `land_worktree_gitignore_check.go` — land worktree gitignore rules

**Migration, formula, overlay (4 files):**

- `priming_check.go` — priming state
- `formula_check.go` — formula checkout
- `overlay_health_check.go` — formula overlay health
- `routes_check.go` — routes.jsonl structure

**Crew & commands (2 files):**

- `crew_check.go` — 2 checks: crew state, crew worktrees
- `commands_check.go` — installed slash commands

That's 70 files grouped into roughly **12 functional areas**, producing
the ~80 distinct `NewXxxCheck` constructors registered by `runDoctor` at
`/home/kimberly/repos/gastown/internal/cmd/doctor.go:154-275`. Several
files define multiple checks (`patrol_check.go` has 5, `rig_check.go`
has 13, `config_check.go` has 6, `workspace_check.go` has 5,
`beads_check.go` has 3, `hook_check.go` has 3, `migration_check.go` has
3, `branch_check.go` has 2, `orphan_check.go` has 2, `crew_check.go`
has 2), which is why the file count (70) is substantially less than the
constructor count (~80).

### Batch constructor functions

The package exposes two batch functions that return `[]Check` slices, so
the command wrapper can register a block of checks atomically:

- **`WorkspaceChecks()`** (`workspace_check.go:382-390`) — returns the
  5 core workspace checks. Called once at the top of `runDoctor` so
  missing workspace structure surfaces immediately.
- **`RigChecks()`** (`rig_check.go` — `func RigChecks() []Check`) —
  returns 13 rig-level checks including `RigIsGitRepoCheck`,
  `GitExcludeConfiguredCheck`, `HooksPathConfiguredCheck`,
  `BareRepoExistsCheck`, `BareRepoRefspecCheck`,
  `DefaultBranchExistsCheck`, `WitnessExistsCheck`,
  `RefineryExistsCheck`, `MayorCloneExistsCheck`,
  `PolecatClonesValidCheck`, `BeadsConfigValidCheck`,
  `BeadsRedirectCheck`, `TestutilSymlinkCheck`. Only registered when
  `--rig <name>` is passed, because these checks need a populated
  `ctx.RigName` (`internal/cmd/doctor.go:277-279`).

All other ~60 checks are registered individually by name in `runDoctor`.

## Representative checks

Five spot-checks with different shapes:

### `TownConfigExistsCheck` — minimal read-only check

`workspace_check.go:11-44`. Embeds `BaseCheck`, so `CanFix()` is
false. `Run` stats `<TownRoot>/mayor/town.json` and returns
`StatusError` with a `FixHint: "Run 'gt install' to initialize
workspace"` if missing, `StatusOK` otherwise. This is the minimal shape:
~30 lines, one stat call, no state.

### `RigsRegistryExistsCheck` — minimal fixable check

`workspace_check.go:124-179`. Embeds `FixableCheck`. `Run` stats
`mayor/rigs.json` and returns a warning if missing. `Fix` writes an
empty `{"version": 1, "rigs": {}}` JSON document. The fix is
idempotent and has no side effects beyond file creation.

### `StaleBinaryCheck` — fixable-shaped but opts out

`stale_binary_check.go:10-85`. Embeds `FixableCheck` but overrides
`CanFix()` back to `false` and returns a manual-action error from
`Fix()` (`"run 'gt install' manually to rebuild"`). A comment at
`stale_binary_check.go:73-78` explains the reasoning: rebuilding is
slow, modifies system files, and should be explicit. This is the
pattern for "we have the information to auto-fix but choose not to."

### `DaemonCheck` — process check with `--no-start` integration

`daemon_check.go:11-79+`. Uses `daemon.IsRunning(ctx.TownRoot)` to
check PID file + process liveness. Returns uptime + heartbeat count as
details in the OK case. The `Fix` method (`daemon_check.go:69+`) starts
by checking `ctx.NoStart` and returns `ErrSkippedNoStart` if set — this
is the canonical usage of `ErrSkippedNoStart`, demonstrating how
`--no-start` propagates through the check context to suppress any fix
that would spawn a long-running process. Daemon fix then uses
`os.Executable()` to find `gt` and spawns `gt daemon start`.

### `ClaudeSettingsCheck` — the 800-line flagship fixable check

`claude_settings_check.go` (805 lines). Embeds `FixableCheck` and caches
a `[]staleSettingsInfo` on the struct between `Run` and `Fix`. This is
the most complex check in the package and illustrates several patterns:

- **Multi-location scan:** `findSettingsFiles` walks the town root
  looking for `.claude/settings.json` under mayor, deacon, and every
  rig's witness/refinery/crew/polecats subdirectory. Each match is
  classified as correct-location, wrong-location, or missing-file.
  `claude_settings_check.go:172-527`.
- **Git-status-aware safety:** `getGitFileStatus`
  (`claude_settings_check.go:577-619`) shells out to `git ls-files`,
  `git check-ignore`, and `git diff --quiet` to classify each stale
  file as untracked / tracked-clean / tracked-modified / ignored /
  unknown. Files with local modifications are **skipped** during fix
  with a warning; tracked-clean files are skipped because deleting
  them would modify the customer repo; only untracked/ignored files
  are deleted.
- **Template conformance scan:** `checkSettings`
  (`claude_settings_check.go:532-573`) JSON-parses each file and
  verifies the presence of `enabledPlugins`, a `SessionStart` hook
  containing `prime --hook`, and a `Stop` hook containing
  `costs record`.
- **Fix with re-provision:** `Fix`
  (`claude_settings_check.go:653-778`) deletes stale files and then
  calls `runtime.EnsureSettingsForRole` to recreate the correct file
  at the correct location. For town-root settings it redirects to
  `mayor/` and tells the user to run `gt up --restore`. For rig-scoped
  roles it resolves the correct `settingsDir` via
  `config.RoleSettingsDir`.
- **Restart gating:** Actually killing the tmux session for a live
  patrol agent only happens if `ctx.RestartSessions` is true (set only
  by the explicit `--restart-sessions` flag). Otherwise the user is
  told to run `gt up --restore` manually. This is deliberate — Deacon
  runs `gt doctor --fix` automatically, and auto-killing sessions
  would create a restart loop.
- **Ordering dependency:** must run before `DaemonCheck` (see
  [ordering constraints](#ordering-constraints)).

### `ZombieSessionCheck` — tmux-layer liveness probe

`zombie_check.go:14-60+`. Embeds `FixableCheck`. Iterates
`tmux.NewTmux().ListSessions()`, for each session confirms it's a
valid Gas Town session name, then probes whether the pane still has a
Claude/node process. Sessions that are tmux-alive but Claude-dead get
added to a cached `zombieSessions` slice. The `Fix` method kills the
dead sessions so Deacon (or `gt up --restore`) can re-spawn them cleanly.
Typical of the `Cleanup` category: cache state in the struct during
`Run`, consume it during `Fix`.

## Notable patterns

**State cached on the struct between Run and Fix.** Several checks
(`ClaudeSettingsCheck.staleSettings`, `RigsRegistryValidCheck.missingRigs`
at `workspace_check.go:182-186`, `ZombieSessionCheck.zombieSessions`,
`OrphanSessionCheck.orphanSessions`) stash results on the check instance
during `Run` and re-use them during `Fix`. This is safe because
`FixStreaming` always calls `Run` right before `Fix` on the same
instance, but it means **check structs are not safe for concurrent use**
— another reason execution is strictly sequential.

**`CheckResult.Name` is often left blank and filled by the runner.**
The runner checks `if result.Name == ""` (`doctor.go:66-68`) and fills
from `check.Name()`. Many check implementations skip setting `Name` on
their `CheckResult` to avoid duplication.

**Category resolution is opportunistic.** A check can either set
`result.Category` directly or satisfy the `categoryGetter` interface.
The runner prefers the result's explicit category and falls back to
the interface (`doctor.go:70-72`).

**Panic recovery is only on Fix, not Run.** `safeFixCheck`
(`doctor.go:117-124`) exists specifically because Dolt's in-process
client has panicked in the past (GH#1769); `Run` has no equivalent
wrapper, so a panic inside `check.Run` will crash `gt doctor`. This
asymmetry is probably because Run is read-only and fixable checks are
more likely to touch Dolt write paths.

**`FixHint` is shown even for non-fixable checks.** Every
`CheckResult` can carry a `FixHint`, and the warnings section
(`types.go:388-406`) prints it as a tree connector under every
failure/warning entry regardless of `CanFix()`. `CanFix()` only gates
the automatic `--fix` pathway; the human-readable hint is always shown.

**Subprocess usage is heavy.** Many checks shell out rather than
use Go libraries: `git` (claude-settings, branch, town-git, sparse,
foreign-remote, worktree-gitdir), `tmux` (via `internal/tmux` wrapper
but ultimately `tmux` commands), `dolt` (via the `internal/doltserver`
or `internal/beads` wrappers), `ps` (orphan-process check). This is
another reason doctor is slow enough that `--slow` is a supported flag.

**Output is streamed, not batched.** Because each check is printed as
it runs (`doctor.go:57-58`, `doctor.go:75-100`), a hung check leaves an
incomplete line on screen showing exactly which check hung — important
for debugging a stuck `gt doctor` against a broken Dolt server. This is
why `PrintSummaryOnly` is used instead of `Print` when streaming is
active: the full `Print` method re-groups by category and re-renders
everything, which would duplicate the streamed output.

## Why it's so big

The doctor package is 70 files and ~36k lines because Gas Town's
workspace layout is **unusually load-bearing**. Every concept the rest
of the codebase defines — workspaces, rigs, roles, sessions, hooks,
beads databases, Dolt servers, patrol molecules, overlays, formulas,
crew worktrees — has a corresponding "is this actually in the right
shape on disk?" check, and most of those checks also carry a
self-repair path. The check framework itself is small (~600 lines
across `doctor.go`, `types.go`, `errors.go`); the rest is the catalog
of things that can go wrong with a Gas Town workspace and how to put
each one back. Every external bug number cited in comments
(`GH#1769`, `GH#2770`, `gt-99u`) represents a real-world failure mode
that someone encoded as a permanent check.

## Related wiki pages

- [gt doctor](../commands/doctor.md) — the CLI command that registers
  all ~80 checks
- [gt repair](../commands/repair.md) — narrow subset of doctor for
  specific repair actions
- [gt upgrade](../commands/upgrade.md) — uses doctor checks internally
  during upgrade flows
- [gt vitals](../commands/vitals.md) — sibling diagnostic command
- [deacon (role)](../roles/deacon.md) — runs doctor patrols in the
  background
- [rig (concept)](../concepts/rig.md) — the unit that `RigChecks()`
  operates on
- [internal/beads](beads.md) — downstream of doctor's beads-prefix,
  null-assignee, and unregistered-dirs checks
- [internal/doltserver](doltserver.md) — doctor's Dolt reachability,
  metadata, and orphaned-database checks call into this package
- [internal/config](config.md) — `ResolveRoleAgentConfig` and
  `RoleSettingsDir` used by `ClaudeSettingsCheck` during fix
- [internal/session](session.md) — session-name helpers used by
  claude-settings, zombie, and orphan checks
- [internal/workspace](workspace.md) — `FindFromCwdOrError` is how
  `runDoctor` locates the town root
- [gt](../binaries/gt.md) — parent binary; documents doctor's double
  exemption from beads-version-check and branch-check

## Notes / open questions

- No dependency graph: ordering is encoded purely in the call sequence
  at `internal/cmd/doctor.go:154-275`, with one documented ordering
  constraint (`ClaudeSettingsCheck` before `DaemonCheck`). A dependency
  graph would be a reasonable future refactor.
- No short-circuit: if `DoltServerReachableCheck` fails, the ~15
  downstream Dolt-dependent checks still run and fail individually. A
  "skip" status distinct from pass/warn/fail might clean up the output
  in that failure mode.
- No parallelism: sequential execution is intentional (stateful checks,
  subprocess-heavy, streaming output model) but means a full
  `gt doctor` run on a well-populated workspace takes tens of seconds.
- **`gt doctor` is beads- and branch-check-exempt** (see
  `../commands/doctor.md`) — doctor is the tool used to fix the
  problems that would otherwise block commands, so the normal
  preamble checks are skipped for it.
- The `Category` / `CategoryOrder` system is mostly vestigial under
  streaming mode — the streamed output is in registration order and
  the summary is bucketed by status, not category. `Report.Print` (the
  category-grouped variant) is still used by some non-interactive
  callers.
- Several check files have more than one check defined inside them
  (`patrol_check.go`, `rig_check.go`, `config_check.go`,
  `workspace_check.go`, `beads_check.go`, `hook_check.go`,
  `migration_check.go`, `branch_check.go`, `orphan_check.go`,
  `crew_check.go`). A future refactor could either split by check or
  document why they're grouped (usually: they share helper functions
  or state).
