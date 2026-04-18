---
title: gt theme
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/cmd/theme.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, configuration, tmux, cli-colors, theming]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase5_audience: user
phase8_audited: 2026-04-17
phase8_findings: [precondition-violation, silent-suppression]
---

# gt theme

Manages two distinct theming surfaces in a single command: (1) tmux
status-bar theme for the current rig, and (2) CLI-output color mode
(dark/light/auto) for all `gt` commands via the `theme cli`
subcommand.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupConfig` ("Configuration")
**Polecat-safe:** no (no `AnnotationPolecatSafe` on the cobra.Command at
`/home/kimberly/repos/gastown/internal/cmd/theme.go:26-42`)
**Beads-exempt:** no (not in `beadsExemptCommands` on `root.go:44-77`)
**Branch-check-exempt:** no (not in `branchCheckExemptCommands` on
`root.go:81-91`)

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/theme.go:26-446`.

### Invocation

```
gt theme                     # show current rig tmux theme
gt theme --list              # list available tmux themes
gt theme <name>              # set tmux theme for current rig
gt theme none                # disable tmux theming for this rig
gt theme apply [--all]       # apply theme to running tmux sessions
gt theme cli                 # show CLI color scheme
gt theme cli <mode>          # set CLI color scheme (auto|dark|light)
```

### Two-theme architecture

This command straddles two storage locations with different shapes:

- **Rig tmux theme** — persisted in
  `<townRoot>/<rig>/settings/config.json` as the `Theme` field of
  `config.RigSettings` (see `theme.go:278-303`, `saveRigTheme`
  at `:306-349`).
- **CLI color scheme** — persisted in the town-wide settings file
  returned by `config.TownSettingsPath(townRoot)` as the `CLITheme`
  field (`theme.go:360-419`, `runThemeCLI`). Overrideable per
  session via `GT_THEME` env var (`theme.go:376-380`).

They share the word "theme" but have no overlap.

### Behavior — `runTheme` (`theme.go:87-135`)

Dispatches on flags / args:

1. **`--list`** (`theme.go:89-100`): enumerates tmux themes via
   `tmux.ListThemeNames()`, printing `name  style`. Also appends
   `none` and the `Mayor` theme (which is listed as "Mayor only").
2. **No args** (`theme.go:109-114`): resolves current rig via
   `detectCurrentRig()` (`:230-269`), prints
   `Rig: <name>\nTheme: <describeRigTheme(rig)>`.
3. **One arg (theme name)** (`theme.go:116-134`): validates via
   `tmux.GetThemeByName(themeName)` (except the special `"none"`
   case), calls `saveRigTheme(rigName, themeName)`, prints success
   line, reminds to run `gt theme apply`.

### `detectCurrentRig` (`theme.go:230-269`)

Fallback chain:
1. `GT_RIG` environment variable.
2. `detectCurrentSession()` (shared helper defined in `issue.go`)
   parsed via `session.ParseSessionName` for its `Rig` field.
3. Path-based detection: `workspace.FindFromCwd()` →
   `filepath.Rel(townRoot, cwd)` → first path component, excluding
   `.`, `mayor`, and `deacon`.

### `describeRigTheme` (`theme.go:271-303`)

Cascade:
- No workspace / load error → `AssignTheme(rigName)` default + "default
  auto-assignment".
- No `settings.Theme` → default auto-assignment.
- `settings.Theme.Disabled` → `"none (configured)"`.
- `settings.Theme.Custom != nil` → `"custom (bg=..., fg=...)"`.
- Named theme: if `GetThemeByName` resolves, `"<name> (<style>,
  configured)"`; otherwise `"<name> (configured)"`.
- Fallback: auto-assignment.

### `runThemeApply` (`theme.go:137-227`)

Walks all tmux sessions via `tmux.NewTmux().ListSessions()`, filters
to `session.IsKnownSession` (`theme.go:152-155`), and for each known
session:

1. Parses the session name via `session.ParseSessionName`
   (`theme.go:161-164`).
2. Dispatches on `identity.Role`:
   - `RoleMayor` → `tmux.ResolveSessionTheme(townRoot, "",
     RoleMayor)`
   - `RoleDeacon` → same with `RoleDeacon`
   - Else → use `identity.Rig` and
     `tmux.ResolveSessionTheme(townRoot, rig, role)`. Skips if
     `!themeApplyAllFlag && currentRig != rig`.
3. Worker label is computed per role
   (`theme.go:181-193`): Witness, Refinery use their role string,
   Crew uses `identity.Name`.
4. Window tint is resolved from `session.ResolveWindowTint(rig,
   role)` with a fallback to the theme's BG/FG if tint is enabled
   for the rig (`theme.go:200-205`).
5. `t.ConfigureGasTownSession(sess, theme, rig, worker, role)`
   applies it, printing per-session status (`theme.go:207-216`).
6. Summary line: `Applied theme to N session(s)` or `No matching
   sessions found` (`theme.go:220-225`).

### `runThemeCLI` (`theme.go:351-429`)

1. Requires a workspace; errors out if not in one.
2. **Show mode** (`theme.go:363-398`): loads town settings via
   `config.LoadOrCreateTownSettings`. Reads `CLITheme` (empty =
   `auto`), optionally overlays `GT_THEME` env. Prints
   `Configured / Override / Effective` block. In `auto` mode,
   additionally calls `detectTerminalBackground()` (`:442-445`,
   `termenv.HasDarkBackground()`) and prints `Detected: <light|dark>
   background`.
3. **Set mode** (`theme.go:401-427`): lowercases arg, validates via
   `isValidCLITheme` (`:432-439`; accepts `auto|dark|light`),
   updates `settings.CLITheme`, saves.

### Subcommands

Registered in `init()` (`theme.go:78-85`):

- `apply` — apply theme to running sessions (terminal-level)
- `cli [mode]` — manage CLI color scheme

### Flags

| Flag            | Scope          | Type | Default | Purpose                                  |
|-----------------|----------------|------|---------|------------------------------------------|
| `--list` / `-l` | `theme` root   | bool | `false` | List available tmux themes               |
| `--all` / `-a`  | `theme apply`  | bool | `false` | Apply to all rigs, not just current      |

## Failure modes

### Precondition violations (what does it assume?)

- **`runTheme` may save theme for rig "unknown":** `theme.go:103-106` calls `detectCurrentRig()` and defaults to `"unknown"` if no rig is detected. If the user then passes a theme name, `saveRigTheme("unknown", themeName)` at `theme.go:123` writes a rig settings file at `<townRoot>/unknown/settings/config.json`. This creates a real directory named `unknown` in the town root. **Absent** — predicted bug surface: running `gt theme forest` outside a rig creates a spurious `unknown/` directory tree.
- **`saveRigTheme` creates rig settings directory if absent:** `theme.go:320-327` calls `config.LoadRigSettings` and on `IsNotExist` creates new settings. Combined with the "unknown" rig issue above, this means the directory is created eagerly. For legitimate rigs, this is fine; for the "unknown" fallback, it's a side effect. **Present** — the `IsNotExist` check is intentional for new rigs, but the "unknown" fallback is the real issue.

### Silent suppression (what errors are swallowed?)

- **`runThemeApply` discards `workspace.FindFromCwd()` error:** `theme.go:139` uses `townRoot, _ := workspace.FindFromCwd()`, discarding the error. If the workspace is not found, `townRoot` is `""`, and `tmux.ResolveSessionTheme("", rig, role)` at `theme.go:196` receives an empty town root. The theme resolution falls back to auto-assignment, but the user gets no indication that workspace resolution failed. **Absent** — predicted bug surface: running `gt theme apply` outside a workspace applies auto-assigned themes instead of configured themes, with no error or warning.
- **`runThemeApply` silently skips sessions with unparseable names:** `theme.go:162-164` catches `session.ParseSessionName` errors and `continue`s to the next session. Non-Gas-Town sessions are expected to be skipped (via `IsKnownSession` at `:153`), but a Gas Town session with a malformed name would pass `IsKnownSession` and then be silently dropped by the parse failure. **Present** — filtering is intentional, but the silent skip of a session that passed the first filter could mask a bug.

## Related

- [gt](../binaries/gt.md) — parent binary. `initCLITheme()` in
  `persistentPreRun` (`root.go:110`) reads the same `CLITheme` field
  that `theme cli` writes, so the setting takes effect on the next
  command invocation.
- [README.md](README.md) — command tree index.
- [config.md](config.md) — sibling Configuration command that also
  exposes `config set cli_theme <dark|light|auto>` — `config set`
  and `theme cli` are **two different code paths writing the same
  field**. See Notes.
- [status.md](status.md) — status reads rig state; a theme-savvy
  status line is plausible but not verified.
- [doctor.md](doctor.md) — the doctor check catalog at
  `doctor.go:215` includes a `theme` check (per the registered-check
  list documented in that page).
- [issue.md](issue.md) — shares the `detectCurrentSession()` helper
  (referenced from `theme.go:237`).

## Notes / open questions

- **Drift hazard: two writers for `CLITheme`**. `theme cli dark`
  (`theme.go:414`) and `config set cli_theme dark` (`config.go:754-760`)
  both write the same town-settings field. They agree today, but
  any future drift between the validators (`isValidCLITheme` vs.
  the `switch` in `config set`) will produce inconsistent behavior.
  Not yet documented as a known drift — noted neutrally.
- `initCLITheme` in the root `persistentPreRun` reads this setting
  for every command. Worth a cross-link from
  `../binaries/gt.md`.
- The `theme apply` skip-unless-same-rig logic without `--all` means
  a user running it at town root (no `GT_RIG`) will skip all rig
  sessions silently. Behavior is intended, but the summary message
  "No matching sessions found" does not hint why.
- `GT_THEME` env var should be captured in an env-vars rollup page.
- `Mayor theme` is listed as a pseudo-theme in `--list` output
  (`theme.go:97-98`) but the `apply` path uses
  `ResolveSessionTheme(townRoot, "", RoleMayor)` which resolves
  based on role, not the `MayorTheme()` directly. Worth tracing
  whether these two agree.
