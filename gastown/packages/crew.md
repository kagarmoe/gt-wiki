---
title: internal/crew
type: package
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/crew/manager.go
  - /home/kimberly/repos/gastown/internal/crew/types.go
tags: [package, agent-runtime, crew, persistent, full-clone, user-managed, locking]
---

# internal/crew

Crew worker management: per-crew lifecycle for full git clones (not
worktrees) under `<rig>/crew/<name>/`, with per-crew flock-based
locking, automatic remote sync from the rig's `mayor/rig`,
overlay file copies, setup-hook execution, shared-beads redirect
setup, and tmux session start/stop with optional Claude session
resume. The Crew package is the structural counterpart of the
[polecat package](polecat.md): both manage rig-scoped workers, but
crew workspaces are full clones and persist until the user removes
them, where polecat sandboxes are worktrees that auto-clean.

**Go package path:** `github.com/steveyegge/gastown/internal/crew`
**File count:** 2 non-test go files (one large `manager.go`, one
small `types.go`), 2 test files.
**Role:** [`crew` role](../roles/crew.md) — domain persona
**CLI command:** [`gt crew`](../commands/crew.md)
**Imports (notable):** `github.com/gofrs/flock`,
[`internal/beads`](beads.md), [`internal/config`](config.md),
`internal/constants`, [`internal/git`](go-packages.md),
[`internal/nudge`](nudge.md), [`internal/rig`](go-packages.md),
`internal/runtime`, [`internal/session`](session.md),
`internal/style`, `internal/tmux`, [`internal/util`](util.md).
**Imported by (notable):** the entire `gt crew` CLI surface in
`/home/kimberly/repos/gastown/internal/cmd/crew*.go` (10 sibling
files), [`gt worktree`](../commands/worktree.md) (via
`detectCrewFromCwd`), and various rig-management commands.

## What it actually does

Two files. `types.go` is 40 lines: just `CrewWorker` and `Summary`
shapes. `manager.go` is the entire surface — about 940 lines of
manager methods, validation, locking, and start/stop logic.

### Public API — types

- `CrewWorker` struct (`types.go:7-25`) — `Name`, `Rig`,
  `ClonePath`, `Branch`, `CreatedAt`, `UpdatedAt`. JSON-tagged
  for state file persistence.
- `Summary` struct (`types.go:27-31`) — concise view returned by
  `CrewWorker.Summary()`.
- `Manager` struct (`manager.go:128-132`) — bound to a `*rig.Rig`
  and a `*git.Git` for the rig's local repo. Constructed via
  `NewManager(r, g)` (`manager.go:135-140`).
- `StartOptions` struct (`manager.go:37-64`) — `Account`,
  `ClaudeConfigDir`, `KillExisting`, `Topic`, `Interactive`,
  `AgentOverride`, `ResumeSessionID` (`"last"` or a specific
  ID).
- `PristineResult` struct (`manager.go:658-665`) — JSON shape
  returned by `Pristine`.

### Public API — sentinel errors

`ErrCrewExists`, `ErrCrewNotFound`, `ErrHasChanges`,
`ErrInvalidCrewName`, `ErrSessionRunning`, `ErrSessionNotFound`
(`manager.go:28-35`).

### Public API — name and ID validation

- `validateCrewName(name) error` (`manager.go:106-126`) — rejects
  empty / `.` / `..`, path separators, traversal sequences, and
  characters that break agent ID parsing (hyphens, dots, spaces).
  Suggests an underscore-substituted alternative on hyphen/dot/
  space rejection. Used by every public mutation method.
- `validateSessionID(id) error` (`manager.go:70-80`) — restricts
  resume session IDs to alphanumeric + `-_.`. Empty and `"last"`
  are special-cased and accepted. Prevents shell injection when
  the ID is interpolated into the resume command string.
- `buildResumeArgs(agentName, sessionID) (string, error)`
  (`manager.go:86-102`) — looks up the agent preset, refuses
  agents without `ResumeFlag`, refuses subcommand-style agents
  (no `--resume` shape support yet), uses `ContinueFlag` for
  `"last"` and `ResumeFlag <id>` otherwise.

### Public API — per-crew locking

`Manager.lockCrew(name) (*flock.Flock, error)`
(`manager.go:167-178`). Acquires an exclusive flock at
`<rig>/.runtime/locks/crew-<name>.lock`. The doc comment is
explicit: prevents concurrent `gt` processes from racing on the
same crew worker's filesystem operations (Add, Remove, Rename,
Start). Callers MUST defer `fl.Unlock()`. `Rename` locks both
names in alphabetical order to prevent deadlock
(`manager.go:570-583`).

### Public API — workspace lifecycle

- `Manager.Add(name, createBranch bool) (*CrewWorker, error)`
  (`manager.go:181-191`) — locks then calls `addLocked`. Refuses
  if the crew already exists.
- `Manager.addLocked` (`manager.go:194-343`) — the heavy lift.
  Rejects rigs without `git_url` (crews need clonable repos).
  Tries `CloneBranchWithReference` → `CloneBranch` → `CloneWith
  Reference` → `Clone` in fallback order, with style warnings
  between attempts. Calls `syncRemotesFromRig` to make the crew
  clone match the rig's `mayor/rig` remote configuration
  (origin, push URLs, upstream); a sync failure with no push URL
  is non-fatal but with a push URL configured it removes the
  partial clone and returns an error. Optionally creates a
  `crew/<name>` feature branch. Creates the mail directory, sets
  up shared beads via `beads.SetupRedirect`, provisions
  `PRIME.md` via `beads.ProvisionPrimeMDForWorktree`, copies
  overlay files via `rig.CopyOverlay`, runs setup hooks via
  `rig.RunSetupHooks`, ensures gitignore patterns via
  `rig.EnsureGitignorePatterns`, installs runtime settings via
  `runtime.EnsureSettingsForRole`, and saves state. Crucially
  does NOT write to `CLAUDE.md` — the comment at
  `manager.go:320-323` explains that Gas Town context is
  injected via the SessionStart hook so that crew commits never
  leak Gas Town internals into the project repo.
- `Manager.Remove(name, force bool) error`
  (`manager.go:438-467`) — locks, refuses on uncommitted changes
  unless force, removes the crew directory.
- `Manager.List() ([]*CrewWorker, error)`
  (`manager.go:470-495`) — directory scan of `<rig>/crew/`.
- `Manager.Get(name) (*CrewWorker, error)`
  (`manager.go:498-508`) → `getLocked` → `loadState`.
- `Manager.Rename(oldName, newName) error`
  (`manager.go:565-618`) — alphabetical double-lock, directory
  rename, state file update with new name and clone path,
  best-effort rollback on save failure.
- `Manager.Pristine(name) (*PristineResult, error)`
  (`manager.go:622-655`) — runs `git pull --rebase` against the
  current branch and reports whether the workspace had
  uncommitted changes.

### Public API — session lifecycle

- `Manager.SessionName(name) string` (`manager.go:675-677`) —
  delegates to `session.CrewSessionName(rigPrefix, name)`. Note
  that crew and polecat session names share the same shape
  `<prefix>-<name>` — disambiguation is by directory layout, not
  session name.
- `Manager.Start(name string, opts StartOptions) error`
  (`manager.go:680-895`) — locks, gets-or-creates the worker
  via `getLocked` / `addLocked`, re-syncs remotes (so existing
  crew clones pick up post-creation config changes), ensures
  runtime settings, computes env vars BEFORE creating the tmux
  session (so the parent's `GT_ROLE=mayor` cannot leak into the
  crew session — gh#1289), builds the startup command in either
  resume mode (with validated session ID + `buildResumeArgs`) or
  fresh-start mode (with a beacon assembled by
  `session.FormatStartupBeacon` + `BeaconRecipient`). Performs
  validation BEFORE killing any existing session (so a
  validation failure cannot leave the user without a running
  session). Then handles the existing-session case: with
  `KillExisting`, kill via `KillSessionWithProcesses`; without
  `KillExisting` and the agent is alive, return
  `ErrSessionRunning`; without `KillExisting` and the agent is
  dead, treat as zombie and recreate. Strips
  `--dangerously-skip-permissions` in interactive/refresh mode.
  Creates the session via `NewSessionWithCommandAndEnv` (not
  `NewSession + SendKeys`) and stores `GT_PANE_ID` for ZFC
  liveness checks (gt-qmsx). Themes the session, sets `C-b n/p`
  cycle bindings via `SetCrewCycleBindings`, tracks the PID,
  then waits for the agent to start and accepts startup dialogs
  if the preset's `EmitsPermissionWarning` is set. Finally
  starts a background nudge poller via `nudge.StartPoller` so
  queued nudges can be delivered to an idle crew agent (gt-dgf).
- `Manager.Stop(name) error` (`manager.go:898-930`) — stops the
  nudge poller, then `KillSessionWithProcesses`. Returns
  `ErrSessionNotFound` if absent.
- `Manager.IsRunning(name) (bool, error)`
  (`manager.go:933-937`).

### Internals — remote sync (`syncRemotesFromRig`)

`Manager.syncRemotesFromRig(crewPath)` (`manager.go:347-435`).
The dual-source-authority model: `origin`'s push URL comes from
`town.json` via `m.rig.PushURL`; non-`origin` remotes get push
URLs from mayor's git config. The function reads remotes from
`<rig>/mayor/rig`, adds missing ones to the crew clone, updates
URLs that drifted, and configures push URLs per remote. Handles
the case where mayor has no custom push URL by clearing stale
push URLs on the crew side, but only if the crew's push URL
differs from its fetch URL (preserving intentional setups).

### Internals — state file persistence

State lives in `<rig>/crew/<name>/state.json`. `saveState` uses
`util.AtomicWriteJSON` (`manager.go:520-527`). `loadState`
(`manager.go:530-562`) reads the file, but **the directory name
is the source of truth** for `Name` and `ClonePath`: even if the
state file says otherwise, `loadState` overwrites those fields
from the directory path. This makes the manager robust to
directory rename / copy / corruption.

### Notable design choices

- **Full clones, not worktrees.** This is the structural
  difference from polecats. A polecat's sandbox is a worktree of
  the rig's bare repo; a crew workspace is its own clone with
  its own `.git/`. Consequence: the rename, pristine, and remove
  paths cannot just shuffle worktree refs — they have to manage
  the full repo. Consequence two: crew workspaces cost more disk
  but can be operated on independently of the rig's main repo.
- **Validation before destruction.** `Start` builds the entire
  startup command and validates everything before killing any
  existing session (`manager.go:740-744` comment). A validation
  failure must not leave the user without a running session.
- **Env vars resolved BEFORE session creation.** `Start` uses
  `tmux -e` to seed the new session's environment, NOT
  send-keys, because the latter races with the shell and could
  let parent env vars (e.g. `GT_ROLE=mayor`) leak in (gh#1289).
- **No CLAUDE.md writes.** Crew workspaces are project
  repositories that get committed and pushed. Writing Gas Town
  context to `CLAUDE.md` would leak Gas Town internals into the
  project repo. Instead, the SessionStart hook fires on agent
  start and injects context ephemerally via `gt prime`.

## Related wiki pages

- [`crew` role](../roles/crew.md) — domain persona; what a crew
  worker IS.
- [`gt crew`](../commands/crew.md) — CLI surface (10 sibling
  command files).
- [`polecat` package](polecat.md) — the ephemeral counterpart;
  shared shape but very different semantics.
- [`internal/session`](session.md) — `CrewSessionName`,
  `BeaconRecipient`, `FormatStartupBeacon`, `TrackSessionPID`,
  `MergeRuntimeLivenessEnv`.
- [`internal/config`](config.md) — `AgentEnv`,
  `BuildCrewStartupCommandWithAgentOverride`,
  `BuildStartupCommandFromConfig`, `ResolveWorkerAgentConfig`,
  `RoleSettingsDir`, `GetAgentPresetByName`, `ShellQuote`.
- [`internal/beads`](beads.md) — `SetupRedirect`,
  `ProvisionPrimeMDForWorktree`.
- [`internal/nudge`](nudge.md) — `StartPoller` / `StopPoller`.
- [`internal/util`](util.md) — `AtomicWriteJSON`, `RedactURL`.
- [`gt worktree`](../commands/worktree.md) — depends on the
  helper `detectCrewFromCwd` for cross-rig crew context.

## Notes / open questions

- `validateCrewName` rejects hyphens, dots, and spaces because
  they collide with agent ID parsing. Crews are named like rigs
  in this respect. The error message includes a sanitized
  suggestion, which is friendly but means the invalid-name
  rejection IS the validation contract — there's no separate
  `IsValidCrewName(name) bool` predicate exported.
- `loadState` treating directory name as source of truth for
  `Name` and `ClonePath` is a robust choice but means an admin
  who renames a crew dir manually (without `gt crew rename`)
  will get a working crew under the new name with stale
  `Branch` / `CreatedAt` data, silently. Not necessarily a bug,
  but worth a note.
- `Start`'s nudge poller is keyed by session ID. If a crew is
  Stop'd and Start'd quickly, the previous poller may still be
  shutting down — `StopPoller` is best-effort. The
  `nudge.StartPoller` contract presumably handles this, but the
  package boundary doesn't enforce it.
- `validateSessionID` is stricter than necessary for some agent
  ID formats — it rejects `+` and `:` which appear in some
  external ID schemes. If Gas Town adds support for an agent
  whose session IDs use those characters, this validator will
  need to learn them.
- The package has only one test file (`manager_test.go`,
  ~30 KB). It covers happy paths but the locking, race, and
  rollback paths in `Start` would benefit from a separate
  concurrency test file.
