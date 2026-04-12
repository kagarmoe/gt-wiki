---
title: gt agents
type: command
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/cmd/agents.go
tags: [command, agents, tmux, session, menu, collision-check]
---

# gt agents

List Gas Town agent sessions across all tmux sockets and provide an
interactive popup menu for switching between them. Also exposes
identity-collision diagnostics for worker locks.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupAgents` ("Agent Management") (`agents.go:78`)
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no
**Alias:** `gt ag`

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/agents.go` (781
lines). Registration: `agents.go:139-148` registers four subcommands
plus a parent fallback on `rootCmd`.

### Invocation

```
gt agents                 # list (same as `gt agents list`)
gt agents list            # plain list to stdout
gt agents menu            # tmux display-menu popup
gt agents check [--json]  # identity-collision report
gt agents fix             # clean stale locks; report collisions
```

### Parent behavior

`agentsCmd` (`agents.go:75-87`): the bare `gt agents` invocation runs
`runAgentsList` (`agents.go:540-580`) — i.e. the default action is the
same as `gt agents list`. The long help text points users at `gt
agents menu` for the interactive popup. There is no `RunE:
requireSubcommand` fallback here — the parent has its own `RunE`.

Persistent flag `--all` / `-a` on the parent (`agents.go:140`) is
inherited by all subcommands and toggles polecat visibility: by
default polecats are hidden from the list and menu (use `gt polecat
list` to see them).

### Subcommands

| subcommand | var | source | purpose |
|---|---|---|---|
| `list`  | `agentsListCmd`  | `agents.go:89-94`   | Plain stdout listing via `runAgentsList`. |
| `menu`  | `agentsMenuCmd`  | `agents.go:96-101`  | Interactive tmux popup via `runAgents`. |
| `check` | `agentsCheckCmd` | `agents.go:103-117` | Find identity collisions and stale locks. |
| `fix`   | `agentsFixCmd`   | `agents.go:119-132` | Clean stale locks; report collisions for manual intervention. |

### Agent type taxonomy

The `AgentType` enum (`agents.go:24-33`) defines 8 categories:
`AgentMayor`, `AgentDeacon`, `AgentWitness`, `AgentRefinery`,
`AgentCrew`, `AgentPolecat`, `AgentPersonal` (non-GT user sessions),
and `AgentTest` (sessions on `gt-test-*` sockets from integration
tests). Each type has a color code (`agents.go:45-54`) and icon
(`agents.go:66-73`) used by the tmux menu.

Sessions are categorized in `categorizeSession` (`agents.go:151-182`)
by calling `session.ParseSessionName` — a session that doesn't parse
as a known Gas Town role (Mayor / Deacon / Witness / Refinery / Crew /
Polecat) is dropped. `RoleOverseer` is explicitly excluded because
"overseer is the human operator, not a display agent"
(`agents.go:176`).

### Multi-socket listing

`getAllSocketSessions` (`agents.go:235-307`) enumerates:

1. The town tmux socket (`tmux.GetDefaultSocket()`) — with a fallback
   to `$GT_TOWN_SOCKET` when `workspace.FindFromCwd` fails during
   `persistentPreRun` (the menu can be invoked via tmux keybinding
   from a non-town directory — `agents.go:240-245`).
2. The `default` socket — rendered as the "Personal" group.
3. Any `gt-test-*` sockets with a live tmux server, via
   `findTestSockets` (`agents.go:203-229`) — rendered as a single
   "testing" group. Bindings are re-installed on each test socket so
   that `prefix+g` works after switching (`agents.go:292`).

### Sort order

`filterAndSortSessions` (`agents.go:310-360`): Mayor first, then
Deacon, then grouped by rig name alphabetically, then by
`rigTypeOrder` within a rig (refinery → witness → crew → polecat),
then by agent name. Boot sessions (name
`session.BootSessionName()`) are filtered out of the list.

### Interactive menu (`runAgents`, `agents.go:445-538`)

Builds a tmux `display-menu` command with centered group headers,
rig subheaders, and keyboard shortcuts (`shortcutKey`,
`agents.go:434-443`) mapping 1-9 then a-z. Each menu entry's action
is produced by `buildMenuAction` (`agents.go:421-431`):

- Same-socket: plain `switch-client -t <session>`.
- Cross-socket: `tmux -L <sock> switch-client` with a fallback to
  `detach-client -E "tmux -L <sock> attach -t <session>"`.

If no sessions are running, the menu is skipped and the caller sees a
prompt to `gt mayor start` / `gt deacon start` (`agents.go:454-459`).

### Collision check (`runAgentsCheck`, `agents.go:601-640`)

Requires a Gas Town workspace via `workspace.FindFromCwdOrError`.
Calls `buildCollisionReport` (`agents.go:684-752`), which:

1. Lists all tmux sessions and filters to known Gas Town sessions via
   `session.IsKnownSession`.
2. Walks all worker locks under the town root via
   `lock.FindAllLocks`.
3. For each lock:
   - If stale (dead PID per `lock.IsStale`), records a `stale`
     issue.
   - Otherwise reconstructs the expected session name via
     `guessSessionFromWorkerDir` (`agents.go:754-781`) from the
     worker directory layout `<rig>/<worker-type>/<worker-name>` and
     checks whether that session exists in tmux. If not, the lock is
     flagged as `orphaned` (held by a live PID with no matching
     session).

Output with no issues: `✓ All agents healthy` plus counts. With
issues: a header (`⚠️  Issues Detected`), per-issue details, and a
hint to run `gt agents fix`.

The `--json` flag (`agents.go:141`) produces a `CollisionReport`
object with `total_sessions`, `total_locks`, `collisions`,
`stale_locks`, `issues`, and `locks` fields
(`agents.go:583-599`).

### Fix action (`runAgentsFix`, `agents.go:642-682`)

Calls `lock.CleanStaleLocks` to remove stale entries, then rebuilds
the collision report. Live-PID collisions are *never* killed
automatically — the fix command prints them and tells the operator to
close duplicate sessions or remove lock files manually.

## Flags (summary)

| flag | type | default | scope | source |
|---|---|---|---|---|
| `-a, --all` | bool | `false` | parent (persistent) | `agents.go:140` |
| `--json` | bool | `false` | `check` only | `agents.go:141` |

## Related commands

- [../binaries/gt.md](../binaries/gt.md) — root, sets up
  `persistentPreRun` which wires the default socket that this
  command reads.
- `gt mayor` ([mayor.md](mayor.md)), `gt deacon`
  ([deacon.md](deacon.md)) — the two town-level agents surfaced
  first in the listing.
- `gt boot` ([boot.md](boot.md)) — boot sessions are explicitly
  *excluded* from the listing (`agents.go:321`) because Boot is
  considered a utility session rather than a user-facing agent.
- `gt dog` ([dog.md](dog.md)) — dogs are not a recognized `AgentType`
  in `agents.go`, so they do *not* appear in `gt agents list`; dogs
  are managed separately under `gt dog list`.
- `gt polecat list` — referenced in the parent long help
  (`agents.go:83`); polecats are hidden unless `--all` is passed.
- `gt whoami` ([whoami.md](whoami.md)) — identity is consumed here
  via `session.ParseSessionName`.

## Notes / open questions

- **Personal sessions** (`AgentPersonal`) are listed from the
  `default` tmux socket but only when it differs from the town
  socket. Machines where the town socket *is* `default` will not see
  a separate Personal group. Confirm whether that is intentional.
- **Dogs are absent.** `AgentType` has no `AgentDog`, and
  `session.RoleDeacon` is the only dog-adjacent role that surfaces.
  Dogs run in their own worktrees under `~/gt/deacon/dogs/` and are
  displayed only via [dog.md](dog.md).
- **`--all` is persistent, `--json` is not.** Running `gt agents
  --all check --json` should work because `--all` is on the parent
  and `--json` is scoped to `check`, but `gt agents --json` on the
  bare parent will fail with an unknown-flag error.
- **`buildCollisionReport` ignores non-Gas-Town locks.** The "orphan"
  detection relies on `guessSessionFromWorkerDir` being able to
  reconstruct a session name from the worker dir's path segments
  (rig / type / name). Locks under unrecognized layouts simply
  return an empty string and are not flagged as orphaned.
- **No `gt agents kill` action.** The fix command stops at stale
  locks; killing live duplicate sessions is explicitly left to the
  operator. Any future command that tears down a collision would
  need to decide which PID is authoritative.
