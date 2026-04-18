---
title: gt directive
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/cmd/directive.go
  - /home/kimberly/repos/gastown/internal/cmd/directive_show.go
  - /home/kimberly/repos/gastown/internal/cmd/directive_edit.go
  - /home/kimberly/repos/gastown/internal/cmd/directive_list.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, configuration, directives]
phase3_audited: 2026-04-15
phase3_findings: [wiki-stale]
phase3_severities: [wrong]
phase3_findings_post_release: false
phase5_audience: user
phase8_audited: 2026-04-17
phase8_findings: [precondition-violation, silent-suppression]
detail_depth: {params: 2, data_flow: 2, errors: 1, side_effects: 1}
---

# gt directive

Parent command for managing operator-provided role
[directives](../concepts/directive.md). The `show`, `edit`, and `list`
subcommands advertised in the `Long` text on `directiveCmd` are wired
up in sibling files — `directive_show.go`, `directive_edit.go`, and
`directive_list.go`, each of which adds itself to `directiveCmd` in its
own `init()` (Phase 3 Batch 1b wiki-stale correction: the Phase 2 page
body incorrectly claimed the subcommands were unwired). See the
[directive concept page](../concepts/directive.md) for the
config-loader side that consumes directive files.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupConfig` ("Configuration")
**Aliases:** `directives`
(`/home/kimberly/repos/gastown/internal/cmd/directive.go:9`)
**Polecat-safe:** no (no `AnnotationPolecatSafe` on the cobra.Command)
**Beads-exempt:** no (not in `beadsExemptCommands` on `root.go:44-77`)
**Branch-check-exempt:** no (not in `branchCheckExemptCommands` on
`root.go:81-91`)

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/directive.go:7-40`
plus sibling files `directive_show.go`, `directive_edit.go`,
`directive_list.go`.

### Invocation

```
gt directive show <role> [--rig <name>]
gt directive edit <role> [--rig <name>] [--town]
gt directive list
```

### Behavior

The parent command only enforces that a subcommand is supplied —
`RunE: requireSubcommand` at
`/home/kimberly/repos/gastown/internal/cmd/directive.go:35`. With no
arguments, it returns an error and prints help.

`directive.go`'s own `init()` registers only the parent on the root
(`directive.go:38-40`):

```go
func init() {
    rootCmd.AddCommand(directiveCmd)
}
```

Each subcommand is defined in its own sibling file and registers
itself on `directiveCmd` through its own `init()`:

- `directive_show.go:30-33` — `directiveCmd.AddCommand(directiveShowCmd)` + `--rig` flag registration
- `directive_edit.go:37-41` — `directiveCmd.AddCommand(directiveEditCmd)` + `--rig`, `--town` flag registration
- `directive_list.go:25-27` — `directiveCmd.AddCommand(directiveListCmd)`

All three are wired at compile time via Go's init ordering — no runtime
registration or plugin loader is involved. The `Long` text on
`directiveCmd` at `directive.go:12-34` names the same three
subcommands and describes the file layout and resolution strategy,
all of which match the behavior implemented in the sibling files.

### Subcommands

#### `show <role>` (`directive_show.go:14-26`, `runDirectiveShow` at `:35-86`)

Displays the resolved directive for a role with source annotation.

1. Validates the role via `isValidRole(role)` against
   `config.AllRoles()` (`directive_show.go:39-41`, `:107-114`) —
   errors with the valid-role list on miss.
2. Resolves the town root and rig name via
   `resolveDirectiveContext(--rig)` (`directive_show.go:43-46`,
   `:89-104`); the rig defaults to `detectRigFromPath(townRoot, cwd)`
   if `--rig` is not passed.
3. Calls `config.LoadRoleDirective(role, townRoot, rigName)`
   (`directive_show.go:54`) to get the concatenated directive text.
4. If empty: prints the "No directive found" message and lists the
   town and rig paths that were checked (`directive_show.go:55-63`).
5. Otherwise prints a source-annotated header identifying whether
   content came from town-only, rig-only, or both
   (`directive_show.go:70-81`), followed by the directive body.

Takes `--rig <name>` (default: auto-detect).

#### `edit <role>` (`directive_edit.go:14-30`, `runDirectiveEdit` at `:43-...`)

Opens the directive file for a role in `$EDITOR`. Creates the
containing directory and file if they don't exist. By default edits
the rig-level directive when a rig is detected, falling back to the
town-level directive. `--town` forces editing the town-level file
even inside a rig.

Takes `--rig <name>` and `--town` flags.

#### `list` (`directive_list.go:13-23`, `runDirectiveList` at `:29-...`)

Walks the town and per-rig `directives/` directories and prints each
found directive file with its scope (town or rig) and role.

Takes no flags.

### Flags

None on the parent command itself; each subcommand declares its own
flags in its sibling file's `init()`.

### Directive file layout (per Long text)

`directive.go:23-28`:

- Town-level: `<townRoot>/directives/<role>.md`
- Rig-level: `<townRoot>/<rig>/directives/<role>.md`
- Resolution: town and rig directives are concatenated, town first,
  rig last — rig-level content gets the last word.

## Failure modes

### Precondition violations (what does it assume?)

- **`directive edit` falls back to `vi` if `$EDITOR` is unset:** `directive_edit.go:79-81` defaults `editor` to `"vi"` when `os.Getenv("EDITOR")` returns empty. On systems where `vi` is not installed (some minimal containers, Windows), this produces a confusing `exec: "vi": executable file not found in $PATH` error. **Absent** — no check that the editor binary exists before attempting to run it.

### Silent suppression (what errors are swallowed?)

- **`resolveDirectiveContext` silently ignores cwd error:** `directive_show.go:97-99` calls `os.Getwd()` and only uses the result if `err == nil`. If `Getwd` fails (deleted cwd, broken mount), `rigName` stays empty and the command silently operates at town-level scope with no indication that rig detection failed. **Absent** — no warning emitted; the user sees town-level output and may not realize rig scope was intended.
- **`directive list` silently skips unreadable rig directories:** `directive_list.go:59-88` scans `os.ReadDir(townRoot)` and silently continues past any rig whose `directives/` subdirectory cannot be read. A permissions error on a rig directory produces no output for that rig, no error, no warning. **Absent** — the user sees a complete-looking list that may be missing entries from inaccessible rigs.

## Outgoing calls

### Subprocess invocations
| Called binary | Command | Flags | Flag source | `file:line` |
|---|---|---|---|---|
| `$EDITOR` (or `vi`) | `<path>` | — | `EDITOR` env var, fallback `vi` | `directive_edit.go:84` |

### Config file writes
| Target | Operation | Value | Purpose | `file:line` |
|---|---|---|---|---|
| `<townRoot>/<rig>/directives/<role>.md` | `os.WriteFile` | header template | create new directive file if absent | `directive_edit.go:72` |

## Related

- [gt](../binaries/gt.md) — parent binary.
- [README.md](README.md) — command tree index.
- [prime.md](prime.md) — `Long` text says directives are "injected at
  prime time", so `gt prime` is the consumer.
- [config.md](config.md) — sibling in the Configuration group that also
  affects role/agent behavior but through a different mechanism (town
  settings vs. markdown overlays).

## Notes / open questions

- **Phase 3 Batch 1b wiki-stale correction (2026-04-15):** the Phase 2
  body of this page claimed `show`, `edit`, and `list` were "not wired
  up here" and speculated they were "either defined in another file
  not mapped yet or not implemented." Re-read at gastown HEAD
  `9f962c4a` confirmed they ARE wired, via sibling files
  (`directive_show.go`, `directive_edit.go`, `directive_list.go`),
  each registering itself on `directiveCmd` in its own `init()`. The
  body above has been rewritten to reflect the actual wiring; the
  original Phase 2 description was incorrect at Phase 2 time (the
  sibling files already existed at v1.0.0) rather than drift against
  later churn.
- The `Long` text claims directives "override formula defaults where
  they conflict" — no code-grounded reference yet for how that override
  actually happens. Tracing from `config.LoadRoleDirective`
  (`directive_show.go:54`) would answer it.
- A full understanding of `directive` needs an entity page for the
  `prime` consumer path and a concept page for directives generally
  (not yet written).
