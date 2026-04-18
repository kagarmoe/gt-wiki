---
title: internal/state
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/state/state.go
tags: [package, state, xdg, global-toggle, per-machine]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase8_audited: 2026-04-17
phase8_findings: [none]
---

# internal/state

Global Gas Town enable/disable toggle and per-machine state, persisted as
an XDG-compliant JSON file at `~/.local/state/gastown/state.json`. This
is **per-user, per-machine state** — not per-town — so it lives outside
any town root. The file records whether Gas Town is globally enabled,
which shell integration is installed, when `gt doctor` last ran, and a
synthesized machine ID used as a weak identity fingerprint.

**Go package path:** `github.com/steveyegge/gastown/internal/state`
**File count:** 1 go file, 1 test file
**Imports (notable):** stdlib (`encoding/json`, `os`, `path/filepath`,
`time`), [`internal/util`](util.md) for
`util.AtomicWriteJSONWithPerm`, and `github.com/google/uuid` for machine
ID generation.
**Imported by (notable):** the [`gt enable`](../commands/enable.md) and
[`gt disable`](../commands/disable.md) commands, `gt doctor` (writes
`LastDoctorRun`), `gt install` (writes `ShellIntegration`), and any
startup path that needs to check whether Gas Town is globally enabled
before running its logic.

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/state/state.go`.

### The `State` struct (`state.go:17-25`)

Seven JSON-serializable fields:

- `Enabled bool` — the master toggle.
- `Version string` — the version of `gt` that last wrote the file; set by
  `Enable(version)`.
- `MachineID string` — first 8 characters of a `uuid.New()`
  (`state.go:146-148`), generated lazily on first `Enable`/`Disable`.
- `InstalledAt time.Time` — set once on first-ever write.
- `UpdatedAt time.Time` — stamped by `Save` on every write
  (`state.go:107`).
- `ShellIntegration string` — which shell's integration was installed
  (`bash`, `zsh`, `fish`, etc.). Written by `SetShellIntegration`.
- `LastDoctorRun time.Time` — stamped by `RecordDoctorRun`.

### XDG directory helpers (`state.go:29-56`)

Three helpers that all follow the same pattern: check the
corresponding `XDG_*` env var, fall back to the default `~/.local/...`,
`~/.config/...`, or `~/.cache/...` path.

- `StateDir()` → `$XDG_STATE_HOME/gastown` or `~/.local/state/gastown`.
- `ConfigDir()` → `$XDG_CONFIG_HOME/gastown` or `~/.config/gastown`.
- `CacheDir()` → `$XDG_CACHE_HOME/gastown` or `~/.cache/gastown`.
- `StatePath()` → `StateDir()/state.json`.

All three helpers swallow `os.UserHomeDir()` errors (`state.go:34`,
`:44`, `:54`), which means a broken `$HOME` gets you a path like
`.local/state/gastown`. Downstream writes will then fail with a clearer
error message than the home-dir failure would have given.

### Enable/disable toggle

- `IsEnabled() bool` (`state.go:65-80`) — the canonical check. Priority
  order:
  1. `GASTOWN_DISABLED=1` forces `false`.
  2. `GASTOWN_ENABLED=1` forces `true`.
  3. Load the state file; return `state.Enabled`.
  4. If the state file is unreadable for any reason, default to
     **disabled** (`state.go:78`). A missing config is a disabled
     config.
- `Enable(version string) error` (`state.go:113-126`) — loads, creates
  fresh state if missing (including generating a MachineID), sets
  `Enabled=true` and `Version=version`, saves.
- `Disable() error` (`state.go:129-143`) — symmetric; creates a
  disabled-state record if none exists so `MachineID` and `InstalledAt`
  are still captured.

### Load / Save

- `Load() (*State, error)` (`state.go:83-97`) — reads the JSON, no
  migrations or defaults. `os.IsNotExist` is passed through unchanged,
  so callers need to detect "never written" via the error, not by a
  nil return.
- `Save(s *State) error` (`state.go:101-110`) — `os.MkdirAll` the state
  dir at 0755, stamp `UpdatedAt = time.Now()`, then delegate the
  write to `util.AtomicWriteJSONWithPerm(..., 0600)`. **The file
  ends up at 0600 even though the directory is 0755.** This is
  deliberate — the state file contains a machine-id fingerprint and
  should not be world-readable — but the asymmetry is worth noting.

### Ancillary helpers

- `generateMachineID()` (`state.go:146-148`) — `uuid.New().String()[:8]`.
  Not a hardware fingerprint, just a random 8-hex-char identifier. It
  persists across runs once saved but is regenerated on every
  "create fresh state" path.
- `GetMachineID()` (`state.go:151-157`) — returns the stored machine ID
  or a freshly-generated one if the file is unreadable/missing. Note
  that the fresh one is **not saved** by this function — it's ephemeral
  unless a subsequent `Enable`/`Disable`/etc. call persists it.
- `SetShellIntegration(shell string) error` (`state.go:160-170`) — load,
  update `ShellIntegration`, save.
- `RecordDoctorRun() error` (`state.go:173-180`) — load, stamp
  `LastDoctorRun = time.Now()`, save. Unlike other setters, this one
  does **not** create a fresh state on load failure — it returns the
  error directly. A machine without Gas Town installed therefore can't
  record a doctor run; `gt doctor` has to install state first.

## Docs claim

The ABOUTME header (`state.go:1-2`) describes the package as "global
state management for Gas Town enable/disable toggle" using "XDG-compliant
paths for per-machine state storage." The function docs accurately
describe the XDG fallback priorities and the env-var override order for
`IsEnabled`.

## Drift

- **`GetMachineID` silent regeneration.** Calling `GetMachineID()` on a
  machine with no state file returns a new UUID every time — it does not
  persist. Code that treats the result as stable across calls will
  observe different IDs in the same process. `Enable`/`Disable` are the
  only codepaths that actually persist a MachineID.
- **`RecordDoctorRun` vs. the other setters.** `SetShellIntegration`
  creates a fresh state on load failure; `RecordDoctorRun` does not.
  This is an inconsistency that could surprise a reader expecting the
  setters to follow the same pattern.

## Notes / open questions

- There is no `Uninstall` or `DeleteState` helper. Removing a Gas Town
  installation has to `os.Remove(StatePath())` directly, or the user has
  to leave the stale `state.json` behind. Not wrong, but worth knowing
  during test cleanup.
- `UpdatedAt` is set on every save, but `InstalledAt` is only set on
  "create fresh state" branches — if the file is ever hand-edited to
  clear `InstalledAt`, no save path will restore it.

## Related wiki pages

- [gt](../binaries/gt.md) — top-level binary; all user-facing toggles
  run through it.
- [gt enable](../commands/enable.md) — primary caller of
  `state.Enable`.
- [gt disable](../commands/disable.md) — primary caller of
  `state.Disable`.
- [internal/util](util.md) — provides the atomic-write primitive used
  by `Save`.
- [go-packages inventory](../inventory/go-packages.md).
