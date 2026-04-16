---
title: gt commit
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/cmd/commit.go
tags: [command, work, git, identity, agent]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
---

# gt commit

Wrapper around `git commit` that injects agent git author identity
derived from `GT_ROLE` / `detectSender()`.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupWork` ("Work Management") — set on `commit.go:44`
**Hidden:** no
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no

> **Batch-plan correction:** `gt commit` is assigned to `GroupWork`
> (`commit.go:44`), **not** ungrouped as the batch briefing suggested.
> This page documents the actual cobra definition. Same applies to
> [forget](forget.md), [memories](memories.md), [remember](remember.md),
> and [show](show.md) — all five are in `GroupWork` rather than
> ungrouped.

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/commit.go:16-123`.

### Invocation

```
gt commit [flags] [-- git-commit-args...]
```

`Use: "commit [flags] [-- git-commit-args...]"` (`commit.go:17`).
`DisableFlagParsing: true` (`commit.go:40`) — all arguments are passed
through untouched so cobra does not consume git's own flags. A custom
`checkHelpFlag(cmd, args)` path on `commit.go:50-52` handles `--help`
itself.

### Behavior

`runCommit` on `commit.go:48-80`:

1. **Handle `--help`** first (`commit.go:50-52`) — otherwise
   `DisableFlagParsing` would pass `--help` on to git.
2. **Detect identity** via `detectSender()` (`commit.go:55`). This is the
   same identity resolver used by [mail](mail.md) and peers — reads
   `GT_ROLE`, `GT_RIG`, etc. and produces a dotted path like
   `gastown/crew/jack` or `mayor/`.
3. **Pass through for the overseer** (`commit.go:58-60`): if `identity ==
   "overseer"`, call `runGitCommit(args, "", "")` with no identity
   overrides. Humans get the default git config.
4. **Load email domain from town settings** (`commit.go:63-70`):
   - Calls `workspace.FindFromCwd()`.
   - If a town root is found, loads
     `config.LoadOrCreateTownSettings(config.TownSettingsPath(townRoot))`.
   - If `settings.AgentEmailDomain` is non-empty, uses it; otherwise
     defaults to `DefaultAgentEmailDomain = "gastown.local"` defined on
     `commit.go:14`.
5. **Convert identity to email** via `identityToEmail(identity, domain)`
   (`commit.go:74`, body on `commit.go:85-93`):
   - Strip trailing `/`.
   - Replace `/` with `.` to form the local part.
   - Append `@<domain>`.
   - Example: `gastown/crew/jack` → `gastown.crew.jack@gastown.local`.
   - Example: `mayor/` → `mayor@gastown.local`.
6. **Name = identity** (unmodified, human-readable, e.g. `gastown/crew/jack`).
7. **Invoke git** via `runGitCommit(args, name, email)`
   (`commit.go:79-80`, body on `commit.go:98-123`):
   - Prepends `-c user.name=<name> -c user.email=<email>` to the argv if
     both are non-empty.
   - Executes `git commit <args>` with stdin/stdout/stderr wired to the
     terminal.
   - On failure, unwraps `*exec.ExitError` and returns
     `NewSilentExit(exitErr.ExitCode())` — so `gt commit`'s exit code
     mirrors git's exit code without adding cobra's noise on top.

### Flags

None defined directly — `DisableFlagParsing: true` sends every flag
through to git. Everything after `gt commit` is forwarded verbatim.

### Subcommands

None.

### Example identity mappings

From the `Long` help on `commit.go:34-36`:

```
Agent: gastown/crew/jack  →  Name: gastown/crew/jack
                              Email: gastown.crew.jack@gastown.local
```

## Related commands

- [mail](mail.md) — shares `detectSender()` for identity resolution.
- [sling](sling.md), [done](done.md) — high-level work-flow verbs that
  typically commit via this wrapper under the hood (to be verified by
  reading those files).
- [bead](bead.md) — issue-tracker commands that generate commit messages.
- [assign](assign.md) — cross-agent work delegation; worth pairing with
  commit identity.
- [handoff](handoff.md) — session handoff can touch the working tree.
- [../binaries/gt.md](../binaries/gt.md) — root.

## Notes / open questions

- **Group mismatch with `ungrouped` batch inventory.** The plan had
  `commit` in the "ungrouped" bucket; the cobra source disagrees. Finalize
  should file this as a small correction to the inventory rather than
  treating it as drift on this page.
- **`AgentEmailDomain` plumbing.** The field is loaded from town settings
  via `LoadOrCreateTownSettings` — which means the first `gt commit` in a
  fresh workspace will create the settings file as a side effect. That's
  an unusual place for file creation; worth flagging.
- **`NewSilentExit`.** The wrapper preserves git's exit code via a
  typed error — this is the pattern other pass-through commands should
  follow when wrapping external tools.
- **Co-authored-by handling.** This command wraps only `git commit`'s
  `-c user.name`/`-c user.email`; it does not inject `Co-Authored-By`
  trailers. Any co-author trailers must come from the caller's `-m`
  argument.
