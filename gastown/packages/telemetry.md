---
title: internal/telemetry
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/telemetry/telemetry.go
  - /home/kimberly/repos/gastown/internal/telemetry/recorder.go
  - /home/kimberly/repos/gastown/internal/telemetry/subprocess.go
tags: [package, platform-service, telemetry, otel, opentelemetry, privacy, victoriametrics, victorialogs, opt-in]
phase3_audited: 2026-04-15
phase3_findings: [drift]
phase3_severities: [wrong]
phase3_findings_post_release: false
phase8_audited: 2026-04-17
phase8_findings: [silent-suppression]
detail_depth: {params: 2, data_flow: 2, errors: 1, side_effects: 1}
---

# internal/telemetry

OpenTelemetry wiring for Gas Town. Initializes metric and log providers
that push to a **local** VictoriaMetrics + VictoriaLogs stack via OTLP
HTTP, defines all the `Record*` helpers the rest of the codebase uses to
emit structured events, and propagates OTEL env vars to `bd` (beads) and
other subprocess children so they contribute to the same waterfall. The
whole thing is **strictly opt-in** and does nothing until an env var is
explicitly set.

**Go package path:** `github.com/steveyegge/gastown/internal/telemetry`
**File count:** 3 go files, 3 test files
**Imports (notable):**

- `go.opentelemetry.io/otel`
- `go.opentelemetry.io/otel/exporters/otlp/otlpmetric/otlpmetrichttp`
- `go.opentelemetry.io/otel/exporters/otlp/otlplog/otlploghttp`
- `go.opentelemetry.io/otel/sdk/metric` (`sdkmetric`)
- `go.opentelemetry.io/otel/sdk/log` (`sdklog`)
- `go.opentelemetry.io/otel/sdk/resource`
- `go.opentelemetry.io/otel/semconv/v1.26.0`
- stdlib (`context`, `os`, `sync`, `time`).

**Imported by (notable):** [`gt`](../binaries/gt.md) root command
(`/home/kimberly/repos/gastown/internal/cmd/root.go:18`), and most
operational commands that emit `Record*` events — especially the bd/mail
layer (every bd CLI invocation is instrumented), session lifecycle
(`RecordSessionStart`/`Stop`, `RecordAgentInstantiate`), prompt
dispatch (`RecordPromptSend`), gt prime (`RecordPrime`,
`RecordPrimeContext`), gt done (`RecordDone`), gt nudge
(`RecordNudge`), daemon restarts (`RecordDaemonRestart`), the molecule
engine (`RecordMolCook`, `RecordMolWisp`, `RecordMolSquash`,
`RecordMolBurn`), formula instantiation (`RecordFormulaInstantiate`),
convoys, polecat spawns/removes, slings, and bead creations.

**The instance that runs this package is you.** This is a self-hosted
observability stack designed to run on the agent's own machine. There is
no Anthropic, no "phone home", and no default external endpoint.

## Privacy summary (answer to wiki-9u4)

**Is it opt-in or opt-out?** **Strictly opt-in.** `telemetry.Init`
(`telemetry.go:112-126`) returns `(nil, nil)` — a no-op — unless at least
one of `GT_OTEL_METRICS_URL` or `GT_OTEL_LOGS_URL` is set. A binary run
with neither env var emits zero OTel traffic. The comment at
`telemetry.go:105-107` states this explicitly: "Returns (nil, nil) if
neither GT_OTEL_METRICS_URL nor GT_OTEL_LOGS_URL is set, so that
telemetry is strictly opt-in. Set either variable to activate."

**Where does data go?** To whatever OTLP HTTP endpoint the env var points
at. When `GT_OTEL_METRICS_URL` is set to any value, that value is used.
When it is set to the empty string alongside `GT_OTEL_LOGS_URL`, the
default endpoint kicks in:

- `DefaultMetricsURL = "http://localhost:8428/opentelemetry/api/v1/push"`
  (`telemetry.go:42`) — VictoriaMetrics local push endpoint.
- `DefaultLogsURL =
  "http://localhost:9428/insert/opentelemetry/v1/logs"`
  (`telemetry.go:45`) — VictoriaLogs local insert endpoint.

**Both defaults are `localhost`**. There is no default that would cause
data to leave the machine. Setting the env var to a remote URL is the
only way to get off-box telemetry, and that is a user choice.

**What's in the resource attributes (always-on labels)?** Built in
`telemetry.go:135-142` via `resource.New` with:

- `semconv.ServiceName(serviceName)` — the literal service name passed
  to `Init` (today this is "gt").
- `semconv.ServiceVersion(serviceVersion)` — the build-time version.
- `resource.WithHost()` — adds `host.name`, `host.id` from the OS.
  Includes **hostname**, which is identifiable.
- `resource.WithOS()` — adds `os.type`, `os.description` (kernel /
  distro info).

Additionally, when telemetry is active, `SetProcessOTELAttrs`
(`subprocess.go:59-73`) writes `OTEL_RESOURCE_ATTRIBUTES` into the
process env so **every OTel child** (bd, mail, …) inherits a second
layer of attributes built by `buildGTResourceAttrs`
(`subprocess.go:11-46`):

- `gt.role` (from `GT_ROLE`)
- `gt.rig` (from `GT_RIG`)
- `gt.actor` (from `BD_ACTOR`)
- `gt.agent` (from `GT_POLECAT` or `GT_CREW`)
- `gt.session` (from `GT_SESSION`)
- `gt.run_id` (from `GT_RUN`)
- `gt.work_rig` (from `GT_WORK_RIG`)
- `gt.work_bead` (from `GT_WORK_BEAD`)
- `gt.work_mol` (from `GT_WORK_MOL`)

None of those are secrets. All are identifiers for the agent's role,
rig, current work item, and session — the machinery that makes a
waterfall trace readable. `BD_ACTOR` is the closest thing to a personal
identifier; it is the author identity bd uses when writing beads.

**What metrics are emitted?** All instruments are registered in
`initInstruments()` (`recorder.go:145-225`). There are 23 counters and
1 histogram:

Counters (`gastown.*.total`):

- `bd.calls` — bd CLI subprocess invocations
- `session.starts` / `session.stops` — tmux session lifecycle
- `prompt.sends` — tmux SendKeys dispatches
- `pane.output` — pane output chunks
- `agent.events` — structured agent conversation events
- `agent.instantiations` — root agent spawns
- `prime` — gt prime invocations
- `agent.state_changes`
- `polecat.spawns` / `polecat.removes`
- `sling.dispatches`
- `mail.operations`
- `nudge`
- `done`
- `daemon.agent_restarts`
- `formula.instantiations`
- `convoy.creates`
- `mol.cooks` / `mol.wisps` / `mol.squashes` / `mol.burns`
- `bead.creates`

Histogram: `gastown.bd.duration_ms` — bd subprocess wall-clock latency.

Metric attributes are small and non-sensitive: `status="ok"|"error"`,
`role`, `rig`, `operation`, `subcommand`, `agent_type`, `new_state`,
`exit_type`, `formula`, `mol_source`. No bead content, no arguments, no
PII.

**What logs are emitted?** Every `emit(...)` call in `recorder.go`
produces an OTel log record whose body is the event name
(`"bd.call"`, `"session.start"`, `"agent.instantiate"`, …) and whose
attributes are a handful of string/int/float key-values. Event bodies
by default include:

- Session IDs, bead IDs, formula names, role strings, operation names,
  error strings.
- `run.id` (the GASTA/GASTOWN waterfall key), injected by `addRunID`
  from ctx or `GT_RUN` at `recorder.go:239-243`.

**Any content-bearing log fields require an additional opt-in env var.**
These are the knobs that matter for privacy:

| Env var                     | Enables                                                        | Truncation                                    | Source                              |
|-----------------------------|----------------------------------------------------------------|-----------------------------------------------|-------------------------------------|
| `GT_LOG_BD_OUTPUT=true`     | bd subprocess `stdout` + `stderr` on `bd.call` events          | `GT_LOG_BD_CONTENT_LIMIT` (default 2048 bytes) | `recorder.go:294-325`               |
| `GT_LOG_PROMPT_KEYS=true`   | Raw tmux `SendKeys` content on `prompt.send` events            | 256 bytes                                     | `recorder.go:361-380`               |
| `GT_LOG_MAIL_BODY=true`     | Mail message body on `mail` events                             | 256 bytes                                     | `recorder.go:440-467`               |
| `GT_LOG_PRIME_CONTEXT=true` | The rendered prime formula (may contain secrets!)              | untruncated                                   | `recorder.go:488-502`               |
| `GT_LOG_PANE_OUTPUT=true`   | Raw tmux pane output                                           | `GT_LOG_PANE_CONTENT_LIMIT` (default 8192)    | `recorder.go:778-791`               |
| `GT_LOG_AGENT_OUTPUT=true`  | Structured agent conversation events                           | `GT_LOG_AGENT_CONTENT_LIMIT` (default 512)    | enforced in `RecordAgentEvent`      |

Every one of these is **off by default**. The comments explicitly warn
why: bd output may contain API tokens; prompts may contain secrets;
mail bodies may contain PII; prime context may contain API keys. The
content limits are parsed once via `initContentLimits`
(`recorder.go:117-130`) and cached — runtime changes to the env vars
have no effect.

**Is there an explicit opt-out kill-switch?** There doesn't need to be,
because the package is opt-in. Unsetting `GT_OTEL_METRICS_URL` and
`GT_OTEL_LOGS_URL` fully disables telemetry; the `IsActive()` helper
(`telemetry.go:92-94`) is the package's own check for this state, and
it's used by downstream code to skip side-effectful propagation
(`SetProcessOTELAttrs`, `OTELEnvForSubprocess`) when the telemetry
plane is dormant. The OTel SDK's own `OTEL_SDK_DISABLED` is not
specifically referenced in this package — it would work at the SDK
level if set, but in practice the env-var-gated `Init` is the
controlling surface here.

**PII / secret leakage checklist:**

- **Hostname** — always included via `resource.WithHost()` when
  telemetry is enabled.
- **File paths** — `town_root` appears on `agent.instantiate` events
  (`recorder.go:426`); other events use bead IDs and session IDs
  instead of paths.
- **Tokens** — only at risk when a user explicitly flips
  `GT_LOG_BD_OUTPUT`, `GT_LOG_PROMPT_KEYS`, or `GT_LOG_PRIME_CONTEXT`,
  because those fields contain raw subprocess output or rendered
  templates. Default state: safe.
- **Mail content** — only at risk when `GT_LOG_MAIL_BODY=true`.
- **Claude Code conversation** — only at risk when
  `GT_LOG_AGENT_OUTPUT=true` (plus content-limit env var for size).

## What it actually does

### Public API — `telemetry.go`

- `telemetry.EnvMetricsURL = "GT_OTEL_METRICS_URL"` and
  `telemetry.EnvLogsURL = "GT_OTEL_LOGS_URL"` (`telemetry.go:34-39`) —
  the canonical env var names.
- `telemetry.DefaultMetricsURL` and `telemetry.DefaultLogsURL`
  (`telemetry.go:41-45`) — both localhost, as documented above.
- `telemetry.ExportInterval = 30 * time.Second`
  (`telemetry.go:48`) — periodic metric push cadence.
- `telemetry.Provider` struct (`telemetry.go:58-63`) — wraps the
  underlying OTel providers plus their shutdown hooks; idempotent
  `Shutdown(ctx)` drains pending data.
- `telemetry.IsActive() bool` (`telemetry.go:92-94`) — true iff at least
  one of the two env vars is set.
- `telemetry.Init(ctx, serviceName, serviceVersion) (*Provider, error)`
  (`telemetry.go:112-185`) — the main entry point. Idempotent via
  `initMu` + `initDone`: the first caller wins, later callers get the
  same provider back. Builds the OTLP HTTP metric and log exporters,
  wires them into `sdkmetric.NewMeterProvider` (with a
  `PeriodicReader` at 30s) and `sdklog.NewLoggerProvider` (with a
  `BatchProcessor`), registers them as the global providers, and calls
  `initInstruments()` so all counters and histograms are ready to use.

### Public API — `subprocess.go`

- `telemetry.SetProcessOTELAttrs()` (`subprocess.go:59-73`) — when
  telemetry is active, writes `OTEL_RESOURCE_ATTRIBUTES` into the current
  process env (from `buildGTResourceAttrs()`), plus mirrors
  `GT_OTEL_METRICS_URL` → `BD_OTEL_METRICS_URL` and `GT_OTEL_LOGS_URL` →
  `BD_OTEL_LOGS_URL` so beads (bd) subprocesses emit to the same
  endpoints. Called once at gt startup (from the root command's
  `Execute`) when telemetry is on.
- `telemetry.OTELEnvForSubprocess() []string`
  (`subprocess.go:83-100`) — returns `[]string` env additions for
  callers that build `cmd.Env` explicitly (e.g. `beads.go run`,
  `mail/bd.go runBdCommand`) instead of starting from `os.Environ()`.
  Ensures OTEL propagation survives explicit env rebuilds.

### Public API — `recorder.go` (the event helpers)

- `telemetry.WithRunID(ctx, runID) context.Context` and
  `telemetry.RunIDFromCtx(ctx) string` (`recorder.go:27-43`) — context
  plumbing for the waterfall primary key. Falls back to `GT_RUN` env
  var so subprocess events correlate even when ctx is pristine.
- `telemetry.MailMessageInfo` struct (`recorder.go:63-72`) and
  `telemetry.AgentInstantiateInfo` struct (`recorder.go:385-410`) —
  carry structured fields for the richer events.
- 23 `Record*` functions that each increment a counter and emit a log
  event, listed in the "What metrics are emitted?" section above. All
  respect the opt-in content env vars where applicable. Key signatures:
  - `RecordBDCall(ctx, args, durationMs, err, stdout, stderr)`
    (`recorder.go:296-326`)
  - `RecordSessionStart(ctx, sessionID, role, err)` /
    `RecordSessionStop(ctx, sessionID, err)`
  - `RecordPromptSend(ctx, session, keys, debounceMs, err)`
  - `RecordAgentInstantiate(ctx, info)`
  - `RecordMailMessage(ctx, operation, msg, err)`
  - `RecordPrime(ctx, role, hookMode, err)` /
    `RecordPrimeContext(ctx, formula, role, hookMode)`
  - `RecordAgentStateChange(ctx, agentID, newState, hookBead, err)`
  - `RecordPolecatSpawn`/`RecordPolecatRemove`
  - `RecordSling(ctx, bead, target, err)`
  - `RecordMail(ctx, operation, err)`
  - `RecordNudge(ctx, target, err)`
  - `RecordDone(ctx, exitType, err)`
  - `RecordDaemonRestart(ctx, agentType)`
  - `RecordFormulaInstantiate(ctx, formulaName, beadID, err)`
  - `RecordConvoyCreate(ctx, beadID, err)`
  - `RecordAgentTokenUsage(ctx, sessionID, nativeSessionID,
    inputTokens, outputTokens, cacheReadTokens, cacheCreationTokens)`
  - `RecordMolCook`/`RecordMolWisp`/`RecordMolSquash`/`RecordMolBurn`
  - `RecordBeadCreate(ctx, beadID, parentID, molSource)`
  - `RecordPaneOutput(ctx, sessionID, content)`
  - `RecordAgentEvent(...)` (enforces `GT_LOG_AGENT_OUTPUT=true`
    internally so callers cannot bypass the opt-in).

### Internals / Notable implementation

- **Global state.** `initDone`, `globalProvider` (`telemetry.go:52-56`),
  and the `instOnce`/`inst` pair (`recorder.go:110-113`) are package-level
  singletons. A second call to `Init` is a no-op that returns the original
  provider. The service name passed on the second call is silently
  ignored.
- **Periodic metrics, batched logs.** Metrics are pushed every 30 s via
  `PeriodicReader`; logs are shipped via `BatchProcessor` (default OTel
  SDK batching) to reduce request volume.
- **Metrics meter name** is hard-coded as
  `"github.com/steveyegge/gastown"` (`recorder.go:75`). Logger name is
  `"gastown"` (`recorder.go:76`).
- **Content truncation is UTF-8 safe.** `truncateOutput`
  (`recorder.go:275-286`) walks back from the cut point until the prefix
  is a valid UTF-8 string, then appends `…`, so multi-byte runes aren't
  split.
- **`BD_OTEL_*` mirroring.** Gas Town uses `GT_OTEL_*` to avoid colliding
  with bd's own `BD_OTEL_*` vars; the mirror logic in
  `SetProcessOTELAttrs` and `OTELEnvForSubprocess` keeps the two
  ecosystems pointed at the same VictoriaMetrics/VictoriaLogs.
- **Instance label.** `instanceID(townRoot)`
  (`recorder.go:49-59`) derives a human-readable label as
  `"<hostname>:<basename(townRoot)>"` (e.g. `laptop:gt`). It appears on
  `agent.instantiate` events at `recorder.go:425`. This is the most
  identifiable field in the default event surface.

### Usage pattern

1. `gt` main/root calls `telemetry.Init(ctx, "gt", version.Commit)`
   once during startup. If neither env var is set, `Init` no-ops.
2. When active, root also calls `telemetry.SetProcessOTELAttrs()` so
   every child subprocess inherits the OTLP env and resource attrs.
3. Throughout the codebase, command and service code calls the relevant
   `telemetry.Record*` helper whenever an interesting thing happens.
   These helpers are always safe to call — when the providers are the
   no-op defaults, the calls dissolve into cheap meter/logger lookups
   that go nowhere.
4. On shutdown, root invokes `provider.Shutdown(ctx)` with a 5 s
   deadline to flush the batcher.

## Related wiki pages

- [gt](../binaries/gt.md) — the binary that hosts this package and
  calls `Init`/`Shutdown`.
- [internal/session](session.md) — heavy consumer (`RecordSessionStart`,
  `RecordSessionStop`, `RecordAgentInstantiate`).
- [internal/config](config.md) — source of `AgentEnvConfig` and
  `IdentityEnvVars`, adjacent to the env propagation that
  `subprocess.go` manages.
- [gt prime](../commands/prime.md) — emits `RecordPrime` and optional
  `RecordPrimeContext`.
- [gt done](../commands/done.md) — emits `RecordDone`.
- [gt nudge](../commands/nudge.md) — emits `RecordNudge`.
- [go-packages inventory](../inventory/go-packages.md).

## Docs claim

### Source
- `/home/kimberly/repos/gastown/docs/design/otel/otel-architecture.md` (lines 76-84, implementation status table)

### Verbatim
> | Molecule lifecycle telemetry | ❌ Roadmap | `mol.cook`/`mol.wisp`/`mol.squash`/`mol.burn` — no `RecordMol*` functions exist |
> | Bead creation telemetry | ❌ Roadmap | `bead.create` — no `RecordBeadCreate` function exists |
> | **Agent instantiation** | ❌ Roadmap | `agent.instantiate` event — no `RecordAgentInstantiate` function exists |

## Drift

### docs/design/otel/otel-architecture.md claims RecordMol*/RecordBeadCreate/RecordAgentInstantiate don't exist but they do
- **Claim source:** `docs/design/otel/otel-architecture.md` (lines 54, 80-81)
- **Docs claim:** The implementation status table marks molecule lifecycle, bead creation, and agent instantiation telemetry as "❌ Roadmap" with explicit notes like "no RecordMol* functions exist" and "no RecordAgentInstantiate function exists."
- **Code does:** `/home/kimberly/repos/gastown/internal/telemetry/recorder.go` has all these functions implemented: `RecordMolCook` (line 692), `RecordMolWisp` (line 710), `RecordMolSquash` (line 731), `RecordMolBurn` (line 749), `RecordBeadCreate` (line 766), `RecordAgentInstantiate` (line 414). Corresponding metric instruments are declared at lines 88, 100-104. Total instrument count is 24 (23 counters + 1 histogram), not the "18 metric instruments" stated in line 44.
- **Category:** `drift`
- **Severity:** `wrong`
- **Fix tier:** `docs` — the otel-architecture.md implementation status table should mark these as ✅ Main and update the metric count.
- **Release position:** `in-release`

## Failure modes

### Silent suppression (what errors are swallowed?)
- **`SetProcessOTELAttrs` discards Setenv errors:**
  `subprocess.go:65-71` uses `_ = os.Setenv(...)` for
  `OTEL_RESOURCE_ATTRIBUTES`, `BD_OTEL_METRICS_URL`, and
  `BD_OTEL_LOGS_URL`. If any of these fail (theoretically possible on
  some platforms with env-size limits), child subprocesses silently
  run without telemetry propagation. **Absent** — no log or fallback;
  bd subprocess metrics silently disappear from the waterfall.
- **Init idempotency silently discards service name:** `telemetry.go:99-103`
  documents that subsequent `Init` calls return the first-caller's
  provider and silently ignore the new `serviceName` and
  `serviceVersion`. **Present** — explicitly documented in the
  function's doc comment, and in practice each process calls `Init`
  once, so this is informational rather than a bug.

## Notes / open questions

- No explicit "kill switch" env var (e.g. `GT_TELEMETRY=0`) — the
  absence of `GT_OTEL_*` env vars is the disable signal. Adding an
  explicit kill switch could be a future UX improvement for users who
  have the vars set in their shell profile but want to temporarily
  suppress telemetry for one run.
- The `BD_ACTOR` → `gt.actor` resource attribute is the most
  personal-data-adjacent default label. Worth flagging to operators
  considering whether to forward this stack off-box.
- Content-limit env vars are read once and cached
  (`recorder.go:117-130`); runtime reconfiguration requires a restart.
- `RecordAgentEvent` (full body at recorder.go:793+) was only partially
  inspected for this page because the file is long; it enforces its
  own opt-in via `GT_LOG_AGENT_OUTPUT`. If a future pass needs the
  exact event schema, read lines 793-end.
