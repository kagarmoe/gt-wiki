---
title: gt disable
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-15
sources:
  - /home/kimberly/repos/gastown/internal/cmd/disable.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, configuration, global-state, lifecycle]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
---

# gt disable

Flips the global Gas Town kill-switch to "disabled" so shell hooks and
Claude Code `SessionStart` hooks become no-ops. Optional `--clean`
mode additionally rips the shell integration lines out of `~/.zshrc`
or `~/.bashrc`.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupConfig` ("Configuration")
**Polecat-safe:** no (no `AnnotationPolecatSafe` on the cobra.Command at
`/home/kimberly/repos/gastown/internal/cmd/disable.go:17-36`)
**Beads-exempt:** no (not in `beadsExemptCommands` on `root.go:44-77`)
**Branch-check-exempt:** no (not in `branchCheckExemptCommands` on
`root.go:81-91`)

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/disable.go:17-72`.

### Invocation

```
gt disable [--clean]
```

### Behavior

`runDisable` (`disable.go:44-68`):

1. Calls `state.Disable()` (`disable.go:45`) to flip the persistent
   on-disk flag. Any error is wrapped and returned.
2. If `--clean` was set (`disable.go:49-56`), calls
   `removeShellIntegration()` which is a one-line wrapper around
   `shell.Remove()` (`disable.go:70-72`). Failures here print a
   warning via `style.Warning.Render("!")` but do **not** fail the
   command — the state flip has already succeeded.
3. Prints `✓ Gas Town disabled` followed by a reminder that all
   agentic coding tools now work vanilla (`disable.go:58-60`).
4. If `--clean` was NOT used, prints a dim hint about
   `gt disable --clean` for users who also want shell integration
   removed (`disable.go:61-64`).
5. Prints a dim hint about re-enabling via `gt enable`
   (`disable.go:65`).

### Subcommands

None (terminal command).

### Flags

| Flag      | Type | Default | Purpose                                        |
|-----------|------|---------|------------------------------------------------|
| `--clean` | bool | `false` | Also remove shell integration from RC files    |

Declared in `init()` (`disable.go:38-42`).

### Effect on tools (per Long text)

`disable.go:21-28`:

- Shell hooks become no-ops
- Claude Code `SessionStart` hooks skip `gt prime`
- Tools work 100% vanilla (no Gas Town behavior)

The workspace (`~/gt`) is **preserved**. Disable flips a flag and
(optionally) removes shell hooks; it does not touch the town directory.

### Environment overrides

Documented at `disable.go:33-34`: `GASTOWN_ENABLED=1` re-enables for the
current session only, bypassing the persistent disabled flag.

## Related

- [enable.md](enable.md) — inverse command; persists "enabled" state.
- [status.md](status.md) — reports the enabled/disabled state.
- [shell.md](shell.md) — `shell remove` is the isolated version of
  `--clean`; both route through `shell.Remove()`.
- [uninstall.md](uninstall.md) — full removal; calls `shell.Remove()`
  among other things. `disable --clean` is a softer subset that
  preserves workspace, state dirs, wrappers.
- [../packages/state.md](../packages/state.md) — the XDG-compliant
  toggle store `state.Disable()` writes.
- [../binaries/gt.md](../binaries/gt.md) — parent binary.
- [README.md](README.md) — command tree index.

## Notes / open questions

- Both `disable` and `disable --clean` leave the `~/gt` workspace
  intact. Users looking to fully remove Gas Town want `uninstall`.
- `state.Disable()` does not take a version argument, unlike
  `state.Enable(Version)` in `enable.go:38`. Asymmetric API — disable
  is version-agnostic. Worth flagging in a `packages/state.md` page.
- `--clean` failure is non-fatal; if `shell.Remove()` errors, the state
  is still disabled but the user sees a warning line. This means
  running `gt disable --clean` then `gt status` may show "disabled"
  while shell hooks are still present — a mild consistency risk.
