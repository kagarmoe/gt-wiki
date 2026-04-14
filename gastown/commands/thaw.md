---
title: gt thaw
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/cmd/estop.go
tags: [command, services, estop, signals, sigcont, nudge]
---

# gt thaw

Resume agent sessions that were frozen by [estop](./estop.md). Sends
`SIGCONT` to every previously-frozen session, removes the ESTOP
sentinel file, and then nudges each thawed session to tell it that
work may resume. Shares a source file with `estopCmd` — the two
commands are a tight pair and should be read together.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupServices` ("Services")
**Polecat-safe:** no
**Beads-exempt:** yes (in `beadsExemptCommands` — thaw must work
when the bead DB is down)
**Branch-check-exempt:** yes (in `branchCheckExemptCommands` — thaw
must not warn on non-main town root)

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/estop.go:52-65`,
`runThaw` at `estop.go:156-195`, `runThawRig` at `estop.go:197-226`.

### Invocation

```
gt thaw                  # Thaw the whole town
gt thaw --rig <name>     # Thaw only one rig
```

### Town-wide thaw (`runThaw`)

1. `workspace.FindFromCwdOrError` — must be in a Gas Town workspace.
2. If `--rig` is set, delegates to `runThawRig` immediately.
3. Checks `estop.IsActive(townRoot)`. If no E-stop sentinel file is
   present, prints "No E-stop active." and returns nil — **this is
   not an error**; a second `gt thaw` after the first one cleans up
   is a no-op.
4. Reads the ESTOP file via `estop.Read` to remember the trigger
   and reason for the final duration log line.
5. If `tmux.IsAvailable`:
   - `thawAllSessions(t, townRoot, "")` — sends `sigThaw` (defined
     elsewhere in the `cmd` package; on POSIX this is `SIGCONT`) to
     every Gas Town session via `signalSessionGroup`, skipping
     `exemptSessions` (Mayor + Overseer — see `estop.go:229-232`,
     they were never frozen in the first place). Returns the count
     of thawed sessions.
   - `nudgeAllSessions(t, townRoot, "")` — sends the notification
     `"E-stop cleared. Work may resume."` via `tmux.NudgeSession` to
     every non-exempt Gas Town session. Nudge failures are
     swallowed per-session.
6. `estop.Deactivate(townRoot, false)` — removes the ESTOP sentinel.
   The trailing `false` argument is the "silent" flag; passing
   false here means a normal (non-silent) deactivation.
7. If we captured a prior `info`, prints:
   `E-stop was active for <duration>` (rounded to seconds).

### Per-rig thaw (`runThawRig`)

`estop.go:197-226`. Same shape, but:
- `estop.IsRigActive(townRoot, rigName)` instead of `IsActive`
- `estop.ReadRig` instead of `Read`
- `thawAllSessions(t, townRoot, rigName)` filters by the rig's
  session prefix via `session.PrefixFor(rigName)` and `isRigSession`
  (`estop.go:321-323`: "starts with `<prefix>-`" or equals the bare
  prefix).
- `nudgeAllSessions(t, townRoot, rigName)` — same rig filter.
- `estop.DeactivateRig(townRoot, rigName)` — removes the per-rig
  ESTOP sentinel.

### Session enumeration — `collectGTSessions`

`estop.go:326-345`. For both freeze and thaw paths:

1. `tmux.ListSessions` enumerates every tmux session.
2. `discoverRigs(townRoot)` (the same helper used by [up](./up.md))
   produces the rig list, and each rig is turned into a prefix via
   `session.PrefixFor`.
3. `isGTSession` (`estop.go:348-363`) classifies a session as
   Gas Town if its name starts with `session.HQPrefix` (town-level)
   *or* starts with `<rigPrefix>-` *or* equals a bare rig prefix.

This means sessions for rigs that are no longer in `rigs.json` (or
visible under the town root) will be *missed* by both freeze and
thaw — a rig removed mid-freeze leaves its sessions stuck in the
`SIGTSTP` state.

### Flags

`estop.go:67-73`:
- `--rig <name>` — thaw only the named rig (string, default empty =
  thaw the whole town)

## Notes / open questions

- **Mayor + Overseer exemption** — Mayor is always excluded by
  `exemptSessions` (`estop.go:229-232`) so the town coordinator can
  run recovery. Overseer likewise. The corollary is that E-stop and
  thaw are both no-ops against those sessions, so a buggy Mayor
  cannot be frozen via this tool.
- **`sigFreeze` / `sigThaw`** are defined elsewhere in the `cmd`
  package — grep them up when the signal-helpers module gets a page.
  On Unix these are `SIGTSTP` and `SIGCONT`; Windows support is
  unclear from this file alone.
- **Beads-exempt + branch-check-exempt** — this pair makes thaw
  usable during a cascading outage. If the Dolt server is down or
  you're on a maintenance branch, `gt thaw` still works. Kept
  parallel with [estop](./estop.md) on purpose.
- **`estop.Deactivate(..., false)` vs `true`** — the boolean is a
  "silent" flag. Worth grounding in an `internal/estop` package
  page.
- **`signalSessionGroup` errors are silently swallowed in thaw**
  (`estop.go:285-287`) but reported for freeze. Thaw errors might
  leave processes permanently stopped.
- **No confirmation prompt** — thaw is assumed to be always safe. A
  user running `gt thaw` on a rig that was never frozen just gets
  "No E-stop active" and exits 0.

## Related

- [estop](./estop.md) — the opposite direction; same source file
  and same session enumeration. Read these two pages together.
- [down](./down.md) — full shutdown (not a pause); stops processes
  entirely rather than freezing them
- [mayor](./mayor.md) — exempt from freeze, so a thaw is idempotent
  with respect to Mayor
- [status](./status.md) — surfaces E-stop state via
  `addEstopToStatus` in the same source file
  (`estop.go:367-398`)
- [../packages/estop.md](../packages/estop.md) — the shared sentinel-file
  primitives that both `estop` and `thaw` manipulate.
