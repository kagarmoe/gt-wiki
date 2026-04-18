---
title: gt status-line
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/cmd/statusline.go
tags: [command, ungrouped, hidden, tmux, status-bar, internal]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase5_audience: internal
phase8_audited: 2026-04-17
phase8_findings: [none]
---

# gt status-line

Hidden internal command that emits one line of tmux `status-right`
content for the current (or `--session`-specified) tmux session.
Called from tmux's `status-right` configuration on every refresh.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** (none — no `GroupID` set on the cobra.Command definition)
**Hidden:** yes (`Hidden: true` on `statusline.go:32`)
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no

> **Canonical `Use:` is `status-line`** (with a dash) per
> `statusline.go:25` — not `statusline` as the source filename
> suggests. The CLI form is `gt status-line`.

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/statusline.go:24-755`.

### Invocation

```
gt status-line [--session <name>]
```

`Use: "status-line"` (`statusline.go:25`). Produces a one-line printout
with no trailing newline — the format tmux's `status-right` expects.

### Flags

| flag | type | default | description |
|------|------|---------|-------------|
| `--session` | string | `""` | Tmux session name to query (else fall back to process env) |

Declared in `init()` on `statusline.go:36-39`.

### Dispatch (`runStatusLine`, `statusline.go:41-113`)

1. **E-stop prefix** (`statusline.go:43-64`): if the workspace has an
   active E-stop (town-wide `estop.IsActive` or per-rig
   `estop.IsRigActive(townRoot, GT_RIG)`), prepend a red
   `#[bg=red,fg=white,bold] ESTOP <HH:MM> #[default]` block to the
   output. The timestamp comes from the `estop.Info.Timestamp` read
   from the estop file.
2. **Resolve role env vars** (`statusline.go:66-85`): read `GT_RIG`,
   `GT_POLECAT`, `GT_CREW`, `GT_ISSUE`, `GT_ROLE` either from the
   target tmux session's environment (`--session` given) or from the
   process env.
3. **Role-branch to one of five renderers**:
   - `runMayorStatusLine(t)` — if `role == "mayor"` or the session
     name matches `getMayorSessionName()`.
   - `runDeaconStatusLine(t)` — likewise for deacon.
   - `runWitnessStatusLine(t, rigName)` — if `role == "witness"` or
     session ends with `-witness`.
   - `runRefineryStatusLine(t, rigName)` — likewise for refinery.
   - `runWorkerStatusLine(t, session, rigName, polecat, crew, issue)`
     — fallback for crew/polecat sessions.

Each renderer reads local beads directories and tmux session lists,
composes a pipe-separated set of parts, and prints them via
`fmt.Print(strings.Join(parts, " | ") + " |")`.

### Renderer summaries

**Crew / polecat** (`statusline.go:116-188`): agent icon + current
work (hooked bead wins over `GT_ISSUE` wins over in-progress query) +
optional mail preview (suppressed when hooked work is shown). Uses the
pane's working directory to locate rig beads.

**Mayor** (`statusline.go:190-415`): per-rig LED indicators for every
registered rig (via `config.LoadRigsConfig` → `mayor/rigs.json`),
sorted by running state + operational state + alphabetical. Shows
per-agent-type working/total counts (`X/Y 👁️` for witness, refinery).
`AgentDeacon` gets an icon but no count. Polecats are excluded — the
comment at `statusline.go:459` explains polecat sessions are "a GC
concern, not a display concern."

**Deacon** (`statusline.go:419-484`): active rig count + hooked work
or mail preview.

**Witness** (`statusline.go:489-552`): crew count in the rig + hooked
work or mail preview.

**Refinery** (`statusline.go:556-642`): merge-queue status (`merging
<id>`, `+N queued`, or `idle`) + hooked work or mail preview. Calls
`getRefineryManager(rigName)` and `mgr.Queue()` to read the MQ.

### Shared helpers defined in this file

- **`isSessionWorking(t, session)`** (`statusline.go:647-663`) —
  captures the last 5 lines of the pane and looks for the `✻` symbol
  that Claude Code prints while processing. Returns `false` for idle
  sessions (showing `❯`) or when capture fails.
- **`getMailPreviewWithRoot(identity, maxLen, townRoot)`**
  (`statusline.go:667-684`) — normalizes the identity via
  `mail.NewMailboxFromAddress`, lists unread, and returns
  `(count, truncated-first-subject)`.
- **`getHookedWork(identity, maxLen, beadsDir)`**
  (`statusline.go:688-719`) — queries the beads in `beadsDir` for
  status=`hooked` + assignee=`identity` and returns a truncated
  `<id>: <title>` string. Empty `beadsDir` means "use town root".
- **`getCurrentWork(t, session, identity, maxLen)`**
  (`statusline.go:722-754`) — queries the session's pane working
  directory for an in-progress bead assigned to `identity`.

### LED / icon helpers (referenced, not defined here)

- `GetRigLED(hasWitness, hasRefinery, opState) string` — produces
  `🟢`/`🟡`/`🔴`/`🅿️` indicators for rig-level composite status
  (called from `runMayorStatusLine`). `🛑` is reserved for error
  states (crashed agents, unreachable Dolt) per the comment on
  `statusline.go:315-317`.
- `AgentTypeIcons[AgentType]` — the icon lookup table for all agent
  roles (mayor, deacon, witness, refinery, crew, polecat).
- `categorizeSession(sessionName) *agentInfo` — parses a tmux session
  name to an agent identity struct.

### Subcommands

None.

## Related commands

- [status](status.md) — user-facing workspace status; **unrelated**
  despite the near-name. `gt status` is what a human types; `gt
  status-line` is what tmux runs on every status-right refresh.
- [mayor](mayor.md), [deacon](deacon.md), [witness](witness.md),
  [refinery](refinery.md), [crew](crew.md), [polecat](polecat.md) —
  the roles whose status-right is rendered here.
- [estop](estop.md) — the E-stop system whose active flag is the first
  thing this command checks.
- [rig](rig.md) — `config.LoadRigsConfig` and `rigs.json` shape is
  managed by the rig command family.
- [hook](hook.md), [sling](sling.md), [molecule](molecule.md) — the
  "hooked work" this command displays is the same primitive these
  commands manage.
- [mq](mq.md), [refinery](refinery.md) — refinery manager queue
  displayed by `runRefineryStatusLine`.
- [../binaries/gt.md](../binaries/gt.md) — root.

## Failure modes

No failure modes discovered. Outputs tmux status-line formatted strings. Pure string computation with no side effects or error paths.

## Notes / open questions

- **Called per status refresh.** tmux refreshes `status-right` every
  15s by default. This command runs ~4 times per minute per pane.
  Worth keeping an eye on latency — especially on refinery panes which
  open a DB to read the queue.
- **File is misnamed.** `statusline.go` defines a command whose
  canonical form is `status-line` (with a dash). Likely legacy — the
  dash was added later. Not renaming the file avoids disturbing git
  history.
- **Polecat count explicitly skipped** (`statusline.go:300`, `459`,
  `488`). The justification in the comments: polecat idle-state is
  misleading noise for the mayor, and polecat sessions are ephemeral.
- **Silent-fail everywhere.** Nearly every helper swallows errors and
  returns `nil` or an empty string — correct for a status-bar renderer
  (errors would splatter the user's tmux), but means debugging must
  be done by capturing stdout directly.
- **`hq-`-prefix assumption** for town-level sessions is noted in
  `town_cycle.go:25-30`; the mayor/deacon lookup here relies on the
  same helpers (`getMayorSessionName`, `getDeaconSessionName`).
