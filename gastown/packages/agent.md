---
title: internal/agent
type: package
status: verified
topic: gastown
created: 2026-04-16
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/agent/state.go
tags: [package, agent, state-management, generics]
phase3_audited: 2026-04-16
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
phase8_audited: 2026-04-17
phase8_findings: [none]
---

# internal/agent

Shared agent types and a generic `StateManager[T]` for persisting agent
state to disk. This is a small (62-line, single-file) package that
provides the common state-persistence abstraction used by agent roles
that need to save and load structured JSON state between daemon ticks.

**Go package path:** `github.com/steveyegge/gastown/internal/agent`
**File count:** 1 non-test Go file (state.go, 62 lines).
**Imports (notable):** [`internal/util`](util.md) (`AtomicWriteJSON`
for crash-safe writes).
**Imported by:** No external importers at current HEAD; the package is
newly introduced and not yet wired into agent runtimes.

## What it actually does

### StateManager[T] (state.go)

`StateManager[T any]` is a generic struct that handles loading and
saving a JSON-serializable state type `T` to a well-known file path
under a rig's `.runtime/` directory (`state.go:15-18`).

**Construction:**

- `NewStateManager[T](rigPath, stateFileName string, defaultFactory func() *T) *StateManager[T]`
  (`state.go:23-28`) -- builds the state file path as
  `<rigPath>/.runtime/<stateFileName>` and stores a factory function
  for creating default state when no file exists.

**Public API:**

- `StateFile() string` (`state.go:31-33`) -- returns the resolved
  state file path.

- `Load() (*T, error)` (`state.go:37-52`) -- reads and unmarshals the
  JSON state file. If the file does not exist, calls `defaultFactory()`
  and returns a fresh default state (no error). This means first-run
  scenarios silently start with defaults.

- `Save(state *T) error` (`state.go:55-62`) -- creates the parent
  directory if needed (`os.MkdirAll`), then delegates to
  `util.AtomicWriteJSON` (`state.go:61`). The atomic write ensures a
  crash mid-save does not corrupt the state file.

### Design notes

The generics-based approach avoids the need for each agent role
(witness, refinery, deacon, etc.) to reimplement JSON file load/save
logic. Each role defines its own state struct and passes it as the type
parameter.

## Related

- [internal/util](util.md) -- `AtomicWriteJSON` used by `Save`.
- [internal/witness](witness.md) -- agent role that would use state
  persistence for health-monitor state.
- [internal/refinery](refinery.md) -- agent role that would use state
  persistence for merge-queue state.
- [internal/deacon](deacon.md) -- agent role that would use state
  persistence for watchdog state.
- [internal/agent/provider](agent-provider.md) -- sibling sub-package
  providing JSON-RPC provider types.
- [go-packages inventory](../inventory/go-packages.md)

## Notes / open questions

- The package has zero importers at current HEAD. It appears to be
  newly added infrastructure awaiting integration into agent runtimes.
  Once agents adopt it, each agent's bespoke state load/save code can
  be replaced by a single `StateManager` instantiation.
- The `defaultFactory` pattern is a clean way to handle first-run
  initialization without requiring callers to check for nil returns.
- No locking is provided by `StateManager` itself; concurrent
  `Load`/`Save` calls from different goroutines would need external
  synchronization. This is acceptable because each agent role runs
  as a single-session process.
