---
title: gt prime
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/cmd/prime.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, diagnostics, role, session-start, hook, polecat-safe, formula, claude-code]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
phase5_audience: agent
phase8_audited: 2026-04-17
phase8_findings: [precondition]
---

# gt prime

Detects the agent's role from the current directory (or `GT_ROLE` env
var), renders the role's context formula to stdout, and performs
session setup side effects: identity lock, beads redirect,
session-start event emission, mail injection, hook/work resolution,
and startup directive output. Also acts as the `SessionStart` /
`PreCompact` hook for Claude Code, Gemini CLI, and other LLM
runtimes.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupDiag` ("Diagnostics")
**Polecat-safe:** **yes** — annotated
`AnnotationPolecatSafe: "true"` at
`/home/kimberly/repos/gastown/internal/cmd/prime.go:61`.
**Beads-exempt:** **yes** — in `beadsExemptCommands` at
`/home/kimberly/repos/gastown/internal/cmd/root.go:56`; prime must
run before `bd` is usable because prime bootstraps the beads
redirect.
**Branch-check-exempt:** no (not in `branchCheckExemptCommands` on
`root.go:82-91`)

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/prime.go:58-1291`
(file is 1291 lines; key control flow is in the first ~400).

### Invocation

```
gt prime [--hook] [--dry-run] [--state [--json]] [--explain]
```

### Top-level flow

`runPrime` (`prime.go:116-219`). Wrapped in
`telemetry.RecordPrime(... retErr)` via a deferred closure — the
telemetry call is a no-op unless OTEL endpoints are configured (see
[internal/telemetry](../packages/telemetry.md)).

1. `validatePrimeFlags()` — rejects illegal combinations
   (`prime.go:256-264`). `--state` can't combine with
   `--hook/--dry-run/--explain`; `--json` requires `--state`.
2. `resolvePrimeWorkspace()` — resolves cwd + town root
   (`prime.go:267-289`). When not in a workspace *and*
   `state.IsEnabled()` is false, returns empty `townRoot` and
   `runPrime` exits silently with `nil` (the "Discover, Don't Track"
   principle, comment at `prime.go:279-280`).
3. If `primeHookMode`: `handlePrimeHookMode(townRoot, cwd)`
   (`prime.go:293-317`). Reads session ID from stdin JSON or env,
   persists it to `.runtime/session_id` in both town root and cwd,
   exports `GT_SESSION_ID` / `CLAUDE_SESSION_ID`, signals agent
   readiness via `tmux set-environment GT_AGENT_READY=1`, and
   emits `[session:<id>]` / `[source:<src>]` beacon lines.
4. `checkHandoffMarker(cwd)` or `checkHandoffMarkerDryRun(cwd)` —
   reads a handoff marker (used to break handoff loops). Sets
   `primeHandoffReason` when the marker carries a `reason` field
   (`prime.go:134-139`).
5. `GetRoleWithContext(cwd, townRoot)` — resolves the role; the
   result carries `Role`, `Rig`, `Polecat`, `Home`, `CwdRole`,
   `Mismatch` (`prime.go:141-144`).
6. `warnRoleMismatch(roleInfo, cwd)` — if `GT_ROLE` disagrees with
   the cwd-detected role, prints a bold
   `⚠️  ROLE/LOCATION MISMATCH` banner (`prime.go:363-378`).
7. **`--state` mode**: calls `outputState(ctx, primeStateJSON)` and
   returns (`prime.go:157-160`). No side effects.
8. **Compact/resume fast path** (`prime.go:167-170, 230-253`): if
   `isCompactResume()` returns true (source is `compact`/`resume`,
   or the handoff marker's reason was `compaction`), calls
   `runPrimeCompactResume(ctx)` which:
   - Prints a brief `Recovery: Context <source> complete. You are
     <actor> (<role>).` line.
   - Outputs session metadata (`outputSessionMetadata(ctx)`).
   - Prints "Continue your current task. If you've lost context,
     run `gt prime` for full reload."
   - For `RolePolecat`, reminds the agent to run [`gt done`](done.md) when
     work is complete (comment at `prime.go:248-250` cites GH#1965).
   - **Skips** `setupPrimeSession`, `findAgentWork`, and the full
     role context rendering, keeping the hook under 1 s for
     runtimes with short hook timeouts (Gemini CLI).
9. `setupPrimeSession(ctx, roleInfo)` (`prime.go:382-395`): in
   non-dry-run, acquires the identity lock
   (`acquireIdentityLock`), ensures the beads redirect is wired
   (unless role mismatched), repairs session env vars via
   `repairSessionEnv`, and emits a `session_start` event via
   `emitSessionEvent`.
10. `findAgentWork(ctx)` — resolves the agent's hooked work bead
    (`prime.go:180-190`). **Critical**: a database error here is
    distinct from "no work assigned". On error, prime prints a
    bold banner:
    ```
    ## ⚠️  DATABASE ERROR — DO NOT RUN gt done ⚠️
    ```
    then the error, plus instructions to escalate to witness/mayor
    and NOT close any beads. This prevents the gt-done-on-db-error
    cycle documented as GH#2638.
11. `injectWorkContext(ctx, hookedBead)` (`prime.go:191`) — sets
    `GT_WORK_RIG`, `GT_WORK_BEAD`, `GT_WORK_MOL` in the current
    process env *and* in the tmux session environment so all
    subprocess `bd`/`mail`/etc. inherit the correct work
    attribution until the next `gt prime` overwrites it. Comment
    at `prime.go:176-179` calls this "P0" attribution.
12. `outputRoleContext(ctx)` (`prime.go:193-196`) — renders the
    role's formula to stdout. Returns the rendered formula string,
    which is also logged via
    `telemetry.RecordPrimeContext(ctx, formula, ...)` so operators
    can see exactly what context each agent started with in
    VictoriaLogs (`prime.go:197-200`).
13. `checkSlungWork(ctx, hookedBead)` — detects hooked/in-progress
    work that means the agent should go into autonomous mode
    (`prime.go:202-203`).
14. Further context outputs (`prime.go:205-207`):
    `outputMoleculeContext(ctx)`, `outputCheckpointContext(ctx)`,
    `runPrimeExternalTools(cwd)`.
15. For `RoleMayor`: `checkPendingEscalations(ctx)` shows
    unresolved escalation beads (`prime.go:209-211`).
16. If **no slung work**: `outputStartupDirective(ctx)` prints the
    role's normal-mode startup instructions (`prime.go:213-216`).

### Hook mode — `handlePrimeHookMode`

Source: `prime.go:293-317`.

Session ID resolution cascade (from the `Long` text at
`prime.go:78-83`):

1. `GT_SESSION_ID` env var
2. `CLAUDE_SESSION_ID` env var
3. Persisted `.runtime/session_id`
4. Stdin JSON (Claude Code format)
5. Auto-generated UUID

Source resolution: `GT_HOOK_SOURCE` env var, then stdin JSON
`source` field. Sources are `startup`, `resume`, `clear`, `compact`.

**Claude Code integration** (`Long` at `prime.go:87-89`):

```json
"SessionStart": [{"hooks": [{"type": "command", "command": "gt prime --hook"}]}]
```

Claude sends `{"session_id":"uuid","source":"startup|resume|compact"}`
on stdin.

**Gemini CLI integration** (`Long` at `prime.go:91-94`): uses env
var variants instead of stdin:

```json
"SessionStart": "export GT_SESSION_ID=$(uuidgen) GT_HOOK_SOURCE=startup && gt prime --hook"
"PreCompress":  "export GT_HOOK_SOURCE=compact && gt prime --hook"
```

### Role detection

Defined at `prime.go:43-56`. The `Role` type and constants:

```go
const (
    RoleMayor    Role = "mayor"
    RoleDeacon   Role = "deacon"
    RoleBoot     Role = "boot"
    RoleWitness  Role = "witness"
    RoleRefinery Role = "refinery"
    RolePolecat  Role = "polecat"
    RoleCrew     Role = "crew"
    RoleDog      Role = "dog"
    RoleUnknown  Role = "unknown"
)
```

From the `Long` text (`prime.go:63-71`):

- Town root → Neutral, no role inferred (fall back to `GT_ROLE`).
- `mayor/` or `<rig>/mayor/` → Mayor.
- `<rig>/witness/rig/` → Witness.
- `<rig>/refinery/rig/` → Refinery.
- `<rig>/polecats/<name>/` → Polecat.

Crew/dog/deacon detection is handled by the shared `GetRole`
helper elsewhere in the cmd package.

### Subcommands

None (terminal command).

### Flags

Defined in `init()` (`prime.go:98-109`):

| Flag            | Type | Default | Purpose                                                         |
|-----------------|------|---------|-----------------------------------------------------------------|
| `--hook`        | bool | `false` | Hook mode: read session ID from stdin JSON or env               |
| `--dry-run`     | bool | `false` | Show what would be injected (no marker removal, no bd, no mail) |
| `--state`       | bool | `false` | Output only detected session state (normal/post-handoff/...)    |
| `--json`        | bool | `false` | JSON output for `--state`                                       |
| `--explain`     | bool | `false` | Show why each section was included                              |

`primeStructuredSessionStartOutput` exists as an internal variable
but has no CLI flag — it's used to suppress `[session:...]` /
`[source:...]` beacons when Codex's auto-JSON-detection would
misclassify them (`prime.go:320-331`).

### Environment side effects

Prime mutates the caller's process and session environment (search
for `os.Setenv` / `SetEnvironment`):

- `GT_SESSION_ID`, `CLAUDE_SESSION_ID` — persisted session UUID
- `GT_AGENT_READY=1` via `tmux.SetEnvironment` (signals readiness
  to `WaitForCommand` callers) (`prime.go:340-347`)
- `GT_WORK_RIG`, `GT_WORK_BEAD`, `GT_WORK_MOL` — work attribution
  for downstream subprocess calls (`injectWorkContext`,
  `prime.go:191`)

### Autonomous mode

When `findAgentWork` returns a hooked bead and `checkSlungWork`
confirms it, prime emits the role's `AUTONOMOUS WORK MODE` block
(inside `outputRoleContext` / downstream formula rendering). When
no hook exists, it falls through to `outputStartupDirective` with
the normal-mode prompt. Explain lines are added via
`explain(cond, msg)` (`prime.go:202-216`) when `--explain` is set.

## Related

- [seance](seance.md) — seance discovers `session_start` events
  that prime emits via `emitSessionEvent`.
- [doctor](doctor.md) — `NewPrimingCheck` verifies prime's
  post-install migration (`doctor.go:244` area). Doctor is a
  consumer of prime's work.
- [checkpoint](checkpoint.md) — checkpoint writes context that
  prime's `outputCheckpointContext` will read and render on the
  next session start.
- [info](info.md) — info is a lightweight introspection that does
  not mutate process environment; prime is the heavy session-setup
  equivalent.
- [status](status.md) — status is polecat-safe like prime, and
  consumes the `GT_ROLE` / work attribution that prime injects.
- [audit](audit.md) — audit reads `session_start` events that
  prime emits.
- [heartbeat](heartbeat.md) — heartbeat consumes the same identity
  and work-context env vars prime sets up.
- [upgrade](upgrade.md) — `NewPrimingCheck` is in upgrade's check
  set (`upgrade.go:148`); upgrade can trigger a re-prime via
  doctor fix.
- [../binaries/gt.md](../binaries/gt.md) — parent binary; prime
  also runs inside `persistentPreRun` as the universal session
  bootstrap.
- [README.md](README.md) — command tree index.

## Failure modes

### Precondition violations (what does it assume?)
- **Database error during hook query triggers safety banner:**
  `findAgentWork` at `prime.go:182-190` detects database connectivity
  errors and emits a loud "DO NOT RUN gt done" banner to stderr. This
  is **present** — the code explicitly handles the DB-error case to
  prevent the destructive cycle documented in GH#2638 (DB error -> "no
  work" -> gt done -> bead lost).
- **Identity lock assumes writable filesystem:** `setupPrimeSession`
  (called at `prime.go:172`) performs identity lock, beads redirect,
  and event emission. Filesystem permission errors in the town root
  would cause hard failures. **Absent** — no pre-check for write
  access.

## Notes / open questions

- The file is 1291 lines; only the top ~400 cover control flow.
  The remainder (not read in this pass) covers: formula rendering,
  hook resolution, handoff marker management, external tools,
  mail injection, escalation checks, and beads redirect wiring.
  Each of those is worth its own note.
- `GetRoleWithContext` and `GetRole` live in a separate file in the
  cmd package (likely `role.go`); this page does not document their
  resolution rules.
- `outputRoleContext` / `outputStartupDirective` read embedded
  formula content from `internal/formula/` (same path used by
  `patrol new` and `patrol report`). Documenting the formula format
  deserves its own page.
- The GH#2638 "database error → gt done → bead lost" cycle is
  prime's only load-bearing error banner; other prime errors fall
  through to cobra's default handling. The contrast is deliberate.
- `primeStructuredSessionStartOutput` has no CLI flag — it's
  controllable only via code. A Codex-specific structured-output
  wrapper presumably flips it.
- The `isCompactResume` short-circuit (`prime.go:358-360`) skips
  both Dolt queries and role context rendering — the fast path
  exists to stay under Gemini CLI's hook timeout, but there is no
  documented upper bound for that timeout.
- `state.IsEnabled()` at `prime.go:282` governs the "not in a
  workspace" silent-exit behavior. The `internal/state/` package
  is the ground truth for the "enabled outside a workspace" flag.
