---
title: gt repair
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/cmd/repair.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, diagnostics, dolt, identity, config, repair]
phase3_audited: 2026-04-15
phase3_findings: [cobra-drift]
phase3_severities: [wrong]
phase3_findings_post_release: false
phase5_audience: dev
phase8_audited: 2026-04-17
phase8_findings: [partial-completion]
---

# gt repair

Runs a narrow subset of `gt doctor --fix` checks targeted at the most
common post-crash failure mode: `metadata.json` pointing at the wrong
Dolt database and related identity/config drift. Built on the
[doctor package](../packages/doctor.md)'s `Check` / `FixStreaming`
machinery, just with a curated subset of the registered checks.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupDiag` ("Diagnostics")
**Polecat-safe:** no (no `AnnotationPolecatSafe` on the cobra.Command at
`/home/kimberly/repos/gastown/internal/cmd/repair.go:12-33`)
**Beads-exempt:** no (not in `beadsExemptCommands` on
`/home/kimberly/repos/gastown/internal/cmd/root.go:45-81`)
**Branch-check-exempt:** no (not in `branchCheckExemptCommands` on
`root.go:82-91`)

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/repair.go:39-91`.

### Invocation

```
gt repair
```

No flags, no subcommands.

### Behavior

`runRepair` (`repair.go:39-91`):

1. Resolves the town root via `workspace.FindFromCwdOrError`
   (`repair.go:40-43`). Errors out with "not in a Gas Town workspace"
   if not found.
2. Builds a `doctor.CheckContext` with `Verbose: true` hard-coded
   (`repair.go:45-48`).
3. Constructs a fixed two-check suite (`repair.go:51-54`):
   - `doctor.NewRigConfigSyncCheck()` — verifies rigs' `config.json`
     matches `rigs.json` on prefix/identity
   - `doctor.NewStaleDoltPortCheck()` — verifies `metadata.json`'s
     Dolt port matches the actually-running server
4. Prints "Repairing database identity and configuration..."
   (`repair.go:56-57`).
5. For each check (`repair.go:60-82`):
   - Runs `check.Run(ctx)`.
   - On `doctor.StatusOK` → prints `✓ <Name>: <Message>` and continues.
   - Otherwise → prints `⚠ <Name>: <Message>`, lists `result.Details`
     indented, and if `check.CanFix()` returns true, prints
     `    Fixing...` and calls `check.Fix(ctx)`. Fix failures go to
     stderr and don't abort the loop.
6. Prints a final status line (`repair.go:84-88`):
   - No issues seen → `All identity checks passed — no repairs needed.`
   - Issues seen → `Repair complete. Run 'gt doctor' for a full diagnostic.`

Note `hasIssues` stays true after a successful `Fix` — the closing
message is "Repair complete" regardless of whether the fix succeeded.
There is no second pass to verify.

### Scope vs. doctor

The `Long` text at `repair.go:16-31` positions this as a narrow
counterpart to `gt doctor --fix`:

> This is a focused version of `gt doctor --fix` that targets the most
> common failure mode: `metadata.json` pointing to the wrong Dolt
> database after a crash, rig addition, or bd init conflict.

See the `## Drift` section below for the relationship between the six
repair targets the `Long` lists and the two checks `runRepair` actually
registers.

### Subcommands

None (terminal command).

### Flags

None defined in `init()` (`repair.go:35-37`) — only command
registration on `rootCmd`.

## Docs claim

### Source
- `/home/kimberly/repos/gastown/internal/cmd/repair.go:16-31` — Cobra `Long` text on `repairCmd`.

### Verbatim

> Repair common database identity mismatches and configuration issues.
>
> This is a focused version of 'gt doctor --fix' that targets the most common
> failure mode: metadata.json pointing to the wrong Dolt database after a crash,
> rig addition, or bd init conflict.
>
> What it repairs:
>   - metadata.json dolt_database pointing to wrong database
>   - Missing config.json for registered rigs
>   - Prefix mismatches between config.json and rigs.json
>   - Missing Dolt databases
>   - Missing rig identity beads
>   - Stale Dolt port in metadata.json
>
> For a full diagnostic, use 'gt doctor' instead.
> For a full diagnostic with auto-fix, use 'gt doctor --fix'.

## Drift

See forward-link: [../drift/README.md](../drift/README.md).

### Cobra `Long` text lists six repair targets; `runRepair` registers two checks

- **Claim source:** Cobra `Long` text at `/home/kimberly/repos/gastown/internal/cmd/repair.go:22-28` (the "What it repairs" bullet list).
- **Docs claim:** the `Long` text enumerates six repair targets — `metadata.json dolt_database` pointing to wrong database, missing `config.json` for registered rigs, prefix mismatches between `config.json` and `rigs.json`, missing Dolt databases, missing rig identity beads, stale Dolt port in `metadata.json`.
- **Code does:** `runRepair` constructs a fixed two-check suite at `/home/kimberly/repos/gastown/internal/cmd/repair.go:51-54` — `doctor.NewRigConfigSyncCheck()` and `doctor.NewStaleDoltPortCheck()`. Notably absent from the registered set: no `NewRigBeadsCheck`/`NewAgentBeadsCheck`/`NewRoleBeadsCheck` (identity bead repair), no `NewDoltOrphanedDatabasesCheck`, no `NewDatabasePrefixCheck`. Any coverage of the other four advertised targets would have to be a side-effect inside the two registered checks' `Fix` methods in `internal/doctor/`. Whether those side-effects exist is unverified by this audit and, per the authority hierarchy, is irrelevant: the code at the cited line registers two checks, the `Long` text claims six, and the disagreement stands.
- **Category:** `cobra drift`
- **Severity:** `wrong`
- **Fix tier:** `code` — either (a) register the missing checks in the `checks` slice, or (b) rewrite the `Long` text to accurately describe what the two registered checks actually cover.
- **Release position:** `in-release` (identical text and registration at `v1.0.0:internal/cmd/repair.go`).

## Related

- [doctor](doctor.md) — the full diagnostic suite; repair is a
  two-check subset. Users are explicitly directed to `gt doctor` for
  full coverage in the closing message.
- [upgrade](upgrade.md) — upgrade also runs a subset of doctor checks
  (`upgrade.go:120-157`) in `Fix` mode but with a broader, post-install
  oriented set that includes identity bead backfill.
- [vitals](vitals.md) — reports Dolt server health and port;
  `StaleDoltPortCheck` is repair's second check.
- [../binaries/gt.md](../binaries/gt.md) — parent binary.
- [README.md](README.md) — command tree index.

## Failure modes

### Partial completion (what doesn't it clean up?)
- **Fix failure does not abort the loop:** `repair.go:76-79` prints
  the fix error to stderr but continues to the next check. The
  `hasIssues` flag remains true, so the closing message says "Repair
  complete" even when fixes failed. The exit code is always zero
  (`runRepair` returns `nil`). **Absent** — fix failure is not
  reflected in the exit code, and there is no re-verification pass
  after fixing.

## Notes / open questions

- `doctor.NewRigConfigSyncCheck` and `doctor.NewStaleDoltPortCheck`
  live in `internal/doctor/`. The semantics of their `Fix` methods
  determine whether `repair`'s `Long` description matches what it
  actually fixes.
- No `--dry-run` flag — repair always attempts to fix on first call.
- `CheckContext.Verbose` is hard-coded to `true`; there is no way to
  run a quiet repair pass.
- The exit code is always zero even when fixes fail (errors only go
  to stderr, `runRepair` returns `nil`).
