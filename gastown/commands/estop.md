---
title: gt estop
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/cmd/estop.go
  - /home/kimberly/repos/gastown/internal/cmd/estop_unix.go
  - /home/kimberly/repos/gastown/internal/cmd/estop_windows.go
tags: [command, services, lifecycle, signals, beads-exempt, branch-check-exempt]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase5_audience: user
phase8_audited: 2026-04-17
phase8_findings: [none]
---

# gt estop

Emergency stop. Freezes Gas Town agent sessions in place by sending
`SIGTSTP` to their process groups, while preserving context (no work
is lost). The Mayor and Overseer are exempt so they can coordinate
recovery. Resume via `gt thaw`. The same source file also defines the
`gt thaw` command.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupServices` ("Services")
**Polecat-safe:** no
**Beads-exempt:** yes (in `beadsExemptCommands` on
`/home/kimberly/repos/gastown/internal/cmd/root.go:78` — must work
even when bd is wedged)
**Branch-check-exempt:** yes (in `branchCheckExemptCommands` on
`root.go:84` — must work from any branch state)

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/estop.go:27-226`.

Original implementation by `outdoorsea` in upstream PR #3237; the
file header (`estop.go:1-4`) notes this is the cherry-picked version
**without daemon auto-trigger** — manual invocation only.

### Invocation

```
gt estop                              # Freeze the whole town
gt estop -r "closing laptop"          # With a reason
gt estop --rig gastown                # Freeze only one rig
gt estop --rig beads -r "maintenance" # Both
```

`gt thaw` (defined in the same file) is the inverse:

```
gt thaw                    # Thaw everything
gt thaw --rig gastown      # Thaw only one rig
```

### Flow — town-wide estop

`runEstop` at `estop.go:75-120`:

1. Resolve town root via `workspace.FindFromCwdOrError`.
2. If `--rig` set, dispatch to `runEstopRig` (see below).
3. If `estop.IsActive(townRoot)`, report the existing trigger and
   reason and **exit cleanly** (idempotent).
4. **Create the sentinel file first** via `estop.Activate(townRoot,
   estop.TriggerManual, estopReason)`. The comment at `estop.go:95-98`
   is explicit: *"the sentinel file is the source of truth."* Even if
   tmux is unavailable, the file is the primary authority.
5. If tmux is unavailable: warn and return — file is created, no
   sessions to freeze.
6. Call `freezeAllSessions(t, townRoot, "")` to send the freeze
   signal to every Gas Town session.
7. Print `N session(s) frozen` and the resume incantation.

### Flow — per-rig estop

`runEstopRig` at `estop.go:122-154`. Mirrors the town-wide flow but
uses `estop.IsRigActive` / `estop.ReadRig` / `estop.ActivateRig` /
`estop.DeactivateRig` and passes a `rigFilter` into the freeze loop.

### `freezeAllSessions`

`estop.go:237-265`. Process-group SIGTSTP (`sigFreeze`, defined
elsewhere in the package) sent via `signalSessionGroup`. Iterates
sessions returned by `collectGTSessions`:

- **Exempt sessions** (`exemptSessions` map at `estop.go:229-232`):
  - `session.MayorSessionName()`
  - `session.OverseerSessionName()`
  - Reported as "exempt" in the output.
- **Rig filter**: when `rigFilter` is set, only sessions whose name
  matches `<rigPrefix>-…` (or equals `rigPrefix`) are frozen, via
  `isRigSession` (`estop.go:321-323`).
- Each successfully frozen session is printed with a paused glyph;
  failures with a warning glyph.

### `collectGTSessions`

`estop.go:326-345`. Lists all tmux sessions via `t.ListSessions`,
discovers rigs via `discoverRigs(townRoot)`, builds a set of rig
prefixes, and includes any session that:

- starts with `session.HQPrefix` (town-level), OR
- starts with `<rigPrefix>-` or equals `<rigPrefix>` for any rig.

### Thaw flow

`runThaw` at `estop.go:156-195` (and `runThawRig` at `:197-226`):

1. Resolve town root.
2. If `--rig` set, dispatch to `runThawRig`.
3. If no E-stop is active, print `No E-stop active.` and return.
4. `thawAllSessions` (`estop.go:269-292`) sends `sigThaw` (SIGCONT)
   via `signalSessionGroup`, skipping exempt and (optionally)
   non-matching-rig sessions.
5. `nudgeAllSessions` (`estop.go:296-318`) sends an in-band tmux
   nudge to each thawed session: *"E-stop cleared. Work may resume."*
6. `estop.Deactivate(townRoot, false)` removes the sentinel file.
7. Reports the duration the E-stop was active (rounded to seconds via
   `time.Since(info.Timestamp).Round(time.Second)`).

### Flags

`estop.go:67-73`:
- `gt estop --reason / -r` (string)
- `gt estop --rig` (string)
- `gt thaw --rig` (string)

### Side effect: status integration

`addEstopToStatus` (`estop.go:367-398`) is called from `gt status` to
print an `E-STOP ACTIVE` banner with trigger, age, and reason. Also
scans for per-rig `ESTOP.<rig>` sentinel files in the town root via
`filepath.Glob` and prints a separate line for each.

### Notable design decisions

- **File first, signals second**: the sentinel file is the source of
  truth. Even if no sessions exist or tmux is broken, the file alone
  marks the town as frozen — and `gt status` / agent startup logic
  must check it before doing work.
- **Mayor/overseer exempt**: required so a human (or the Mayor) can
  drive recovery without un-freezing themselves first.
- **`SIGTSTP`, not `SIGSTOP`**: SIGTSTP is catchable. Process groups
  receive it via the tmux pane PGID — preserves context inside the
  agent process tree.
- **Manual only**: this command does not auto-trigger from daemon
  health checks (the deliberate divergence from upstream PR #3237).

## Related

- [thaw](./thaw.md) — defined in the same source file; the resume
  primitive (page may not yet exist; if missing, follow the source
  references in this page for now)
- [down](./down.md) — *stops* services entirely; estop *freezes* them
  with context preserved
- [doctor](./doctor.md) — should report E-stop state in diagnostics
- [status](./status.md) — calls `addEstopToStatus`
- [mayor](./mayor.md), [witness](./witness.md) — exempt sessions
  during E-stop
- [../packages/estop.md](../packages/estop.md) — the sentinel-file
  primitives (town-wide and per-rig) backing this command.

## Failure modes

No failure modes discovered. `estop.go` writes a sentinel file and sends SIGTSTP to tmux sessions. Sentinel-first design ensures E-stop state is durable even if session freezing fails. All errors propagated. SIGTSTP delivery failures per-session are non-fatal (some sessions may be gone).

## Notes / open questions

- **`thaw.md` page**: `gt thaw` lives in this same file. A separate
  page is warranted, since thaw is an independent top-level command.
  This batch only owns `estop`; `thaw` is on the Sub B (lifecycle)
  list.
- **`signalSessionGroup`, `sigFreeze`, `sigThaw`** are defined
  elsewhere in `internal/cmd` — likely a small helper file
  (`signal.go` or similar). Worth a follow-up grep.
- **Per-rig sentinels live in the town root** as `ESTOP.<rig>` files
  (per `addEstopToStatus`). Town-wide sentinel naming TBD — likely
  `ESTOP` or in the `estop` package.
- **`estop.TriggerManual` vs other triggers**: the package supports
  multiple trigger kinds (manual, auto, …) but only `TriggerManual`
  is used here. Worth confirming what other triggers exist and
  whether daemon-trigger gets re-introduced upstream.
- **Idempotency edge**: `gt estop` on an already-active estop is a
  no-op (`estop.go:86-93`). `gt thaw` on an inactive estop is also a
  no-op (`estop.go:167-170`). Both are safe to run repeatedly.
