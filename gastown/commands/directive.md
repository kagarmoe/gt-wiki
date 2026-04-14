---
title: gt directive
type: command
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/cmd/directive.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, configuration, directives, stub-parent]
---

# gt directive

Parent command for managing operator-provided role
[directives](../concepts/directive.md). In the mapped source file,
`directive` is declared as a parent-only shell — the `show`, `edit`,
and `list` subcommands advertised in its `Long` text are not wired
up here. See the [directive concept page](../concepts/directive.md)
for the config-loader side that actually consumes directive files.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupConfig` ("Configuration")
**Aliases:** `directives`
(`/home/kimberly/repos/gastown/internal/cmd/directive.go:9`)
**Polecat-safe:** no (no `AnnotationPolecatSafe` on the cobra.Command)
**Beads-exempt:** no (not in `beadsExemptCommands` on `root.go:44-77`)
**Branch-check-exempt:** no (not in `branchCheckExemptCommands` on
`root.go:81-91`)

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/directive.go:7-40`.

### Invocation

```
gt directive <subcommand>
```

### Behavior

The parent command only enforces that a subcommand is supplied —
`RunE: requireSubcommand` at
`/home/kimberly/repos/gastown/internal/cmd/directive.go:35`. With no
arguments, it returns an error and prints help.

Registration is a single call —
`/home/kimberly/repos/gastown/internal/cmd/directive.go:38-40`:

```go
func init() {
    rootCmd.AddCommand(directiveCmd)
}
```

No subcommands are added in this file. The `Long` help
(`directive.go:12-34`) advertises three subcommands — `show`, `edit`,
`list` — and describes the file layout and resolution strategy, but
the cobra wiring for them is not in `directive.go`. They are either
defined in another file not mapped yet or not implemented.

### Subcommands

Advertised by `Long` text (`directive.go:18-22`) but not wired in this
file:

- `show <role>` — display active directive for a role
- `edit <role>` — open directive in `$EDITOR`, create if needed
- `list` — list all directive files

### Flags

None declared on the parent command.

### Directive file layout (per Long text)

`directive.go:23-28`:

- Town-level: `<townRoot>/directives/<role>.md`
- Rig-level: `<townRoot>/<rig>/directives/<role>.md`
- Resolution: town and rig directives are concatenated, town first,
  rig last — rig-level content gets the last word.

## Related

- [gt](../binaries/gt.md) — parent binary.
- [README.md](README.md) — command tree index.
- [prime.md](prime.md) — `Long` text says directives are "injected at
  prime time", so `gt prime` is the consumer.
- [config.md](config.md) — sibling in the Configuration group that also
  affects role/agent behavior but through a different mechanism (town
  settings vs. markdown overlays).

## Notes / open questions

- **Missing subcommand wiring.** `directive.go:38-40` does not call
  `directiveCmd.AddCommand(...)` for `show`, `edit`, or `list`. A
  follow-up read should grep the rest of `internal/cmd/` for
  `directiveCmd.AddCommand` to find where (or if) they are registered.
- The `Long` text claims directives "override formula defaults where
  they conflict" — no code-grounded reference yet for how that override
  actually happens.
- A full understanding of `directive` needs an entity page for the
  `prime` consumer path and a concept page for directives generally
  (not yet written).
