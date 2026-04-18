---
title: gastown plugins
type: note
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/plugins/
tags: [plugins, inventory, deacon-patrol, toml-frontmatter]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
phase8_audited: 2026-04-17
phase8_findings: [cross-ref-only]
detail_depth: {synthesis: 2, relationships: 2}
---

# gastown plugins inventory

`plugins/` at the gastown repo root contains **14 plugin
directories** that the [Deacon](../roles/deacon.md) loads via
the [`internal/plugin` package](../packages/plugin.md) and
runs on schedule (or on events) via [`gt plugin`](../commands/plugin.md).
All plugins except [`dolt-snapshots`](dolt-snapshots.md) are
**declarative** — a `plugin.md` with TOML frontmatter plus an
optional `run.sh` shell script. Only `dolt-snapshots` has Go
source code (a standalone binary compiled on first run by its
`run.sh`).

Every plugin follows the schema the scanner at
`/home/kimberly/repos/gastown/internal/plugin/scanner.go:156-202`
parses: `+++`-delimited TOML frontmatter followed by a markdown
body that is either the instructions for a dog agent (the default
`agent` execution type) or reference documentation for the
companion `run.sh` (the `script` execution type — triggered by
`HasRunScript` when `scanner.loadPlugin` finds `run.sh` alongside
`plugin.md`).

## Plugin catalog

| plugin | description (from frontmatter) | files | Go? | gate |
|---|---|---:|:---:|---|
| compactor-dog | Monitor Dolt commit growth across production DBs and escalate when compaction is needed | 3 (+`run_test.sh`) | no | cooldown 30m |
| dolt-archive | Offsite backup: JSONL snapshots to git, dolt push to GitHub/DoltHub | 2 | no | cooldown 1h |
| dolt-backup | Smart Dolt database backup with change detection (v2) | 2 | no | cooldown 15m |
| dolt-log-rotate | Rotate Dolt server log file when it exceeds size threshold | 2 | no | cooldown 6h |
| [dolt-snapshots](dolt-snapshots.md) | Tag Dolt databases at convoy boundaries for audit, diff, and rollback | 7 | **yes** | event `convoy.created` |
| github-sheriff | Monitor GitHub CI checks on open PRs and create beads for failures | 1 | no | cooldown 2h |
| git-hygiene | Clean up stale git branches, stashes, and loose objects across all rig repos | 2 | no | cooldown 12h |
| gitignore-reconcile | Auto-untrack files that are tracked but match an active .gitignore rule | 2 | no | cooldown 6h |
| quality-review | Review merge quality and track per-worker trends | 1 | no | cooldown 6h |
| rate-limit-watchdog | Auto-estop on API rate limit, auto-thaw when clear — no LLM needed | 2 | no | cooldown 3m |
| rebuild-gt | Rebuild stale gt binary from gastown source (v2) | 2 | no | cooldown 1h |
| stuck-agent-dog | Context-aware stuck/crashed agent detection and restart for polecats and deacons | 2 | no | cooldown 5m |
| submodule-commit | Auto-commit accumulated changes inside git submodules and update parent pointer | 2 | no | cooldown 2h |
| tool-updater | Upgrade beads (bd) and dolt via Homebrew when updates are available | 2 | no | cooldown 168h |

**Totals:** 14 plugins, 2 Go files (both in `dolt-snapshots`),
13 declarative (shell + TOML), 13 on cooldown gates, 1 on an
event gate.

## How plugins work

Plugin discovery happens via
[`internal/plugin.Scanner.DiscoverAll`](../packages/plugin.md)
at `/home/kimberly/repos/gastown/internal/plugin/scanner.go:29-61`.
The scanner walks `<townRoot>/plugins/` **and** each rig's
`<townRoot>/<rigName>/plugins/` — rig-level plugins **override**
town-level plugins of the same name. These 14 directories are
the **gastown source-of-truth tree** — they get copied into a
town root via `gt plugin sync` (backed by
`plugin.SyncPlugins` at `sync.go:23-89`) which detects source
location via `FindGastownSource` (`sync.go:184-202`) looking for
`<townRoot>/gastown/crew/den/plugins/` or
`<townRoot>/gastown/plugins/`.

The Deacon's patrol loop is the primary consumer: on each tick,
it evaluates each plugin's gate (cooldown queried via the `bd
list type:plugin-run` ledger from
[`recording.go`](../packages/plugin.md#recordinggo--plugin-run-ledger),
cron via time match, event from the watchdog, condition via a
probe command, manual skipped). When a gate opens, the Deacon
dispatches a dog worker with the body produced by
`Plugin.FormatMailBody` (`types.go:213-248`):

- **If `HasRunScript`** (12 of the 14 — everything except
  `github-sheriff` and `quality-review`, which have no `run.sh`):
  the dog gets an unambiguous "run `bash run.sh` and report" —
  do NOT interpret the markdown body as instructions. These
  plugins use the markdown body as developer documentation
  alongside the script.
- **Otherwise** (`github-sheriff`, `quality-review`): the
  markdown body IS the dog's task prompt. The dog interprets the
  Steps sections in order, making judgment calls in the sections
  that say "this is where you (the dog agent) use judgment."

Plugin runs are recorded as **ephemeral `bd` beads** (wisps)
with labels `type:plugin-run`, `plugin:<name>`, and
`result:{success,failure,warning,skipped}`. The bead is created
and then immediately closed so it doesn't clutter `bd list`;
queryable via `bd list --all`. See
[`plugin.Recorder.RecordRun`](../packages/plugin.md#recordinggo--plugin-run-ledger)
(`recording.go:54-115`) for the write path — every plugin
manifest includes a "Record Result" section demonstrating the
convention even though the recording is actually done by the
dispatcher wrapper, not the plugin body itself.

## Plugin shape summary

Every `plugin.md` starts with `+++`-delimited TOML frontmatter
containing four tables:

```toml
+++
name = "plugin-name"
description = "one-line"
version = N

[gate]
type = "cooldown" | "event" | "cron" | "condition" | "manual"
duration = "1h"   # for cooldown
on = "event.name" # for event
schedule = "..."  # for cron
check = "..."     # for condition

[tracking]
labels = ["plugin:plugin-name", "category:..."]
digest = true

[execution]
timeout = "5m"
notify_on_failure = true
severity = "low" | "medium" | "high" | "critical"
+++
```

## Neutral observations

- **Two plugins have no `run.sh`.** `github-sheriff` and
  `quality-review` are **agent-interpreted** — their entire
  markdown bodies are dog instructions. The scanner sets
  `HasRunScript = false` and `FormatMailBody` sends the markdown
  body as the task prompt.
- **`dolt-snapshots` is the only plugin with Go code.** Its
  `run.sh` is a build-if-stale shim that compiles `main.go`
  into a binary and execs it. See the
  [`dolt-snapshots` page](dolt-snapshots.md) for details.
- **`dolt-snapshots` is also the only `event`-gated plugin.**
  Its gate is `type = "event"`, `on = "convoy.created"`, and
  the Go binary has a `--watch` mode that tails `~/gt/.events.jsonl`
  directly for sub-second response instead of relying on patrol
  polling.
- **Three plugins share the same Go binary.** The
  `dolt-snapshots` plugin.md declares siblings
  `dolt-snapshots-staged` (on `convoy.staged`) and
  `dolt-snapshots-launched` (on `convoy.launched`) — but only
  `dolt-snapshots/` exists as a directory in the inventory. The
  sibling plugins are not present in `plugins/` at the repo
  root; they are either runtime-only, documented-but-unshipped,
  or live in a rig's plugin directory.
- **Cooldown durations span six orders of magnitude.**
  `rate-limit-watchdog` runs every 3 minutes; `tool-updater`
  runs every 168 hours (1 week).
- **Categories are free-form labels.** `category:maintenance`,
  `category:data-safety`, `category:ci-monitoring`,
  `category:cleanup`, `category:git-hygiene`, `category:quality`,
  `category:health`, `category:safety` — no enum enforcement;
  just the labels attached to tracking wisps.
- **Severity tiers are used.** Critical:
  `dolt-archive`. High: `dolt-backup`, `rate-limit-watchdog`,
  `stuck-agent-dog`. Medium: `compactor-dog`, `dolt-log-rotate`,
  `quality-review`, `rebuild-gt`, `tool-updater`. Low: remaining.
- **Rate-limit-watchdog is the only purely shell plugin that
  explicitly calls itself "shell-only — no LLM calls."** Its
  3-minute cooldown assumes no agent in the loop.
- **`submodule-commit` is opt-in per rig** — its frontmatter
  comment explains the rig-level plugin config shape
  (`[plugin.submodule-commit] enabled = true`). It's the only
  plugin with this documented opt-in pattern.
- **`stuck-agent-dog` has a hard-coded session-safety list.**
  Its "Scope" section enumerates exactly which tmux session
  types it may inspect (polecats, `hq-deacon`) and which it
  must never touch (crew, mayor, witness, refinery). Killing
  the wrong session is classified as a critical incident.
- **`rebuild-gt` has a post-incident safety gate.** Its
  preamble calls out a past crash loop caused by rebuilding
  backwards and mandates `gt stale --json` pre-flight with
  `safe_to_rebuild: true` before any action.
- **`tool-updater` has a hard-coded path** in its "Run" section:
  `cd /Users/jeremy/gt/plugins/tool-updater && bash run.sh` —
  this path is from one developer's machine and would not work
  on a fresh install. Documentation-only drift; `run.sh` itself
  uses relative paths.

## Related wiki pages

- [internal/plugin (package)](../packages/plugin.md) — the
  scanner + recorder + sync engine that loads these directories.
- [gt plugin (command)](../commands/plugin.md) — CLI surface
  (`list`, `show`, `run`, `sync`, `history`).
- [deacon (role)](../roles/deacon.md) — primary dispatcher;
  patrol loop evaluates gates and fires plugins.
- [deacon (package)](../packages/deacon.md) — the
  dispatcher code.
- [dog (role)](../roles/dog.md) — disposable worker that
  executes `FormatMailBody` output.
- [wisp (concept)](../concepts/wisp.md) — the ephemeral bead
  shape of plugin run records.
- [`gt doctor`](../commands/doctor.md) — runs the
  `PatrolPluginsAccessible` and `PatrolPluginDrift` health
  checks.

## Notes / open questions

- **No inventory of the sibling `dolt-snapshots-*` plugins.**
  The `dolt-snapshots/plugin.md` body promises three plugins
  sharing the same binary, but only one directory exists at
  `/home/kimberly/repos/gastown/plugins/`. Either the siblings
  are expected to be created at town-init time, or the
  documentation describes an aspirational state. Worth
  confirming.
- **No per-plugin entity pages for the other 13.** They are
  declarative shell + TOML and the inventory table is enough
  for current mapping purposes; future phases (drift analysis,
  quality review) may promote them to full pages.
- **`HasRunScript` detection is purely by filename.** A plugin
  author who names their script something other than `run.sh`
  will silently get agent-execution behavior. Worth confirming
  against the scanner code in a future lint pass.
