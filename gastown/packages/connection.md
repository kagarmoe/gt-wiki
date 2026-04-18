---
title: internal/connection
type: package
status: verified
topic: gastown
created: 2026-04-16
updated: 2026-04-18
sources:
  - /home/kimberly/repos/gastown/internal/connection/address.go
  - /home/kimberly/repos/gastown/internal/connection/connection.go
  - /home/kimberly/repos/gastown/internal/connection/local.go
  - /home/kimberly/repos/gastown/internal/connection/registry.go
tags: [package, connection, address, ssh, remote, federation, tmux]
phase3_audited: 2026-04-16
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
phase8_audited: 2026-04-17
phase8_findings: [none]
detail_depth: {params: 2, data_flow: 2, errors: 1, side_effects: 1}
---

# internal/connection

Address parsing and execution abstraction for agent/rig/polecat routing
across local and remote machines. This package defines the `Connection`
interface (file operations, command execution, tmux management), an
`Address` type for the `[machine:]rig[/polecat]` addressing scheme, a
`MachineRegistry` for managing machine configurations, and a
`LocalConnection` implementation for same-host operations.

**Go package path:** `github.com/steveyegge/gastown/internal/connection`
**File count:** 4 non-test Go files (address.go 138 lines,
connection.go 171 lines, local.go 190 lines, registry.go 190 lines;
689 total).
**Imports (notable):** [`internal/tmux`](tmux.md) (`NewTmux` for local
tmux operations in `LocalConnection`). Stdlib: `os`, `os/exec`,
`path/filepath`, `io/fs`, `encoding/json`, `sync`.
**Imported by:** No external importers at current HEAD; the package is
infrastructure for the planned multi-machine federation feature.

## What it actually does

### Address parsing (address.go)

The `Address` struct (`address.go:16-20`) represents a parsed
three-component address:
- `Machine` -- machine name (empty means local).
- `Rig` -- rig name (required).
- `Polecat` -- polecat name (empty means broadcast to rig).

**Address format:** `[machine:]rig[/polecat]`

Examples from the doc comment (`address.go:9-15`):
- `"gastown/rictus"` -- local machine, gastown rig, rictus polecat.
- `"vm:gastown/rictus"` -- vm machine, gastown rig, rictus polecat.
- `"gastown/"` -- local machine, gastown rig, broadcast.

**`ParseAddress(s string) (*Address, error)`** (`address.go:28-57`) --
splits on `:` for machine prefix, then on `/` for rig/polecat. Returns
errors for empty strings, empty machine names, or missing rig names.

**Address methods:**
- `String()` (`address.go:60-76`) -- canonical form reconstruction.
- `IsLocal()` (`address.go:79-81`) -- true if machine is empty or
  `"local"`.
- `IsBroadcast()` (`address.go:84-86`) -- true if polecat is empty.
- `RigPath()` (`address.go:89-94`) -- `rig/polecat` without machine.
- `Validate(registry *MachineRegistry)` (`address.go:98-110`) --
  checks machine exists in registry (rig validation deferred to
  caller).
- `Equal(other *Address)` (`address.go:113-128`) -- normalizes
  empty/`"local"` machine names for comparison.
- `MustParseAddress(s string)` (`address.go:132-138`) -- panics on
  error, for known-good constants.

### Connection interface (connection.go)

**`Connection` interface** (`connection.go:13-79`) -- abstracts three
categories of operations for both local and remote (SSH) contexts:

**File operations** (8 methods):
`ReadFile`, `WriteFile`, `MkdirAll`, `Remove`, `RemoveAll`, `Stat`,
`Glob`, `Exists`.

**Command execution** (3 methods):
`Exec`, `ExecDir`, `ExecEnv` -- run commands with optional directory
and environment overrides.

**Tmux operations** (6 methods):
`TmuxNewSession`, `TmuxKillSession` (uses `KillSessionWithProcesses`
per `connection.go:66`), `TmuxSendKeys`, `TmuxCapturePane`,
`TmuxHasSession`, `TmuxListSessions`.

**Identification** (2 methods):
`Name()`, `IsLocal()`.

**`FileInfo` interface** (`connection.go:84-99`) -- abstracts
`fs.FileInfo` for remote use (the standard `fs.FileInfo` contains
methods that cannot be serialized over SSH). `BasicFileInfo`
(`connection.go:102-108`) provides a JSON-serializable implementation
with `FromOSFileInfo` converter (`connection.go:126-134`).

**Error types** (`connection.go:137-171`):
- `ConnectionError` -- connection-level failure with `Op` and `Machine`.
- `NotFoundError` -- file/resource not found.
- `PermissionError` -- access denied with `Op` and `Path`.

### LocalConnection (local.go)

`LocalConnection` (`local.go:13-15`) implements `Connection` for
same-host operations, delegating to `os` for file operations, `exec`
for commands, and [`internal/tmux`](tmux.md) for session management.

Key implementation details:
- File operation errors are wrapped into typed `NotFoundError` /
  `PermissionError` for consistent error handling across local and
  future remote implementations (`local.go:35-58`).
- `Remove` tolerates already-removed files (returns nil for
  `os.IsNotExist`) (`local.go:77-78`).
- `TmuxKillSession` delegates to `tmux.KillSessionWithProcesses`
  to ensure all descendant processes are killed (`local.go:165-167`).
- Compile-time interface check: `var _ Connection = (*LocalConnection)(nil)`
  (`local.go:190`).

### MachineRegistry (registry.go)

`MachineRegistry` (`registry.go:28-33`) manages machine configurations
persisted as a JSON file.

**`Machine` struct** (`registry.go:13-19`): `Name`, `Type` (`"local"`
or `"ssh"`), `Host` (for SSH: `user@host`), `KeyPath` (SSH private
key), `TownPath` (remote town root).

**Registry lifecycle:**
- `NewMachineRegistry(configPath string)` (`registry.go:36-56`) --
  loads from JSON if present, ensures a `"local"` machine always
  exists.
- `load()` / `save()` (`registry.go:59-106`) -- JSON
  read/write with `registryData` envelope (version + machines map).

**CRUD operations:**
- `Get(name string)` (`registry.go:109-118`), `Add(m *Machine)`
  (`registry.go:121-137`) -- validates type and host requirements,
  auto-saves.
- `Remove(name string)` (`registry.go:140-154`) -- prevents removal
  of the `"local"` machine.
- `List()` (`registry.go:157-166`) -- returns all machines.

**Connection factory:**
- `Connection(name string)` (`registry.go:169-184`) -- returns a
  `Connection` for the named machine. Local machines get
  `NewLocalConnection()`. SSH returns `"not yet implemented"` error
  (`registry.go:180`).
- `LocalConnection()` (`registry.go:188-190`) -- convenience shortcut.

## Related

- [internal/tmux](tmux.md) -- tmux wrapper used by `LocalConnection`
  for all session operations.
- [rig concept](../concepts/rig.md) -- the rig abstraction that
  addresses target.
- [polecat role](../roles/polecat.md) -- the polecat worker that
  addresses identify.
- [go-packages inventory](../inventory/go-packages.md)

## Outgoing calls

### Subprocess invocations
| Called binary | Command | Flags | Flag source | `file:line` |
|---|---|---|---|---|
| (dynamic) | (dynamic) | `args...` (CombinedOutput) | caller | `local.go:138` |
| (dynamic) | (dynamic) | `args...` (Start) | caller | `local.go:143` |
| (dynamic) | (dynamic) | `args...` (Run) | caller | `local.go:150` |

### File writes
| Target | What is written | Purpose | `file:line` |
|---|---|---|---|
| arbitrary path | caller data | `LocalConnection.WriteFile` | `local.go:51` |
| registry file | JSON data | Connection registry persistence | `registry.go:101` |

## Notes / open questions

- SSH connections are explicitly stubbed: `registry.go:180` returns
  `"ssh connections not yet implemented"`. The `Machine` struct and
  registry infrastructure are in place, but the `SSHConnection`
  implementation does not exist yet. This is the foundation for Gas
  Town's planned multi-machine federation.
- The package has zero importers at current HEAD, confirming it is
  pre-integration infrastructure.
- The `Connection` interface is comprehensive (19 methods across 4
  categories). Future SSH implementation will need to handle all file,
  exec, and tmux operations over SSH, which is a significant surface
  area.
- `FileInfo` was extracted as a separate interface rather than using
  `fs.FileInfo` because the standard interface's `Sys()` method
  returns `any`, making it unsuitable for serialization across SSH
  boundaries.
- The `Address.Validate` method only checks machine existence; rig
  validation is explicitly deferred to the caller because it would
  require an active connection to the target machine.
