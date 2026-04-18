---
title: gt shell
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/cmd/shell.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, configuration, shell-integration, lifecycle]
phase3_audited: 2026-04-15
phase3_findings: [cobra-drift]
phase3_severities: [wrong]
phase3_findings_post_release: false
phase5_audience: user
phase8_audited: 2026-04-17
phase8_findings: [silent-suppression]
---

# gt shell

Manages the Gas Town shell integration hook — the `cd` hook in your
shell RC file that auto-sets `GT_TOWN_ROOT` and `GT_RIG` when you enter
a rig directory. Install, remove, and check status.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupConfig` ("Configuration")
**Polecat-safe:** no (no `AnnotationPolecatSafe` on the cobra.Command at
`/home/kimberly/repos/gastown/internal/cmd/shell.go:15-26`)
**Beads-exempt:** no (not in `beadsExemptCommands` on `root.go:44-77`)
**Branch-check-exempt:** no (not in `branchCheckExemptCommands` on
`root.go:81-91`)

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/shell.go:15-113`.

### Invocation

```
gt shell install
gt shell remove
gt shell status
```

Parent `shellCmd` enforces `RunE: requireSubcommand`
(`shell.go:25`) — calling it bare returns an error.

### Behavior

#### `shell install` (`shell.go:67-81`, `runShellInstall`)

1. Calls `shell.Install()` (`shell.go:68`) from the
   `internal/shell` package to add the integration hook to the
   detected RC file.
2. As a side effect, calls `state.Enable(Version)` (`shell.go:72`)
   so a fresh install also flips the global "enabled" flag. Failures
   here print a dim warning but do **not** fail the command.
3. Prints `✓ Shell integration installed (<rcPath>)` where `rcPath`
   is resolved via `shell.RCFilePath(shell.DetectShell())`
   (`shell.go:76`).
4. Prints a reminder to `source <rcPath>` or open a new terminal.

This is the "hidden re-enable" — running `gt shell install` after
`gt disable` will flip the state back to enabled even though the
documentation only advertises shell-hook installation.

#### `shell remove` (`shell.go:83-90`, `runShellRemove`)

Calls `shell.Remove()` (`shell.go:84`) from `internal/shell` and
prints `✓ Shell integration removed`. Does **not** flip the enabled
state — symmetric with `shell install` only on hook installation, not
on state.

#### `shell status` (`shell.go:92-113`, `runShellStatus`)

1. `state.Load()` (`shell.go:93`). On error, prints:
   ```
   Gas Town: not configured
   Shell integration: not installed
   ```
   and returns nil (so `shell status` is non-failing on a pristine
   system).
2. Reports the persistent `Enabled` flag as
   `Gas Town: enabled` or `Gas Town: disabled` (`shell.go:100-104`).
3. If the state records a detected shell
   (`s.ShellIntegration != ""`), prints it plus the RC-file path
   (`shell.go:106-110`). Otherwise prints
   `Shell integration: not installed`.

### Subcommands

Registered in `init()` (`shell.go:60-65`):

- `install` — install or update shell integration
- `remove` — remove shell integration
- `status` — show enabled + shell-integration state

### Flags

None on any subcommand.

## Docs claim

### Source
- `/home/kimberly/repos/gastown/internal/cmd/shell.go:31-37` — Cobra `Long` text on `shellInstallCmd`.

### Verbatim

> Install or update the Gas Town shell integration.
>
> This adds a hook to your shell RC file that:
>   - Sets GT_TOWN_ROOT and GT_RIG when you cd into a Gas Town rig
>   - Offers to add new git repos to Gas Town on first visit
>
> Run this after upgrading gt to get the latest shell hook features.

## Drift

See forward-link: [../drift/README.md](../drift/README.md).

### `shell install` silently re-enables Gas Town without mentioning it in the `Long` text

- **Claim source:** Cobra `Long` text at `/home/kimberly/repos/gastown/internal/cmd/shell.go:31-37`. The text describes only two effects: adding the `cd` hook to the shell RC file, and the side behaviors of that hook (`GT_TOWN_ROOT`/`GT_RIG` setting, new-repo prompting). Nothing in the text suggests touching the persistent `Enabled` flag.
- **Code does:** `runShellInstall` at `/home/kimberly/repos/gastown/internal/cmd/shell.go:67-81` performs two independent actions. First, `shell.Install()` writes the `cd` hook (`shell.go:68-70`). Second, `state.Enable(Version)` is called at `shell.go:72-74` to persist the "enabled" state in `~/.local/state/gastown/state.json`. A `state.Enable` error prints a dim warning but does NOT fail the command, so the hook-install path almost always leaves the user with `Gas Town: enabled`. This means a user who previously ran `gt disable` and then reinstalls shell hooks via `gt shell install` — perhaps to pick up "latest shell hook features" as the `Long` text encourages — will find their global kill-switch silently flipped back to enabled with no mention in the help text or the success output at `shell.go:77`. The symmetry is broken: `shell remove` (`shell.go:83-90`) does NOT call `state.Disable()`, so the state flag is asymmetric across the two commands.
- **Category:** `cobra drift`
- **Severity:** `wrong`
- **Fix tier:** `code` — add one line to `shellInstallCmd.Long` at `shell.go:31-37` documenting the `state.Enable(Version)` side effect. Suggested wording: "Also sets the global 'enabled' state (equivalent to `gt enable`)." If the product intent is to make `shell install` install-only, remove the `state.Enable` call at `shell.go:72-74` and document separately that users who previously disabled need to run `gt enable` after reinstalling shell hooks. The minimal fix is the doc change; the behavior change is a follow-up decision.
- **Release position:** `in-release` (`shellInstallCmd.Long` and `runShellInstall`'s `state.Enable(Version)` call both byte-identical at `v1.0.0:internal/cmd/shell.go`).

## Failure modes

### Silent suppression (what errors are swallowed?)

- **`shell install` swallows `state.Enable` error:** `shell.go:72-74` catches the `state.Enable(Version)` error and prints a dim warning (`style.Dim`), but the command returns `nil` (success). The shell hook is installed and the user sees a success message, but the persistent "enabled" state may not have been set. On next boot, `gt status` might show "disabled" despite shell hooks being active. **Present** — warning printed, but the dim styling makes it easy to miss.
- **`shell status` treats all `state.Load()` errors as "not configured":** `shell.go:93-98` catches any error from `state.Load()` and prints "Gas Town: not configured / Shell integration: not installed" then returns `nil`. A corrupt state file, a permissions error, or a disk I/O error all produce the same "not configured" output with no indication that something is broken. **Absent** — predicted bug surface: user with a corrupt state file sees "not configured" instead of a parse error.

## Related

- [gt](../binaries/gt.md) — parent binary.
- [../packages/shell.md](../packages/shell.md) — the installer package
  backing every subcommand here; defines the zsh/bash `cd` hook script.
- [README.md](README.md) — command tree index.
- [enable.md](enable.md) — `shell install` quietly re-enables via
  `state.Enable(Version)`; `enable` does the same thing without
  touching shell hooks.
- [disable.md](disable.md) — `disable --clean` calls
  `shell.Remove()` — the same code path as `shell remove`.
- [uninstall.md](uninstall.md) — full removal; also calls
  `shell.Remove()`. `shell remove` is the narrow version.
- [status.md](status.md) — there may be overlap with what
  `shell status` reports vs. what the broader `gt status` shows;
  worth a lint pass.

## Notes / open questions

- The `install` → `enable` coupling → promoted to `## Drift` above (`shell install` cobra-drift finding). The asymmetric symmetry with `shell remove` (which does NOT flip state to disabled) is captured in the same finding.
- **Completion is NOT here.** Tab-completion is cobra's built-in
  `completion` command on the root (see `root.go` imports and
  cobra's default `AddCommand`), exempt from beads on `root.go:47`.
  `gt shell` is only about the `cd`-hook shell integration, not
  completion scripts.
- `internal/shell` package needs its own `packages/shell.md` page:
  what `Install()` actually writes, how it detects the shell, the
  RC-file snippet it injects.
- `shell status` has three possible states but renders them
  non-uniformly: "not configured" (no state file), "enabled" or
  "disabled" (has state), and an optional shell-integration line.
  Users confused by `Gas Town: disabled` with
  `Shell integration: bash (~/.bashrc)` may want a single "effective
  state" summary.
