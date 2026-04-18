---
title: gt show
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/cmd/show.go
tags: [command, work, beads, display, passthrough]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase5_audience: agent
phase8_audited: 2026-04-17
phase8_findings: [none]
---

# gt show

Display a bead by ID by delegating to `bd show`. Accepts any bead
prefix (`gt-`, `bd-`, `hq-`, …) and routes to the correct beads
database automatically.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupWork` ("Work Management") — set on `show.go:11`
**Hidden:** no
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no

> **Batch-plan correction:** `gt show` is in `GroupWork`, not
> ungrouped. See the same note on [commit](commit.md).

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/show.go:15-45`.

### Invocation

```
gt show <bead-id> [flags]
```

`Use: "show <bead-id> [flags]"` (`show.go:16`).
`DisableFlagParsing: true` (`show.go:30`) — every argument is passed
through to `bd show` unchanged, including `--json`, `-v`, and any
future `bd show` flags.

### Behavior

`runShow` on `show.go:34-45`:

1. **Handle `--help`** via `checkHelpFlag(cmd, args)` (`show.go:36-38`)
   before `DisableFlagParsing` hands the flag to `bd show`. This keeps
   `gt show --help` displaying cobra's own help, not `bd show`'s.
2. **Require at least one arg** (`show.go:40-42`) — returns
   `bead ID required\n\nUsage: gt show <bead-id> [flags]`. Cobra's arg
   validator is not used because `DisableFlagParsing: true` makes
   `Args: cobra.MinimumNArgs(1)` unreliable in this mode.
3. **Delegate** via `execBdShow(args)` (body lives elsewhere — helper
   that shells out to `bd show <args>` with appropriate environment
   routing).

### Helper utilities defined in this file

Two helpers with the file-scoped purpose of supporting `runShow` and
any other command that has to chain into `bd`:

#### `extractBeadIDFromArgs(args []string)` (`show.go:49-56`)

Returns the first non-flag positional from `args`. Used to find the
bead ID when `DisableFlagParsing` mode leaves the cobra arg slice as a
flat mix of `-flag` and positionals. Returns empty string if every arg
starts with `-`.

#### `stripEnvKey(env []string, key string)` (`show.go:59-68`)

Removes every entry matching `<key>=*` from an environment slice. The
comment notes `key is always BEADS_DIR today but the function is
intentionally generic` (`show.go:59` inline `//nolint:unparam`). This
is the helper that lets `execBdShow` route across beads databases by
explicitly unsetting `BEADS_DIR` so that `bd` re-resolves the database
from the bead ID prefix.

### Subcommands

None.

### Flags

None declared directly — everything is forwarded to `bd show`.

## Related commands

- [bead](bead.md) — the parent bead command group; `bead show` / `bead
  read` delegate to this command (per the batch brief). Put another
  way: `gt show` is the low-level primitive; `gt bead show` is the
  idiomatic wrapper.
- [cat](cat.md) — sibling display primitive for different data types.
- [assign](assign.md), [close](close.md), [ready](ready.md) — other bead
  operations that complement `show` in a typical work flow.
- [peek](peek.md) — lightweight inspection command; worth comparing
  scopes.
- [issue](issue.md) — legacy/alias surface area for bead operations.
- [../binaries/gt.md](../binaries/gt.md) — root.

## Failure modes

No failure modes discovered. Opens a file or URL in the default viewer. Single `exec.Command` invocation with platform-specific opener. Errors propagated.

## Notes / open questions

- **Thin but load-bearing.** Only 68 lines of Go, but this command is
  the ingress point for every `bd show` invocation from `gt`. The
  auto-routing-via-ID-prefix is what makes `gt` a unified frontend
  across the hq/gt/mo beads databases.
- **`execBdShow` lives elsewhere.** The actual shell-out and env
  manipulation is in a different file — likely `bead.go` or a shared
  `bd_exec.go`. Worth mapping in a future `bd shell-out` concept page.
- **`stripEnvKey` is generic** (per the inline `nolint:unparam`) even
  though the only caller passes `BEADS_DIR`. The generic signature
  signals intent to reuse; worth watching for new callers.
- **`DisableFlagParsing` forces `--help` hand-holding**. Every
  passthrough command in the tree that wraps a `bd` subcommand has to
  repeat this pattern. Worth a shared helper if the set grows.
