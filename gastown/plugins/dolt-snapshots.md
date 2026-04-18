---
title: dolt-snapshots plugin
type: file
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-15
sources:
  - /home/kimberly/repos/gastown/plugins/dolt-snapshots/plugin.md
  - /home/kimberly/repos/gastown/plugins/dolt-snapshots/main.go
  - /home/kimberly/repos/gastown/plugins/dolt-snapshots/main_test.go
  - /home/kimberly/repos/gastown/plugins/dolt-snapshots/run.sh
  - /home/kimberly/repos/gastown/plugins/dolt-snapshots/go.mod
tags: [plugin, dolt, snapshots, convoy, go-plugin, event-gate]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase8_audited: 2026-04-17
phase8_findings: [cross-ref-only]
detail_depth: {synthesis: 2, relationships: 2}
---

# dolt-snapshots plugin

The only gastown plugin with Go source code. Tags production Dolt
databases at convoy lifecycle boundaries so that audit
(`dolt_diff`), rollback (`DOLT_CHECKOUT`), and cross-convoy
comparison work against immutable references that always point
to the exact state when a convoy entered each stage. Implemented
as a standalone Go binary compiled on first run by the plugin's
`run.sh`, connected to the per-town Dolt MySQL server
(127.0.0.1:3307) via `github.com/go-sql-driver/mysql`, using
**parameterized SQL** rather than shell-interpolated `dolt sql
-q` commands. PR #2324 review drove the Go rewrite to fix three
specific problems in a prior bash implementation: shell
interpolation / SQL injection, subshell counter bugs, and
auto-commit of dirty state.

**Plugin directory:** `/home/kimberly/repos/gastown/plugins/dolt-snapshots/`
**File count:** 7 — `plugin.md`, `run.sh`, `main.go`,
`main_test.go`, `go.mod`, `go.sum`, plus the compiled
`dolt-snapshots` binary that `run.sh` builds on first run.
**Language:** Go 1.23; one dependency —
`github.com/go-sql-driver/mysql v1.9.3`
(`/home/kimberly/repos/gastown/plugins/dolt-snapshots/go.mod:1-7`).
**Invoked by:** the [Deacon](../roles/deacon.md) patrol via
[`gt plugin`](../commands/plugin.md) or — more importantly —
**directly by the binary's own `--watch` mode** tailing
`~/gt/.events.jsonl` for sub-second response to convoy events.
**Gate:** `type = "event"`, `on = "convoy.created"`
(`plugin.md:7-9`) — the only event-gated plugin in the gastown
inventory.

## What it actually does

### Frontmatter and dispatch shape

Source: `/home/kimberly/repos/gastown/plugins/dolt-snapshots/plugin.md:1-18`.

```toml
+++
name = "dolt-snapshots"
description = "Tag Dolt databases at convoy boundaries for audit, diff, and rollback"
version = 3

[gate]
type = "event"
on = "convoy.created"

[tracking]
labels = ["plugin:dolt-snapshots", "category:data-safety"]
digest = true

[execution]
timeout = "2m"
notify_on_failure = true
severity = "low"
+++
```

The body (`plugin.md:19-141`) is documentation + a `Step 1` shell
block that builds the binary, runs one catch-up cycle, and
launches the `--watch` daemon in the background with a PID file
at `$PLUGIN_DIR/.snapshot.pid`. Re-runs skip launching if the PID
is still live. Because `HasRunScript` is true (a `run.sh` exists
alongside the markdown), the scanner's
[`FormatMailBody`](../packages/plugin.md) would send a dog the
"run `bash run.sh` and don't interpret the markdown" branch — so
the Step 1 bash block in `plugin.md` is effectively reference
documentation. The actual execution path is `run.sh → go build →
exec binary`.

### Files

| file | lines | purpose |
|---|---:|---|
| `plugin.md` | 141 | frontmatter + documentation describing the three-plugin family and the SQL operations enabled |
| `run.sh` | 18 | build-if-stale shim: `go build -o dolt-snapshots .` then `exec "$BINARY" "$@"` |
| `main.go` | 665 | the Go binary (modes, SQL, snapshot logic, event watcher) |
| `main_test.go` | 418 | unit tests for `sanitizeName`, `sanitizeDBName`, `isSystemDB`, `loadRoutes`, `resolveDependencyDB`, `resolveHost/Port/RoutesFile`, `ConvoyRow_SnapshotLogic` |
| `go.mod` | 7 | Go 1.23 + `go-sql-driver/mysql` |
| `go.sum` | — | checksum file |

### Binary modes

Source: `/home/kimberly/repos/gastown/plugins/dolt-snapshots/main.go:49-109`.

The binary has **three modes** selected by flags:

1. **One-shot catch-up (default):** connect to Dolt, list
   non-system databases, load `routes.jsonl`, run
   `snapshotConvoys` once, print stats, exit. Used by the
   plugin's Step 1 bash block to catch anything missed while
   `--watch` was down.
2. **`--watch`** (`main.go:63-68`, implemented at `main.go:584-665`):
   tail `~/gt/.events.jsonl` via 100ms `time.Ticker`, parse each
   new JSON line, and on `convoy.created`, `convoy.staged`,
   `convoy.launched`, or `convoy.closed`, open a fresh DB
   connection and run one `snapshotConvoys` cycle. This is
   **the actual long-lived daemon** the plugin launches. The
   seek-to-end on startup (`main.go:597-600`) means it only
   sees NEW events — replay is handled by the one-shot
   catch-up that ran first.
3. **`--cleanup`** (`main.go:55-55`, `main.go:98-100`,
   implemented in `escalateStale` at `main.go:523-573`): walk
   every database's `dolt_branches`, find `convoy/*` branches
   whose backing convoy is `closed`/`landed` and older than 7
   days, print them. The comment at `main.go:569-571` is
   emphatic: **tags are never deleted** — only branches are
   proposed for cleanup; tags are immutable and cheap.

Flag resolution prefers explicit `--flag` values, then environment
variables (`DOLT_HOST`, `GT_DOLT_PORT` → `DOLT_PORT`,
`ROUTES_FILE`), then hard-coded defaults (`127.0.0.1`, `3307`,
`~/gt/.beads/routes.jsonl`). See `resolveHost` / `resolvePort` /
`resolveRoutesFile` at `main.go:111-143` — tested explicitly in
`main_test.go:282-357`.

### Connection and database filtering

The DSN is built at `main.go:70` / `main.go:606`:

```
root@tcp(<host>:<port>)/information_schema?parseTime=true&timeout=5s&readTimeout=30s&writeTimeout=30s
```

`SetMaxOpenConns(2)` and `SetConnMaxLifetime(time.Minute)` —
small pool, short lifetime.

`listDatabases` (`main.go:146-166`) calls `SHOW DATABASES` and
filters via `isSystemDB` (`main.go:168-179`), which rejects
`information_schema`, `mysql`, `dolt_cluster`, and any name
starting with `testdb_`, `beads_t`, `beads_pt`, or `doctest_`.
The test pollution prefixes match the `gt dolt cleanup` logic
in the project CLAUDE.md. Tested at `main_test.go:78-109` —
notably, edge cases confirm that `testdb` (no underscore) and
`beads_production` (doesn't match `beads_t` / `beads_pt`) are
**not** treated as system DBs.

### Routes file parsing

`loadRoutes` (`main.go:181-214`) reads JSONL lines of the form
`{"prefix":"pe-","path":"petals/mayor/rig"}` and produces a
`prefix → database-name` map. Key quirks:

- **`path="."` skips the entry** (`main.go:202-204`). This is
  why `hq-` and `hq-cv-` — which have `path="."` — aren't in
  the route map; HQ is always tagged explicitly (see
  `snapshotConvoys` below).
- **Database name is the first path component**
  (`main.go:206-211`): `petals/mayor/rig` → `petals`. Tested at
  `main_test.go:111-152`.
- **Malformed lines are silently skipped** (`main.go:198-201`),
  not fatal. Tested at `main_test.go:161-183`.
- **Missing file is non-fatal** — logs a warning and returns an
  empty map. Tested at `main_test.go:154-159`.
- **Last duplicate prefix wins** (tested at `main_test.go:205-221`).

### Convoy discovery and the snapshot cycle

`snapshotConvoys` (`main.go:243-336`) is the main workhorse.

**Step 1 — find convoys needing tags.** `findConvoysNeedingSnapshots`
(`main.go:339-377`) runs this SQL (paraphrased):

```sql
SELECT i.id, i.title, i.status,
  EXISTS(SELECT 1 FROM hq.dolt_tags
         WHERE tag_name LIKE CONCAT('open/%-', i.id)) AS has_open_tag,
  EXISTS(SELECT 1 FROM hq.dolt_tags
         WHERE tag_name LIKE CONCAT('staged/%-', i.id)) AS has_staged_tag
FROM hq.issues i
WHERE i.issue_type = 'convoy'
  AND (i.status IN ('staged_ready','staged_warnings','launched','open')
       OR (i.status = 'closed' AND i.updated_at >= NOW() - INTERVAL 24 HOUR))
  AND EXISTS(SELECT 1 FROM hq.dependencies d
             WHERE d.issue_id = i.id AND d.type = 'tracks')
HAVING has_open_tag = 0 OR has_staged_tag = 0
```

Only convoys with at least one `tracks` dependency are
considered — a convoy that has never staged anything has
nothing to snapshot against.

**Step 2 — discover which rig databases this convoy touches.**
`discoverConvoyDatabases` (`main.go:381-419`) queries
`hq.dependencies` for `depends_on_id` values (`type='tracks'`)
and resolves each via `resolveDependencyDB`
(`main.go:425-442`). Two ID formats are handled:

- `external:<rig_name>:<bead_id>` → `rig_name` (tested at
  `main_test.go:235-237`).
- `<prefix>-<id>` → `routes[prefix]` (tested at
  `main_test.go:240-244`).
- **Unknown prefixes return empty string** — the db is skipped.

Discovered rig DBs are verified to exist on the server before
inclusion (`main.go:405-411`) — exact match, not substring.
`hq` is **always** added to the set (`main.go:277-278`)
regardless of what the convoy tracks, because the convoy
metadata itself lives in `hq` and needs to be snapshotted.

**Step 3 — create tags and branches.**

For each convoy-database pair:

- **`open/<safe-name>` tag**, created if `!c.HasOpenTag`
  (`main.go:288-304`). Every convoy gets this — it's the
  pre-work baseline.
- **`staged/<safe-name>` tag AND `convoy/<safe-name>` branch**,
  created if `!c.HasStagedTag && c.Status != "open"`
  (`main.go:307-332`). The status guard means open convoys
  don't get a staged tag even if the column is missing —
  enforced in the Go logic, not the SQL. This is tested at
  `main_test.go:359-418` as the `TestConvoyRow_SnapshotLogic`
  truth table.

`safeName` is produced by `sanitizeName` (`main.go:217-234`) —
lowercase, non-alphanumeric → `-`, multiple dashes collapsed,
trimmed, truncated to 50 char slug, suffixed with `-<convoy-id>`.
Example from the tests: `"Pi Rust Bug Fixes"` + `"hq-cv-xrwki"`
→ `"pi-rust-bug-fixes-hq-cv-xrwki"` (`main_test.go:15`).

### Tag and branch creation — per-connection race-safety

`createTag` (`main.go:446-475`) and `createBranch`
(`main.go:479-508`) share an important design constraint.

**The shared pool cannot be trusted for `USE <db>; CALL DOLT_TAG/BRANCH(...)`
sequences** because the second statement may run on a different
connection where `USE` never happened. The fix (see comments at
`main.go:458-459` and `main.go:491-492`) is to **acquire a
dedicated connection from the pool** via `db.Conn(ctx)`, run
both statements on that same `conn`, and close it on return.

The flow:

1. **Existence check** via a backtick-quoted `FROM \`<db>\`.dolt_tags`
   query — idempotent short-circuit if the tag/branch already
   exists.
2. **`USE \`<db>\``** on a dedicated connection.
3. **`CALL DOLT_TAG(?, 'HEAD', '-m', ?)`** or
   **`CALL DOLT_BRANCH(?, 'HEAD')`** — **parameterized** — so
   the tag name and message can contain arbitrary characters
   without shell quoting bugs.

The database name is pre-sanitized via `sanitizeDBName`
(`main.go:512-520`), which strips everything except `[a-zA-Z0-9_]`.
This is defensive even though DB names come from an internal
`SHOW DATABASES` result — tested against Bobby Tables style
injection at `main_test.go:53-76`.

### Event watcher — sub-second response to convoy lifecycle

`watchEvents` (`main.go:584-665`) is the reason this plugin has
an `event` gate. Deacon patrol polling runs every ~60s; for
`convoy.launched` where agents begin writing to databases
immediately, that's way too slow — by the time the patrol
cycle fires, the staging baseline is already contaminated.

The watcher:

1. Opens `<home>/gt/.events.jsonl`, seeks to end
   (`main.go:597-600`) — ignores the backlog.
2. `time.NewTicker(100 * time.Millisecond)` polls for new lines
   (`main.go:602-603`).
3. For each line, attempts to JSON-unmarshal as `convoyEvent`;
   silently skips malformed lines (`main.go:617-620`).
4. On `convoy.created` | `convoy.staged` | `convoy.launched` |
   `convoy.closed` events (`main.go:622-661`), opens a fresh
   DB connection (a new `sql.Open` per cycle — not long-lived),
   pings, lists databases, re-loads routes, runs
   `snapshotConvoys`, optionally `escalateStale`, and closes the
   connection.

The 100ms ticker + fresh-per-event connections give low latency
without holding idle pool connections against Dolt when the
town is quiet.

### Stats and output

`snapshotStats` (`main.go:236-240`): `tagsCreated`,
`branchesCreated`, `tagsFailed`. Printed at end of run
(`main.go:102-108`). Dry-run mode (`--dry-run`) logs what
**would** happen without executing, and the stats reflect the
no-op.

## Config / schedule

From the frontmatter:

- **Gate:** `type = "event"`, `on = "convoy.created"`.
  **Note:** despite the `on` being singular, the `watchEvents`
  switch statement (`main.go:622-623`) fires on all four
  lifecycle events. The gate is effectively "any convoy event"
  once the watcher is running — the gate only controls when
  the plugin is **dispatched**, not what the running binary
  reacts to.
- **Tracking labels:** `plugin:dolt-snapshots`,
  `category:data-safety`. Daily digest inclusion on.
- **Execution:** 2-minute timeout, notify on failure,
  `severity = "low"`. The binary's long-lived `--watch` mode is
  backgrounded via `nohup ... &` in the `plugin.md` Step 1
  block and survives the 2-minute timeout (the timeout applies
  to the dispatch command, not to the backgrounded child).
- **Environment variables consumed:** `DOLT_HOST`,
  `GT_DOLT_PORT` (preferred) / `DOLT_PORT`, `ROUTES_FILE`.

## Companion plugins (documented, not present in inventory)

The `plugin.md` body describes **three plugins sharing this
binary**:

| plugin | event | snapshot |
|---|---|---|
| `dolt-snapshots` | `convoy.created` | `open/` tags (pre-work baseline) |
| `dolt-snapshots-staged` | `convoy.staged` | `staged/` tags + branches (staging baseline) |
| `dolt-snapshots-launched` | `convoy.launched` | `staged/` tags + branches (launch baseline) |

Only `dolt-snapshots/` exists as a directory in
`/home/kimberly/repos/gastown/plugins/`. The sibling directories
are **not** present in the inventory — they may be
runtime-only (created at town init) or aspirational
documentation. This is called out in the
[plugin inventory](README.md#neutral-observations) as an open
question.

## Related wiki pages

- [plugin inventory](README.md) — all 14 gastown plugins.
- [internal/plugin (package)](../packages/plugin.md) —
  scanner, recorder, sync; explains how `run.sh` detection
  drives execution behavior.
- [gt plugin (command)](../commands/plugin.md) — CLI surface
  that can also invoke this plugin manually.
- [deacon (role)](../roles/deacon.md) — the agent that
  dispatches the plugin on `convoy.created` events.
- [deacon (package)](../packages/deacon.md) — dispatch
  machinery.
- [doltserver (package)](../packages/doltserver.md) — the
  per-town Dolt MySQL server this binary connects to at
  127.0.0.1:3307.
- [events (package)](../packages/events.md) — the writer of
  the `.events.jsonl` file the `--watch` mode tails;
  `convoy.created` / `staged` / `launched` / `closed` are four
  of its 21 event types.
- [convoy (concept)](../concepts/convoy.md) — the lifecycle
  this plugin takes snapshots of.
- [convoy-launch (workflow)](../workflows/convoy-launch.md) —
  the broader flow that fires these events.

## Notes / open questions

- **Sibling plugin directories are missing from the inventory.**
  `plugin.md` documents `dolt-snapshots-staged` and
  `dolt-snapshots-launched` but neither exists at
  `/home/kimberly/repos/gastown/plugins/`. Worth confirming
  whether they're expected at town-init time or the docs are
  aspirational.
- **The `--watch` mode long-lived daemon is launched by a shell
  block inside the markdown body, not `run.sh`.** Since
  `HasRunScript=true`, the scanner's `FormatMailBody` tells
  dogs NOT to interpret the markdown body — which means the
  Step 1 block (the one that actually starts the watcher) is
  reference-only when a dog dispatches. **The watcher is only
  launched via the bash block if the plugin body IS being
  interpreted** — i.e., if this plugin is somehow dispatched in
  agent mode rather than script mode, or if a human runs the
  block manually. `run.sh` itself only runs the binary in
  one-shot or `--watch` mode depending on args passed. Worth
  tracing how the watcher actually gets its initial launch in
  practice.
- **PID file leak is possible.** The shell block writes
  `$PLUGIN_DIR/.snapshot.pid` and guards re-runs with
  `kill -0`, but `kill -0` on a stale PID that's been recycled
  to an unrelated process will incorrectly skip the launch.
  No PID-to-process-name verification.
- **`escalateStale` only prints — it doesn't actually escalate.**
  Despite the name, `main.go:560-572` just prints the stale
  branches and a note that "tags are never deleted." No `gt
  escalate` call is made from the Go binary. The name is
  aspirational.
- **No tests for `snapshotConvoys`, `createTag`, `createBranch`,
  or `watchEvents`.** All DB-touching code is untested; tests
  are limited to pure-function helpers (sanitizers, routes
  parsing, dependency resolution, flag/env resolution, and the
  truth-table `ConvoyRow_SnapshotLogic`).
