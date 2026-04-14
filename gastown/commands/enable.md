---
title: gt enable
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/cmd/enable.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, configuration, global-state, lifecycle]
---

# gt enable

Flips the global Gas Town kill-switch to "enabled" so shell hooks and
Claude Code `SessionStart` hooks become active again after a previous
`gt disable`.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupConfig` ("Configuration")
**Polecat-safe:** no (no `AnnotationPolecatSafe` on the cobra.Command at
`/home/kimberly/repos/gastown/internal/cmd/enable.go:14-31`)
**Beads-exempt:** no (not in `beadsExemptCommands` on `root.go:44-77`)
**Branch-check-exempt:** no (not in `branchCheckExemptCommands` on
`root.go:81-91`)

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/enable.go:14-54`.

### Invocation

```
gt enable
```

No arguments and no flags.

### Behavior

`runEnable` (`enable.go:37-54`):

1. Calls `state.Enable(Version)` (`enable.go:38`) to persist the
   "enabled" state in the on-disk Gas Town state directory. `Version`
   is the ldflag-populated version string from `version.go` — the
   state file records which `gt` version last enabled the system.
2. On success, prints a bulleted summary of what enabling now means
   (`enable.go:42-47`):
   - Inject context into Claude Code sessions
   - Set `GT_TOWN_ROOT` and `GT_RIG` environment variables
   - Auto-register git repos as rigs (if configured)
3. Prints a dim hint reminding the user about `gt disable` and
   `gt status` (`enable.go:49-51`).

### Subcommands

None (terminal command).

### Flags

None.

### Environment overrides (documented in Long text)

`enable.go:27-29` — the persistent on-disk state can be overridden
per-session via two env vars:

- `GASTOWN_DISABLED=1` — disable for current session only
- `GASTOWN_ENABLED=1` — enable for current session only

Neither override is consulted by `runEnable` itself; they are honored
by downstream checks that read the combined env + state.

## Related

- [disable.md](disable.md) — inverse command; same `internal/state`
  package, additionally supports `--clean` to rip out shell integration.
- [status.md](status.md) — reports the enabled/disabled state that
  `enable` writes.
- [shell.md](shell.md) — `shell install` also calls `state.Enable(Version)`
  (`shell.go:72`) as a side effect so installing shell hooks implicitly
  re-enables Gas Town.
- [../packages/state.md](../packages/state.md) — the XDG-compliant
  `~/.local/state/gastown/state.json` toggle store this command writes.
- [../binaries/gt.md](../binaries/gt.md) — parent binary.
- [README.md](README.md) — command tree index.

## Notes / open questions

- `state.Enable` signature takes a version string, which implies the
  state file records the version that last toggled it. See
  [../packages/state.md](../packages/state.md) for the full schema.
- The three behaviors listed in the success output are advertised
  rather than verified here — the actual code paths that check the
  enabled flag live in other commands (`prime`, shell hooks, session
  init). The inventory is worth a lint pass once those pages exist.
- Enable writes state but does not install shell integration. A fresh
  user running only `gt enable` without `gt shell install` gets an
  "enabled" flag with no hooks to act on it — see `shell install` for
  the full bring-up.
