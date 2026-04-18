---
title: gt wl
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/cmd/wl.go
  - /home/kimberly/repos/gastown/internal/cmd/wl_browse.go
  - /home/kimberly/repos/gastown/internal/cmd/wl_charsheet.go
  - /home/kimberly/repos/gastown/internal/cmd/wl_claim.go
  - /home/kimberly/repos/gastown/internal/cmd/wl_done.go
  - /home/kimberly/repos/gastown/internal/cmd/wl_post.go
  - /home/kimberly/repos/gastown/internal/cmd/wl_scorekeeper.go
  - /home/kimberly/repos/gastown/internal/cmd/wl_show.go
  - /home/kimberly/repos/gastown/internal/cmd/wl_stamp.go
  - /home/kimberly/repos/gastown/internal/cmd/wl_stamps.go
  - /home/kimberly/repos/gastown/internal/cmd/wl_sync.go
  - /home/kimberly/repos/gastown/internal/wasteland/
  - /home/kimberly/repos/gastown/internal/doltserver/
tags: [command, work, wasteland, federation, dolthub, rig-registry]
phase3_audited: 2026-04-15
phase3_findings: [wiki-stale]
phase3_severities: [wrong]
phase3_findings_post_release: false
phase5_audience: agent
phase8_audited: 2026-04-17
phase8_findings: [none]
---

# gt wl

Wasteland federation commands. **Despite `wl.go` being small, `wl`
does not stand for "worklist" — it stands for "wasteland"** — a
federation of Gas Towns via DoltHub, where each rig holds a
sovereign fork of a shared commons database containing the wanted
board, rig registry, and validated completions.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupWork` ("Work Management") (`wl.go:24`)
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/wl.go` (154
lines, parent declaration + `join` subcommand) plus ten sibling
files `wl_browse.go`, `wl_charsheet.go`, `wl_claim.go`, `wl_done.go`,
`wl_post.go`, `wl_scorekeeper.go`, `wl_show.go`, `wl_stamp.go`,
`wl_stamps.go`, `wl_sync.go`. Delegates to `internal/wasteland` for
the service layer and `internal/doltserver` for DoltHub credentials.

### Invocation

```
gt wl <subcommand> [args]
```

`wlCmd.RunE = requireSubcommand` (`:26`) — bare `gt wl` errors with
help text.

### Subcommands

Eleven subcommands wired on `wlCmd`. `wl.go`'s own `init()` at
`wl.go:65-71` only registers `join`; the other ten are registered
by sibling files, each in its own `init()`. (Phase 3 Batch 1c
wiki-stale correction: the Phase 2 page body said "Only one is
wired in this file" and speculated "Other wasteland subcommands …
must be wired elsewhere, if at all." Re-read at gastown HEAD
`9f962c4a` and at `v1.0.0` confirmed all ten sibling files exist
and wire their subcommands — Phase 2 had the sources listed in
frontmatter as `wl.go` only and missed the per-file registrations.)

| subcommand | source | one-liner |
|---|---|---|
| `join <upstream>` | `wl.go:39-63`, `runWlJoin :73-154` | Fork a wasteland's commons database, clone locally, register the rig, push registration, save wasteland config. |
| `browse` | `wl_browse.go:<init>` | Browse wanted items on the commons board. |
| `charsheet [handle]` | `wl_charsheet.go:<init>` | Display a rig's character sheet (reputation and activity summary). |
| `claim <wanted-id>` | `wl_claim.go:<init>` | Claim a wanted item on the shared wanted board. |
| `done <wanted-id>` | `wl_done.go:<init>` | Submit completion evidence for a claimed wanted item. |
| `post` | `wl_post.go:<init>` | Post a new wanted item to the commons. |
| `scorekeeper` | `wl_scorekeeper.go:<init>` | Compute tier standings and update the leaderboard. |
| `show <work-id>` | `wl_show.go:<init>` | Show full details of a wanted item from the wl-commons database. |
| `stamp` | `wl_stamp.go:<init>` | Create a reputation stamp for a rig. |
| `stamps <rig-handle>` | `wl_stamps.go:<init>` | Query stamps for a rig. |
| `sync` | `wl_sync.go:<init>` | Pull upstream changes into the local wl-commons fork. |

The parent `Long` text (`wl.go:27-36`) advertises "Getting started:
`gt wl join steveyegge/wl-commons`" but does not enumerate the
other ten subcommands. Cobra's auto-`Available Commands:` block
surfaces them. The individual subcommand pages (gap — not yet
created; see Notes below) are the place to document each one in
depth.

### `join` pipeline (`runWlJoin`, `wl.go:73-154`)

1. **Parse upstream** (`:77-80`) — `wasteland.ParseUpstream(upstream)`
   validates the `<org>/<db>` shape early before touching Dolt.
2. **Require credentials** (`:82-91`):
   - `DOLTHUB_TOKEN` via `doltserver.DoltHubToken()`. Error message
     points to `https://www.dolthub.com/settings/tokens`.
   - `DOLTHUB_ORG` via `doltserver.DoltHubOrg()`. The user's own
     DoltHub org for the fork.
3. **Find town root** via `workspace.FindFromCwdOrError` (`:94-97`).
4. **Fast path: already joined?** (`:101-110`) —
   `wasteland.LoadConfig(townRoot)`:
   - If the existing config's `Upstream` matches the passed
     upstream, print "Already joined" + handle/fork/local and
     return success.
   - If it's joined to a *different* upstream, error with
     "already joined to X; run `gt wl leave` first". Note: `gt wl
     leave` is referenced but not implemented in this file.
5. **Load town config** (`:113-117`) — `config.LoadTownConfig` of
   the primary marker file. Only needed for a fresh join, so the
   fast path skips it to avoid failing on unrelated town-config
   errors for the no-op case.
6. **Handle + display name** (`:120-132`):
   - `handle` defaults to the DoltHub org.
   - `displayName` defaults to `townCfg.PublicName` or
     `townCfg.Name`.
7. **Call service** (`:137-146`):
   ```go
   svc := wasteland.NewService()
   svc.OnProgress = func(step string) { print step }
   cfg, err := svc.Join(upstream, forkOrg, token, handle,
       displayName, ownerEmail, gtVersion, townRoot)
   ```
   According to the Long text (`:43-49`), the service layer:
   1. Forks the upstream commons to your DoltHub org.
   2. Clones the fork locally.
   3. Registers your rig in the `rigs` table.
   4. Pushes the registration to your fork.
   5. Saves wasteland configuration locally.
8. **Report success** (`:148-152`) — handle, fork path, local
   path, and a hint: `Next: gt wl browse — browse the wanted
   board`.

### `gtVersion` is hardcoded

`wl.go:135` — `gtVersion := "dev"`. The registration in the rigs
table will always say `"dev"` regardless of the actual built
version. Likely a TODO.

### Flags (on `wlJoinCmd`, `:66-67`)

| flag | default | notes |
|---|---|---|
| `--handle <name>` | `""` | Rig handle for registration. Defaults to DoltHub org. |
| `--display-name <text>` | `""` | Display name for the rig registry. Defaults to town's `PublicName`/`Name`. |

## Drift

See forward-link: [../drift/README.md](../drift/README.md).

### Phase 2 wiki body claimed only `join` was wired; ten sibling files actually wire ten more subcommands (wiki-stale)

- **Category:** `wiki-stale`
- **Severity:** `wrong`
- **Phase 2 root cause:** `phase-2-incomplete` (heuristic
  determination per Sweep 1 convention — Phase 2 read `wl.go` in
  isolation, saw only `wlCmd.AddCommand(wlJoinCmd)` at `wl.go:69`,
  and concluded "Only one is wired in this file." The sibling files
  `wl_browse.go` / `wl_charsheet.go` / `wl_claim.go` / `wl_done.go`
  / `wl_post.go` / `wl_scorekeeper.go` / `wl_show.go` / `wl_stamp.go`
  / `wl_stamps.go` / `wl_sync.go` each contain their own `init()`
  calling `wlCmd.AddCommand(<subCmd>)`. Both Phase 2's HEAD and the
  current HEAD disagree with the wiki body the same way, and every
  sibling file exists at `v1.0.0` with its `AddCommand` registration
  intact — verified via
  `git -C /home/kimberly/repos/gastown ls-tree -r --name-only v1.0.0`
  and per-file
  `git -C /home/kimberly/repos/gastown show v1.0.0:internal/cmd/wl_<name>.go | grep AddCommand`.
  So this is "wrong at Phase 2 time," not churn-induced drift.)
- **Phase 2 claim (removed):** the Phase 2 page body said "Only
  one is wired in this file (`wl.go:69-70`)" and "Other wasteland
  subcommands (like the `gt wl browse` hinted at `:152`) must be
  wired elsewhere, if at all." The Notes section further said
  "`gt wl browse` and `gt wl leave` are referenced but not wired
  here. Expect other wl subcommands in sibling files or planned but
  unimplemented."
- **Current reality (2026-04-15, gastown HEAD `9f962c4a`):** eleven
  subcommands are wired — `join` (in `wl.go`) plus `browse`,
  `charsheet`, `claim`, `done`, `post`, `scorekeeper`, `show`,
  `stamp`, `stamps`, `sync` (each in its own sibling file). All ten
  sibling files are present at `v1.0.0`. `wl leave` is still NOT
  wired — the Phase 2 page correctly noted that `runWlJoin` at
  `wl.go:109` references `gt wl leave first` in an error message
  but the subcommand itself does not exist; this half of the
  original Phase 2 observation remains accurate and is preserved in
  the Notes section below.
- **Fix tier:** `wiki` — already fixed inline in the
  `### Subcommands` section of `## What it actually does` above
  (the Phase 2 "only one wired" claim is replaced with the full
  11-row subcommand table; the frontmatter `sources:` list now
  enumerates all ten sibling files).

## Notes / open questions

- **`wl` ≠ worklist.** This is important for wiki readers: the
  name is short for **wasteland** (as in Gas Town's post-apoc
  aesthetic, see [../binaries/gt.md](../binaries/gt.md)). There
  is no "worklist" command — the equivalent surface is
  [ready](ready.md) for town-wide ready work or
  [scheduler](scheduler.md) for deferred dispatch.
- **Thin parent, thick service layer.** Almost all `join` logic
  lives in `internal/wasteland/Service` — see
  [../packages/wasteland.md](../packages/wasteland.md) for the
  federation model, Spider Protocol fraud detection, trust tiers,
  the `rigs` table schema, and the fork-and-push flow. Each of the
  ten sibling-file subcommands has its own helper surface under
  `internal/wasteland` which the Phase 2 mapping did not trace; a
  proper map of that layer is a follow-up beyond this Phase 3 pass.
- **Wiki sub-pages for each `wl` subcommand are a gap.** Each of
  the ten sibling-file subcommands (`browse`, `charsheet`, `claim`,
  `done`, `post`, `scorekeeper`, `show`, `stamp`, `stamps`, `sync`)
  deserves its own entity page under `gastown/commands/` —
  following the convention used for e.g. `gt sling respawn-reset`
  living inline on the parent page only because it's a single
  subcommand. The Wasteland family is large enough that per-file
  pages would be more scannable. Filed as a Phase 4 (Coverage)
  follow-up rather than a Batch 1c action item.
- **`gt wl leave` is referenced but not wired.** `runWlJoin` at
  `wl.go:109` returns `"already joined to %s; run gt wl leave
  first"` as an error message, but no `wlLeaveCmd` is declared in
  any `wl*.go` file at HEAD `9f962c4a` or at `v1.0.0`. This is a
  latent gap (the error message directs users to a command that
  doesn't exist), classified here as neutral observation — it would
  be promoted to an `implementation-status: unbuilt` finding in a
  Sweep 2 pass if upstream docs describe `wl leave` behavior.
- **Required env vars** (`DOLTHUB_TOKEN`, `DOLTHUB_ORG`) turn
  this into an externally-dependent command. Offline use is
  impossible.
- **`--display-name` / `--handle` flags** take precedence over
  the town config defaults, so this is how a rig advertises a
  "public persona" distinct from its internal identity.
- **Related.** [config](config.md) (reads town config + writes
  wasteland config), [doctor](doctor.md) (may check wasteland
  health), [convoy](convoy.md) (the in-town tracking unit;
  wasteland is cross-town).
