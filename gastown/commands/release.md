---
title: gt release
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/cmd/release.go
  - /home/kimberly/repos/gastown/internal/beads/
tags: [command, work, beads, recovery, stuck-work, idempotence]
---

# gt release

Release stuck `in_progress` issues back to `open`/`pending` so
another worker can claim them. Used to recover stuck steps after a
worker dies mid-task. Despite the name, this has **nothing to do
with software releases** — see [changelog](changelog.md) for
release notes, and `../files/goreleaser-yml.md` for binary release
automation.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupWork` ("Work Management") (`release.go:15`)
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/release.go` (full
file, 78 lines).

### Invocation

```
gt release <issue-id>...
gt release gt-abc -r "worker died"
```

`Args: cobra.MinimumNArgs(1)` (`release.go:31`).

### Behavior (`runRelease`, `release.go:40-78`)

1. **Find cwd.** `os.Getwd()` — the beads database is looked up from
   the current working directory via `beads.New(cwd)` (`:41-47`).
2. **Release each issue** (`:49-66`): iterate over args, call either
   `bd.ReleaseWithReason(id, releaseReason)` if `--reason` was
   provided, or `bd.Release(id)`. Track `released` / `failed`
   counts and print per-issue success (`✓ Released <id> → open`)
   or failure (`✗ Failed to release <id>: <err>`).
3. **Summary** (`:69-71`): if more than one argument, print
   `Released: N, Failed: M`.
4. **Return** — if any release failed, return an error stating how
   many failed; otherwise `nil` (`:73-77`).

The implementation of `beads.Release` and `beads.ReleaseWithReason`
lives in [`internal/beads/`](../packages/beads.md) (not read here). Per the Long help
(`:21-23`), a release moves the issue to `open` status and clears
the assignee so another worker can claim it.

### Flags

| flag | default | notes |
|---|---|---|
| `-r`, `--reason <text>` | `""` | Reason recorded as a note on the bead. Optional. |

### Subcommands

None.

### Nondeterministic idempotence

The Long text (`:29-30`) frames this as: "work can be safely retried
by releasing and reclaiming stuck steps." This is the recovery lever
for the Gas Town workflow — if a polecat dies mid-task, the work
bead stays in `in_progress` until someone releases it. The lever is
meant to be pulled by Deacon patrol or a human operator, not by the
stuck worker itself.

## Notes / open questions

- **Name collision with software-release semantics.** `gt release`
  does NOT publish binaries, cut tags, or manage changelogs. A
  newcomer typing `gt release` expecting those actions will release
  every issue they list. The risk is mitigated by the required
  issue-ID argument (`MinimumNArgs(1)`), but still worth flagging.
- **No `--dry-run`.** Unlike most state-mutating commands in this
  group, `release` has no preview mode. The reported success or
  failure is post-facto.
- **Single-DB scope.** `beads.New(cwd)` means this only operates
  on whichever beads database corresponds to cwd — if a polecat
  dies with an hq-* bead in progress (stored in the town db), the
  user must run `gt release` from the town root.
- **Related commands.**
  - [done](done.md) — the happy path that releases a polecat's
    hook via completion rather than recovery.
  - [unsling](unsling.md) — clears a specific hook slot; partial
    overlap with `release` when the hook is the only thing stuck.
  - [orphans](orphans.md) — finds stranded work that may need a
    `release` call.
  - [resume](resume.md) — inbox-side counterpart for resuming
    after interruption (checks handoff mail, doesn't touch bead
    status).
