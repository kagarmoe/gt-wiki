# Wiki Log

Chronological event log. Append-only. Each entry:
`## [YYYY-MM-DD] <verb> | <subject>`.

Verbs: `ingest`, `query`, `experiment`, `lint`, `decision`, `drift-found`.

---

## [2026-04-11] decision | wiki scaffolded

- Created `~/repos/wiki/` per Karpathy's "LLM Wiki" pattern.
- Purpose: compounding knowledge base, starting with the gastown drift investigation.
- Schema in `CLAUDE.md` (v1). Topic namespacing, YAML frontmatter required, flexible entity-page skeleton, 3-level nesting cap, relative markdown links (not wikilinks), 17 page types (open for extension), closed log-verb starter set.
- Permission scoping via `.claude/settings.local.json` (gitignored): full Read/Write/Edit/Bash within `~/repos/wiki/`, read-only outside.
- First topic: gastown.

## [2026-04-11] experiment | docker compose build with golang:1.26-bookworm

- Context: building `~/gt/Dockerfile` so `gt` can run inside a container.
- Switched base image from `debian:bookworm` (+ Go tarball) back to `golang:1.26-bookworm`, unpinning Go patch version.
- `docker compose build` succeeded through apt + dolt install steps.
- `go install github.com/steveyegge/gastown/cmd/gt@latest` inside the running container installed to `/go/bin/gt` (GOPATH in the `golang:` image is `/go`, not `/root/go` as the original Dockerfile assumed).
- `gt completion bash` build step failed: binary self-killed with "ERROR: This binary was built with 'go build' directly. macOS will SIGKILL unsigned binaries. Use 'make build' instead."
- Root cause: `/home/kimberly/repos/gastown/internal/cmd/root.go:96-106` — check fires when `BuiltProperly == "" && Build == "dev"`. `go install` does not set the `BuiltProperly` ldflag.
- → [gastown/binaries/gt.md](gastown/binaries/gt.md), [gastown/files/makefile.md](gastown/files/makefile.md)

## [2026-04-11] ingest | /home/kimberly/repos/gastown/internal/cmd/root.go

- Read the self-kill check at lines 96-106.
- Bypass: `-ldflags "-X github.com/steveyegge/gastown/internal/cmd.BuiltProperly=1"`.
- Stale comment on line 97 claims "Warning only - doesn't block execution" — but code on line 106 calls `os.Exit(1)`.
- Error message attributes the issue to macOS SIGKILL of unsigned binaries, but the check fires on Linux too.
- → [gastown/binaries/gt.md](gastown/binaries/gt.md)

## [2026-04-11] ingest | /home/kimberly/repos/gastown/Makefile

- Captured canonical LDFLAGS recipe, including `-X ...BuiltProperly=1` (the self-kill bypass).
- Makefile `build` target produces three binaries: `gt`, `gt-proxy-server`, `gt-proxy-client`. README only documents `gt`; the two proxy binaries are undocumented.
- Install dir: `$(HOME)/.local/bin`.
- → [gastown/files/makefile.md](gastown/files/makefile.md), [gastown/binaries/gt.md](gastown/binaries/gt.md)

## [2026-04-11] decision | gastown purpose reframe + Obsidian vault + task-tracking split (schema v1.1)

- **gastown topic purpose reframed.**
  [gastown/README.md](gastown/README.md) rewritten to make the mission
  explicit: the wiki maps gastown code to produce an **authoritative
  rewrite of gastown's documentation**. Code is the source of truth;
  docs are downstream and not authoritative. The Docker build at
  `~/gt/` is a concrete milestone along that path. "Drift" items are
  the work list for the doc rewrite, not passive observation. Project
  purpose lives in the topic README, not in CLAUDE.md, per separation
  of concerns (Claude/wiki mechanics in CLAUDE.md; project purpose in
  the topic).
- **Obsidian vault** initialized via `.obsidian/app.json` with
  wiki-aligned link config: `useMarkdownLinks: true`,
  `newLinkFormat: "relative"`, `alwaysUpdateLinks: true`. Matches the
  CLAUDE.md rule against `[[wikilinks]]`. Ephemeral Obsidian state
  (`workspace.json`, caches) is gitignored.
- **Task tracking split** between two tools by audience:
  - Agents use **beads** (`.beads/` with embedded local dolt —
    operationally independent from the Gas Town dolt server on port
    3307). Committed `bd init` scaffolding as its own commit.
  - Kimberly uses the **Obsidian Tasks plugin** for personal tasks
    (plain GFM checkboxes aggregated by plugin queries).
  - Cross-tool handoff via the `wants-wiki-entry` bead label
    (agent → Kimberly) and `→ handed to bd-<id>` notation on closed
    Kimberly tasks (Kimberly → agent). No sync bridge.
- Did **not** add a `task` wiki page type or `task-opened`/`task-closed`
  log verbs — both trackers have native state semantics and the wiki
  schema should not duplicate them. The log captures *findings* and
  *decisions*, not task state transitions.
- Wiki-specific bd conventions: `--actor wiki-curator` for
  LLM-originated beads; labels `wiki-investigation`, `wiki-content`,
  `drift`, plus topic (`gastown`), plus `wants-wiki-entry` for
  handoff. Descriptions must link wiki entity pages and cite source
  `file:line` references.
- Seeded three beads from the 2026-04-11 scaffolding session's open
  investigation threads:
  - `wiki-ztf` — Verify BuiltProperly ldflag bypass in Docker build.
  - `wiki-13o` — Document undocumented proxy binaries
    (`gt-proxy-server`, `gt-proxy-client`).
  - `wiki-cqx` — Annotate stale comment + misattributed error on
    `internal/cmd/root.go`.
- **Schema version:** bumped to v1.1. CLAUDE.md updated: directory
  layout tree adds `.obsidian/`, `.beads/`, `.claude/plans/`,
  `.claude/llm-wiki.md`, `AGENTS.md`; new "Task tracking" section
  added.
- → [CLAUDE.md](CLAUDE.md), [gastown/README.md](gastown/README.md),
  `.gitignore`, `.obsidian/app.json`
  (`.beads/` scaffolding and `AGENTS.md` were committed earlier by
  `bd init` in commit `0532fe7`, not this decision)
