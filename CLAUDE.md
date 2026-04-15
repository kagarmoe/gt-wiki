# CLAUDE.md — Wiki Schema & Coordination

**Schema version:** 1.2 (2026-04-14)

## What this is

This is an LLM-maintained knowledge wiki following Karpathy's "LLM Wiki"
pattern. It's a compounding artifact: findings, cross-references, and
syntheses accumulate here rather than being re-derived every session.

**Scope (current):** the `gastown` project — its source code, documentation,
build system, runtime behavior, and the drift between docs and code.

**Scope (planned):** may grow to cover other repos in `~/repos/`.

**Ownership:** the LLM writes and maintains this wiki. Kimberly curates
sources, directs investigation, asks questions, and reviews changes. Raw
sources are never modified — only read.

## Layers

1. **Raw sources (immutable):** source repos under `~/repos/<name>/`,
   documents Kimberly drops in, command outputs captured during
   investigations. Read-only. Source of truth.
2. **Wiki (this directory):** LLM-owned markdown. Pages, index, log. Fully
   rewriteable.
3. **Schema (this file + project-local skills):** conventions and
   workflows. CLAUDE.md is the slim coordinator; detailed schema reference
   lives in `.claude/skills/`.

## Directory layout

```
~/repos/gt-wiki/
  CLAUDE.md          # this file — slim coordinator
  AGENTS.md          # agent-facing task-tracking entrypoint (bd integration)
  CONTRIBUTING.md    # community contributor guidelines
  README.md          # top-level project readme
  LICENSE            # CC BY 4.0
  index.md           # global catalog of all pages
  log.md             # chronological event log (append-only)
  .claude/
    settings.json         # Claude Code settings (committed)
    settings.local.json   # permission scoping; gitignored
    plans/                # LLM workflow plans; gitignored, per-phase
    skills/               # project-local skills (triggered runbooks + reference)
      writing-entity-pages/    # frontmatter, page types, scaffold, drift taxonomy
      tracking-wiki-work/      # bd conventions + Obsidian Tasks + handoff protocol
      maintaining-wiki-schema/ # schema evolution, lint workflow, maintenance principles
      syncing-gastown-updates/ # release-sync runbook (triggered by new gastown tag)
    llm-wiki.md           # Karpathy LLM-wiki pattern reference; gitignored
  .obsidian/         # Obsidian vault config (app.json committed; workspace state gitignored)
  .beads/            # beads issue tracker (config + hooks committed; dolt db + runtime gitignored)
  <topic>/           # one folder per topic namespace (gastown, ...)
    README.md        # topic overview + sub-index (project purpose lives here, per-topic)
    binaries/        # entity pages: one per binary
    commands/        # entity pages: one per CLI subcommand
    packages/        # entity pages: Go packages / source modules
    files/           # entity pages: notable single files
    config/          # entity pages: config file shapes, env vars
    concepts/        # abstract ideas
    roles/           # personified agents / positions
    services/        # long-running processes
    workflows/       # multi-step flows
    plugins/         # plugin-directory pages
    drift/           # cross-entity drift index + Phase 3 findings aggregation (Phase 3+)
    inventory/       # A-level enumeration index pages (not entity pages)
    ...              # create additional categories as content arrives
```

Sub-folders are created lazily. Don't scaffold empty folders.

## Hard rules (every session)

Seven non-negotiable constraints that apply to every wiki work session:

1. **Code is the only source of truth.** The code at `~/repos/gastown/` is authoritative. Wiki pages, Cobra `Long` help text, `docs/*.md`, and READMEs are all synthesis/derivation and must be verified against code.
2. **Nesting capped at 3 levels:** `<topic>/<category>/<entity>.md`. If a page feels like it needs deeper nesting, split the category into sibling categories instead.
3. **Relative markdown links only.** Use `[text](../category/page.md)`, never Obsidian wikilinks (`[[...]]`). External source refs use absolute paths in backticks: `` `/home/kimberly/repos/gastown/internal/cmd/root.go:96-106` ``.
4. **YAML frontmatter required** on every entity page. See `.claude/skills/writing-entity-pages/` for the full template.
5. **`log.md` is append-only.** Historical entries are preserved verbatim, even when superseded. Corrections land as new entries, not in-place edits. Grep the timeline with `grep "^## \[" log.md`.
6. **Kimberly curates; the LLM writes.** Raw sources in `~/repos/gastown/` are read-only. The LLM never modifies source repos.
7. **Write access is scoped to `~/repos/gt-wiki/`.** Everything outside the wiki is read-only reference. See "Permission scoping" below.

## Session startup (required reading)

New session? Read these in order:

1. **`log.md` timeline** — `grep "^## \[" log.md | tail -20` to see the last 20 events. The most recent entries tell you what state the investigation is in, what's in progress, what decisions have landed, and what's blocked.
2. **Active plan** — `ls -t .claude/plans/*.md | head -1` to find the most recent plan file; read it in full. The plan is the authoritative "what are we doing right now" document for whichever phase is active.
3. **Topic README** under `<topic>/` (currently `gastown/README.md`) — the topic's own scope statement, sub-index, and working-model summary.
4. **This file (already loaded)** — coordinates the rest. Use the Pointers table below to find operational detail.

## Pointers to operational detail

The coordination surface. Schema reference, workflows, and decision history live in dedicated artifacts:

| Content | Where it lives |
|---|---|
| **Writing / editing entity pages** — frontmatter template, 17 page types, naming + link conventions, 5-section scaffold, drift taxonomy with fix-tier classification, Code → Cobra → Docs authority hierarchy, cross-link discipline, non-retroactive Phase 3 section rule | `.claude/skills/writing-entity-pages/SKILL.md` |
| **Tracking wiki work** — bd actor (`BEADS_ACTOR=wiki-curator`), label set, bead description format, Obsidian Tasks plugin conventions, cross-tool handoff protocol (`wants-wiki-entry` label, `→ handed to bd-<id>` notation) | `.claude/skills/tracking-wiki-work/SKILL.md` |
| **Schema evolution + lint + maintenance** — schema version bump policy, lint workflow (orphans, broken links, wiki-stale, stale dates, mention gaps), maintenance principles, common patterns for adding page types/fields/folders | `.claude/skills/maintaining-wiki-schema/SKILL.md` |
| **Release sync after a new stable gastown release tag** — tag-to-tag delta, scope proposal, per-file audit, commit cadence, batch entry format | `.claude/skills/syncing-gastown-updates/SKILL.md` |
| **Active phase plan** — current phase's batch decomposition, task-level steps, drift taxonomy (if Phase 3+), PR-delta scoping, review gates | `.claude/plans/` (gitignored; grep for the most recent file) |
| **Schema decision history** — every `decision` entry in `log.md` is a schema-evolution event. Most recent: `[2026-04-14] decision | Phase 3 schema change: re-enable Docs claim / Drift / Implementation status sections (schema v1.2)` | `log.md` — `grep "decision \|" log.md` |
| **Event timeline** — `ingest`, `query`, `experiment`, `lint`, `drift-found` entries | `log.md` — `grep "^## \[" log.md` |
| **Page catalog** | `index.md` |

## Ingest / Query / Lint workflows

These three workflows remain inline as terse paragraphs. Each will be promoted to its own `.claude/skills/` skill when the workflow grows beyond a single paragraph of guidance.

### Ingest workflow

When a new source is read (a file, a command output, a doc page): note the source path → read it in full → discuss key takeaways with Kimberly (unless she has delegated) → update or create the entity page for each entity the source touches → update `index.md` if new pages were created → append a `log.md` entry with the `ingest` verb citing sources read and pages touched. Plan: promote to `.claude/skills/ingesting-sources/` when the workflow exceeds one paragraph.

### Query workflow

When Kimberly asks a question: read `index.md` to find candidate pages → read candidate pages in full (not chunks) → synthesize a code-grounded answer with links to the pages used → if the synthesis is valuable, file it back as a new page under the appropriate category, update `index.md`, and log with the `query` verb. Plan: promote to `.claude/skills/answering-queries/` when the workflow exceeds one paragraph.

### Lint workflow (summary)

Periodically or on request, scan for orphans, broken cross-links, claims contradicted by newer sources, stale `updated:` dates, entity mentions that should have their own page, and gaps that could be filled by reading a specific file. **Produces findings, not fixes.** Discuss with Kimberly before acting. Log with the `lint` verb. Full runbook: `.claude/skills/maintaining-wiki-schema/SKILL.md`.

## Permission scoping

`.claude/settings.local.json` (gitignored) grants this wiki full
Read/Write/Edit/Bash permissions within `~/repos/gt-wiki/` and read-only
access everywhere else. The LLM owns the wiki and can mutate it freely, but
treats everything outside `~/repos/gt-wiki/` as reference only.

## Wiki-specific overrides to the Beads Integration block

The auto-managed `<!-- BEGIN BEADS INTEGRATION -->` block below was
dropped in by `bd init` and is written for generic Gas Town agent
work. Its rules do not all apply here — the following override takes
precedence over the boilerplate:

**Markdown TODO lists are PERMITTED** for Kimberly's personal task
tracking via the Obsidian Tasks plugin. The boilerplate rule "do NOT
use … markdown TODO lists" is scoped to *agent* work (which lives in
beads), not to human curation work in the vault. See
`.claude/skills/tracking-wiki-work/SKILL.md` for the full policy: bd
for agents, Tasks plugin for Kimberly, with documented cross-tool
handoff via `wants-wiki-entry` labels.

## Beads Issue Tracker

This project uses **bd (beads)** for issue tracking. Run `bd prime` to see full workflow context and commands.

### Quick Reference

```bash
bd ready              # Find available work
bd show <id>          # View issue details
bd update <id> --claim  # Claim work
bd close <id>         # Complete work
```

### Rules

- Use `bd` for ALL task tracking — do NOT use TodoWrite, TaskCreate, or markdown TODO lists
- Run `bd prime` for detailed command reference and session close protocol
- Use `bd remember` for persistent knowledge — do NOT use MEMORY.md files

## Session Completion

**When ending a work session**, you MUST complete ALL steps below. Work is NOT complete until `git push` succeeds.

**MANDATORY WORKFLOW:**

1. **File issues for remaining work** - Create issues for anything that needs follow-up
2. **Run quality gates** (if code changed) - Tests, linters, builds
3. **Update issue status** - Close finished work, update in-progress items
4. **PUSH TO REMOTE** - This is MANDATORY:
   ```bash
   git pull --rebase
   bd dolt push
   git push
   git status  # MUST show "up to date with origin"
   ```
5. **Clean up** - Clear stashes, prune remote branches
6. **Verify** - All changes committed AND pushed
7. **Hand off** - Provide context for next session

**CRITICAL RULES:**
- Work is NOT complete until `git push` succeeds
- NEVER stop before pushing - that leaves work stranded locally
- NEVER say "ready to push when you are" - YOU must push
- If push fails, resolve and retry until it succeeds
