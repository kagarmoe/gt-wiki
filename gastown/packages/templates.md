---
title: internal/templates
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-18
sources:
  - /home/kimberly/repos/gastown/internal/templates/templates.go
  - /home/kimberly/repos/gastown/internal/templates/townroot.go
  - /home/kimberly/repos/gastown/internal/templates/commands/provision.go
tags: [package, templates, embed, roles, supervisor, claude-md]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase8_audited: 2026-04-17
phase8_findings: [none]
detail_depth: {params: 2, data_flow: 2, errors: 1, side_effects: 1}
---

# internal/templates

All embedded text assets that Gas Town needs to render at runtime:
role-context CLAUDE.md templates (mayor, witness, refinery, polecat,
crew, deacon, boot), agent-to-agent message templates (spawn, nudge,
escalation, handoff), supervisor unit files (launchd plist, systemd
service), and the town-root and polecat CLAUDE.md templates. Wraps all
of them with `text/template` rendering and a `{{ cmd }}` function that
substitutes the user's configured `GT_COMMAND` name so the templates
work under a rebranded binary (important when the real `gt` is
Graphite's).

**Go package path:** `github.com/steveyegge/gastown/internal/templates`
(plus the subpackage
`github.com/steveyegge/gastown/internal/templates/commands`)
**File count (top-level):** 2 go files (`templates.go`,
`townroot.go`), 1 test file. Plus subdirectories `commands/`
(1 additional Go subpackage), `roles/`, `messages/`, `launchd/`,
`systemd/`, `townroot/`, each holding template assets.
**Embedded asset directories:**

- `roles/*.md.tmpl` — one file per role (boot, crew, deacon, dog,
  mayor, polecat, refinery, witness).
- `messages/*.md.tmpl` — escalation, handoff, nudge, spawn.
- `launchd/com.gastown.daemon.plist` — macOS supervisor unit.
- `systemd/gastown-daemon.service` — Linux supervisor unit.
- `townroot/claude.md` — the town-root CLAUDE.md template.
- `polecat-CLAUDE.md` — the polecat worktree CLAUDE.md overlay
  (embedded as a plain string, not a template, but with
  `{{rig}}`/`{{name}}` string substitutions done manually).
- `commands/bodies/*.md` — slash-command bodies for `done`, `handoff`,
  `review`, consumed by the `commands` subpackage.

**Imports (notable):** stdlib (`embed`, `text/template`, `os`,
`os/exec`, `path/filepath`, `runtime`, `sync`, `bytes`),
[`internal/cli`](cli.md) for `cli.Name()` in `townroot.go`, and
`templates/commands` for slash-command provisioning.
**Imported by (notable):** [`internal/config`](config.md) (which loads
templates), `gt install` and `gt doctor --fix` (CreateMayorCLAUDEmd),
the polecat spawn path (CreatePolecatCLAUDEmd), the daemon install path
(ProvisionSupervisor), and anything that renders role contexts or
agent-to-agent messages.

## What it actually does

### Command-name substitution

`CmdName()` (`templates.go:27-35`) returns `$GT_COMMAND` if set,
otherwise `"gt"`. It's cached via `sync.Once` so the env var is read
exactly once per process. The template `FuncMap` at
`templates.go:38-40` exposes it as `{{ cmd }}` so every role and
message template can emit the correct binary name without hard-coding
`gt`. This is what makes the whole templates package compatible with
rebranded installations.

### The `Templates` facade (`templates.go:52-175`)

- `New()` (`templates.go:124-142`) parses both role and message
  templates from the embedded FS with the `cmd` function attached.
- `RenderRole(role, data RoleData)` (`templates.go:145-154`) looks up
  `<role>.md.tmpl` and executes it. Role data carries the role name,
  rig name, town root/name, working directory, default branch,
  polecat/dog names, beads dir/prefix, and dynamic mayor/deacon
  session names (see `RoleData` at `templates.go:58-72`).
- `RenderMessage(name, data interface{})` (`templates.go:157-166`) —
  same pattern for messages. Each message has a typed data struct
  alongside: `SpawnData` (`templates.go:75-83`), `NudgeData`
  (`:86-93`), `EscalationData` (`:96-103`), `HandoffData`
  (`:106-115`).
- `RoleNames()` returns the canonical list of role templates
  (`templates.go:170` — mayor, witness, refinery, polecat, crew,
  deacon, boot) and `MessageNames()` returns the four message
  templates. These are hard-coded lists that must stay in sync with
  the files in `roles/` and `messages/`.

### Mayor CLAUDE.md creation (`templates.go:183-213`)

`CreateMayorCLAUDEmd(mayorDir, townRoot, townName, mayorSession,
deaconSession) (created bool, err error)` writes `<mayorDir>/CLAUDE.md`
from the mayor role template, but **preserves an existing file**
(`:186-191`) so user customizations survive `gt doctor --fix`. Returns
`(false, nil)` when the file already exists.

### Polecat CLAUDE.md creation (`templates.go:215-269`)

`CreatePolecatCLAUDEmd(worktreePath, rigName, polecatName)` is more
elaborate. The polecat template is an overlay containing lifecycle
instructions (`gt done`, etc.) and must land in the polecat's worktree
or the agent won't know how to finish its work.

Three behaviors:

1. If either `CLAUDE.md` or `CLAUDE.local.md` in the worktree already
   contains `PolecatLifecycleMarker = "IDLE POLECAT HERESY"`
   (`templates.go:220`), skip — already installed. The marker string is
   deliberately unusual so it doesn't collide with real content.
2. If `CLAUDE.md` exists (typically a tracked repo file), write the
   overlay to `CLAUDE.local.md` instead
   (`templates.go:256-265`) so the tracked file stays untouched. This
   matters because `gt done`'s auto-save safety net would otherwise
   sweep the new CLAUDE.md content into the polecat's branch and
   pollute the PR diff with hundreds of lines of agent context
   (`:226-232`).
3. If no `CLAUDE.md` exists at all, write the full template to
   `CLAUDE.md` (`:267-268`).

Template substitutions are done via `strings.ReplaceAll` on `{{rig}}`
and `{{name}}` (`templates.go:240-242`) — **not** through `text/
template` — because the embedded file is stored as a raw string, not
a parsed template. This is an inconsistency with the role/message
path.

### Town-root CLAUDE.md (`townroot.go`)

- `TownRootCLAUDEmd() string` (`townroot.go:19-21`) — returns the
  embedded `townroot/claude.md` with `{{cmd}}` replaced via
  `strings.ReplaceAll` using `cli.Name()` (not the `CmdName()` in
  `templates.go`). This is a second code path for command-name
  substitution; see Drift.
- `TownRootCLAUDEmdVersion = 1` (`townroot.go:15`) — manual version
  number to bump when the template gains new sections. There is no
  automatic detection.
- `TownRootRequiredSections()` (`townroot.go:31-42`) declares which
  headings must appear in an installed town-root `CLAUDE.md` for the
  agent behavior to work: "Dolt Server" (at `## Dolt Server`) and
  "Communication hygiene" (at `### Communication hygiene`). This is
  the list that `gt doctor` uses to detect drift between a user's
  edited CLAUDE.md and the canonical one.

### Supervisor provisioning (`templates.go:313-438`)

`ProvisionSupervisor(townRoot)` dispatches by `runtime.GOOS`:

- **darwin** → `provisionLaunchd` writes
  `~/Library/LaunchAgents/com.gastown.daemon.plist` from the embedded
  template, `launchctl unload`s (best-effort) any existing copy, then
  `launchctl load`s the new one.
- **linux** → `provisionSystemd` writes
  `$XDG_DATA_HOME/systemd/user/gastown-daemon.service` (or
  `~/.local/share/...`), then runs `systemctl --user daemon-reload`,
  `enable`, and `start`.
- **anything else** → returns a "not supported yet" message without an
  error, so the install path degrades gracefully.

Both paths template-expand `SupervisorData{GTPath, TownRoot}` into the
embedded unit file, where `GTPath` is the result of `os.Executable()`
so the supervisor launches the exact binary that ran `gt install`.

### Command provisioning (`templates/commands` subpackage)

The `commands` subpackage (`provision.go`) handles agent-specific
slash commands (`done`, `handoff`, `review`). Each command has a body
under `bodies/*.md` and an `AgentFields` map keyed by agent name
(`claude`, `opencode`, ...) that injects agent-specific frontmatter
like `allowed-tools` and `argument-hint` (`provision.go:41-73`).

- `BuildCommand(cmd, agent)` (`provision.go:76-96`) — assembles
  frontmatter + body as one string.
- `ProvisionFor(workspacePath, agent)` (`provision.go:99-130`) — writes
  every command to `<workspacePath>/<agent-config-dir>/commands/<name>.md`,
  skipping existing files. The agent config dir comes from
  [`config.GetAgentPresetByName`](config.md), so adding a new agent
  means registering a preset there rather than editing this package.
- `MissingFor(workspacePath, agent)` (`provision.go:133-151`) — inverse:
  returns the names of commands not yet installed. The top-level
  `templates` package exposes this as
  `HasCommands`/`MissingCommands` wrappers at `templates.go:290-307`.

## Docs claim

The package comment (`templates.go:1`) describes the package as
"embedded templates for role contexts and messages" — accurate but
understates the scope, which also covers supervisor units, town-root
CLAUDE.md, polecat overlays, and slash commands. `PolecatLifecycleMarker`'s
doc comment (`templates.go:215-220`) explicitly explains the marker's
role in detecting whether an existing CLAUDE.md is already "overlay'd."

## Drift

- **Two command-name substitution paths.** `templates.go` uses `CmdName()`
  (reads `GT_COMMAND`) threaded through the `{{ cmd }}` template
  function. `townroot.go` uses `cli.Name()` for the same purpose via
  `strings.ReplaceAll`. If these two ever drift (e.g., `cli.Name()`
  gains priority logic not reflected in `CmdName`), the town-root
  CLAUDE.md and the role templates will render different binary names.
  Worth a lint pass.
- **Polecat overlay is raw-string templated.** The polecat CLAUDE.md is
  the only embedded asset not parsed by `text/template`. It uses
  `{{rig}}` and `{{name}}` as literal substrings replaced via
  `strings.ReplaceAll` (`templates.go:240-242`). Any future need for a
  conditional or loop in the polecat overlay would require converting
  it to a real template — minor but a foot-gun.
- **`RoleNames()`/`MessageNames()` hard-coded lists.** Both return
  compile-time constant slices (`templates.go:170, 174`). Adding a new
  file under `roles/` or `messages/` requires editing both the file
  list and these functions. The embedded FS knows the real list but
  isn't consulted.

## Outgoing calls

### Subprocess invocations
| Called binary | Command | Flags | Flag source | `file:line` |
|---|---|---|---|---|
| `launchctl` | `unload` | `<plistPath>` (macOS) | runtime | `templates.go:371` |
| `launchctl` | `load` | `<plistPath>` (macOS) | runtime | `templates.go:374` |
| `systemctl` | `--user daemon-reload` | (none) (Linux) | hardcoded | `templates.go:423` |
| `systemctl` | `--user enable` | `gastown-daemon.service` (Linux) | hardcoded | `templates.go:428` |
| `systemctl` | `--user start` | `gastown-daemon.service` (Linux) | hardcoded | `templates.go:433` |

### File writes
| Target | What is written | Purpose | `file:line` |
|---|---|---|---|
| CLAUDE.md | generated content | Town CLAUDE.md template | `templates.go:212` |
| settings.local.json | merged JSON | Claude local settings | `templates.go:261` |
| settings.local.json | JSON content | Claude local settings (new) | `templates.go:264` |
| CLAUDE.md | content | CLAUDE.md template (fallback) | `templates.go:268` |
| launchd plist | plist XML | macOS daemon supervisor | `templates.go:366` |
| systemd service | unit content | Linux daemon supervisor | `templates.go:418` |
| provision script | shell script | Provision commands | `commands/provision.go:124` |

## Notes / open questions

- `ProvisionSupervisor` writes the plist/service with `0644`. Per-user
  LaunchAgents and `--user` systemd services typically want this
  permission, but it's worth flagging because a misconfigured supervisor
  is hard to debug.
- `launchctl unload` is called with output discarded (`templates.go:371`).
  That's intentional ("ignore errors" comment) but means an "I can't
  unload the old copy because of SIP" style failure goes unreported.
- Subdirectory layout is five levels deep from the package root
  (`templates/commands/bodies/*.md`) — notable because the wiki schema
  caps its own nesting at three levels. The source tree isn't subject
  to wiki rules, just noting.

## Related wiki pages

- [gt](../binaries/gt.md) — top-level binary.
- [internal/config](config.md) — loads templates and registers agent
  presets used by `commands.ProvisionFor`.
- [internal/cli](cli.md) — provides `cli.Name()` used by
  `TownRootCLAUDEmd`.
- [go-packages inventory](../inventory/go-packages.md).
