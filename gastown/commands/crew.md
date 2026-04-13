---
title: gt crew
type: command
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/cmd/crew.go
  - /home/kimberly/repos/gastown/internal/cmd/crew_add.go
  - /home/kimberly/repos/gastown/internal/cmd/crew_at.go
  - /home/kimberly/repos/gastown/internal/cmd/crew_cycle.go
  - /home/kimberly/repos/gastown/internal/cmd/crew_helpers.go
  - /home/kimberly/repos/gastown/internal/cmd/crew_lifecycle.go
  - /home/kimberly/repos/gastown/internal/cmd/crew_list.go
  - /home/kimberly/repos/gastown/internal/cmd/crew_maintenance.go
  - /home/kimberly/repos/gastown/internal/cmd/crew_status.go
tags: [command, workspace, crew, domain-noun, session, tmux, beads-exempt]
---

# gt crew

Manage **crew workers** — persistent, user-managed workspaces for
human developers (or long-lived agent sessions) on top of a rig.
Unlike polecats, crew workspaces are full git clones and do not
auto-clean.

This page documents the `gt crew` CLI subcommands. The "crew worker"
concept as a persona — how crew members integrate with mail,
nudges, beads, and cross-rig identity — will get a dedicated
role/concept page in a later batch (pending Batch 6).

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupWorkspace` ("Workspace") (`crew.go:31`)
**Polecat-safe:** no
**Beads-exempt:** yes (in `beadsExemptCommands` on
`root.go:44-77`; listing and attaching to crew must work even if
beads is broken)
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/crew.go` (432
lines) registers the command tree. The actual run functions are
spread across ten sibling files:

| file | contains |
|---|---|
| `crew.go` | command/subcommand definitions, flags, registration |
| `crew_add.go` | `runCrewAdd` |
| `crew_at.go` | `runCrewAt` (start/attach session) |
| `crew_cycle.go` | `runCrewNext`, `runCrewPrev`, `crewCycleSession` flag |
| `crew_helpers.go` | helpers such as `detectCrewFromCwd` used by [worktree.md](worktree.md) |
| `crew_lifecycle.go` | `runCrewStart`, `runCrewStop`, `runCrewRestart`, `runCrewRefresh` |
| `crew_list.go` | `runCrewList` |
| `crew_maintenance.go` | `runCrewPristine`, `runCrewRename`, `runCrewRemove` |
| `crew_status.go` | `runCrewStatus` |
| `*_test.go` | tests |

The parent `crewCmd` (`crew.go:29-59`) uses `RunE:
requireSubcommand` as its fallback, so `gt crew` with no arguments
prints the subcommand list. The Long help (`crew.go:34-58`)
contrasts crew with polecats:

> CREW VS POLECATS:
> Polecats: Ephemeral sessions. Witness-managed. Auto-nuked after work.
> Crew: Persistent. User-managed. Stays until you remove it.

### Invocation

```
gt crew add     <name...>      [--rig <name>] [--branch]
gt crew list    [rig]          [--rig <name>] [--all] [--json]
gt crew at      [name]         [--rig <name>] [--no-tmux] [-d|--detached]
                                [--account <h>] [--agent <alias>] [--debug] [--reset]
gt crew remove  <name...>      [--rig <name>] [--force] [--purge]
gt crew refresh <name>         [--rig <name>] [-m|--message <msg>]
gt crew status  [<name>]       [--rig <name>] [--json]
gt crew rename  <old> <new>    [--rig <name>]
gt crew pristine [<name>]      [--rig <name>] [--json]
gt crew restart [name...]      [--rig <name>] [--all] [--dry-run]
gt crew start   [rig] [name...] [--rig <name>] [--all] [--account <h>]
                                 [--agent <alias>] [--resume[=<id>]]
gt crew stop    [name...]      [--rig <name>] [--all] [--dry-run] [--force]
gt crew next    (hidden)       [-s|--session <tmux>]
gt crew prev    (hidden)       [-s|--session <tmux>]
```

`gt crew at` has alias `attach` (`crew.go:100`). `gt crew restart`
has alias `rs` (`crew.go:202`). `gt crew start` has alias `spawn`
(`crew.go:293`).

### Subcommands

| subcommand | var | source | run-fn |
|---|---|---|---|
| `add`      | `crewAddCmd`      | `crew.go:61-79`    | `runCrewAdd`      (`crew_add.go`)         |
| `list`     | `crewListCmd`     | `crew.go:81-96`    | `runCrewList`     (`crew_list.go`)        |
| `at`       | `crewAtCmd`       | `crew.go:98-137`   | `runCrewAt`       (`crew_at.go`)          |
| `remove`   | `crewRemoveCmd`   | `crew.go:139-165`  | `runCrewRemove`   (`crew_maintenance.go`) |
| `refresh`  | `crewRefreshCmd`  | `crew.go:167-180`  | `runCrewRefresh`  (`crew_lifecycle.go`)   |
| `status`   | `crewStatusCmd`   | `crew.go:182-198`  | `runCrewStatus`   (`crew_status.go`)      |
| `rename`   | `crewRenameCmd`   | `crew.go:239-252`  | `runCrewRename`   (`crew_maintenance.go`) |
| `pristine` | `crewPristineCmd` | `crew.go:254-267`  | `runCrewPristine` (`crew_maintenance.go`) |
| `restart`  | `crewRestartCmd`  | `crew.go:200-237`  | `runCrewRestart`  (`crew_lifecycle.go`)   |
| `start`    | `crewStartCmd`    | `crew.go:291-325`  | `runCrewStart`    (`crew_lifecycle.go`)   |
| `stop`     | `crewStopCmd`     | `crew.go:327-361`  | `runCrewStop`     (`crew_lifecycle.go`)   |
| `next`     | `crewNextCmd`     | `crew.go:269-278`  | `runCrewNext`     (`crew_cycle.go`) — hidden, tmux keybinding |
| `prev`     | `crewPrevCmd`     | `crew.go:280-289`  | `runCrewPrev`     (`crew_cycle.go`) — hidden, tmux keybinding |

### Workspace shape

From the Long help on `crewAddCmd` (`crew.go:64-76`), a crew
workspace is laid out under `<rig>/crew/<name>/` with:

- A **full git clone** of the project repository (not a worktree,
  unlike polecats).
- A mail directory for message delivery.
- A `CLAUDE.md` with crew-specific prompting.
- An optional feature branch `crew/<name>` when `--branch` is
  passed.

The `gt-<rig>-crew-<name>` tmux session naming convention is
mentioned in `crewRenameCmd` Long help (`crew.go:246`): the new
session inherits the new name after rename.

### Notable behavioral quirks

- **`gt crew at <name>` auto-discovers the rig** (`crew.go:118-127`).
  If called with no arg, tries to detect from the current
  directory. If called with just a name, searches all rigs for a
  unique match. If two rigs both have a crew named `joe`, the
  command fails — you need `gt crew at <rig>/joe` to disambiguate.
- **`gt crew at --reset`** checks out the rig's default branch
  and `git pull`s before attaching (`crew.go:113-116, 378`). By
  default, the workspace stays on whatever branch it was on.
- **`gt crew remove --purge`** (`crew.go:145-155, 381-382`) goes
  beyond the default close-and-keep-bead behavior. It:
  - Deletes the agent bead entirely (not just closes it).
  - Unassigns any beads currently assigned to the crew member.
  - Clears the crew member's inbox.
  - Handles git worktrees, not just full clones.
- **`gt crew refresh`** sends a handoff mail *to the crew member's
  own inbox* then restarts the session, so the new session picks
  up the handoff automatically (`crew.go:169-180`). `restart`
  (or `rs`) does the same kill-and-restart without the handoff
  mail (`crew.go:203-208`).
- **`gt crew start --resume`** (`crew.go:301-305, 403-404`) uses
  cobra's `NoOptDefVal = "last"` trick — `--resume` with no value
  resumes the most recent session; `--resume <id>` resumes a
  specific one. The startup command passes the agent's own
  resume flag (e.g. Claude's `--resume`) so the session picks up
  where it left off with proper Gas Town metadata so GC doesn't
  kill it.
- **`gt crew stop --force`** skips output capture for faster
  shutdown (`crew.go:338-340, 409`). The default stop captures
  pane output for debugging purposes.
- **`crewAll` is shared between `start`, `stop`, and `restart`**
  (`crew.go:21, 396, 400, 407`). The `Args` validators on those
  three commands (`crew.go:224-234, 314-322, 348-359`) all check
  `crewAll` together with the positional count, using subtly
  different rules — e.g. `start` allows `--all` with *no* other
  checks; `stop` disallows naming crew together with `--all`.

### Flags (grouped by command)

**Global-ish:** `--rig` (used by most subcommands; string;
stored in the shared `crewRig` variable).

| subcommand | flag | type | var | default | source |
|---|---|---|---|---|---|
| `add`      | `--branch`   | bool   | `crewBranch`        | `false` | `crew.go:366` |
| `list`     | `--all`      | bool   | `crewListAll`       | `false` | `crew.go:369` |
| `list`     | `--json`     | bool   | `crewJSON`          | `false` | `crew.go:370` |
| `at`       | `--no-tmux`  | bool   | `crewNoTmux`        | `false` | `crew.go:373` |
| `at`       | `-d, --detached` | bool | `crewDetached`    | `false` | `crew.go:374` |
| `at`       | `--account`  | string | `crewAccount`       | `""`    | `crew.go:375` |
| `at`       | `--agent`    | string | `crewAgentOverride` | `""`    | `crew.go:376` |
| `at`       | `--debug`    | bool   | `crewDebug`         | `false` | `crew.go:377` |
| `at`       | `--reset`    | bool   | `crewReset`         | `false` | `crew.go:378` |
| `remove`   | `--force`    | bool   | `crewForce`         | `false` | `crew.go:381` |
| `remove`   | `--purge`    | bool   | `crewPurge`         | `false` | `crew.go:382` |
| `refresh`  | `-m, --message` | string | `crewMessage`    | `""`    | `crew.go:385` |
| `status`   | `--json`     | bool   | `crewJSON`          | `false` | `crew.go:388` |
| `pristine` | `--json`     | bool   | `crewJSON`          | `false` | `crew.go:393` |
| `restart`  | `--all`      | bool   | `crewAll`           | `false` | `crew.go:396` |
| `restart`  | `--dry-run`  | bool   | `crewDryRun`        | `false` | `crew.go:397` |
| `start`    | `--all`      | bool   | `crewAll`           | `false` | `crew.go:400` |
| `start`    | `--account`  | string | `crewAccount`       | `""`    | `crew.go:401` |
| `start`    | `--agent`    | string | `crewAgentOverride` | `""`    | `crew.go:402` |
| `start`    | `--resume[=<id>]` | string | `crewResume`    | `""` (default `"last"` when bare) | `crew.go:403-404` |
| `stop`     | `--all`      | bool   | `crewAll`           | `false` | `crew.go:407` |
| `stop`     | `--dry-run`  | bool   | `crewDryRun`        | `false` | `crew.go:408` |
| `stop`     | `--force`    | bool   | `crewForce`         | `false` | `crew.go:409` |
| `next/prev` | `-s, --session` | string | `crewCycleSession` | `""` | `crew.go:424-425` |

## Related commands

- [polecat.md](polecat.md) — the ephemeral counterpart. The Long
  help at the top of `crew.go:36-38` is the canonical
  crew-vs-polecat contrast: polecats are witness-managed, crew is
  user-managed.
- [rig.md](rig.md) — rigs host crew workspaces under
  `<rig>/crew/`. The `--rig` flag is how most crew subcommands
  disambiguate which rig.
- [mayor.md](mayor.md) — a fresh crew workspace's `CLAUDE.md`
  wires crew into the same mail / nudge addressing the Mayor uses.
- [sling.md](sling.md) — the polecat analogue to
  [sling.md](sling.md) for crew is `gt crew add` (create) +
  `gt crew start` (start session). There is no direct analogue to
  `gt sling <issue>`.
- [agents.md](agents.md) — `gt agents list` includes crew
  members alongside polecats and patrols.
- [role.md](role.md) — `gt role` is for role contexts in general;
  crew-specific role content lives in the crew workspace's
  `CLAUDE.md`.
- [worktree.md](worktree.md) — `gt worktree` depends on
  `detectCrewFromCwd` from `crew_helpers.go` and is the only
  supported way for a crew member to hold a working copy of a
  different rig.
- [handoff.md](handoff.md) / [mail.md](mail.md) — handoff mail
  is what `gt crew refresh` sends internally.

## Notes / open questions

- **`crew` is both a domain noun and a CLI command.** This page
  covers the CLI command only. A `gastown/roles/crew.md` or
  `gastown/concepts/crew-vs-polecat.md` page is pending and is
  where the "what is a crew worker" conceptual content belongs.
- **Full clones, not worktrees.** The Long help on `crewAddCmd`
  (`crew.go:65`) is explicit: `"A full git clone of the project
  repository"`. This is the big structural difference from
  polecats and explains why `gt crew` needs its own rename,
  pristine, and remove paths — worktrees share the `.git`
  directory with the main clone; full clones do not.
- **`crewListAll` vs `crewAll`.** Both are bool flags, same
  underlying type, but they're distinct variables
  (`crew.go:21, 22`). `crewListAll` is only `list`'s flag;
  `crewAll` is shared across `start`, `stop`, `restart`. Mixing
  them up in a future refactor would silently break the
  validators.
- **`gt crew start` and `gt crew stop` share ambiguous arg
  grammars.** The `Args` functions (`crew.go:314-322, 348-359`)
  both accept "0 args", "1 arg" (which might be a rig name OR a
  crew name), or "N args" (all crew names). The ambiguity is
  resolved downstream by probing filesystem state, not by
  cobra. Users who alias a rig name to a crew name will see
  surprising behavior.
- **Hidden `next`/`prev` subcommands are tmux-only.** They are
  not documented in the visible help and exist specifically for
  tmux run-shell keybinding cycling. `--session` is the
  escape hatch so the cycling logic knows which session is
  "current" when run from a keybinding context where tmux's
  session detection is unreliable (`crew.go:422-423`).
- **Beads-exempt but still beads-integrated.** `gt crew remove
  --purge` directly manipulates agent beads, yet `crew` is
  beads-exempt. This means the exemption covers the command's
  ability to *run* when beads is down, but individual subcommand
  paths may still error if they need beads. Worth verifying
  what each subcommand actually does when the Dolt server is
  unreachable.
