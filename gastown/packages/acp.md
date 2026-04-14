---
title: internal/acp
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/acp/proxy.go
  - /home/kimberly/repos/gastown/internal/acp/proxy_unix.go
  - /home/kimberly/repos/gastown/internal/acp/proxy_windows.go
  - /home/kimberly/repos/gastown/internal/acp/propulsion.go
tags: [package, acp, jsonrpc, mayor, ide, proxy, agent-client-protocol]
---

# internal/acp

Agent Client Protocol (ACP) implementation for Gas Town. Runs a
bidirectional JSON-RPC proxy between an ACP client (typically an IDE
like Zed) on one side and a Claude Code (or similar) agent process
on the other, watching for mayor mail / nudge / escalation events and
injecting them into the live conversation as `session/update`
notifications (UI-visible) and `session/prompt` requests (agent-visible).
This is the mechanism by which the Mayor runs as an IDE-attached agent
rather than a tmux session.

**Go package path:** `github.com/steveyegge/gastown/internal/acp`
**File count:** 4 go files (`proxy.go` + platform-split
`proxy_unix.go`/`proxy_windows.go` + `propulsion.go`), 8 test files
**Imports (notable):** stdlib (`bufio`, `context`, `encoding/json`,
`io`, `os`, `os/exec`, `os/signal`, `sync`, `sync/atomic`,
`syscall` + `time`), plus [`internal/nudge`](nudge.md) for the queue
watcher, [`internal/style`](style.md) for warnings,
`internal/townlog` for lifecycle events (no wiki page yet), and
[`internal/util`](util.md) for process-group helpers.
**Imported by (notable):** [`gt mayor`](../commands/mayor.md) —
specifically the ACP-mode entry points that wrap Claude Code behind
the proxy. The mayor role's ACP mode is what makes this package
load-bearing for the Mayor-as-IDE-agent flow.

## What it actually does

### Proxy core (`proxy.go`)

- `JSONRPCMessage` (`proxy.go:94-106`) — shared request/notification
  envelope: `JSONRPC`, `ID`, `Method`, `Params`, `Result`, `Error`.
  Matches the JSON-RPC 2.0 shape the ACP spec requires.
- `SessionMode` / `SessionModeState` / `SessionNewResult`
  (`proxy.go:109-125`) — ACP session lifecycle types.
- `handshakeState` constants (`proxy.go:22-27`) —
  `handshakeInit` → `handshakeWaitingForInit` →
  `handshakeWaitingForSessionNew` → `handshakeComplete`, tracking the
  ACP initialization handshake as requests flow through.
- `Proxy` (`proxy.go:42-83`) — the central struct. Holds the agent
  subprocess (`cmd`, stdin/stdout/stderr), the UI streams
  (`stdin`, `stdout`, `uiEncoder`), the session ID under
  `sessionMux`, the handshake state under `handshakeMux`, a prompt
  mutex tracking `activePromptID` for `IsBusy()`, startup-prompt
  state, and a grab-bag of lifecycle atomics (`isShuttingDown`,
  `lastActivity`, `Propelled`, stderr-overflow counters, heartbeat
  state, `pidFilePath`, `townRoot`). The `Propelled` atomic
  distinguishes propeller-driven sessions from plain ACP proxies.
- `NewProxy()` (`proxy.go:127-138`) — initializes defaults;
  `setStreams(in, out)` (`proxy.go:141-145`) lets tests redirect.
- `Start(ctx, agentPath, agentArgs, cwd)` (`proxy.go:181-302`) —
  spawns the agent subprocess with pipes, applies
  platform-specific process group setup (`proxy_unix.go` /
  `proxy_windows.go`), and launches goroutines for stderr
  forwarding + crash diagnostics.
- `Forward()` (`proxy.go:303-363`) — the main loop. Launches
  `forwardToAgent`, `forwardFromAgent`, `runKeepAlive` (30s ticker),
  and optionally `monitorPIDFile`. Waits on signals, the internal
  `done` channel, or the agent process exit. After any trigger,
  calls `Shutdown()` and waits up to 200ms for goroutines before
  returning.
- `forwardToAgent()` (`proxy.go:365-421`) — reads UI JSON-RPC from
  stdin, unmarshals, tracks the handshake state, and writes to the
  agent stdin. Uses a 1MiB buffered reader to handle large IDE
  messages.
- `forwardFromAgent()` (`proxy.go:437-...`) — reads agent stdout,
  parses JSON-RPC, and forwards to the UI. Non-JSON lines are
  funneled into a 2KiB rolling `propulsionBuffer` that is scanned
  for raw-text propulsion triggers (`propulsion.go`'s
  `isPropulsionTrigger`); a trigger sets `Propelled=true`.
- `InjectNotificationToUI(method, params)` (`proxy.go:852-893`) —
  constructs a JSON-RPC notification with the current `sessionID`
  mixed into params and writes it to the UI side. Refuses to inject
  `session/update` when `sessionID == ""` (the handshake hasn't
  completed). This is the primary channel for delivering nudges to
  the UI.
- `InjectPrompt(prompt)` (`proxy.go:895-943`) — sends a
  `session/prompt` request directly to the agent. Gated on
  `IsBusy()` — if the agent is mid-turn, `InjectPrompt` returns an
  error unless the startup prompt is still in-flight, in which case
  it waits up to 15s on `WaitForReady`. Each injection gets a
  unique `gt-inject-prompt-<nano>` ID.
- `SessionID()` / `WaitForSessionID(ctx)` (`proxy.go:945-972`) —
  accessor + polling wait (50ms ticker). Propulsion uses this to
  gate notification delivery until the IDE has actually completed
  the ACP handshake.
- `WaitForReady(ctx)` (`proxy.go:974-1004`) — waits for sessionID
  **and** an idle prompt state. Used by `InjectPrompt` when the
  startup prompt is still pending.
- `IsBusy()` (`proxy.go:1006-1010`) — `activePromptID != ""`.
- `SendCancelNotification()` (`proxy.go:1012-1032`) — sends
  `session/cancel` to the agent; called on shutdown paths.
- `monitorPIDFile(ctx)` (`proxy.go:1034-1058`) — optional goroutine
  that polls a PID-file path every 500ms and shuts down when the
  file disappears (the caller's "am I still alive?" signal, used
  when the ACP proxy is supervised by `gt mayor attach` etc.).
- `Shutdown()` (`proxy.go:1060-...`) — `sync.Once`-gated teardown:
  sets `isShuttingDown`, closes `done`, cancels the context, and
  terminates the agent via the platform-specific
  `terminateProcess()`.

### Platform-specific process handling (`proxy_unix.go` / `proxy_windows.go`)

- `signalsToHandle()` (`proxy_unix.go:15-17`, `proxy_windows.go:15-17`)
  — unix returns `[SIGTERM, SIGINT]`; windows returns `[os.Interrupt]`.
- `setupProcessGroup()` (`proxy_unix.go:21-23`, `proxy_windows.go:22-24`)
  — unix sets a new process group via `util.SetProcessGroup` so the
  whole tree can be signaled; windows uses
  `util.SetDetachedProcessGroup` to suppress the transient console
  window for GUI-parented children.
- `isProcessAlive()` (`proxy_unix.go:27-33`, `proxy_windows.go:28-39`)
  — unix uses `kill -0`; windows uses `OpenProcess` with
  `PROCESS_QUERY_LIMITED_INFORMATION` (treating `ERROR_ACCESS_DENIED`
  as "alive, just not our process").
- `terminateProcess()` (`proxy_unix.go:38-70`, `proxy_windows.go:43-47`)
  — unix sends SIGTERM to the process **group**, then SIGKILL after
  2 seconds if it's still running. A critical safety check:
  compares pgid against `Getpgid(0)` and refuses to signal the whole
  group if the child shares our own pgid, so tests and local runs
  never self-terminate. Windows just calls `cmd.Process.Kill()`.

### Propeller / event-driven nudge delivery (`propulsion.go`)

- `acpDebugLogger` (`propulsion.go:19-91`) — lazy file-based debug
  logger that opens `<townRoot>/logs/acp.log` on first use when
  `GT_ACP_DEBUG=1` is set. Keeps the file open for the session,
  serialized by a mutex, closed on `Propeller.Stop`.
- `debugLog(townRoot, format, args...)` (`propulsion.go:95-107`) and
  `logEvent(townRoot, eventType, context)` (`propulsion.go:110-118`)
  — the two emission paths: debug log (only when `GT_ACP_DEBUG=1`)
  and town.log lifecycle events (always, via
  `townlog.NewLogger`).
- `Propeller` (`propulsion.go:120-128`) — the driver that connects a
  `Proxy` to the nudge queue. Holds the proxy, townRoot, session
  name, a cancellable context, a waitgroup, and a
  `warnedNoSID bool` that dedupes the "no IDE connected" warning.
- `Start(ctx)` (`propulsion.go:138-143`) — launches
  `waitForSessionAndStart` which blocks up to 30s on
  `Proxy.WaitForSessionID`. If the handshake never completes, the
  propeller logs an `acp_degraded` event and runs in a reduced
  mode (mail/hook detection still works, but notifications are
  skipped because there's no session to target).
- `eventLoop()` (`propulsion.go:175-194`) — watches
  `nudge.WatcherForSession(townRoot, session)` for events; when an
  event fires, calls `deliverNudges()` to drain the queue.
- `deliverNudges()` (`propulsion.go:197-239`) — drains the nudge
  queue via `nudge.Drain`, formats for injection with
  `nudge.FormatForInjection`, computes a per-batch urgency flag
  (any `nudge.PriorityUrgent` → urgent), builds an ACP session
  update `_meta` map via `buildSessionUpdateMeta`, and calls
  `notify`. If delivery fails (no session yet, injection error),
  `requeue` puts the nudges back on the queue and logs a
  degraded-mode event.
- `buildSessionUpdateMeta(nudges, session)` (`propulsion.go:265-286`)
  — builds a `map[string]string` of `gt/eventType`, `gt/count`,
  `gt/drained`, `gt/session`, `gt/urgent` counts. When an escalation
  is in the batch, `gt/escalation=true` plus `gt/threadID`,
  `gt/severity`, and `gt/kind` are added.
- `notify(text, meta, urgent)` (`propulsion.go:336-368`) — sends to
  both the UI (via `InjectNotificationToUI("session/update", ...)`)
  and the Agent (via `InjectPrompt`). The agent prompt is skipped
  when `IsBusy()` unless the nudge is urgent. When urgent, it
  retries `InjectPrompt` up to 3 times with 100ms backoff before
  giving up and logging `acp_error`.

## Related wiki pages

- [gt](../binaries/gt.md) — binary whose mayor subcommand drives the
  ACP proxy.
- [gt mayor](../commands/mayor.md) — the user-facing command that
  wraps the mayor's Claude Code instance in this proxy.
- [mayor role](../roles/mayor.md) — conceptual description of the
  mayor agent that runs behind this proxy when in IDE-attached mode.
- [internal/nudge](nudge.md) — provides the queue, watcher, drain,
  requeue, and injection formatter.
- `internal/townlog` — destination for
  `acp_stop`/`acp_error`/`acp_prompt`/`acp_shutdown`/`acp_degraded`
  lifecycle events (no wiki page yet).
- [go-packages inventory](../inventory/go-packages.md).

## Notes / open questions

- `Proxy.Forward` waits only **200ms** for goroutines to exit after
  shutdown (`proxy.go:350-355`). Any stuck reader loop past that
  window is abandoned — the process exits regardless. This was a
  deliberate choice to avoid hangs when the agent misbehaves on exit,
  but means clean shutdowns may race with the last few in-flight
  messages.
- The `propulsionBuffer` is a 2KiB rolling window of **non-JSON**
  lines from the agent stdout. When the agent emits raw-text output
  containing a propulsion trigger, the buffer captures it even
  across line boundaries (`proxy.go:470-479`). This is the
  fallback path for agents that don't emit proper JSON-RPC when
  propelled.
- `GT_ACP_DEBUG=1` gates the file-based `acp.log`; lifecycle events
  always go to `town.log`. There is no way to turn off `town.log`
  logging short of omitting `townRoot` (`propulsion.go:111-114`).
