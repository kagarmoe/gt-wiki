---
title: gastown topic landing
type: note
status: stub
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/
tags: [landing, index, mission]
---

# gastown

## Mission

This topic maps the gastown codebase in order to produce an
**authoritative rewrite of gastown's documentation**. The existing
gastown docs (README, inline help, comments) are **not authoritative** —
when docs and code disagree, code wins. The deliverable of this topic is
an updated doc set that aligns with what gastown actually does.

**Concrete milestone:** get `gt` to build and run inside Docker at
`~/gt/`. Reaching that milestone exercises the full
build → install → run path and will surface (or retire) most of the doc
drift worth fixing on the critical path.

**Working model:**

- Read from `/home/kimberly/repos/gastown/` (source of truth, never
  modified).
- File findings as wiki pages under this topic (entity pages with
  frontmatter + "What it actually does" + "Docs claim" + "Drift").
- Track follow-up investigation work in **beads** — run
  `bd list -l gastown` for the live queue. See the "Task tracking"
  section of `../CLAUDE.md` for bd conventions.
- Once wiki pages converge on actual behavior, produce the upstream doc
  edits as the final output.

## Coordinates

- **Source under investigation:** `/home/kimberly/repos/gastown/`
- **Upstream:** https://github.com/gastownhall/gastown
- **Go module path declared in `go.mod`:** `github.com/steveyegge/gastown`
  (the `gastownhall` repo self-identifies under the `steveyegge`
  namespace, so `go install` must use the `steveyegge` path)
- **Runtime workspace (separate):** `~/gt/` — the Docker-containerized
  deployment of `gt` that we are trying to get working. This is the
  *target* of the mapping effort, not part of the wiki itself.

## Scope

gastown is a multi-agent orchestration system for AI coding runtimes,
managed by the `gt` CLI. This topic covers:

- The `gt` binary and its CLI surface
- Build system, Makefile recipes, Docker build stages
- Docker runtime environment (entrypoint, compose, volumes)
- Internal packages, roles, services, workflows
- Drift between current documentation and code reality — tracked here
  as **the work list for the doc rewrite**, not as passive observation.

## Drift to resolve (as of 2026-04-11)

Each item is a mismatch between current gastown docs and actual code
behavior. Resolving an item means: confirm the behavior against code,
update the relevant wiki entity page, and produce the upstream doc edit.
Active investigation is tracked in beads — run `bd list -l gastown` for
the live queue.

1. **README `go install` instructions produce a self-killing binary.**
   Correct build path is `make build` or `go install` with
   `-ldflags "-X github.com/steveyegge/gastown/internal/cmd.BuiltProperly=1"`.
   See [gt](binaries/gt.md) and [Makefile](files/makefile.md). On the
   critical path for the Docker milestone — tracked in beads as a
   `wiki-investigation` task.
2. **`make build` produces three binaries, README documents one.**
   `gt`, `gt-proxy-server`, and `gt-proxy-client` are all built;
   `gt-proxy-server` and `gt-proxy-client` have no upstream
   documentation at all. See [Makefile](files/makefile.md). Tracked in
   beads as a `wiki-content` task; requires reading the proxy `main.go`
   sources and creating new entity pages.
3. **Error message at `/home/kimberly/repos/gastown/internal/cmd/root.go:101`
   claims "macOS will SIGKILL"** but the check fires on Linux too.
   Misattributes the cause. Tracked in beads as a `drift` task
   (annotate on the `gt` entity page; no code fix).
4. **Stale comment at
   `/home/kimberly/repos/gastown/internal/cmd/root.go:97`** claims the
   self-kill check is "Warning only - doesn't block execution" — but
   line 106 calls `os.Exit(1)`. Factually wrong. Tracked in beads
   alongside the error-message drift (same entity page).

## Sub-index

### Binaries

- [gt](binaries/gt.md) — main CLI

### Files

- [Makefile](files/makefile.md) — canonical build recipe
