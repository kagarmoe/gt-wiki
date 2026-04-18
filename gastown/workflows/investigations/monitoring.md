---
title: "Investigating: monitoring and visibility gaps"
type: investigation
status: verified
topic: gastown
created: 2026-04-17
updated: 2026-04-17
sources:
  - gastown/commands/dashboard.md
  - gastown/commands/agents.md
  - gastown/commands/status.md
  - gastown/commands/feed.md
  - gastown/packages/web.md
  - /home/kimberly/repos/gastown/internal/cmd/dashboard.go
  - /home/kimberly/repos/gastown/internal/cmd/agents.go
  - /home/kimberly/repos/gastown/internal/cmd/status.go
  - /home/kimberly/repos/gastown/internal/cmd/feed.go
  - /home/kimberly/repos/gastown/internal/web/handler.go
  - /home/kimberly/repos/gastown/internal/web/fetcher.go
tags: [investigation, monitoring, diagnostic, dashboard, status, agents, feed]
---

# Investigating: monitoring and visibility gaps

**Symptom:** `gt status` shows stale or misleading data. `gt agents`
is missing agents or showing dead ones as alive. `gt dashboard`
won't start or shows empty panels. `gt feed` shows no events or
crashes. Operators can't tell what's happening in the town.

Gas Town has four monitoring surfaces:

- **`gt status`** — CLI status report: daemon, Dolt, agents, rigs,
  hooks, merge queue. See [gt status](../../commands/status.md).
- **`gt agents`** — agent session listing with tmux popup menu. See
  [gt agents](../../commands/agents.md).
- **`gt dashboard`** — web UI for convoy tracking and system overview.
  See [gt dashboard](../../commands/dashboard.md) and
  [web](../../packages/web.md).
- **`gt feed`** — event stream (Bubble Tea TUI or plain text). See
  [gt feed](../../commands/feed.md).

## Decision tree

### 1. Which monitoring surface is broken?

- **`gt status` shows wrong data** ->
  [Step 2: Status data accuracy](#2-status-data-accuracy)
- **`gt agents` missing or showing wrong agents** ->
  [Step 4: Agent listing issues](#4-agent-listing-issues)
- **`gt dashboard` won't start** ->
  [Step 6: Dashboard startup failures](#6-dashboard-startup-failures)
- **`gt dashboard` shows empty or stale data** ->
  [Step 7: Dashboard data issues](#7-dashboard-data-issues)
- **`gt feed` shows no events** ->
  [Step 8: Feed event stream issues](#8-feed-event-stream-issues)

### 2. Status data accuracy

`gatherStatus` at `status.go:607-948` collects data from multiple
sources. Each source can fail independently.

**a. Zombie sessions shown as "running":**
`status.go:645-664` pre-fetches all tmux sessions and fans out
`IsAgentAlive` checks in parallel. An agent is "running" only if the
process is alive inside the session, not just because tmux has the
session. This was fixed in gt-bd6i3. If you still see zombies as
"running," the process-name resolution may be wrong.

**Check:** `gt agents check --json` — identity collision report. Also:

```bash
tmux list-panes -t <session> -F '#{pane_pid} #{pane_current_command}'
```

Compare the pane process against what `GT_PROCESS_NAMES` expects (set
at agent startup). If the process name doesn't match, `IsAgentAlive`
returns false but the status check may have already cached a stale
result.

**b. Agent beads data stale or missing (Dolt down):**
`status.go:672-761` pre-fetches agent beads from Dolt in parallel.
If Dolt is unreachable, the bead fetch fails silently — agents appear
without their bead metadata (hook state, work assignment).

**Check:** Is Dolt reachable? Follow the
[data-plane investigation](data-plane.md) starting at Step 1.

**c. Rig discovery failure:**
`status.go:667-670` discovers rigs via `rig.Manager.DiscoverRigs`.
If `mayor/rigs.json` is corrupt or missing, no rigs appear in the
status output.

**Check:** `cat ~/gt/mayor/rigs.json` — is it valid JSON?

### 3. Status `--fast` mode differences

`--fast` skips slower data collection (bead fetches, queue summaries)
and shows only tmux-based state. If `--fast` shows correct data but
normal mode doesn't, the problem is in beads/Dolt, not in tmux.

### 4. Agent listing issues

`gt agents list` at `agents.go:540-580` and the parent `runAgentsList`
scan all tmux sockets for Gas Town sessions.

**a. Polecats hidden by default:**
Polecats are hidden from `gt agents list` by default. Use `--all` /
`-a` to include them. This is a persistent flag on the parent
`agentsCmd` at `agents.go:140`, inherited by all subcommands.

**b. Boot excluded from listing:**
`agents.go:321` (referenced in
[gt boot](../../commands/boot.md)) filters out the Boot session by
name (`session.BootSessionName()`). Boot is intentionally excluded —
it's a short-lived triage agent, not a display agent.

**c. Overseer excluded:**
`categorizeSession` at `agents.go:151-182` explicitly excludes
`RoleOverseer` because "overseer is the human operator, not a display
agent."

**d. Session on wrong tmux socket:**
Gas Town uses a per-town tmux socket (name derived from a hash of the
town's canonical path). Sessions on the default tmux socket or on
other towns' sockets won't appear in `gt agents list`.

**Check:** What socket is the town using?

```bash
gt town socket-name  # Shows the socket name
tmux -L <socket-name> list-sessions  # Verify sessions are on this socket
```

**e. Identity parsing failure:**
`categorizeSession` calls `session.ParseSessionName` to identify the
agent type. Sessions that don't parse as a known Gas Town role
(Mayor, Deacon, Witness, Refinery, Crew, Polecat) are silently
dropped. Non-Gas-Town tmux sessions on the same socket will be
ignored.

### 5. Agent identity collisions

`gt agents check` at `agents.go:103-117` and `gt agents fix` at
`agents.go:119-132` detect and clean stale worker locks.

**a. Stale locks:**
Agent identity locks (see [lock](../../packages/lock.md)) use tmux-
aware stale detection. If an agent dies without releasing its lock,
the lock becomes stale. `gt agents fix` cleans these.

**b. True collisions:**
Two agents with the same identity (e.g., same polecat name on two
rigs that share a prefix) create a genuine collision. `gt agents check`
reports these; fixing them requires renaming or reconfiguring the
conflicting agents.

### 6. Dashboard startup failures

`runDashboard` at `dashboard.go:58-154`.

**a. Port already in use:**
The HTTP server binds to `--port` (default 8080) on `--bind`
(default `127.0.0.1`, or `0.0.0.0` when `IS_SANDBOX` is set). If
the port is held by another process, `ListenAndServe` fails.

**Fix:** `gt dashboard --port <different-port>` or kill the holder.

**b. Not in a workspace (setup mode):**
If `workspace.FindFromCwdOrError` fails, the dashboard falls back
to `web.NewSetupMux()` — the setup wizard UI. This is by design,
not an error. If you expected the full dashboard, you're not inside
a Gas Town workspace.

**c. Dolt port env var conflict:**
`ensureDoltPortEnv` at `dashboard.go:161-177` reads
`daemon/dolt-state.json` to find the real Dolt port, then sets
`GT_DOLT_PORT`, `BEADS_DOLT_PORT`, and `BEADS_DOLT_SERVER_HOST`. The
comment at `dashboard.go:73-76` explains: inherited env vars could
point `bd` at the dashboard's HTTP port instead of Dolt's SQL port.

**Check:** If dashboard data is wrong, verify the Dolt port env vars
are pointing at the right place:

```bash
echo $GT_DOLT_PORT $BEADS_DOLT_PORT
```

### 7. Dashboard data issues

The dashboard uses `web.NewLiveConvoyFetcher()` to poll data and
`web.NewDashboardMux(fetcher, webCfg)` to serve it.

**a. Empty convoy panel:**
The convoy fetcher queries beads for convoy data. If Dolt is down or
the beads database has no convoys, the panel is empty. See
[data-plane](data-plane.md) for Dolt issues.

**b. Stale data (no auto-refresh):**
The dashboard serves data at request time. If the browser tab is
stale, data is stale. Refresh the page.

**c. Web timeouts too aggressive:**
`config.LoadOrCreateTownSettings` loads `TownSettings.WebTimeouts`.
If timeouts are too short, fetcher queries may be killed before
completing.

**Check:** Review town settings:

```bash
cat ~/gt/mayor/settings.json | grep -A5 web_timeouts
```

**d. Template rendering failure:**
`web.LoadTemplates()` parses embedded `templates/*.html`. Since
templates are embedded at build time, rendering failures indicate
either a corrupt binary or a data shape mismatch between the fetcher
and the template's expected `ConvoyData` struct.

### 8. Feed event stream issues

`runFeed` at `feed.go:110-160` dispatches to TUI, plain, or
tmux-window mode.

**a. No event sources created:**
`feed.go:260-262` errors out only if ALL sources failed to create.
Sources are optional:
- `feed.NewBdActivitySource(workDir)` — `bd activity` stream.
  Requires beads/Dolt. Optional failure at `feed.go:243-246`.
- `feed.NewMQEventSourceFromWorkDir(workDir)` — merge queue events.
  Optional failure at `feed.go:249-252`.
- `feed.NewGtEventsSource(townRoot)` — `.events.jsonl` stream.
  Optional failure at `feed.go:255-258`.

If all three fail, you see "no event sources available."

**Check:** Which sources are failing?
- `bd activity` — is Dolt reachable? See
  [data-plane](data-plane.md).
- `.events.jsonl` — does the file exist at `<townRoot>/.events.jsonl`?
  Is it readable?

**b. Plain mode shows nothing:**
`runFeedDirect` at `feed.go:209-230` reads `.events.jsonl` directly.
If the file is empty or doesn't exist, no events appear.

**Check:** `wc -l ~/gt/.events.jsonl` — is the file populated? Agent
lifecycle events are written by `events.LogFeed` calls scattered
across the codebase (e.g., `up.go:426`, `down.go:464` — both use
`_ =` fire-and-forget, so events may be silently lost).

**c. TUI crashes:**
The Bubble Tea TUI at `feed.go:282-286` runs with `tea.WithAltScreen()`.
If the terminal doesn't support alt-screen, or if a source emits
malformed data, the TUI may crash.

**Fix:** Use `--plain` to bypass the TUI and read events directly.

**d. Tmux window mode fails:**
`runFeedInWindow` at `feed.go:291-352` requires being inside a tmux
session (`tmux.IsInsideTmux`). If you're not in tmux, window mode
fails.

**Fix:** Use TUI mode (default) or plain mode instead.

**e. Rig filter produces no data:**
`--rig NAME` at `feed.go:138-154` rewrites the working directory to
the rig's beads path. If the rig name is wrong or the rig has no
`.beads` directory, the filter produces an error or empty results.

**Check:** `gt rig list` — does the rig exist? Does it have a
`.beads` directory?

### 9. Cross-monitoring gaps

**a. Beads-exempt commands can't report bead state:**
`gt status` and `gt feed` are both beads-exempt (they must work when
Dolt is down). This means they degrade gracefully: status shows
tmux-only state, feed shows only `.events.jsonl`. But the degraded
output omits work assignments, hook state, and merge queue data.

**b. `gt status --watch` refresh gaps:**
Watch mode re-runs `gatherStatus` on each tick. Between ticks, state
can change. The tick interval is controlled by `-n` (default varies).
Short intervals increase Dolt load; long intervals miss transient
failures.

**c. Agent-reported state vs tmux-observed state:**
`gt status` uses `IsAgentAlive` (tmux process tree) for running
state, NOT agent self-reported heartbeats. An agent that is alive but
not doing useful work (stuck in a loop, waiting for input) appears
"running." The Deacon's `health-check` command at
`deacon.go:838+` uses a different mechanism (immediate nudge +
response timeout) that is more reliable for detecting stuck agents.

## Related investigation workflows

- [Investigating: data-plane failures](data-plane.md) -- Dolt
  issues that cause empty or stale monitoring data.
- [Investigating: agent lifecycle](agent-lifecycle.md) -- agent
  start/stop failures visible through monitoring surfaces.
- [Investigating: daemon infrastructure](daemon-infrastructure.md) --
  daemon/deacon issues that affect the supervision chain visible
  in status output.
