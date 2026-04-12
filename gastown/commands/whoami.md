---
title: gt whoami
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/cmd/whoami.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, diagnostics, identity, mail, overseer]
---

# gt whoami

Prints the identity that mail commands will use for the current process,
plus the source of that identity (env var vs. overseer config).

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupDiag` ("Diagnostics")
**Polecat-safe:** no (no `AnnotationPolecatSafe` on the cobra.Command at
`/home/kimberly/repos/gastown/internal/cmd/whoami.go:13-30`)
**Beads-exempt:** no (not in `beadsExemptCommands` on
`/home/kimberly/repos/gastown/internal/cmd/root.go:45-81`)
**Branch-check-exempt:** no (not in `branchCheckExemptCommands` on
`root.go:82-91`)

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/whoami.go:36-80`.

### Invocation

```
gt whoami
```

No flags, no subcommands.

### Behavior

`runWhoami` (`whoami.go:36-80`):

1. Calls `detectSender()` — the same function mail commands use to
   resolve the sender identity (`whoami.go:38`). The shared helper
   lives elsewhere in the cmd package; `whoami` is explicitly
   documented in its `Long` text as "same logic as mail commands."
2. Prints `Identity: <sender>` (`whoami.go:40`).
3. Reads `GT_ROLE` from the environment (`whoami.go:43`):
   - **If set**: prints `Source: GT_ROLE=<value>` and additionally
     surfaces any of `GT_RIG`, `GT_POLECAT`, `GT_CREW` that are set
     (`whoami.go:44-56`). These are the four env vars that identify an
     agent session.
   - **If unset**: prints
     `Source: no GT_ROLE set (human at terminal)` (`whoami.go:58`).
4. **Overseer detail block** (`whoami.go:60-76`): when no `GT_ROLE` is
   set *and* `detectSender()` returned exactly `"overseer"`, whoami
   resolves the town root via `workspace.FindFromCwd`, loads
   `config.OverseerConfig` from `config.OverseerConfigPath(townRoot)`,
   and prints:
   - `Name:` — overseer display name
   - `Email:` — only if non-empty
   - `User:` — only if non-empty (username)
   - A dim `(detected via <source>)` tail, where `<source>` is the
     `OverseerConfig.Source` field (e.g. `accounts.json`, `git config`).
   All failures here are silent — if the town root can't be found or
   the config fails to load, the overseer detail block is simply
   omitted.

### Identity rule

The `Long` text at `whoami.go:17-28` states the rule the command
verifies:

> Identity is determined by:
> 1. `GT_ROLE` env var (if set) — indicates an agent session
> 2. No `GT_ROLE` — you are the overseer (human)

Note that this simplifies the actual `detectSender` logic, which also
reads `GT_RIG`, `GT_POLECAT`, `GT_CREW` to build a fully-qualified
address like `<rig>/<polecat>`. `whoami` shows the composed result.

### Subcommands

None (terminal command).

### Flags

None defined in `init()` (`whoami.go:32-34`) — `init` only registers
the command on `rootCmd`.

## Related

- [audit](audit.md) — records the actor for every workspace event;
  whoami shows the live actor that audit will record for the next
  action.
- [info](info.md) — a broader environment report; whoami is narrowly
  scoped to "who is sending mail."
- [status](status.md) — surfaces `Overseer` (from the same
  `OverseerConfig`) and current agent DND state; whoami is the
  process-local flavor.
- [../binaries/gt.md](../binaries/gt.md) — parent binary; documents
  the `GT_ROLE` / `GT_RIG` env var family used for role detection.
- [README.md](README.md) — command tree index.

## Notes / open questions

- `detectSender` is shared with `gt mail` and lives elsewhere in the
  cmd package — worth a dedicated note under identity resolution.
- `config.OverseerConfig.Source` has a finite set of origins
  (accounts.json, git config, environment fallback); documenting that
  set would make the "detected via" tail unambiguous for users.
- The command deliberately prints to stdout and never errors out.
  Even if workspace discovery fails, it returns `nil`. This makes it
  safe to pipe into shell prompts.
