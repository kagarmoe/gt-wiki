---
title: gt checkpoint
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/cmd/checkpoint_cmd.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, diagnostics, checkpoint, crash-recovery, polecat, crew]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
---

# gt checkpoint

Writes, reads, and clears a per-session checkpoint file used by polecat
and crew sessions for crash recovery.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupDiag` ("Diagnostics")
**Polecat-safe:** no (no `AnnotationPolecatSafe` on the cobra.Command at
`/home/kimberly/repos/gastown/internal/cmd/checkpoint_cmd.go:15-32`)
**Beads-exempt:** no (not in `beadsExemptCommands`)
**Branch-check-exempt:** no (not in `branchCheckExemptCommands`)

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/checkpoint_cmd.go:15-298`.

### Invocation

```
gt checkpoint write [--notes TXT] [--molecule ID] [--step ID]
gt checkpoint read
gt checkpoint clear
```

Parent cobra.Command has no `Run` of its own (`checkpoint_cmd.go:15-32`);
all work happens in the three subcommands.

### Behavior

Checkpoints are written to `.polecat-checkpoint.json` in the session's
current working directory via the `internal/checkpoint` package
(imported at `checkpoint_cmd.go:10`). The Long description
(`checkpoint_cmd.go:19-31`) says a checkpoint captures:

- current molecule and step
- hooked bead
- modified files list
- git branch and last commit
- timestamp

#### `checkpoint write`

`runCheckpointWrite` (`checkpoint_cmd.go:83-152`):

1. Reads current working directory (`os.Getwd`, `checkpoint_cmd.go:84-87`).
2. Resolves the town root via `workspace.FindFromCwd`
   (`checkpoint_cmd.go:90-93`).
3. Resolves role context via `GetRoleWithContext(cwd, townRoot)`
   (`checkpoint_cmd.go:95-98`).
4. **Gates on role** — only `RolePolecat` and `RoleCrew` write
   checkpoints; other roles print a no-op message and return nil
   (`checkpoint_cmd.go:101-105`).
5. Calls `checkpoint.Capture(cwd)` (`checkpoint_cmd.go:108-111`) — the
   `internal/checkpoint` package handles git-state capture, modified-file
   enumeration, and timestamping.
6. Attaches notes if `--notes` was provided (`checkpoint_cmd.go:114-116`).
7. Attempts to auto-detect the molecule and step via
   `detectMoleculeContext` (`checkpoint_cmd.go:119-135`, defined at
   `checkpoint_cmd.go:222-262`), which queries the agent's assigned beads
   for an `instantiated_from:` description line.
8. Attempts to detect the hooked bead via `detectHookedBead`
   (`checkpoint_cmd.go:138-141`, defined at `checkpoint_cmd.go:265-290`)
   by listing `StatusHooked` beads for the agent.
9. Calls `checkpoint.Write(cwd, cp)` (`checkpoint_cmd.go:144-146`) to
   persist the checkpoint.
10. Prints a one-line summary from `cp.Summary()`
    (`checkpoint_cmd.go:148-151`).

#### `checkpoint read`

`runCheckpointRead` (`checkpoint_cmd.go:154-205`): calls
`checkpoint.Read(cwd)` and prints each populated field — timestamp +
age, molecule ID, step, step title, hooked bead, branch, last commit
(truncated to 12 chars), modified files count and list, notes, session
ID. Prints `"No checkpoint exists"` if `cp == nil`.

#### `checkpoint clear`

`runCheckpointClear` (`checkpoint_cmd.go:207-219`): calls
`checkpoint.Remove(cwd)` to delete the checkpoint file. Prints
`"✓ Checkpoint cleared"`.

### Subcommands

Registered in `init()` (`checkpoint_cmd.go:68-81`):

- `write` (`checkpoint_cmd.go:34-46`) — capture state
- `read` (`checkpoint_cmd.go:48-53`) — display current
- `clear` (`checkpoint_cmd.go:55-60`) — delete

### Flags

Only `checkpoint write` has flags (`checkpoint_cmd.go:73-78`):

| Flag         | Type   | Default | Purpose                                                  |
|--------------|--------|---------|----------------------------------------------------------|
| `--notes`    | string | `""`    | Free-form notes attached to the checkpoint               |
| `--molecule` | string | `""`    | Override auto-detected molecule ID                       |
| `--step`     | string | `""`    | Override auto-detected step ID                           |

## Related

- [log](log.md) — town log records `spawn`, `handoff`, `done`, `crash`
  events; checkpoints are the complementary per-session state snapshot
  that survives a crashed session.
- [doctor](doctor.md) — runs broader health checks but does not touch
  checkpoint files.
- [../binaries/gt.md](../binaries/gt.md) — parent binary and role
  detection context.
- [README.md](README.md) — command tree index.

## Notes / open questions

- Role detection (`GetRoleWithContext`) and role-context helpers
  (`RoleInfo`, `RoleContext`) are defined elsewhere in `internal/cmd/`
  and not yet documented in the wiki.
- The checkpoint file lives in the cwd of the calling process — not in
  a well-known per-session location relative to the town root. A session
  that changes directory mid-flight will have its checkpoint read/write
  operate on the new directory.
- `internal/checkpoint` package internals (Capture, Write, Read, Remove,
  `cp.WithNotes`, `cp.WithMolecule`, `cp.WithHookedBead`, `cp.Summary`,
  `cp.Age`) are an untracked package — worth a dedicated package page.
- `min` helper is redefined locally (`checkpoint_cmd.go:292-297`) — a
  Go 1.21+ builtin would suffice.
