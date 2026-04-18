---
title: gt heartbeat
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/cmd/heartbeat.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, diagnostics, polecat, heartbeat, zfc, beads-exempt]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase5_audience: agent
phase8_audited: 2026-04-17
phase8_findings: [silent-suppression]
detail_depth: {params: 2, data_flow: 2, errors: 2, side_effects: 2}
---

# gt heartbeat

Updates the current session's polecat heartbeat file with a specific
agent state. Intended to be called by agents themselves to self-report
to the witness (rather than having the witness infer state from
timers).

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupDiag` ("Diagnostics")
**Polecat-safe:** no (no `AnnotationPolecatSafe` on the cobra.Command at
`/home/kimberly/repos/gastown/internal/cmd/heartbeat.go:13-33`)
**Beads-exempt:** **yes** — `heartbeat` is in `beadsExemptCommands` at
`/home/kimberly/repos/gastown/internal/cmd/root.go:76`, with the
inline comment: "Heartbeat state update — must be fast and
dependency-free."
**Branch-check-exempt:** no (not in `branchCheckExemptCommands`)

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/heartbeat.go:13-69`.

### Invocation

```
gt heartbeat [--state working|idle|exiting|stuck] [context words...]
```

Single terminal command (no subcommands). Runs `runHeartbeat`
(`heartbeat.go:42-69`).

### Behavior

`runHeartbeat` (`heartbeat.go:42-69`):

1. Reads `GT_SESSION` from the environment. Fails with "GT_SESSION not
   set (not running in a Gas Town session)" if empty
   (`heartbeat.go:43-46`).
2. Resolves the town root via `workspace.FindFromCwd`. Fails if not
   found (`heartbeat.go:48-51`).
3. Parses the `--state` flag into a `polecat.HeartbeatState` value and
   validates it against the four accepted constants (`heartbeat.go:53-59`):
   - `polecat.HeartbeatWorking`
   - `polecat.HeartbeatIdle`
   - `polecat.HeartbeatExiting`
   - `polecat.HeartbeatStuck`
4. Joins any positional arguments into a `context` string with spaces
   (`heartbeat.go:61-64`).
5. Calls `polecat.TouchSessionHeartbeatWithState(townRoot, sessionName, state, context, "")`
   (`heartbeat.go:66`) — writes to a per-session heartbeat file in the
   polecat directory, exact path determined by the `internal/polecat`
   package.
6. Prints `"Heartbeat updated: state=<state>"` (`heartbeat.go:67`).

No error is returned from the touch call — it is fire-and-forget. This
matches the beads-exempt rationale in `root.go:76`: the command must be
fast and must not fail for dependency reasons.

### Heartbeat states

From `heartbeat.go:22-27` (Long description):

- `working` — actively processing (default if no `--state` given)
- `idle` — waiting for input
- `exiting` — in `gt done` flow
- `stuck` — self-reporting stuck (triggers witness escalation)

### Relationship to `persistentPreRun`

[../binaries/gt.md](../binaries/gt.md) documents that every `gt`
invocation with `GT_SESSION` set (and an appropriate `GT_ROLE`) calls
`polecat.TouchSessionHeartbeat` in `persistentPreRun`. `gt heartbeat`
is the **explicit** variant that lets the agent set a state value
rather than relying on the implicit "touched via any command" signal.
The internal commit reference in both places is `gt-3vr5` /
`gt-qjtq` ("ZFC liveness fix"). See
`/home/kimberly/repos/gastown/internal/cmd/root.go` `persistentPreRun`
section.

### Subcommands

None (terminal command).

### Flags

Defined in `init()` (`heartbeat.go:37-40`):

| Flag      | Type   | Default   | Purpose                                                |
|-----------|--------|-----------|--------------------------------------------------------|
| `--state` | string | `working` | Agent state (working, idle, exiting, stuck)            |

Positional args: optional context string (joined with spaces) passed
to `TouchSessionHeartbeatWithState` as the context field.

## Related

- [doctor](doctor.md) — workspace-level health checks; doctor runs
  many checks that eventually feed back into witness decisions, while
  heartbeat is the single-agent self-report.
- [log](log.md) — town log records agent lifecycle events from the
  outside; heartbeat is the agent-side self-report.
- [../binaries/gt.md](../binaries/gt.md) — parent binary; documents
  how `persistentPreRun` already touches the heartbeat for every
  polecat/crew/dog session on every `gt` invocation.
- [README.md](README.md) — command tree index.

## Failure modes

### Silent suppression (what errors are swallowed?)
- **Heartbeat file write errors:** `TouchSessionHeartbeatWithState`
  (`polecat/heartbeat.go:78`) has a `void` return type. Errors from
  `os.MkdirAll` (line 80), `json.Marshal` (line 91), and the
  underlying file write are all silently swallowed. The caller at
  `heartbeat.go:66` prints "Heartbeat updated" unconditionally, even
  if the file was never written. **Absent** — no error propagation.

This is a deliberate design choice (heartbeat must be fast and
dependency-free per the beads-exempt comment at `root.go:76`), but it
means the witness may see a stale heartbeat and misclassify the agent
as stuck when the real problem is a filesystem error.

## Notes / open questions

- The final empty-string argument to `TouchSessionHeartbeatWithState`
  (`heartbeat.go:66`) is a fifth parameter whose purpose is not
  obvious from the call site — likely a hook or molecule context. Worth
  reading `internal/polecat/` to clarify.
- Return value of `TouchSessionHeartbeatWithState` is discarded; a
  failing heartbeat write is silent.
- `gt heartbeat --state stuck` is described in the Long text as a
  trigger for witness escalation, but the escalation path is implemented
  in the witness reading the heartbeat file, not in this command.
