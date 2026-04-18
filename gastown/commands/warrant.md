---
title: gt warrant
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/cmd/warrant.go
tags: [command, ungrouped, lore, termination, escalation, triage]
phase3_audited: 2026-04-15
phase3_findings: [cobra-drift]
phase3_severities: [wrong]
phase3_findings_post_release: false
phase5_audience: dev
phase8_audited: 2026-04-17
phase8_findings: [none]
---

# gt warrant

File, list, and execute "death warrants" for stuck or unresponsive
agents. Warrants are JSON files in `<townRoot>/warrants/` that Boot
picks up and executes during triage cycles.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** (none — no `GroupID` set on the cobra.Command definition)
**Hidden:** no
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/warrant.go:38-401`.

### Invocation

```
gt warrant file <target> --reason "..."     # or --stdin
gt warrant list [--all]
gt warrant execute <target> [--force]
```

`Use: "warrant"` (`warrant.go:39`). The parent command has no `RunE`
— it's a pure subcommand group. Three subcommands, all registered in
`init()` on `warrant.go:115-119`.

### Concept

From the `Long` help on `warrant.go:40-52`:

> Manage death warrants for agents that need termination.
>
> Death warrants are filed when an agent is stuck, unresponsive, or
> needs forced termination. Boot handles warrant execution during
> triage cycles.
>
> The warrant system provides a controlled way to terminate agents:
> 1. Deacon/Witness files a warrant with a reason
> 2. Boot picks up the warrant during triage
> 3. Boot executes the warrant (terminates session, updates state)
> 4. Warrant is marked as executed
>
> Warrants are stored in ~/gt/warrants/ as JSON files.

(The help says `~/gt/warrants/` but `getWarrantDir` on
`warrant.go:123-129` actually returns `<townRoot>/warrants/`.)

### The `Warrant` struct (`warrant.go:28-36`)

```go
type Warrant struct {
    ID         string     `json:"id"`
    Target     string     `json:"target"`  // e.g. "gastown/polecats/alpha"
    Reason     string     `json:"reason"`
    FiledBy    string     `json:"filed_by"`
    FiledAt    time.Time  `json:"filed_at"`
    Executed   bool       `json:"executed,omitempty"`
    ExecutedAt *time.Time `json:"executed_at,omitempty"`
}
```

ID is `warrant-<unix-millis>`. `FiledBy` is `$BD_ACTOR` or `"unknown"`.

### Subcommands

#### `gt warrant file <target>` (`warrant.go:55-70`, run on `warrant.go:138-211`)

`Use: "file <target>"`. `Args: cobra.ExactArgs(1)`.

Flags:

| flag | short | type | default | description |
|------|-------|------|---------|-------------|
| `--reason` | `-r` | string | `""` | Reason for the warrant (required unless `--stdin`) |
| `--stdin` | — | bool | `false` | Read reason from stdin (avoids shell quoting issues) |

Behavior:

1. **Stdin mode** (`warrant.go:140-149`): if `--stdin`, require that
   `--reason` wasn't also passed (error: `cannot use --stdin with
   --reason/-r`), then `io.ReadAll(os.Stdin)` and trim trailing `\n`.
2. **Require reason** (`warrant.go:152-154`).
3. **Resolve warrant directory** = `<townRoot>/warrants/` via
   `getWarrantDir()`. Create with 0755 if missing
   (`warrant.go:163-166`).
4. **Short-circuit on duplicate** (`warrant.go:169-180`): if
   `<dir>/<sanitized-target>.warrant.json` already exists and the
   existing warrant is not executed, print the existing warrant's
   reason and filed-at timestamp and return nil. A second `gt warrant
   file` call does not overwrite a pending warrant.
5. **Build warrant** with ID = `warrant-<unix-millis>`, `FiledBy =
   os.Getenv("BD_ACTOR")` (defaults to `"unknown"`), current time.
6. **Write** `json.MarshalIndent`-formatted warrant to the path
   returned by `warrantFilePath(dir, target)` (`warrant.go:131-136`):
   `/` in the target is replaced with `_` so the filename is safe.
7. **Print** `✓ Filed death warrant for <target>` with reason and ID.

#### `gt warrant list` (`warrant.go:72-83`, run on `warrant.go:213-276`)

`Use: "list"`. Flag: `--all`/`-a` (bool) to include executed warrants
(`warrant.go:110`).

1. Read the warrant directory. If it doesn't exist, print `No
   warrants filed` and return.
2. Iterate `.warrant.json` files, parse each as a `Warrant`, include
   if `warrantListAll || !Executed`.
3. Print header `Death Warrants`, then per-entry: `⚠️ PENDING` or
   `✓ EXECUTED` status, target, reason, filed-at + filed-by, and
   optional executed-at timestamp.

#### `gt warrant execute <target>` (`warrant.go:85-102`, run on `warrant.go:278-332`)

`Use: "execute <target>"`. `Args: cobra.ExactArgs(1)`. Flag:
`--force`/`-f` (bool) to execute even when no warrant file exists
(`warrant.go:113`).

1. **Load warrant file** from `<dir>/<target>.warrant.json` if present
   (`warrant.go:290-295`).
2. **Require warrant or `--force`** (`warrant.go:297-299`).
3. **Idempotent on executed** (`warrant.go:301-304`): if already
   executed, print the timestamp and return nil.
4. **With warrant**: call `executeOneWarrant(warrant, warrantPath,
   tm)` (`warrant.go:338-370`) which:
   - Converts target to tmux session name via `targetToSessionName`.
   - Calls `tm.HasSession(name)`.
   - If present, `tm.KillSessionWithProcesses(name)` — kills the
     tmux session including the full process tree.
   - Mark the warrant executed on disk (`Executed = true`,
     `ExecutedAt = now`) and rewrite the JSON file.
   - On error, **the warrant is NOT marked executed** so it can be
     retried on the next triage cycle (`warrant.go:336-337` comment).
5. **With `--force` but no warrant** (`warrant.go:312-327`): just
   compute the session name, kill if present, and print "already dead"
   otherwise. No warrant file is created or touched.

### `targetToSessionName` (`warrant.go:373-401`)

Maps target paths to tmux session names via five cases:

| target form | session name | |
|---|---|---|
| `gastown/polecats/alpha` | `session.PolecatSessionName(PrefixFor(gastown), alpha)` | `warrant.go:378-380` |
| `gastown/crew/bob` | `session.CrewSessionName(...)` | `warrant.go:381-383` |
| `gastown/witness` | `session.WitnessSessionName(...)` | `warrant.go:384-385` |
| `gastown/refinery` | `session.RefinerySessionName(...)` | `warrant.go:386-388` |
| `deacon/dogs` (2-part) | error: `need dog name` | `warrant.go:389-390` |
| `deacon/dogs/alpha` | `hq-dog-alpha` | `warrant.go:391-393` |
| default | `<prefix>-<target-with-/-replaced-by-->` | `warrant.go:394-400` |

### Flags (summary)

**`file`:** `--reason`/`-r`, `--stdin`.
**`list`:** `--all`/`-a`.
**`execute`:** `--force`/`-f`.

## Docs claim

### Source
- `/home/kimberly/repos/gastown/internal/cmd/warrant.go:40-52` — Cobra Long text

### Verbatim
> Manage death warrants for agents that need termination.
>
> Death warrants are filed when an agent is stuck, unresponsive, or needs
> forced termination. Boot handles warrant execution during triage cycles.
>
> The warrant system provides a controlled way to terminate agents:
> 1. Deacon/Witness files a warrant with a reason
> 2. Boot picks up the warrant during triage
> 3. Boot executes the warrant (terminates session, updates state)
> 4. Warrant is marked as executed
>
> Warrants are stored in ~/gt/warrants/ as JSON files.

## Drift

### Long text hardcodes `~/gt/warrants/` but code uses `<townRoot>/warrants/`
- **Claim source:** Cobra Long text at `warrant.go:52`
- **Docs claim:** "Warrants are stored in ~/gt/warrants/ as JSON files."
- **Code does:** `getWarrantDir()` at `warrant.go:122-128` returns `filepath.Join(townRoot, "warrants")` where `townRoot` is resolved dynamically via `workspace.FindFromCwd()`. For the default installation `townRoot` happens to be `~/gt`, but the path is workspace-relative, not hardcoded. A non-default `$GT_ROOT` installation would store warrants elsewhere.
- **Category:** `cobra drift`
- **Severity:** `wrong`
- **Fix tier:** `code` — change `~/gt/warrants/` to `<town-root>/warrants/` in the Long text
- **Release position:** `in-release` — `getWarrantDir()` exists at `v1.0.0`

→ [gastown/drift/README.md](../drift/README.md)

## Related commands

- [boot](boot.md) — the "triage" component that the Long help says
  picks up and executes warrants in the background. Worth verifying
  whether boot actually reads `<townRoot>/warrants/` or whether this
  handoff has drifted.
- [escalate](escalate.md) — complementary escalation primitive for
  Dolt/infra problems. Warrants target individual stuck agents;
  escalate targets systemic issues.
- [mail](mail.md) — escalations sometimes flow as mail; warrants are
  a separate on-disk path.
- [deacon](deacon.md), [witness](witness.md) — the roles that file
  warrants per the Long text.
- [reaper](reaper.md) — may be the counterpart cleanup command for
  expired warrants / dead sessions.
- [estop](estop.md) — town-wide stop; warrants are targeted stops.
- [polecat](polecat.md), [crew](crew.md), [dog](dog.md) — the targets
  of `targetToSessionName`.
- [../binaries/gt.md](../binaries/gt.md) — root.

## Failure modes

No failure modes discovered. `warrant.go` creates, lists, shows, and executes death warrants for agent sessions. Warrant creation writes a JSON file; execution kills sessions. All errors propagated. Warrant files are idempotent (re-execution skips already-executed warrants).

## Notes / open questions

- **Location drift in the help text.** → promoted to `## Drift`.
  `Long` says `~/gt/warrants/` but `getWarrantDir` returns
  `<townRoot>/warrants/`.
- **No cleanup of executed warrants.** `gt warrant list` without
  `--all` hides them, but the files stay. A rolling `--older-than`
  or `prune` subcommand may be worth adding.
- **`targetToSessionName` is the wrong abstraction boundary.** It's
  a private helper here but duplicates logic from
  [internal/session](../packages/session.md). A future refactor
  should push this into `session` itself.
- **Boot-triage coupling is informal.** The help text declares that
  Boot executes warrants, but this file contains no pointer to the
  consumer. A `seance`/`reaper`/`boot` page should list this as a
  data dependency.
- **Race between `file` and `execute`.** If a warrant is filed and
  immediately executed by Boot, a second `gt warrant file` call
  targeting the same agent will succeed (because the executed
  warrant's duplicate check only short-circuits on non-executed files
  — `warrant.go:174`). That's arguably correct: a freshly stuck agent
  after revival deserves a fresh warrant.
- **`--stdin` avoids shell quoting** — a nice pattern for any command
  that takes human-written prose as a required field (reasons,
  messages, mail bodies).
- **Lore naming.** The batch brief calls out "warrant" as a lore name.
  "Death warrant" and "triage cycle" fit the law-enforcement naming
  that runs through [witness](witness.md), [mayor](mayor.md),
  [deacon](deacon.md), and [polecats](polecat.md).
