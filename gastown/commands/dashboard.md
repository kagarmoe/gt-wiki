---
title: gt dashboard
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/cmd/dashboard.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, diagnostics, web, dashboard, dolt, http-server]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
phase5_audience: dev
phase8_audited: 2026-04-17
phase8_findings: [precondition, silent-suppression]
---

# gt dashboard

Starts a blocking HTTP server that serves the Gas Town convoy tracking
web dashboard (or a setup-mode UI when run outside a workspace).

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupDiag` ("Diagnostics")
**Polecat-safe:** no (no `AnnotationPolecatSafe` on the cobra.Command at
`/home/kimberly/repos/gastown/internal/cmd/dashboard.go:27-45`)
**Beads-exempt:** no (not in `beadsExemptCommands`)
**Branch-check-exempt:** no (not in `branchCheckExemptCommands`)

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/dashboard.go:27-194`.

### Invocation

```
gt dashboard [--port N] [--bind ADDR] [--open]
```

Single terminal command (no subcommands). Runs `runDashboard`
(`dashboard.go:58-154`) which calls
`http.Server.ListenAndServe()` and blocks until killed.

### Behavior

`runDashboard` (`dashboard.go:58-154`):

1. **Detects workspace vs setup mode** (`dashboard.go:63-95`):
   - If `workspace.FindFromCwdOrError` fails, builds the handler via
     `web.NewSetupMux()` — the setup wizard UI for users without a
     workspace yet.
   - Otherwise, builds the handler via `web.NewDashboardMux(fetcher, webCfg)`.
2. **Forces Dolt port env vars** (`dashboard.go:76`, implemented at
   `ensureDoltPortEnv`, `dashboard.go:161-177`): reads
   `daemon/dolt-state.json` via `doltserver.LoadState`; falls back to
   `doltserver.DefaultPort`. Sets `GT_DOLT_PORT`, `BEADS_DOLT_PORT`, and
   `BEADS_DOLT_SERVER_HOST` so child `bd` subprocesses connect to the
   real Dolt server, not the dashboard's HTTP port. The comment at
   `dashboard.go:73-76` explains the bug being prevented: inherited env
   vars could point `bd` at the dashboard listen port.
3. **Creates a live convoy fetcher** via `web.NewLiveConvoyFetcher()`
   (`dashboard.go:78-81`).
4. **Loads web timeouts config** via
   `config.LoadOrCreateTownSettings(config.TownSettingsPath(townRoot))`
   (`dashboard.go:83-89`). Loads `TownSettings.WebTimeouts`.
   `NewDashboardMux` applies defaults when nil.
5. **Builds listen address and display URL**
   (`dashboard.go:97-107`): `<bind>:<port>`. When `bind == "0.0.0.0"`,
   display host becomes `os.Hostname()` (or `"localhost"` on failure).
6. **Opens the browser** in a goroutine when `--open` is set
   (`dashboard.go:109-112`) via `openBrowser` (`dashboard.go:180-193`),
   which shells out to `open` (macOS), `xdg-open` (Linux), or
   `cmd /c start` (Windows).
7. **Prints a banner** — a large ASCII "WELCOME TO GASTOWN" banner when
   `term.GetSize` reports at least 98 cols, otherwise a short fallback
   (`dashboard.go:114-142`).
8. **Starts the HTTP server** (`dashboard.go:145-153`) with fixed
   timeouts: `ReadHeaderTimeout=10s`, `ReadTimeout=30s`,
   `WriteTimeout=60s`, `IdleTimeout=120s`.

The server runs until killed with Ctrl+C or a signal.

### Sandbox auto-binding

`dashboard.go:49-52`: when the `IS_SANDBOX` env var is set, the default
bind address switches from `127.0.0.1` to `0.0.0.0` so the dashboard is
reachable from the host running the sandbox.

### Subcommands

None (terminal command).

### Flags

`dashboard.go:48-54`:

| Flag      | Type   | Default                                  | Purpose                       |
|-----------|--------|------------------------------------------|-------------------------------|
| `--port`  | int    | `8080`                                   | HTTP port to listen on        |
| `--bind`  | string | `127.0.0.1` (or `0.0.0.0` in sandbox)    | Listen address                |
| `--open`  | bool   | `false`                                  | Open browser automatically    |

## Related

- [internal/daemon package](../packages/daemon.md) — `daemon/dolt-state.json` + the heartbeat counter surfaced by this dashboard are written by the daemon main loop
- [../packages/web.md](../packages/web.md) — the dashboard HTTP layer
  itself: embedded templates, CSRF API, setup mode.
- [feed](feed.md) — the TUI equivalent of the dashboard; both expose
  convoy status and event streams but from different UI surfaces.
- [../binaries/gt.md](../binaries/gt.md) — parent binary.
- [README.md](README.md) — command tree index.

## Failure modes

### Precondition violations (what does it assume?)
- **Port availability:** `ListenAndServe` at `dashboard.go:153`
  assumes port 8080 (or `--port`) is available. If another process
  holds the port, the error message is Go's raw `bind: address already
  in use` with no retry or suggestion. **Absent** — no pre-check or
  helpful error message.

### Silent suppression (what errors are swallowed?)
- **Browser open failure:** `openBrowser` at `dashboard.go:192`
  discards the error from `cmd.Start()` with `_ = cmd.Start()`. If
  `xdg-open`/`open` is missing or fails, no error reaches the user.
  **Absent** — fire-and-forget.
- **Town settings load failure:** `config.LoadOrCreateTownSettings`
  error at `dashboard.go:86-89` is printed as a warning to stderr but
  execution continues with nil config (defaults applied by
  `NewDashboardMux`). **Present** — warned but non-fatal.

## Notes / open questions

- The dashboard depends on a reachable Dolt server — the env-var
  propagation in `ensureDoltPortEnv` is defensive against a specific
  bug (dashboard listen port vs Dolt port confusion), not a substitute
  for the server actually running.
- The ASCII banner is gated on terminal width ≥ 98 cols; headless runs
  (via `nohup`, systemd, etc.) fall through to the short banner because
  `term.GetSize` returns an error.
- Handler construction is all in `internal/web/` — not yet documented
  in the wiki. `web.NewDashboardMux`, `web.NewSetupMux`,
  `web.NewLiveConvoyFetcher`.
- No graceful shutdown — `ListenAndServe` blocks and returns only on
  fatal errors; SIGINT kills the process cleanly but there is no
  in-flight request drain.
