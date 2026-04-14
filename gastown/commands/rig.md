---
title: gt rig
type: command
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/cmd/rig.go
  - /home/kimberly/repos/gastown/internal/cmd/rig_config.go
  - /home/kimberly/repos/gastown/internal/cmd/rig_detect.go
  - /home/kimberly/repos/gastown/internal/cmd/rig_dock.go
  - /home/kimberly/repos/gastown/internal/cmd/rig_helpers.go
  - /home/kimberly/repos/gastown/internal/cmd/rig_park.go
  - /home/kimberly/repos/gastown/internal/cmd/rig_quick_add.go
  - /home/kimberly/repos/gastown/internal/cmd/rig_settings.go
tags: [command, workspace, rig, domain-noun, lifecycle, beads-exempt]
---

# gt rig

Manage [**rigs**](../concepts/rig.md) — the project containers that
hold a cloned source repo, agent workspaces
([Witness](../roles/witness.md), [Refinery](../roles/refinery.md),
[Mayor](../roles/mayor.md) clone), [crew](../roles/crew.md)
workspaces, [polecat](../roles/polecat.md) workspaces, and
rig-level beads tracking.

This page documents the `gt rig` CLI surface; see the
[rig concept page](../concepts/rig.md) for the domain model.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupWorkspace` ("Workspace") (`rig.go:38`)
**Polecat-safe:** no
**Beads-exempt:** yes (in `beadsExemptCommands` on
`root.go:44-77`; operators must be able to list, park, and shut
down rigs even if beads is broken)
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/rig.go` (2481
lines) owns the core subcommands plus `rigCmd` itself. Several
large subcommands live in sibling files:

| file | contains |
|---|---|
| `rig.go`            | `rigCmd`, add, list, remove, reset, boot, start, reboot, shutdown, status, stop, restart, menu (hidden) |
| `rig_config.go`     | `rigConfigCmd` (show/set/unset) |
| `rig_detect.go`     | `rigDetectCmd` — detect rig from cwd |
| `rig_dock.go`       | `rigDockCmd`, `rigUndockCmd` — dock/undock lifecycle state |
| `rig_helpers.go`    | shared helpers (`getRig`, operational state, LED rendering) |
| `rig_park.go`       | `rigParkCmd`, `rigUnparkCmd` — park/unpark lifecycle state |
| `rig_quick_add.go`  | `rigQuickAddCmd` — minimal-args rig registration |
| `rig_settings.go`   | `rigSettingsCmd` (show/set/unset) |
| `*_test.go`         | tests |

The parent `rigCmd` (`rig.go:36-50`) uses `RunE: requireSubcommand`
as its fallback, so `gt rig` with no arguments prints the
subcommand list.

### Invocation

```
gt rig add      <name> <git-url>  [--prefix <p>] [--local-repo <path>]
                                   [--branch <b>] [--push-url <url>] [--upstream-url <url>]
                                   [--adopt] [--url <url>] [--force]
                                   [--filter <f>] [--sparse-checkout <paths>]
gt rig list                       [--json]
gt rig remove   <name>            [-f|--force]
gt rig reset                      [--handoff] [--mail] [--stale]
                                   [--dry-run] [--role <role>]
gt rig boot     <rig>
gt rig start    <rig>...
gt rig reboot   <rig>             [-f|--force] [--nuclear]
gt rig shutdown <rig>             [-f|--force] [--nuclear]
gt rig status   [rig]
gt rig stop     <rig>...          [-f|--force] [--nuclear]
gt rig restart  <rig>...          [-f|--force] [--nuclear]
gt rig menu                       (hidden — tmux popup)
gt rig park     <rig>             (see rig_park.go)
gt rig unpark   <rig>             (see rig_park.go)
gt rig dock     <rig>             (see rig_dock.go)
gt rig undock   <rig>             (see rig_dock.go)
gt rig detect                     (see rig_detect.go — detect rig from cwd)
gt rig quick-add <args...>        (see rig_quick_add.go)
gt rig config   show|set|unset    (see rig_config.go)
gt rig settings show|set|unset    (see rig_settings.go)
```

### Subcommand inventory

Defined in `rig.go`:

| subcommand | var | source | run-fn |
|---|---|---|---|
| `add`      | `rigAddCmd`      | `rig.go:52-83`    | `runRigAdd` (elsewhere in `rig.go`) |
| `list`     | `rigListCmd`     | `rig.go:85-100`   | `runRigList` |
| `remove`   | `rigRemoveCmd`   | `rig.go:102-122`  | `runRigRemove` |
| `reset`    | `rigResetCmd`    | `rig.go:124-138`  | `runRigReset` |
| `boot`     | `rigBootCmd`     | `rig.go:140-156`  | `runRigBoot` |
| `start`    | `rigStartCmd`    | `rig.go:158-177`  | `runRigStart` |
| `reboot`   | `rigRebootCmd`   | `rig.go:179-192`  | `runRigReboot` |
| `shutdown` | `rigShutdownCmd` | `rig.go:194-218`  | `runRigShutdown` |
| `status`   | `rigStatusCmd`   | `rig.go:220-241`  | `runRigStatus` |
| `stop`     | `rigStopCmd`     | `rig.go:243-269`  | `runRigStop` |
| `restart`  | `rigRestartCmd`  | `rig.go:271-294`  | `runRigRestart` |
| `menu`     | `rigMenuCmd`     | `rig.go:858-864`  | `runRigMenu` (hidden — tmux keybinding) |

Defined in sibling files:

| subcommand | var | source file |
|---|---|---|
| `detect`    | `rigDetectCmd`    | `rig_detect.go:21`   |
| `quick-add` | `rigQuickAddCmd`  | `rig_quick_add.go:24` |
| `park`      | `rigParkCmd`      | `rig_park.go:21`     |
| `unpark`    | `rigUnparkCmd`    | `rig_park.go:44`     |
| `dock`      | `rigDockCmd`      | `rig_dock.go:25`     |
| `undock`    | `rigUndockCmd`    | `rig_dock.go:51`     |
| `config`    | `rigConfigCmd`    | `rig_config.go:18`   |
| `settings`  | `rigSettingsCmd`  | `rig_settings.go:20` |

All registered on `rigCmd` via `rigCmd.AddCommand(...)` from each
sibling's `init()`.

### Lifecycle semantics

From the Long help on each subcommand, the lifecycle verbs form
three distinct axes:

- **Patrol lifecycle** (`boot`/`start`/`reboot`/`shutdown`/
  `stop`/`restart`). `boot` and `shutdown` take **one** rig;
  `start`/`stop`/`restart` each take **one or more** — they are
  the multi-rig variants. `reboot` is single-rig. Patrols
  started are specifically Witness and Refinery; polecats are
  **not** started by these commands (see `rigBootCmd` Long help
  at `rig.go:148-150`) — polecats spawn on demand when work is
  assigned.
- **Operational state** (`park`/`unpark`/`dock`/`undock`). Parked
  and docked are two distinct non-operational states reflected
  in the rig LED on `gt rig list` and `gt rig menu`.
- **Data state** (`reset`). Resets various rig state: handoff
  content, mail, stale orphaned in-progress issues. `--dry-run`
  previews without mutating.

### Rig structure

From the `rigCmd` Long help (`rig.go:41-49`) and `rigAddCmd`
(`rig.go:54-80`), a rig container has:

- `refinery/rig/` — canonical main clone (Refinery's working copy).
- `mayor/rig/` — Mayor's working clone for this rig.
- `crew/<name>/` — full clones for human crew workers, managed by
  [crew.md](crew.md).
- `witness/` — Witness agent (no clone).
- `polecats/` — Worker directories, managed by
  [polecat.md](polecat.md).
- `.beads/` — Rig-level issue tracking.
- `config.json` — Rig configuration.
- `plugins/` — Rig-level plugin directory.

`gt rig add` also seeds patrol molecules for Deacon, Witness, and
Refinery (`rig.go:68`), and creates both a town-level
`~/gt/plugins/` (if missing) and a rig-level `<rig>/plugins/`
directory.

### `rig add` flags (`rig.go:361-370`)

| flag | type | source |
|---|---|---|
| `--prefix`         | string        | `rig.go:361` |
| `--local-repo`     | string        | `rig.go:362` |
| `--branch`         | string        | `rig.go:363` |
| `--push-url`       | string        | `rig.go:364` |
| `--upstream-url`   | string        | `rig.go:365` |
| `--adopt`          | bool          | `rig.go:366` |
| `--url`            | string        | `rig.go:367` |
| `--force`          | bool          | `rig.go:368` |
| `--filter`         | string        | `rig.go:369` |
| `--sparse-checkout` | []string     | `rig.go:370` |

Notable: `--adopt` registers an existing on-disk rig instead of
cloning, reading existing `config.json` if present and
auto-detecting the git URL from the `origin` remote (the
`git-url` positional arg becomes optional in that mode, per
`rig.go:72-75`). `--force` on `rig add --adopt` registers even
when git remote detection fails (`rig.go:368`). Note: `--force`
is shared between `rigAddAdoptForce` and other uses (the file has
multiple commands where this var name appears).

### `rig reset` flags (`rig.go:372-376`)

| flag | type | default | meaning |
|---|---|---|---|
| `--handoff` | bool | `false` | clear handoff content only |
| `--mail`    | bool | `false` | clear stale mail only |
| `--stale`   | bool | `false` | reset orphaned in_progress issues |
| `--dry-run` | bool | `false` | preview without mutating (pairs with `--stale`) |
| `--role`    | string | `""` | role to reset (default: auto-detect from cwd) |

With no flags, `gt rig reset` resets all resettable state.

### Shutdown / stop / restart / reboot safety

All four commands share a `--force` / `--nuclear` pair — defined
as four separate variables (`rig.go:313-320`) but with identical
meaning:

- **Default** — check every polecat for uncommitted work,
  stashes, and unpushed commits. Refuse to proceed if any
  polecat has pending work.
- **`-f, --force`** — prompt interactively if the stdin is a TTY
  (`confirmUnsafeProceed`, `rig.go:391-402`), otherwise block with
  a hint pointing at `--nuclear`.
- **`--nuclear`** — DANGER: bypass ALL safety checks (explicit
  warnings in all four Long helps — `rig.go:211, 215, 266, 290`).

Test seams live at `rig.go:325-340`: `listPolecatsForWorkCheck`,
`checkPolecatWorkStatus`, `isStdinTerminal`, and
`promptYesNoUnsafeProceed` can all be replaced in tests, which
is how the safety logic is unit-tested.

### `rig list` flags

| flag | type | default | source |
|---|---|---|---|
| `--json` | bool | `false` | `rig.go:357` |

`gt rig list --json` dumps the enumerated rigs as JSON
(`rig.go:818-822`). The non-JSON path prints a colored status
block per rig (`rig.go:824-853`): LED glyph (with special
spacing for the `🅿️` (parked) glyph), bold name, witness +
refinery state icons, polecat count, crew count.

### `rig status [rig]`

`rigStatusCmd` has `SuggestFor: []string{"health",
"health-check", "healthcheck"}` (`rig.go:222`) — a rare cobra
feature that makes typos of `health` suggest `status`. If no
rig name is given, the rig is inferred from the current
directory (see `rig.go:226, 237`). Displays rig info, witness
status, refinery status, polecats (name, state, assigned issue,
session status), and crew (name, branch, session, git status).

### Hidden `rig menu` — tmux popup

`runRigMenu` (`rig.go:866-`) uses `tmux display-menu` to show
a popup listing all rigs with LED glyphs and per-rig contextual
actions (Stop / Start / Status). Sorted by `rigStatePriority` —
running rigs first, then the rest alphabetically. Hidden from
the normal help output and intended to be wired to a tmux
keybinding.

## Related commands

- [crew.md](crew.md) — crew workspaces live under
  `<rig>/crew/`, and the parent `rigCmd` Long help lists them as
  part of the rig structure.
- [polecat.md](polecat.md) — polecats live under
  `<rig>/polecats/`; the same rig-level init creates this
  directory.
- [mayor.md](mayor.md) — `mayor/rig/` is the Mayor's working
  clone for the rig. `gt rig add` creates it alongside the
  Refinery clone.
- [boot.md](boot.md) — unrelated to `gt rig boot`; the two share
  a verb but Boot is a watchdog role.
- [worktree.md](worktree.md) — cross-rig worktrees use
  `getRig(targetRig)` from `rig_helpers.go` to validate the
  target.
- [namepool.md](namepool.md) — per-rig namepool configuration
  lives under `<rig>/settings/config.json`; `namepool.set` writes
  to the same settings file that `gt rig settings` manages.
- [formula.md](formula.md) — patrol molecules seeded by
  `gt rig add` (`rig.go:68`).
- [install.md](install.md) — pre-requisite. `gt install` creates
  the HQ; `gt rig add` fills it with rigs.
- [doctor.md](doctor.md) — health check.
- [agents.md](agents.md) — cross-rig enumeration of agents.

## Notes / open questions

- **`rig` is both a domain noun and a CLI command.** This page
  covers the CLI only. A `gastown/concepts/rig.md` is pending
  and should describe the rig as a structural primitive (town
  → rigs → agents + crew).
- **The `rig add` positional arg is `RangeArgs(1, 2)`**
  (`rig.go:81`): with `--adopt`, the second positional (git-url)
  is optional. Without `--adopt`, a single positional fails the
  downstream validation — worth confirming in `runRigAdd`.
- **`boot` / `start` / `reboot` / `shutdown` overlap.** `boot`
  does the same thing as `start` for a single rig, and
  `shutdown` does the same as `stop`. The one-rig vs multi-rig
  split is historical — `gt rig stop <rig>` is strictly more
  useful than `gt rig shutdown <rig>` because the former accepts
  multiple rigs. Operator muscle memory still uses both.
- **Four copies of `--force`/`--nuclear`** is a lot of
  duplication. A shared var would simplify the file but
  complicate test seams (which currently poke individual
  variables). Worth a decision note if the file grows further.
- **`rig status` as a health-check replacement.** `SuggestFor:
  health, health-check, healthcheck` means cobra nudges users
  away from the health idiom. If a dedicated health command
  ever lands, this suggestion list should be revisited. See
  [doctor.md](doctor.md) for the broader diagnostics story.
- **`confirmUnsafeProceed` requires a TTY.** Non-interactive
  environments (CI, automation) cannot use `--force` for any of
  the safety-gated commands — they must use `--nuclear` or
  nothing. Worth a workflow note on the automation story for
  rig lifecycle.
- **Hidden `rig menu` is tmux-only.** There is no non-tmux
  fallback; calling `gt rig menu` outside tmux just fails.
  Worth a decision note — the hidden annotation is intentional.
- **`reset --stale` semantics.** Only `--stale` pairs with
  `--dry-run`. The Long help at `rig.go:132-136` suggests the
  other resets (`--handoff`, `--mail`) are either always safe
  or have no preview mode. Worth verifying in `runRigReset`.
