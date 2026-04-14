---
title: internal/plugin
type: package
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/plugin/types.go
  - /home/kimberly/repos/gastown/internal/plugin/scanner.go
  - /home/kimberly/repos/gastown/internal/plugin/recording.go
  - /home/kimberly/repos/gastown/internal/plugin/sync.go
tags: [package, agent-runtime, plugin, deacon, patrol, toml-frontmatter, exec-wrapper, sync, drift]
---

# internal/plugin

The plugin-discovery + plugin-run-recording + plugin-sync
package. Plugins are `plugin.md` files with TOML frontmatter
that define periodic automation tasks run by the Deacon's
patrol loop. Each plugin declares a **gate** (how it decides
whether it should run now), an **execution type** (agent
interpreted, shell script, or exec-wrapper), and optional
**tracking** metadata (labels, digest flag). This package owns
the scanner that finds plugins, the recorder that writes
ephemeral bd beads for runs, and the sync engine that copies
plugin directories between a source tree and a runtime
location with drift detection.

**Go package path:** `github.com/steveyegge/gastown/internal/plugin`
**File count:** 4 non-test `.go` files. `sync.go` is 334 lines,
`scanner.go` is 273, `types.go` is 248, `recording.go` is 232.
**Concept:** plugins are the Deacon's runtime-extension story.
There is no separate `concepts/plugin.md` page yet (see Notes).
**CLI command:** [`gt plugin`](../commands/plugin.md) —
`list`, `show`, `run`, `sync`, `history`.
**Imports (notable):** `github.com/BurntSushi/toml`
(frontmatter parsing), [`internal/beads`](beads.md)
(`ResolveBeadsDir` + bd subprocess calls), `internal/constants`
(`BdCommandTimeout`), `crypto/sha256` (directory hashing for
drift).
**Imported by:** [`internal/cmd/plugin.go`](../commands/plugin.md)
(the CLI), the Deacon's patrol loop and dispatcher (for running
plugins during patrol cycles), [`internal/doctor`](doctor.md)
(the `PatrolPluginsAccessible` and `PatrolPluginDrift` health
checks), and the session-startup path that reads exec-wrapper
plugins to wrap agent commands.

## What it actually does

Four files. Three concerns: **the data shape and frontmatter
parsing** (types.go, scanner.go), **run recording** via
ephemeral beads (recording.go), and **directory sync with drift
detection** (sync.go).

### types.go — data model and execution envelope

Source: `/home/kimberly/repos/gastown/internal/plugin/types.go`.

**The `Plugin` struct** (`types.go:17-52`):

- **Identity:** `Name` (from frontmatter), `Description`,
  `Version`, `Location` (`town` vs `rig`), `Path` (absolute
  path to the plugin directory), `RigName` (empty for town).
- **Gating:** `Gate *Gate` — nil means manual.
- **Tracking:** `Tracking *Tracking` — labels + digest flag.
- **Execution:** `Execution *Execution` — type, timeout,
  notification settings, and wrapper args for exec-wrapper.
- **Body:** `Instructions` (the markdown body after the
  frontmatter) and `HasRunScript bool` (set when a `run.sh`
  exists alongside `plugin.md`).

**`Location`** (`types.go:55-63`): `town` or `rig`. Town-level
plugins live at `<townRoot>/plugins/<name>/plugin.md`; rig-level
plugins live at `<townRoot>/<rigName>/plugins/<name>/plugin.md`.
Rig-level plugins **override** town-level plugins of the same
name (see `scanner.DiscoverAll` below).

**`GateType`** (`types.go:84-101`): five gate shapes, one per
decision style:

- **`cooldown`** — run if enough time has passed since the last
  run. Uses `Gate.Duration` (e.g., `"1h"`, `"24h"`). This is
  the only gate type actively enforced by `gt plugin run`
  (see the [command page](../commands/plugin.md)).
- **`cron`** — run on a cron schedule. Uses `Gate.Schedule`
  (e.g., `"0 9 * * *"`).
- **`condition`** — run if a check command returns exit 0. Uses
  `Gate.Check`.
- **`event`** — run on specific events like `startup`. Uses
  `Gate.On`.
- **`manual`** — never auto-runs; must be explicitly triggered.

**`Tracking`** (`types.go:104-110`): `Labels []string` applied
to execution wisps; `Digest bool` controls daily-digest
inclusion.

**`ExecutionType`** (`types.go:113-127`):

- **`agent`** (default) — a dog worker interprets the markdown
  instructions as a prompt. The `FormatMailBody` output is the
  dog's session input.
- **`script`** — a `run.sh` is executed directly. Set when the
  scanner detects `run.sh` alongside `plugin.md` (`HasRunScript`
  is true).
- **`exec-wrapper`** — wraps session startup commands. Instead
  of being dispatched to a dog, the wrapper tokens are inserted
  between `exec env VAR=val ...` and the agent binary in the
  startup command. Example: `["exitbox", "run",
  "--profile=gastown-polecat", "--"]`. This is how the sandbox
  wrapper around polecat agent processes is pluggable.

**`Execution`** (`types.go:130-148`): `Type`, `Timeout`
(e.g., `"5m"`), `NotifyOnFailure bool`, `Severity string`,
`Wrapper []string` (only used for `exec-wrapper`).

**`IsExecWrapper()`** (`types.go:161-163`) and
**`ExecWrapperArgs()`** (`types.go:167-172`) are the two
accessors the session-startup path uses to look up wrapper
plugins.

**`PluginSummary`** (`types.go:175-183`): the compact JSON
shape used by `list` output. `Summary()` (`types.go:186-208`)
derives `GateType` (defaulting to `manual` when `Gate` is nil)
and `ExecutionType`.

**`FormatMailBody()`** (`types.go:213-248`): the canonical
text sent to a dog that's dispatched to run a plugin. Two
branches:

- **If `HasRunScript`:** send an unambiguous "run `bash run.sh`"
  message with instructions to NOT interpret the markdown. Dogs
  with a script plugin don't get creative — they execute the
  script and report.
- **Otherwise:** send the plugin name, description, rig name,
  timeout, and the markdown `Instructions` as the dog's task
  prompt.

Both branches end with a "after completion" postscript telling
the dog to record a plugin-run receipt via `bd create
--ephemeral` and then call `gt dog done`.

### scanner.go — plugin discovery

Source: `/home/kimberly/repos/gastown/internal/plugin/scanner.go`.

**`Scanner`** (`:13-23`) is bound to `townRoot + rigNames`.
`NewScanner(townRoot, rigNames)` is a pure constructor.

**`DiscoverAll()`** (`scanner.go:29-61`): the canonical
discovery path.

1. Scan town-level plugins (`<townRoot>/plugins/`).
2. Scan each rig's plugin directory
   (`<townRoot>/<rigName>/plugins/`).
3. Deduplicate by name — **rig-level entries override
   town-level** when they share a name. This is the override
   semantics that makes `<rig>/plugins/` take precedence over
   `~/gt/plugins/`.
4. Return the union as a slice.

Per-rig scan errors are logged to stderr and skipped — one
broken rig doesn't break plugin discovery for the other rigs.

**`scanDirectory`** (`scanner.go:76-117`): walks a single
plugins directory, loading each subdirectory that contains a
`plugin.md`. Non-directories and dotfile-prefixed entries are
skipped. Individual plugin parse errors are warnings, not
fatal.

**`loadPlugin`** (`scanner.go:120-145`): reads `plugin.md`,
calls `parsePluginMD`, then checks for `run.sh` alongside it
and sets `HasRunScript` if present.

**`parsePluginMD`** (`scanner.go:156-202`): parses the
frontmatter format:

```
+++
name = "plugin-name"
description = "..."
[gate]
type = "cooldown"
duration = "1h"
+++
# Instructions
...markdown body...
```

Delimited by `+++` at both ends. Parses the frontmatter as
TOML via `toml.Decode`, validates that `name` is set, and
returns a `Plugin` with `Instructions` set to the body
(everything after the closing `+++`, trimmed).

**`GetPlugin(name)`** (`scanner.go:206-230`): single-plugin
lookup. **Searches rig-level FIRST** (rigs override town),
then falls back to town-level. Returns an error if not found.

**Exec-wrapper discovery:**

- `DiscoverExecWrappers()` (`scanner.go:234-247`): filter
  `DiscoverAll()` by `IsExecWrapper()`.
- `GetExecWrapper(name)` (`scanner.go:251-264`): single
  exec-wrapper lookup with a type-check error message.

**`ListPluginDirs()`** (`scanner.go:267-273`): returns the list
of directories the scanner will look in, used by `gt plugin
sync` and doctor.

### recording.go — plugin run ledger

Source: `/home/kimberly/repos/gastown/internal/plugin/recording.go`.

Plugin runs are recorded as **ephemeral beads** — wisps whose
purpose is audit + cooldown gate computation. This file is the
thin abstraction over `bd create --ephemeral` / `bd list`.

**`RunResult`** (`recording.go:17-23`): `success`, `failure`,
`skipped`.

**`PluginRunRecord`** (`recording.go:26-31`): the create
payload — plugin name, rig name, result, body (optional).

**`PluginRunBead`** (`recording.go:34-40`): the query result
shape — `ID`, `Title`, `CreatedAt`, `Labels`, and a derived
`Result`.

**`Recorder`** (`recording.go:43-50`): bound to `townRoot`.

**`RecordRun(record)`** (`recording.go:54-115`): the write path.

1. Build `bd create --ephemeral --json --title=<...>` with
   labels `type:plugin-run`, `plugin:<name>`, `result:<result>`,
   and optionally `rig:<rigName>`.
2. Run `bd` with `BEADS_DIR` set explicitly to the town's
   resolved beads dir — prevents inherited environment vars
   from routing writes to the wrong prefix.
3. Parse the JSON output to get the created bead ID.
4. **Immediately close the receipt** with `bd close <id>
   --reason "plugin run recorded"`. The receipt exists for
   audit and cooldown-gate queries (which use `--all` to
   include closed beads) but should not stay open. Close is
   best-effort: `_ = closeCmd.Run()`.

**`GetLastRun(pluginName)`** (`recording.go:119-128`) and
**`GetRunsSince(pluginName, since)`** (`recording.go:132-134`):
thin wrappers around `queryRuns`.

**`queryRuns`** (`recording.go:137-222`): the read path.
`bd list --json --all -l type:plugin-run -l plugin:<name>
[--limit=N] [--created-after=<cutoff>]`. Two subtleties:

- **Duration parsing:** `time.ParseDuration(since)` is used
  (Go's duration grammar — `1h`, `24h`, `30m`). The absolute
  cutoff is computed and passed as RFC3339 via
  `--created-after`. The comment at `recording.go:149-152`
  explains why: bd's compact duration uses `m` for *months*,
  but Go durations use `m` for minutes. Passing an absolute
  timestamp avoids the unit mismatch.
- **Empty-result handling:** `bd` with no matches returns
  `[]\n` on stdout; `queryRuns` detects both that and empty
  stdout as the no-results case and returns `nil, nil`.

**`CountRunsSince`** (`recording.go:226-232`) is the cooldown
gate's primitive: count how many runs have happened in the
last duration, used by `gt plugin run` to evaluate whether the
gate is open.

### sync.go — directory sync with drift detection

Source: `/home/kimberly/repos/gastown/internal/plugin/sync.go`.

**`SyncResult`** (`sync.go:14-20`): `Copied`, `Removed`,
`Skipped`, `Errors` — the outcome categories for a sync run.

**`SyncPlugins(sourceDir, targetDir, clean)`**
(`sync.go:23-89`): copies plugin directories from source to
target.

1. Enumerate source directories that contain a `plugin.md`
   (skip everything else, including dotfiles).
2. For each source plugin:
   - If `dirsMatch(src, dst)` (same hash), skip.
   - Otherwise, `copyDir` (atomic swap via temp dir +
     rename).
3. If `clean`: remove target plugins that don't exist in
   source.

**`dirsMatch`** (`sync.go:92-96`) uses **`DirHash`**
(`sync.go:99-121`) — a deterministic content hash of every
file in a plugin directory, walked depth-first with relative
paths in the hash input. Empty directories hash to the empty
state; any mismatch triggers a copy.

**`copyDir`** (`sync.go:125-155`): **atomic directory
replacement.** Copies to a temp dir in the same parent, then
removes the old destination and renames the temp into place.
The temp dir is cleaned up on failure by a `defer`.

**`FindGastownSource(townRoot)`** (`sync.go:184-202`): locates
the source-of-truth `plugins/` directory via three strategies:

1. **Walk up from CWD:** look for a `go.mod` that declares a
   `gastown` module and a sibling `plugins/` directory. Uses
   `isGastownModule` (`sync.go:226-240`) to confirm the module
   path ends in `/gastown`.
2. **Town-relative fallback 1:** `<townRoot>/gastown/crew/den/plugins/`.
3. **Town-relative fallback 2:** `<townRoot>/gastown/plugins/`.

Returns an error if none are found. The CLI's `--source` flag
short-circuits this search when the operator wants to specify
a path.

**Drift detection:**

- `DriftReport` (`sync.go:258-264`) has `Drifted`, `Missing`
  (in source but not target), and `Extra` (in target but not
  source) categories.
- `DriftEntry` (`sync.go:267-271`): `Name`, `SourceHash`,
  `TargetHash`.
- `DetectDrift(sourceDir, targetDir)` (`sync.go:274-329`):
  walks both sides, hashes each, and categorizes. Extras are
  only included for target plugin directories that have a
  `plugin.md` (so unrelated dotfiles aren't reported).
- `HasDrift()` (`sync.go:332-334`): true if `Drifted` or
  `Missing` is non-empty. Note: `Extra` is NOT part of the
  drift predicate — extra target-side plugins aren't
  considered drift because they may be rig-specific additions.

## Notable design choices

- **TOML frontmatter in markdown.** Plugins are a hybrid
  shape: TOML for the declarative config, markdown for the
  instructions-to-the-agent body. This lets a plugin be
  readable as docs AND parseable as config without a
  sidecar file.
- **Rig-level overrides town-level.** Unlike
  [formula overlays](formula.md) (where town and rig are
  concatenated), plugins use a clean override: a rig plugin
  with the same name fully replaces the town one. Plugins are
  replaceable units, not composable definitions.
- **Cooldown gate is the only one enforced by `gt plugin
  run`.** The other gate types (`cron`, `condition`, `event`,
  `manual`) fall through unconditionally — manual invocation
  bypasses them. Automatic dispatch by the Deacon is where
  those gates actually fire (handled elsewhere — see the
  `gt-n08ix.2` reference in
  [`gt plugin`](../commands/plugin.md)).
- **Run records are ephemeral beads, immediately closed.**
  The audit trail is bd beads, not a separate ledger file.
  Closing them immediately makes them invisible to `bd list`
  by default but queryable via `--all`. The [reaper](reaper.md)
  eventually purges them after the purge age.
- **Duration unit-mismatch defense.** `bd`'s `--created-after`
  with a compact-duration string would interpret `m` as
  months; this code pre-computes an absolute RFC3339 timestamp
  using Go's duration parser so the unit semantics are
  gastown-side, not bd-side. This is a real bug class, not
  theoretical.
- **`run.sh` presence flips execution behavior.** The scanner
  sets `HasRunScript` on disk-layout detection, and
  `FormatMailBody` branches on it to produce unambiguous
  "run the script" instructions instead of "interpret the
  markdown." This keeps plugin authors from having to pick a
  `type: script` flag manually — the script's existence is
  the signal.
- **Atomic directory swap.** `copyDir` uses temp-dir + rename
  so an interrupted sync doesn't leave a half-copied plugin
  directory. Partial copy would be worse than no copy because
  the scanner would still try to load it.
- **Content-hashed drift, not mtime.** `DirHash` is
  file-content-based, so cloning or touching files doesn't
  create spurious drift. Mtime-based drift detection is a
  perennial source of false positives this implementation
  avoids.
- **Extras are not drift.** `HasDrift()` ignores `Extra`
  deliberately — rigs may have rig-specific plugins that
  shouldn't be reported against a town source. Only `Drifted`
  and `Missing` count as drift.

## Related wiki pages

- [`gt plugin`](../commands/plugin.md) — CLI surface (five
  subcommands).
- [`deacon` role](../roles/deacon.md) — primary consumer of
  plugins (auto-dispatches plugin runs during patrol).
- [`deacon` package](deacon.md) — the Deacon code that reads
  the scanner and dispatches dogs.
- [`dog` role](../roles/dog.md) — the disposable worker that
  executes a plugin's `FormatMailBody` instructions.
- [`internal/rig`](rig.md) — `createPluginDirectories` seeds
  `<rig>/plugins/` during `AddRig`. `ListRigNames` feeds
  `Scanner.NewScanner`.
- [`internal/beads`](beads.md) — `ResolveBeadsDir` and the
  bd subprocess shell-outs for create/list/close.
- [`internal/doctor`](doctor.md) —
  `PatrolPluginsAccessible` and `PatrolPluginDrift` health
  checks.
- [wisp concept](../concepts/wisp.md) — plugin run records
  are ephemeral beads (wisps).
- [`gt doctor`](../commands/doctor.md) — invokes the plugin
  health checks.
- [`gt dog`](../commands/dog.md) — the dispatcher that
  receives `FormatMailBody` output.

## Notes / open questions

- **No plugin concept page exists yet.** A
  `concepts/plugin.md` would consolidate the plugin model
  (TOML frontmatter + markdown body + run.sh + exec-wrapper +
  gates) into one narrative instead of scattering it across
  this page, `commands/plugin.md`, and the Deacon's role
  page.
- **`FindGastownSource` couples plugin sync to the gastown
  repo layout.** The fallback paths
  (`<townRoot>/gastown/crew/den/plugins/`,
  `<townRoot>/gastown/plugins/`) assume a specific on-disk
  layout. An operator with a differently-structured source
  tree must use `--source`.
- **No `PluginError` type.** Individual loader errors are
  formatted as warnings via `fmt.Fprintf(os.Stderr, ...)`.
  There's no structured error channel for the CLI to pipe
  into JSON output — `list` silently skips broken plugins.
- **`closeCmd.Run()` is fire-and-forget.** If the close fails,
  an open plugin-run receipt stays in the beads database.
  The reaper eventually catches it, but until then it's
  visible in `bd list`. Worth a counter in `gt doctor`.
- **Exec-wrappers are discovered via the same scanner.** The
  session-startup code that wraps polecat commands queries
  the scanner for every session start. If the scan is slow
  (many plugins, cold filesystem), startup latency grows.
  Worth a cache for the hot path.
- **`DetectDrift` doesn't report per-file hashes.** The report
  has `SourceHash`/`TargetHash` at the plugin-directory
  level, not per-file. A drift case where a single file
  changed is reported as the whole plugin differing.
