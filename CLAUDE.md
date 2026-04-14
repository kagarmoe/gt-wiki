# CLAUDE.md — Wiki Schema

**Schema version:** 1.1 (2026-04-11)

## Purpose

This is an LLM-maintained knowledge wiki following Karpathy's "LLM Wiki"
pattern. It's a compounding artifact: findings, cross-references, and
syntheses accumulate here rather than being re-derived every session.

**Scope (current):** the `gastown` project — its source code, documentation,
build system, runtime behavior, and the drift between docs and code.

**Scope (planned):** will grow to cover other repos in `~/repos/`, and may
eventually absorb content from the personal Obsidian vault at
`~/Documents/obsidian/`.

**Ownership:** the LLM writes and maintains this wiki. Kimberly curates
sources, directs investigation, asks questions, and reviews changes. Raw
sources are never modified — only read.

## Layers

1. **Raw sources (immutable):** source repos under `~/repos/<name>/`,
   documents Kimberly drops in, command outputs captured during
   investigations. Read-only. Source of truth.
2. **Wiki (this directory):** LLM-owned markdown. Pages, index, log. Fully
   rewriteable.
3. **Schema (this file):** conventions and workflows. Co-evolves with the
   wiki as patterns emerge.

## Directory layout

```
~/repos/gt-wiki/
  CLAUDE.md          # this file
  AGENTS.md          # agent-facing task-tracking entrypoint (bd integration)
  index.md           # global catalog of all pages
  log.md             # chronological event log
  .claude/
    settings.json         # Claude Code settings (committed)
    settings.local.json   # permission scoping; gitignored
    plans/                # LLM workflow plans; gitignored
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
    ...              # create additional categories as content arrives
```

Sub-folders are created lazily. Don't scaffold empty folders.

**Nesting is capped at 3 levels:** `wiki/<topic>/<category>/<entity>.md`. If
a page feels like it needs deeper nesting, split the category into sibling
categories instead.

## Page conventions

**Naming:** kebab-case filenames matching the entity (`gt.md`,
`mayor-attach.md`, `gt-proxy-server.md`).

**Wiki-internal links** use relative markdown links:
`[gt completion](../commands/completion.md)`. Do NOT use Obsidian wikilinks
(`[[...]]`) — they can't disambiguate when two topics have a page with the
same basename.

**External source refs** (to files outside the wiki) use absolute paths in
backticks, optionally with line numbers:
`/home/kimberly/repos/gastown/internal/cmd/root.go:96-106`. Obsidian cannot
follow out-of-vault links, so treat these as grep/copy-paste targets rather
than clickable navigation.

### YAML frontmatter (required)

Every page starts with frontmatter:

```yaml
---
title: gt
type: binary            # see type list below
status: verified        # verified | partial | stub
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:                # raw source paths this page is grounded in
  - /home/kimberly/repos/gastown/cmd/gt/
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [cli, build]      # free-form, lowercase, kebab-case
---
```

Frontmatter is required from day one so Dataview queries work across the
whole wiki without needing a retrofit pass later.

### Page types (starter set — open for extension)

**Code surface:**

- `binary` — executables produced by the project
- `command` — CLI subcommands
- `package` — source code packages / modules (Go packages, directories)
- `file` — notable single files (Makefile, Dockerfile, entrypoint scripts)
- `data-type` — structs, schemas, data shapes

**Config & runtime surface:**

- `config-file` — shapes of config files
- `env-var` — environment variables
- `build-target` — Makefile targets, Docker build stages

**Design / mental model:**

- `concept` — abstract ideas
- `role` — personified agents or positions
- `service` — long-running processes
- `workflow` — multi-step flows

**External:**

- `dependency` — external tools the project relies on

**Meta (investigation artifacts):**

- `experiment` — captured runs, build attempts, test results
- `drift` — cross-entity drift themes (when docs are broadly wrong)
- `decision` — design or process decisions made during investigation
- `note` — freeform synthesis filed back from queries

**This set is open.** When a new kind of thing doesn't fit any existing
type, add a type to this schema and log the decision in `log.md`. Don't
force-fit.

### Entity-page starter template (flexible, not a contract)

Required: frontmatter. Recommended: the structure below as a scaffold.
Allowed: any additional structure the page needs.

```markdown
# <Title>

## What it actually does
<code-grounded description with absolute-path file:line references>

## Docs claim
<what README, docs/, help text say — with source references>

## Drift
<contradictions between "actually does" and "docs claim">

## Notes / open questions
```

Drift annotations live **inline on the entity page**, not as separate files.
Truth and lie stay adjacent. A separate `<topic>/drift/` folder may hold
cross-entity drift themes ("the whole install story is wrong") without
double-maintaining per-entity details.

## Ingest workflow

When a new source is read (a file, a command output, a doc page):

1. Note the source path / identifier.
2. Read it in full.
3. Discuss key takeaways with Kimberly before writing (unless she has
   delegated).
4. For each entity the source touches, update or create the entity page.
5. If the source is a doc making claims, add those to the entity's "Docs
   claim" section with a source reference.
6. Update `index.md` if new pages were created.
7. Append an entry to `log.md` with the `ingest` verb.

## Query workflow

When Kimberly asks a question:

1. Read `index.md` to find candidate pages.
2. Read candidate pages in full (not chunks).
3. Synthesize an answer grounded in wiki content, with links to the pages
   used.
4. **If the answer is a valuable synthesis, file it back as a new page**
   under `<topic>/concepts/` or `<topic>/notes/`, update `index.md`, and log
   with the `query` verb.

## Lint workflow

Periodically or on request:

- Orphan pages (not linked from `index.md` or any other page)
- Broken cross-links
- Claims contradicted by newer sources
- Stale `updated` dates on pages that should be revisited
- Entity mentions that should have their own page but don't
- Gaps that could be filled by reading a specific file

**Lint produces findings, not fixes.** Discuss with Kimberly before acting.
Log with the `lint` verb.

## Task tracking

Task tracking uses two tools, split by who is doing the work:

### Agent work → beads (`bd`)

Agent investigation threads, follow-ups, and anything a future Claude
Code session (or Gas Town crew agent) should pick up live in **beads**.
The wiki has its own embedded `.beads/` database (local dolt), not
connected to the Gas Town dolt server on port 3307 — the wiki is
operationally independent.

See `~/repos/CLAUDE.md` for generic `bd` usage (`bd ready`, `bd show`,
`bd update --claim`, `bd close`, etc.). **Wiki-specific conventions on
top of that:**

- **Actor:** LLM sessions working on the wiki use
  `--actor wiki-curator` so audit trails distinguish wiki-curation
  beads from other work. Set via `BEADS_ACTOR=wiki-curator` in the
  session env, or pass `--actor wiki-curator` on each command.
- **Core labels (extend as needed):**
  - `wiki-investigation` — open investigation thread (verify something,
    read a source, capture an outcome).
  - `wiki-content` — a page needs writing or updating.
  - `drift` — drift-annotation work on an existing page.
  - `<topic>` — topic scoping (`gastown`, etc.).
  - `wants-wiki-entry` — handoff from agent to Kimberly (see below).
  - Free-form tags (e.g. `docker`, `makefile`) are fine when they help
    narrow a query — add them alongside the core labels, not in place
    of them.
- **Descriptions must link to wiki pages and source refs:** every bead
  description should name the wiki entity pages it touches (absolute
  paths, e.g. `~/repos/gt-wiki/gastown/binaries/gt.md`) and cite
  source-code references in the form `file:line` for anything in
  `~/repos/gastown/`.
- **Outcome filing:** when closing a bead that produced a finding,
  append an entry to `log.md` with the appropriate verb (`ingest`,
  `experiment`, `drift-found`) and update the relevant entity page.
  The bead closure itself is not a `log.md` entry — the finding is.

### Kimberly's work → Obsidian Tasks plugin

Kimberly's personal tasks (curation decisions, questions to
investigate, reading queue, things she wants to revisit) live as GFM
checkboxes **anywhere in the vault**, aggregated via the Obsidian
**Tasks plugin**.

- Plain `- [ ]` checkboxes are sufficient. Tasks plugin emoji metadata
  (`📅` due date, `⏫` priority, `🔁` recurrence) is optional.
- No fixed file location — file tasks wherever they fit (inline on an
  entity page, in a personal notes file, etc.). Tasks plugin scans the
  whole vault.
- Tasks plugin query blocks aggregate on demand:

  ````markdown
  ```tasks
  not done
  sort by priority
  ```
  ````

### Cross-tool handoff

The two trackers occasionally hand work off. Use conventions, not a
sync bridge:

- **Agent → Kimberly:** the agent files a bead normally and adds the
  `wants-wiki-entry` label. Kimberly surfaces these with
  `bd list -l wants-wiki-entry`. She either resolves the bead herself
  and closes it, or promotes it into her Tasks-plugin tracker (and
  closes the bead with a pointer to where she filed it).
- **Kimberly → agent:** Kimberly closes her checkbox with a trailing
  pointer: `- [x] ... → handed to bd-<id>`. She then runs `bd create`
  with whatever actor makes sense (the default git-user identity is
  fine for human-originated beads).

**Do not duplicate tasks across trackers.** A task lives in exactly
one system at a time; handoff moves it across.

## index.md format

One section per topic. Within each section, pages grouped by category, one
line each with a brief descriptor.

## log.md format

Append-only. Each entry starts with `## [YYYY-MM-DD] <verb> | <subject>` so
`grep "^## \[" log.md` gives a timeline.

**Verb starter set** (closed for now; loosen later if needed):
`ingest`, `query`, `experiment`, `lint`, `decision`, `drift-found`.

## Schema evolution

This schema is **version 1.1**. It will change as the wiki grows. Update this
file when new page types emerge, when conventions start hurting rather than
helping, or when a better workflow appears. Note schema changes in `log.md`
with the `decision` verb. Old pages don't need to be retrofitted unless a
lint pass says so.

## Maintenance principles

- **Don't redo work.** If something has been captured, trust it unless
  evidence says otherwise.
- **When evidence contradicts a page, update the page** and note the
  correction in `log.md`.
- **Code references anchor truth.** Every "what it actually does" claim
  should cite `file:line` in the raw source.
- **Keep pages short and linkable.** Split long pages into sub-pages that
  reference each other.
- **Drift is information, not embarrassment.** Record it plainly. The goal
  is an accurate map.

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
beads), not to human curation work in the vault. See the "Task
tracking" section above for the wiki's full policy: bd for agents,
Tasks plugin for Kimberly, with documented cross-tool handoff.

<!-- BEGIN BEADS INTEGRATION v:1 profile:minimal hash:ca08a54f -->
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
<!-- END BEADS INTEGRATION -->
