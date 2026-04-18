---
title: gt bead
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/cmd/bead.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, work, bd-wrapper, cross-repo]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
phase5_audience: agent
phase8_audited: 2026-04-17
phase8_findings: [partial-completion]
---

# gt bead

Parent command for cross-repository bead operations — move beads
between repos and view beads by ID with automatic prefix-based
routing.

**Also known as:** `gt bd` (alias, `bead.go:17`).
**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupWork` ("Work Management") (`bead.go:18`)
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/bead.go:15-214`.

The top-level `gt bead` is a pure parent command with no direct action —
its `Long` description at `bead.go:20-29` lists the three subcommands
it exposes.

### Invocation

```
gt bead <subcommand> [args]
gt bd   <subcommand> [args]    # alias
```

### Subcommands

Registered in `init()` at `bead.go:89-95`. Three total:

1. **`gt bead move <bead-id> <target-prefix>`**
   (`beadMoveCmd`, `bead.go:32-49`)

   Moves a bead from one repository to another by creating a copy in
   the target repo (with the new prefix) and closing the source bead
   with a reference to the new location. The body is in `runBeadMove`
   at `bead.go:109-214`:

   1. Normalize the target prefix to end in `-` (`bead.go:113-116`).
   2. Fetch source bead JSON via `BdCmd("show", sourceID, "--json")`
      with `resolveBeadDir(sourceID)` to route to the right rig
      database, using `StripBeadsDir()` (`bead.go:119-126`).
   3. Unmarshal the response array into `[]moveBeadInfo` and take the
      first entry (`bead.go:128-136`).
   4. Refuse to move closed beads (`bead.go:139-141`).
   5. Guard against flag-like titles propagating during move via
      `beads.IsFlagLikeTitle` — a reference to `gt-e0kx5`
      (`bead.go:148-150`).
   6. On `--dry-run`, print the intended actions and return
      (`bead.go:152-157`).
   7. Run `bd create --prefix <target> --title=... --type ... --priority
      ... --silent` (plus optional description, assignee, labels) via
      `exec.Command("bd", ...)` to create the new bead
      (`bead.go:161-189`). Output is the new bead ID.
   8. Run `bd close <sourceID> --reason "Moved to <newID>"` on the
      source. If the close fails, rolls back by closing the new bead
      with a cleanup reason (`bead.go:194-207`). If that rollback also
      fails, both beads remain open and a manual-cleanup warning is
      printed.
   9. Print `Bead moved: <sourceID> → <newID>`.

   `--dry-run` flag (`bead.go:51`): `-n, --dry-run`.

2. **`gt bead show <bead-id> [flags]`** (`beadShowCmd`, `bead.go:53-69`)

   Alias for `gt show`. Delegates directly to `runShow(cmd, args)`. Uses
   `DisableFlagParsing: true` so any `bd show` flags (e.g., `--json`)
   pass through unchanged.

3. **`gt bead read <bead-id> [flags]`** (`beadReadCmd`, `bead.go:71-87`)

   Alias for `gt bead show` — also delegates to `runShow(cmd, args)`.
   Same `DisableFlagParsing: true` setup.

### `moveBeadInfo` shape

Internal struct at `bead.go:97-107` used to deserialize `bd show
--json`: `ID`, `Title`, `IssueType` (tag `issue_type`), `Priority`,
`Description`, `Labels`, `Assignee`, `Status`.

### Related commands

- [cat](cat.md) — also wraps `bd show`, but as a top-level command with
  prefix routing.
- [close](close.md) — the top-level `gt close` command, also a bd
  wrapper. `runBeadMove` shells directly to `bd close` (not `gt close`)
  for its rollback path.
- [../binaries/gt.md](../binaries/gt.md) — root.

## Failure modes

### Partial completion (what doesn't it clean up?)

- **`bead move` source close failure leaves duplicates:** `bead.go:196-207` — after creating the new bead, if closing the source bead fails, the code attempts to close the new bead as cleanup. But if *that* cleanup also fails, both old and new beads remain open with a manual cleanup message to stderr. **Present** — cleanup attempt exists but can fail, leaving both beads open. The error message at `bead.go:202-203` explicitly warns of the double-open state.
- **`bead move` creates new bead via raw `exec.Command`:** `bead.go:183-189` — unlike the source show which uses `BdCmd` with routing, the create uses bare `exec.Command("bd", ...)` without explicit `Dir`. **Absent** — if cwd is not in the target prefix's rig, the new bead may be created in the wrong database.

## Notes / open questions

- **`runShow` lives elsewhere** — both `show` and `read` subcommands
  delegate to a `runShow` function not defined in `bead.go`. Probably
  lives in `show.go` (which is registered as a top-level `gt show`
  command per [README.md](README.md)). The wiki entry for `gt show`
  would be the place to document the routing logic shared between the
  two commands.
- **Prefix normalization quirk** — the normalization converts `""` →
  `"-"`, and then the create path skips `--prefix` when the normalized
  value equals `"-"` (`bead.go:161-164`). So an empty `targetPrefix`
  results in creating a bead with no `--prefix` flag at all, falling
  back to whatever `bd create` considers its default for the target
  working directory. Slightly subtle — flag it.
- **`StripBeadsDir`** vs `filterEnvKey(..., "BEADS_DIR")` — two different
  env-stripping patterns used in the same file family (this file uses
  `StripBeadsDir` via `BdCmd`, `cat.go` uses `filterEnvKey`). Worth
  confirming they're equivalent.
