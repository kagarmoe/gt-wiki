---
title: internal/runtime
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/runtime/runtime.go
  - /home/kimberly/repos/gastown/internal/runtime/runtime_test.go
tags: [package, runtime, startup, hooks, nudge, fallback]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase8_audited: 2026-04-17
phase8_findings: [none]
detail_depth: {params: 2, data_flow: 2, errors: 1, side_effects: 1}
---

# internal/runtime

Agent-runtime integration helpers: provisioning per-role hook/slash-command
installs, resolving the runtime session ID from environment variables,
and computing **startup fallback** actions for runtimes that lack hooks
and/or CLI prompt support. Single file, 272 lines, tight dependency footprint.

**Go package path:** `github.com/steveyegge/gastown/internal/runtime`
**File count:** 1 non-test go file (`runtime.go`) + 1 test file
(`runtime_test.go`).
**Imports:** [`internal/cli`](cli.md), [`internal/config`](config.md),
`internal/hooks`, `internal/hookutil`, `internal/templates/commands`,
[`internal/tmux`](tmux.md).
**Imported by (notable):** [`internal/session`](session.md) lifecycle
code, every role manager (`polecat`, `crew`, `witness`, `refinery`,
`deacon`), [`gt install`](../commands/install.md),
[`gt hook`](../commands/hook.md), [`gt prime`](../commands/prime.md),
[`gt doctor`](../commands/doctor.md), [`internal/checkpoint`](checkpoint.md),
[`internal/beads`](beads.md), [`internal/mail`](mail.md). Despite being one
file, this package is load-bearing across the whole agent spawn pipeline.

## What it actually does

Four concerns live in one file; they are related but distinct.

### 1. Per-role settings provisioning

Source: `/home/kimberly/repos/gastown/internal/runtime/runtime.go:24-57`.

`runtime.EnsureSettingsForRole(settingsDir, workDir, role, rc *config.RuntimeConfig) error`
installs everything an agent needs in its working directory before
launch:

1. Reads `rc.Hooks.Provider` from the
   [RuntimeConfig](config.md#runtimeconfig). If it's empty or `"none"`,
   returns without doing anything.
2. Looks up the agent preset via `config.GetAgentPresetByName(provider)`
   to get `preset.HooksUseSettingsDir` (line 41-43), which decides
   whether hooks are installed into `settingsDir` or into `workDir`.
   This distinction matters because for crew/witness/refinery/polecat,
   `settingsDir` is a gastown-managed parent directory (passed via a
   `--settings` flag) while `workDir` is the **customer repo** — we
   don't want to write provider settings into a repo we don't own.
   For mayor/deacon the two directories are the same.
3. Calls `hooks.InstallForRole(provider, settingsDir, workDir, role,
   rc.Hooks.Dir, rc.Hooks.SettingsFile, useSettingsDir)` — delegates
   the actual provider-specific settings install to `internal/hooks`.
4. If the provider is a "known agent" per
   `commands.IsKnownAgent(provider)`, calls
   `commands.ProvisionFor(workDir, provider)` from
   `internal/templates/commands` to drop slash-command bodies into
   the work directory (agent-agnostic body with provider-specific
   frontmatter).

The comment is explicit about the `settingsDir` vs. `workDir`
separation: it's the only thing keeping gastown from polluting
customer repos.

### 2. Runtime session ID resolution

Source: `runtime.go:67-83`.

`runtime.SessionIDFromEnv() string` resolves the current agent's
runtime-session UUID from environment variables, with this lookup
order:

1. `$GT_SESSION_ID_ENV` — if set, its value is the *name* of the env
   var to read the session ID from. Indirection layer so rigs with
   different runtimes can point at their own session-ID var.
2. The current agent preset's `SessionIDEnv` field —
   `config.GetAgentPresetByName(os.Getenv("GT_AGENT")).SessionIDEnv`.
   Every runtime preset in `config.AgentPreset` declares which env
   var it uses for its session ID (Claude Code uses
   `CLAUDE_SESSION_ID`, Codex uses something else, etc.).
3. Backwards-compatible fallback: `$CLAUDE_SESSION_ID` directly, for
   sessions that predate the `GT_AGENT` wiring.

This is the single lookup path any code should use when it needs the
current agent's runtime session ID. Callers include checkpoint
persistence, beads metadata, mail envelope construction, and
`gt prime` for setting up session-scoped state.

### 3. Startup fallback (hooks vs. no hooks × prompt vs. no prompt)

Source: `runtime.go:85-200`.

Different agent runtimes support different subsets of two capabilities:

- **Hooks** — the runtime can call a `SessionStart` hook that invokes
  `gt prime` before the user sees a prompt. Detected via
  `rc.Hooks.Provider != "" && provider != "none" && !rc.Hooks.Informational`.
- **Prompt** — the runtime accepts an initial user prompt as a CLI
  argument (e.g., `claude 'first message'`). Detected via
  `rc.PromptMode != "none"`.

The cross product produces four cases. `StartupFallbackInfo`
(`runtime.go:139-155`) encodes what the session-creation code has to
do in each case, via four bool/int fields that encode the fallback
matrix documented in the doc comment at `runtime.go:121-138`:

| Hooks | Prompt | Beacon content           | Context source     | Work instructions |
|-------|--------|--------------------------|--------------------|-------------------|
| Y     | Y      | Standard                 | Hook runs gt prime | In beacon         |
| Y     | N      | Standard (via nudge)     | Hook runs gt prime | Same nudge        |
| N     | Y      | "Run gt prime" (prompt)  | Agent runs manually | Delayed nudge    |
| N     | N      | "Run gt prime" (nudge)   | Agent runs manually | Delayed nudge    |

`StartupFallbackInfo` fields:

- `IncludePrimeInBeacon bool` — beacon text should include a "Run
  `gt prime`" instruction. True only for non-hook agents.
- `SendBeaconNudge bool` — the beacon must be delivered via tmux
  nudge instead of as the CLI prompt. True whenever the runtime has
  no prompt support.
- `SendStartupNudge bool` — work instructions need a separate nudge
  after the beacon. True unless **both** hooks and prompt are
  available.
- `StartupNudgeDelayMs int` — wall-clock delay before sending work
  instructions, to give `gt prime` time to finish on non-hook agents.
  Set to `DefaultPrimeWaitMs = 2000` (`runtime.go:125-126`) for
  non-hook cases, `0` for the hooks+no-prompt case (hook runs
  synchronously).

`GetStartupFallbackInfo(rc *config.RuntimeConfig) *StartupFallbackInfo`
(`runtime.go:170-200`) computes this struct by inspecting `rc.Hooks`
and `rc.PromptMode`.

`StartupFallbackCommands(role, rc) []string` (`runtime.go:86-104`)
returns the commands to run when hooks are unavailable. For non-hook
agents the command is `gt prime`, and for **autonomous roles**
(anything where `hookutil.IsAutonomousRole(role)` returns true —
polecat, crew, witness, refinery, dog) it's extended to
`gt prime && gt mail check --inject` so the agent immediately picks
up its inbox on startup. For hook-capable agents this returns nil.

The comment at `runtime.go:99-102` notes that a session-started nudge
to the deacon was **removed** because it interrupted the deacon's
`await-signal` exponential-backoff sleep — the deacon already wakes
on beads activity via `bd activity --follow`. Small but load-bearing
historical note about deacon liveness.

`RunStartupFallback(t *tmux.Tmux, sessionID, role, rc) error`
(`runtime.go:107-115`) is the thin driver that applies
`StartupFallbackCommands` via `t.NudgeSession` on the given session
ID.

### 4. Startup prompt delivery fallback

Source: `runtime.go:157-235`.

This is the complement: the **startup prompt** (the first real
message to an agent) must be delivered via nudge for runtimes that
can't accept it as a CLI argument.

- `StartupPromptFallback struct` (`runtime.go:159-167`) — `Send bool`,
  `DelayMs int`.
- `GetStartupPromptFallback(rc) StartupPromptFallback`
  (`runtime.go:204-210`) — derives the struct from
  `GetStartupFallbackInfo`.
- `DeliverStartupPromptFallback(t startupPromptSession, sessionID,
  prompt string, rc *config.RuntimeConfig, timeout time.Duration) error`
  (`runtime.go:214-235`) — if `fallback.Send` is false, returns nil.
  Otherwise waits for runtime readiness via
  `t.WaitForRuntimeReady(...)` (passing a modified config that
  forces the delay-based path — see below), then nudges the prompt
  via `t.NudgeSession`.

`startupPromptSession` (`runtime.go:59-62`) is an interface declaring
only `NudgeSession` and `WaitForRuntimeReady`, so the function takes
the smallest possible surface from [`*tmux.Tmux`](tmux.md) rather
than depending on the full concrete type. Test fakes can implement it
without pulling in the whole tmux package.

### 5. Content helpers

Source: `runtime.go:237-245`.

- `StartupNudgeContent() string` (`runtime.go:238-240`) — returns the
  work instructions body sent as a startup nudge. Uses `cli.Name()`
  (from [`internal/cli`](cli.md)) so the binary name is resolved at
  runtime rather than hardcoded:

  ```
  Check your hook with `<cli> hook`. If work is present, begin immediately.
  ```

- `BeaconPrimeInstruction() string` (`runtime.go:243-245`) — returns
  the instruction appended to the beacon for non-hook agents:

  ```

  Run `<cli> prime` to initialize your context.
  ```

### 6. The `RuntimeConfigWithMinDelay` escape hatch

Source: `runtime.go:247-272`.

`RuntimeConfigWithMinDelay(rc *config.RuntimeConfig, minMs int)
*config.RuntimeConfig` returns a **shallow copy** of `rc` with:

1. `Tmux.ReadyDelayMs` set to at least `minMs`.
2. `Tmux.ReadyPromptPrefix` **cleared to empty**.

The second step is the whole point of the function. The comment at
`runtime.go:253-258` explains: `WaitForRuntimeReady` normally
short-circuits as soon as it detects the prompt prefix. For the
prime-wait path specifically, we want a guaranteed wall-clock delay
floor (so the agent has had time to actually process the beacon and
run `gt prime`), **not** immediate prompt detection. Clearing
`ReadyPromptPrefix` forces the delay-based code path.

This is the only caller-visible trick in the file: the code elsewhere
assumes prompt detection is strictly better than wall-clock waiting,
and this function is the one place that deliberately disables it.

### Internals / notable implementation

- **`isAutonomousRole` is a one-liner alias.** `runtime.go:120-122`
  delegates to `hookutil.IsAutonomousRole` rather than maintaining its
  own role list. The comment calls out that `hookutil` is the single
  source of truth for role classification.
- **Shallow-copy config mutation.** `RuntimeConfigWithMinDelay` uses
  `cp := *rc` and `tmuxCp := *cp.Tmux` to avoid aliasing the caller's
  config. This is defensive — the function is called in a fast path
  where a stray mutation would corrupt the session's runtime config.
- **Interface-narrowed dependency.** `startupPromptSession` is the
  package's only way of touching tmux externally, and it exposes just
  two methods. `RunStartupFallback` uses `*tmux.Tmux` directly because
  it's a thin delegator; `DeliverStartupPromptFallback` takes the
  interface because it has branching logic worth fake-testing.

### Usage pattern

```go
// 1. Install hooks + slash commands for a freshly-spawned polecat:
err := runtime.EnsureSettingsForRole(settingsDir, workDir, "polecat", rc)

// 2. From inside session.StartSession after tmux session is up:
info := runtime.GetStartupFallbackInfo(rc)
if info.SendBeaconNudge {
    _ = t.NudgeSession(sessionID, beaconText)
}
if len(runtime.StartupFallbackCommands("polecat", rc)) > 0 {
    _ = runtime.RunStartupFallback(t, sessionID, "polecat", rc)
}
if info.SendStartupNudge {
    time.Sleep(time.Duration(info.StartupNudgeDelayMs) * time.Millisecond)
    _ = t.NudgeSession(sessionID, runtime.StartupNudgeContent())
}

// 3. Deliver the first real prompt (e.g., a scheduled work assignment):
_ = runtime.DeliverStartupPromptFallback(t, sessionID, promptText, rc, 30*time.Second)

// 4. Elsewhere, when any code needs the agent's runtime session ID:
sessionID := runtime.SessionIDFromEnv()
```

## Related wiki pages

- [gt](../binaries/gt.md) — every spawn path funnels through this
  package.
- [internal/tmux](tmux.md) — every delivery call in this package
  dispatches to `*tmux.Tmux`. See the [nudge protocol](tmux.md#nudge-protocol-the-send-keys-machinery)
  section for what actually happens when `NudgeSession` is called.
- [internal/session](session.md) — the biggest caller.
  `session.StartSession` uses `GetStartupFallbackInfo`,
  `RunStartupFallback`, and (indirectly via session managers)
  `DeliverStartupPromptFallback`.
- [internal/config](config.md) — source of `RuntimeConfig`,
  `AgentPreset`, `RuntimeTmuxConfig`, `BuildStartupCommand*` that this
  package composes with.
- [internal/cli](cli.md) — `cli.Name()` is interpolated into the
  beacon strings so the binary name isn't hardcoded.
- Roles that trigger the fallback paths:
  [polecat](../roles/polecat.md), [crew](../roles/crew.md),
  [witness](../roles/witness.md), [refinery](../roles/refinery.md),
  [dog](../roles/dog.md) (autonomous roles get
  `&& gt mail check --inject` appended to `gt prime`);
  [mayor](../roles/mayor.md), [deacon](../roles/deacon.md) use
  plain `gt prime`.
- Commands that consume the package: [gt install](../commands/install.md)
  (provisioning), [gt prime](../commands/prime.md) (session ID),
  [gt hook](../commands/hook.md) (fallback detection),
  [gt doctor](../commands/doctor.md) (priming check).
- [go-packages inventory](../inventory/go-packages.md).

## Notes / open questions

- The package is named `runtime`, which collides with the Go stdlib
  `runtime`. `runtime.go:13` imports stdlib `runtime` only via
  `internal/tmux` transitively; this package itself has no stdlib
  `runtime` import, so the collision is contained. Still a mild
  naming smell — something like `agentruntime` would avoid
  shadowing in future refactors.
- The fallback matrix doc comment (`runtime.go:121-138`) is the only
  place that spells out the four-case cross product in one view. If
  a new capability axis appears (e.g., streaming vs.
  non-streaming prompt), this table needs updating in lockstep with
  `GetStartupFallbackInfo`. There is no schema check keeping them in
  sync.
- `StartupFallbackCommands` returns nil for hook-capable agents, but
  `RunStartupFallback` still iterates an empty slice and returns nil.
  That's fine, just noting it: callers don't need to guard.
- The removed "session-started nudge to deacon" comment
  (`runtime.go:99-102`) is a fossil worth preserving in the wiki. It's
  the kind of interaction that is invisible until it breaks, and the
  inline note is the only breadcrumb.
- `DefaultPrimeWaitMs = 2000` is hardcoded. For slow-spawning agents
  or busy machines this may not be enough; there is currently no way
  to tune it per-role or per-runtime without editing source.
