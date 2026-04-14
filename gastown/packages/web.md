---
title: internal/web
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/web/handler.go
  - /home/kimberly/repos/gastown/internal/web/templates.go
  - /home/kimberly/repos/gastown/internal/web/fetcher.go
  - /home/kimberly/repos/gastown/internal/web/api.go
  - /home/kimberly/repos/gastown/internal/web/commands.go
  - /home/kimberly/repos/gastown/internal/web/setup.go
  - /home/kimberly/repos/gastown/internal/web/validate.go
tags: [package, web, http, dashboard, ui, templates]
---

# internal/web

HTTP server, templates, data-fetchers, REST API, and CSRF-protected
command runner for the Gas Town dashboard. Produces a single `http.Handler`
(from `NewDashboardMux` or `NewSetupMux`) that serves a live view of
convoys, workers, mail, rigs, issues, merge queue, escalations, and
activity — plus a `/api/*` surface for running whitelisted gt
commands, browsing mail, CRUDing issues, and fetching option lists.
Everything — templates, static assets — is embedded into the binary.

**Go package path:** `github.com/steveyegge/gastown/internal/web`
**File count:** 7 go files, 7 test files, plus `static/` and
`templates/` subdirectories embedded at build time.
**Imports (notable):** stdlib (`embed`, `html/template`, `net/http`,
`crypto/rand`, `os/exec`, `context`, `sync`, `time`), plus
`internal/activity` for colored activity info (no wiki page yet),
[`internal/beads`](beads.md) (for `InjectFlatForListJSON`),
[`internal/config`](config.md) (for `WebTimeoutsConfig`,
`WorkerStatusConfig`, `TownSettings`),
[`internal/session`](session.md) for prefix registry parsing,
[`internal/tmux`](tmux.md) for session env lookup,
[`internal/workspace`](workspace.md) for town-root resolution,
`internal/constants` for shared timeouts (no wiki page yet).
**Imported by (notable):** [`gt dashboard`](../commands/dashboard.md) —
the only production importer. `gt dashboard start` constructs a
`LiveConvoyFetcher`, calls `NewDashboardMux(fetcher, webCfg)`, and
binds the returned handler to an HTTP listener.
[`internal/keepalive`](keepalive.md) wraps the underlying
`http.Server` lifecycle.

## What it actually does

### Data model (`templates.go`)

- `ConvoyData` (`templates.go:17-35`) — the single `html/template`
  data bundle that the dashboard root template receives. Carries
  slices of `ConvoyRow`, `MergeQueueRow`, `WorkerRow`, `MailRow`,
  `RigRow`, `DogRow`, `EscalationRow`, `SessionRow`, `HookRow`,
  `IssueRow`, `ActivityRow`, plus an `*HealthRow`, `*MayorStatus`,
  `*DashboardSummary`, a `Queues` slice, and an `Expand`
  fullscreen-panel name and a `CSRFToken`.
- Row structs (`templates.go:38-221`) — one per dashboard panel.
  Each carries only presentation-ready strings (ages, color-class
  hints, pre-formatted timestamps) so the template files can stay
  pure rendering.
- `DashboardSummary` (`templates.go:140-157`) — the at-a-glance
  alert counts (`PolecatCount`, `HookCount`, `StuckPolecats`,
  `StaleHooks`, `UnackedEscalations`, `DeadSessions`,
  `HighPriorityIssues`, `HasAlerts`).
- `LoadTemplates()` (`templates.go:224-...`) — parses the embedded
  `templates/*.html` with a `FuncMap` of class-naming helpers
  (`activityClass`, `statusClass`, `workStatusClass`,
  `senderColorClass`, `severityClass`, `dogStateClass`,
  `queueStatusClass`, `polecatStatusClass`, `activityTypeClass`)
  plus a `contains` string helper.

### Dashboard handler (`handler.go`)

- `ConvoyFetcher` interface (`handler.go:23-38`) — the 14 methods
  that supply dashboard data: `FetchConvoys`, `FetchMergeQueue`,
  `FetchWorkers`, `FetchMail`, `FetchRigs`, `FetchDogs`,
  `FetchEscalations`, `FetchHealth`, `FetchQueues`,
  `FetchSessions`, `FetchHooks`, `FetchMayor`, `FetchIssues`,
  `FetchActivity`. This interface exists so tests can inject
  canned data without invoking `bd` subprocesses.
- `ConvoyHandler` (`handler.go:46-67`) — holds the fetcher, parsed
  templates, fetch timeout, CSRF token, and two response caches:
  one for the main page body and one keyed by `?expand=` panel
  name. The caches prevent "bd process storms" where multiple
  browser tabs or htmx auto-refresh requests would otherwise
  trigger overlapping fetches.
- `defaultCacheTTL = 10 * time.Second` (`handler.go:71`) — the
  minimum interval between full dashboard fetches.
- `NewConvoyHandler(fetcher, fetchTimeout, csrfToken)`
  (`handler.go:74-87`) — constructor; errors out if templates fail
  to parse.
- `ServeHTTP` (`handler.go:93-...`) — checks the cache, optionally
  serializes concurrent fetches via `cacheInUse`, fetches all
  panels under a single per-request timeout context, and renders
  the template. `?expand=<panel>` renders a fullscreen variant
  whose cache is keyed separately.
- `NewDashboardMux(fetcher, webCfg)` (`handler.go:459-489`) — the
  top-level builder. Generates a random 32-byte hex CSRF token,
  builds `ConvoyHandler` and `APIHandler` with derived timeouts,
  mounts `/api/` → api, `/static/` → embedded static files,
  `/` → dashboard.
- `generateCSRFToken()` (`handler.go:449-455`) —
  `crypto/rand` + hex encode. `log.Fatalf`s if randomness is
  unavailable.

### Data fetcher (`fetcher.go`)

- `runCmd(timeout, name, args...)` (`fetcher.go:31-46`) — the
  subprocess shell. Returns empty buffer on timeout or error.
  **Security-critical**: errors from this function are logged
  server-side only and never placed in HTTP responses (fetches
  that fail render as empty panels).
- `fetcherRunCmd` / `fetcherGetSessionEnv` (`fetcher.go:48-51`) —
  function-value hooks so tests can replace the subprocess and
  tmux surfaces.
- `fetchCircuitBreaker` (`fetcher.go:86-127`) — per-fetch-operation
  exponential-backoff tracker. `allow()` returns false until the
  backoff interval has elapsed; `recordFailure` doubles the
  backoff (`10s, 20s, 40s, 80s, 160s`, capped at `maxBackoff = 5
  * time.Minute`); `recordSuccess` resets. Currently only
  `FetchConvoys` uses it.
- `LiveConvoyFetcher` (`fetcher.go:130-157`) — the production
  implementation of `ConvoyFetcher`. Carries townRoot, town beads
  dir, optional `bdBin` override, a locally-built
  `session.PrefixRegistry`, and a grab-bag of configurable
  timeouts and worker-status thresholds loaded from
  `TownSettings`. Includes a `convoyBreaker` field.
- `NewLiveConvoyFetcher()` (`fetcher.go:161-201`) — resolves the
  town root, loads `TownSettings` (falling back to
  defaults on error), builds a prefix registry via
  `session.BuildPrefixRegistryFromTown`, and returns a populated
  fetcher.
- Per-panel fetch methods (`fetcher.go:206-1800+`) — one per
  `ConvoyFetcher` interface method. Most shell out to
  `bd list --flat --json --type=<X>`, `gh pr list --json`, or
  `tmux list-sessions` and marshal the output into the
  presentation row structs. `runBdCmd` (`fetcher.go:54-82`) is
  the shared bd subprocess path; it injects `--flat` via
  `beads.InjectFlatForListJSON` for bd v0.59+ compatibility.

### REST API (`api.go`)

- `CommandRequest` / `CommandResponse` / `CommandListResponse`
  (`api.go:23-45`) — shared JSON envelopes for `/api/run` and
  `/api/commands`.
- `APIHandler` (`api.go:48-64`) — holds a `gtPath` (always `"gt"`
  from PATH — test binaries would cause fork bombs), a workDir,
  the default and max run timeouts, an options cache with
  `optionsCacheTTL = 30 * time.Second` (`api.go:66`), a
  `cmdSem chan struct{}` semaphore capped at `maxConcurrentCommands
  = 12` (`api.go:70`) to prevent resource exhaustion, and a
  required `csrfToken`.
- `NewAPIHandler(defaultRunTimeout, maxRunTimeout, csrfToken)`
  (`api.go:73-88`) — warns if `csrfToken` is empty.
- `ServeHTTP` (`api.go:91-...`) — validates the
  `X-Dashboard-Token` header on every POST request (refusing
  with 403 on mismatch), then dispatches to the concrete
  sub-handlers: `/run`, `/commands`, `/options`, `/mail/inbox`,
  `/mail/threads`, `/mail/read`, `/mail/send`, `/issues/show`,
  `/issues/create`, `/issues/close`, `/issues/update`,
  `/pr/show` etc. No CORS headers are emitted — the dashboard
  serves from the same origin, so cross-origin requests stay
  blocked by default browser policy.
- Per-feature types like `MailMessage`, `MailThread`,
  `MailSendRequest`, `IssueShowResponse`, `IssueCreateRequest`,
  `IssueUpdateRequest`, `IssueCloseRequest`, `PRShowResponse`,
  `CrewMember`, `ReadyItem`, `OptionItem`, `SessionPreviewResponse`
  (`api.go:283-...`) are the JSON shapes for each endpoint.

### Command whitelist (`commands.go`)

- `CommandMeta` (`commands.go:10-23`) — per-command metadata for
  the palette UI: `Safe` (no confirmation needed), `Confirm`
  (requires explicit user confirmation), `Desc`, `Category`,
  `Args` placeholder hint, and an `ArgType` that points at an
  option list fetched from `/api/options`.
- `AllowedCommands` (`commands.go:27-95`) — the authoritative
  allowlist. Any command not in this map is rejected. Covers
  read-only status/mail/rig/polecat/crew/doctor commands plus
  action commands like `mail send`, `escalate ack/resolve/reassign`,
  `convoy create/refresh/add`, `rig boot/start`,
  `witness/refinery start`, `mayor attach`, `deacon start`,
  `polecat add/remove`, `sling`/`unsling`/`hook attach/detach`,
  `notify`, `broadcast`.
- `BlockedPatterns` (`commands.go:99-...`) — regex list that further
  blocks dangerous flags (`--force`, etc.) even when the base
  command is on the allowlist.
- `ValidateCommand(rawCommand)` (`commands.go:113-167`) — parses
  the caller's command string, looks it up in `AllowedCommands`,
  runs each `BlockedPatterns` regex against the argument tail,
  and returns the matching `*CommandMeta` or an error describing
  why the command was rejected.
- `SanitizeArgs(args)` (`commands.go:169-187`) — a second layer of
  defense applied to arguments before they reach `exec.Command`.
- `GetCommandList()` / `CommandInfo` (`commands.go:189-...`) — the
  list-for-UI path that `/api/commands` serves.

### Setup mode (`setup.go`)

- Separate `SetupHandler` + `SetupAPIHandler` + `NewSetupMux`
  (`setup.go:19-435`) for the "no workspace exists" onboarding
  flow. Serves an embedded `setupHTML` string (built inline at the
  bottom of the file) that guides the user through `gt install`,
  `gt rig add`, and launching the dashboard in the resulting
  workspace.
- Request types: `InstallRequest` (`setup.go:83-87`),
  `CheckWorkspaceRequest` (`setup.go:90-92`),
  `LaunchRequest` (`setup.go:95-99`),
  `RigAddRequest` (`setup.go:111-114`).
- The setup API mirrors the dashboard API's CSRF token validation
  (`setup.go:57-63`) and uses the same `X-Dashboard-Token` header.
- `handleInstall` (`setup.go:124-...`) shells out to `gt install
  [flags] -- <path>` with the `--` separator specifically to
  prevent paths like `--help` being parsed as flags.

### Input validation (`validate.go`)

A thin file (129 lines) with input-validation helpers used by the
API layer for sanitizing query parameters and request-body fields
before they're passed to subprocess invocations. No exported types.

## Related wiki pages

- [gt](../binaries/gt.md) — binary containing the dashboard command.
- [gt dashboard](../commands/dashboard.md) — user-facing CLI that
  instantiates `LiveConvoyFetcher` and mounts the handler.
- [internal/keepalive](keepalive.md) — the web server supervisor;
  `gt dashboard` wraps the web mux in a keepalive loop so the
  HTTP server is restarted if it dies.
- `internal/activity` — colored activity rendering embedded into
  `WorkerRow.LastActivity` and `ConvoyRow.LastActivity` (no wiki
  page yet).
- [internal/beads](beads.md) — provides `InjectFlatForListJSON` for
  bd v0.59+ compatibility.
- [internal/session](session.md) — provides
  `BuildPrefixRegistryFromTown` and `PrefixRegistry` used for
  parsing tmux session names into rig/role components.
- [go-packages inventory](../inventory/go-packages.md).

## Notes / open questions

- The package is **large** (~3500 lines of Go outside tests) and
  effectively owns the entire dashboard surface including data
  fetching, HTTP routing, template rendering, and command
  execution. A future split into `web/handler`, `web/api`,
  `web/fetcher` sub-packages is plausible but hasn't happened yet.
- `api.go`'s `gtPath` is hardcoded to `"gt"` from PATH rather
  than `os.Executable()` specifically because test binaries would
  otherwise fork-bomb by invoking themselves
  (`api.go:78-79`). Production deployments must have `gt` on PATH.
- Fetch errors are never surfaced to HTTP clients — they result in
  empty panels and server-side log lines (`fetcher.go:28-30`). The
  dashboard will silently degrade if the backing `bd` / `gh` /
  `tmux` tools fail, which is deliberate but may confuse operators
  debugging "why is my panel empty?".
- `commands.go`'s allowlist is the only line of defense for the
  `/api/run` endpoint. Adding a new gt command that needs
  dashboard exposure requires editing `AllowedCommands`; forgetting
  to do so means the command is silently unavailable rather than
  failing loudly.
