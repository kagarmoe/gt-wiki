---
name: tracking-wiki-work
description: Use when filing, claiming, or closing a bd bead for wiki work in ~/repos/gt-wiki, when encountering a `wants-wiki-entry` label, or when Kimberly asks about the task-tracking split between beads (agents) and the Obsidian Tasks plugin (Kimberly's personal work). Covers the wiki-specific bd actor convention, label set, bead description format, Obsidian Tasks plugin conventions, and the cross-tool handoff protocol.
---

# Tracking Wiki Work

## Overview

Task tracking in the gt-wiki project is split across two tools by audience:

- **Agents (LLM sessions, Gas Town crew) use beads (`bd`).** Investigation threads, follow-ups, batch anchors, drift findings — anything that a future session should pick up — live in the wiki's embedded `.beads/` dolt database. Agents claim, update, and close beads as part of wiki work.
- **Kimberly (the human curator) uses the Obsidian Tasks plugin.** Personal tasks (curation decisions, questions to investigate, reading queue, things to revisit) live as plain GFM checkboxes anywhere in the vault, aggregated by plugin query blocks.

The two tools are NOT synced. Tasks live in exactly one system at a time; a task moves between systems via an explicit cross-tool handoff protocol.

## When to use

Triggers:
- Filing a new bead for wiki investigation or content work (`bd create`).
- Claiming an existing bead to work on it (`bd update <id> --claim`).
- Closing a bead after wiki work completes (`bd close <id>`).
- Encountering a bead labeled `wants-wiki-entry` (that's a handoff from an agent to Kimberly).
- Kimberly marking an Obsidian Tasks plugin checkbox complete with a pointer to a bead (that's a handoff from Kimberly to an agent).
- Starting a Phase 3+ audit batch that files sub-batch anchor beads.

When NOT to use:
- Filing a bead for non-wiki work (that's a different project's bd database).
- Using `bd` in the gastown repo (`.beads/` is the wiki's embedded database; gastown has its own bd infrastructure via the Gas Town dolt server on port 3307; the wiki's bd is operationally independent).
- Personal reminders that don't affect wiki content (use a personal notes file, not either tool).

## Wiki-specific bd conventions

### Actor

LLM sessions working on the wiki use `wiki-curator` as the bd actor. Set via environment variable at session start or on each command:

```bash
export BEADS_ACTOR=wiki-curator

# Or per-command:
BEADS_ACTOR=wiki-curator bd create --title="..." --description="..." --type=task
```

This distinguishes wiki-curation beads from other agent work in the audit trail. Human-originated beads (Kimberly filing directly) use the default git-user identity and should NOT use `wiki-curator`.

### Core label set

Every wiki bead carries labels from this set:

- **`wiki-investigation`** — open investigation thread (verify something, read a source, capture an outcome). Investigation beads close when the question is answered.
- **`wiki-content`** — a page needs writing or updating. Content beads close when the page lands.
- **`drift`** — drift-annotation work on an existing page (Phase 3+).
- **`phase3`** — Phase 3 work specifically. Batch anchors and sub-batch children.
- **`wiki-stale`** — wiki-stale lint finding (Phase 2 synthesis drifted against current source). Logged under `lint` verb, not `drift-found`.
- **`<topic>`** — topic scoping. Currently only `gastown`. Add new topics as they come online.
- **`wants-wiki-entry`** — handoff from agent to Kimberly (see "Cross-tool handoff" below).

Free-form tags (e.g. `docker`, `makefile`, `polecat`) are fine when they help narrow a query — add them alongside the core labels, not in place of them.

### Bead description format

Every wiki bead description includes:

1. **Wiki entity pages the bead touches** — absolute paths (e.g. `~/repos/gt-wiki/gastown/binaries/gt.md`).
2. **Source code references** — `file:line` format for anything in `~/repos/gastown/` (e.g. `internal/cmd/root.go:96-106`).
3. **Plan reference (if applicable)** — path to the active plan file (e.g. `.claude/plans/2026-04-14-phase3-drift.md`) and the specific sub-batch or task this bead corresponds to.
4. **Acceptance criteria (optional)** — what needs to be true for the bead to close. Especially useful for investigation beads where the "done" state isn't obvious.

Example:

```
bd create \
  --title="Verify BuiltProperly ldflag bypass in Docker build" \
  --description="Investigation thread: Phase 2 Batch 1 surfaced the BuiltProperly self-kill gate at internal/cmd/root.go:96-106 and the Makefile LDFLAGS recipe at Makefile:18-22. Need to verify that Docker-build path in Dockerfile actually sets BuiltProperly=1 on the produced binary. Touches ~/repos/gt-wiki/gastown/files/dockerfile.md and ~/repos/gt-wiki/gastown/files/makefile.md. Plan: .claude/plans/2026-04-11-gastown-map.md Batch 1 Task 5. Acceptance: docker run gastown:latest gt --version exits 0 without the self-kill error." \
  --type=task \
  --priority=2
```

### Priority

Wiki beads default to **priority 2** (medium). Exceptions:
- **P0 (critical)** — Phase-blocking discoveries, audit-trail corruption, schema inconsistencies that could propagate.
- **P1 (high)** — Investigation threads that other work depends on.
- **P3 (low)** — Nice-to-have coverage extensions, stylistic notes.
- **P4 (backlog)** — Future-phase work that's captured but not yet active.

Priority is a number 0-4 (or `P0`-`P4`), NOT `"high"`/`"medium"`/`"low"` (the bd CLI rejects those words).

## Kimberly's Obsidian Tasks plugin conventions

Kimberly's personal tasks (curation decisions, questions, reading queue, things she wants to revisit) live as GFM checkboxes anywhere in the vault. The Obsidian Tasks plugin scans the whole vault and aggregates them via query blocks.

### Task format

Plain GFM checkboxes are sufficient:

```markdown
- [ ] Review the writing-entity-pages skill for accuracy before Phase 3 Batch 1
- [ ] Decide whether to add Bitbucket as a topic or defer to a release-sync bead
- [x] Confirm Option B version numbering (1.1 → 1.2)
```

Tasks plugin emoji metadata is optional but supported:
- `📅 2026-04-20` — due date
- `⏫` high priority (or `🔼` medium, `🔽` low)
- `🔁 every week` — recurrence

### Location

**No fixed file location.** File tasks wherever they fit:
- Inline on an entity page where the task relates to the page's content
- In a personal notes file Kimberly maintains (e.g. `personal-notes.md` in the wiki root — currently not a schema requirement, but possible)
- In an existing wiki page's `## Notes / open questions` section if the task is a wiki-curation concern

The Tasks plugin scans the whole vault, so location doesn't affect discoverability.

### Query blocks

Aggregate on demand via plugin query blocks:

    ```tasks
    not done
    sort by priority
    ```

    ```tasks
    not done
    path includes gastown/
    sort by due
    ```

Kimberly uses these in her personal dashboard pages. Agents generally don't need to write query blocks — the plugin's UI handles it.

## Cross-tool handoff protocol

The two trackers occasionally hand work off. Use conventions, not a sync bridge.

### Agent → Kimberly (via `wants-wiki-entry` label)

When an agent discovers something that needs Kimberly's judgment (a gap finding, a scope question, a decision that's out of scope for the current batch), it files a bead with the `wants-wiki-entry` label:

```bash
BEADS_ACTOR=wiki-curator bd create \
  --title="Gap finding: internal/bitbucket/ has no Phase 2 wiki page" \
  --description="During Phase 3 Batch 0 Task 1 Part A scoping, PR-delta against v1.0.0 revealed a new package internal/bitbucket/ (5 files, PR #3600, Bitbucket Cloud merge queue integration). This package was introduced post-v1.0.0 and is not in Phase 2's go-packages inventory. Decision needed: expand Phase 3 Batch 2 to include bitbucket/ as a new sub-batch, OR defer to a release-sync follow-up. Plan: .claude/plans/2026-04-14-phase3-drift.md Batch 0 Task 1." \
  --type=task \
  --priority=2
bd update <new-id> --labels=wants-wiki-entry,wiki-investigation,phase3
```

Kimberly surfaces these periodically with:

```bash
bd list -l wants-wiki-entry
```

She either resolves the bead herself (investigates, makes a decision, closes) OR promotes it into her Obsidian Tasks plugin tracker (filing a `- [ ]` checkbox with a pointer) and closes the bead with a reason citing where she filed it.

### Kimberly → agent (via `→ handed to bd-<id>` notation)

When Kimberly wants to convert one of her Obsidian Tasks checkboxes into a tracked bead (because it's become agent work she wants a session to pick up), she files the bead herself using her default git-user identity (NOT `wiki-curator`, because the bead is human-originated):

```bash
bd create --title="..." --description="..." --type=task --priority=2
```

Then she closes her checkbox with a trailing pointer:

```markdown
- [x] Verify BuiltProperly ldflag bypass in Docker build → handed to bd-wiki-ztf
```

The closed checkbox stays in its original file for audit purposes; the bead takes over the live tracking.

### Handoff rules

- **Do not duplicate tasks across trackers.** A task lives in exactly one system at a time. Handoff moves it; it doesn't copy it.
- **The live tracker's update is authoritative.** If a bead is open, the bead's state is truth. If a checkbox is open (and no `→ handed to` marker), the checkbox's state is truth. If both are open for the same work, one of them is a stale duplicate and should be closed with a pointer to the live one.
- **Handoff direction doesn't change actor conventions.** Agent → Kimberly handoffs still use `wiki-curator` when filing the bead (the agent files it). Kimberly → agent handoffs use her default git-user identity (she files it).

## Common bd commands for wiki work

```bash
# Find available wiki work
bd ready

# List all open wiki beads
bd list --status=open -l wiki-content,wiki-investigation,drift,phase3

# List beads awaiting Kimberly's attention
bd list -l wants-wiki-entry

# Show a specific bead's details + dependencies
bd show wiki-xxx

# Claim a bead before starting work
BEADS_ACTOR=wiki-curator bd update wiki-xxx --claim

# Update a bead's description in place (use instead of bd edit, which opens $EDITOR)
BEADS_ACTOR=wiki-curator bd update wiki-xxx --description="updated text"

# Close a bead after work completes
bd close wiki-xxx

# Close with a reason
bd close wiki-xxx --reason="Resolved by commit <sha>; see gastown/commands/done.md"

# File a new bead with full context
BEADS_ACTOR=wiki-curator bd create \
  --title="..." \
  --description="..." \
  --type=task \
  --priority=2

# Add a dependency (bead A depends on bead B)
bd dep add wiki-aaa wiki-bbb
```

**WARNING: do NOT use `bd edit`** — it opens `$EDITOR` (vim/nano) which blocks agent sessions. Use `bd update --description=` / `--notes=` / `--design=` instead.

## Common mistakes

| Mistake | Fix |
|---|---|
| Using default git-user actor for an LLM-session bead | Set `BEADS_ACTOR=wiki-curator` before `bd create`. |
| Duplicating an Obsidian Tasks checkbox as a bead without closing the checkbox | Always close one side with a pointer when you file the other. |
| Using `bd edit` in an agent session | Opens `$EDITOR` and blocks. Use `bd update --description=` instead. |
| Filing a bead without labels | Every wiki bead gets at least topic (`gastown`) + work type (`wiki-content`/`wiki-investigation`/`drift`/`phase3`). |
| Using `"high"` / `"medium"` / `"low"` for priority | Use `0`-`4` or `P0`-`P4`. |
| Closing a bead without a reason when the reason matters | Use `bd close <id> --reason="..."` to preserve the audit trail. |
| Filing a drift finding as a bead instead of annotating the entity page | Drift findings live inline on the entity page's `## Drift` section. The bead (if any) tracks the audit work that surfaces the drift, not the drift itself. |

## Reference

- **Wiki schema and coordination:** [CLAUDE.md](../../../CLAUDE.md)
- **How to write entity pages (including drift section format):** `.claude/skills/writing-entity-pages/SKILL.md`
- **Schema evolution and lint workflow:** `.claude/skills/maintaining-wiki-schema/SKILL.md`
- **Active phase plan:** `.claude/plans/` (gitignored)
- **bd CLI reference:** run `bd prime` for full command documentation; the wiki's `.beads/` is an embedded dolt database operationally independent from Gas Town's port-3307 server.
