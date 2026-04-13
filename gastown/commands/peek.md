---
title: gt peek
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/cmd/peek.go
tags: [command, communication, tmux, capture-pane, read-only]
---

# gt peek

Capture recent terminal output from any agent session — the read
half of the `nudge` (write) / `peek` (read) agent I/O pair.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupComm` ("Communication") (`peek.go:24`)
**Polecat-safe:** no (no `AnnotationPolecatSafe` on `peekCmd`)
**Beads-exempt:** no (not in `beadsExemptCommands` at
`/home/kimberly/repos/gastown/internal/cmd/root.go:44-77`)
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/peek.go:1-122`.

### Invocation

```
gt peek <address> [count] [flags]
```

`Args: cobra.RangeArgs(1, 2)`:

- `<address>` — who to peek at (see address forms below).
- `[count]` — optional positional; overrides `-n` if both are
  provided (`peek.go:55-62`).

### Address forms

`peek.go:64-86` handles three "town-level" aliases up front via
an in-function map:

```go
townAgentSessions := map[string]string{
    "mayor":     "hq-mayor",
    "hq/mayor":  "hq-mayor",
    "deacon":    "hq-deacon",
    "hq/deacon": "hq-deacon",
    "boot":      "hq-boot",
    "hq/boot":   "hq-boot",
}
```

When the address matches, `peek` skips the rig-aware path
entirely: it verifies it's in a workspace, spins up a `tmux.NewTmux()`,
calls `t.CapturePane(sessionName, lines)`, prints the output, and
returns.

Non-town addresses take the `parseAddress(address)` branch
(`peek.go:88-102`), yielding `(rigName, polecatName)`. If the
caller is not in a rig directory **and** the address doesn't
contain `/`, the error is rewritten to a friendlier
`"not in a rig directory. Use full address format: gt peek
<rig>/<polecat>"` (`peek.go:89-94`, repeated at `98-101`).

Then on the parsed (rig, polecat):

- `polecatName` starts with `crew/`
  (`peek.go:108-112`) → strip prefix, derive the session ID via
  `session.CrewSessionName(session.PrefixFor(rigName), crewName)`,
  and capture with `mgr.CaptureSession(sessionID, lines)`.
- Otherwise (`peek.go:113`) → `mgr.Capture(polecatName, lines)`.

In all paths, the captured output is printed to stdout verbatim
(`peek.go:120`).

### Line count

Flag: `-n/--lines` (int, default 100). Registered at
`peek.go:19`. When both the flag and the positional `[count]`
are provided, the positional wins because it's assigned
into `lines` after the flag was read (`peek.go:55-62`).

### Subcommands

None.

### Flags

| flag | short | type | default | description |
|------|-------|------|---------|-------------|
| `--lines` | `-n` | int | `100` | Number of lines to capture |

### The nudge/peek pair

From the Long help at `peek.go:31-33`:

> The nudge/peek pair provides the canonical interface for
> agent sessions:
>   gt nudge - send messages TO a session (reliable delivery)
>   gt peek  - read output FROM a session (capture-pane wrapper)

`peek` is described in the same Long text (`peek.go:28-30`) as
"the ergonomic alias for `gt session capture`" — meaning there's
a lower-level `gt session capture` that `peek` shells down to.
The wrapper saves the caller from remembering session-name
construction (the `hq-mayor` / `gt-<rig>-crew-<name>` etc.
prefixing rules).

### Supported target types

From `peek.go:35-39`:

- **Polecats** — `rig/name` (e.g. `greenplace/furiosa`).
- **Crew** — `rig/crew/name` (e.g. `beads/crew/dave`).
- **Town-level** — `mayor`, `deacon`, `boot`, or the `hq/...`
  equivalents.

Notably **absent**: `witness` and `refinery`. The Long help
doesn't mention them and the `townAgentSessions` map doesn't
include them. To peek at a witness or refinery you'd have to
either address them via full rig path (if the session naming
supports it) or drop to `gt session capture`.

### Related commands

- [nudge](nudge.md) — the write partner. `peek` reads, `nudge`
  writes. Together they form the canonical agent I/O pair.
- [session](session.md) — the lower-level session-management
  command; `peek` is an ergonomic shortcut for
  `gt session capture`.
- [feed](feed.md) — event-stream reader (trail-adjacent). `feed`
  reads logged events; `peek` reads raw pane text. Different
  angles on "what is that agent doing?".
- [agents](agents.md) — enumerate agents first, then `gt peek
  <address>` to look at one.
- [trail](trail.md) — another reader-side command in the
  observability family.

## Notes / open questions

- **Read-only and side-effect-free.** `peek` does not touch
  beads, does not update notification state, does not log
  events. It's a pure read of the tmux capture-pane buffer.
  This makes it safe to run frequently.
- **No `--json` output.** Raw terminal content only. If the
  target was running a TUI with ANSI escapes, they'll come
  through as-is (or be stripped by `CapturePane` — verify in
  `internal/tmux`).
- **No way to peek historical output.** Whatever's no longer in
  the pane's scrollback buffer is gone. Compare with
  [feed](feed.md) which reads a persistent JSONL log.
- **Witness/refinery addressing gap.** If these have distinct
  session names (likely `gt-<rig>-witness`, `gt-<rig>-refinery`),
  `peek` doesn't offer an alias for them, and `rig/witness`
  would go through `parseAddress` → `mgr.Capture("witness")`
  which probably doesn't match. Worth testing.
- **`parseAddress` is shared.** The same parser appears in
  [nudge](nudge.md) (`peek.go:88` calls `parseAddress(address)`
  the same way `nudge.go` does). A single-source-of-truth
  helper; if it changes, both commands change together.
