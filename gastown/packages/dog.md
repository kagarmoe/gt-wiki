---
title: internal/dog
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-15
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
sources:
  - /home/kimberly/repos/gastown/internal/dog/manager.go
  - /home/kimberly/repos/gastown/internal/dog/types.go
  - /home/kimberly/repos/gastown/internal/dog/health.go
  - /home/kimberly/repos/gastown/internal/dog/session_manager.go
tags: [package, agent-runtime, dog, kennel, cross-rig, worktree, deacon-helper, idle-working]
---

# internal/dog

Dog (the Deacon's helper worker) lifecycle: kennel-level
add/remove/list/refresh, per-rig worktree creation into every
configured rig, persistent `idle`/`working` state with per-dog
flock locking, dog tmux session management with mail-driven work
discovery, and a health checker that classifies dog sessions as
healthy / dead / agent-dead / hung / orphan and optionally
auto-clears zombies.

**Go package path:** `github.com/steveyegge/gastown/internal/dog`
**File count:** 4 non-test go files, 5 test files.
**Role:** [`dog` role](../roles/dog.md) — domain persona
**CLI command:** [`gt dog`](../commands/dog.md)
**Imports (notable):** `github.com/gofrs/flock`,
[`internal/cli`](go-packages.md), [`internal/config`](config.md),
`internal/constants`, [`internal/git`](go-packages.md),
[`internal/rig`](go-packages.md),
[`internal/session`](session.md), `internal/style`,
`internal/tmux`, [`internal/util`](util.md), stdlib.
**Imported by (notable):** [`gt dog`](../commands/dog.md)
(`/home/kimberly/repos/gastown/internal/cmd/dog.go`),
[`internal/deacon`](deacon.md) feed-stranded indirectly (via shell
out to `gt sling deacon/dogs`), and the dog dispatch path inside
the Deacon agent.

## What it actually does

Four files. Two domains: kennel management (`manager.go`,
`types.go`) and tmux session management (`session_manager.go`,
`health.go`).

### types.go — state shapes

Source: `/home/kimberly/repos/gastown/internal/dog/types.go`.

- `State` string type with constants `StateIdle = "idle"` and
  `StateWorking = "working"` (`types.go:11-18`). Only two
  states. Comment says "Unlike polecats (single-rig, ephemeral
  sessions), dogs handle cross-rig infrastructure work."
- `Dog` struct (`types.go:20-30`) — `Name`, `State`, `Path`,
  `Worktrees map[string]string` (rig name → worktree path),
  `LastActive`, `Work`, `WorkStartedAt`, `CreatedAt`.
- `DogState` struct (`types.go:33-42`) — the persistent shape
  written to `.dog.json`. JSON-tagged with omitempty fields.

### manager.go — kennel and worktree management

Source: `/home/kimberly/repos/gastown/internal/dog/manager.go`.

- `Manager` struct (`manager.go:31-35`) — `townRoot`,
  `kennelPath` (`<townRoot>/deacon/dogs/`), `rigsConfig`.
  Constructed via `NewManager(townRoot, rigsConfig)`
  (`manager.go:38-44`).
- Sentinel errors `ErrDogExists`, `ErrDogNotFound`,
  `ErrDogWorking`, `ErrNoRigs`, `ErrInvalidName`
  (`manager.go:22-28`).
- `lockDog(name)` (`manager.go:49-60`) — flock at
  `<dog-dir>/.dog.lock`. Prevents load-modify-save races on
  `.dog.json`.
- `validateDogName(name)` (`manager.go:63-74`) — rejects empty,
  path separators, `.`/`..`/traversal.
- `Manager.Add(name) (*Dog, error)` (`manager.go:95-163`) — the
  cross-rig fan-out. Creates `<kennel>/<name>/` and then a
  worktree for EVERY rig in `rigsConfig.Rigs` via
  `createRigWorktree`. On any failure, the deferred cleanup
  removes the entire dog directory. Writes the initial
  `DogState` with `StateIdle`. The "every rig" worktree fan-out
  is what makes a dog cross-rig: a single dog has working trees
  into each project simultaneously.
- `createRigWorktree(dogPath, dogName, rigName) (string, error)`
  (`manager.go:168-195`) — looks up the rig's repo base via
  `findRepoBase`, picks `origin/<default-branch>` as the start
  point (loaded from rig config if available), constructs a
  unique branch name `dog/<dog>-<rig>-<unix-millis>`, and
  creates the worktree at `<dogPath>/<rigName>/`.
- `findRepoBase(rigPath)` (`manager.go:199-212`) — prefers
  `<rig>/.repo.git` (bare repo), falls back to `<rig>/mayor/rig`.
  Returns the appropriate `*git.Git` handle.
- `Manager.Remove(name) error` (`manager.go:216-259`) — loads
  state, removes each per-rig worktree via
  `repoGit.WorktreeRemove`, prunes stale entries with
  `WorktreePrune`, removes the dog directory.
- `Manager.List() ([]*Dog, error)` (`manager.go:262-285`) — scans
  the kennel, calls `Get` for each entry. Skips invalid dogs
  (e.g. `boot/`, which uses a `.boot-status.json` instead of a
  `.dog.json` — comment at `manager.go:299-301`).
- `Manager.Get(name) (*Dog, error)` (`manager.go:289-314`) —
  returns `ErrDogNotFound` if either the directory or the
  `.dog.json` state file is missing. The latter is how `Get`
  knows that "boot is not a real dog."
- State mutators: `SetState(name, state)` (`manager.go:317-342`),
  `AssignWork(name, work)` (`manager.go:345-372`),
  `ClearWork(name)` (`manager.go:375-402`). All three lock,
  load, mutate, save, with `LastActive` and `UpdatedAt` updates.
- `Manager.Refresh(name) error` (`manager.go:408-482`) —
  recreates ALL worktrees for a dog with fresh branches.
  Refuses if the dog is in `StateWorking`. Atomic per-rig: each
  rig is removed, fetched, recreated, and the state is saved
  before moving to the next rig, so a failure at rig N leaves
  rigs 1..N-1 correctly updated.
- `Manager.RefreshRig(name, rigName) error`
  (`manager.go:485-551`) — same but for a single rig.
- `Manager.CleanupStaleBranches() (int, error)`
  (`manager.go:555-574`) — iterates all rigs and per-rig calls
  `cleanupStaleBranchesForRig` (`manager.go:577-622`), which
  lists `dog/*` branches in the rig's repo, builds a set of
  in-use branches by walking each dog's current worktree, and
  deletes orphans.
- `loadState` / `saveState` (`manager.go:625-643`) — JSON via
  `util.AtomicWriteJSON` for the save side.
- Pool predicates: `GetIdleDog()` (`manager.go:646-660`),
  `IdleCount()` (`manager.go:663-676`), `WorkingCount()`
  (`manager.go:679-692`).

### session_manager.go — dog tmux session lifecycle

Source: `/home/kimberly/repos/gastown/internal/dog/session_manager.go`.

- `SessionManager` struct (`session_manager.go:25-29`) — bound to
  a `*tmux.Tmux`, a `*Manager`, and a `townRoot`. The Manager
  reference is so session start/stop can flip persistent state
  to working/idle.
- Sentinel errors `ErrSessionRunning`, `ErrSessionNotFound`
  (`session_manager.go:19-22`).
- `SessionStartOptions` (`session_manager.go:43-49`) —
  `WorkDesc`, `AgentOverride`.
- `SessionInfo` (`session_manager.go:52-67`).
- `SessionName(dogName) string` (`session_manager.go:74-76`) —
  returns `hq-dog-<name>`. The comment is explicit: "We use
  'hq-dog-' instead of 'hq-deacon-' to avoid tmux prefix-matching
  collisions with the 'hq-deacon' session." Dogs are town-level
  but cannot share the deacon's session prefix.
- `SessionManager.Start(dogName, opts) error`
  (`session_manager.go:85-151`) — verifies the kennel directory
  exists, kills any zombie session, builds an instructions
  string (with a special branch for `plugin:` work that tells
  the dog to read its mail and not search for the plugin
  locally — prevents dogs from escalating "plugin not found"
  for town-level plugins), and calls
  `session.StartSession` with a `BeaconConfig` of
  `Recipient: BeaconRecipient("dog", dogName, ""), Sender:
  "deacon", Topic: "assigned"`. After successful session creation
  it flips the manager state to `StateWorking` (best-effort —
  the session is running either way).
- `SessionManager.Stop(dogName, force bool) error`
  (`session_manager.go:154-183`) — graceful `C-c` + wait unless
  force, then `KillSessionWithProcesses`, then flips state to
  `StateIdle`.
- `IsRunning`, `Status`, `GetPane`, `EnsureRunning`
  (`session_manager.go:186-256`).

### health.go — zombie / hung / orphan detection

Source: `/home/kimberly/repos/gastown/internal/dog/health.go`.

- `sessionChecker` interface (`health.go:12-16`) —
  `CheckSessionHealth`, `HasSession`, `KillSession`. Lets tests
  mock the tmux side.
- `DogHealthResult` struct (`health.go:19-27`) — `Name`, `State`,
  `SessionStatus`, `WorkDuration`, `NeedsAttention`,
  `AutoCleared`, `Recommendation`.
- `HealthChecker` (`health.go:30-33`) bound to a `*Manager` and
  a `sessionChecker`.
- `dogSessionName(name)` (`health.go:41-43`) — computes
  `"hq-dog-<name>"` (mirrors `SessionManager.SessionName`).
- `HealthChecker.Check(d *Dog, maxInactivity time.Duration,
  autoClear bool) DogHealthResult` (`health.go:46-127`) — the
  state-machine. For working dogs, calls `CheckSessionHealth`
  and switches on the `tmux.ZombieStatus` enum:
  - `SessionDead` → "zombie: session dead but state=working"; if
    `autoClear`, calls `mgr.ClearWork`.
  - `AgentDead` → "zombie: agent dead in session"; if
    `autoClear`, kills the session and `ClearWork`.
  - `AgentHung` → "hung: agent alive but no tmux activity"; if
    `autoClear`, kills and `ClearWork` ("the dog almost certainly
    finished its work but failed to call `gt dog done`").
  - `SessionHealthy` → no action.
  For idle dogs, checks for orphan tmux sessions (idle-but-
  session-exists) and either kills (auto-clear) or reports.
- `CheckAll(maxInactivity, autoClear) ([]DogHealthResult, error)`
  (`health.go:130-141`) — iterates the kennel.
- `NeedsAttentionCount(results)` (`health.go:144-152`).

### Notable design choices

- **Two states, no more.** Dogs are either idle or working.
  There's no "stalled" / "stuck" / "zombie" persistent state —
  zombie is a detected condition reconciled by the health
  checker, not stored.
- **One worktree per rig per dog.** A dog with worktrees into 4
  rigs has 4 working trees. This is the structural feature that
  makes dogs cross-rig. The branch naming
  `dog/<dog>-<rig>-<millis>` ensures uniqueness across dogs and
  refreshes.
- **`hq-dog-` not `hq-deacon-`.** Dogs are town-level (live
  under the deacon's kennel) but their session prefix
  deliberately avoids `hq-deacon-` to prevent tmux prefix-match
  collisions with the actual `hq-deacon` session.
- **Plugin dispatch branch in startup instructions.** A dog
  dispatched with `WorkDesc: "plugin:foo"` gets explicitly told
  to read its mail for instructions and NOT to look for the
  plugin locally — because plugins can be town-level and a dog
  searching its worktree's `plugins/` directory would escalate
  "plugin not found" incorrectly.
- **State sync is best-effort.** Both `Start` and `Stop` log a
  warning if `SetState` fails but do not roll back the session
  operation. The session is the source of truth; the persistent
  state is a label.
- **Hung detection without auto-cleanup.** `health.go`'s
  `AgentHung` branch will report `NeedsAttention` but only auto-
  clear if `autoClear` is set — and even then, the comment is
  explicit that the call is best-effort. Dogs being "hung" is a
  judgement call deferred to the Deacon agent rather than
  resolved in Go (per ZFC principle).

## Related wiki pages

- [`dog` role](../roles/dog.md) — domain persona.
- [`gt dog`](../commands/dog.md) — CLI surface (9 subcommands).
- [`deacon` package](deacon.md) — the dog's manager. Deacon's
  feed-stranded path dispatches dogs via `gt sling deacon/dogs`
  with the `mol-convoy-feed` molecule.
- [`gt deacon dispatch`](../commands/deacon.md) and
  [`gt dog dispatch`](../commands/dog.md) — CLI dispatch entry
  points.
- [`internal/session`](session.md) — `StartSession`,
  `BeaconRecipient`, `KillExistingSession`.
- [`internal/config`](config.md) — `RigsConfig`, agent env
  resolution.
- [`internal/rig`](go-packages.md) — `LoadRigConfig`.
- [`internal/git`](go-packages.md) — `WorktreeAddFromRef`,
  `WorktreeRemove`, `WorktreePrune`, `ListBranches`,
  `DeleteBranch`.
- [`mayor` package](mayor.md) — sibling town-level package.
- [`polecat` package](polecat.md) — the cats vs dogs counterpart.
  "Cats build features. Dogs clean up messes."

## Notes / open questions

- "boot is a special dog" — the comment at
  `manager.go:299-301` says the boot watchdog uses
  `.boot-status.json` instead of `.dog.json`, and `Get` returns
  `ErrDogNotFound` for boot. So even though boot lives at
  `<kennel>/boot/`, the dog manager refuses to treat it as a
  dog. There is no `AgentDog` registration in the agents config
  either — dogs are invisible to `gt agents list`.
- Refresh's atomicity is per-rig but not per-dog: a partial
  refresh leaves earlier rigs updated and later rigs untouched
  (with `state.Worktrees[rigName]` reflecting the last
  successful save). Callers must inspect the returned error and
  the saved state to know what happened.
- `cleanupStaleBranchesForRig` walks every dog's worktree on
  every rig, which is O(dogs × rigs) git invocations. For a
  large town this could be slow — there's no caching.
- The `health.go` `AgentHung` branch is the only place in this
  package that touches the "should we clear a probably-finished
  dog" question. The default for the auto-clear flag at the CLI
  level is false, so by default the Deacon agent decides.
- The health checker's `sessionChecker` interface is one of the
  few mockable boundaries in the agent-runtime layer. The dog
  manager itself is not mockable behind an interface — tests
  exercise it against a real (or temp-dir) filesystem.
