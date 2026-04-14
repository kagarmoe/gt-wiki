---
title: internal/hooks
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/hooks/config.go
  - /home/kimberly/repos/gastown/internal/hooks/installer.go
  - /home/kimberly/repos/gastown/internal/hooks/merge.go
tags: [package, hooks, claude-code, settings, agent-config]
---

# internal/hooks

Centralized Claude Code (and other agent) hook configuration management
for Gas Town. Generates `.claude/settings.json` (and similar
per-provider files) for every role in a town by layering a base config
under role-specific overrides — plus built-in defaults that enforce
policy (patrol-formula guards, session cycling on compaction, polecat
auto-`gt done` on session stop). Also provides a generic template-based
installer for agent runtimes that don't speak the JSON-merge path
(Gemini, OpenCode, Copilot, etc.).

**Go package path:** `github.com/steveyegge/gastown/internal/hooks`
**File count:** 3 go files, 4 test files, plus a `templates/` subdir
containing per-provider hook/settings templates (embedded via
`//go:embed`).
**Imports (notable):** stdlib (`encoding/json`, `embed`, `os`,
`path/filepath`, `runtime`, `strings`), plus
`internal/hookutil` for `IsAutonomousRole`.
**Imported by (notable):** [`gt hook`](../commands/hook.md) /
[`gt hooks`](../commands/hooks.md) (install/sync/override subcommands),
[`gt prime`](../commands/prime.md) (invoked via `SessionStart`
hook), and [`gt signal`](../commands/signal.md) /
`gt tap` command tree referenced from default hook commands.

## What it actually does

### Data model (`config.go`)

- `HookEntry` (`config.go:18-21`) pairs a Claude matcher string with a
  list of `Hook`s to run.
- `Hook` (`config.go:24-27`) is a `{type, command}` pair where `type`
  is always `"command"` in practice.
- `HooksConfig` (`config.go:30-39`) groups entries by event type:
  `PreToolUse`, `PostToolUse`, `SessionStart`, `Stop`, `PreCompact`,
  `UserPromptSubmit`, `WorktreeCreate`, `WorktreeRemove`. The event
  list is also exported as `EventTypes` (`config.go:583`) so callers
  can iterate.
- `SettingsJSON` (`config.go:43-49`) is the full Claude Code
  `settings.json` schema with an `Extra map[string]json.RawMessage`
  that roundtrips unknown fields verbatim so `gt hooks sync`
  doesn't clobber user customizations.
- `SettingsIntegrityError` (`config.go:53-71`) — fail-closed signal
  for malformed settings files. `IsSettingsIntegrityError` unwraps
  error chains to detect it.
- `UnmarshalSettings` / `MarshalSettings` / `LoadSettings`
  (`config.go:74-155`) do the JSON↔struct conversion while preserving
  `Extra` fields on roundtrip. `LoadSettings` wraps parse errors in
  `SettingsIntegrityError` so callers can refuse to overwrite
  corrupt files.

### Base / override resolution (`config.go` + `merge.go`)

- `DefaultBase()` (`config.go:813-906`) — the starter hook set every
  role inherits: `PreToolUse` guards for dangerous commands
  (`rm -rf /*`, `git push --force`, `git push -f`, `gh pr create*`,
  `git checkout -b*`, `git switch -c*`), `SessionStart`→`gt prime --hook`,
  `PreCompact`→`gt prime --hook`, `UserPromptSubmit`→`gt mail check --inject`,
  `Stop`→`gt costs record &`. Every command is wrapped in
  `hookChain(pathSetup, "gt ...")` so Claude's hook runner can find
  `gt` on PATH.
- `DefaultOverrides()` (`config.go:207-351`) — role-specific built-in
  overrides:
  - `polecats`: `Stop → gt tap polecat-stop-check` (catches idle
    polecats who forgot to run `gt done`).
  - `crew`: `PreCompact → gt handoff --cycle --reason compaction`
    (replaces context compaction with a fresh-session handoff).
  - `witness`/`deacon`/`refinery`: `PreToolUse` patrol-formula guards
    that block `bd mol pour` on molecules matching `*patrol*`,
    `*mol-witness*`, `*mol-deacon*`, `*mol-refinery*` — forcing
    patrol formulas to use wisps instead of persistent molecules.
- `LoadBase()` / `LoadOverride(target)` (`config.go:719-747`) — cascade
  through `gtConfigDirs()` (`$GT_HOME/.gt`, then `~/.gt`), returning
  the first file found. `SaveBase` / `SaveOverride`
  (`config.go:751-759`) always write to the **primary** dir.
- `ComputeExpected(target)` (`config.go:365-400`) — the full merge
  engine. Loads the on-disk base (backfilling new hook types from
  `DefaultBase` as a floor), iterates `GetApplicableOverrides(target)`
  (role alone, then `rig/role`), and for each applies built-in
  defaults first then on-disk overrides on top. The resulting
  `HooksConfig` is what should actually be written to the role's
  `.claude/settings.json`.
- `GetApplicableOverrides(target)` (`config.go:916-924`) — given
  `"gastown/crew"` returns `["crew", "gastown/crew"]` so rig-specific
  overrides win over role-only ones.
- `NormalizeTarget` / `ValidTarget` (`config.go:769-809`) — alias
  handling (`polecat`→`polecats`) and validation against the fixed
  role set.

### Merge rules (`merge.go`)

- `MergeHooks(base, overrides, target)` (`merge.go:24-41`) is the
  lower-level merge primitive (no built-in defaults). `ComputeExpected`
  is the full production path; `MergeHooks` is the "just this user
  config" variant.
- `applyOverride` (`merge.go:77-87`) calls `mergeEntries` for every
  event type.
- `mergeEntries(base, override)` (`merge.go:93-130`) implements the
  matcher-level merge: same matcher in both → override replaces base;
  same matcher with empty `Hooks` list → explicit disable (entry is
  removed); new matcher → appended. Different matchers in the base
  that aren't overridden are preserved.
- `cloneConfig` / `cloneEntries` (`merge.go:133-159`) — deep copy so
  merge operations don't mutate the inputs.
- `LoadAllOverrides()` (`merge.go:44-74`) — scans the overrides
  directory, mapping filenames like `gastown__crew.json` back to
  target keys like `gastown/crew`. Invalid files are skipped with
  a stderr warning rather than failing the load.

### Target discovery (`config.go`)

- `DiscoverTargets(townRoot)` (`config.go:406-490`) — walks the town
  directory tree and returns one `Target` per managed
  `.claude/settings.json` location: `mayor/.claude/settings.json`,
  `deacon/.claude/settings.json`, and per-rig `crew/`, `polecats/`,
  `witness/`, `refinery/` parents. All crew members in a rig share
  one settings file (mounted via `--settings` flag at launch), as do
  all polecats.
- `DiscoverRoleLocations(townRoot)` (`config.go:505-550`) is the
  agent-agnostic variant — returns directories rather than
  Claude-specific paths so non-Claude agent config can layer on top.
- `DiscoverWorktrees(roleDir)` (`config.go:555-569`) — lists individual
  worktree subdirectories within a role parent (e.g., `crew/alice`,
  `polecats/toast`).
- `isRig(path)` (`config.go:572-580`) — a directory counts as a rig if
  it contains any of `crew/`, `witness/`, `polecats/`, or `refinery/`.

### Generic installer (`installer.go`)

- Uses `//go:embed templates/*` (`installer.go:20-21`) to ship a
  per-provider template tree inside the binary.
- `InstallForRole(provider, settingsDir, workDir, role, hooksDir,
  hooksFile, useSettingsDir)` (`installer.go:47-62`) — writes the
  template if the file doesn't exist, or overwrites only when
  `needsUpgrade(existing)` detects a stale pattern (legacy
  `export PATH=` format which breaks Gemini CLI's hook runner).
- `SyncForRole(...)` (`installer.go:89-133`) — the explicit-sync
  path used by `gt hooks sync` for template-based providers.
  Structurally compares existing content against the current
  template (tolerating JSON whitespace via `TemplateContentEqual`)
  and overwrites on mismatch. Returns `SyncUnchanged`/`SyncCreated`/
  `SyncUpdated`.
- `resolveTemplate(provider, hooksFile, role)` (`installer.go:191-219`)
  — picks the right template file from the embed. Role-aware agents
  get `<base>-autonomous.<ext>` or `<base>-interactive.<ext>` based
  on `hookutil.IsAutonomousRole`; role-agnostic agents get a single
  template.
- `resolveAndSubstitute` (`installer.go:145-165`) substitutes
  `{{GT_BIN}}` with the resolved gt binary path (via `os.Executable`
  first, then `exec.LookPath`, fallback `"gt"` — `installer.go:245-253`).
  JSON files get the path JSON-encoded so Windows backslashes
  serialize safely.

## Related wiki pages

- [gt](../binaries/gt.md) — binary exposing the hook management CLI.
- [gt hook](../commands/hook.md) / [gt hooks](../commands/hooks.md) —
  user-facing surface.
- [gt prime](../commands/prime.md) — called by the default
  `SessionStart` / `PreCompact` hooks.
- [gt signal](../commands/signal.md) — one of the gt commands that
  hooks route into.
- [go-packages inventory](../inventory/go-packages.md).

## Notes / open questions

- `DiscoverTargets` special-cases the `"mayor"` and `"deacon"`
  directories at the town root (`config.go:410-419`). Other town-level
  agents will not automatically appear as targets without edits here.
- The patrol-formula guards (`config.go:250-349`) are repeated almost
  verbatim for witness, deacon, and refinery — the matcher list is
  literally identical. A future refactor could collapse these into a
  single helper, but the current duplication is explicit and makes
  it grep-discoverable which roles are affected.
- `LoadAllOverrides` writes warnings to stderr for invalid files
  (`merge.go:67`) rather than failing — intentional resilience, but
  it means corrupted overrides silently vanish from `gt hooks sync`.
