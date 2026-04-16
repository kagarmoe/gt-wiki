---
title: gt namepool
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/cmd/namepool.go
tags: [command, workspace, naming, themes, polecat]
phase3_audited: 2026-04-15
phase3_findings: [cobra-drift]
phase3_severities: [wrong]
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
---

# gt namepool

Manage the themed name pool a rig draws on to name its polecats.
Show pool status, switch themes, add custom names, and create or
delete custom themes.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupWorkspace` ("Workspace") (`namepool.go:24`)
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/namepool.go`
(520 lines). Registration: `namepool.go:110-120` attaches six
subcommands to `namepoolCmd` and registers the parent on
`rootCmd`.

The parent `namepoolCmd` (`namepool.go:22-39`) has its own
`RunE: runNamepool` — so `gt namepool` with no subcommand prints
the **current pool status** for the rig inferred from the cwd,
rather than a usage line.

### Invocation

```
gt namepool               [--list|-l]
gt namepool themes        [<theme>]
gt namepool set <theme>
gt namepool add <name>
gt namepool reset
gt namepool create <name> [names...] [--from-file <path>]
gt namepool delete <name>
```

### Subcommands

| subcommand | var | source | run-fn |
|---|---|---|---|
| (bare)   | `namepoolCmd`        | `namepool.go:22-39`  | `runNamepool`       (`namepool.go:122-183`) |
| `themes` | `namepoolThemesCmd`  | `namepool.go:41-49`  | `runNamepoolThemes` (`namepool.go:185-244`) |
| `set`    | `namepoolSetCmd`     | `namepool.go:51-60`  | `runNamepoolSet`    (`namepool.go:246-298`) |
| `add`    | `namepoolAddCmd`     | `namepool.go:62-71`  | `runNamepoolAdd`    (`namepool.go:300-367`) |
| `reset`  | `namepoolResetCmd`   | `namepool.go:73-81`  | `runNamepoolReset`  (`namepool.go:369-390`) |
| `create` | `namepoolCreateCmd`  | `namepool.go:83-97`  | `runNamepoolCreate` (`namepool.go:419-463`) |
| `delete` | `namepoolDeleteCmd`  | `namepool.go:99-108` | `runNamepoolDelete` (`namepool.go:465-490`) |

### Bare `gt namepool` — pool status

`runNamepool` (`namepool.go:122-183`):

- If `--list`/`-l` is set, just calls `runNamepoolThemes` with no
  args (listing all themes) and returns.
- Otherwise it requires being in a rig directory. `detectCurrentRigWithPath`
  (`namepool.go:392-417`) walks up from cwd to find the town root
  via `workspace.FindFromCwd`, then returns the first path
  component as the rig name — excluding `mayor` and `deacon`
  (`namepool.go:412`).
- Loads `<rig>/settings/config.json` via `config.LoadRigSettings`
  and constructs a `polecat.NamePool`, preferring any
  rig-configured style, custom names, and
  `MaxBeforeNumbering`. Falls back to `polecat.NewNamePool(rigPath,
  rigName)` with defaults if settings are absent.
- `pool.Load()` reads the live pool state. If the pool file
  doesn't exist yet, prints a synthetic empty summary (theme
  default, 0 polecats, `polecat.DefaultPoolSize` max). Otherwise
  prints rig, theme (annotated `built-in` or `custom`),
  polecat count, list of in-use names, and a `"configured in
  settings/config.json"` footer when rig settings contain a
  namepool block (`namepool.go:178-180`).

### `themes` — list / inspect

`runNamepoolThemes` (`namepool.go:185-244`):

- With no args: looks up the town root and calls
  `polecat.ListAllThemes(townRoot)` to enumerate built-ins plus any
  custom themes in `<town>/settings/themes/*.txt`. For each, prints
  name, label (`custom, ` or blank), total count, and a
  10-name preview.
- With one arg: resolves names via `polecat.ResolveThemeNames` (if
  a town root is available) or `polecat.GetThemeNames` (built-in
  only), formatted five per row.

### `set <theme>`

`runNamepoolSet` (`namepool.go:246-298`):

- Requires being in a rig directory.
- Validates the theme exists via `polecat.ResolveThemeNames` — so
  `set` with an unknown theme produces
  `"unknown theme: <name> (use 'gt namepool themes' to list available themes)"`.
- Constructs / loads the pool, calls `pool.SetTheme(theme)`, and
  persists it to the pool file via `pool.Save()`.
- **Also** writes to `<rig>/settings/config.json` via
  `saveRigNamepoolConfig` (`namepool.go:493-520`), preserving any
  previously-configured custom names. So the theme is stored in
  two places: the runtime pool and the settings config.

### `add <name>`

`runNamepoolAdd` (`namepool.go:300-367`) — the most nuanced
subcommand:

- Lowercases and validates the name via `polecat.ValidatePoolName`.
- Loads `settings/config.json` (creates a new one if not found).
- **Custom-theme branch** (`namepool.go:330-347`). If the rig
  currently uses a **custom** theme and has no per-rig name
  overrides, `name` is appended to the underlying theme file via
  `polecat.AppendToCustomTheme(townRoot, style, name)`. The
  comment at `namepool.go:331-332` explains why: "This prevents a
  single `add` from shadowing the entire custom theme."
- **Built-in or override branch** (`namepool.go:349-366`). If the
  theme is built-in, or the rig already has per-rig overrides,
  appends to `settings.Namepool.Names` instead, dedup-checking
  against the existing list. Saves via
  `config.SaveRigSettings`.

### `reset`

`runNamepoolReset` (`namepool.go:369-390`) — clears all claimed
names in the pool and re-saves. Does **not** change the theme or
touch custom names in settings.

### `create` / `delete` — custom-theme files

`runNamepoolCreate` (`namepool.go:419-463`):

- Requires the proposed theme name is not a built-in and passes
  `ValidatePoolName`.
- Names come from positional args or from `--from-file <path>`
  via `polecat.ParseThemeFile`.
- Each positional name is lowercased and validated.
- Writes the theme via `polecat.SaveCustomTheme(townRoot, name,
  names)` — which drops a file at `<town>/settings/themes/<name>.txt`
  per the Long help on `namepoolCreateCmd`.

`runNamepoolDelete` (`namepool.go:465-490`):

- Validates the theme name to prevent path traversal (the comment
  at `namepool.go:468` flags this).
- Warns on stderr if any rigs are currently using the theme,
  listing them, and noting they will fall back to
  `polecat.DefaultTheme`. Deletion proceeds either way.
- Calls `polecat.DeleteCustomTheme(townRoot, themeName)`.

### Flags

| flag | type | default | scope | source |
|---|---|---|---|---|
| `--list` / `-l` | bool | `false` | bare `namepool` | `namepool.go:118` |
| `--from-file <path>` | string | `""` | `create` | `namepool.go:119` |

## Related commands

- [polecat.md](polecat.md) — polecats are the consumers of the
  name pool. Every new polecat claims a name from whatever theme
  this rig is currently configured with.
- [rig.md](rig.md) — rigs are the unit of namepool scope. Each
  rig has its own pool state and settings.
- [crew.md](crew.md) — crew workers **do not** use the namepool;
  their names come from positional arguments to
  [crew.md](crew.md) `add`.
- [config.md](config.md) — for general rig config.

## Docs claim

### Source
- `internal/cmd/namepool.go:31-38` — Cobra `Long` text, "Examples:" block

### Verbatim
> Examples:
>   gt namepool              # Show current pool status
>   gt namepool --list       # List available themes
>   gt namepool themes       # Show theme names
>   gt namepool set minerals # Set theme to 'minerals'
>   gt namepool add ember    # Add custom name to pool
>   gt namepool reset        # Reset pool state

## Drift

### namepoolCmd.Long "Examples:" shows 6 operations; 8 subcommands registered
- **Claim source:** Cobra `Long` text at `internal/cmd/namepool.go:31-38`
- **Docs claim:** The "Examples:" block demonstrates 6 operations (bare, --list, themes, set, add, reset), implying those are the available subcommands.
- **Code does:** `namepool.go:111-117` registers 6 named subcommands (themes, set, add, reset, **create**, **delete**) plus the bare RunE. The Examples omit `create` (`namepool.go:83-97`) and `delete` (`namepool.go:99-108`), which are the custom-theme management commands. Users reading only the Long text would not discover they can create or delete custom themes.
- **Category:** `cobra drift`
- **Severity:** `wrong`
- **Fix tier:** `code` — add `create` and `delete` examples to `namepoolCmd.Long`
- **Release position:** `in-release` — both subcommands present at v1.0.0

See [gastown/drift/README.md](../drift/README.md) for the consolidated corrections list.

## Notes / open questions

- **Two sources of truth.** The live pool file (managed by
  `polecat.NamePool`) and `settings/config.json` can diverge if
  users edit one without the other. `set` tries to keep them in
  sync; `add` writes to the theme file or `settings/config.json`
  depending on mode. Manual edits are risky.
- **No dedicated "remove name" subcommand.** You can add to the
  pool but not remove. `reset` is the sledgehammer, and
  `delete` operates on whole themes, not names within a theme.
- **Name collisions during custom-theme add** are reported as
  `"Name '<name>' already in theme '<theme>'"` — but the function
  uses `polecat.AppendToCustomTheme`'s `alreadyExists` return
  value (`namepool.go:340-344`). That implies the append is
  idempotent at the polecat-package level — worth confirming
  when a `packages/polecat.md` page lands.
- **`detectCurrentRigWithPath` excludes `mayor` and `deacon`**
  but not other agent directories (`witness`, `refinery`,
  `plugins`, etc.). If cwd happens to be `deacon/<something>`, the
  function returns empty. If cwd is `plugins/<something>`, it
  treats `plugins` as a rig name — likely a latent bug. Worth a
  drift note once a plugin workflow page lands.
- **`--from-file` file format** is `polecat.ParseThemeFile`'s job —
  not documented here.
