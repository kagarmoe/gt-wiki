---
title: gt shell
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/cmd/shell.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, configuration, shell-integration, lifecycle]
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

- The `install` → `enable` coupling is convenient but asymmetric:
  `shell install` enables, `shell remove` does not disable. A user
  running `shell remove` still has `Enabled=true` in state until
  they run `gt disable`. Non-obvious from the command's help text.
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
