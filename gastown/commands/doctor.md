---
title: gt doctor
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/cmd/doctor.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, diagnostics, health-checks, beads-exempt, branch-check-exempt]
phase3_audited: 2026-04-15
phase3_findings: [cobra-drift]
phase3_severities: [wrong]
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
---

# gt doctor

Runs a registered suite of diagnostic checks against the Gas Town
workspace and prints a streaming report with an optional `--fix` mode.
The single largest catalog of health checks in the binary. Backed by
the [doctor package](../packages/doctor.md), which implements the
`Check` interface, the registration model, and the streaming execution
engine.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupDiag` ("Diagnostics")
**Polecat-safe:** no (no `AnnotationPolecatSafe` on the cobra.Command at
`/home/kimberly/repos/gastown/internal/cmd/doctor.go:22-120`)
**Beads-exempt:** **yes** — `doctor` is in `beadsExemptCommands` at
`/home/kimberly/repos/gastown/internal/cmd/root.go:58`; the `bd`
version check is skipped so doctor can diagnose beads problems itself.
**Branch-check-exempt:** **yes** — `doctor` is in
`branchCheckExemptCommands` at
`/home/kimberly/repos/gastown/internal/cmd/root.go:85`; the town-root
branch warning is skipped because doctor is the tool used to fix it
(comment: "Used to fix the problem").

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/doctor.go:22-309`.

### Invocation

```
gt doctor [--fix] [-v] [--rig NAME] [--restart-sessions] [--no-start] [--slow [DUR]]
```

Single terminal command (no subcommands). Runs `runDoctor`
(`doctor.go:134-309`).

### Behavior

`runDoctor` (`doctor.go:134-309`):

1. Requires a Gas Town workspace — `workspace.FindFromCwdOrError`
   (`doctor.go:136-139`).
2. Builds a `doctor.CheckContext` with town root, rig filter, verbose,
   restart-sessions flag, and no-start flag (`doctor.go:141-148`).
3. Creates a `doctor.NewDoctor()` instance (`doctor.go:151`) and
   registers the full check catalog (`doctor.go:154-275`).
4. Parses the optional `--slow` threshold
   (`doctor.go:282-289`). When `--slow` is passed without a value, the
   default is `1s` via `NoOptDefVal = "1s"` (`doctor.go:130`).
5. Runs the checks streaming to stdout: `d.FixStreaming` if `--fix`
   else `d.RunStreaming` (`doctor.go:293-298`).
6. Prints the summary via `report.PrintSummaryOnly`
   (`doctor.go:301`) and exits non-zero if `report.HasErrors()`
   (`doctor.go:304-306`).

### Registered checks

The catalog is assembled in `runDoctor` between `doctor.go:154-275`.
Every check lives in the `internal/doctor/` package.

**Workspace-level (registered first as the fundamental prerequisites):**

- `doctor.WorkspaceChecks()` (`doctor.go:154`) — batch that includes the
  basic town-config checks documented in the Long text.
- `NewGlobalStateCheck` (`doctor.go:156`).

**Infrastructure prerequisites** (explicit ordering comment at
`doctor.go:158-163`):

- `NewStaleBinaryCheck` — is `gt` fresh vs repo
- `NewBeadsBinaryCheck` — is `bd` installed, min version
- `NewDoltBinaryCheck` — is `dolt` installed, min version
- `NewDoltServerReachableCheck` — is the Dolt SQL server up

**Town root protection** (`doctor.go:169-172`):
`NewTownGitCheck`, `NewTownRootBranchCheck`, `NewForeignRemoteCheck`,
`NewPreCheckoutHookCheck`.

**Claude settings must run before daemon** — inline comment at
`doctor.go:173-177` documents `gt-99u` race fix:
`NewClaudeSettingsCheck` must run before `NewDaemonCheck` so the daemon
picks up correct settings.

**Infrastructure and runtime checks** (`doctor.go:178-215`): daemon,
tmux global env, boot health, town beads config, custom types/statuses,
formula, overlay health, prefix conflict, rig name mismatch, rig config
sync, stale Dolt port, stale sql-server.info (comment: "GH#2770"),
prefix mismatch, database prefix, idle timeout (verifies
`dolt.idle-timeout: "0"` for all rigs), routes, rig routes JSONL,
routing mode, malformed session name, orphan session, zombie session,
orphan process, wisp GC, misclassified wisps, JSONL bloat, stale beads
redirect, beads redirect target, stale runtime files, branch, clone
divergence, default branch all rigs, identity collision, linked pane,
socket split brain, theme, crash report, env vars.

**Patrol system checks** (`doctor.go:218-226`): `PatrolMoleculesExist`,
`PatrolHooksWired`, `PatrolNotStuck`, `PatrolPluginsAccessible`,
`PatrolPluginDrift`, `AgentBeads`, `StaleAgentBeads`, `RigBeads`,
`RoleBeads`.

**Config architecture checks** (`doctor.go:231-238`): settings, session
hook, runtime gitignore, legacy gastown, deprecated merge-queue keys,
land-worktree gitignore, hooks-path all rigs.

**Migration checks** (`doctor.go:241-246`): sparse checkout, priming,
town-root CLAUDE.md version.

**Crew workspace checks** (`doctor.go:249-252`): crew state, crew
worktrees, commands.

**Lifecycle hygiene** (`doctor.go:255-256`):
`NewLifecycleHygieneCheck`, `NewLifecycleDefaultsCheck`.

**Hook attachment** (`doctor.go:259-261`): hook attachment valid, hook
singleton, orphaned attachments.

**Hooks sync** (`doctor.go:264-265`): stale task dispatch, hooks sync.

**Dolt data health** (`doctor.go:268-271`): dolt metadata, dolt
orphaned database, unregistered beads dirs, null assignee.

**Worktree gitdir validity** (`doctor.go:274`).

**Rig-specific checks** — only when `--rig` is set
(`doctor.go:277-279`): `doctor.RigChecks()`.

### Subcommands

None (terminal command).

### Flags

Defined in `init()` (`doctor.go:122-132`):

| Flag                  | Type   | Default | Purpose                                                                |
|-----------------------|--------|---------|------------------------------------------------------------------------|
| `--fix`               | bool   | `false` | Attempt automatic fixes for issues that support it                     |
| `--verbose` / `-v`    | bool   | `false` | Show detailed output                                                   |
| `--rig`               | string | `""`    | Check a specific rig only (also enables `RigChecks` suite)             |
| `--restart-sessions`  | bool   | `false` | Restart patrol sessions when `--fix` fixes stale settings              |
| `--no-start`          | bool   | `false` | Suppress starting daemon/agents during `--fix`                         |
| `--slow [DUR]`        | string | `""`    | Highlight slow checks; optional threshold with default `1s`            |

### Long description categories

The `Long` text (`doctor.go:26-118`) organizes the checks into narrative
groups: workspace, town root protection, infrastructure, cleanup, clone
divergence, crew workspace, migration, rig-specific (`--rig`),
routing, lifecycle, formula overlay, migration, session hooks, Dolt,
and patrol. Those groups are documentation-only — the actual
registration order in `runDoctor` above is what ships, and the registered
set is a strict superset of what `Long` enumerates. See the `## Drift`
section below for details.

## Docs claim

### Source
- `/home/kimberly/repos/gastown/internal/cmd/doctor.go:26-118` — Cobra `Long` text on `doctorCmd`, which is a narrative catalog of checks grouped by category. Not reproduced verbatim here because the entire block is ~93 lines of bulleted check names; the drift below cites specific omissions.

## Drift

See forward-link: [../drift/README.md](../drift/README.md).

### Cobra `Long` text enumerates a selective subset of the checks `runDoctor` registers

- **Claim source:** Cobra `Long` text at `/home/kimberly/repos/gastown/internal/cmd/doctor.go:26-118`.
- **Docs claim:** the `Long` text presents a bulleted catalog under named sections (Workspace checks, Town root protection, Infrastructure checks, Cleanup checks, Clone divergence checks, Crew workspace checks, Migration checks, Rig checks, Routing checks, Lifecycle checks, Formula overlay checks, Session hook checks, Dolt checks, Patrol checks) and asks readers to use the list to understand what doctor tests.
- **Code does:** `runDoctor` registers roughly 80 checks at `/home/kimberly/repos/gastown/internal/cmd/doctor.go:154-279`, of which many are not named anywhere in the `Long` text. Examples of registered checks absent from the catalog: `NewTmuxGlobalEnvCheck`, `NewOverlayHealthCheck`, `NewRigNameMismatchCheck`, `NewRigConfigSyncCheck`, `NewStaleDoltPortCheck`, `NewStaleSQLServerInfoCheck`, `NewMalformedSessionNameCheck`, `NewZombieSessionCheck`, `NewOrphanProcessCheck`, `NewWispGCCheck`, `NewMisclassifiedWispsCheck`, `NewBranchCheck`, `NewIdentityCollisionCheck`, `NewLinkedPaneCheck`, `NewSocketSplitBrainCheck`, `NewThemeCheck`, `NewCrashReportCheck`, `NewEnvVarsCheck`, `NewTownClaudeMDCheck`, `NewLandWorktreeGitignoreCheck`, `NewHooksPathAllRigsCheck`, `NewPatrolPluginDriftCheck`, `NewAgentBeadsCheck`, `NewStaleAgentBeadsCheck`, `NewRigBeadsCheck`, `NewRoleBeadsCheck`, `NewCrewCommandsCheck`, `NewHookAttachmentValidCheck`, `NewHookSingletonCheck`, `NewOrphanedAttachmentsCheck`, `NewLifecycleHygieneCheck`, `NewLifecycleDefaultsCheck`, `NewNullAssigneeCheck`, `NewDoltMetadataCheck`, `NewUnregisteredBeadsDirsCheck`, `NewRigRoutesJSONLCheck`, `NewRoutingModeCheck`. The `Long` catalog is thus a partial, manually-maintained snapshot of a much larger registered set.
- **Category:** `cobra drift`
- **Severity:** `wrong`
- **Fix tier:** `code` — edit the `Long` text in `doctor.go` to either (a) accurately enumerate the registered set, (b) summarize the categories without committing to a bulleted catalog, or (c) generate the catalog from the registered checks at runtime so it cannot drift.
- **Release position:** `in-release` (doctor.go file and its check-registration block both exist at `v1.0.0`).

## Related

- [heartbeat](heartbeat.md) — doctor's health-check cousin at the
  agent-state level; doctor runs workspace-wide checks, heartbeat
  reports a single agent's state.
- [dashboard](dashboard.md) — `NewDoltServerReachableCheck` is a shared
  concern; dashboard forces Dolt env vars for child bd processes while
  doctor tests the server is up.
- [metrics](metrics.md) — reports command usage; doctor reports
  workspace health.
- [../binaries/gt.md](../binaries/gt.md) — parent binary; documents
  doctor's double exemption (beads + branch check).
- [README.md](README.md) — command tree index.

## Notes / open questions

- The `internal/doctor/` package contains every `NewXxxCheck`
  constructor referenced above — roughly 80 distinct checks. A
  dedicated package page would enumerate their semantics.
- Comment at `doctor.go:228` says `StaleAttachmentsCheck` was removed
  because staleness detection belongs in the Deacon molecule.
- `--fix` mode runs `d.FixStreaming` which is a different code path
  than `d.RunStreaming`; what each check's `Fix` method actually does
  is check-specific and untracked here.
- The registration ordering is load-bearing in at least one documented
  case (`ClaudeSettingsCheck` before `DaemonCheck` per `gt-99u`). Other
  ordering constraints may be implicit.
- "Dolt server reachable" is listed as a prerequisite but nothing stops
  downstream checks from running if it fails — they just fail
  individually. Worth confirming there is no short-circuit.
