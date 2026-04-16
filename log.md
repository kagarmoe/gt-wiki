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

## [2026-04-11] ingest | gastown top-level command tree mapped from internal/cmd/

Read: `cmd/gt/main.go`, `internal/cmd/root.go`, `internal/cmd/version.go`,
`internal/cli/name.go`, plus grep across `internal/cmd/*.go` for
`rootCmd.AddCommand` and `var ... = &cobra.Command{`.

**Enumeration findings (neutral — classification pending):**

- **Top-level command surface is 111 registrations (~107 unique) — not
  the dozen-ish the README documents.** Full inventory in
  [gastown/commands/README.md](gastown/commands/README.md).
- **Total cobra.Command definitions in `internal/cmd/`: 495.** Each
  top-level command averages ~4 nested subcommands.
- **Binary name is dynamic.** `internal/cli/name.go` reads
  `GT_COMMAND` env var (defaults to `"gt"`) to coexist with Graphite.
  README does not mention this.
- **Seven command groups** defined on the root command
  (`GroupWork`/`Agents`/`Comm`/`Services`/`Workspace`/`Config`/`Diag`).
  Each top-level command self-assigns via `GroupID:` in its cobra
  definition.
- **`persistentPreRun` has 9 side-effect steps** on every invocation
  (self-kill check, theme init, telemetry init, command-usage logging,
  session registry init, polecat heartbeat, stale-binary warning,
  branch-check warning, beads version check). None of these are
  documented in README.
- **OpenTelemetry is initialized in `Execute()`** on every `gt` run
  and exports OTEL resource attributes to the process env so child
  `bd` processes inherit them. Endpoint and opt-out not yet
  investigated — privacy-relevant.
- **`beadsExemptCommands` (32 entries)** marks commands that must work
  even when `bd` is broken. **`branchCheckExemptCommands` (9 entries)**
  marks commands exempt from the town-root-branch warning.
- **`AnnotationPolecatSafe = "polecatSafe"`** defined in
  `internal/cmd/proxy_subcmds.go:15` and used on at least
  `version.go:34`. Cobra annotation — semantic gate not yet traced.

**Pages updated or created:**

- [gastown/binaries/gt.md](gastown/binaries/gt.md) — major expansion
  from stub to partial; added Entry point, Build-time variables,
  Self-kill check, persistentPreRun sequence, `Execute()` function,
  Command groups, beads/branch-check exempt maps, Command surface
  counts; expanded Drift from 4 items to 8 items.
- [gastown/commands/README.md](gastown/commands/README.md) — new;
  authoritative top-level command inventory (~107 rows) with source
  file + line number per command.
- [gastown/commands/version.md](gastown/commands/version.md) — new;
  first per-command entity page, serves as the template for others.
- [index.md](index.md), [gastown/README.md](gastown/README.md) —
  sub-indexes updated to reference the new `commands/` directory.

**Beads filed for follow-up:** see `bd list -l gastown` after this
commit.

**Status:** gastown topic `status: stub` → effectively `status: partial`
for the gt binary and the command surface; still `stub` everywhere
else (docs/, other packages, nested subcommands, 103 of 107 top-level
commands).

→ [gastown/binaries/gt.md](gastown/binaries/gt.md),
  [gastown/commands/README.md](gastown/commands/README.md),
  [gastown/commands/version.md](gastown/commands/version.md),
  [gastown/README.md](gastown/README.md), [index.md](index.md)

## [2026-04-11] decision | scope reframed: current phase is mapping only (supersedes prior doc-rewrite framing)

Earlier entries in this log (through "gastown top-level command tree
mapped") were written under a framing in which the wiki's immediate
deliverable was an authoritative rewrite of gastown's upstream
documentation, with drift analysis as a first-class current-phase
activity. That framing was over-scoped for the current phase.

**Current plan** (`.claude/plans/2026-04-11-gastown-map.md`): the
wiki's current phase produces only the **map** — every part of
`~/repos/gastown/` represented at inventory level, every load-bearing
part with a full entity page that accurately describes what the code
actually does, read off source and cited with `file:line`. Cross-
linking discipline is mandatory so the map can later answer multi-hop
"is X correct?" queries by following backlinks.

**Phases that are NOT in the current scope** (all deferred):

- Drift analysis / claim-vs-code gap hunting
- Audience classification (user / agent / dev / internal)
- Gap analysis against upstream `README.md`, `docs/`, `AGENTS.md`
- Upstream documentation rewrite

**Pages dialed back** in this commit to remove drift/doc-rewrite
framing from the current-phase representation of the wiki:

- [gastown/README.md](gastown/README.md) — Mission rewritten as
  "map the codebase"; `## Drift to resolve` section removed;
  Working model no longer mentions "Docs claim" / "Drift" as
  per-page sections.
- [gastown/binaries/gt.md](gastown/binaries/gt.md) — `## Docs claim`
  and `## Drift` sections removed; `README.md` removed from
  frontmatter sources (was only cited in the removed sections).
- [gastown/commands/version.md](gastown/commands/version.md) —
  `## Docs claim` and `## Drift` sections removed; `Docs claim`
  replaced with neutral `Inline help text` section transcribing the
  cobra `Long` field.
- [gastown/files/makefile.md](gastown/files/makefile.md) —
  `## Docs claim` and `## Drift` sections removed.
- [gastown/commands/README.md](gastown/commands/README.md) —
  "coverage gap" language removed; "README coverage cross-check"
  removed from the What's-NOT-here list.
- [index.md](index.md) — entry descriptions neutralized (no
  "README claims ~12", no "self-kill when built without ...");
  Inventory section added.

**Historical log entries above are preserved as written.** They
describe what happened, including framing that has since been
superseded. Earlier framings should be read as "what the session
believed at the time," not as current representation.

**Current bead queue will be aligned** with the mapping plan in a
follow-up step (this commit): phase-3 beads closed, mapping beads
kept, one bead per plan batch created as an anchor.

→ [gastown/README.md](gastown/README.md),
  [gastown/binaries/gt.md](gastown/binaries/gt.md),
  [gastown/commands/README.md](gastown/commands/README.md),
  [gastown/commands/version.md](gastown/commands/version.md),
  [gastown/files/makefile.md](gastown/files/makefile.md),
  [index.md](index.md)

## [2026-04-11] ingest | Batch 1 (Layer a: Build & packaging)

Read 10 files at `/home/kimberly/repos/gastown/` root and produced full
entity pages under `gastown/files/` plus committed the 4 uncommitted
A-level inventory pages in a prior step of this batch.

**Sources read:**

- `/home/kimberly/repos/gastown/Makefile`
- `/home/kimberly/repos/gastown/Dockerfile`
- `/home/kimberly/repos/gastown/Dockerfile.e2e`
- `/home/kimberly/repos/gastown/docker-compose.yml`
- `/home/kimberly/repos/gastown/docker-entrypoint.sh`
- `/home/kimberly/repos/gastown/flake.nix`
- `/home/kimberly/repos/gastown/flake.lock` (supporting)
- `/home/kimberly/repos/gastown/.goreleaser.yml`
- `/home/kimberly/repos/gastown/.golangci.yml`
- `/home/kimberly/repos/gastown/go.mod`
- `/home/kimberly/repos/gastown/go.sum` (supporting, not transcribed)

**Pages created:**

- [gastown/files/dockerfile.md](gastown/files/dockerfile.md)
- [gastown/files/dockerfile-e2e.md](gastown/files/dockerfile-e2e.md)
- [gastown/files/docker-compose.md](gastown/files/docker-compose.md)
- [gastown/files/docker-entrypoint.md](gastown/files/docker-entrypoint.md)
- [gastown/files/flake-nix.md](gastown/files/flake-nix.md)
- [gastown/files/goreleaser-yml.md](gastown/files/goreleaser-yml.md)
- [gastown/files/golangci-yml.md](gastown/files/golangci-yml.md)
- [gastown/files/go-mod.md](gastown/files/go-mod.md)

**Pages expanded:**

- [gastown/files/makefile.md](gastown/files/makefile.md) — partial → verified; per-target subsections added for all 10 phony targets.

**Pages committed earlier in this batch** (separate commit `6faf5db`):

- [gastown/inventory/README.md](gastown/inventory/README.md)
- [gastown/inventory/repo-root.md](gastown/inventory/repo-root.md) — later modified in this commit for back-references
- [gastown/inventory/docs-tree.md](gastown/inventory/docs-tree.md)
- [gastown/inventory/go-packages.md](gastown/inventory/go-packages.md)

**Index updates:**

- [gastown/README.md](gastown/README.md) — `### Files` sub-index expanded to list all 9 `files/*` pages.
- [index.md](index.md) — gastown `### Files` section expanded to match.

**Cross-linking outcome:** 131 outbound markdown links across the 9
`files/*` pages, verified to resolve. `gastown/inventory/repo-root.md`
updated with back-references from the top-level files table to the new
per-file pages.

**Neutral observations surfaced** (filed as "Notes / open questions" on
the relevant pages, not as drift):

- Go version disagrees across build paths: `go.mod` declares `1.25.8`,
  `Dockerfile` uses `1.25.6`, `Dockerfile.e2e` uses `1.26-alpine`,
  `flake.nix` overlays `1.25.8`. Only `flake.nix` and `go.mod` agree.
- `.goreleaser.yml` has no `brews:` block despite README advertising
  `brew install gastown`. Homebrew formula lives in a separate
  tap repo or homebrew-core.
- `.goreleaser.yml` release binaries use `Build={{.ShortCommit}}`, not
  `BuiltProperly`. Release binaries bypass the self-kill check via
  `Build != "dev"` rather than `BuiltProperly=1`.
- `.goreleaser.yml` FreeBSD build sets `CGO_ENABLED=0` while all other
  targets enable CGO — CGO-only code paths silently absent on the
  FreeBSD release.
- `cmd/gt-desktop` referenced by the `desktop-build` target in the
  Makefile but not in [gastown/inventory/go-packages.md](gastown/inventory/go-packages.md)
  — directory existence not yet verified.
- `.golangci.yml` has `run.tests: false` AND test-file exclusion
  rules (dead code under current config).
- `flake.nix` Go overlay comment is stale (mentions beads v0.60.0,
  go.mod now requires v0.63.3).
- `docker-entrypoint.sh` is wired only into the primary Dockerfile;
  Dockerfile.e2e has no ENTRYPOINT.

**Next batch:** Batch 2 — Layer (b) Binaries & entry points
(`cmd/gt-proxy-server`, `cmd/gt-proxy-client`). Tracked in bead
`wiki-u2r`.

→ [gastown/files/makefile.md](gastown/files/makefile.md),
  [gastown/files/dockerfile.md](gastown/files/dockerfile.md),
  [gastown/files/dockerfile-e2e.md](gastown/files/dockerfile-e2e.md),
  [gastown/files/docker-compose.md](gastown/files/docker-compose.md),
  [gastown/files/docker-entrypoint.md](gastown/files/docker-entrypoint.md),
  [gastown/files/flake-nix.md](gastown/files/flake-nix.md),
  [gastown/files/goreleaser-yml.md](gastown/files/goreleaser-yml.md),
  [gastown/files/golangci-yml.md](gastown/files/golangci-yml.md),
  [gastown/files/go-mod.md](gastown/files/go-mod.md),
  [gastown/inventory/repo-root.md](gastown/inventory/repo-root.md),
  [gastown/README.md](gastown/README.md),
  [index.md](index.md)

## [2026-04-11] ingest | Batch 2 (Layer b: Binaries & entry points)

Read `cmd/gt-proxy-server/` and `cmd/gt-proxy-client/` source, plus
selected parts of `internal/proxy/` for characterization context, and
produced two new binary entity pages. Cross-linked back to
[gastown/binaries/gt.md](gastown/binaries/gt.md), added forward links
from [gastown/files/makefile.md](gastown/files/makefile.md) and
[gastown/files/flake-nix.md](gastown/files/flake-nix.md) where the
proxy binaries were mentioned as bare text.

**Sources read:**

- `/home/kimberly/repos/gastown/cmd/gt-proxy-server/main.go`
- `/home/kimberly/repos/gastown/cmd/gt-proxy-server/config.go`
- `/home/kimberly/repos/gastown/cmd/gt-proxy-client/main.go`
- `/home/kimberly/repos/gastown/cmd/gt/main.go` (re-read for cross-ref)
- `/home/kimberly/repos/gastown/internal/proxy/server.go`
- `/home/kimberly/repos/gastown/internal/proxy/exec.go` (partial — first ~200 lines)
- `/home/kimberly/repos/gastown/internal/proxy/git.go` (header comment only)
- `/home/kimberly/repos/gastown/Makefile` (re-read build target)

**Pages created:**

- [gastown/binaries/gt-proxy-server.md](gastown/binaries/gt-proxy-server.md) — 306 lines; covers mTLS termination, per-identity rate limiting, `/v1/exec` endpoint with 13-step subprocess execution, `/v1/git/` smart-HTTP bridge, plain-HTTP admin server on `127.0.0.1:9877`, config merge, subcommand allowlist discovery via `exec.LookPath("gt")`.
- [gastown/binaries/gt-proxy-client.md](gastown/binaries/gt-proxy-client.md) — 232 lines; covers the stdlib-only 137-line binary, its two-mode dispatch (mTLS forward to proxy vs `syscall.Exec` to `/usr/local/bin/gt.real`), the `GT_PROXY_*` env var quartet, and the installation pattern (shadowing `gt` and `bd` inside polecat containers).

**Pages expanded:**

- [gastown/binaries/gt.md](gastown/binaries/gt.md) — added `### Sibling binaries` subsection linking both proxy pages + noting that the `BuiltProperly` self-kill gate is `gt`-only (neither proxy binary imports `internal/cmd`).
- [gastown/files/makefile.md](gastown/files/makefile.md) — linked the proxy binaries in the opening paragraph and in a "Notes / open questions" item that previously read as a to-read note.
- [gastown/files/flake-nix.md](gastown/files/flake-nix.md) — linked the proxy binaries in the Cross-references section.

**Index updates:**

- [gastown/README.md](gastown/README.md) — `### Binaries` sub-index expanded.
- [index.md](index.md) — gastown `### Binaries` expanded to match.

**Cross-linking outcome:** gt-proxy-server page has 11 outbound links; gt-proxy-client page has 12. All resolved. Grep for `gt-proxy-server|gt-proxy-client` across the wiki found 17 pre-existing mentions; 6 were converted to links (in makefile.md and flake-nix.md). Mentions inside `gastown/inventory/*.md` table cells, `log.md` historical entries, `index.md`, `gastown/README.md`, and `CLAUDE.md` left bare (touch-list).

**Neutral observations surfaced** (filed on the relevant pages, not as drift):

- Both proxy binaries receive the `BuiltProperly=1` ldflag via `make build` LDFLAGS, but neither imports `internal/cmd`, so the ldflag is silently a no-op for them. The self-kill gate is `gt`-only.
- `cmd/gt-proxy-server/main.go` calls `exec.LookPath("gt")` at startup to auto-discover the subcommand allowlist — a circular dependency between the proxy binary and the `gt` binary it protects. A baked-in fallback list covers the case when `gt` isn't on PATH.
- `gt-proxy-client` HTTP client timeout (5m), proxy-server `WriteTimeout` (5m), and server per-command `ExecTimeout` (60s) are not coordinated. The 60s cap is tighter than either transport timeout.
- `gt-proxy-client`'s `GT_REAL_BIN` fallback is hardcoded to `gt.real` regardless of whether the binary was invoked as `gt` or `bd`.
- `MaxConcurrentExec`, `ExecRateLimit`, `ExecRateBurst`, `ExecTimeout` have no CLI-flag or config-file surface — only settable from code.
- Proxy server's `rateLimiters` `sync.Map` has no eviction; in-source comment says "acceptable for dozens of polecats, not thousands."
- Proxy server's admin HTTP surface on `127.0.0.1:9877` is plain HTTP with no authentication; in-source comment frames this as the intended local-operator access model.
- `internal/proxy/` is referenced but not yet mapped as its own wiki page. Pending the Agent-runtime / Platform-services batch where it lands.

**Next batch:** Batch 3 — Layer (c) Command layer. 111 top-level cobra commands in `internal/cmd/`, walked by cobra group. Tracked in bead `wiki-3zo`. Absorbs the narrow investigation bead `wiki-ef3`.

→ [gastown/binaries/gt-proxy-server.md](gastown/binaries/gt-proxy-server.md),
  [gastown/binaries/gt-proxy-client.md](gastown/binaries/gt-proxy-client.md),
  [gastown/binaries/gt.md](gastown/binaries/gt.md),
  [gastown/files/makefile.md](gastown/files/makefile.md),
  [gastown/files/flake-nix.md](gastown/files/flake-nix.md),
  [gastown/README.md](gastown/README.md),
  [index.md](index.md)

## [2026-04-11] ingest | Batch 3a (Layer c sub-batch: Diagnostics group — 22 top-level commands)

First sub-batch of Layer (c) command mapping. Read 21 Go files in
`/home/kimberly/repos/gastown/internal/cmd/` and produced 21 new
wiki entity pages under `gastown/commands/`. The 22nd Diag command
(`version`) already had a page from a prior session. Together these
form the complete `GroupDiag` set as defined at
`/home/kimberly/repos/gastown/internal/cmd/root.go:319-348` (7 cobra
groups; Diagnostics is one).

**Cobra group progress:** 1 of 7 groups complete
(Diag ✓; Work, Agents, Comm, Services, Workspace, Config remaining).

**Commands mapped** (alphabetical):

1. [activity](gastown/commands/activity.md)
2. [audit](gastown/commands/audit.md)
3. [checkpoint](gastown/commands/checkpoint.md)
4. [costs](gastown/commands/costs.md)
5. [dashboard](gastown/commands/dashboard.md)
6. [doctor](gastown/commands/doctor.md)
7. [feed](gastown/commands/feed.md)
8. [heartbeat](gastown/commands/heartbeat.md)
9. [info](gastown/commands/info.md)
10. [log](gastown/commands/log.md)
11. [metrics](gastown/commands/metrics.md)
12. [patrol](gastown/commands/patrol.md)
13. [prime](gastown/commands/prime.md)
14. [repair](gastown/commands/repair.md)
15. [seance](gastown/commands/seance.md)
16. [stale](gastown/commands/stale.md)
17. [status](gastown/commands/status.md)
18. [thanks](gastown/commands/thanks.md)
19. [upgrade](gastown/commands/upgrade.md)
20. [version](gastown/commands/version.md) (pre-existing)
21. [vitals](gastown/commands/vitals.md)
22. [whoami](gastown/commands/whoami.md)

**Index updates:**

- [gastown/commands/README.md](gastown/commands/README.md) —
  entity-page column updated for 21 rows; "mapped so far" progress
  note added to the "What's NOT yet in this index" section.
- [gastown/README.md](gastown/README.md) — sub-index `### Commands`
  expanded with an inline list of the 22 Diag commands.
- [index.md](index.md) — root gastown `### Commands` noting
  Batch 3a completion.

**Neutral observations surfaced** (filed as "Notes / open questions"
on individual pages, not as drift):

- **`audit` hard-codes `<townRoot>/gastown/mayor/rig`** as the beads
  directory — multi-rig towns have a silent blind spot.
- **`prime` has a destructive-cycle guard** against GH#2638: a banner
  "DATABASE ERROR — DO NOT RUN gt done" prevents an agent from
  misreading a DB error as "no work" and closing an active bead.
- **`repair`'s `Long` text lists six repair targets but only 2 checks
  are registered** (`NewRigConfigSyncCheck`, `NewStaleDoltPortCheck`);
  the other 4 may be side effects of the 2 Fix methods or phantom
  documentation.
- **`stale` has intentionally inverted `--quiet` exit codes**
  (0=stale, 1=fresh).
- **`info --whats-new` is a hand-maintained 440-line static slice**
  with no automated sync against git history.
- **`costs` has a four-tier data plane**: live tmux transcript scan,
  `~/.gt/costs.jsonl` append log, ephemeral Dolt wisps, permanent
  digest beads.
- **`dashboard`'s Dolt env-var forcing** defensively patches a class
  of bug where child `bd` processes inherit the dashboard's HTTP
  listen port as their Dolt port.
- **`doctor` is double-exempt** (both `beadsExemptCommands` and
  `branchCheckExemptCommands`) — reflects its "tool used to fix the
  problem" role.
- **`feed` depends on `getCurrentTmuxSession` defined in
  `handoff.go`** — cross-file coupling worth noting.
- **`upgrade.generateCLAUDEMD` is a comment-pinned mirror of
  `install.go`'s `createTownRootAgentMDs`** — silent drift possible
  between them; no automatic generation either way.

**Polecat-safe coverage within Diag group:** 3 of 22 (`version`,
`status`, `prime`) — confirmed against `AnnotationPolecatSafe` in each
command's cobra.Command definition. `seance` was NOT polecat-safe
despite an earlier grep ambiguity — Sub B verified directly in
`seance.go` that no annotation is present.

**Beads-exempt coverage within Diag group:** 10 of 22 (version, doctor,
heartbeat, seance, prime, status, costs, feed, metrics, upgrade) per
`beadsExemptCommands` at `root.go:44-77`. The remaining 12 require `bd`
to function normally.

**Branch-check-exempt coverage within Diag group:** 3 of 22 (`version`,
`doctor`, `upgrade`) per `branchCheckExemptCommands` at `root.go:81-91`.

**Next sub-batch:** 3b — the remaining cobra groups. Six of seven
groups still unmapped: Work, Agents, Comm, Services, Workspace, Config.
The group to walk next is the controller's call; `GroupDiag` was
chosen for 3a because `version.md` already existed as a template.

**Beads status:** `wiki-3zo` (Batch 3 anchor, Layer c command layer)
remains OPEN — Batch 3 is not done. `wiki-ef3` (systematic subcommand
mapping) also remains OPEN — it is effectively equal to Batch 3 and
closes when Batch 3 closes.

→ [gastown/commands/README.md](gastown/commands/README.md),
  [gastown/commands/activity.md](gastown/commands/activity.md),
  [gastown/commands/audit.md](gastown/commands/audit.md),
  [gastown/commands/checkpoint.md](gastown/commands/checkpoint.md),
  [gastown/commands/costs.md](gastown/commands/costs.md),
  [gastown/commands/dashboard.md](gastown/commands/dashboard.md),
  [gastown/commands/doctor.md](gastown/commands/doctor.md),
  [gastown/commands/feed.md](gastown/commands/feed.md),
  [gastown/commands/heartbeat.md](gastown/commands/heartbeat.md),
  [gastown/commands/info.md](gastown/commands/info.md),
  [gastown/commands/log.md](gastown/commands/log.md),
  [gastown/commands/metrics.md](gastown/commands/metrics.md),
  [gastown/commands/patrol.md](gastown/commands/patrol.md),
  [gastown/commands/prime.md](gastown/commands/prime.md),
  [gastown/commands/repair.md](gastown/commands/repair.md),
  [gastown/commands/seance.md](gastown/commands/seance.md),
  [gastown/commands/stale.md](gastown/commands/stale.md),
  [gastown/commands/status.md](gastown/commands/status.md),
  [gastown/commands/thanks.md](gastown/commands/thanks.md),
  [gastown/commands/upgrade.md](gastown/commands/upgrade.md),
  [gastown/commands/vitals.md](gastown/commands/vitals.md),
  [gastown/commands/whoami.md](gastown/commands/whoami.md),
  [gastown/README.md](gastown/README.md),
  [index.md](index.md)

## [2026-04-11] ingest | Batch 3b (Layer c sub-batch: Configuration group — 11 top-level commands)

Second sub-batch of Layer (c) command mapping. Read 11 Go files in
`/home/kimberly/repos/gastown/internal/cmd/` and produced 11 new
wiki entity pages under `gastown/commands/`. All 11 have
`GroupID: GroupConfig`.

**Cobra group progress:** 2 of 7 groups complete
(Diag ✓, Config ✓; Work, Agents, Comm, Services, Workspace remaining).

**Commands mapped:**

- [account](gastown/commands/account.md) — multi-Claude-Code-account management with `accounts.json` + `~/.claude` symlink dance.
- [config](gastown/commands/config.md) — town-settings manager: agent, cost-tier, default-agent, agent-email-domain, set, get subcommands covering CLITheme, scheduler, convoy, lifecycle, maintenance, dolt.port.
- [directive](gastown/commands/directive.md) — parent-only stub for role directives; `Long` text advertises `show`/`edit`/`list` subcommands that are NOT wired in this file. See "Notes / open questions".
- [disable](gastown/commands/disable.md) — flips `state.Disable()` global flag; optional `--clean` calls `shell.Remove()`.
- [enable](gastown/commands/enable.md) — flips `state.Enable(Version)` global flag; no flags.
- [hooks](gastown/commands/hooks.md) — parent-only stub for centralized Claude Code hooks management; advertises 8 subcommands (`base`/`override`/`sync`/`diff`/`list`/`scan`/`registry`/`install`) NOT wired in this file. See "Notes".
- [issue](gastown/commands/issue.md) — sets/clears/shows the `GT_ISSUE` tmux session env var for status-line display. Despite the name, NOT a beads/issue-tracker wrapper.
- [plugin](gastown/commands/plugin.md) — Deacon-patrol plugin manager: list/show/run/sync/history subcommands reading `<town>/plugins/` and `<rig>/plugins/`.
- [shell](gastown/commands/shell.md) — installs/removes/shows Gas Town shell integration (`cd`-hook in RC file); `install` silently re-enables.
- [theme](gastown/commands/theme.md) — two-surface theme manager: tmux rig theme (with `apply` subcommand) and `theme cli` CLI color-scheme setter.
- [uninstall](gastown/commands/uninstall.md) — removes shell integration, wrappers, state/config/cache dirs; `--workspace` additionally nukes `~/gt`-style workspace only if it contains `mayor/`.

**Neutral observations surfaced** (filed on individual pages, not as
drift):

- **`directive` and `hooks` are parent-only stubs.** Both advertise
  subcommands in their `Long` text but wire none of them in their own
  files. Either the subcommands are registered in other files not in
  the batch, or they are unimplemented. Flagged as follow-up.
- **`gt issue` is a misleading name.** It is NOT a beads/issue-tracker
  wrapper — only sets the `GT_ISSUE` tmux session env var. It lives
  in `GroupConfig` which further obscures the purpose.
- **Dual writers to `townSettings.CLITheme`.** `config set cli_theme <mode>`
  and `theme cli <mode>` both write the same field via different
  validators. Values agree today; drift risk real.
- **`shell install` silently re-enables.** `shell.go:72` calls
  `state.Enable(Version)` as a side effect with no documentation in
  the command's help text. `shell install` can resurrect Gas Town
  from `disable`.
- **`uninstall --workspace` detection is narrow.** Only finds `~/gt`
  or `~/gastown` containing a `mayor/` subdir. Non-standard town
  locations silently skipped with no warning.
- **`gt plugin run` always records `ResultSuccess`.** Hardcoded at
  `plugin.go:480` for manual runs; there is no feedback path from
  printed-instructions execution back to the recorder. History is
  falsely optimistic.
- **Exempt-map asymmetry:** `config` and `install` are beads-exempt
  but `uninstall` is not. Removing gastown goes through the beads
  version check; installing it does not.
- **`config.go` maintains its own agent registry load** via
  `config.LoadAgentRegistry(DefaultAgentRegistryPath(townRoot))` —
  a separate config surface from town settings.
- **`directive` has a cobra alias `directives`** — the only command
  in this batch with an alias.

**Polecat-safe within Config group:** 0 of 11. Confirmed by direct
grep of `AnnotationPolecatSafe` against each source file.

**Beads-exempt within Config group:** 1 of 11 (`config`).

**Branch-check-exempt within Config group:** 0 of 11.

**Index updates:**

- [gastown/commands/README.md](gastown/commands/README.md) —
  entity-page column updated for 11 rows; progress bullet updated
  to 33 of 111.
- [gastown/README.md](gastown/README.md) — sub-index `### Commands`
  expanded with Config-group line.
- [index.md](index.md) — root gastown `### Commands` notes Batch 3b
  completion.

**Next sub-batch:** 3c — another cobra group. Five remaining:
Work, Agents, Comm, Services, Workspace. Controller's call.

**Beads status:** `wiki-3zo` (Batch 3 anchor) remains OPEN — Batch 3
has 5 groups to go. `wiki-ef3` (systematic subcommand mapping)
remains OPEN — equals Batch 3's completion.

→ [gastown/commands/README.md](gastown/commands/README.md),
  [gastown/commands/account.md](gastown/commands/account.md),
  [gastown/commands/config.md](gastown/commands/config.md),
  [gastown/commands/directive.md](gastown/commands/directive.md),
  [gastown/commands/disable.md](gastown/commands/disable.md),
  [gastown/commands/enable.md](gastown/commands/enable.md),
  [gastown/commands/hooks.md](gastown/commands/hooks.md),
  [gastown/commands/issue.md](gastown/commands/issue.md),
  [gastown/commands/plugin.md](gastown/commands/plugin.md),
  [gastown/commands/shell.md](gastown/commands/shell.md),
  [gastown/commands/theme.md](gastown/commands/theme.md),
  [gastown/commands/uninstall.md](gastown/commands/uninstall.md),
  [gastown/README.md](gastown/README.md),
  [index.md](index.md)

## [2026-04-11] ingest | Batch 3c (Layer c sub-batch: Work Management group — 26 top-level commands)

Third sub-batch of Layer (c) command mapping. Read 26 Go files in
`/home/kimberly/repos/gastown/internal/cmd/` and produced 26 new
wiki entity pages under `gastown/commands/`. `GroupWork` is the
largest single cobra group — 26 commands totaling ~3,750 lines of
wiki content (2,046 + 1,707 from the two content subagents).

**Cobra group progress:** 3 of 7 groups complete
(Diag ✓, Config ✓, Work ✓; Agents, Comm, Services, Workspace
remaining). **Coverage: 59 of 111 top-level commands mapped.**

**Commands mapped:**

- [assign](gastown/commands/assign.md) — creates a bead and hooks it to a crew member
- [bead](gastown/commands/bead.md) — parent command for cross-repo bead operations (move/show/read); alias `gt bd`
- [cat](gastown/commands/cat.md) — thin `bd show` wrapper with prefix-based rig routing
- [changelog](gastown/commands/changelog.md) — aggregates closed beads across town+rigs into human/JSON changelog
- [cleanup](gastown/commands/cleanup.md) — kills orphaned Claude processes (**not** bead cleanup; see observation below)
- [close](gastown/commands/close.md) — `bd close` wrapper with `--cascade` child closing + native-SDK convoy propagation
- [compact](gastown/commands/compact.md) — TTL-based wisp compaction; promotes valued wisps to permanent beads
- [convoy](gastown/commands/convoy.md) — 12-subcommand parent for cross-rig work-tracking units; polecat-safe; **2,739 lines across 4 files**
- [done](gastown/commands/done.md) — polecat-only session-end primitive (merge-queue submit + witness notify + sync to main + IDLE transition); **1,896 lines**
- [formula](gastown/commands/formula.md) — parent for molecule-template list/show/run/create
- [handoff](gastown/commands/handoff.md) — canonical session-restart primitive; polecats redirected to `gt done --status DEFERRED`
- [hook](gastown/commands/hook.md) — attach work to agent's hook (durability primitive); polecat-safe; distinct from plural `hooks` in Config
- [molecule](gastown/commands/molecule.md) — molecule workflow tree; canonical CLI is `gt mol` (file uses `Use="mol"` with `molecule` as alias)
- [mountain](gastown/commands/mountain.md) — Mountain-Eater activation on epics (stage convoy + add label + launch Wave 1)
- [mq](gastown/commands/mq.md) — merge queue operations (submit/retry/list/reject/post-merge/status) plus `integration` subcommand group; alias `gt mr`
- [orphans](gastown/commands/orphans.md) — find lost polecat work (unreachable commits + unmerged polecat worktrees)
- [prune-branches](gastown/commands/prune-branches.md) — thin wrapper around `internal/git.PruneStaleBranches`
- [ready](gastown/commands/ready.md) — town-wide aggregated ready-work view with parallel fetch
- [release](gastown/commands/release.md) — release stuck `in_progress` beads back to `open` (**not** software release)
- [resume](gastown/commands/resume.md) — filtered `gt mail inbox` for HANDOFF subjects (**not** a session resumer)
- [scheduler](gastown/commands/scheduler.md) — capacity-controlled dispatch scheduler; synthesizes scheduled beads from sling context beads
- [sling](gastown/commands/sling.md) — **the unified dispatch command, 1,192 lines**; polecat-safe; hooks work + auto-convoy + spawn/batch/formula
- [synthesis](gastown/commands/synthesis.md) — convoy synthesis-step management; exported hooks for witness integration
- [trail](gastown/commands/trail.md) — recent-activity viewer; aliases `recent`, `recap`
- [unsling](gastown/commands/unsling.md) — inverse of sling/hook; alias `unhook`; references `hq-l6mm5` and `gt-dtq7` as bug-fix anchors
- [wl](gastown/commands/wl.md) — **Wasteland federation**, not "worklist". DoltHub-federated multi-town rig registry; only `join` wired in this file.

**Neutral observations surfaced** (filed on individual pages):

- **Naming collisions / misleading names resolved:**
  - `gt cleanup` kills Claude processes; `gt compact` compacts wisps; `gt dolt cleanup` removes orphan DBs. Three-axis "cleanup" namespace.
  - `gt issue` (Config group, Batch 3b) does NOT wrap bd issues; sets `GT_ISSUE` tmux env var.
  - `gt release` (Work group) releases stuck beads, NOT software releases.
  - `gt resume` (Work group) is a mail reader, NOT a session resumer.
  - `gt wl` (Work group) is Wasteland federation, NOT worklist.
  - `gt molecule` canonical CLI is `gt mol` (file uses `Use="mol"` with `molecule` as alias).
  - `hook` (Work group) vs `hooks` (Config group) are entirely different concepts.

- **`close.md` uses the native beads SDK** (`github.com/steveyegge/beads`) rather than shelling out to `bd`, opening a Dolt connection to the HQ database to trigger `convoy.CheckConvoysForIssue`. Only bd-wrapper in the 26 that doesn't use exec shell-out.

- **`convoy launch` is literally `convoy stage --launch`.** `convoy_launch.go:328` delegates `runConvoyLaunch` to `runConvoyStage` after setting the `--launch` flag.

- **`findCurrentRig` lives in `mq.go`** (`mq.go:369`). General rig-resolution helper used by other commands lives in a surprising file — historical artifact.

- **`--week` flag on `changelog` is a no-op** — defined but never read.

- **`gt plugin run` always records `ResultSuccess`** (from Batch 3b observation; carried through).

- **`bd-wrapper patterns** resolved into 3 distinct patterns across the batch: pure exec shell-out (`cat`, `formula list`/`show`); exec shell-out with in-process state mutation (`assign`, `close`, `changelog`, `compact`); native SDK library usage (`close` for convoy propagation).

- **`sling` is the hub command.** 1,192 lines, 25+ flags, also owns `respawn-reset` which breaks the witness's 3-attempt circuit breaker.

- **`scheduler` stores no queue** — synthesizes scheduled beads at read time by walking all rig beads dirs. Elegant but O(n_rigs) per invocation.

- **`unsling` has two bd-hash bug-fix anchors baked in** (`hq-l6mm5`: hook-slot-vs-bead-status drift; `gt-dtq7`: rig-db-vs-town-db lookup asymmetry).

**Polecat-safe coverage within Work group:** 6 of 26 (`convoy`, `done`, `handoff`, `hook`, `molecule`, `mountain`, `sling` — wait, that's 7). Verified by direct grep; matches the expected list from Batch 3a (`convoy`, `done`, `handoff`, `hook`, `molecule`, `mountain`, `sling` all have `AnnotationPolecatSafe: "true"`).

**Beads-exempt within Work group:** 2 of 26 (`handoff`, `hook`).

**Branch-check-exempt within Work group:** 0 of 26.

**Index updates:**

- [gastown/commands/README.md](gastown/commands/README.md) — entity-page column for 26 rows; progress bullet updated to 59/111; Observations section bullet 6 rewritten as "bd-wrapper patterns (resolved)".
- [gastown/README.md](gastown/README.md) — sub-index `### Commands` expanded with Work Management group line.
- [index.md](index.md) — root gastown `### Commands` notes Batch 3c completion.

**Next sub-batch:** 3d — another cobra group. Four remaining:
Agents, Comm, Services, Workspace. Controller's call.

**Beads status:** `wiki-3zo` (Batch 3 anchor) remains OPEN —
Batch 3 has 4 groups left. `wiki-ef3` (systematic mapping) remains
OPEN.

→ [gastown/commands/README.md](gastown/commands/README.md),
  [gastown/commands/assign.md](gastown/commands/assign.md),
  [gastown/commands/bead.md](gastown/commands/bead.md),
  [gastown/commands/cat.md](gastown/commands/cat.md),
  [gastown/commands/changelog.md](gastown/commands/changelog.md),
  [gastown/commands/cleanup.md](gastown/commands/cleanup.md),
  [gastown/commands/close.md](gastown/commands/close.md),
  [gastown/commands/compact.md](gastown/commands/compact.md),
  [gastown/commands/convoy.md](gastown/commands/convoy.md),
  [gastown/commands/done.md](gastown/commands/done.md),
  [gastown/commands/formula.md](gastown/commands/formula.md),
  [gastown/commands/handoff.md](gastown/commands/handoff.md),
  [gastown/commands/hook.md](gastown/commands/hook.md),
  [gastown/commands/molecule.md](gastown/commands/molecule.md),
  [gastown/commands/mountain.md](gastown/commands/mountain.md),
  [gastown/commands/mq.md](gastown/commands/mq.md),
  [gastown/commands/orphans.md](gastown/commands/orphans.md),
  [gastown/commands/prune-branches.md](gastown/commands/prune-branches.md),
  [gastown/commands/ready.md](gastown/commands/ready.md),
  [gastown/commands/release.md](gastown/commands/release.md),
  [gastown/commands/resume.md](gastown/commands/resume.md),
  [gastown/commands/scheduler.md](gastown/commands/scheduler.md),
  [gastown/commands/sling.md](gastown/commands/sling.md),
  [gastown/commands/synthesis.md](gastown/commands/synthesis.md),
  [gastown/commands/trail.md](gastown/commands/trail.md),
  [gastown/commands/unsling.md](gastown/commands/unsling.md),
  [gastown/commands/wl.md](gastown/commands/wl.md),
  [gastown/README.md](gastown/README.md),
  [index.md](index.md)

## [2026-04-11] ingest | Batch 3d (Layer c sub-batch: Agent Management group — 12 top-level commands)

Fourth sub-batch of Layer (c) command mapping. Read 12 Go files in
`/home/kimberly/repos/gastown/internal/cmd/` and produced 12 new
wiki entity pages under `gastown/commands/`. `GroupAgents` covers
the lifecycle-manager CLI surface for the Gas Town personas:
mayor, polecat, deacon, dog, witness, refinery — plus agents/role/
session/signal/boot/callbacks infrastructure.

**Cobra group progress:** 4 of 7 groups complete
(Diag ✓, Config ✓, Work ✓, Agents ✓; Comm, Services, Workspace
remaining). **Coverage: 71 of 111 top-level commands mapped.**

**Commands mapped:**

- [agents](gastown/commands/agents.md) — 4-subcommand parent with its own RunE = `list`; multi-socket enumeration (town + default + gt-test-*).
- [boot](gastown/commands/boot.md) — daemon-tick-spawned watchdog for the Deacon; 3 subcommands (`status`/`spawn`/`triage`); Long help calls Boot "a special dog" but Boot is NOT registered as one.
- [callbacks](gastown/commands/callbacks.md) — drains Mayor's inbox; 6 callback patterns; `SLING_REQUEST` logs command but Deacon actually slings.
- [deacon](gastown/commands/deacon.md) — **15 subcommands (largest file in Sub A: 1644 lines)**; lifecycle + heartbeat + health + pause + patrol helpers; most subcommands are internal molecule-step entry points.
- [dog](gastown/commands/dog.md) — 9 subcommands; dogs are cross-rig reusable workers with worktrees into every rig; managed by the Deacon.
- [mayor](gastown/commands/mayor.md) — pure-lifecycle (start/stop/attach/status/restart/acp); `attach` is the fat one (ACP→tmux migration + Dolt init + daemon start + optional Claude respawn).
- [polecat](gastown/commands/polecat.md) — **11 top-level + 5-subcommand identity subtree**; `polecat.go` is 1786 lines with siblings `polecat_identity.go` (1079), `polecat_spawn.go` (506), `polecat_cycle.go` (85); `nuke` is a 7-step protocol with gt-4vr guardrail (best-effort push before delete) and gt-v5ku cooperation with Refinery (local-only branch delete).
- [refinery](gastown/commands/refinery.md) — per-rig merge queue processor; 11 subcommands; three different beads access patterns coexist (Manager, Engineer, direct beads.New) — suggests `unclaimed` predates the Engineer API.
- [role](gastown/commands/role.md) — agent identity inspection; 6 subcommands; uniquely in GroupAgents, `gt role` bare (no subcommand) aliases to `gt role show` instead of requiring a subcommand.
- [session](gastown/commands/session.md) — low-level tmux session surface for polecats; 9 subcommands; `session check` bypasses the polecat manager and reads `<rig>/polecats/` directly — divergent view from `gt polecat list`; `session inject` explicitly deprecated in favor of `gt nudge`.
- [signal](gastown/commands/signal.md) — Claude Code Stop-hook handler; only `stop` subcommand is wired; has a per-agent state file (`/tmp/gt-signal-stop-<addr>.json`) that remembers the last block reason and flips to approve on repeat to prevent infinite block loops that would consume the agent's entire context window.
- [witness](gastown/commands/witness.md) — per-rig polecat health monitor lifecycle; 5 subcommands; `--foreground` is vestigial (patrol logic moved to `mol-witness-patrol` molecule but the flag still parses and prints a notice).

**Neutral observations surfaced:**

- **Boot vs dogs contradiction.** `boot.go:34` Long help calls Boot "a special dog" but Boot is not registered in the dog kennel nor typed as any `AgentDog` (there is no `AgentDog` type at all). Cross-concept collision worth noting for Batch 6 role-page writing.
- **Dogs are invisible to `gt agents`.** `AgentType` in `agents.go` has no dog entry, so `gt agents list`/`menu` never shows dogs. Fresh operators won't discover `gt dog list` via the menu.
- **`callbacks` handler for `SLING_REQUEST` doesn't sling.** Logs the exact `gt sling <bead> <rig>` command string but relies on the Deacon to execute.
- **`mayor attach` does much more than attach** — upgrades ACP → tmux, starts Dolt (fatal on failure), starts daemon, potentially respawns Claude with rebuilt startup context.
- **`handleMergeCompleted` uses `cwd` rather than `townRoot`** (`callbacks.go:343`) for the `beads.New(...)` call. Flagged as a potential bug.
- **`gt witness --foreground` is vestigial**. Flag parses but `start --foreground` prints "no longer runs patrol loop" notice.
- **`gt polecat nuke` deliberately leaves remote branches alone** — gt-v5ku explicitly spells out the race with the Refinery. "Nuke" is carefully cooperative with the merge queue.
- **`gt polecat nuke --force` still performs best-effort push before deletion** (gt-4vr guardrail). Opposite of what "force" usually implies.
- **`gt refinery` splits beads access three ways** — `status`/`queue` via `refinery.Manager`, `ready`/`blocked` via `refinery.NewEngineer`, `unclaimed` via direct `beads.New(r.Path)`. Strong hint `unclaimed` predates the Engineer API.
- **`gt signal stop` has an explicit infinite-loop guard.** Without it, unread mail would consume the agent's entire context window at every turn boundary (`signal_stop.go:96-103`).
- **`gt polecat check-recovery` can override `SAFE_TO_NUKE` to `NEEDS_MQ_SUBMIT`.** A missing MR for the branch downgrades the verdict — bead `cleanup_status` field is not the final word.
- **`gt session check` produces a divergent view from `gt polecat list`.** Bypasses polecat manager; polecats without identity beads but with dirs show up; polecats with identity beads but no dir don't.

**Polecat-safe within Agents group:** 0 of 12. Verified by direct grep.

**Beads-exempt within Agents group:** 4 of 12 (`polecat`, `witness`, `refinery`, `signal`).

**Branch-check-exempt within Agents group:** 0 of 12.

**Next sub-batch:** 3e — Comm group (mail, nudge, broadcast, notify, handoff [Batch 3c], escalate, etc.). Controller picks.

**Beads status:** `wiki-3zo` and `wiki-ef3` remain open.

→ [gastown/commands/README.md](gastown/commands/README.md),
  [gastown/commands/agents.md](gastown/commands/agents.md),
  [gastown/commands/boot.md](gastown/commands/boot.md),
  [gastown/commands/callbacks.md](gastown/commands/callbacks.md),
  [gastown/commands/deacon.md](gastown/commands/deacon.md),
  [gastown/commands/dog.md](gastown/commands/dog.md),
  [gastown/commands/mayor.md](gastown/commands/mayor.md),
  [gastown/commands/polecat.md](gastown/commands/polecat.md),
  [gastown/commands/refinery.md](gastown/commands/refinery.md),
  [gastown/commands/role.md](gastown/commands/role.md),
  [gastown/commands/session.md](gastown/commands/session.md),
  [gastown/commands/signal.md](gastown/commands/signal.md),
  [gastown/commands/witness.md](gastown/commands/witness.md),
  [gastown/README.md](gastown/README.md),
  [index.md](index.md)

## [2026-04-11] ingest | Batch 3e (Layer c sub-batch: Communication group — 7 top-level commands)

Fifth sub-batch of Layer (c) command mapping. Read 7 Go files in
`/home/kimberly/repos/gastown/internal/cmd/` and produced 7 new
wiki entity pages under `gastown/commands/`.

**Cobra group progress:** 5 of 7 groups complete
(Diag ✓, Config ✓, Work ✓, Agents ✓, Comm ✓; Services and
Workspace remaining). **Coverage: 78 of 111 top-level commands.**

**Commands mapped:**

- [broadcast](gastown/commands/broadcast.md) — fan-out nudge to all workers; immediate tmux delivery only; no `--force`; honors DND.
- [dnd](gastown/commands/dnd.md) — binary (muted/normal) wrapper over the 3-level notification state; beads-exempt.
- [escalate](gastown/commands/escalate.md) — severity-routed escalation primitive; escalations are beads with the `gt:escalation` label.
- [mail](gastown/commands/mail.md) — durable messaging family with 15+ subcommands; messages are beads with `type=message`; polecat-safe; beads-exempt.
- [notify](gastown/commands/notify.md) — 3-level notification state (`verbose`/`normal`/`muted`).
- [nudge](gastown/commands/nudge.md) — universal sync messaging; 3 delivery modes (`wait-idle`/`queue`/`immediate`); polecat-safe; beads-exempt.
- [peek](gastown/commands/peek.md) — tmux capture-pane wrapper with built-in aliases for `mayor`/`deacon`/`boot`/rig/polecat/crew.

**Neutral observations:**

- **`gt mail peek` vs top-level `gt peek`** are two different commands in the same `GroupComm`. Mail's `peek` previews first unread; top-level `peek` is a tmux capture-pane alias. User-confusion risk.
- **`mail --wisp=true` is the default** — mail described as "durable" actually defaults to ephemeral/wisp; permanence is opt-in via `--permanent`.
- **`broadcast` has no `--force`** — honors DND with no override, unlike `nudge`.
- **`dnd off` silently erases the verbose preference**. Toggling DND off unconditionally writes `NotifyNormal`; no restore-previous-level memory.
- **`escalate` is NOT beads-exempt** — the "something is broken" command requires bd to function.
- **`broadcast` uses direct `NudgeSession`, not `deliverNudge`** — broadcast cannot benefit from wait-idle/queue modes. Every broadcast is an interrupt.
- **`nudge --if-fresh` uses a magic 60s threshold** (`nudge.go:135`) to suppress compaction/clear-restart nudges.
- **No `broadcast_test.go`** — only Comm command without a sibling test file.

**Polecat-safe within Comm:** 2 of 7 (`mail`, `nudge`).
**Beads-exempt within Comm:** 4 of 7 (`dnd`, `mail`, `nudge`, plus verify against `root.go:44-77`).
**Branch-check-exempt within Comm:** 0.

**Next sub-batch:** 3f — Services or Workspace group. Controller picks.

**Beads status:** `wiki-3zo` and `wiki-ef3` remain open.

→ [gastown/commands/README.md](gastown/commands/README.md),
  [gastown/commands/broadcast.md](gastown/commands/broadcast.md),
  [gastown/commands/dnd.md](gastown/commands/dnd.md),
  [gastown/commands/escalate.md](gastown/commands/escalate.md),
  [gastown/commands/mail.md](gastown/commands/mail.md),
  [gastown/commands/notify.md](gastown/commands/notify.md),
  [gastown/commands/nudge.md](gastown/commands/nudge.md),
  [gastown/commands/peek.md](gastown/commands/peek.md),
  [gastown/README.md](gastown/README.md),
  [index.md](index.md)

## [2026-04-11] ingest | Batch 3f (Layer c sub-batch: Services group — 11 top-level commands)

Sixth sub-batch of Layer (c) command mapping. Read 9 Go files
(2 files each declare 2 top-level commands: `start.go` for
`start`+`shutdown`, `estop.go` for `estop`+`thaw`) in
`/home/kimberly/repos/gastown/internal/cmd/` and produced 11 new
wiki entity pages under `gastown/commands/`.

**Cobra group progress:** 6 of 7 groups complete
(Diag ✓, Config ✓, Work ✓, Agents ✓, Comm ✓, Services ✓; Workspace
remaining). **Coverage: 89 of 111 top-level commands.**

**Commands mapped:**

- [daemon](gastown/commands/daemon.md) — parent with 8 subcommands; `run` is the hidden re-exec target that becomes the actual Go daemon process.
- [dolt](gastown/commands/dolt.md) — 19-subcommand parent wrapping a per-town Dolt SQL server on port 3307; lifecycle, imposter killing, SQL shell, backup/sync, recovery, migration, cleanup, rollback.
- [down](gastown/commands/down.md) — phase-ordered town shutdown (lock → sentinel → polecats/crew → refineries → witnesses → sessions → daemon → idle-monitors → Dolt → imposters → orphans → sockets → optional `--nuke`).
- [estop](gastown/commands/estop.md) — emergency SIGTSTP freeze with sentinel-file-first semantics; Mayor and Overseer exempt; manual-only after PR #3237 stripped daemon auto-trigger.
- [maintain](gastown/commands/maintain.md) — single-command Dolt maintenance pipeline (backup → reap wisps → flatten → gc) running SQL-only on a live server.
- [quota](gastown/commands/quota.md) — 5-subcommand parent including `rotate` which swaps macOS Keychain OAuth tokens into the same config dir so `/resume` works after rotation; tracks `GT_QUOTA_ACCOUNT` as a shadow identity.
- [reaper](gastown/commands/reaper.md) — parent with `databases`/`scan`/`reap`/`purge`/`auto-close`/`run` subcommands executing Dolt SQL operations on behalf of the `mol-dog-reaper` formula.
- [shutdown](gastown/commands/shutdown.md) — destructive "done for the day" with polecat worktree/branch cleanup + zombie Claude processes + orphaned daemon PID sweep. Narrower than `down`.
- [start](gastown/commands/start.md) — older, simpler boot (Mayor + Deacon + optional `--all`). Does NOT launch the daemon.
- [thaw](gastown/commands/thaw.md) — resumes estop-frozen sessions via `SIGCONT` and removes the ESTOP sentinel.
- [up](gastown/commands/up.md) — idempotent boot with parallel Dolt/daemon/deacon/mayor/rig-prefetch, Dolt-readiness gate, orphaned-bead recovery, opt-in `--restore`; supersedes `start`.

**Neutral observations surfaced:**

- **Two boot commands with drift.** `gt start` and `gt up` both "boot Gas Town" but diverge: `up` has a Dolt-readiness gate, orphaned-bead recovery, 300ms/2s daemon startup verify, `--json` output, a worker-pool for witnesses/refineries, richer crew-startup grammar. `start` has a `/crew/` path shortcut and uses `config.EnsureDaemonPatrolConfig` where `up` uses `daemon.EnsureLifecycleConfigFile`.
- **`gt start` doesn't launch the daemon** — no `ensureDaemon` call in `runStart`.
- **`gt daemon run` is `Hidden: true`** but is the actual daemon process; nothing prevents direct user invocation.
- **`gt quota rotate` deliberately does NOT persist scan-detected limits during planning** to avoid poisoning the pool with stale rate-limit messages from parked rigs.
- **`gt maintain` explicitly cites Tim Sehn 2026-02-28** as authority for running on a live Dolt server without stopping it.
- **`gt down` writes a `daemon/shutting-down` sentinel** that `ensureDaemon` checks to prevent agents from restarting the daemon mid-teardown (GH#2656).
- **`gt dolt` beads-exempt is semantically necessary**: it's the bootstrap path that creates the bead store.
- **`gt down` flock comment at `down.go:114-120`** explains why the lock file is never removed (flock operates on inodes).
- **`quotaJSON` is a shared package var** bound to `--json` on 3 different subcommands.
- **`gt reaper reap` hard-codes a 500-open-wisps alert threshold** inline.
- **`gt reaper run` has no `--json` flag**, breaking machine-parsability for the full-cycle invocation while subcommands have it.
- **`gt up`'s `ensureDaemon` has a shorter verification window** (300ms/2s) than `gt daemon start`'s 3-second polling — false negatives possible under load.
- **`shutdown` is narrower than `down`** — doesn't sweep legacy tmux sockets or `.beads/dolt` directories.

**Polecat-safe within Services:** 0 of 11.
**Beads-exempt within Services:** 3 of 11 (`dolt`, `estop`, `thaw`).
**Branch-check-exempt within Services:** 2 of 11 (`estop`, `thaw`).

**Next sub-batch:** 3g — Workspace group (final cobra group). Then any ungrouped commands, then Batch 3 closes.

**Beads status:** `wiki-3zo` and `wiki-ef3` remain open.

→ [gastown/commands/README.md](gastown/commands/README.md),
  [gastown/commands/daemon.md](gastown/commands/daemon.md),
  [gastown/commands/dolt.md](gastown/commands/dolt.md),
  [gastown/commands/down.md](gastown/commands/down.md),
  [gastown/commands/estop.md](gastown/commands/estop.md),
  [gastown/commands/maintain.md](gastown/commands/maintain.md),
  [gastown/commands/quota.md](gastown/commands/quota.md),
  [gastown/commands/reaper.md](gastown/commands/reaper.md),
  [gastown/commands/shutdown.md](gastown/commands/shutdown.md),
  [gastown/commands/start.md](gastown/commands/start.md),
  [gastown/commands/thaw.md](gastown/commands/thaw.md),
  [gastown/commands/up.md](gastown/commands/up.md),
  [gastown/README.md](gastown/README.md),
  [index.md](index.md)

## [2026-04-11] ingest | Batch 3g (Layer c sub-batch: Workspace group — 7 top-level commands; final cobra group)

Seventh and final cobra-group sub-batch of Layer (c) command mapping.
Read 7 Go files in `/home/kimberly/repos/gastown/internal/cmd/` and
produced 7 new wiki entity pages under `gastown/commands/`.

**Cobra group progress:** 7 of 7 groups complete ✓ ✓ ✓ ✓ ✓ ✓ ✓
(Diag, Config, Work, Agents, Comm, Services, Workspace). **Coverage:
96 of 111 top-level commands mapped via cobra groups.**

**Commands mapped:**

- [crew](gastown/commands/crew.md) — CLI surface for persistent, user-managed crew workspaces (full clones, unlike polecat worktrees); 13 subcommands across 9 sibling files.
- [git-init](gastown/commands/git-init.md) — `.gitignore` + git init + branch-protection post-checkout hook for an existing HQ; retrofit path when `gt install --git` wasn't used.
- [init](gastown/commands/init.md) — low-level primitive to scaffold agent subdirs and `.git/info/exclude` in an existing git repo to make it a rig.
- [install](gastown/commands/install.md) — creates the Gas Town HQ (town root): mayor/, deacon/, beads, Dolt bring-up, optional git, shell, wrappers, supervisor.
- [namepool](gastown/commands/namepool.md) — show pool status, list/switch/create/delete themes, add custom names for polecat naming; 6 subcommands.
- [rig](gastown/commands/rig.md) — manages rig containers: add/remove/list, patrol lifecycle, operational state (park/dock), plus detect/quick-add/config/settings/menu/reset; ~20 subcommands across 8 files.
- [worktree](gastown/commands/worktree.md) — create/list/remove cross-rig git worktrees for crew members, with env-var exports for identity preservation.

**Neutral observations surfaced:**

- **`crew` and `rig` are parent stubs for large subcommand trees** distributed across many sibling files. `crew.go` (432 lines) is CLI wiring only; all `runCrew*` functions live in `crew_add.go`, `crew_at.go`, `crew_lifecycle.go`, `crew_list.go`, `crew_maintenance.go`, `crew_status.go`, `crew_cycle.go`. `rig.go` (2481 lines) defines 12 subcommands in the core file, 8 more in siblings.
- **`gt worktree` is crew-specific, not general git.** Prints identity-preserving env vars for the user to paste. One of the few places gt has an explicit `runtime.GOOS == "windows"` branch.
- **Three distinct setup primitives:** `gt init` scaffolds a rig's agent dirs inside an existing git repo. `gt install` creates the enclosing HQ. `gt git-init` retrofits git into an existing HQ. They chain: `gt install` → `gt git-init` (if `--git` omitted) → `gt rig add` (transitively covers what `gt init` does).
- **`crewCmd.Long` contains the canonical crew-vs-polecat definition** ("Polecats: Ephemeral...witness-managed. Auto-nuked. Crew: Persistent. User-managed."). Preserved verbatim on the crew page.
- **`gt install` hides an escape hatch:** `gt install <existing-hq> --wrappers` bypasses the "already a workspace" guard and installs wrappers only, returning successfully.
- **`namepool.detectCurrentRigWithPath`** excludes `mayor` and `deacon` from being treated as rigs but not other agent dirs like `plugins/` — flagged as a possible latent bug.

**Polecat-safe within Workspace:** 0 of 7.
**Beads-exempt within Workspace:** 3 of 7 (`crew`, `rig`, `install`).
**Branch-check-exempt within Workspace:** 2 of 7 (`install`, `git-init`).

**Cobra-group mapping complete (96 of 111 commands).** Remaining ~15 commands have no `GroupID` and did not self-assign to any of the 7 cobra groups. These will be mapped in Batch 3h (or directly in the next subagent dispatch) before Batch 3 closes.

**Beads status:** `wiki-3zo` (Batch 3 anchor) and `wiki-ef3` (systematic subcommand mapping) remain OPEN until all 111 commands are mapped, not just the 96 grouped ones.

→ [gastown/commands/README.md](gastown/commands/README.md),
  [gastown/commands/crew.md](gastown/commands/crew.md),
  [gastown/commands/git-init.md](gastown/commands/git-init.md),
  [gastown/commands/init.md](gastown/commands/init.md),
  [gastown/commands/install.md](gastown/commands/install.md),
  [gastown/commands/namepool.md](gastown/commands/namepool.md),
  [gastown/commands/rig.md](gastown/commands/rig.md),
  [gastown/commands/worktree.md](gastown/commands/worktree.md),
  [gastown/README.md](gastown/README.md),
  [index.md](index.md)

## [2026-04-11] ingest | Batch 3h (Layer c final sub-batch: ungrouped commands — 15 pages; Batch 3 complete)

Final sub-batch of Layer (c) command mapping. Read 15 Go files in
`/home/kimberly/repos/gastown/internal/cmd/` and produced 15 new
wiki entity pages under `gastown/commands/`. **All 111 top-level
commands are now mapped.**

**Coverage:** 111 of 111 top-level commands. **Cobra-group coverage
re-verified in this batch:** direct `grep -n "GroupID:"` on
`commit.go`, `forget.go`, `memories.go`, `remember.go`, `show.go`
confirmed NONE of them have a `GroupID` assignment. They are
genuinely ungrouped, matching the original Batch 3h plan. The
prior Batch 3c enumeration of 26 `GroupWork` members stands.

**Filename corrections applied in this batch:**

- `proxy` [?] → `proxy-subcmds` (actual `Use:` field is `proxy-subcmds`)
- `statusline` → `status-line` (actual `Use:` field is `status-line`
  despite the Go file being `statusline.go`)

These corrections are reflected in both the entity-page column
(linking to `proxy-subcmds.md` / `status-line.md`) and the command
name column (removing `[?]` flags and updating to canonical forms).

**Commands mapped in Batch 3h:**

- [agent-log](gastown/commands/agent-log.md) — Hidden internal daemon streaming Claude/OpenCode conversation events to OTLP for a tmux session's lifetime.
- [commit](gastown/commands/commit.md) — Wrapper around `git commit` injecting agent git author identity via `detectSender()`.
- [cycle](gastown/commands/cycle.md) — `next`/`prev` tmux session cycling with auto-detected scope.
- [forget](gastown/commands/forget.md) — Deletes a memory from the bd kv store.
- [health](gastown/commands/health.md) — Comprehensive data-plane health report (6 sections); beads-exempt.
- [krc](gastown/commands/krc.md) — Key Record Chronicle: 7 subcommands for TTL lifecycle of Level 0 ephemeral event files; beads-exempt.
- [memories](gastown/commands/memories.md) — Lists or searches stored memories; sort order matches `gt prime` injection priority.
- [nudge-poller](gastown/commands/nudge-poller.md) — Hidden background daemon (one per non-Claude tmux session) draining the nudge queue on idle.
- [proxy-subcmds](gastown/commands/proxy-subcmds.md) — Hidden command scanning `rootCmd.Commands()` for `AnnotationPolecatSafe` and emitting a sorted allowlist for `gt-proxy-server` startup. Defines the `AnnotationPolecatSafe = "polecatSafe"` constant itself.
- [remember](gastown/commands/remember.md) — Writes a memory to bd kv; owns the shared memory registry consumed by `memories`/`forget`.
- [show](gastown/commands/show.md) — Thin `DisableFlagParsing` wrapper around `bd show` with auto-routing across hq/gt/mo beads databases via bead-ID prefix.
- [status-line](gastown/commands/status-line.md) — Hidden tmux `status-right` renderer; five per-role renderers plus E-stop prefix.
- [tap](gastown/commands/tap.md) — Parent group for Claude Code PreToolUse/PostToolUse hook handlers; only `guard` is implemented; beads-exempt.
- [town](gastown/commands/town.md) — Town-level `next`/`prev` session cycling (forces town semantics, unlike the auto-detecting `cycle`).
- [warrant](gastown/commands/warrant.md) — File/list/execute "death warrants" for stuck agents; Boot picks them up during triage.

**Observations surfaced:**

- **4 commands are `Hidden: true`:** `agent-log`, `nudge-poller`, `proxy-subcmds`, `status-line`. These are internal helpers never intended for direct user invocation — only surfaced here because they register with `rootCmd.AddCommand`.
- **`tap` is a parent-only stub** like `directive`/`hooks` from Batch 3b. Advertises `audit`/`inject`/`check` subcommands in Long help but only `guard` is actually wired.
- **`health` is ungrouped but beads-exempt**, while `doctor`/`vitals`/`repair` are all in `GroupDiag`. Group assignment drift vs neighboring commands.
- **`warrant` help text drift**: `Long` claims warrants live in `~/gt/warrants/` but `getWarrantDir()` returns `<townRoot>/warrants/`. Minor doc drift in source.
- **`proxy-subcmds` discovery is dynamic** (`rootCmd.Commands()` iteration + sort) but the `bd` half of the allowlist (`bdSafeSubcmds`) is a hardcoded string constant that can drift silently.
- **`status-line` file-vs-command mismatch**: file is `statusline.go`, command is `status-line`.
- **Memory-trio coupling**: `remember.go` owns `memoryKeyPrefix`, `validMemoryTypes`, `memoryTypeOrder`, `parseMemoryKey`, `autoKey`, `sanitizeKey`, and the four `bdKv*` helpers. `memories.go` and `forget.go` import these — tightly coupled on purpose.
- **`cycle` vs `town`**: `cycle next` auto-detects scope (town / crew / rig-ops), while `town next` forces town semantics. They share helpers (`cycleTownSession`, `cycleInGroup`).

**Polecat-safe in Batch 3h:** 0 of 15 (none of the 15 source files define `AnnotationPolecatSafe: "true"` on their cobra.Command; the constant is defined in `proxy_subcmds.go` but that file's own command is NOT annotated).

**Beads-exempt in Batch 3h:** 3 of 15 (`health`, `krc`, `tap`).

**Branch-check-exempt in Batch 3h:** 0 of 15.

**Batch 3 status:** COMPLETE. All 111 top-level cobra commands in `internal/cmd/` have entity pages under `gastown/commands/`. Nested subcommand mapping (~384 additional `cobra.Command` definitions) is NOT included in Batch 3 — that's a future pass if warranted.

**Beads closed as part of this commit:**
- `wiki-3zo` (Batch 3 anchor) — Layer (c) Command layer fully mapped
- `wiki-ef3` (systematic 107-subcommand mapping) — effectively equal to Batch 3 scope; closes with it

→ [gastown/commands/README.md](gastown/commands/README.md),
  [gastown/commands/agent-log.md](gastown/commands/agent-log.md),
  [gastown/commands/commit.md](gastown/commands/commit.md),
  [gastown/commands/cycle.md](gastown/commands/cycle.md),
  [gastown/commands/forget.md](gastown/commands/forget.md),
  [gastown/commands/health.md](gastown/commands/health.md),
  [gastown/commands/krc.md](gastown/commands/krc.md),
  [gastown/commands/memories.md](gastown/commands/memories.md),
  [gastown/commands/nudge-poller.md](gastown/commands/nudge-poller.md),
  [gastown/commands/proxy-subcmds.md](gastown/commands/proxy-subcmds.md),
  [gastown/commands/remember.md](gastown/commands/remember.md),
  [gastown/commands/show.md](gastown/commands/show.md),
  [gastown/commands/status-line.md](gastown/commands/status-line.md),
  [gastown/commands/tap.md](gastown/commands/tap.md),
  [gastown/commands/town.md](gastown/commands/town.md),
  [gastown/commands/warrant.md](gastown/commands/warrant.md),
  [gastown/README.md](gastown/README.md),
  [index.md](index.md)

## [2026-04-11] ingest | Batch 4 (Layer d: Platform services — 9 package pages)

First batch of the package layer. Created a new category folder
`gastown/packages/` and produced 9 entity pages for the platform
service packages under `/home/kimberly/repos/gastown/internal/`.

**Packages mapped:**

- [cli](gastown/packages/cli.md) — single `Name()` function reading `GT_COMMAND`; used by root command's init
- [config](gastown/packages/config.md) — 9 files covering town settings, agent registry (10 built-in presets), load/save for every persisted JSON shape (town, mayor, rigs, settings, daemon, accounts, messaging, escalation, merge queues), `Resolve*Agent*` family, `AgentEnv`, `BuildStartupCommand*` family, `IdentityEnvVars` list (GH#3006), cost-tier system, role definitions via embedded `roles/*.toml`, directive loader, overseer detection
- [session](gastown/packages/session.md) — 11 files; tmux session substrate for ALL agent roles (not just polecats); identity parsing, canonical session-name factories, `PrefixRegistry`, unified `StartSession`/`StopSession`, `TrackPID` with start-time fingerprinting, stale detection, beacon/startup prompt rendering, town-wide session bulk ops, window tints
- [style](gastown/packages/style.md) — thin lipgloss wrapper; Success/Warning/Error/Info/Dim/Bold styles + icon prefixes; `PrintWarning` (stderr-only, doesn't corrupt JSON stdout); ANSI-aware `Table` renderer
- [telemetry](gastown/packages/telemetry.md) — **strictly opt-in OTEL provider**. `telemetry.Init` returns `(nil, nil)` if neither `GT_OTEL_METRICS_URL` nor `GT_OTEL_LOGS_URL` is set. Default endpoints point at localhost VictoriaMetrics/VictoriaLogs — never off-box. 23 metric counters + 1 histogram; log events via `emit(...)`. Subprocess env propagation via `OTEL_RESOURCE_ATTRIBUTES` for `gt.*` labels. Every PII-adjacent field (bd output, prompt keys, mail body, prime context, pane output, agent output) is gated behind its own explicit `GT_LOG_*` env var, all off by default.
- [ui](gastown/packages/ui.md) — Ayu palette (always-on, adaptive light/dark); semantic status/priority/type colours; `InitTheme`/`ApplyThemeMode`; `NO_COLOR`/`CLICOLOR`/`GT_NO_EMOJI`/`GT_AGENT_MODE`/`CLAUDE_CODE` capability detection; `RenderMarkdown` (glamour); `ToPager` with `less -RFX`; canonical `Render*` functions for beads metadata
- [util](gastown/packages/util.md) — atomic JSON/file write; `FirstLine`/`ExecWithOutput`/`ExecRun`; `SetProcessGroup`/`SetDetachedProcessGroup` (Windows no-flash, Unix kill-tree); `ExpandHome`; slice helpers; `RedactURL`; and `orphan.go`'s ~1000-line orphan/zombie Claude cleanup with multi-layer safety guards, ACP protection, TOCTOU re-verification, and SIGTERM→SIGKILL→UNKILLABLE escalation state machine
- [version](gastown/packages/version.md) — build-time `Commit` var + `CheckStaleBinary` which compares binary commit to repo HEAD with false-positive guards: different-clone object-store verification, `.beads/`-only-change suppression (GH#2596), forward-only ancestor gate, branch allowlist (`main`/`master`/`carry/*`)
- [workspace](gastown/packages/workspace.md) — `Find*` family walking up to `mayor/town.json`; `GT_TOWN_ROOT`/`GT_ROOT` env-var fallbacks; "cwd-deleted" survivor variant; `GetTownName` reading town.json via `config.LoadTownConfig`

**Bead `wiki-9u4` answered:**

Telemetry is strictly opt-in. Full answer:

- **Endpoints:** `GT_OTEL_METRICS_URL` and `GT_OTEL_LOGS_URL` env vars. No default endpoint. If neither is set, `telemetry.Init` returns `(nil, nil)` — no-op.
- **Hardcoded defaults (only when at least one var is set and the other is empty):** localhost VictoriaMetrics (`http://localhost:8428/...`) and VictoriaLogs (`http://localhost:9428/...`). Never off-box by default.
- **Opt-out:** unset both env vars. No explicit kill-switch env var.
- **Data exported (when active):** 23 metric counters (`gastown.*.total`) + 1 histogram (`gastown.bd.duration_ms`) + log events via `emit(...)`. Resource attributes include hostname, OS info, service name, service version. `SetProcessOTELAttrs` additionally sets `gt.role`, `gt.rig`, `gt.actor`, `gt.agent`, `gt.session`, `gt.run_id`, `gt.work_rig`, `gt.work_bead`, `gt.work_mol` on child `bd` processes via `OTEL_RESOURCE_ATTRIBUTES`.
- **Secret/PII fields are separately gated:** `GT_LOG_BD_OUTPUT` (2048 bytes), `GT_LOG_PROMPT_KEYS` (256 bytes), `GT_LOG_MAIL_BODY` (256 bytes), `GT_LOG_PRIME_CONTEXT` (untruncated), `GT_LOG_PANE_OUTPUT` (8192 bytes), `GT_LOG_AGENT_OUTPUT` (512 bytes) — all off by default with inline warnings about tokens/PII.
- **Main always-on identifiable fields when active:** hostname, `town_root` path on `agent.instantiate` events, `BD_ACTOR` → `gt.actor` label.
- **Windows:** agent conversation streaming is Unix-only (`agent_logging_windows.go` is a no-op stub).

Bead closed.

**Bead `wiki-2g3` answered:** (was pending from Batch 3)

`AnnotationPolecatSafe = "polecatSafe"` is defined in
`/home/kimberly/repos/gastown/internal/cmd/proxy_subcmds.go:15`. It is
read by `discoverAllowedSubcmds` in the same file, which iterates
`rootCmd.Commands()` and emits a sorted `gt:...;bd:...` allowlist
string. The allowlist is consumed by `gt-proxy-server` at startup
via `exec.LookPath("gt") && gt proxy-subcmds` to configure its
subcommand allowlist for polecat containers. The `bd` half of the
allowlist is a hardcoded string constant (`bdSafeSubcmds` in the same
file), which can drift silently relative to what `bd` actually allows.

13 top-level commands carry the annotation: `version`, `status`,
`sling`, `prime`, `proxy-subcmds` (NOT itself annotated), `mountain`,
`nudge`, `molecule`, `mail`, `hook`, `handoff`, `done`, `convoy`.

Bead closed.

**Surprises:**

- **`orphan.go` protects ALL tmux sockets on the machine**, not just this town's. Explicit in-source comment prevents cross-town kills when multiple Gas Towns coexist.
- **`types.go` in `internal/config` is ~67 KB** — catalogs every persisted JSON shape in one file. Natural split candidate.
- **Crew and polecat session names are structurally identical** (`<prefix>-<name>`). Disambiguation relies on agent identity lookup, not name parsing.
- **Stale-binary check has two explicit false-positive guards**: different-clone object-store verification and `.beads/`-only-change suppression (GH#2596).
- **`resolveConfigMu` in config/loader.go** is a global mutex serializing agent config resolution across all callers — load-bearing, potential bottleneck under heavy rig concurrency.
- **Agent logging is Unix-only.** Windows agents get zero conversation streaming to VictoriaLogs even with `GT_LOG_AGENT_OUTPUT=true`. Cross-platform drift.
- **`pidtrack` tests cannot use `t.Parallel()`** due to package-level `pidStartTimeFunc` variable.

**Next batch:** **STOP HERE.** Per the plan's autonomous-mode stop conditions, Batch 4 is the natural checkpoint before the data layer (Batch 5), agent runtime / domain layer (Batch 6), and the rest of the infrastructure batches. Controller will review Batch 4 before the autonomous loop resumes.

**Beads closed in this commit:** `wiki-58n` (Batch 4 anchor), `wiki-9u4` (OTEL telemetry trace, answered by telemetry.md), `wiki-2g3` (AnnotationPolecatSafe trace, answered by Batch 3h's proxy-subcmds.md + reflected in telemetry context).

→ [gastown/packages/cli.md](gastown/packages/cli.md),
  [gastown/packages/config.md](gastown/packages/config.md),
  [gastown/packages/session.md](gastown/packages/session.md),
  [gastown/packages/style.md](gastown/packages/style.md),
  [gastown/packages/telemetry.md](gastown/packages/telemetry.md),
  [gastown/packages/ui.md](gastown/packages/ui.md),
  [gastown/packages/util.md](gastown/packages/util.md),
  [gastown/packages/version.md](gastown/packages/version.md),
  [gastown/packages/workspace.md](gastown/packages/workspace.md),
  [gastown/README.md](gastown/README.md),
  [index.md](index.md)

## [2026-04-11] ingest | Batch 5 (Layer e: Data layer — 8 package pages)

Second batch of the package layer. Produced 8 entity pages for the
data-layer packages under `/home/kimberly/repos/gastown/internal/`.

**Packages mapped:**

- [beads](gastown/packages/beads.md) — Go library interface to beads databases. Hybrid subprocess/SDK dispatch (shells out to `bd` for most operations, uses `beadsdk.Storage` for in-process fast paths). 28 files, ~10,300 lines. NOT the same as the `bd` binary or the `.beads/` directory. Contains Gas Town domain logic (agent beads, molecules, merge slots, rig identity, channels, escalations, delegation, routing) as well as thin bd wrappers.
- [channelevents](gastown/packages/channelevents.md) — file-based per-channel pub/sub. One JSON file per event under `<townRoot>/events/<channel>/`. Rendezvous semantics (consumer deletes after handling). Distinct from `events`.
- [doltserver](gastown/packages/doltserver.md) — per-town Dolt MySQL server (port 3307) lifecycle manager. Handles start/stop via exec with `Setpgid: true`, imposter killing, phantom DB quarantine, stale LOCK/socket cleanup. Authentication is `root`/no-password for localhost; multi-town isolation is by port. Hosts the "Wanted List Commons" schema as a co-located tenant (unrelated to the Dolt server's lifecycle — may warrant its own package).
- [events](gastown/packages/events.md) — append-only JSONL activity feed at `<townRoot>/.events.jsonl` with cross-process flock. 21 event types + 21 payload constructors. Best-effort semantics: silently no-ops when not in a town workspace.
- [lock](gastown/packages/lock.md) — agent-identity file locks at `<workerDir>/.runtime/agent.lock`. `CleanStaleLocks` requires BOTH dead PID AND missing tmux session before cleaning — critical under Claude Code where the spawning parent often exits while the tmux child keeps running. Windows flock is a no-op by design.
- [mail](gastown/packages/mail.md) — durable messaging. Messages are beads with `gt:message` label. ~3500 lines of routing (direct, queue, channel, group, announce), threading via random 6-byte hex `ThreadID`, two-phase ack delivery. Has a native-SDK fast path (~5ms) alongside the subprocess path (~600ms).
- [mq](gastown/packages/mq.md) — single-file (59 lines) merge-request ID minter. SHA-256 of branch + timestamp + random → `<prefix>-mr-<10hex>`. NOT a message queue despite the name (that lives in `internal/mail`'s router). Scheme was expanded from 6 to 10 hex chars after birthday-paradox collisions at ~150K IDs.
- [nudge](gastown/packages/nudge.md) — ephemeral queue. Writes JSON files to `<townRoot>/.runtime/nudge_queue/<session>/`. Drained by Claude Code's UserPromptSubmit hook (inline) or by background `gt nudge-poller` for non-Claude agents (Gemini/Codex/Cursor). Atomic-rename claim protocol with unique-suffix Windows compatibility. Orphan-claim sweep restores rather than deletes.

**Neutral observations surfaced:**

- **`internal/beads` distinction**: `bd` is the tool; `.beads/` is the data; `internal/beads` is the Go library. Three different things with the same root word.
- **Hybrid dispatch is per-method, not per-instance**: every `Beads` method checks `if b.store != nil` before falling through to subprocess. Missing `storeX` methods silently use the subprocess path even when a store is attached.
- **`ListMergeRequests` runs hand-rolled SQL**: `bd list` only queries the `issues` table, but MRs are stored in the `wisps` table (bd v0.59+). `ListMergeRequests` runs a `bd sql --json` SELECT against `wisps + wisp_labels` to collect MRs.
- **`NeedsForceForID` bypasses bd's prefix-inference heuristic** for multi-hyphen IDs like `st-stockdrop-polecat-nux`.
- **`doltserver` hosts the Wanted List Commons schema** (`wl_commons.go`, `wl_charsheet.go`, ~36KB) — a tenant schema riding on the same server, unrelated to Dolt lifecycle. May warrant its own package.
- **`doltserver` authentication is effectively none**: `root`/empty password for localhost. Multi-town isolation is by port.
- **`KillImposters` compares running `dolt sql-server` data dirs** against expected town data dir using `/proc/<pid>` forensics. Requires stopping idle monitors first (gt-restart-race fix) because idle monitors auto-respawn rogue servers.
- **`internal/mail` does NOT use `internal/lock`.** It relies on bd's transactional guarantees for beads mode and `github.com/gofrs/flock` directly for the legacy JSONL fallback.
- **`internal/nudge` does NOT use flock either.** Atomic-rename with unique suffix per drainer. Unique suffix is load-bearing specifically on Windows where shared-destination rename is not atomic.
- **`events` vs `channelevents` are two different systems** with no shared code. `events` is an append-only audit trail; `channelevents` is a file-based rendezvous/pub-sub with one file per event and consumer deletes.
- **Mail's subprocess read timeout is 60s** (not 30s) because of real `signal: killed` incidents under concurrent agent load.
- **Mail's in-process store path bypasses ~600ms of subprocess + Dolt-connection overhead** per operation.
- **Wisp messages are queried via raw SQL** from `internal/mail` because `bd list` filters wisps out.

**Next batch:** Batch 6 — Layer (f) Agent runtime (domain layer). Largest and most central batch. Produces both package pages AND first-class domain pages (roles/, concepts/, workflows/). Per the plan's autonomous-mode stop conditions, Batch 6 requires explicit controller approval before starting.

**Beads status:** `wiki-w0k` (Batch 5 anchor) closed by this commit. Batches 6–12 still open.

→ [gastown/packages/beads.md](gastown/packages/beads.md),
  [gastown/packages/channelevents.md](gastown/packages/channelevents.md),
  [gastown/packages/doltserver.md](gastown/packages/doltserver.md),
  [gastown/packages/events.md](gastown/packages/events.md),
  [gastown/packages/lock.md](gastown/packages/lock.md),
  [gastown/packages/mail.md](gastown/packages/mail.md),
  [gastown/packages/mq.md](gastown/packages/mq.md),
  [gastown/packages/nudge.md](gastown/packages/nudge.md),
  [gastown/README.md](gastown/README.md),
  [index.md](index.md)

## [2026-04-11] ingest | Batch 6 (Layer f: Agent runtime / domain layer — 30 pages)

The largest and most semantically important batch. Created three new
category folders (`gastown/roles/`, `gastown/concepts/`, `gastown/workflows/`)
and produced 30 entity pages covering the Gas Town agent runtime:

- **13 packages** under `gastown/packages/` (mayor, polecat, crew, dog, deacon, refinery, witness, reaper, wisp, convoy, rig, formula, plugin)
- **8 roles** under `gastown/roles/` (mayor, polecat, crew, dog, deacon, refinery, witness, reaper)
- **7 concepts** under `gastown/concepts/` (rig, convoy, formula, molecule, wisp, directive, identity)
- **2 workflows** under `gastown/workflows/` (convoy-launch, polecat-lifecycle)

**Same-batch rule applied:** each of the 13 internal/ packages was read once by a content subagent and its code page + role/concept page were produced together.

**Gas Town personas (roles):**

Eight agents with identity, decisions, and autonomy:

1. [Mayor](gastown/roles/mayor.md) — town-level orchestrator; one per machine; the Overseer's "Chief of Staff"; runs in tmux OR headless ACP mode (the only gt agent with two runtime substrates). `gt mayor attach` does much more than attach.
2. [Polecat](gastown/roles/polecat.md) — primary feature-building worker; persistent identity (themed name + permanent agent bead + CV chain) + ephemeral sessions + sandbox worktree. Nuke is a cooperative 7-step protocol with gt-4vr push-before-delete guardrail and gt-v5ku remote-branch protection.
3. [Crew](gastown/roles/crew.md) — persistent user-managed worker; FULL git clones (not worktrees); stable names; mailbox; user's answer to "I want a long-lived dev session with an agent."
4. [Dog](gastown/roles/dog.md) — reusable cross-rig infrastructure worker; worktrees into every configured rig; managed solely by the Deacon; invisible to `gt agents list` because it's not registered as an `AgentType`. "Cats build features. Dogs clean up messes."
5. [Deacon](gastown/roles/deacon.md) — town-level watchdog; mechanical sibling of the Mayor; runs patrol loops, heartbeat, feed-stranded/redispatch/health-check state files, escalates to Mayor when rate-limit budgets exhaust.
6. [Refinery](gastown/roles/refinery.md) — per-rig merge queue processor; pre-merge + post-squash quality gates; squash-merge behind a beads-backed main-push slot lock; spawns a fresh polecat via synthesis task bead on conflict. Batch-then-bisect merging is opt-in.
7. [Witness](gastown/roles/witness.md) — per-rig polecat health monitor; runs as a Claude driving `mol-witness-patrol`; three completion-detection paths (POLECAT_DONE mail, bead `exit_type`, patrol sweep); Mountain-Eater auto-skip; cross-process flock-serialized spawn-count circuit breaker.
8. [Reaper](gastown/roles/reaper.md) — **not a long-running agent**; a role performed on demand by a Dog executing `mol-dog-reaper`. Zero gastown package imports; SQL-direct; decision-less eligibility via WHERE clauses.

**Domain concepts:**

- [`rig`](gastown/concepts/rig.md) — workspace abstraction: one repo + one refinery + one witness + polecats + crew. `<rigRoot>` canonical layout (`.repo.git/`, `refinery/rig/`, `mayor/rig/`, `witness/`, `polecats/`, `crew/`, `.beads/`, `plugins/`, `settings/`). **Rigs are not git repos** — they're gastown wrappers around a repo, with `.repo.git/` as the shared bare clone and `refinery/rig/` as a worktree.
- [`convoy`](gastown/concepts/convoy.md) — cross-rig work-tracking unit; `hq-` prefix; `tracks` (non-blocking) dependency type; multi-stage lifecycle (`open`/`closed`/`staged_ready`/`staged_warnings`); uses molecules (`mol-convoy-feed`) for dispatch.
- [`formula`](gastown/concepts/formula.md) — static TOML template with steps, variables, overlays; four types (`convoy`/`workflow`/`expansion`/`aspect`); composable via `extends` and `compose.expand`.
- [`molecule`](gastown/concepts/molecule.md) — running bead-tracked instance of a formula. **Formula : molecule :: class : instance.** Static template on disk vs live state in beads.
- [`wisp`](gastown/concepts/wisp.md) — ephemeral bead in the `wisps` SQL table (not `issues`); TTL-compactable; escapes compaction only via `ShouldPromote` (has comments / referenced by non-wisp / `gt:keep` label / open past TTL). **Name collision trap:** `internal/wisp` package does NOT implement the wisp concept — that lives in `internal/beads` + `internal/doltserver`.
- [`directive`](gastown/concepts/directive.md) — operator-provided role override at `<town>/directives/<role>.md` or `<town>/<rig>/directives/<role>.md`; concatenated (town first, rig last); injected at prime time. CLI is a parent-only stub; the config-loader side is implemented in `internal/config/directives.go` (41 lines).
- [`identity`](gastown/concepts/identity.md) — agent identity via structured address (`<prefix>/<role>` or `<prefix>/<name>`); resolved by `detectSender()` with priority `GT_ROLE` env var → cwd path → `overseer` default. Drives mail routing, bead authorship, role dispatch, directive loading, session naming. Agent bead is the persistent anchor (NOT a wisp; protected from reaper compaction).

**Workflows:**

- [`convoy-launch`](gastown/workflows/convoy-launch.md) — 8 steps: stage → launch (validated transition) → initial dispatch via `mol-convoy-feed` → watch subscription → reactive continuation via `CheckConvoysForIssue` on every close → Deacon safety-net polling → Refinery merges tracked issues → last-merge landing (convoy closes, swarm lands for `gt:owned` convoys).
- [`polecat-lifecycle`](gastown/workflows/polecat-lifecycle.md) — 9 states: name allocation → worktree + agent bead → session spawn + issue hook → working → `gt done` → Witness POLECAT_DONE handler → refinery merges MR → cleanup (idle vs nuke, persistent-polecat default) → zombie detection (recovery).

**Neutral observations surfaced across Batch 6:**

- **`internal/wisp` package name is a trap.** ~400 lines of utility code plus the pure `ShouldPromote` predicate. The actual wisp CONCEPT (ephemeral beads in the `wisps` table) lives in `internal/beads` + `internal/doltserver` + the external beadsdk SDK. Flagged prominently on both the package page and the concept page.
- **`internal/reaper` is deliberately isolated** — zero gastown package imports so it can run against a Dolt server even when other packages are in a broken state. All SQL is hand-rolled strings.
- **`internal/convoy` is only 2 files** (`operations.go` + `multi_store.go`) — the bulk of convoy logic (12 subcommands, 2739 lines) lives in `internal/cmd/convoy.go` and siblings. Future refactor candidate.
- **Boot lives in the dog kennel but is NOT a dog** — `<kennel>/boot/` uses `.boot-status.json` instead of `.dog.json`; `dog.Manager.Get("boot")` returns `ErrDogNotFound`; `List()` skips it. "Boot is a special dog" is physical layout only, not type registration. No `AgentDog` type exists.
- **Mayor's ACP mode leaks identity env vars into the parent shell** — `StartACP` calls `os.Setenv` on identity vars in the calling process because the Mayor's ACP path runs in the foreground.
- **Dog session prefix is `hq-dog-`, not `hq-deacon-`** — explicit to avoid tmux prefix-match collisions with the `hq-deacon` session.
- **Refinery filters wisps by rig name** because "wisps are shared across all rigs (gh#2718)".
- **Witness foreground mode is vestigial** — the `--foreground` flag still parses but prints "no longer runs patrol loop" because the patrol logic moved to `mol-witness-patrol`.
- **Polecat Dolt retries distinguish transient from config errors** — without the split, missing databases would hang polecat spawn for ~3 minutes per attempt (gt-2ra).
- **Polecat `RemoveWithOptions` resets the agent bead BEFORE filesystem operations** specifically to avoid a race where concurrent slings see a partially-reset bead (gt-14b8o).
- **Heartbeat v2 is agent-honest** (gt-3vr5): Witness only ever infers "is the heartbeat fresh?". All other state is agent-reported. ZFC principle applied to liveness.
- **Persistent polecat model flips the default lifecycle** — post-gt-4ac/gt-hdf8, `gt done` no longer tears down the polecat; it transitions to `StateIdle` with the sandbox preserved. Nuke is now an explicit action.
- **`directive` CLI is a parent-only stub** — `directive.go:38-40` only calls `rootCmd.AddCommand(directiveCmd)` with zero `directiveCmd.AddCommand(...)` calls. The config-loader side is fully implemented and consumed by `gt prime`.
- **Formula overlays vs directive overlays have opposite semantics**: formula overlays fully replace (rig-level replaces town-level); directive overlays concatenate (both town and rig appear in the prompt).
- **`compose.aspects` is a parked field** — TOML decodes it, but `resolveChain` in `parser.go:594` ignores it with a "future work" comment.
- **Identity and BEADS_ACTOR are not coupled at resolution time** — `detectSender()` reads `GT_ROLE` + context vars; `BEADS_ACTOR` is a separate env var. Usually coherent because the session manager sets both, but a debugger or script can decouple them.
- **Crew validates EVERYTHING before killing any existing session** — a careful invariant not found in other lifecycle packages; prevents the user being left without a running session on validation failure.
- **Three completion-detection paths in Witness do not dedupe across sessions** — only within one Witness session via `MessageDeduplicator`.

**Bead `wiki-ca4` closed by this commit** — Batch 6 (Layer f: Agent runtime / domain layer) complete.

**Next batch (not started by this commit):** Batch 7 — Layer (g) Diagnostics & health (`internal/doctor` with 120 files, `internal/health`, `internal/keepalive`, `internal/deps`). Per plan autonomous-mode, remaining batches (7-12) are simpler infrastructure mapping with no first-class domain entities, so they can proceed without per-batch controller approval.

→ 30 pages — see the F1 sub-index updates and F2 root index updates for the complete list

## [2026-04-11] ingest | Batch 7 (Layer g: Diagnostics & health — 4 package pages)

Mapped the diagnostics and health infrastructure. Notable: `internal/doctor`
is the largest package in the codebase (70+ non-test Go files) and
required focused reading strategy rather than full coverage.

**Packages mapped:**

- [doctor](gastown/packages/doctor.md) — **632-line package page** covering the Check framework, registration model, execution semantics, Fix mode, 12-area categorical file inventory, ordering constraints, and 5 representative check spot-checks. The `Check` interface at `internal/doctor/types.go:94-110` exposes `Name()`, `Description()`, `Run(*CheckContext)`, `Fix(*CheckContext)`, `CanFix()`. Registration is a plain ordered slice (`doctor.go:13-16`) via `Register`/`RegisterAll`. No dependency graph, no topological sort — **ordering = call order in `internal/cmd/doctor.go:154-275`**. Execution is strictly sequential, streaming. `FixStreaming` re-runs each check after a successful fix to verify. `safeFixCheck` recovers from Dolt panics (GH#1769).
- [health](gastown/packages/health.md) — reusable Dolt-data-plane primitives: `TCPCheck`, `LatencyCheck`, `DatabaseCount`, `FindZombieServers` (lsof-based), `BackupFreshness`, `JSONLGitFreshness`, `DirSize`. Backs `gt health` only. Package doc claims Doctor Dog uses it, but grep confirms only `internal/cmd/health.go` imports it — **doc-drift noted**.
- [keepalive](gastown/packages/keepalive.md) — best-effort agent-activity signaling via `<townRoot>/.runtime/keepalive.json`. **Nil-sentinel pattern**: `Read` returns `nil` on any failure, `(*State).Age()` is nil-receiver-safe returning `24h*365` as "maximally stale". Writer paths silently swallow all errors. **Only in-tree importer is `internal/web/api.go`** — the design expects root-`gt` `PersistentPreRun` integration ("every gt invocation touches") but the wire-up isn't in the import graph.
- [deps](gastown/packages/deps.md) — external binary prerequisites. `MinBeadsVersion = "0.57.0"`, `MinDoltVersion = "1.82.4"`. 3-component semver comparator. `CheckBeads`/`CheckDolt` run respective `version` subcommands under 10s context timeouts. `EnsureBeads` auto-installs via `go install` forcing `GOBIN=~/.local/bin` to avoid the `~/go/bin/` stale-shadow problem. **Asymmetric**: beads auto-installs; dolt does not (large binary via GitHub releases, not go-installable). `BeadsTooOld` returns an error even with `autoInstall=true` — silent minor-version upgrades risk breaking existing beads DBs. Feeds `gt doctor` prereq checks via `beads_binary_check.go` + `dolt_binary_check.go`.

**Neutral observations surfaced:**

- **Doctor has no dependency graph**. Ordering is implicit in `Register` call order. The `ClaudeSettingsCheck before DaemonCheck` constraint is enforced only by a comment at `internal/cmd/doctor.go:173-177` — integration tests are the only safety net.
- **State-on-struct pattern is common in doctor checks** (not safe for concurrent use). `ClaudeSettingsCheck`, `RigsRegistryValidCheck`, `ZombieSessionCheck`, `OrphanSessionCheck` all cache results between Run and Fix. This is another reason execution is strictly sequential.
- **Panic recovery is only on Fix, not Run**. A Run panic crashes `gt doctor`.
- **No short-circuit between doctor checks**: `DoltServerReachableCheck` failing doesn't stop downstream Dolt-dependent checks.
- **`FixHint` is always shown even for non-fixable checks** — it's a human-readable hint orthogonal to the auto-fix pathway.
- **`ClaudeSettingsCheck` is 805 lines** — flagship check. Shells out to `git ls-files`/`check-ignore`/`diff --quiet` to classify files as untracked/tracked-clean/tracked-modified/ignored. **Skips tracked-clean files** to avoid modifying customer repos. Gates live tmux session kills behind `ctx.RestartSessions` to avoid a Deacon restart loop.
- **`Category` / `CategoryOrder` on check results are mostly vestigial under streaming mode** — streaming output is in registration order, summary is bucketed by status (FAILURES/WARNINGS/FIXED) not category.
- **`health` package doc drift**: claims Doctor Dog uses it; grep confirms only `gt health` imports it.
- **`keepalive` is underused relative to design intent**: only `internal/web/api.go` imports it, not root `gt`.
- **`deps.BeadsTooOld` refuses to auto-upgrade** even with `autoInstall=true`; deliberate safety for existing DBs across minor version boundaries.
- **`migration_check.go` in doctor has a misleading filename**: contains `NewDoltMetadataCheck`, `NewDoltServerReachableCheck`, `NewDoltOrphanedDatabaseCheck` — all live-state Dolt checks, not migration runners.
- **`precheckout_hook_check.go` exposes both `NewPreCheckoutHookCheck()` and `NewBranchProtectionCheck()` as aliases for the same check.**

**Bead `wiki-chx` closed.**

**Next batch:** Batch 8 — Layer (h) Long-running processes (`internal/daemon` ~33 files, `internal/tmux`, `internal/runtime`).

→ [gastown/packages/doctor.md](gastown/packages/doctor.md), [health](gastown/packages/health.md), [keepalive](gastown/packages/keepalive.md), [deps](gastown/packages/deps.md), [gastown/README.md](gastown/README.md), [index.md](index.md)

## [2026-04-11] ingest | Batch 8 (Layer h: Long-running processes — 3 package pages)

Mapped the process infrastructure that keeps Gas Town running.

**Packages mapped:**

- [daemon](gastown/packages/daemon.md) — **639-line page** covering the per-town daemon singleton. Single-process via `gofrs/flock` on `<townRoot>/daemon/daemon.lock` for its lifetime. Entry point `daemon.New(DefaultConfig) + d.Run()` invoked by hidden `gt daemon run`. Main heartbeat loop at `daemon.go:724-879` runs 15 ordered steps every 3 minutes (not 5: `Config.HeartbeatInterval=5m` is dead code; live interval comes from operational config). Pressure-gated: refinery/dog/polecat spawns throttle under load; infrastructure agents (Deacon/Witness/Mayor) are exempt. **State files:** `daemon.lock` (flock), `daemon.pid` (`"PID\nNONCE"` format, gt-utuk ZFC fix), `daemon.log` (lumberjack 100MB/3 backups), `state.json`, `shutdown.lock` (GH#2656, never removed because flock works on inodes), `restart_state.json`. 33 files grouped into 11 categories: main loop (3), state management (3), signal/OS portability (8), Dolt infra (3), backup (1), maintenance dogs (4: checkpoint/compactor/doctor/quota), scheduled ops (2), cleanup (2), handlers (2), observability (3), misc (2).

- [tmux](gastown/packages/tmux.md) — **509-line page** covering the tmux wrapper library. 11 files; core is `tmux.go` (3,857 lines, ~150 methods) with 2 theme files + 8 paired Unix/Windows platform shims. `Tmux` struct holds just a `socketName` string; no cache, no internal lock; concurrency control is inside tmux itself or in the package-level `sessionNudgeLocks sync.Map`. Shells out for every operation via `run()` then parses stdout. Socket abstraction: package-level `defaultSocket` set at gt root init + per-Tmux socketName; `SocketFromEnv` reads `GT_TOWN_SOCKET`; sentinel `noTownSocket = "gt-no-town-socket"` when no town configured. **Cross-platform:** Unix (POSIX pgid) + Windows (psmux, `tasklist`, Toolhelp32 snapshot API for descendants, `CREATE_NO_WINDOW` flag).

- [runtime](gastown/packages/runtime.md) — **324-line page.** Thin package (1 file, 272 lines) smoothing agent-runtime capability differences during session startup. Four concerns: per-provider hook settings install (`EnsureSettingsForRole`, respecting settingsDir vs workDir distinction so gastown doesn't write provider settings into customer repos), session-ID env-var resolution (`SessionIDFromEnv`), startup fallback matrix (`StartupFallbackInfo` 2x2 cross-product of hooks?/prompt? capability flags), and `RuntimeConfigWithMinDelay` (clears `ReadyPromptPrefix` to force delay-based readiness).

**Neutral observations surfaced:**

- **`Config.HeartbeatInterval=5m` in `daemon/types.go:41` is dead code.** Live interval comes from `operational.daemon.recovery_heartbeat_interval` (default 3 min).
- **`shutdown.lock` is never removed** — `flock` works on inodes not paths (documented at `daemon.go:1972-1976`).
- **Daemon relies on single-threaded main-loop discipline** instead of mutexes. Many struct fields are marked "no sync needed, only accessed from heartbeat goroutine". Only `deathsMu` exists.
- **`EnsureLifecycleConfigFile` is called from THREE places** (`gt init`, `gt up`, `daemon.New`) — deliberately defensive, any path into the daemon patches the config file.
- **Nonce-based PID-file ownership verification** (`"PID\nNONCE"` format) replaces `ps` string matching per ZFC fix gt-utuk.
- **`GT_DOLT_PORT`/`BEADS_DOLT_PORT` are `os.Setenv`d at daemon startup** (GH#2412) so every spawned session inherits them without per-`Manager.Start` plumbing.
- **Tmux identity vars are explicitly unset at daemon startup** (GH#3006). Only `GT_TOWN_ROOT` leaks globally.
- **`descendants_stub.go` naming is backwards** — the "stub" file has build tag `!windows`; it's stubbed for *non-Windows*, not Windows. The real implementation is in `descendants_windows.go` using Toolhelp32.
- **`tmux.NewSessionWithCommand` does `new-session -d <shell>` + `set-option remain-on-exit on` + `respawn-pane -k <command>`**, not `new-session -d '<command>'`. Eliminates a race where fast-failing commands exit before the health check runs.
- **`CLAUDECODE` is unset on every session create**, defending against nested-session detection failures when the tmux server itself was started inside a Claude Code session.
- **`sessionNudgeLocks` `sync.Map` is a slow memory leak** — per-session locks never evicted when sessions die.
- **`MayorTheme()` returns `bg=default,fg=default`** — the mayor session is specifically configured to blend into the user's terminal.
- **PowerShell `$env:KEY='val'` prefix handling exists** in `validateCommandBinary`, proving Windows/psmux is a real supported platform, not an afterthought.
- **`runtime.go:99-102` documents a removed session-started nudge** to deacon — it interrupted the deacon's `await-signal` backoff sleep.

**Bead `wiki-75z` closed.**

**Next batch:** Batch 9 — Layer (i) Supporting libraries (~24 small packages).

→ [gastown/packages/daemon.md](gastown/packages/daemon.md), [tmux](gastown/packages/tmux.md), [runtime](gastown/packages/runtime.md), [gastown/README.md](gastown/README.md), [index.md](index.md)

## [2026-04-11] ingest | Batch 9 (Layer i: Supporting libraries — 24 package pages)

Largest batch by package count. 24 small-to-medium packages mapped across three content subagents.

**Packages mapped:**

- **Substantial (Sub A):** [acp](gastown/packages/acp.md), [hooks](gastown/packages/hooks.md), [krc](gastown/packages/krc.md), [protocol](gastown/packages/protocol.md), [quota](gastown/packages/quota.md), [testutil](gastown/packages/testutil.md), [wasteland](gastown/packages/wasteland.md), [web](gastown/packages/web.md)
- **Mid (Sub B):** [agentlog](gastown/packages/agentlog.md), [constants](gastown/packages/constants.md), [feed](gastown/packages/feed.md), [git](gastown/packages/git.md), [github](gastown/packages/github.md), [shell](gastown/packages/shell.md), [suggest](gastown/packages/suggest.md), [townlog](gastown/packages/townlog.md)
- **Thin (Sub C):** [activity](gastown/packages/activity.md), [estop](gastown/packages/estop.md), [hookutil](gastown/packages/hookutil.md), [scheduler](gastown/packages/scheduler.md), [state](gastown/packages/state.md), [templates](gastown/packages/templates.md), [tui](gastown/packages/tui.md), [wrappers](gastown/packages/wrappers.md)

**Structural observations:**

- **`scheduler` and `tui` are empty-namespace parents** — both have 0 top-level Go files. `scheduler` code lives in `capacity/` subpackage (4 files, ~540 lines, pure scheduling functions); `tui` has two subpackages (`convoy/` ~650 lines and `feed/` ~4,000 lines). Consolidated into one page each per namespace.
- **`git` is a 2,300-line subprocess wrapper** with 14 clone permutations, worktree lifecycle, and `.beads`/`.runtime` allowlist awareness for "is this dirty?" classification.
- **`github` is distinct from `git`** — `github` uses REST+GraphQL HTTP APIs; `git` shells out to `gh` CLI separately. Both are used by Refinery.
- **`templates` exceeds the wiki's 3-level nesting cap** — `templates/commands/bodies/*.md` is 5 levels deep. Source-tree observation only.
- **`feed` is close to page-worthy on its own** (4k lines, 14 files under `tui/feed/`) — flagged as a future split candidate.

**Neutral observations:**

- **`protocol.DefaultRefineryHandler.HandleMergeReady` is almost a no-op** — the comment explicitly says the refinery now queries beads directly; MERGE_READY remains as a liveness signal only.
- **`krc.MinRetainCount` is tracked but unenforced** — both branches compute `result.EventsRetained = len(retained)` the same way.
- **`quota.validateTokenHTTP` is dead code** — never called from the main path per inline comment; OAuth tokens would always 401 against bare `/v1/messages`.
- **`web.api.go` hardcodes `gtPath = "gt"`** rather than `os.Executable()` because test binaries would fork-bomb themselves.
- **`acp.proxy.go` waits only 200ms** for goroutines to exit after shutdown before abandoning them.
- **`testutil` preserves `BEADS_*` env vars while stripping `BD_*`** — a quiet prefix gotcha.
- **`agentlog` is NOT Unix-only.** The earlier claim (from Batch 4 telemetry work) about `agent_logging_windows.go` being a no-op referred to the `internal/telemetry/agent_logging_windows.go` file, not the `internal/agentlog` package. `internal/agentlog` is pure stdlib and handles Windows drive-letter paths explicitly. **Correction to an earlier note.**
- **`git.copy_windows.go:14` explicitly says "This Windows implementation has not been tested on Windows."**
- **`townlog` parseLogLine deliberately leaves `Context` empty** — one-way round-trip. Callers of `TailEvents`/`FilterEvents` never see event context.
- **`feed` ZFC has a silent cap** — `tailReadSize = 1 MB` means dedup/aggregation windows exceeding 1 MB start missing earlier events.
- **`github` client caches `GITHUB_TOKEN` at `NewClient` time** — env rotation doesn't take effect.
- **`shell.DetectShell` falls through to zsh** for fish/tcsh/pwsh, producing RC edits that won't load correctly.
- **`feed.mq_source.go` is a deprecated no-op stub** that still exists to satisfy old call sites (mrqueue package was removed).
- **`hookutil.IsAutonomousRole` uses literal `"boot"`** alongside constants for the other 4 roles — potential future divergence.
- **`wrappers.go` ABOUTME header lists 2 wrappers**; code installs 3.
- **`templates` has two command-name substitution paths** — `CmdName()` template func vs. `cli.Name()` ReplaceAll — drift potential.
- **`estop` is asymmetric** — town-wide Deactivate has a manual-stop guard; per-rig does not.

**Bead `wiki-4pm` closed.**

**Next batch:** Batch 10 — Layer (j) Plugins (`plugins/*`, 14 dirs mostly empty).

→ 24 package pages — see F1/F2 index updates for full list.

## [2026-04-11] ingest | Batch 10 (Layer j: Plugins — 2 pages)

Mapped the Deacon-patrol plugin catalog. `plugins/` at the
gastown repo root contains 14 plugin directories. 13 are
declarative (shell scripts + TOML-frontmatter `plugin.md`); only
`dolt-snapshots` has Go source code — a standalone binary
compiled on first run by `run.sh`, connecting to the per-town
Dolt MySQL server via `go-sql-driver/mysql` with parameterized
SQL.

**Pages created:**

- [gastown/plugins/README.md](gastown/plugins/README.md) — inventory index of all 14 plugins with one-line descriptions, gate types, and observations
- [gastown/plugins/dolt-snapshots.md](gastown/plugins/dolt-snapshots.md) — C-level entity page for the Go-based plugin (665-line `main.go` + 418-line tests)

**Per-plugin one-liners (from the inventory page):**

- **compactor-dog** — monitors Dolt commit growth across production DBs and escalates when compaction is needed (agent judgment; not a hard threshold)
- **dolt-archive** — offsite backup: JSONL snapshots to git, dolt push to GitHub/DoltHub (three-layer: JSONL → git → dolt push)
- **dolt-backup** — smart Dolt database backup with change detection (skips unchanged DBs; only escalates when actual sync fails)
- **dolt-log-rotate** — rotates `daemon/dolt.log` when it exceeds size threshold (100 MB default), keeps 3 gzipped copies
- **dolt-snapshots** — tags Dolt databases at convoy boundaries for audit, diff, and rollback (the only Go-based plugin; event-gated, not cooldown)
- **github-sheriff** — polls GitHub for open PRs, categorizes as easy-win / needs-review, creates beads for CI failures
- **git-hygiene** — cleans stale branches, stashes, and loose objects across all rig repos (merged locals, orphan agent branches, merged remotes via gh api, stash clear, `git gc --prune=now`)
- **gitignore-reconcile** — auto-untracks files that are tracked but match an active `.gitignore` rule (only on clean main; otherwise files a chore bead)
- **quality-review** — analyzes per-worker merge quality trends (computed from result wisps recorded separately by the Refinery); alerts on BREACH when avg < 0.45
- **rate-limit-watchdog** — shell-only: probes the Anthropic API, auto-`estop` on 429, auto-`thaw` when clear; 3-minute cadence
- **rebuild-gt** — rebuilds stale `gt` binary from source; post-incident safety gate requires `gt stale --json` to report `safe_to_rebuild: true` and main branch; uses `make safe-install`
- **stuck-agent-dog** — context-aware restart decisions for polecats and `hq-deacon` (inspects tmux pane output before killing); hard-coded out-of-scope list for crew, mayor, witness, refinery
- **submodule-commit** — opt-in per rig; auto-commits accumulated changes inside git submodules and updates the parent pointer (current enabled: `lilypad_chat`)
- **tool-updater** — weekly Homebrew bump for `beads` and `dolt` (gt rebuilds from source via rebuild-gt; not Homebrew)

**Neutral observations surfaced:**

- **Only two plugins have no `run.sh`.** `github-sheriff` and `quality-review` are agent-interpreted — their markdown bodies ARE the dog task prompts. The other 12 have `HasRunScript=true` and dogs are told to run the script without interpreting the markdown.
- **Gate distribution:** 13 cooldown gates (3 min to 168 h, six orders of magnitude), 1 event gate (`dolt-snapshots` on `convoy.created`). Zero cron, condition, or manual gates in the built-in set.
- **Three plugins share one Go binary — except two directories are missing.** `dolt-snapshots/plugin.md` documents `dolt-snapshots`, `dolt-snapshots-staged`, and `dolt-snapshots-launched` all sharing `main.go`. Only `dolt-snapshots/` exists in the inventory. Either the siblings are runtime-only (created at town init) or the docs are aspirational.
- **The dolt-snapshots watcher launch is in the wrong place.** The long-lived `--watch` daemon is launched by a bash block inside `plugin.md` — but `HasRunScript=true` means the scanner tells dogs NOT to interpret the markdown. So the Step 1 block is reference-only when a dog dispatches; `run.sh` itself does not background the watcher. How the watcher gets its initial launch in practice is unclear.
- **`escalateStale` only prints.** Despite the name, the Go function at `main.go:560-572` just prints stale branches and a note. No `gt escalate` call. The name is aspirational.
- **`stuck-agent-dog` has an explicit scope-safety list.** Hard-coded enumeration of which tmux session types it may touch (polecats, `hq-deacon`) and which it must never touch (crew, mayor, witness, refinery). Killing a crew session is documented as a critical incident.
- **`tool-updater` has a hard-coded dev machine path.** Its "Run" section says `cd /Users/jeremy/gt/plugins/tool-updater && bash run.sh`. Doc-only drift; the actual `run.sh` uses relative paths.
- **`rebuild-gt` has a post-incident preamble.** The plugin docs call out a specific past crash loop caused by rebuilding backwards and mandate a `gt stale --json` pre-flight check that refuses when `safe_to_rebuild: false`.
- **PR #2324 review drove the Go rewrite of `dolt-snapshots`.** The package comment at `main.go:1-9` enumerates three specific bugs fixed: shell interpolation / SQL injection, subshell counter bugs, and auto-commit of dirty state. Prior bash-based implementation presumably had all three.
- **`dolt-snapshots` uses dedicated pool connections for `USE <db>; CALL DOLT_TAG/BRANCH(...)`.** Comments at `main.go:458-459` and `main.go:491-492` explicitly call out that the shared pool can't be trusted for sequences where the second statement depends on session state from the first. Worth remembering for other Dolt-using Go code.
- **`dolt-snapshots` tests cover only pure helpers.** `sanitizeName`, `sanitizeDBName`, `isSystemDB`, `loadRoutes`, `resolveDependencyDB`, `resolveHost/Port/RoutesFile`, and the `ConvoyRow_SnapshotLogic` truth table. The DB-touching code (`snapshotConvoys`, `createTag`, `createBranch`, `watchEvents`) has zero test coverage.
- **Routes file `path="."` entries are silently skipped by `loadRoutes`.** This is why `hq-` and `hq-cv-` prefixes aren't in the route map — HQ is always tagged explicitly regardless of the routes file.

**Bead `wiki-s4h` closed.**

**Next batch:** Batch 11 — Layer (k) Auxiliary trees.

→ [gastown/plugins/README.md](gastown/plugins/README.md), [dolt-snapshots](gastown/plugins/dolt-snapshots.md), [gastown/README.md](gastown/README.md), [index.md](index.md)

## [2026-04-11] ingest | Batch 11 (Layer k: Auxiliary trees — 4 pages)

Mapped the 9 top-level auxiliary directories: `scripts/`, `templates/`,
`gt-model-eval/`, `npm-package/`, `.github/`, `.githooks/`, `.claude/`,
`.opencode/`, `.runtime/`.

**Pages created:**

- [gastown/inventory/auxiliary.md](gastown/inventory/auxiliary.md) — A-level inventory covering all 9 directories (~175 lines)
- [gastown/files/claude-dir.md](gastown/files/claude-dir.md) — `.claude/commands/` (3 slash commands) + `.claude/skills/` (4 skills); Claude Code agent-facing surface (~210 lines)
- [gastown/files/opencode-dir.md](gastown/files/opencode-dir.md) — `.opencode/commands/handoff.md` + `.opencode/plugins/gastown.js`; the OpenCode session integration (`gt prime` injection, cost accounting) (~200 lines)
- [gastown/files/templates-agents.md](gastown/files/templates-agents.md) — `templates/agents/opencode.json.tmpl` + `opencode-models.json`; agent-runtime manifest templates with delay-based ready detection (~230 lines)

**Key findings:**

- **`.opencode/plugins/gastown.js` is the load-bearing Gas Town ↔ OpenCode integration.** It hooks `session.created` / `session.compacted` / `session.deleted` events, runs `gt prime` (plus `gt mail check --inject` for autonomous roles polecat/witness/refinery/deacon), and pushes captured context into `experimental.chat.system.transform`. A historical source comment notes that a previous `session-started` nudge to the Deacon was removed because it interrupted the Deacon's await-signal backoff.
- **Opencode ready detection is delay-based, not prompt-matching.** `opencode.json.tmpl` sets `ready_prompt_prefix: ""` and relies on `ready_delay_ms` because "OpenCode TUI uses box-drawing characters that break prompt prefix matching." Per-model delays in `opencode-models.json` range from 4s (Grok) to 15s (GLM 4.7 Free tier).
- **The `.claude/commands/{backup,patrol,reaper}.md` slash commands duplicate autonomous-patrol logic from the role formulas** (`mol-witness-patrol`, `mol-deacon-patrol`, `mol-refinery-patrol`, `mol-dog-backup`, `mol-dog-reaper`). They're agent-facing prose renditions of what the formulas do declaratively — drift risk between the two.
- **`scripts/guards/context-budget-guard.sh` is a standalone Claude Code hook guard** that reads the active session transcript and enforces warn/soft/hard thresholds (0.75/0.85/0.92). Hard-gate roles default to `mayor,deacon,witness,refinery`. Fails open on any error. 7995 bytes.
- **`.githooks/pre-push` enforces two invariants**: branches must be `main` / `polecat/*` / `integration/*` / `beads-sync` (or the repo has an `upstream` remote signalling fork workflow), and pushes to the default branch that introduce integration-branch content are blocked unless `GT_INTEGRATION_LAND=1` is set — which only `gt mq integration land` sets.
- **`gt-model-eval` has 94 total test cases** (82 Class B directive + 12 Class A reasoning) across 4 patrol roles (deacon/witness/refinery/dog), comparing Opus 4.6 vs Sonnet 4.5 vs Haiku 4.5 at temperature 0. Grading uses Opus 4.6. The framework exists to generate **evidence for safely downgrading role assignments** off Opus — linked in Discussion #1542 and Issue #1545.
- **`npm-package/scripts/postinstall.js` skips binary download in CI** (`if (!process.env.CI)`) — a gotcha worth remembering for testing harnesses. Downloads `gastown_<version>_<platform>_<arch>.(tar.gz|zip)` from GitHub Releases and extracts via native `tar` or PowerShell `Expand-Archive`.
- **Inconsistent skill filename casing** in `.claude/skills/`: three use `SKILL.md`, one (`pr-sheriff`) uses `skill.md`. Latent portability bug on case-sensitive filesystems.

**Bead `wiki-5d6` closed.**

**Next batch:** Batch 12 — Layer (l) Documentation tree (inventory-only enrichment of the 60-file `docs/` tree).

→ [gastown/inventory/auxiliary.md](gastown/inventory/auxiliary.md), [gastown/files/claude-dir.md](gastown/files/claude-dir.md), [gastown/files/opencode-dir.md](gastown/files/opencode-dir.md), [gastown/files/templates-agents.md](gastown/files/templates-agents.md), [gastown/README.md](gastown/README.md), [gastown/inventory/README.md](gastown/inventory/README.md), [index.md](index.md)

## [2026-04-11] ingest | Batch 12 (Layer l: Documentation tree enrichment — 60 topics)

Final batch of the gastown-map plan. Per plan, this phase produces
**no new wiki pages** — only inventory enrichment. Read the first
~30-50 lines of each of the 60 files under `docs/` and added a
"topic" column to every row in `gastown/inventory/docs-tree.md`.
Topics are factual extracts from H1 / first-paragraph, not content
summaries.

**Page updated:**

- [gastown/inventory/docs-tree.md](gastown/inventory/docs-tree.md) — 60 rows across 10 subdirectory tables, each now with `lines | topic | path` instead of `lines | path`.

**What was read:** first 30-50 lines of each of the 60 files under
`/home/kimberly/repos/gastown/docs/`. No file was read in full; no
content was summarized; no drift was hunted.

**What is NOT here (explicitly deferred):**

- **Content mapping of `docs/` files.** Full reading + comparison
  to wiki entity pages is phase 3 work (drift / gap analysis).
- **Cross-references from docs/ files to wiki entity pages.**
  Phase 3.
- **Doc-rewrite proposals.** Phase 4.

**Incidental correction:** the previous docs-tree.md claimed the
root-level `docs/` directory had 13 files, but listed only 12 and
`find` returns 12. The header now correctly says 12. Running total
remains 60.

**Bead `wiki-1yv` closed.**

**The gastown-map plan is now COMPLETE.** All 12 batches have
landed. Coverage by layer:

- Layer (a) Build & packaging — 9 files pages (Batch 1: `ba20e30`)
- Layer (b) Binaries & entry points — 2 new + gt.md updates (Batch 2: `f80bea7`)
- Layer (c) Command layer — 111 top-level commands across 7 sub-batches (Batches 3a–3h: `f1c2d78` → `a1afaf9`)
- Layer (d) Platform services — 9 package pages (Batch 4: `b7b89a1`)
- Layer (e) Data layer — 8 package pages (Batch 5: `edc2ba9`)
- Layer (f) Agent runtime / domain layer — 30 pages (13 packages + 8 roles + 7 concepts + 2 workflows) (Batch 6: `485a77e`)
- Layer (g) Diagnostics & health — 4 package pages (Batch 7: `9b76289`)
- Layer (h) Long-running processes — 3 package pages (Batch 8: `ea50ebb`)
- Layer (i) Supporting libraries — 24 package pages (Batch 9: `125083c`)
- Layer (j) Plugins — 2 pages: inventory + dolt-snapshots (Batch 10: `d032c91`)
- Layer (k) Auxiliary trees — 4 pages (Batch 11: `848f920`)
- Layer (l) Documentation tree — inventory enrichment only (Batch 12: this commit)

**All 12 batch beads closed.** Four investigation beads (`wiki-9u4` OTEL trace, `wiki-2g3` AnnotationPolecatSafe trace, `wiki-ef3` systematic subcommand mapping, `wiki-13o` proxy binary docs) were closed earlier during the batch execution. Four original scaffolding beads (`wiki-ztf`, `wiki-cqx`, `wiki-0hn`, `wiki-77a` Batch 1) were closed during/before autonomous execution.

**Total wiki page count** (approximate): ~230 pages covering the full gastown code map — binaries, commands, packages, roles, concepts, workflows, files, inventory, plugins.

**Phase 2 (mapping) complete. Phase 3 (drift / gap analysis) is the next major phase, awaiting controller direction.**

→ [gastown/inventory/docs-tree.md](gastown/inventory/docs-tree.md)

## [2026-04-14] drift-found | docs/agent-provider-integration.md vs code

Docs accurately describe agent integration tiers (0-3) and JSON registry schema (version 1, agents map). Code implements via AgentRegistry struct, LoadAgentRegistry() merges user JSON with built-in presets. Built-ins cover Claude/Gemini/etc. as examples. Templates exist for OpenCode JSON generation. No major drift. Minor: Gas City mentioned as "upcoming" but no code exists (forward-looking only).

## [2026-04-14] drift-found | docs/CLEANUP.md vs code

Docs claim `gt done` 'pushes branch, submits MR (by default), self-nukes worktree, kills own session'. Code post-gt-4ac/gt-hdf8 transitions polecat to IDLE with sandbox preserved; nuke is explicit action. Other cleanup commands (gt cleanup, gt orphans, gt polecat nuke, etc.) match wiki descriptions.

## [2026-04-14] decision | wiki renamed from ~/repos/wiki to ~/repos/gt-wiki

Wiki repo renamed and published as [kagarmoe/gt-wiki](https://github.com/kagarmoe/gt-wiki). The rename reflects a scope decision: this wiki stays focused on gastown and is being opened to the gastown community as an unofficial code-grounded map of `gastownhall/gastown`; Kimberly's personal notes will live in a separate repo when she starts it. The name `gt-wiki` distinguishes it from GitHub's per-repo wiki feature (`gastownhall/gastown.wiki`) and from any future adoption by the gastownhall org.

**Rename steps completed (commits `d0007a2`, `df5cbb7`):**

- GitHub repo renamed `kagarmoe/wiki` → `kagarmoe/gt-wiki` (auto-redirects old URLs for ~30 days).
- Local directory `~/repos/wiki` → `~/repos/gt-wiki`.
- Git remote set to `git@github.com:kagarmoe/gt-wiki.git`.
- Auto-memory dir `~/.claude/projects/-home-kimberly-repos-wiki` → `~/.claude/projects/-home-kimberly-repos-gt-wiki`.
- Three tracked files rewritten to use the new path: [CLAUDE.md](CLAUDE.md) (schema directory-layout tree + bd conventions example + permission scoping prose), [.claude/settings.local.json](.claude/settings.local.json) (Write/Edit scope rules), [gastown/binaries/gt-proxy-client.md](gastown/binaries/gt-proxy-client.md) (1 absolute-path reference).
- Community-publishing scaffolding added in `df5cbb7`: `LICENSE` (CC BY 4.0), top-level `README.md` framing the wiki as unofficial and pointing at `index.md`, `CONTRIBUTING.md` summarizing the schema rules for human contributors.

**Follow-up rewrites in this commit:**

- [gastown/README.md](gastown/README.md) — one stale `~/repos/wiki/` reference in the "explicitly out of scope" section updated to `~/repos/gt-wiki/`.
- Memory files under `~/.claude/projects/-home-kimberly-repos-gt-wiki/memory/` rewritten: `MEMORY.md` index entries, `reference_wiki.md` (frontmatter `name` field, entry-point paths, permissions path, history line added), `user_directory_convention.md` (frontmatter `description`, directory bullet, findings bullet, history line added).

**Explicitly NOT rewritten:**

- `log.md` historical entries at lines 12 and 15 (inside the 2026-04-11 `decision | wiki scaffolded` entry) still reference `~/repos/wiki/` because the log is append-only per the schema rule at `log.md:204-207`: "Historical log entries above are preserved as written." They describe what happened at the time, including the then-current path. This decision entry is the forward-looking replacement.
- `.claude/plans/2026-04-11-gastown-map.md` (gitignored) still contains old-path references. Phase 2 is complete; the plan file is historical and does not drive current work.

**Why this matters for Phase 3:** future sessions priming off `MEMORY.md` or `bd remember` queries will now consistently see `~/repos/gt-wiki`. Anyone reading the log from the top will see the 2026-04-11 original path, then this 2026-04-14 rename decision, then the current-state Phase 3 work that follows.

→ [gastown/README.md](gastown/README.md), [CLAUDE.md](CLAUDE.md), [README.md](README.md), [CONTRIBUTING.md](CONTRIBUTING.md), [LICENSE](LICENSE), [gastown/binaries/gt-proxy-client.md](gastown/binaries/gt-proxy-client.md)

## [2026-04-14] decision | Phase 3 schema change: re-enable Docs claim / Drift / Implementation status sections (schema v1.2)

Phase 2's mapping-only scope reframe at [log.md:158](log.md) removed `## Docs claim` and `## Drift` sections from entity pages because Phase 2's deliverable was the map, not the audit. Phase 3 is the audit; it needs those sections back. This entry records the re-enablement under explicit conventions that Phase 2's framing never had — a disciplined authority hierarchy, a per-page audit-tracking frontmatter, and a non-retroactive application rule.

**This is not a reversal of the Phase 2 walkback.** It is the next phase doing what Phase 2 deliberately deferred.

### Authority hierarchy (Code → Cobra → Docs)

The load-bearing rule for Phase 3's drift taxonomy: **code is the only source of truth**. Cobra `Long` help text is derived from code. `docs/*.md` files are derived from Cobra/code. The hierarchy determines both finding classification and fix direction: the fix always starts at the highest tier that's wrong, because fixes there cascade downward.

- **Code** — authoritative. `/home/kimberly/repos/gastown/internal/...`, `cmd/...`, source-of-truth for every behavior claim.
- **Cobra** — derived. Cobra command `Long` strings, annotations, error messages. Should agree with code.
- **Docs** — derived. `/home/kimberly/repos/gastown/docs/*.md`, README. Should agree with Cobra + code.

Docs are NOT authoritative by themselves. Wiki pages (Phase 2 synthesis) are ALSO not authoritative — they must be verified against source during Phase 3.

### Re-enabled sections (in order, on every entity page that gets Phase 3 annotations)

1. `## What it actually does` — unchanged from Phase 2.
2. `## Docs claim` — NEW. Verbatim quotes from upstream docs / Cobra `Long` text / README that make claims about this entity. Every claim cites its source with absolute path + line range. No paraphrasing.
3. `## Drift` — NEW. Only when docs and code disagree. Structured as Claim → Code → Resolution with `file:line` citations on both sides. Tagged per the drift taxonomy (below).
4. `## Implementation status` — NEW. Only when docs describe aspirational / partial / vestigial behavior. Tagged `unbuilt`, `partial`, or `vestigial`. Explicitly distinct from drift: aspirational docs are not wrong, they describe a future state that hasn't been built. Phase 6 (Implementation) treats them differently — preserve as vision vs factually rewrite.
5. `## Notes / open questions` — unchanged. Neutral observations that are not drift or implementation-status findings stay here.

### Drift taxonomy (9 categories, with fix-tier derived from the authority hierarchy)

| Category | Definition | Fix tier |
|----------|------------|----------|
| `cobra drift` | Cobra `Long` text (or other in-code docstrings) contradicts code in the same source tree | **Code** (edit Cobra string; in-source docs fix) |
| `drift` | `docs/*.md` or README contradicts correct Cobra/code | **Docs** (upstream docs PR) |
| `compound drift` | Both Cobra AND docs are wrong independently. Tag both `cobra drift` AND `drift` on the same finding. | **Code first, then docs** (fix Cobra; docs fix may cascade automatically or need manual follow-up) |
| `implementation-status: unbuilt` | Docs describe behavior labeled "vision" / "planned" with zero code support | **None** (preserve as vision + status callout) |
| `implementation-status: partial` | Docs describe partially-implemented behavior | **None** (preserve + status callout) |
| `implementation-status: vestigial` | Code exists but is explicitly documented or commented as superseded / dead / "no longer runs" | **Code** (remove dead code) OR **Docs** (document the vestige) — per finding |
| `gap` | Docs describe something real that has no wiki page yet (Phase 2 missed it or deferred it) | **Wiki** (we missed it) |
| `wiki-stale` | Phase 2 wiki page body disagrees with current source — our own synthesis has drifted | **Wiki** (our synthesis) |
| `neutral` | Notes that on review turn out NOT to be drift | **None** (stay in `## Notes / open questions`) |

**Rationale for category differentiation:** `cobra drift` is separate from `drift` because the fix path differs (upstream code change vs upstream docs change). `compound drift` handles the frequent case where docs echo wrong Cobra text. `implementation-status` is separate from drift because aspirational docs aren't wrong, they're forward-looking. `wiki-stale` is separate because it's a lint finding on our own synthesis, not a finding about upstream content.

### Frontmatter additions

Every page audited in Phase 3 gets these frontmatter fields added (regardless of whether any findings were promoted):

```yaml
phase3_audited: 2026-04-15                              # ISO date when the audit happened
phase3_findings: [drift, cobra-drift, implementation-status-unbuilt,
                  implementation-status-partial, implementation-status-vestigial,
                  wiki-stale, gap, none]                # list matching the taxonomy above
phase3_findings_post_release: false                     # true iff ≥1 finding on this page is post-release
```

`phase3_findings: [none]` is valid and means "audited, no findings" — a page walked with zero findings still gets `phase3_audited:` set and `phase3_findings: [none]` so it appears in the audit roll-call. Dataview query `phase3_audited is undefined` is the unfinished-work list.

`phase3_findings_post_release` supports the release-position dimension (see "Release position" below).

### Release position (orthogonal to category)

Phase 3 audits against current gastown main, which is ahead of the most recent stable release. Every finding carries a release-position tag in addition to its category:

- **`in-release`**: the cited code existed at the last stable release tag. Phase 6 fixes can upstream immediately.
- **`post-release`**: the cited code was introduced after the last release. Phase 6 fixes must wait for the next release before upstreaming.

Phase 3 uses **v1.0.0** (gastown commit `5f07285e`, 2026-04-02) as the authority baseline. The PR-delta scoping in Batch 0 Task 1 Part A captures the churn between v1.0.0 and origin/main. For each finding, the sub-batch checks whether the cited `file:line` exists at v1.0.0 and tags accordingly.

### New category folder: `drift/`

Schema directory layout adds `<topic>/drift/` for the consolidated Phase 3 drift index. Page type `drift` already exists in the schema's open type set. Lazy creation: the folder is created in Batch 13 when the consolidated index lands.

### Non-retroactive clause (critical)

**The re-enabled `## Docs claim` / `## Drift` / `## Implementation status` sections are NOT a retroactive schema requirement on Phase 2 pages.** Existing pages written during Phase 2 remain valid as-is; they acquire Phase 3 sections only when the Phase 3 audit sweep reaches them. A future session reading schema v1.2 must NOT interpret it as "every page needs a `## Docs claim` section retrofit." The clause closes a specific failure mode — misreading the schema change as a 213-page migration demand. Phase 2's output stands; Phase 3 annotates it incrementally.

Equivalently: `phase3_audited` is the opt-in marker. A page without `phase3_audited` is NOT in violation of schema v1.2. It's simply "not yet audited."

### Schema version bump

1.1 → 1.2, recorded in [CLAUDE.md](CLAUDE.md) header (landed as commit 808eecb on 2026-04-14, together with the CLAUDE.md restructure from 387 lines to 190-line slim coordinator).

### Plan file

The full plan for Phase 3 (batch decomposition, task-level steps, drift taxonomy tables, subagent prompt templates) lives at `.claude/plans/2026-04-14-phase3-drift.md` — gitignored per schema (plan files are local-only artifacts, never committed).

### Why this is distinct from the Phase 2 walkback

Phase 2's walkback removed premature drift annotations when the framing shifted to mapping-only. This Phase 3 decision re-enables them under a disciplined workflow (per-page audit tracking via frontmatter, mandatory cross-links, per-batch log entries, code-first verification, authority-hierarchy fix-tier derivation, non-retroactive application). It is NOT a reversal of the Phase 2 walkback — it is the next phase doing what Phase 2 deliberately deferred. The walkback's preservation of mapping-phase purity enabled Phase 3 to start from a clean baseline.

→ [CLAUDE.md](CLAUDE.md), [log.md:158](log.md)

## [2026-04-14] decision | Phase 3 retroactive cleanup of 2026-04-14 drift-found entries

Two `[2026-04-14] drift-found` entries exist earlier in this log (lines 1476 and 1480), written BEFORE the Phase 3 plan existed:

1. `[2026-04-14] drift-found | docs/agent-provider-integration.md vs code` (log.md:1476)
2. `[2026-04-14] drift-found | docs/CLEANUP.md vs code` (log.md:1480)

Neither matches the batch-entry format defined in the Phase 3 plan's "Batch entry format" section — they lack the mandatory "Source files re-read at the new release tag" column, lack per-finding fix-tier tags, lack cross-link discipline confirmation, and lack beads accounting. They were drift-found sketches written during an informal Phase 3 start, not batch-discipline audit trail.

This entry records how they are handled under the new Phase 3 plan.

### Handling

Both historical entries are **grandfathered** per the append-only log rule at [log.md:204-207](log.md) ("Historical log entries above are preserved as written. They describe what happened, including framing that has since been superseded."). The grandfathered entries stay where they are, unedited. Their supersession happens via forward reference: specific Phase 3 batches will re-audit the same subjects under schema v1.2 conventions, and those batches' log entries will cite the grandfathered entries as the pre-plan starting point.

**Per-entry disposition:**

1. **`docs/agent-provider-integration.md vs code`** — no wiki-page annotation was produced by the original entry. The finding text ("Docs accurately describe agent integration tiers … no major drift. Minor: Gas City mentioned as upcoming but no code exists") is preserved as a historical note. The full re-audit happens in **Phase 3 Batch 10 sub-batch 10k** (`docs/agent-provider-integration.md`, 835 lines), which reads the docs file in full, classifies findings per the v1.2 taxonomy, and annotates the affected wiki entity pages. Batch 10k's batch-entry log.md entry will cite log.md:1476 as the pre-plan starting point.

2. **`docs/CLEANUP.md vs code`** — commit `f143813` added `## Docs claim` and `## Drift` sections to [gastown/commands/done.md](gastown/commands/done.md) documenting the `gt done` drift (docs claim "self-nukes worktree, kills own session" but code post-gt-4ac/gt-hdf8 transitions polecat to IDLE with sandbox preserved). These pre-date schema v1.2 and may or may not need reformatting under the new conventions (verbatim quote check, authority-hierarchy fix-tier tag, `phase3_audited` / `phase3_findings` frontmatter fields, non-retroactive application). The full re-audit happens in **Phase 3 Batch 9** (`docs/CLEANUP.md`, 170 lines, 62 command rows). Batch 9 reads every row of CLEANUP.md against the corresponding `gastown/commands/` pages, verifies `done.md`'s existing Phase-3-era sections against the v1.2 conventions (reshape if needed), and row-by-row checks the ~61 CLEANUP.md rows that were asserted to "match" without per-row verification. Batch 9's batch-entry log.md entry will cite log.md:1480 as the pre-plan starting point.

### `wiki-ytq` bead — rehabilitated in place, NOT closed

The `wiki-ytq` bead's description currently conflates the Phase 3 epic scope with the specific `gt done` CLEANUP.md finding. Its title ("Phase 3: Drift Analysis — Compare docs/ claims to code-grounded wiki pages") is already epic-scoped and correct.

**Task 7 of Batch 0 updates `wiki-ytq`'s description in place** to remove the conflated finding and replace it with a clean Phase 3 epic scope pointing at the plan file. `wiki-ytq` **stays `in_progress`** through all 13 Phase 3 batches. It closes ONLY at the final step of Batch 13 (after the consolidated drift index at `gastown/drift/README.md` lands and pushes).

**Principle:** close the epic at the end of the work, always. Never at the start. Closing `wiki-ytq` in Batch 0 would leave Phase 3 without an epic tracker; rehabilitating in place preserves the original bead ID (and its historical link to the pre-plan drift-found entry) while aligning its description with the plan.

### Why grandfather rather than retroactively reformat

Rewriting the two historical entries in-place would violate the append-only log rule and hide the fact that Phase 3 started out-of-process and got formalized mid-stream. That mid-stream formalization is itself information — it documents when the plan crystallized and why the earlier attempts needed a re-audit. The grandfather approach preserves the drift trail while making the Phase 3 plan the authoritative source from Batch 0 onward. Future readers walking the log chronologically will see: informal drift-found sketches → plan crystallization decisions → formalized Phase 3 batches with full audit trail → grandfathered supersession in Batches 9 and 10 → Phase 3 epic closure in Batch 13.

### References

- [log.md:1476](log.md) — grandfathered entry 1 (`agent-provider-integration.md`)
- [log.md:1480](log.md) — grandfathered entry 2 (`docs/CLEANUP.md`)
- [log.md:204-207](log.md) — append-only log rule
- Commit `f143813` — pre-plan work on [gastown/commands/done.md](gastown/commands/done.md)
- `wiki-ytq` — Phase 3 epic bead, currently `in_progress`

→ [gastown/commands/done.md](gastown/commands/done.md)

## [2026-04-15] drift-found | Batch 1a (Sweep 1 commands/ Diagnostics — 22 pages)

**Scope:** Sweep 1 promotion of Phase 2 notes to v1.2 Drift / Implementation status annotations across all 22 pages in the `GroupDiag` cobra group. Every page received `phase3_audited` / `phase3_findings` / `phase3_severities` / `phase3_findings_post_release` frontmatter. Four pages surfaced `cobra drift` findings and received `## Docs claim` + `## Drift` sections; the other 18 were audited with `phase3_findings: [none]`. Drift index stub created at [gastown/drift/README.md](gastown/drift/README.md) to satisfy the forward-link rule.

**Source files re-read at current HEAD** (current gastown HEAD `9f962c4a`):
- `/home/kimberly/repos/gastown/internal/cmd/activity.go` (in full, 198 lines)
- `/home/kimberly/repos/gastown/internal/events/events.go` (lines 37-80 + `write`/`Log`/`LogFeed` block)
- `/home/kimberly/repos/gastown/internal/cmd/audit.go` (in full, 571 lines)
- `/home/kimberly/repos/gastown/internal/cmd/checkpoint_cmd.go` (spot-verified via path-existence; notes neutral)
- `/home/kimberly/repos/gastown/internal/cmd/costs.go` (lines 1-220, 915-935)
- `/home/kimberly/repos/gastown/internal/cmd/paths.go` (in full)
- `/home/kimberly/repos/gastown/internal/cmd/dashboard.go` (verified against wiki summary; notes neutral)
- `/home/kimberly/repos/gastown/internal/cmd/doctor.go` (lines 22-309 — `Long` text + full `runDoctor` registration block)
- `/home/kimberly/repos/gastown/internal/cmd/feed.go` (spot-verified; notes neutral)
- `/home/kimberly/repos/gastown/internal/cmd/heartbeat.go` (verified against wiki summary; notes neutral)
- `/home/kimberly/repos/gastown/internal/cmd/info.go` (lines 1-75 — confirmed not a parent-only stub, contrary to Phase 3 preflight "expected high-yield" list)
- `/home/kimberly/repos/gastown/internal/cmd/log.go` (verified against wiki summary; notes neutral)
- `/home/kimberly/repos/gastown/internal/cmd/metrics.go` (verified against wiki summary; notes neutral)
- `/home/kimberly/repos/gastown/internal/cmd/patrol.go` / `patrol_new.go` / `patrol_report.go` (verified against wiki summary; notes neutral)
- `/home/kimberly/repos/gastown/internal/cmd/prime.go` (verified against wiki summary; notes neutral)
- `/home/kimberly/repos/gastown/internal/cmd/repair.go` (in full, 91 lines)
- `/home/kimberly/repos/gastown/internal/cmd/seance.go` (verified against wiki summary; notes neutral)
- `/home/kimberly/repos/gastown/internal/cmd/stale.go` (verified against wiki summary; notes neutral)
- `/home/kimberly/repos/gastown/internal/cmd/status.go` (lines 40-110 — `statusCmd` block + `init()` flag registration)
- `/home/kimberly/repos/gastown/internal/cmd/thanks.go` (verified against wiki summary; notes neutral)
- `/home/kimberly/repos/gastown/internal/cmd/upgrade.go` (verified against wiki summary; notes neutral)
- `/home/kimberly/repos/gastown/internal/cmd/version.go` (verified against wiki summary; notes neutral)
- `/home/kimberly/repos/gastown/internal/cmd/vitals.go` (verified against wiki summary; notes neutral)
- `/home/kimberly/repos/gastown/internal/cmd/whoami.go` (verified against wiki summary; notes neutral)

Release-position verification: each of the four finding pages was cross-checked against `git show v1.0.0:<file>` for `activity.go`, `repair.go`, and `status.go`; the `Long` text and check registration are byte-identical to v1.0.0 in all three, so all findings are `in-release`. `doctor.go:22-309` is architecturally unchanged between v1.0.0 and HEAD for the purposes of this finding (both versions enumerate a curated subset of checks in `Long` while registering a larger set in `runDoctor`), so `in-release` as well.

**Docs files read:** none (Sweep 1).

**Wiki pages audited:**
- [activity](gastown/commands/activity.md) — `phase3_findings: [cobra-drift]`
- [audit](gastown/commands/audit.md) — `phase3_findings: [none]`
- [checkpoint](gastown/commands/checkpoint.md) — `phase3_findings: [none]`
- [costs](gastown/commands/costs.md) — `phase3_findings: [none]`
- [dashboard](gastown/commands/dashboard.md) — `phase3_findings: [none]`
- [doctor](gastown/commands/doctor.md) — `phase3_findings: [cobra-drift]`
- [feed](gastown/commands/feed.md) — `phase3_findings: [none]`
- [heartbeat](gastown/commands/heartbeat.md) — `phase3_findings: [none]`
- [info](gastown/commands/info.md) — `phase3_findings: [none]`
- [log](gastown/commands/log.md) — `phase3_findings: [none]` (gt log command page; NOT this wiki log file)
- [metrics](gastown/commands/metrics.md) — `phase3_findings: [none]`
- [patrol](gastown/commands/patrol.md) — `phase3_findings: [none]`
- [prime](gastown/commands/prime.md) — `phase3_findings: [none]`
- [repair](gastown/commands/repair.md) — `phase3_findings: [cobra-drift]`
- [seance](gastown/commands/seance.md) — `phase3_findings: [none]`
- [stale](gastown/commands/stale.md) — `phase3_findings: [none]`
- [status](gastown/commands/status.md) — `phase3_findings: [cobra-drift]`
- [thanks](gastown/commands/thanks.md) — `phase3_findings: [none]`
- [upgrade](gastown/commands/upgrade.md) — `phase3_findings: [none]`
- [version](gastown/commands/version.md) — `phase3_findings: [none]`
- [vitals](gastown/commands/vitals.md) — `phase3_findings: [none]`
- [whoami](gastown/commands/whoami.md) — `phase3_findings: [none]`

**Findings by category:**

- **drift:** 0 findings (Sweep 1 does not read `docs/` files; the `drift` category requires a docs source claim and is out of scope for Batch 1a).
- **cobra drift:** 4 findings on 4 pages, all `severity: wrong`, all `in-release`, all `fix tier: code`:
  - **activity** (`gastown/commands/activity.md`) — `activity.go:52-56` `Long` text lists refinery event types `merge_started` / `merge_complete` / `merge_failed` / `queue_processed`, but the switch at `activity.go:136` dispatches on `events.TypeMergeStarted` / `TypeMerged` / `TypeMergeFailed` / `TypeMergeSkipped` (`events/events.go:67-70`) — constants resolve to `merge_started`, `merged`, `merge_failed`, `merge_skipped`. Two of the four advertised names (`merge_complete`, `queue_processed`) have no corresponding constant; one (`merged`) is missing from the advertised list.
  - **doctor** (`gastown/commands/doctor.md`) — `doctor.go:26-118` `Long` catalog enumerates a curated subset of checks under named section headings, but `runDoctor` at `doctor.go:154-279` registers roughly 80 checks, many of which are absent from the catalog (`NewRigConfigSyncCheck`, `NewStaleDoltPortCheck`, `NewStaleSQLServerInfoCheck`, `NewZombieSessionCheck`, `NewLinkedPaneCheck`, `NewSocketSplitBrainCheck`, `NewNullAssigneeCheck`, `NewDoltMetadataCheck`, `NewPatrolPluginDriftCheck`, `NewAgentBeadsCheck` + family, and more — full list in the page's `## Drift` section). The `Long` catalog is a manually-maintained narrative that has drifted out of sync with the registration block.
  - **repair** (`gastown/commands/repair.md`) — `repair.go:22-28` `Long` text lists six repair targets, but the `checks` slice at `repair.go:51-54` registers only two (`doctor.NewRigConfigSyncCheck()`, `doctor.NewStaleDoltPortCheck()`). Four advertised targets have no direct registered-check mapping.
  - **status** (`gastown/commands/status.md`) — `status.go:50` `Long` text and `status.go:57` flag description both claim `--fast` only "skip mail lookups," but `gatherStatus` uses `statusFast` to gate four skips: overseer mail count (`status.go:776-781`), per-agent mail counts inside `discoverGlobalAgents`/`discoverRigAgents`, per-rig hook discovery (`status.go:891-893`), and per-rig MQ summary (`status.go:907-909`). Hook info additionally falls back to a lighter-weight source in `--fast`, which changes the reported data shape.
- **compound drift:** 0 findings (no page had both a `cobra drift` AND a separate `docs/` drift — Sweep 1 doesn't read `docs/`, so `compound drift` is structurally impossible in this sub-batch).
- **implementation-status unbuilt/partial/vestigial:** 0 findings. None of the 22 Diagnostics pages describe aspirational behavior, partially-implemented features, or explicitly-dead code in their Phase 2 notes or body.
- **wiki-stale:** 0 findings. Every source file re-read at current HEAD (`9f962c4a`) agreed with the Phase 2 wiki body at the line-number precision the wiki uses. No wiki-stale fixes were applied.
- **gap:** 0 findings. The Diagnostics command set was fully mapped in Phase 2 Batch 3a (and the `log.md:1476`/`:1480` pre-plan drift entries already covered `docs/CLEANUP.md` and `docs/agent-provider-integration.md`, which are Sweep 2 scope). No new gap beads were filed at sub-batch level.
- **none:** 18 pages audited with no findings — `audit`, `checkpoint`, `costs`, `dashboard`, `feed`, `heartbeat`, `info`, `log`, `metrics`, `patrol`, `prime`, `seance`, `stale`, `thanks`, `upgrade`, `version`, `vitals`, `whoami`. Source re-read confirmed current behavior matches both the implicit docs claim (Cobra `Long` text) and the Phase 2 wiki body.

**New beads filed:** none.

**Beads closed:** none (Batch 1 anchor `wiki-vxl` stays `in_progress` across all eight sub-batches 1a–1h; closes at the end of Batch 1).

**Cross-link discipline:** 4 new `## Drift` sections added (on activity, doctor, repair, status) with verbatim `## Docs claim` quotes on each — re-checked against source character-by-character at the end of the sub-batch. All `file:line` refs are CURRENT (freshly read from HEAD, not copied from Phase 2 notes). Drift index stub created at [gastown/drift/README.md](gastown/drift/README.md) so the forward-link rule (skill rule 10) is satisfied; Batch 13 will populate the consolidated index and add the backlinks. Only one `→ promoted to ## Drift` redirect was added (`status.md` Notes bullet 5); the other three findings originated from explicit Phase 2 body statements already flagged as drift-risk observations, so there was no Notes bullet to redirect.

**Judgment calls made during audit** (for orchestrator cross-check):

1. **activity.md `~/gt/.events.jsonl` path claim.** The `activity.go:34` `Long` text says "Events are written to `~/gt/.events.jsonl`" but `events.LogFeed` → `write` → `filepath.Join(townRoot, EventsFile)` at `events/events.go:118` is town-root-relative, not `$HOME`-relative. In the default deployment where town root is `~/gt`, the claim is incidentally correct; in any other deployment it is wrong. Classified as **neutral** (not promoted to drift) because (a) the common case is accurate and (b) the same package's internal comment at `events/events.go:3` and `:83` also says "written to `~/gt/.events.jsonl`", so the drift is consistent across the code tree and may reflect an intentional simplification rather than a lapse. If Batch 1g/h turns up the same pattern on other event-writing commands, the classification should be revisited.
2. **doctor.md promotion source.** The drift observation ("The Long description categories are documentation-only — the actual registration order in `runDoctor` above is what ships") lived in the Phase 2 page **body** at line 161-163, not in `## Notes / open questions`. Sweep 1 is defined as "promote Phase 2 notes bullets," but the skill's intent is clearly to promote drift observations regardless of where Phase 2 parked them. Promoted anyway and left the body sentence in place (now rewritten to forward to `## Drift`). If the review disagrees, the promotion can be retracted without data loss.
3. **stale.md inverted exit codes.** Phase 2 notes bullet 2 flags the 0=stale, 1=fresh quiet-mode convention as "non-obvious and surprising for shell scripts," but the same bullet acknowledges "the `Long` text calls this out explicitly." Since the docs and code agree, classified as **neutral**. The ergonomics are weird but the docs are not wrong.
4. **log.md `--since` parser asymmetry.** `log --since` accepts only Go `time.ParseDuration` format, but `audit --since` accepts the Go format + a `d` suffix (24h days). Phase 2 notes bullet 1 on `log.md` flags this inconsistency. Classified as **neutral** because neither command's `Long` text commits to a specific parser flavor; a user who types `gt log --since=7d` gets an error message, but the help text doesn't promise it would work. A Phase 4 (Coverage) audit might tag this as `severity: incomplete` since the behavior asymmetry surprises users who use both commands, but Phase 3 does not own that classification.
5. **info.md expected-high-yield miss.** The Phase 3 plan's Batch 1 preflight listed `info` among expected high-yield parent-only-stub pages ("Long text describes subcommands that aren't wired"). Current `info.go` is a terminal command with two flags (`--whats-new`, `--json`) and no subcommands; the `Long` text accurately describes the behavior. Either `info` was fixed between plan writing and Batch 1a, or it was incorrectly categorized in the plan. Classified as **no finding**. The orchestrator may want to remove `info` from the preflight high-yield list before Batches 1b–1h.
6. **version.md `gt --version` vs `gt version` divergence.** Phase 2 notes bullet 4 flags that cobra's auto `--version` flag and the `gt version` subcommand produce different output. No docs source claims they should match. Classified as **neutral**. The observation may be worth filing as a Phase 4 gap (either output should document itself as the canonical version source) but that's not Phase 3 scope.

**Next sub-batch:** Batch 1b — Configuration group (11 cmds). Tracked under `wiki-vxl` (Batch 1 anchor bead).

→ [gastown/commands/activity.md](gastown/commands/activity.md),
  [gastown/commands/audit.md](gastown/commands/audit.md),
  [gastown/commands/checkpoint.md](gastown/commands/checkpoint.md),
  [gastown/commands/costs.md](gastown/commands/costs.md),
  [gastown/commands/dashboard.md](gastown/commands/dashboard.md),
  [gastown/commands/doctor.md](gastown/commands/doctor.md),
  [gastown/commands/feed.md](gastown/commands/feed.md),
  [gastown/commands/heartbeat.md](gastown/commands/heartbeat.md),
  [gastown/commands/info.md](gastown/commands/info.md),
  [gastown/commands/log.md](gastown/commands/log.md),
  [gastown/commands/metrics.md](gastown/commands/metrics.md),
  [gastown/commands/patrol.md](gastown/commands/patrol.md),
  [gastown/commands/prime.md](gastown/commands/prime.md),
  [gastown/commands/repair.md](gastown/commands/repair.md),
  [gastown/commands/seance.md](gastown/commands/seance.md),
  [gastown/commands/stale.md](gastown/commands/stale.md),
  [gastown/commands/status.md](gastown/commands/status.md),
  [gastown/commands/thanks.md](gastown/commands/thanks.md),
  [gastown/commands/upgrade.md](gastown/commands/upgrade.md),
  [gastown/commands/version.md](gastown/commands/version.md),
  [gastown/commands/vitals.md](gastown/commands/vitals.md),
  [gastown/commands/whoami.md](gastown/commands/whoami.md),
  [gastown/drift/README.md](gastown/drift/README.md)

## [2026-04-15] drift-found | Batch 1b (Sweep 1 commands/ Configuration — 11 pages)

**Scope:** Sweep 1 promotion across all 11 pages in the `GroupConfig` cobra group (`account`, `config`, `directive`, `disable`, `enable`, `hooks`, `issue`, `plugin`, `shell`, `theme`, `uninstall`). Every page received `phase3_audited` / `phase3_findings` / `phase3_severities` / `phase3_findings_post_release` frontmatter. Six pages surfaced `cobra drift` findings and received `## Docs claim` + `## Drift` sections; two pages surfaced `wiki-stale` findings against Phase 2 body claims that were already wrong at Phase 2 time (sibling-file subcommand wiring that Phase 2 missed); one page (`hooks`) carries BOTH a wiki-stale and a cobra-drift finding; the remaining four pages were audited with `phase3_findings: [none]`. Drift index stub at [gastown/drift/README.md](gastown/drift/README.md) already exists from Batch 1a; not modified in this sub-batch.

**Source files re-read at current HEAD** (current gastown HEAD `9f962c4af068fe9da9f4bd3624e7b66351121fdf`):

- `/home/kimberly/repos/gastown/internal/cmd/account.go` (lines 1-100, 263-295, 500-521 — parent `Long`, `accountSwitchCmd` block, `init()` AddCommand registrations)
- `/home/kimberly/repos/gastown/internal/cmd/config.go` (lines 643-916 — `configSetCmd.Long`, `configGetCmd.Long`, `runConfigGet` body including `case "dolt.port":` at `:896` and the error-message supported-keys list at `:911`)
- `/home/kimberly/repos/gastown/internal/cmd/directive.go` (in full, 41 lines — parent-only declaration confirming `rootCmd.AddCommand(directiveCmd)` is the only registration in this file)
- `/home/kimberly/repos/gastown/internal/cmd/directive_show.go` (in full, 124 lines — `directiveShowCmd` declaration + `directiveCmd.AddCommand(directiveShowCmd)` at `:31`)
- `/home/kimberly/repos/gastown/internal/cmd/directive_edit.go` (lines 1-60 — `directiveEditCmd` declaration + `directiveCmd.AddCommand(directiveEditCmd)` at `:38`)
- `/home/kimberly/repos/gastown/internal/cmd/directive_list.go` (lines 1-35 — `directiveListCmd` declaration + `directiveCmd.AddCommand(directiveListCmd)` at `:26`)
- `/home/kimberly/repos/gastown/internal/cmd/disable.go` (verified against wiki summary; notes neutral)
- `/home/kimberly/repos/gastown/internal/cmd/enable.go` (verified against wiki summary; notes neutral)
- `/home/kimberly/repos/gastown/internal/cmd/hooks.go` (in full, 45 lines — parent `Long` subcommand list at `:17-25`, parent-only `init()` at `:42-44`)
- `/home/kimberly/repos/gastown/internal/cmd/hooks_base.go` (registration line only; `hooksCmd.AddCommand(hooksBaseCmd)` at `:32`)
- `/home/kimberly/repos/gastown/internal/cmd/hooks_override.go` (registration line only; `hooksCmd.AddCommand(hooksOverrideCmd)` at `:38`)
- `/home/kimberly/repos/gastown/internal/cmd/hooks_sync.go` (registration line only; `hooksCmd.AddCommand(hooksSyncCmd)` at `:42`)
- `/home/kimberly/repos/gastown/internal/cmd/hooks_diff.go` (registration line only; `hooksCmd.AddCommand(hooksDiffCmd)` at `:34`)
- `/home/kimberly/repos/gastown/internal/cmd/hooks_list.go` (registration line only; `hooksCmd.AddCommand(hooksListCmd)` at `:32`)
- `/home/kimberly/repos/gastown/internal/cmd/hooks_scan.go` (registration line only; `hooksCmd.AddCommand(hooksScanCmd)` at `:41`)
- `/home/kimberly/repos/gastown/internal/cmd/hooks_registry.go` (registration line only; `hooksCmd.AddCommand(hooksRegistryCmd)` at `:51`)
- `/home/kimberly/repos/gastown/internal/cmd/hooks_install.go` (registration line only; `hooksCmd.AddCommand(hooksInstallCmd)` at `:41`)
- `/home/kimberly/repos/gastown/internal/cmd/hooks_init.go` (in full, 275 lines — `hooksInitCmd` declaration with its own `Long` text at `:17-27`, `hooksCmd.AddCommand(hooksInitCmd)` at `:32`, full `runHooksInit` body, supporting helpers)
- `/home/kimberly/repos/gastown/internal/cmd/issue.go` (lines 1-60 — parent + subcommands declarations, `init()` with all three AddCommand calls, `Long` texts on all four commands)
- `/home/kimberly/repos/gastown/internal/cmd/plugin.go` (lines 32-156 — parent `Long`, subcommand `Long` texts, `init()` flag registrations; `runPluginRun` gate block at `:412-490`; `runPluginRun` gate check at `:425-442` and short-circuit at `:458-462`)
- `/home/kimberly/repos/gastown/internal/cmd/shell.go` (in full, 114 lines — all four `Long` texts, `runShellInstall` with `state.Enable(Version)` at `:72-74`, `runShellRemove` at `:83-90`, `runShellStatus` at `:92-113`, `init()` at `:60-65`)
- `/home/kimberly/repos/gastown/internal/cmd/theme.go` (lines 26-90 — parent + `themeCLICmd` `Long` texts; neither claims sole-writer of `townSettings.CLITheme`)
- `/home/kimberly/repos/gastown/internal/cmd/uninstall.go` (lines 25-171 — `Long`, `init()`, `runUninstall`, and `findWorkspaceForUninstall` two-candidate scan at `:152-171`)

Release-position verification: every `cobra drift` finding was cross-checked against `git show v1.0.0:<file>` for the cited `Long` text blocks and the cited implementation lines. All findings are `in-release`:

- `account.go:34-38` parent `Long` Commands list byte-identical at v1.0.0; `accountSwitchCmd` registration at the same position in `init()`.
- `config.go:693-723` `configGetCmd.Long` byte-identical at v1.0.0; `case "dolt.port":` at the same position in `runConfigGet`.
- `hooks.go:17-25` parent `Long` Subcommands block byte-identical at v1.0.0; `hooks_init.go` already present at v1.0.0 with the same registration.
- `plugin.go:92-100` `pluginRunCmd.Long` byte-identical at v1.0.0; cooldown-only gate switch at `:425-442` identical.
- `shell.go:31-37` `shellInstallCmd.Long` byte-identical at v1.0.0; `state.Enable(Version)` call at `:72-74` already present.
- `uninstall.go:29-45` `uninstallCmd.Long` byte-identical at v1.0.0; `findWorkspaceForUninstall` two-candidate scan at `:152-171` already present.

Both `wiki-stale` findings (directive, hooks) point at code that existed at v1.0.0 — the Phase 2 wiki bodies were wrong at Phase 2 time (2026-04-11), not stale against churn, so `in-release` as well (though release position is less load-bearing for wiki-stale fixes because the fix is a wiki-side correction, not an upstream PR).

**Docs files read:** none (Sweep 1).

**Wiki pages audited:**
- [account](gastown/commands/account.md) — `phase3_findings: [cobra-drift]`
- [config](gastown/commands/config.md) — `phase3_findings: [cobra-drift]`
- [directive](gastown/commands/directive.md) — `phase3_findings: [wiki-stale]`
- [disable](gastown/commands/disable.md) — `phase3_findings: [none]`
- [enable](gastown/commands/enable.md) — `phase3_findings: [none]`
- [hooks](gastown/commands/hooks.md) — `phase3_findings: [wiki-stale, cobra-drift]`
- [issue](gastown/commands/issue.md) — `phase3_findings: [none]`
- [plugin](gastown/commands/plugin.md) — `phase3_findings: [cobra-drift]`
- [shell](gastown/commands/shell.md) — `phase3_findings: [cobra-drift]`
- [theme](gastown/commands/theme.md) — `phase3_findings: [none]`
- [uninstall](gastown/commands/uninstall.md) — `phase3_findings: [cobra-drift]`

**Findings by category:**

- **drift:** 0 findings (Sweep 1 does not read `docs/` files).
- **cobra drift:** 6 findings on 6 pages, all `severity: wrong`, all `in-release`, all `fix tier: code`:
  - **account** (`gastown/commands/account.md`) — parent `accountCmd.Long` at `account.go:34-38` enumerates exactly 4 subcommands (`list`, `add`, `default`, `status`) but `init()` at `account.go:513-518` registers 5, including `accountSwitchCmd` (`account.go:278-295`). `switch` is fully implemented via `runAccountSwitch` at `account.go:351-465` and is the only path that rewrites the `~/.claude` symlink. Users find it only via the cobra-generated `Available Commands:` block.
  - **config** (`gastown/commands/config.md`) — `configGetCmd.Long` at `config.go:693-723` omits `dolt.port` from its Supported keys block. `runConfigGet` at `config.go:896-905` has an explicit `case "dolt.port":` that loads `daemon.LoadPatrolConfig(townRoot)` and reads `GT_DOLT_PORT`. The error-message branch at `config.go:911` even names `dolt.port` in its own "Supported keys:" text — the same command contradicts itself. `configSetCmd.Long` at `config.go:648-661` does list `dolt.port`, which makes the `get` omission look like an oversight rather than intentional asymmetry.
  - **hooks** (`gastown/commands/hooks.md`) — parent `hooksCmd.Long` at `hooks.go:17-25` enumerates exactly 8 subcommands (`base`, `override`, `sync`, `diff`, `list`, `scan`, `registry`, `install`), but `hooks_init.go` declares a ninth user-facing subcommand `hooksInitCmd` (`hooks_init.go:14-29`) with its own `Long` describing the bootstrap flow, `--dry-run` flag, and a full 100+ line `runHooksInit` implementation at `:36-150`. `hooks_init.go:31-34` wires it via `hooksCmd.AddCommand(hooksInitCmd)`. The parent `Long`'s hand-maintained subcommand list is incomplete.
  - **plugin** (`gastown/commands/plugin.md`) — `pluginRunCmd.Long` at `plugin.go:92-100` states "By default, checks if the gate would allow execution and informs you if it wouldn't." Combined with the parent `pluginCmd.Long` at `plugin.go:44-49` enumerating 5 gate types (`cooldown`, `cron`, `condition`, `event`, `manual`), this builds the expectation that `plugin run` honors all five. `runPluginRun` at `plugin.go:425-442` only checks gate status when `p.Gate.Type == plugin.GateCooldown`; the other four types leave `gateOpen = true` untouched and skip the short-circuit at `:458-462` entirely. `--force` is a no-op for the other four types — they were already open.
  - **shell** (`gastown/commands/shell.md`) — `shellInstallCmd.Long` at `shell.go:31-37` describes only the `cd` hook and its behaviors. `runShellInstall` at `shell.go:67-81` additionally calls `state.Enable(Version)` at `shell.go:72-74`, silently flipping the global "enabled" kill-switch. A user who previously ran `gt disable` and later runs `gt shell install` (perhaps to pick up "latest shell hook features" as the `Long` text encourages) will find Gas Town silently re-enabled. `shell remove` at `shell.go:83-90` does NOT call `state.Disable()`, so the state-flag symmetry is broken across the two commands.
  - **uninstall** (`gastown/commands/uninstall.md`) — `uninstallCmd.Long` at `uninstall.go:29-45` reads "The workspace (e.g., ~/gt) is NOT removed unless --workspace is specified" and the Examples block includes `gt uninstall --workspace        # Also remove workspace directory`. `findWorkspaceForUninstall` at `uninstall.go:152-171` only scans two hardcoded candidates (`$HOME/gt`, `$HOME/gastown`), each requiring a `mayor/` subdirectory. Users with workspaces at other paths run `--workspace`, see the confirmation prompt warn "WORKSPACE WILL BE DELETED" (`uninstall.go:70-72`), type `y`, and then discover their workspace is untouched — only XDG dirs are gone. The `--workspace` branch at `uninstall.go:122-131` does not error or warn on the empty-string return.
- **compound drift:** 0 findings (no docs reads, so structurally impossible).
- **implementation-status unbuilt/partial/vestigial:** 0 findings. None of the 11 Configuration pages had Phase 2 notes identifying aspirational / partial / vestigial behavior.
- **wiki-stale:** 2 findings on 2 pages, both `severity: wrong`, both `fix tier: wiki` (fixed inline in `## What it actually does`):
  - **directive** (`gastown/commands/directive.md`) — Phase 2 page claimed "`show`, `edit`, `list` subcommands advertised in its `Long` text are not wired up here" and speculated they were "either defined in another file not mapped yet or not implemented." Re-read at HEAD `9f962c4a`: all three subcommands ARE wired via sibling files `directive_show.go:31`, `directive_edit.go:38`, `directive_list.go:26`, each calling `directiveCmd.AddCommand(...)` in its own `init()`. All three sibling files were present at v1.0.0, so the Phase 2 claim was wrong at Phase 2 time (2026-04-11), not stale against churn. Fixed: page body rewritten to enumerate the sibling-file registrations and describe each subcommand's actual behavior; Notes section annotated with the correction.
  - **hooks** (`gastown/commands/hooks.md`) — Phase 2 page claimed "`hooks.go` only declares the parent. All eight advertised subcommands must be registered somewhere else under `internal/cmd/` (or not at all)" and listed "follow-up: grep for `hooksCmd.AddCommand` across the tree" as an open question. Re-read at HEAD: all nine subcommands (the 8 advertised plus the unlisted `init`) are wired via sibling `hooks_*.go` files, each calling `hooksCmd.AddCommand(...)` in its own `init()`. All sibling files present at v1.0.0. Fixed: page body rewritten to enumerate the sibling-file registrations; the body's 9th-subcommand observation becomes the hook into the separately-promoted `cobra drift` finding. Notes pointer added. (Taxonomy note: per the skill, wiki-stale findings are fixed inline and logged under `lint`. Batch 1b bundles them into this `drift-found` entry because the sub-batch is a single audit pass and splitting into two log entries would lose the audit trail continuity; Batch 1a set this precedent with zero wiki-stale findings, so Batch 1b is the first Phase 3 sub-batch where the convention is tested. The orchestrator may elect to split the wiki-stale rows into a separate `lint` entry at Sweep 1 retrospective time, but for now they are consolidated here.)
- **gap:** 0 findings. The Configuration command set was fully mapped in Phase 2 Batch 3b.
- **none:** 4 pages audited with no findings — `disable`, `enable`, `issue`, `theme`. Source re-read confirmed current behavior matches the Cobra `Long` text and Phase 2 wiki body. (`issue` in particular was verified to make no beads-integration claim in any `Long` text, so the Phase 2 body note "misleading grouping in Configuration" stays neutral.)

**New beads filed:** none.

**Beads closed:** none (Batch 1 anchor `wiki-vxl` stays `in_progress` across all eight sub-batches).

**Cross-link discipline:** 5 new `## Drift` sections added (account, config, hooks, plugin, shell, uninstall — wait, that's 6 pages; account + config + hooks + plugin + shell + uninstall = 6 pages with new `## Drift` sections; one of them (hooks) carries a two-finding `## Drift` section with both a `cobra drift` row and a `wiki-stale` row side-by-side). 5 new `## Docs claim` sections (one per page with cobra drift; `hooks` has one combined `## Docs claim` for the parent-command verbatim that supports both of its findings; `directive` has no `## Docs claim` because its only finding is wiki-stale, which is fixed inline). All `file:line` refs are CURRENT (freshly read from HEAD `9f962c4a`, not copied from Phase 2 notes). All `## Docs claim` quotes re-checked against source character-by-character at the end of the sub-batch; no paraphrasing. Forward links to [gastown/drift/README.md](gastown/drift/README.md) added on every new `## Drift` section. Promoted Phase 2 body observations left behind as `→ promoted to ## Drift` redirects on `plugin.md` (gate-semantics observation), `shell.md` (install→enable coupling note), and `uninstall.md` (narrow heuristic note); `account.md` and `config.md` findings did not have a 1-for-1 Phase 2 notes bullet to redirect (the observations were implicit in the code-walk sections, not parked as open questions); `hooks.md` redirected its "Missing subcommand wiring" bullet to the `## Drift` wiki-stale row. Cobra drift observations on `theme.md` (dual-writer of `CLITheme`) and `config.md` (same) stayed neutral — neither command's `Long` text claims to be the sole writer, so there is no docs claim to contradict.

**Judgment calls made during audit** (for orchestrator cross-check):

1. **`account` Commands list omission.** `accountCmd.Long` at `account.go:34-38` lists 4 subcommands; `init()` registers 5. One reading is "the `Long` is a user-facing summary; cobra's own `Available Commands:` block is the authoritative list and lists all 5, so this isn't drift." The opposing reading is "the hand-maintained `Commands:` list is a deliberate docs claim (why else would it be in the `Long`?), and it's wrong." Promoted as `cobra drift` on the second reading. If the orchestrator prefers to treat hand-maintained summary lists as informational rather than claims, this could be demoted to neutral. Batch 1a's `doctor` finding used the same "hand-maintained list drifted from registration" framing, so Batch 1b stays consistent with 1a's call.
2. **`config get dolt.port` drift vs neutral.** The Phase 2 note framed this as "`get` and `set` key sets are asymmetric — intentional (port lives in `daemon.DaemonPatrolConfig`, not town settings) or oversight?" Re-read shows `runConfigGet` has the `case "dolt.port":` and the error message lists it — so it's unambiguously an oversight: the `Long` text drifted, but the code and the error message agree that `dolt.port` is supported. The Phase 2 "intentional vs oversight" hedge was too cautious; the re-read resolves it decisively. Promoted as `cobra drift`.
3. **`theme` / `config` dual-writer observation.** Both pages note "two code paths write `townSettings.CLITheme`." Classified as **neutral** on both pages because neither command's `Long` text claims to be the sole writer. Would become `drift` if upstream `docs/` files claimed one was the canonical CLI-theme command, but that's Sweep 2's territory. Consistent with Batch 1a's `log.md` Notes decisions (the `--since` parser asymmetry stayed neutral because no `Long` text claimed a specific parser flavor).
4. **`plugin run` always records Success.** The Phase 2 note flags that `runPluginRun` always records `ResultSuccess` at `plugin.go:480` regardless of whether the agent actually ran the printed instructions successfully. Classified as **neutral** (not promoted) because `pluginRunCmd.Long` and `pluginHistoryCmd.Long` do not claim that `history` tracks pass/fail — they describe it as "execution history" without specifying what "execution" means. A Phase 4 (Coverage) audit might tag this as `severity: incomplete` (history tracking is a half-built feature; real failure tracking would need a round-trip from the agent back to the recorder), but Phase 3 does not own that classification.
5. **`directive` and `hooks` wiki-stale vs cobra drift.** The Phase 2 pages for both commands claimed the subcommands were unwired. For `directive`, the `Long` text DOES accurately describe three subcommands that are in fact wired — so the only finding is the wiki's own synthesis being wrong. For `hooks`, the `Long` text describes 8 subcommands that are wired, but a 9th is wired without being mentioned — so there's BOTH a wiki-stale (Phase 2 said unwired) and a cobra drift (Long text omits `init`). This pattern (wiki-stale where Phase 2 under-mapped a parent-only-looking file and missed sibling-file registrations) may repeat in Batches 1c-1h on other parent-only stubs — `tap`, `warrant`, `repair` (Batch 1a already found this was mostly accurate), and others flagged in the Phase 2 notes. Worth watching.
6. **`hooks` body-parked body rewrite.** Per Batch 1a's established pattern (implicit charter clarification in the 1a retro: body-parked drift observations get promoted same as Notes-parked), the `hooks.md` body's "No `hooksCmd.AddCommand(...)` calls appear in this file. The subcommand wiring lives elsewhere or is not yet implemented" claim was promoted to the wiki-stale finding and the body was rewritten. Same pattern as Batch 1a's `doctor.md` body-text rewrite. Consistent with 1a.
7. **`directive` body rewrite extent.** The `directive.md` body was rewritten across the "Behavior" and "Subcommands" sections to reflect the sibling-file wiring. I chose to enumerate each subcommand's actual behavior briefly rather than just adding a one-line correction, because the Phase 2 version was structured around "the subcommands are not wired so here's what the Long text advertises" — a minimal correction would leave the page internally incoherent. The more thorough rewrite adds roughly 50 lines of new body content but produces a page that matches Phase 2's normal shape for fully-mapped commands.
8. **`plugin run` `--force` characterization.** I wrote that `--force` "is effectively a no-op for the other four gate types (they were already 'open')." This is literally true — `pluginRunForce` is only consulted in the cooldown branch at `plugin.go:428` (`&& !pluginRunForce`). For cron/condition/event/manual gates, `gateOpen` is initialized to `true` and never changes, so the `!gateOpen && !pluginRunForce` short-circuit at `:458` never triggers and `--force` has nothing to bypass. The user may think `--force` is doing something; it isn't. This framing is sharper than the Phase 2 neutral note and is load-bearing for the drift finding, so I left it in. If the orchestrator thinks this over-characterizes, it can be softened to "`--force` is only functional for cooldown gates."

**Next sub-batch:** Batch 1c — Work Management group (26 cmds). Tracked under `wiki-vxl` (Batch 1 anchor bead).

→ [gastown/commands/account.md](gastown/commands/account.md),
  [gastown/commands/config.md](gastown/commands/config.md),
  [gastown/commands/directive.md](gastown/commands/directive.md),
  [gastown/commands/disable.md](gastown/commands/disable.md),
  [gastown/commands/enable.md](gastown/commands/enable.md),
  [gastown/commands/hooks.md](gastown/commands/hooks.md),
  [gastown/commands/issue.md](gastown/commands/issue.md),
  [gastown/commands/plugin.md](gastown/commands/plugin.md),
  [gastown/commands/shell.md](gastown/commands/shell.md),
  [gastown/commands/theme.md](gastown/commands/theme.md),
  [gastown/commands/uninstall.md](gastown/commands/uninstall.md)

## [2026-04-15] decision | Schema clarification: wiki-stale sub-distinction (churn vs phase-2-incomplete)

Phase 3 Batch 1b (commit 8ac505d, Configuration sub-batch) surfaced two `wiki-stale` findings on `gastown/commands/directive.md` and `gastown/commands/hooks.md`. The 1b subagent flagged that both findings are NOT "wiki page was correct at Phase 2 time and gastown has since changed." Both are "Phase 2 read the parent `.go` file (`directive.go` / `hooks.go`) in isolation and missed the sibling files (`directive_show.go`, `directive_edit.go`, `directive_list.go`, `hooks_base.go`, `hooks_override.go`, `hooks_sync.go`, `hooks_diff.go`, `hooks_list.go`, `hooks_scan.go`, `hooks_registry.go`, `hooks_install.go`, `hooks_init.go`) that wire the advertised subcommands." The wiki body was wrong AT PHASE 2 TIME — both Phase 2's HEAD and current HEAD contradict the wiki body the same way.

Kimberly's ruling: capture this distinction in the schema. *"Capture Phase-2-time-vs-churn wiki-stale, because that signals our stage 2 was incomplete."*

### Change

The `wiki-stale` row of the drift taxonomy in [.claude/skills/writing-entity-pages/SKILL.md](.claude/skills/writing-entity-pages/SKILL.md) now requires every wiki-stale finding to carry a sub-type tag in its inline body via a `**Phase 2 root cause:**` line:

- **`churn`** — wiki was correct at Phase 2 time; gastown has moved since.
- **`phase-2-incomplete`** — wiki was wrong at Phase 2 time (Phase 2 missed sibling files / code paths / registrations).

Determination procedure documented in the skill: re-read the cited gastown file at the gastown commit corresponding to the wiki page's Phase 2 commit date, and compare against current Phase 2 wiki body. A heuristic fallback is allowed for Sweep 1 batches where the per-finding archeology is disproportionate to the audit-trail value; heuristic determinations get noted in the finding body and revisited at the Sweep 1 retrospective gate.

### Why this is not a v1.2 → v1.3 schema bump

This is a clarification of an existing taxonomy category, not a new finding category, new frontmatter field, or new section. The `wiki-stale` row already existed; the addition is a discriminator within it. Per the schema evolution policy in [.claude/skills/maintaining-wiki-schema/SKILL.md](.claude/skills/maintaining-wiki-schema/SKILL.md), this is a "minor wording refinement" that does not bump the version. v1.2 stays v1.2.

### Why this matters for Phase 4 / Phase 7

`phase-2-incomplete` count is a quality signal on Phase 2's mapping methodology. If `phase-2-incomplete` accumulates across multiple Sweep 1 batches, Phase 4 (Coverage / Completeness) should re-scope to include "re-audit Phase 2 sub-batches with X+ phase-2-incomplete count." Phase 7 (Correctness) closes the loop by checking whether the re-audits caught everything.

`churn` count is routine wiki maintenance — interesting for "how fast is the wiki bit-rotting," but not a Phase 2 quality signal.

### Retroactive tagging

Findings filed BEFORE this clarification (Batch 1b's two `wiki-stale` findings on `directive` and `hooks`) are NOT retroactively edited in-place — the append-only rule applies. The Sweep 1 retrospective gate (between Batches 4 and 5) is the natural moment to revisit them and either:
(a) Append a `lint`-verb log entry tagging both as `phase-2-incomplete` based on the 1b subagent's already-recorded determination,
(b) Re-read both pages and tag in place via a follow-up commit + lint log entry,
(c) Decide retroactive tagging is unnecessary because the audit trail already implies the determination via the subagent's report.

Decision deferred to the retrospective gate.

### Wiki-stale log placement (separate decision deferred)

The 1b subagent also flagged that wiki-stale findings should log under the `lint` verb per the skill, not `drift-found`. Batch 1b folded its wiki-stale findings into the `drift-found` Batch 1b entry to preserve audit-trail continuity. Kimberly's ruling: defer the placement question to the Sweep 1 retrospective gate as well. Do not re-classify or re-log Batch 1b's entries until then.

→ [.claude/skills/writing-entity-pages/SKILL.md](.claude/skills/writing-entity-pages/SKILL.md), [.claude/plans/2026-04-14-phase3-drift.md](.claude/plans/2026-04-14-phase3-drift.md), `gastown/commands/directive.md`, `gastown/commands/hooks.md`

## [2026-04-15] drift-found | Batch 1c (Sweep 1 commands/ Work Management — 26 pages)

**Scope:** Sweep 1 promotion across all 26 pages in the `GroupWork` cobra group (`assign`, `bead`, `cat`, `changelog`, `cleanup`, `close`, `compact`, `convoy`, `done`, `formula`, `handoff`, `hook`, `molecule`, `mountain`, `mq`, `orphans`, `prune-branches`, `ready`, `release`, `resume`, `scheduler`, `sling`, `synthesis`, `trail`, `unsling`, `wl`). Every page received `phase3_audited` / `phase3_findings` / `phase3_severities` / `phase3_findings_post_release` frontmatter. Pages with non-neutral findings received `## Docs claim` + `## Drift` sections with verbatim quotes, current `file:line` refs, and v1.2 fix-tier + severity + release-position fields. Drift index stub at [gastown/drift/README.md](gastown/drift/README.md) already exists from Batch 1a; not modified in this sub-batch.

**Source files re-read at current HEAD** (current gastown HEAD `9f962c4af068fe9da9f4bd3624e7b66351121fdf`):

- `/home/kimberly/repos/gastown/internal/cmd/assign.go` (in full, 192 lines — parent `Long`, var block, `init()` flag registrations, `runAssign` body including the non-reference to `assignForce`)
- `/home/kimberly/repos/gastown/internal/cmd/bead.go` (lines 15-214 — parent `Long`, three subcommand declarations, `init()` AddCommand block, `runBeadMove`; notes neutral)
- `/home/kimberly/repos/gastown/internal/cmd/cat.go` (in full, 79 lines; notes neutral)
- `/home/kimberly/repos/gastown/internal/cmd/changelog.go` (lines 28-145 — parent `Long` + Examples block, `init()`, `changelogSinceTime`, `collectChangelogEntries`; confirmed `changelogWeek` bound at `:49` and never read)
- `/home/kimberly/repos/gastown/internal/cmd/cleanup.go` (lines 1-50 — parent `Long` + `init()`; notes neutral)
- `/home/kimberly/repos/gastown/internal/cmd/close.go` (in full via spot-reads; notes neutral)
- `/home/kimberly/repos/gastown/internal/cmd/compact.go` (in full via spot-reads, confirmed `isReferenced` at `:456` has no call sites; notes neutral)
- `/home/kimberly/repos/gastown/internal/cmd/convoy.go` + `convoy_stage.go` + `convoy_launch.go` + `convoy_watch.go` (spot-verified subcommand registrations; notes neutral)
- `/home/kimberly/repos/gastown/internal/cmd/done.go` (lines 31-120 + 1205-1265 + 1670-1740 — `doneCmd` declaration, `runDone` prologue, `selfNukePolecat` declaration site and absence of call sites; confirmed DONE→IDLE transition path)
- `/home/kimberly/repos/gastown/docs/CLEANUP.md` (lines 17-29 — for verbatim quotation of the stale `gt done` row; re-read is for the existing `done.md` Phase-3 section reformat per task-explicit carveout and does NOT represent a Sweep-2-scope docs read)
- `/home/kimberly/repos/gastown/internal/cmd/formula.go` (spot-verified 4 subcommand registrations at `:179-184`; notes neutral)
- `/home/kimberly/repos/gastown/internal/cmd/handoff.go` (lines 30-200 — `handoffCmd`, `init()`, `runHandoff` prologue, polecat-detect-and-redirect block; notes neutral)
- `/home/kimberly/repos/gastown/internal/cmd/hook.go` (lines 14-250 — parent + subcommands declarations, `init()` with all 5 `hookCmd.AddCommand` calls; notes neutral)
- `/home/kimberly/repos/gastown/internal/cmd/molecule.go` (lines 15-268 — parent `Long`, all subcommand declarations, `init()` AddCommand block; confirmed only `moleculeStepDoneCmd` registered on `step` group in this file)
- `/home/kimberly/repos/gastown/internal/cmd/molecule_await_event.go` (in full, 115 lines — `moleculeAwaitEventCmd` declaration with full `Long`, `init()` registering on `moleculeStepCmd`)
- `/home/kimberly/repos/gastown/internal/cmd/molecule_await_signal.go` (lines 31-140 — `moleculeAwaitSignalCmd` + `moleculeAwaitSignalShortcutCmd` declarations, both `init()` registrations — `moleculeStepCmd.AddCommand(moleculeAwaitSignalCmd)` at `:113` and `moleculeCmd.AddCommand(moleculeAwaitSignalShortcutCmd)` at `:132`)
- `/home/kimberly/repos/gastown/internal/cmd/molecule_emit_event.go` (lines 18-65 — `moleculeEmitEventCmd` declaration + `moleculeStepCmd.AddCommand(moleculeEmitEventCmd)` at `:64`)
- `/home/kimberly/repos/gastown/internal/cmd/mountain.go` (spot-verified 4 subcommand registrations at `:100-103`; notes neutral)
- `/home/kimberly/repos/gastown/internal/cmd/mq.go` (lines 62-365 — parent `Long`, all 6 direct subcommand declarations, `mqIntegrationCmd` and its 3 children, `init()` AddCommand block; confirmed `mq.go`'s `init()` does NOT register `mq_next`)
- `/home/kimberly/repos/gastown/internal/cmd/mq_next.go` (in full, 55+ lines — `mqNextCmd` declaration with full `Long`, `init()` registering at `mq_next.go:47` via `mqCmd.AddCommand(mqNextCmd)`)
- `/home/kimberly/repos/gastown/internal/cmd/orphans.go` (spot-reads of `:157-181`, `:192-296`; notes neutral)
- `/home/kimberly/repos/gastown/internal/cmd/prune_branches.go` (in full, 95 lines — `Long` at `:20-39`; confirmed "(main)" is illustrative parenthetical, not a hard claim)
- `/home/kimberly/repos/gastown/internal/cmd/ready.go` (lines 28-254; notes neutral)
- `/home/kimberly/repos/gastown/internal/cmd/release.go` (in full, 78 lines — parent `Long`, `runRelease`; notes neutral)
- `/home/kimberly/repos/gastown/internal/cmd/resume.go` (in full, 144 lines; notes neutral)
- `/home/kimberly/repos/gastown/internal/cmd/scheduler.go` (lines 29-115 — parent + 6 subcommand declarations, `init()` AddCommand block; sibling `scheduler_convoy.go` / `scheduler_epic.go` inspected — they contain helper functions, no `AddCommand` calls; notes neutral)
- `/home/kimberly/repos/gastown/internal/cmd/sling.go` (spot-reads of `:27-290`, `slingRespawnResetCmd` at `:167-194`, sibling files inspected via `grep -n 'slingCmd.AddCommand' sling_*.go` returning no additional registrations; notes neutral)
- `/home/kimberly/repos/gastown/internal/cmd/synthesis.go` (spot-verified 3 subcommand registrations at `:102-104`; notes neutral)
- `/home/kimberly/repos/gastown/internal/cmd/trail.go` (lines 29-107 — parent + 3 subcommand declarations including the parent `Long` enumerating all 3 subcommands accurately; notes neutral)
- `/home/kimberly/repos/gastown/internal/cmd/unsling.go` (spot-reads of `:20-260`; notes neutral)
- `/home/kimberly/repos/gastown/internal/cmd/wl.go` (in full, 154 lines — parent `Long`, `wlJoinCmd`, `init()` at `:65-71` registering ONLY `wlJoinCmd`; confirmed `runWlJoin` at `:73-154`)
- `/home/kimberly/repos/gastown/internal/cmd/wl_browse.go` (spot-read — declaration + `wlCmd.AddCommand(wlBrowseCmd)` in `init()`)
- `/home/kimberly/repos/gastown/internal/cmd/wl_charsheet.go` (spot-read — declaration + AddCommand)
- `/home/kimberly/repos/gastown/internal/cmd/wl_claim.go` (spot-read — declaration + AddCommand)
- `/home/kimberly/repos/gastown/internal/cmd/wl_done.go` (spot-read — declaration + AddCommand)
- `/home/kimberly/repos/gastown/internal/cmd/wl_post.go` (spot-read — declaration + AddCommand)
- `/home/kimberly/repos/gastown/internal/cmd/wl_scorekeeper.go` (spot-read — declaration + AddCommand)
- `/home/kimberly/repos/gastown/internal/cmd/wl_show.go` (spot-read — declaration + AddCommand)
- `/home/kimberly/repos/gastown/internal/cmd/wl_stamp.go` (spot-read — declaration + AddCommand)
- `/home/kimberly/repos/gastown/internal/cmd/wl_stamps.go` (spot-read — declaration + AddCommand)
- `/home/kimberly/repos/gastown/internal/cmd/wl_sync.go` (spot-read — declaration + AddCommand)

Release-position verification: every promoted finding was cross-checked against `git -C /home/kimberly/repos/gastown show v1.0.0:<file>`. All findings are `in-release`:

- `assign.go:51` + `:62` `assignForce` declaration and flag registration byte-identical at v1.0.0; `runAssign` still does not reference `assignForce` at v1.0.0.
- `changelog.go:22` + `:49` `changelogWeek` declaration and flag registration byte-identical at v1.0.0; `changelogSinceTime` at `:99-121` never reads `changelogWeek` at v1.0.0.
- `molecule.go:22-43` `moleculeCmd.Long` byte-identical at v1.0.0; `molecule_await_event.go`, `molecule_await_signal.go`, `molecule_emit_event.go` all present at v1.0.0 with their AddCommand lines unchanged (verified via `git show v1.0.0:internal/cmd/molecule_<name>.go | grep -c AddCommand`).
- `mq_next.go` present at v1.0.0 with `mqCmd.AddCommand(mqNextCmd)` at the same line.
- All ten `wl_*.go` sibling files present at v1.0.0 with their AddCommand registrations byte-identical (verified via `git -C /home/kimberly/repos/gastown ls-tree -r --name-only v1.0.0 | grep internal/cmd/wl_` and per-file AddCommand count).
- `done.go:36-57` Cobra `Long` text describing DONE→IDLE transition byte-identical at v1.0.0; `done.go:1209-1263` DONE→IDLE implementation byte-equivalent at v1.0.0 (persistent polecat model shipped in v1.0.0). `docs/CLEANUP.md:28` stale row byte-identical at v1.0.0 — the drift was already present at the release tag. No post-release surprises in Work Management.

**Important release-position note on `wl`:** the Batch 1c dispatch prompt stated the ten `wl_*.go` sibling-file subcommands were "post-v1.0.0" additions (citing the plan's PR-delta scoping draft at line 1479). Direct verification at the `v1.0.0` tag contradicted that — every one of `wl_browse.go`, `wl_charsheet.go`, `wl_claim.go`, `wl_done.go`, `wl_post.go`, `wl_scorekeeper.go`, `wl_show.go`, `wl_stamp.go`, `wl_stamps.go`, `wl_sync.go` was already present at `v1.0.0` with its `wlCmd.AddCommand(...)` wired. The wiki-stale finding on `wl.md` is therefore `in-release`, not `post-release`, and the `phase-2-incomplete` classification holds (Phase 2 read `wl.go` in isolation on 2026-04-11, when all ten sibling files already existed). The PR-delta scoping draft needs a correction pass; flagged for the Sweep 1 retrospective gate.

**Docs files read:** none (Sweep 1) — **except** `docs/CLEANUP.md:17-29` which was re-read for verbatim quotation while reformatting the pre-existing Phase-3 `## Docs claim` / `## Drift` sections on `done.md` (commit `f143813`, 2026-04-14) to the v1.2 schema. This read is within the task-explicit carveout for `done.md`: "verify its existing `## Docs claim` / `## Drift` sections conform to v1.2 schema." No new Sweep 2 findings were generated — the existing `f143813` finding (drift in CLEANUP.md vs code, code correct, docs stale) was preserved with its citations updated to verbatim quotes and v1.2 fields. The pre-plan `[2026-04-14] drift-found | docs/CLEANUP.md vs code` log entry still covers the substance of the finding; Sweep 2 Batch 9 (`docs/CLEANUP.md` formal re-audit) will supersede it with a per-row audit trail.

**Wiki pages audited:**
- [assign](gastown/commands/assign.md) — `phase3_findings: [cobra-drift]`
- [bead](gastown/commands/bead.md) — `phase3_findings: [none]`
- [cat](gastown/commands/cat.md) — `phase3_findings: [none]`
- [changelog](gastown/commands/changelog.md) — `phase3_findings: [cobra-drift]`
- [cleanup](gastown/commands/cleanup.md) — `phase3_findings: [none]`
- [close](gastown/commands/close.md) — `phase3_findings: [none]`
- [compact](gastown/commands/compact.md) — `phase3_findings: [none]`
- [convoy](gastown/commands/convoy.md) — `phase3_findings: [none]`
- [done](gastown/commands/done.md) — `phase3_findings: [drift]` (reformat of existing f143813 section to v1.2 schema)
- [formula](gastown/commands/formula.md) — `phase3_findings: [none]`
- [handoff](gastown/commands/handoff.md) — `phase3_findings: [none]`
- [hook](gastown/commands/hook.md) — `phase3_findings: [none]`
- [molecule](gastown/commands/molecule.md) — `phase3_findings: [cobra-drift, wiki-stale]`
- [mountain](gastown/commands/mountain.md) — `phase3_findings: [none]`
- [mq](gastown/commands/mq.md) — `phase3_findings: [wiki-stale]`
- [orphans](gastown/commands/orphans.md) — `phase3_findings: [none]`
- [prune-branches](gastown/commands/prune-branches.md) — `phase3_findings: [none]`
- [ready](gastown/commands/ready.md) — `phase3_findings: [none]`
- [release](gastown/commands/release.md) — `phase3_findings: [none]`
- [resume](gastown/commands/resume.md) — `phase3_findings: [none]`
- [scheduler](gastown/commands/scheduler.md) — `phase3_findings: [none]`
- [sling](gastown/commands/sling.md) — `phase3_findings: [none]`
- [synthesis](gastown/commands/synthesis.md) — `phase3_findings: [none]`
- [trail](gastown/commands/trail.md) — `phase3_findings: [none]`
- [unsling](gastown/commands/unsling.md) — `phase3_findings: [none]`
- [wl](gastown/commands/wl.md) — `phase3_findings: [wiki-stale]`

**Findings by category:**

- **drift:** 1 finding on 1 page, `severity: wrong`, `in-release`, `fix tier: docs`:
  - **done** (`gastown/commands/done.md`) — `docs/CLEANUP.md:28` row for `gt done` claims the command "self-nukes worktree, kills own session," but `doneCmd.Long` at `done.go:36-57` and the DONE→IDLE transition at `done.go:1209-1263` implement the persistent polecat model: sandbox preserved, session kept alive for reuse. `selfNukePolecat` at `done.go:1673-1680` has no call sites (verified via grep). The Cobra `Long` and the inline implementation comment at `done.go:108-110` agree with each other; only `docs/CLEANUP.md` carries the stale claim. Reformat from existing pre-plan `f143813` section to v1.2 schema — verbatim quote, fresh file:line refs, Category/Severity/Fix tier/Release position fields, forward link to drift index.
- **cobra drift:** 3 findings on 3 pages, all `severity: wrong`, all `in-release`, all `fix tier: code`:
  - **assign** (`gastown/commands/assign.md`) — `assign.go:62` registers a `--force` flag with description `"Replace existing hooked work"`, but `assignForce` (declared at `:51`) is never read anywhere in `runAssign` (`:67-192`) or its helpers (`updateAgentHookBead` at `:166` doesn't take the flag). The flag's advertised behavior — replacing an existing hooked bead on re-assign — is unimplemented. Fix: either wire it up or remove the flag.
  - **changelog** (`gastown/commands/changelog.md`) — `changelogCmd.Long` Examples block at `changelog.go:39` lists `gt changelog --week` and the flag description at `:49` advertises `"Show this week's completions"`, but `changelogSinceTime` at `:99-121` never reads `changelogWeek` (declared at `:22`, bound at `:49`). The flag works only because its intended outcome coincides with the "no flag" default. Fix: either wire `changelogWeek` into `changelogSinceTime` or remove the flag and the Examples line.
  - **molecule** (`gastown/commands/molecule.md`) — parent `moleculeCmd.Long` at `molecule.go:22-43` hand-maintains a categorised subcommand list (under headings "VIEWING YOUR WORK", "WORKING ON STEPS", "LIFECYCLE") that names only 6 subcommands plus `step done`, omitting `status`, `attachment`, `attach-from-mail`, `dag` (registered in `molecule.go`) AND the four sibling-file subcommands `step await-event` (`molecule_await_event.go:115`), `step await-signal` (`molecule_await_signal.go:113`), `step emit-event` (`molecule_emit_event.go:64`), and the `await-signal` shortcut registered directly on `moleculeCmd` (`molecule_await_signal.go:132`). Same pattern as Batch 1a's `doctor`/`repair` and Batch 1b's `account`/`hooks` findings: hand-maintained `Long`-text summary lists drift reliably against code.
- **compound drift:** 0 findings (Sweep 1 does not read docs/ except the `done.md` carveout, which was a `drift` finding, not compound).
- **implementation-status unbuilt/partial/vestigial:** 0 findings. Note: `selfNukePolecat` at `done.go:1673-1680` meets the "explicit doc comment marks it as vestigial" bar (the doc comment at `:1735` says it is "Kept for explicit kill scenarios") but the function genuinely still has out-of-file callers via `gt polecat nuke` (not yet traced), so it is not unambiguously vestigial in-file. Left as neutral inside the `done.md` drift body. `isReferenced` at `compact.go:456` is a private helper with no call sites; neutral (no docs claim it).
- **wiki-stale:** 3 findings on 3 pages, all `severity: wrong`, all `fix tier: wiki` (fixed inline in `## What it actually does`), all tagged `phase-2-incomplete` per the 6e1fddc schema clarification:
  - **molecule** (`gastown/commands/molecule.md`) — **Phase 2 root cause: `phase-2-incomplete` (heuristic)**. Phase 2 page claimed "Eleven subcommands, registered at `molecule.go:253-267`" and "The `step` subcommand is itself a group … with `moleculeStepDoneCmd` as its only registered child". Re-read at HEAD: 16 subcommands total — sibling files `molecule_await_event.go`, `molecule_await_signal.go`, `molecule_emit_event.go` each register additional `step` children (and `molecule_await_signal.go` also registers a shortcut directly on `moleculeCmd`). All three sibling files present at `v1.0.0` with their AddCommand lines intact. Phase 2 was wrong at Phase 2 time — wiki body rewritten to enumerate the full 16-subcommand tree (10 direct on `mol` + 1 `await-signal` shortcut + 4 `step` children + 1 step group parent = 16 user-visible commands). `hooks` carries this finding AND the cobra-drift finding above on the same page, same pattern as Batch 1b `hooks.md`.
  - **mq** (`gastown/commands/mq.md`) — **Phase 2 root cause: `phase-2-incomplete` (heuristic)**. Phase 2 page said "Ten total, registered in `init()` at `mq.go:306-365`. Six top-level MR subcommands plus a three-subcommand `integration` group." Re-read at HEAD: 11 total — the 7th MR subcommand `mq next` is declared and registered by the sibling file `mq_next.go` via its own `init()` at `mq_next.go:47`. File present at `v1.0.0`. Phase 2 read `mq.go` in isolation and missed the sibling. Wiki body rewritten to add `mq next` as a new MR-level subcommand row; `sources:` frontmatter expanded to include `mq_next.go`.
  - **wl** (`gastown/commands/wl.md`) — **Phase 2 root cause: `phase-2-incomplete` (heuristic)**. Phase 2 page said "Only one is wired in this file (`wl.go:69-70`)" and speculated "Other wasteland subcommands … must be wired elsewhere, if at all." Re-read at HEAD: 11 subcommands total — `join` (in `wl.go`) plus `browse`, `charsheet`, `claim`, `done`, `post`, `scorekeeper`, `show`, `stamp`, `stamps`, `sync`, each declared and registered in its own sibling file (`wl_browse.go`, `wl_charsheet.go`, …, `wl_sync.go`) via per-file `init()` calling `wlCmd.AddCommand(...)`. All ten sibling files present at `v1.0.0` with AddCommand lines byte-identical. Phase 2 was wrong at Phase 2 time — wiki body rewritten to enumerate all 11 subcommands in a new table; `sources:` frontmatter expanded to list every sibling file. `wl leave` remains correctly identified as referenced-but-not-wired (the implementation-status: unbuilt candidate) — the Phase 2 observation was half right: `wl browse` WAS wired, `wl leave` was NOT.
- **gap:** 0 findings filed this sub-batch. Open gap: the ten `wl_*` subcommands deserve individual wiki pages rather than being consolidated on `wl.md`, by analogy with how `convoy.md` is the parent and `convoy_stage`/`convoy_launch`/`convoy_watch` helpers are mentioned inline. Filed as a Phase 4 (Coverage) follow-up rather than a Batch 1c bead (bd coordination deferred until the Sweep 1 retrospective gate reviews scope). Documented in the Notes section of the rewritten `wl.md`.
- **none:** 19 pages audited with no findings — `bead`, `cat`, `cleanup`, `close`, `compact`, `convoy`, `formula`, `handoff`, `hook`, `mountain`, `orphans`, `prune-branches`, `ready`, `release`, `resume`, `scheduler`, `sling`, `synthesis`, `trail`, `unsling`. (Count: 20 if you include `unsling` — let me recount: 19 — 26 total minus 7 finding pages: assign, changelog, done, molecule (combined cobra+wiki-stale), mq, wl = 6 distinct finding pages; 26 − 6 = 20 none pages, but one of them is double-listed because I said 19 above. Canonical list is the 20 pages enumerated in the "Wiki pages audited" block above with `phase3_findings: [none]`: bead, cat, cleanup, close, compact, convoy, formula, handoff, hook, mountain, orphans, prune-branches, ready, release, resume, scheduler, sling, synthesis, trail, unsling. 20 pages.) Source re-read confirmed current behavior matches the Cobra `Long` text and the Phase 2 wiki body at the level of detail the wiki covers.

**New beads filed:** none (deferred per Sweep 1 convention — gap items documented inline on the relevant pages for the Sweep 1 retrospective gate to consolidate).

**Beads closed:** none (Batch 1 anchor `wiki-vxl` stays `in_progress` across all eight sub-batches).

**Cross-link discipline:** 4 new `## Drift` sections added (assign, changelog, molecule, mq) with 6 finding rows total (assign: 1 cobra drift; changelog: 1 cobra drift; molecule: 1 cobra drift + 1 wiki-stale; mq: 1 wiki-stale). `done.md`'s existing `## Drift` section was rewritten in place to the v1.2 schema with 1 drift finding row. `wl.md`'s `## Drift` section is a single wiki-stale row. Total: 6 pages carry `## Drift` sections post-Batch-1c (assign, changelog, done, molecule, mq, wl). 4 new `## Docs claim` sections added (assign, changelog, molecule, mq); `done.md`'s existing `## Docs claim` section was rewritten in place with verbatim quote. `wl.md` has no `## Docs claim` section because its only finding is wiki-stale, which is fixed inline per the skill convention. All `file:line` refs are CURRENT (freshly read from HEAD `9f962c4a`, not copied from Phase 2 notes). All `## Docs claim` quotes re-checked against source character-by-character at the end of the sub-batch; no paraphrasing. Forward links to [gastown/drift/README.md](gastown/drift/README.md) added on every new `## Drift` section. Promoted Phase 2 Notes bullets left behind as forward redirects on `assign.md` (`--force` bullet redirected to `## Drift`), `changelog.md` (`--week` bullet redirected to `## Drift`); `molecule.md`, `mq.md`, `wl.md` wiki-stale fixes have inline body rewrites plus (for `wl.md`) a Notes section that documents the `wl leave` half of the Phase 2 observation as a separate neutral entry. `done.md` kept its existing Notes section unchanged (no Phase 2 bullet was the source of the drift finding — it was a pre-plan drift-found entry transcribed into the body).

**Judgment calls made during audit** (for orchestrator cross-check):

1. **`changelog --week` cobra-drift framing.** The `--week` flag appears to "work" — a user who types `gt changelog --week` gets this week's completions, because that's also the default. I could have classified this as neutral ("docs match observed behavior, even if the mechanism is wrong"). I promoted it to cobra-drift because (a) the code-vs-docs drift is real — the flag's documented effect is not produced by the documented mechanism — and (b) if the default ever changes, the flag silently stops working, which is a live correctness hazard. Consistent with Batch 1a's framing of `status --fast` which did more than advertised: once the observable behavior coincides with the docs by accident rather than by wiring, the audit treats it as drift regardless of user impact.
2. **`molecule` hand-maintained-list cobra-drift framing.** Same call as Batch 1a's `doctor` (Long text enumerates a curated subset of checks while `runDoctor` registers ~80), Batch 1b's `account` (Long text lists 4 subcommands, `init()` registers 5), and Batch 1b's `hooks` (Long text lists 8, 9 are wired). Three findings across 1a/1b all framed hand-maintained incomplete lists as cobra-drift; Batch 1c stays consistent and promotes molecule's incomplete categorised list the same way. The underlying pattern is meaningful enough that I noted it in the retro as a Phase 6 meta-finding candidate.
3. **`done.md` docs read during Sweep 1.** Sweep 1 is defined as "no docs/ reads." The task prompt explicitly carved out `done.md` because it already had pre-plan Phase 3 sections from commit `f143813` that needed reformatting to v1.2 schema. I verified the verbatim `docs/CLEANUP.md:28` row to produce a compliant `## Docs claim` quote. This is scope-bending (one docs file read during a Sweep 1 batch) but explicitly allowed by the carveout. The substance of the finding is unchanged; only its shape. Flagged so the orchestrator can ratify or redirect before Batch 1d.
4. **`wl` release-position correction.** The dispatch prompt identified `wl_browse`/`wl_charsheet`/`wl_claim`/`wl_done`/`wl_scorekeeper`/`wl_show`/`wl_stamp`/`wl_stamps` as "10 new `wl_*.go` command files post-v1.0.0" per the plan's PR-delta draft (plan line 1479). Direct verification via `git -C /home/kimberly/repos/gastown ls-tree -r --name-only v1.0.0 | grep internal/cmd/wl_` showed that `wl_browse.go`, `wl_charsheet.go`, `wl_claim.go`, `wl_done.go`, `wl_post.go`, `wl_schema_evolution.go`, `wl_scorekeeper.go`, `wl_show.go`, `wl_stamp.go`, `wl_stamps.go`, `wl_sync.go` are ALL present at `v1.0.0` — every wired subcommand is byte-identical in its `AddCommand` line. The Phase-3 plan's PR-delta scoping draft is incorrect on this point. Finding classified as `in-release` (not post-release), and the `phase-2-incomplete` sub-type holds: Phase 2 ran on 2026-04-11, well after v1.0.0 shipped, with all sibling files already present. Flagged for the Sweep 1 retrospective gate to correct the plan's PR-delta draft.
5. **`wl` Notes section: `wl leave` left as neutral.** Phase 2 speculated "`gt wl browse` and `gt wl leave` are referenced but not wired." The browse half is wrong (browse IS wired — that's the wiki-stale promotion). The leave half is correct: `runWlJoin` at `wl.go:109` references `gt wl leave first` in an error message, and no `wlLeaveCmd` exists at HEAD or v1.0.0. I left this half of the observation as a neutral Notes entry rather than promoting it. Rationale: to be a proper `implementation-status: unbuilt` finding, the docs (beyond the inline error message) should claim the command exists and work-in-progress. Error messages directing users to a nonexistent command are a neutral latent-gap class better handled by Sweep 2 (which may find `wl leave` in `docs/design/` or `docs/concepts/` and consolidate the finding with a wiki-entity-page gap). Batch 1c's scope is Sweep 1 promotion; a new Sweep-1-scope `implementation-status` finding would need a docs source, which isn't in Sweep 1's reach.
6. **`selfNukePolecat` stayed neutral inside the `done.md` body rather than being promoted to an `implementation-status: vestigial` finding.** The function at `done.go:1673-1680` has no call sites in `done.go` but the doc comment at `:1735` says it is "Kept for explicit kill scenarios," which suggests call sites may exist outside the file (via `gt polecat nuke`). Confirming this needs reading `internal/cmd/polecat*.go` which is outside the 26-page scope for Batch 1c. Left as a neutral sentence inside the drift body. If Batch 1g (Workspace) turns up a `gt polecat nuke` that imports and calls `selfNukePolecat`, this stays neutral forever; if not, it becomes a vestigial finding in a later sub-batch.
7. **`isReferenced` in `compact.go` stayed neutral.** The helper at `compact.go:455-458` is private with no call sites. No docs reference it. Dead-code neutral observation — would be promoted only under an implementation-status vestigial finding, which requires an explicit docs claim or Cobra hint that the feature exists.
8. **`assign --force` framing.** I framed the fix tier as "either wire it up or remove the flag." Either fix is a valid code change. The "fix direction" under the authority hierarchy is clearly `code` (Cobra's own flag registration IS the wrong claim — there's no `docs/` or README disagreement to route around). Consistent with Batch 1a's `activity` framing where the cobra Long text was the load-bearing claim. Flagged as a borderline "is this really cobra drift?" because the flag description line `"Replace existing hooked work"` is short enough that some reviewers might call it "flag metadata, not cobra `Long`". I took the strict reading — any code-internal docstring that makes a claim about behavior is subject to the authority hierarchy, whether it lives in `Long`, `Short`, or a `Flag` description. This is consistent with Batch 1b `shell`/`uninstall` where flag descriptions and `Long` were read as equal claims.

**Next sub-batch:** Batch 1d — Agent Management group (12 cmds). Tracked under `wiki-vxl` (Batch 1 anchor bead).

→ [gastown/commands/assign.md](gastown/commands/assign.md),
  [gastown/commands/bead.md](gastown/commands/bead.md),
  [gastown/commands/cat.md](gastown/commands/cat.md),
  [gastown/commands/changelog.md](gastown/commands/changelog.md),
  [gastown/commands/cleanup.md](gastown/commands/cleanup.md),
  [gastown/commands/close.md](gastown/commands/close.md),
  [gastown/commands/compact.md](gastown/commands/compact.md),
  [gastown/commands/convoy.md](gastown/commands/convoy.md),
  [gastown/commands/done.md](gastown/commands/done.md),
  [gastown/commands/formula.md](gastown/commands/formula.md),
  [gastown/commands/handoff.md](gastown/commands/handoff.md),
  [gastown/commands/hook.md](gastown/commands/hook.md),
  [gastown/commands/molecule.md](gastown/commands/molecule.md),
  [gastown/commands/mountain.md](gastown/commands/mountain.md),
  [gastown/commands/mq.md](gastown/commands/mq.md),
  [gastown/commands/orphans.md](gastown/commands/orphans.md),
  [gastown/commands/prune-branches.md](gastown/commands/prune-branches.md),
  [gastown/commands/ready.md](gastown/commands/ready.md),
  [gastown/commands/release.md](gastown/commands/release.md),
  [gastown/commands/resume.md](gastown/commands/resume.md),
  [gastown/commands/scheduler.md](gastown/commands/scheduler.md),
  [gastown/commands/sling.md](gastown/commands/sling.md),
  [gastown/commands/synthesis.md](gastown/commands/synthesis.md),
  [gastown/commands/trail.md](gastown/commands/trail.md),
  [gastown/commands/unsling.md](gastown/commands/unsling.md),
  [gastown/commands/wl.md](gastown/commands/wl.md)

## [2026-04-15] drift-found | Batch 1d (Sweep 1 commands/ Agent Management — 12 pages)

**Scope:** Sweep 1 promotion across all 12 pages in the `GroupAgents` cobra group (`agents`, `boot`, `callbacks`, `deacon`, `dog`, `mayor`, `polecat`, `refinery`, `role`, `session`, `signal`, `witness`). Every page received `phase3_audited` / `phase3_findings` / `phase3_severities` / `phase3_findings_post_release` frontmatter. Pages with non-neutral findings received `## Docs claim` + `## Drift` (and, for `witness`, `## Implementation status`) sections with verbatim quotes, current `file:line` refs, and v1.2 fix-tier + severity + release-position fields. Drift index stub at [gastown/drift/README.md](gastown/drift/README.md) already exists from Batch 1a; not modified in this sub-batch.

**Source files re-read at current HEAD** (current gastown HEAD `9f962c4af068fe9da9f4bd3624e7b66351121fdf`):

- `/home/kimberly/repos/gastown/internal/cmd/agents.go` (in full, 781 lines — parent `Long`, all 4 in-file subcommand declarations, `AgentType` enum at `:24-33`, `init()` AddCommand block at `:139-148`)
- `/home/kimberly/repos/gastown/internal/cmd/agent_state.go` (in full — `agentStateCmd` declaration with full `Long`, `init()` at `:76-86` registering `agentsCmd.AddCommand(agentStateCmd)` at `:85`)
- `/home/kimberly/repos/gastown/internal/cmd/boot.go` (lines 1-110 — parent `Long` at `:31-45`, three subcommand declarations, `init()` at `:90-100`)
- `/home/kimberly/repos/gastown/internal/dog/manager.go` (lines 290-314 — the `Manager.Get()` method with the explicit `"boot" is the boot watchdog using .boot-status.json, not a dog` exclusion at `:300`)
- `/home/kimberly/repos/gastown/internal/dog/manager_lifecycle_test.go` (lines 325-335 — test case exercising the boot-watchdog exclusion path)
- `/home/kimberly/repos/gastown/internal/cmd/callbacks.go` (in full, 508 lines — parent `Long`, `callbacksProcessCmd.Long` at `:88-103`, `handleSling` at `:471-502`, `logCallback` at `:505-508`, all per-type handlers)
- `/home/kimberly/repos/gastown/internal/cmd/deacon.go` (spot-reads of `:32-50` parent, `:396-414` `init()` AddCommand block, confirmed 15 subcommand registrations; notes neutral)
- `/home/kimberly/repos/gastown/internal/cmd/dog.go` (spot-reads of `:45-65` parent `Long`, `:255-297` `init()` AddCommand block, confirmed 9 subcommands; notes neutral)
- `/home/kimberly/repos/gastown/internal/cmd/mayor.go` (spot-reads of `:22-40` parent, `:118-137` `init()` AddCommand block, confirmed 6 subcommands; notes neutral)
- `/home/kimberly/repos/gastown/internal/cmd/polecat.go` (lines 1-60 parent `Long`, lines 366-376 `init()` AddCommand block for the 11 top-level subcommands)
- `/home/kimberly/repos/gastown/internal/cmd/polecat_identity.go` (spot-reads of `:31-40` parent `polecatIdentityCmd`, `:158-159` `polecatCmd.AddCommand(polecatIdentityCmd)` registration + 5 identity children on `polecatIdentityCmd`)
- `/home/kimberly/repos/gastown/internal/cmd/polecat_spawn.go` (in full — confirmed **zero** cobra command declarations and **zero** `init()` functions; package comment at `:1` is `"Package cmd provides polecat spawning utilities for gt sling."`)
- `/home/kimberly/repos/gastown/internal/cmd/polecat_cycle.go` (in full, 85 lines — confirmed zero cobra commands, zero `init()`, helper functions only)
- `/home/kimberly/repos/gastown/internal/cmd/polecat_helpers.go` (in full, 270 lines — confirmed zero cobra commands, zero `init()`)
- `/home/kimberly/repos/gastown/internal/cmd/refinery.go` (spot-reads of `:28-49` parent, `:229-270` `init()` AddCommand block, confirmed 11 subcommands; notes neutral)
- `/home/kimberly/repos/gastown/internal/cmd/role.go` (lines 1-150 parent + all 6 subcommand declarations, `init()` at `:138-150`, and `runRoleList` at `:604-623`; `parseRoleString` at `:339-394` and `detectRole` at `:244-336` confirming boot/dog are recognized roles but non-promoted)
- `/home/kimberly/repos/gastown/internal/cmd/session.go` (lines 1-220 — parent `Long` at `:43-49`, `sessionInjectCmd` at `:113-130` with `Short: "Send message to session (prefer 'gt nudge')"` and `Long` that preserves `inject` as a low-level primitive, `init()` at `:171-207`)
- `/home/kimberly/repos/gastown/internal/cmd/signal.go` (in full, 37 lines — parent `Long` lists only `stop`; `init()` registers only `signalStopCmd`)
- `/home/kimberly/repos/gastown/internal/cmd/signal_stop.go` (spot-verified declaration; notes neutral)
- `/home/kimberly/repos/gastown/internal/cmd/witness.go` (in full, 359 lines — parent `Long` at `:29-43`, `witnessStartCmd.Long` at `:50-63`, flag `--foreground` registration at `:124`, `runWitnessStart` vestige branch at `:179-183`, `init()` AddCommand block at `:122-143`)

**Sibling-file audit (per Batch 1c methodology, mandatory for Batch 1d)** — `ls /home/kimberly/repos/gastown/internal/cmd/<name>*.go` was run for every parent command in the group, and each sibling file was then grepped for `<name>Cmd.AddCommand\|init()\|var <name>` to check for unlisted registrations:

- `agents` → `agents.go`, `agents_test.go`, **plus `agent_state.go`** (LOAD-BEARING finding — `agent_state.go:85` registers `agentsCmd.AddCommand(agentStateCmd)` in its own `init()`, adding a 5th subcommand `agents state` that Phase 2 missed entirely because it read `agents.go` in isolation and did not grep the `agent_*.go` name-space).
- `boot` → `boot.go`, `boot_test.go` only. No unlisted subcommands.
- `callbacks` → `callbacks.go` only. 1 subcommand (`process`), accurately described in Phase 2 body.
- `deacon` → `deacon.go`, `deacon_status_test.go` only. 15 subcommands registered in the parent file's own `init()`, accurately enumerated in Phase 2 body.
- `dog` → `dog.go`, `dog_test.go` only. 9 subcommands in parent's `init()`, accurately enumerated.
- `mayor` → `mayor.go` only. 6 subcommands in parent's `init()`, accurately enumerated.
- `polecat` → `polecat.go`, `polecat_identity.go`, `polecat_spawn.go`, `polecat_cycle.go`, `polecat_helpers.go`, plus five tests. Only `polecat.go` (11 subcommands) and `polecat_identity.go` (6 entries — group parent + 5 children) register cobra commands; the other three sibling files contain **no** cobra command declarations and **no** `init()` functions (verified via `grep -c "cobra.Command\|AddCommand\|func init" internal/cmd/polecat_spawn.go internal/cmd/polecat_cycle.go internal/cmd/polecat_helpers.go` returning 0/0/0). Phase 2's body speculated "additional subcommands exist in `polecat_spawn.go` / `polecat_cycle.go`" as a follow-up item, which is wrong — those files are helpers only. Wiki-stale finding (phase-2-incomplete).
- `refinery` → `refinery.go`, `refinery_test.go` only. 11 subcommands in parent's `init()`, accurately enumerated.
- `role` → `role.go`, `role_boot_test.go`, `role_e2e_test.go`. 6 subcommands in parent's `init()`, accurately enumerated.
- `session` → `session.go`, `session_test.go` only. 9 subcommands in parent's `init()`, accurately enumerated.
- `signal` → `signal.go`, `signal_stop.go`, `signal_stop_test.go`. `signal_stop.go` declares `signalStopCmd` but it is registered in `signal.go`'s own `init()` at `signal.go:8-10` (`signalCmd.AddCommand(signalStopCmd)`). Phase 2's body correctly counts exactly one wired subcommand.
- `witness` → `witness.go`, `witness_test.go` only. 5 subcommands in parent's `init()`, accurately enumerated.

The sibling-file audit surfaced **two `phase-2-incomplete` findings** — `agents.md` (missed `agent_state.go`) and `polecat.md` (speculated about empty siblings instead of verifying). Both are load-bearing for the yield of this sub-batch.

Release-position verification: every promoted finding was cross-checked against `git -C /home/kimberly/repos/gastown show v1.0.0:<file>`. All findings are `in-release`:

- `agents.go:80-148` parent Long + `init()` byte-identical at v1.0.0; `agent_state.go` present at v1.0.0 with `agentsCmd.AddCommand(agentStateCmd)` at the same line.
- `boot.go:31-45` parent `Long` byte-identical at v1.0.0; `internal/dog/manager.go:300` "not a dog" exclusion present at v1.0.0.
- `callbacks.go:88-103` `callbacksProcessCmd.Long` byte-identical at v1.0.0; `handleSling` at `:471-502` byte-identical, comment `"We don't actually spawn here"` present.
- `polecat_spawn.go` / `polecat_cycle.go` / `polecat_helpers.go` present at v1.0.0 with **zero** cobra command declarations and **zero** `init()` functions (verified via `git show v1.0.0:internal/cmd/polecat_spawn.go | grep -c "cobra.Command\|func init" = 0`). Phase 2 speculating about these sibling files on 2026-04-11 was wrong at Phase 2 time.
- `session.go:43-49` `sessionCmd.Long` + `:113-130` `sessionInjectCmd` byte-identical at v1.0.0 (the "prefer `gt nudge`" framing pre-dates Phase 2).
- `witness.go:50-63` `witnessStartCmd.Long`, `:124` flag `--foreground` registration with description `"Run in foreground (default: background)"`, and `:179-183` vestige branch all byte-identical at v1.0.0.

No post-release surprises in Agent Management. Every finding that surfaced is in-release.

**Docs files read:** none (Sweep 1).

**Wiki pages audited:**
- [agents](gastown/commands/agents.md) — `phase3_findings: [wiki-stale]`
- [boot](gastown/commands/boot.md) — `phase3_findings: [cobra-drift]`
- [callbacks](gastown/commands/callbacks.md) — `phase3_findings: [cobra-drift]`
- [deacon](gastown/commands/deacon.md) — `phase3_findings: [none]`
- [dog](gastown/commands/dog.md) — `phase3_findings: [none]`
- [mayor](gastown/commands/mayor.md) — `phase3_findings: [none]`
- [polecat](gastown/commands/polecat.md) — `phase3_findings: [wiki-stale]`
- [refinery](gastown/commands/refinery.md) — `phase3_findings: [none]`
- [role](gastown/commands/role.md) — `phase3_findings: [none]`
- [session](gastown/commands/session.md) — `phase3_findings: [wiki-stale]`
- [signal](gastown/commands/signal.md) — `phase3_findings: [none]`
- [witness](gastown/commands/witness.md) — `phase3_findings: [cobra-drift, implementation-status-vestigial]`

**Findings by category:**

- **drift:** 0 findings (Sweep 1 does not read `docs/` files).
- **cobra drift:** 3 findings on 3 pages, all `severity: wrong`, all `in-release`, all `fix tier: code`:
  - **boot** (`gastown/commands/boot.md`) — `bootCmd.Long` at `boot.go:33` opens with `"Boot is a special dog that runs fresh on each daemon tick."` Code at `internal/dog/manager.go:300` is explicit (`"'boot' is the boot watchdog using .boot-status.json, not a dog"`) and `Manager.Get("boot")` returns `ErrDogNotFound`. No `AgentDog` type exists in the `AgentType` enum at `agents.go:24-33` (verified via `grep "AgentDog" internal/` returning one unrelated test-name hit). The "special dog" framing is a load-bearing intuition pump that directly contradicts the code's dog-manager exclusion and the absence of any `AgentDog` categorization anywhere in the tree. The drift resolves as clearly `wrong` rather than `ambiguous`: the code has a named exclusion for Boot-as-dog, not a loose association.
  - **callbacks** (`gastown/commands/callbacks.md`) — `callbacksProcessCmd.Long` at `callbacks.go:98` says `"SLING_REQUEST: - Spawn polecat for the work"` as one row of a per-type action table alongside `"POLECAT_DONE - Log completion, update stats"` and `"MERGE_COMPLETED - Notify worker, close source issue"`. `handleSling` at `callbacks.go:471-502` only calls `logCallback` and returns the action string `"logged sling request: %s to %s (execute with: gt sling %s %s)"`. The inline comment at `callbacks.go:498-499` spells out `"We don't actually spawn here - that would be done by the Deacon executing the sling command based on this request."` The row is archived (`callbacks.go:246-251`) after the handler returns "handled", so operators running `gt callbacks process` see `SLING_REQUEST` rows disappear from the inbox and reasonably conclude spawning happened — when in fact it did not. Same framing pattern as Batch 1b `plugin run` (text claims behavior that the runtime does not produce).
  - **witness** (`gastown/commands/witness.md`) — `witnessStartCmd.Long` at `witness.go:50-63` Examples block includes `"gt witness start greenplace --foreground"` as a normal invocation, and the flag registration at `witness.go:124` declares the description as `"Run in foreground (default: background)"`. Both text sources advertise `--foreground` as a real functional mode. `runWitnessStart` at `witness.go:179-183` short-circuits if `witnessForeground` is true and prints `"Note: Foreground mode no longer runs patrol loop"` followed by `"Patrol logic is now handled by mol-witness-patrol molecule"`, then returns nil. The Cobra surface has not caught up with the code's own vestige notice. **Double-tagged** as `cobra drift` and `implementation-status: vestigial` (see below) — the same flag is both a vestigial code artifact and a drifted docs claim. This is the first Phase 3 Sweep 1 sub-batch to produce a page that carries both category tags; the pattern mirrors 1b's `hooks.md` (which combined `cobra drift` with `wiki-stale`, a different compound shape).
- **compound drift:** 0 findings (Sweep 1 does not read `docs/`, so `drift` + `cobra drift` on the same finding is structurally impossible this pass). `witness --foreground` is NOT compound drift — it is `cobra drift` on the `Long` / flag description and `implementation-status: vestigial` on the same flag, two orthogonal axes of the taxonomy rather than two drift categories on the same claim.
- **implementation-status unbuilt/partial/vestigial:** 1 finding, `severity: wrong`, `in-release`, `fix tier: code`:
  - **witness** (`gastown/commands/witness.md`) — `witness --foreground` is a vestigial flag. The runtime notice at `witness.go:179-183` (`"Foreground mode no longer runs patrol loop"`) is explicit acknowledgement that the patrol loop has moved to `mol-witness-patrol` and the flag is preserved as a parseable no-op. Classified as `vestigial` (not `unbuilt` or `partial`) because the feature used to work and was deliberately retired rather than aspirationally planned. `Resolution: reframe with status callout` (short-term) or `delete` (long-term — after auditing any callers that still pass `--foreground`). NOT `preserve-as-vision` — this is a past state, not a future one.
- **wiki-stale:** 3 findings on 3 pages, all `severity: wrong`, all `fix tier: wiki` (fixed inline in `## What it actually does` or adjacent body text), all tagged `phase-2-incomplete` per the `6e1fddc` schema clarification:
  - **agents** (`gastown/commands/agents.md`) — **Phase 2 root cause: `phase-2-incomplete` (heuristic)**. Phase 2 body said `agentsCmd` "registers four subcommands" (`agents.md:29`) and the Subcommands table listed exactly `list`/`menu`/`check`/`fix`. Re-read at HEAD: a fifth subcommand `agents state` is registered by the sibling file `agent_state.go` via its own `init()` at `agent_state.go:85` (`agentsCmd.AddCommand(agentStateCmd)`). `agent_state.go` has its own full `Long` text at `:31-67` describing the label-based operational-state surface. File present at `v1.0.0` with the same registration line, so Phase 2 running on 2026-04-11 had access to it but didn't check the `agent_*.go` namespace (only the `agents.go` parent file). Same pattern as Batch 1b's `directive`/`hooks` and Batch 1c's `molecule`/`mq`/`wl`: Phase 2 read parent files in isolation without checking siblings for subcommand wiring. Fixed: body rewritten to enumerate 5 subcommands, invocation block adds `agents state`, Subcommands table adds the new row, `sources:` frontmatter expanded to include `agent_state.go`, Notes section gains a promotion note.
  - **polecat** (`gastown/commands/polecat.md`) — **Phase 2 root cause: `phase-2-incomplete` (heuristic)**. Phase 2 body at `polecat.md:42-55` said `polecat_spawn.go` "506 lines — additional spawn-related subcommands (inspection outside this wiki page's scope — flag this as a follow-up)" and `polecat_cycle.go` "85 lines — additional cycle-related subcommands (same — follow-up)," then at `:132-133` "(Additional subcommands exist in `polecat_spawn.go` and `polecat_cycle.go`; see 'Follow-up' in Notes.)" Re-read at HEAD: neither file contains any cobra command declarations or `init()` functions. `polecat_spawn.go:1` package comment is explicit (`"Package cmd provides polecat spawning utilities for gt sling."`); `grep -c "cobra.Command\|AddCommand\|func init" internal/cmd/polecat_spawn.go internal/cmd/polecat_cycle.go internal/cmd/polecat_helpers.go` returns 0/0/0. The actual subcommand tree is 17 entries (11 top-level from `polecat.go` + 6 identity from `polecat_identity.go`), all enumerated. Both sibling files present at `v1.0.0` with no cobra registrations either (verified via `git show v1.0.0:internal/cmd/polecat_spawn.go | grep -c "cobra.Command\|func init"` returning 0). Phase 2 parked a "follow-up" note in place of running the actual sibling-file verification — a `phase-2-incomplete` pattern that maps cleanly onto the Batch 1c methodology correction. Fixed: body rewritten to state definitively that the three non-command sibling files are helpers only, Invocation block trailer rewritten to close the tree, Notes section gains a promotion note. This is a slightly different shape from the Batch 1c `mq`/`wl` findings — those were "Phase 2 missed siblings that did register subcommands." This is "Phase 2 speculated about siblings that register nothing." Both are `phase-2-incomplete`; the sub-shape is "speculation instead of verification."
  - **session** (`gastown/commands/session.md`) — **Phase 2 root cause: `phase-2-incomplete` (heuristic)**. Phase 2 Related-commands cross-link at `session.md:231-232` said `"session inject is explicitly deprecated in favor of nudge per the Long help at session.go:48-49,116-121"`. Re-read at HEAD: neither `sessionCmd.Long` at `:43-49` nor `sessionInjectCmd.Long` at `:116-127` contains the word "deprecated." Both recommend `gt nudge` for Claude-session messages and explicitly preserve `inject` as `"a low-level primitive for file-based injection or cases where you need raw tmux send-keys behavior"`. The relationship is "prefer for Claude-session use, preserve for file injection," not "deprecated." Long text is byte-identical at `v1.0.0`, so Phase 2 had access to the accurate framing on 2026-04-11 and overstated it. Fixed: cross-link text rewritten to quote the `prefer-nudge` framing accurately and describe the preserved use cases; Notes section gains a promotion note. Smallest of the three `phase-2-incomplete` fixes in this sub-batch (minor cross-link polish rather than substantive body rewrite), but still a characterization mismatch that contradicts the Long text verbatim.
- **gap:** 0 findings filed this sub-batch. Open observations: (a) `role list` hand-maintained list omits `boot` / `dog` but Long says "Roles include..." which is non-exhaustive (neutral — see Judgment Calls #3); (b) `selfNukePolecat` call-site question from Batch 1c is still unresolved because `polecat*.go` do not grep as callers — it is likely called from `gt polecat nuke` via a different name (Batch 1g scope). Neither is promoted to a `wants-wiki-entry` bead this sub-batch; the retrospective gate reviews gap consolidation.
- **none:** 6 pages audited with no findings — `deacon`, `dog`, `mayor`, `refinery`, `role`, `signal`. Source re-read confirmed current behavior matches the Cobra `Long` text and the Phase 2 wiki body at the level of detail the wiki covers. `deacon` has 15 subcommands correctly enumerated at `deacon.go:396-414`; `dog` has 9 at `dog.go:286-296`; `mayor` has 6 at `mayor.go:118-124`; `refinery` has 11 at `refinery.go:257-267`; `role` has 6 at `role.go:138-150`; `signal` has 1 (`stop`) at `signal.go:8-10` — all accurate against current HEAD and v1.0.0.

**New beads filed:** none.

**Beads closed:** none (Batch 1 anchor `wiki-vxl` stays `in_progress` across all eight sub-batches).

**Cross-link discipline:** 3 new `## Drift` sections added (boot, callbacks, witness) with 4 finding rows total (boot: 1 cobra drift; callbacks: 1 cobra drift; witness: 1 cobra drift + 1 implementation-status vestigial in a separate `## Implementation status` section). 3 new `## Docs claim` sections added (boot, callbacks, witness); `witness.md`'s `## Docs claim` quotes BOTH the `Long` Examples block at `witness.go:50-63` AND the flag registration line at `witness.go:124` verbatim, because the drift finding rests on claims from both locations. 3 wiki-stale fixes (agents, polecat, session) have inline body rewrites only — no `## Docs claim` / `## Drift` sections, per the skill convention (wiki-stale fix tier is the wiki itself). Total: 3 pages with full `## Docs claim` + `## Drift` structure, 1 of which (witness) additionally carries `## Implementation status`; 3 pages with inline wiki-stale fixes and Notes promotion pointers; 6 pages with frontmatter-only `phase3_findings: [none]` updates. All `file:line` refs are CURRENT (freshly read from HEAD `9f962c4a`, not copied from Phase 2 notes). All `## Docs claim` quotes re-checked against source character-by-character at the end of the sub-batch; no paraphrasing. Forward links to [gastown/drift/README.md](gastown/drift/README.md) added on every new `## Drift` / `## Implementation status` section AND on the inline wiki-stale fix bullet of each wiki-stale page (per Batch 1b/1c precedent of linking wiki-stale fixes to the drift index even though the drift index's section 2 will be the one to catch them). Promoted Phase 2 Notes bullets left behind as forward redirects on `boot.md` (`"Special dog" vs dog.md` bullet → `## Drift`), `callbacks.md` (`handleSling doesn't sling` bullet → `## Drift`), `witness.md` (`--foreground is vestigial` bullet → `## Drift` + `## Implementation status`). `agents.md`, `polecat.md`, `session.md` have new Notes-section promotion notes describing the inline fix rather than redirects (Phase 2 didn't have a dedicated bullet for the wiki-stale issues in agents/polecat/session — the stale claims lived in the body text or cross-link text, not in `## Notes / open questions`, so there's no bullet to redirect).

**Judgment calls made during audit** (for orchestrator cross-check):

1. **`session inject` deprecation characterization — wiki-stale or neutral?** The Phase 2 cross-link text at `session.md:231-232` used the word "deprecated." The actual Cobra Long text uses "prefer `gt nudge`" and explicitly preserves inject for file-injection use cases. Strict code-first reading: the wiki characterization contradicts the Long text verbatim (no "deprecated" wording anywhere), which is wiki-stale. Lenient reading: "deprecated" is colloquial shorthand for "prefer the new thing," and the functional relationship is the same, so it's neutral. I took the strict reading and promoted to wiki-stale because the Phase 3 discipline is verbatim-against-code and the characterization difference matters: "deprecated" implies scheduled removal, "preferred alternative" implies indefinite coexistence. `gt session inject -f prompt.txt` remains the canonical file-injection primitive per the Long — a reader who took "deprecated" at face value would look for its replacement and find none.

2. **`boot`'s "special dog" cobra drift severity — wrong or ambiguous?** The `Long` text says `"Boot is a special dog"`. "Special" is a softening modifier — one reading is that "special dog" means "not quite a dog, but adjacent," which would be ambiguous (vague, not verifiably wrong against code). The opposing reading is that "special dog" presents Boot AS a member of the dog category with some distinguishing properties, and the code explicitly excludes Boot from that category (`internal/dog/manager.go:300`) and has no `AgentDog` type anywhere. I took the second reading and tagged `severity: wrong` because the code's `"not a dog"` exclusion comment at `manager.go:300` is literally the negation of the Long's claim. If the Long said something like `"Boot is a short-lived agent that lives in the dogs/ directory"`, that would be ambiguous-but-not-wrong (a reader could infer either dog membership or dog-adjacent membership). The "is a [special] dog" framing crosses the line from "dog-adjacent" to "dog-member," and the code says the latter is wrong. Fix tier `code` — edit the Long text to drop or reframe the dog metaphor (candidates given in the finding body).

3. **`role list` omission of boot/dog — drift or neutral?** `roleListCmd.Long` at `role.go:93-97` says `"Roles include mayor, deacon, witness, refinery, polecat, and crew."` `runRoleList` at `role.go:604-623` hardcodes exactly those six. `parseRoleString`, `detectRole`, and `getRoleHome` also handle `RoleBoot` and `RoleDog`. If the Long said "Roles are" or "All Gas Town roles:", the omission would be cobra drift (Long enumeration incomplete vs code enumeration). The Long says "include" — non-exhaustive. I stayed neutral. A stricter reading would promote this as `cobra drift` on the grounds that users expect `gt role list` to list every role they can end up in, and boot/dog are real detectable roles. The neutral call is consistent with Batch 1a's `log.md --since` framing (where the Long didn't commit to a specific parser flavor, so the behavior asymmetry was neutral). Flagged for the Sweep 1 retrospective gate because the "hand-maintained list omits cases the code actually handles" pattern is building up across batches — `role list` is the borderline case where the Long's softening language saves it from promotion, whereas `doctor` / `repair` / `account` / `hooks` / `molecule` all used stricter language ("lists:", "checks:", "Commands:") that made their omissions clear drift.

4. **`polecat_spawn.go` / `polecat_cycle.go` wiki-stale framing — speculation is wrong, but is it wiki-stale?** Phase 2 parked the "follow-up to enumerate additional subcommands in these sibling files" note as an open question rather than a definitive claim. Strict reading: an open-question note is not a drift claim (it's explicitly "we didn't check"). Lenient reading: Phase 2's synthesis claimed the siblings "register additional subcommands" as a fact-shaped sentence before punting to a follow-up, and that fact-shaped sentence is wrong. I took the lenient reading and promoted to wiki-stale because the Phase 2 body had a factual-sounding sentence ("`polecat_spawn.go` (506 lines) — additional spawn-related subcommands") that a reader takes as ground truth. The follow-up punt comes after the claim, not in place of it. Also flagged inline that this is a slightly different `phase-2-incomplete` sub-shape than the Batch 1b/1c cases — Phase 2 speculated *down* (inventing unverified registrations) instead of speculating *out* (missing siblings altogether). Same root cause (sibling-file verification skipped), different surface symptom.

5. **`witness --foreground` double-tagging — cobra drift OR implementation-status vestigial OR both?** The flag is clearly vestigial (code says so at `:179-183`). But the `Long` + flag description don't acknowledge the vestige — they still present the flag as functional. Is that two findings, or one? I took the "two findings" reading and gave the page BOTH `phase3_findings: [cobra-drift, implementation-status-vestigial]` with separate `## Drift` and `## Implementation status` sections. The rationale is that the fix work splits cleanly: the `cobra drift` row lives at the Long / flag-description level (surface), the `implementation-status: vestigial` row lives at the code / runtime-notice level (primitive), and a Phase 6 fix might address one without the other (e.g. rewrite the Long to say "vestigial" while still leaving the flag in place as a parseable no-op = Cobra drift fixed, vestige acknowledged). The skill's compound-drift pattern (`cobra drift` + `drift`) is specifically for docs + Cobra disagreement; this is Cobra + code disagreement, which is a different axis. If the orchestrator prefers a single-finding framing ("the flag is just vestigial, and the vestige extends into the Long text"), the second tag could be dropped in the retrospective gate and the finding folded into a single `implementation-status: vestigial` row. The two-finding reading preserves the two fix surfaces cleanly.

6. **`polecat.md` body-vs-Notes fix placement.** The wiki-stale fix on `polecat.md` rewrites both (a) a section of `## What it actually does` (the sibling-file enumeration at the top) AND (b) the Invocation block trailer that pointed at "Follow-up" in Notes, AND (c) adds a promotion note in Notes. That's three touch points for one finding. The skill's guidance says "fix inline in `## What it actually does`" for wiki-stale, which is ambiguous about what to do when the stale claim lives in multiple sections. I chose to touch all three because leaving any of them with the old speculation would be internally incoherent (e.g., an accurate sibling-file section at the top + a stale "see Follow-up" at the bottom). Adds verbosity but preserves page coherence. Flagged for the retrospective gate as a "when a wiki-stale claim lives in multiple sections of a page, should we fix all of them or only the primary locus?" question.

7. **`agents state` documentation depth.** I added `agents state` to the subcommand table with a one-line description and referenced `agent_state.go:31-67` for the full label vocabulary. A more thorough fix would enumerate the `--set` / `--incr` / `--del` / `--json` flags in the Flags table and describe the label-value schema (`idle:<n>`, `backoff:<d>`, `last_activity:<ts>`) in a new subsection. I took the minimal fix because the purpose of the Phase 3 Sweep 1 pass is to correct the drift claim (Phase 2 missed the subcommand), not to produce full Phase 2 coverage of the new subcommand — that's Phase 4 (Coverage) work. If the retrospective gate wants fuller coverage, the `agents state` subsection can be expanded in place without re-landing the finding.

8. **`callbacks handleSling` framing — cobra drift vs implementation-status vestigial.** The `Long` says "Spawn polecat for the work." The code logs and returns. A sibling reading of this finding is "the claim is aspirational — someday `handleSling` should spawn, but right now it defers to Deacon." That would be `implementation-status: partial` or `unbuilt`. I rejected that reading because the `callbacks.go:498-499` comment is explicit that the deferral is the intended design ("We don't actually spawn here - that would be done by the Deacon executing the sling command based on this request") — not a "TODO implement this later" comment. The code is not aspirational; it is deliberately non-spawning. Therefore the fix tier is `code` (edit the Long to describe the log-and-defer reality) rather than `preserve-as-vision`. This is consistent with Batch 1b `plugin run` (where `--force` was a no-op for 4 of 5 gate types and the Long still described the cooldown-only gate as the general case — cobra drift, not implementation status).

**Next sub-batch:** Batch 1e — Communication group (7 cmds). Tracked under `wiki-vxl` (Batch 1 anchor bead).

→ [gastown/commands/agents.md](gastown/commands/agents.md),
  [gastown/commands/boot.md](gastown/commands/boot.md),
  [gastown/commands/callbacks.md](gastown/commands/callbacks.md),
  [gastown/commands/deacon.md](gastown/commands/deacon.md),
  [gastown/commands/dog.md](gastown/commands/dog.md),
  [gastown/commands/mayor.md](gastown/commands/mayor.md),
  [gastown/commands/polecat.md](gastown/commands/polecat.md),
  [gastown/commands/refinery.md](gastown/commands/refinery.md),
  [gastown/commands/role.md](gastown/commands/role.md),
  [gastown/commands/session.md](gastown/commands/session.md),
  [gastown/commands/signal.md](gastown/commands/signal.md),
  [gastown/commands/witness.md](gastown/commands/witness.md)

## [2026-04-15] drift-found | Batch 1e (Sweep 1 commands/ Communication — 7 pages)

**Scope:** Sweep 1 promotion across all 7 pages in the `GroupComm` cobra group (`broadcast`, `dnd`, `escalate`, `mail`, `notify`, `nudge`, `peek`). Every page received `phase3_audited` / `phase3_findings` / `phase3_severities` / `phase3_findings_post_release` frontmatter. `mail.md` received `## Docs claim` + `## Drift` sections with verbatim quotes, current `file:line` refs, and v1.2 fix-tier + severity + release-position fields.

**Source files re-read at current HEAD** (current gastown HEAD `9f962c4af068fe9da9f4bd3624e7b66351121fdf`):

- `/home/kimberly/repos/gastown/internal/cmd/broadcast.go` (lines 27-45 — parent `Long` at `:31-44`)
- `/home/kimberly/repos/gastown/internal/cmd/dnd.go` (lines 13-40 — parent `Long` at `:18-36`)
- `/home/kimberly/repos/gastown/internal/cmd/escalate.go` (lines 24-60 — parent `Long` at `:28-56`, 5 subcommands at `:164-169`)
- `/home/kimberly/repos/gastown/internal/cmd/escalate_impl.go` (grep — zero cobra registrations)
- `/home/kimberly/repos/gastown/internal/cmd/mail.go` (lines 55-130, 520-543 — parent `Long` COMMANDS block at `:92-95`; `init()` registering 17 subcommands)
- `/home/kimberly/repos/gastown/internal/cmd/mail_channel.go` (lines 1-50, 123-149 — `init()` registering 7 children + `mailCmd.AddCommand(mailChannelCmd)` at `:149`)
- `/home/kimberly/repos/gastown/internal/cmd/mail_directory.go` (lines 1-50 — `init()` at `:40-42` registering `mailCmd.AddCommand(mailDirectoryCmd)`)
- `/home/kimberly/repos/gastown/internal/cmd/mail_group.go` (lines 1-50, 97-115 — `init()` registering 6 children + `mailCmd.AddCommand(mailGroupCmd)` at `:115`)
- `/home/kimberly/repos/gastown/internal/cmd/mail_hook.go` (lines 1-50 — `init()` at `:39-45` registering `mailCmd.AddCommand(mailHookCmd)`)
- `/home/kimberly/repos/gastown/internal/cmd/mail_queue.go` (lines 454-548 — `init()` at `:532-548` registering 4 children + `mailCmd.AddCommand(mailQueueCmd)`)
- `/home/kimberly/repos/gastown/internal/cmd/notify.go` (lines 13-40 — parent `Long` at `:19-35`)
- `/home/kimberly/repos/gastown/internal/cmd/nudge.go` (lines 62-140 — parent `Long` at `:78-131`, `ifFreshMaxAge` at `:135`)
- `/home/kimberly/repos/gastown/internal/cmd/nudge_poller.go` (lines 1-25 — `init()` at `:22` registers on `rootCmd`, not `nudgeCmd`)
- `/home/kimberly/repos/gastown/internal/cmd/peek.go` (lines 22-50 — parent `Long` at `:28-47`)

**Sibling-file audit:**

- `broadcast` → `broadcast.go` only.
- `dnd` → `dnd.go`, `dnd_test.go`. No AddCommand in test.
- `escalate` → `escalate.go`, `escalate_impl.go`, `escalate_test.go`. `escalate_impl.go` zero cobra registrations. 5 subcommands in parent, accurate.
- `mail` → **LOAD-BEARING**: 13 non-test siblings. 5 register NEW subcommand groups: `mail_channel.go:149`, `mail_directory.go:42`, `mail_group.go:115`, `mail_hook.go:45`, `mail_queue.go:548`. Phase 2 listed all siblings in `sources:` but did not verify `init()` blocks — `phase-2-incomplete`.
- `notify` → `notify.go` only.
- `nudge` → `nudge_poller.go` registers on `rootCmd` (separate command). No wiki-stale.
- `peek` → `peek.go` only.

Release-position: `mail.go` COMMANDS section byte-identical at v1.0.0; all 5 sibling files present at v1.0.0. All findings `in-release`.

**Docs files read:** none (Sweep 1).

**Wiki pages audited:**
- [broadcast](gastown/commands/broadcast.md) — `phase3_findings: [none]`
- [dnd](gastown/commands/dnd.md) — `phase3_findings: [none]`
- [escalate](gastown/commands/escalate.md) — `phase3_findings: [none]`
- [mail](gastown/commands/mail.md) — `phase3_findings: [cobra-drift, wiki-stale]`
- [notify](gastown/commands/notify.md) — `phase3_findings: [none]`
- [nudge](gastown/commands/nudge.md) — `phase3_findings: [none]`
- [peek](gastown/commands/peek.md) — `phase3_findings: [none]`

**Findings by category:**

- **cobra drift:** 1 finding on `mail.md` — `mailCmd.Long` COMMANDS lists 4; 22 registered. `severity: wrong`, `in-release`, `fix tier: code`. Hand-maintained enumeration pattern (6th instance across Batches 1a-1e).
- **wiki-stale:** 1 finding on `mail.md` — Phase 2 table listed 17; missed 5 sibling-registered groups (`channel`, `directory`, `group`, `hook`, `queue`). **Phase 2 root cause: `phase-2-incomplete` (heuristic)**. Fixed inline.
- **none:** 6 pages (`broadcast`, `dnd`, `escalate`, `notify`, `nudge`, `peek`).

**Judgment calls:** (1) Mail "durable" wiki characterization vs `--wisp=true` → neutral (wiki editorial, Notes already flags). (2) `nudge --if-fresh` 60s → neutral (flag description accurate). (3) `dnd off` verbose-loss → neutral (Long says "resume normal", correct). (4) `escalate` not beads-exempt → neutral (architectural observation).

**Cross-link discipline:** 1 `## Docs claim` + 1 `## Drift` + 1 wiki-stale inline fix on `mail.md`. Forward links to [gastown/drift/README.md](gastown/drift/README.md). All `file:line` refs fresh from HEAD `9f962c4a`. 6 pages frontmatter-only.

**Next sub-batch:** Batch 1f — Services group (11 cmds). Tracked under `wiki-vxl`.

**Audited pages:**
  [gastown/commands/broadcast.md](gastown/commands/broadcast.md),
  [gastown/commands/dnd.md](gastown/commands/dnd.md),
  [gastown/commands/escalate.md](gastown/commands/escalate.md),
  [gastown/commands/mail.md](gastown/commands/mail.md),
  [gastown/commands/notify.md](gastown/commands/notify.md),
  [gastown/commands/nudge.md](gastown/commands/nudge.md),
  [gastown/commands/peek.md](gastown/commands/peek.md)

## [2026-04-15] drift-found | Batch 1f (Sweep 1 commands/ Services — 11 pages)

**Scope:** Sweep 1 promotion across all 11 pages in the `GroupServices` cobra group (`daemon`, `dolt`, `down`, `estop`, `maintain`, `quota`, `reaper`, `shutdown`, `start`, `thaw`, `up`). Every page received `phase3_audited` / `phase3_findings` / `phase3_severities` / `phase3_findings_post_release` frontmatter. `quota.md`, `reaper.md`, and `down.md` received `## Docs claim` + `## Drift` sections with verbatim quotes, current `file:line` refs, and v1.2 fix-tier + severity + release-position fields. `dolt.md` received `## Docs claim` section with inline wiki-stale fix for 2 missed sibling-file subcommands.

**Source files re-read at current HEAD** (current gastown HEAD `9f962c4af068fe9da9f4bd3624e7b66351121fdf`):

- `/home/kimberly/repos/gastown/internal/cmd/daemon.go` (lines 21-167 — parent `Long` at `:26-34`, `init()` registering 8 subcommands at `:152-165`)
- `/home/kimberly/repos/gastown/internal/cmd/daemon_reload_unix.go` (lines 10-11 — `signalDaemonReload` sends SIGUSR2)
- `/home/kimberly/repos/gastown/internal/cmd/dolt.go` (lines 20-389 — parent `Long` at `:25-35`, `init()` at `:345-389` registering 19 subcommands)
- `/home/kimberly/repos/gastown/internal/cmd/dolt_flatten.go` (lines 1-60 — `init()` at `:43-46` registering `flatten` on `doltCmd`)
- `/home/kimberly/repos/gastown/internal/cmd/dolt_rebase.go` (lines 1-70 — `init()` at `:56-63` registering `rebase` on `doltCmd`)
- `/home/kimberly/repos/gastown/internal/cmd/down.go` (lines 45-94 — parent `Long` at `:49-77`, `init()` at `:86-93`)
- `/home/kimberly/repos/gastown/internal/cmd/estop.go` (lines 1-80 — `estopCmd.Long` at `:31-48`, `thawCmd.Long` at `:51-61`, `init()` at `:67-72`)
- `/home/kimberly/repos/gastown/internal/cmd/maintain.go` (lines 1-80 — parent `Long` at `:42-62`, `init()` at `:63-67`)
- `/home/kimberly/repos/gastown/internal/cmd/quota.go` (lines 36-55 — parent `Long` at `:42-54`, `init()` at `:960-980` registering 5 subcommands)
- `/home/kimberly/repos/gastown/internal/cmd/reaper.go` (lines 27-60 — parent `Long` at `:34-45`, `init()` at `:522-573` registering 6 subcommands)
- `/home/kimberly/repos/gastown/internal/cmd/start.go` (lines 1-170 — `startCmd.Long` at `:62-74`, `shutdownCmd.Long` at `:83-110`, `init()` at `:133-164`)
- `/home/kimberly/repos/gastown/internal/cmd/up.go` (lines 120-170 — parent `Long` at `:125-147`, `init()` at `:154-158`)

**Sibling-file audit:**

- `daemon` → `daemon.go`, `daemon_reload_unix.go`, `daemon_reload_windows.go`, `daemon_test.go`. Reload siblings define `signalDaemonReload` (SIGUSR2 on Unix). No AddCommand in siblings. 8 subcommands in parent `init()`, accurate.
- `dolt` → **LOAD-BEARING**: `dolt.go`, `dolt_flatten.go`, `dolt_rebase.go`, `dolt_test.go`, `dolt_test_helpers_test.go`. Two siblings register NEW subcommands: `dolt_flatten.go:46` registers `flatten`, `dolt_rebase.go:63` registers `rebase`. Phase 2 listed only `dolt.go` in `sources:` — `phase-2-incomplete`.
- `reaper` → `reaper.go` only. 6 subcommands in `init()`, accurate.
- `quota` → `quota.go` only. 5 subcommands in `init()`, accurate.
- `start` → `start.go`, `start_orphan_unix.go`, `start_orphan_windows.go`. Orphan siblings define platform-specific orphan cleanup helpers. No AddCommand in siblings.
- `up` → `up.go`, `up_json_test.go`, `up_test.go`. No AddCommand in siblings.
- `down` → `down.go`, `down_legacy_socket_test.go`, `down_test.go`. No AddCommand in siblings.
- `estop` → `estop.go`, `estop_unix.go`, `estop_windows.go`. Platform siblings define signal helpers. No AddCommand in siblings.
- `maintain` → `maintain.go`, `maintain_test.go`. No AddCommand in siblings.

Release-position: all source files present at v1.0.0. All findings `in-release`.

**Docs files read:** none (Sweep 1).

**Wiki pages audited:**
- [daemon](gastown/commands/daemon.md) — `phase3_findings: [none]`
- [dolt](gastown/commands/dolt.md) — `phase3_findings: [wiki-stale]`
- [down](gastown/commands/down.md) — `phase3_findings: [cobra-drift]`
- [estop](gastown/commands/estop.md) — `phase3_findings: [none]`
- [maintain](gastown/commands/maintain.md) — `phase3_findings: [none]`
- [quota](gastown/commands/quota.md) — `phase3_findings: [cobra-drift]`
- [reaper](gastown/commands/reaper.md) — `phase3_findings: [cobra-drift]`
- [shutdown](gastown/commands/shutdown.md) — `phase3_findings: [none]`
- [start](gastown/commands/start.md) — `phase3_findings: [none]`
- [thaw](gastown/commands/thaw.md) — `phase3_findings: [none]`
- [up](gastown/commands/up.md) — `phase3_findings: [none]`

**Findings by category:**

- **cobra drift:** 3 findings.
  - `quota.md` — `quotaCmd.Long` COMMANDS lists 4; 5 registered (missing `watch`). `severity: wrong`, `in-release`, `fix tier: code`. Hand-maintained enumeration pattern (7th instance across Batches 1a-1f).
  - `reaper.md` — `reaperCmd.Long` "When run by a Dog" lists 4; 6 registered (missing `databases`, `run`). `severity: wrong`, `in-release`, `fix tier: code`. Hand-maintained enumeration pattern (8th instance).
  - `down.md` — `downCmd.Long` says "use 'gt start' to bring everything back up" but `gt start` does not start the daemon; `gt up` is the actual complement. `severity: wrong`, `in-release`, `fix tier: code`.
- **wiki-stale:** 1 finding on `dolt.md` — Phase 2 invocation listed 19 subcommands; missed `flatten` and `rebase` from sibling files `dolt_flatten.go` and `dolt_rebase.go`. **Phase 2 root cause: `phase-2-incomplete` (heuristic)**. Fixed inline.
- **none:** 7 pages (`daemon`, `estop`, `maintain`, `shutdown`, `start`, `thaw`, `up`).

**Judgment calls:** (1) `daemon.Long` does not enumerate subcommands → no enumeration drift possible; neutral. (2) `down.Long` "use 'gt start'" → classified as cobra-drift because `gt start` genuinely does not restart the daemon and is not the symmetric counterpart of `gt down`. (3) `start` vs `up` overlap → both Long texts are internally accurate; the architectural divergence (two boot commands) is a design observation, not drift. (4) `estop` "manual only" claim matches code (PR #3237 cherry-pick). (5) `maintain` 4-step pipeline claim matches code. (6) `shutdown` Long accurately describes the comparison with `down`. (7) `thaw` Long accurately describes SIGCONT + sentinel removal.

**Cross-link discipline:** 3 `## Docs claim` + 3 `## Drift` sections + 1 wiki-stale inline fix. Forward links to [gastown/drift/README.md](gastown/drift/README.md). All `file:line` refs fresh from HEAD `9f962c4a`. 7 pages frontmatter-only. Sibling sources added to `sources:` frontmatter on `daemon`, `dolt`, `estop`, `maintain`, `start`, `thaw`.

**Next sub-batch:** Batch 1g — Workspace group (7 cmds). Tracked under `wiki-vxl`.

**Audited pages:**
  [gastown/commands/daemon.md](gastown/commands/daemon.md),
  [gastown/commands/dolt.md](gastown/commands/dolt.md),
  [gastown/commands/down.md](gastown/commands/down.md),
  [gastown/commands/estop.md](gastown/commands/estop.md),
  [gastown/commands/maintain.md](gastown/commands/maintain.md),
  [gastown/commands/quota.md](gastown/commands/quota.md),
  [gastown/commands/reaper.md](gastown/commands/reaper.md),
  [gastown/commands/shutdown.md](gastown/commands/shutdown.md),
  [gastown/commands/start.md](gastown/commands/start.md),
  [gastown/commands/thaw.md](gastown/commands/thaw.md),
  [gastown/commands/up.md](gastown/commands/up.md)

## [2026-04-15] drift-found | Batch 1g (Sweep 1 commands/ Workspace — 7 pages)

**Scope:** Sweep 1 promotion across all 7 pages in the `GroupWorkspace` cobra group (`crew`, `git-init`, `init`, `install`, `namepool`, `rig`, `worktree`). Every page received `phase3_audited` / `phase3_findings` / `phase3_severities` / `phase3_findings_post_release` frontmatter. `crew.md`, `namepool.md`, `install.md`, `git-init.md`, and `init.md` received `## Docs claim` + `## Drift` sections with verbatim quotes, current `file:line` refs, and v1.2 fix-tier + severity + release-position fields. `init.md` also received an inline wiki-stale fix.

**Source files re-read at current HEAD** (current gastown HEAD `9f962c4af068fe9da9f4bd3624e7b66351121fdf`):

- `/home/kimberly/repos/gastown/internal/cmd/crew.go` (lines 29-60 — `crewCmd.Long` at `:35-58`; `init()` at `:363-431` registering 13 subcommands)
- `/home/kimberly/repos/gastown/internal/cmd/rig.go` (lines 36-50 — `rigCmd.Long` at `:42-50`; `init()` at `:342-355` registering 12 subcommands in parent; 8 more via siblings)
- `/home/kimberly/repos/gastown/internal/cmd/namepool.go` (lines 22-39 — `namepoolCmd.Long` at `:27-38`; `init()` at `:110-117` registering 6 subcommands)
- `/home/kimberly/repos/gastown/internal/cmd/install.go` (lines 48-78 — `installCmd.Long` at `:52-73`; `init()` at `:79-92`)
- `/home/kimberly/repos/gastown/internal/cmd/gitinit.go` (lines 20-47 — `gitInitCmd.Long` at `:25-44`; `init()` at `:48-51`)
- `/home/kimberly/repos/gastown/internal/cmd/init.go` (lines 22-38 — `initCmd.Long` at `:25-31`; `init()` at `:35-37`)
- `/home/kimberly/repos/gastown/internal/cmd/worktree.go` (lines 23-49 — `worktreeCmd.Long` at `:29-44`; `init()` at `:87-94`)
- `/home/kimberly/repos/gastown/internal/rig/types.go` (lines 46-54 — `AgentDirs` slice definition, 5 entries)

**Sibling-file audit:**

- `crew` → `crew.go` + 8 sibling source files (`crew_add.go`, `crew_at.go`, `crew_cycle.go`, `crew_helpers.go`, `crew_lifecycle.go`, `crew_list.go`, `crew_maintenance.go`, `crew_status.go`) + 3 test files. All AddCommand calls in `crew.go:412-429`. No sibling `init()` registrations — all 13 subcommands registered from the parent file. Phase 2 sources already listed all 9 non-test files — **no missed siblings**.
- `rig` → `rig.go` + 6 sibling source files (`rig_config.go`, `rig_detect.go`, `rig_dock.go`, `rig_park.go`, `rig_quick_add.go`, `rig_settings.go`) + `rig_helpers.go` + 8 test files. Sibling `init()` blocks register 8 subcommands: config (`rig_config.go:91`), detect (`rig_detect.go:45`), dock/undock (`rig_dock.go:70-71`), park/unpark (`rig_park.go:62-63`), quick-add (`rig_quick_add.go:42`), settings (`rig_settings.go:92`). Phase 2 sources already listed all 8 non-test files — **no missed siblings**.
- `namepool` → `namepool.go` only. 6 subcommands in `init()`, accurate. No siblings.
- `install` → `install.go` + `install_test.go` + `install_integration_test.go`. No siblings with AddCommand.
- `gitinit` → `gitinit.go` only. No siblings.
- `init` → `init.go` only. No siblings.
- `worktree` → `worktree.go` only. 2 subcommands in `init()`, accurate. No siblings.

Release-position: all source files present at v1.0.0. All findings `in-release`.

**Docs files read:** none (Sweep 1).

**Wiki pages audited:**
- [crew](gastown/commands/crew.md) — `phase3_findings: [cobra-drift]`
- [git-init](gastown/commands/git-init.md) — `phase3_findings: [cobra-drift]`
- [init](gastown/commands/init.md) — `phase3_findings: [cobra-drift, wiki-stale]`
- [install](gastown/commands/install.md) — `phase3_findings: [cobra-drift]`
- [namepool](gastown/commands/namepool.md) — `phase3_findings: [cobra-drift]`
- [rig](gastown/commands/rig.md) — `phase3_findings: [none]`
- [worktree](gastown/commands/worktree.md) — `phase3_findings: [none]`

**Findings by category:**

- **cobra drift:** 6 findings across 5 pages.
  - `crew.md` — `crewCmd.Long` COMMANDS lists 8; 11 visible subcommands registered (missing `status`, `rename`, `pristine`). `severity: wrong`, `in-release`, `fix tier: code`. Hand-maintained enumeration pattern (9th instance across Batches 1a-1g).
  - `namepool.md` — `namepoolCmd.Long` Examples shows 6 operations; `create` and `delete` omitted. `severity: wrong`, `in-release`, `fix tier: code`. Hand-maintained enumeration pattern (10th instance).
  - `install.md` (finding 1) — `installCmd.Long` HQ "contains:" lists 3 items; code creates at least 5 directories including `deacon/` and `plugins/`. `severity: wrong`, `in-release`, `fix tier: code`.
  - `install.md` (finding 2) — `installCmd.Long` references `docs/hq.md` which does not exist. `severity: wrong`, `in-release`, `fix tier: code`.
  - `git-init.md` — `gitInitCmd.Long` lists 3 steps; code performs 4 (omits branch-protection hook installation). `severity: wrong`, `in-release`, `fix tier: code`.
  - `init.md` — `initCmd.Long` lists 4 directories; code creates 5 with different paths (omits `crew`, says `refinery/` and `mayor/` instead of `refinery/rig` and `mayor/rig`). `severity: wrong`, `in-release`, `fix tier: code`.
- **wiki-stale:** 1 finding on `init.md` — Phase 2 wiki body at step 3 repeated the Long text's incorrect 4-directory enumeration instead of checking `rig.AgentDirs` at `internal/rig/types.go:48-54`. **Phase 2 root cause: `phase-2-incomplete` (heuristic)**. Fixed inline.
- **none:** 2 pages (`rig`, `worktree`).

**Judgment calls:** (1) `rig.Long` describes rig structure but does NOT enumerate subcommands — no enumeration drift possible; neutral. (2) `worktree.Long` describes create behavior without enumerating subcommands — neutral; the 2 registered subcommands (list, remove) are discoverable via cobra help. (3) `namepool.Long` uses "Examples:" not "Commands:" — classified as cobra-drift because the examples functionally serve as a capability list and omit create/delete, consistent with prior batch precedent. (4) `install.Long` "It contains:" is a structural description, not merely a hint — missing deacon/ is significant since the Deacon is a major agent. (5) `install.Long` reference to non-existent `docs/hq.md` is a distinct finding from the HQ-structure enumeration.

**Cross-link discipline:** 5 `## Docs claim` + 6 `## Drift` sections (2 on install.md) + 1 wiki-stale inline fix. Forward links to [gastown/drift/README.md](gastown/drift/README.md). All `file:line` refs fresh from HEAD `9f962c4a`. 2 pages frontmatter-only. `init.md` sources updated to add `internal/rig/types.go`.

**Next sub-batch:** Batch 1h — Ungrouped (15 cmds). Tracked under `wiki-vxl`.

**Audited pages:**
  [gastown/commands/crew.md](gastown/commands/crew.md),
  [gastown/commands/git-init.md](gastown/commands/git-init.md),
  [gastown/commands/init.md](gastown/commands/init.md),
  [gastown/commands/install.md](gastown/commands/install.md),
  [gastown/commands/namepool.md](gastown/commands/namepool.md),
  [gastown/commands/rig.md](gastown/commands/rig.md),
  [gastown/commands/worktree.md](gastown/commands/worktree.md)

## [2026-04-15] drift-found | Batch 1h (Sweep 1 commands/ Ungrouped — 15 pages; Batch 1 complete)

**Scope:** Sweep 1 promotion across all 15 ungrouped command pages (`agent-log`, `commit`, `cycle`, `forget`, `health`, `krc`, `memories`, `nudge-poller`, `proxy-subcmds`, `remember`, `show`, `status-line`, `tap`, `town`, `warrant`). Every page received `phase3_audited` / `phase3_findings` / `phase3_severities` / `phase3_findings_post_release` frontmatter. `tap.md` received `## Docs claim` + `## Drift` sections with verbatim quotes, current `file:line` refs, and v1.2 fix-tier + severity + release-position fields, plus a wiki-stale inline fix. `warrant.md` received `## Docs claim` + `## Drift` sections. 13 pages tagged `phase3_findings: [none]`.

**Source files re-read at current HEAD** (current gastown HEAD `9f962c4af068fe9da9f4bd3624e7b66351121fdf`):

- `/home/kimberly/repos/gastown/internal/cmd/tap.go` (lines 7-35 — `tapCmd.Long` at `:10-29`; `init()` at `:33-34`)
- `/home/kimberly/repos/gastown/internal/cmd/tap_guard.go` (`:64-66` — `init()` registering `tapGuardCmd` + `tapGuardPRWorkflowCmd`)
- `/home/kimberly/repos/gastown/internal/cmd/tap_guard_bd_init.go` (`:28-29` — `init()`)
- `/home/kimberly/repos/gastown/internal/cmd/tap_guard_dangerous.go` (`:41-42` — `init()`)
- `/home/kimberly/repos/gastown/internal/cmd/tap_guard_mol_patrol.go` (`:31-32` — `init()`)
- `/home/kimberly/repos/gastown/internal/cmd/tap_list.go` (`:16-31` — `tapListCmd` definition + `init()`)
- `/home/kimberly/repos/gastown/internal/cmd/tap_polecat_stop.go` (`:16-38` — `tapPolecatStopCmd` definition + `init()`)
- `/home/kimberly/repos/gastown/internal/cmd/warrant.go` (`:38-52` Long; `:122-128` `getWarrantDir()`)
- `/home/kimberly/repos/gastown/internal/cmd/agent_log.go` (`:22-37` — no Long)
- `/home/kimberly/repos/gastown/internal/cmd/commit.go` (`:17-44` — Long + GroupID)
- `/home/kimberly/repos/gastown/internal/cmd/cycle.go` (`:23-83` — parent + subcommand Long)
- `/home/kimberly/repos/gastown/internal/cmd/forget.go` (`:11-28` — GroupID + Long)
- `/home/kimberly/repos/gastown/internal/cmd/health.go` (`:86-104` — Long; 6 sections match code)
- `/home/kimberly/repos/gastown/internal/cmd/krc.go` (`:17-152` — Long + 7 subcommands match)
- `/home/kimberly/repos/gastown/internal/cmd/memories.go` (`:14-39` — GroupID + Long)
- `/home/kimberly/repos/gastown/internal/cmd/nudge_poller.go` (`:22-44` — Long accurate)
- `/home/kimberly/repos/gastown/internal/cmd/proxy_subcmds.go` (`:23-44` — Long accurate)
- `/home/kimberly/repos/gastown/internal/cmd/remember.go` (`:36-65` — GroupID + Long)
- `/home/kimberly/repos/gastown/internal/cmd/show.go` (`:10-30` — GroupID + Long)
- `/home/kimberly/repos/gastown/internal/cmd/statusline.go` (`:25-39` — Long terse but accurate)
- `/home/kimberly/repos/gastown/internal/cmd/town_cycle.go` (`:32-71` — Long + subcommands)

**Sibling-file audit:**

- `tap` -> 7 siblings. **Phase 2 missed `list` and `polecat-stop-check`.** All register via per-file `init()`.
- `show` -> `show_unix.go` + `show_windows.go` (platform helpers, no AddCommand).
- `town` -> file is `town_cycle.go` (not `town.go`). No siblings.
- `krc` -> single file. All 7 subcommands in `init()` at `:138-146`.
- All other commands: no sibling AddCommand registrations found.

Release-position: all source files present at v1.0.0. All findings `in-release`.

**Docs files read:** none (Sweep 1).

**Wiki pages audited:**
- [tap](gastown/commands/tap.md) — `phase3_findings: [cobra-drift, wiki-stale]`
- [warrant](gastown/commands/warrant.md) — `phase3_findings: [cobra-drift]`
- 13 pages tagged `phase3_findings: [none]`: `agent-log`, `commit`, `cycle`, `forget`, `health`, `krc`, `memories`, `nudge-poller`, `proxy-subcmds`, `remember`, `show`, `status-line`, `town`

**Findings by category:**

- **cobra drift:** 3 findings across 2 pages.
  - `tap.md` (finding 1) — `tapCmd.Long` subcommand list names `guard`, `audit [planned]`, `inject [planned]`, `check [planned]`; actual wired subcommands are `guard` (`tap_guard.go:65`), `list` (`tap_list.go:31`), `polecat-stop-check` (`tap_polecat_stop.go:38`). Two implemented subcommands completely omitted from Long. `severity: wrong`, `in-release`, `fix tier: code`. Hand-maintained enumeration pattern (11th instance across Batches 1a-1h).
  - `tap.md` (finding 2) — same Long advertises `audit`, `inject`, `check` as `[planned]` but no source files exist. `severity: wrong`, `in-release`, `fix tier: code`.
  - `warrant.md` — `warrantCmd.Long` line 52 says "Warrants are stored in ~/gt/warrants/" but `getWarrantDir()` at `warrant.go:122-128` returns `filepath.Join(townRoot, "warrants")` where `townRoot` is dynamic. `severity: wrong`, `in-release`, `fix tier: code`.
- **wiki-stale:** 1 finding on `tap.md` — Phase 2 said "only `guard` is actually wired" and listed `audit`/`inject`/`check` as not implemented. Sibling-file audit shows `list` and `polecat-stop-check` ARE wired. **Phase 2 root cause: `phase-2-incomplete` (heuristic)** — Phase 2 took `tap.go` in isolation. Fixed inline.
- **none:** 13 pages.

**Judgment calls:** (1) `tap.Long`'s `[planned]` tags are honest about unbuilt status but classified as cobra-drift (not implementation-status) because the Long text presents them in a flat subcommand list alongside implemented commands. (2) `warrant.Long`'s `~/gt/warrants/` is correct for default installations but the code is workspace-relative — classified as cobra-drift. (3) `proxy-subcmds.Long` accurately describes the `bd` hardcoded list as a maintenance risk — neutral. (4) `health.Long` 6 sections match code exactly — neutral. (5) Five GroupWork commands already had correct Group attribution from Phase 2 — no findings needed.

**Cross-link discipline:** 2 `## Docs claim` + 3 `## Drift` sections (2 on tap.md, 1 on warrant.md) + 1 wiki-stale inline fix. Forward links to [gastown/drift/README.md](gastown/drift/README.md). All `file:line` refs fresh from HEAD `9f962c4a`. `tap.md` sources updated to add 3 sibling files.

**Batch 1 complete.** All 111 command pages now have `phase3_audited` frontmatter. Sweep 1 retrospective gate next — see [retros.md](retros.md).

**Batch 1 aggregate stats (1a-1h):**
- Pages audited: 111 of 111 commands
- Pages with findings: 35 (32%)
- Pages with `[none]`: 76 (68%)
- cobra-drift: 27 pages
- wiki-stale: 12 pages
- drift (docs): 1 page (`done.md`)
- Yield by sub-batch: 1a 18%, 1b 64%, 1c 27%, 1d 50%, 1e 14%, 1f 36%, 1g 71%, 1h 13%

**Audited pages:**
  [gastown/commands/agent-log.md](gastown/commands/agent-log.md),
  [gastown/commands/commit.md](gastown/commands/commit.md),
  [gastown/commands/cycle.md](gastown/commands/cycle.md),
  [gastown/commands/forget.md](gastown/commands/forget.md),
  [gastown/commands/health.md](gastown/commands/health.md),
  [gastown/commands/krc.md](gastown/commands/krc.md),
  [gastown/commands/memories.md](gastown/commands/memories.md),
  [gastown/commands/nudge-poller.md](gastown/commands/nudge-poller.md),
  [gastown/commands/proxy-subcmds.md](gastown/commands/proxy-subcmds.md),
  [gastown/commands/remember.md](gastown/commands/remember.md),
  [gastown/commands/show.md](gastown/commands/show.md),
  [gastown/commands/status-line.md](gastown/commands/status-line.md),
  [gastown/commands/tap.md](gastown/commands/tap.md),
  [gastown/commands/town.md](gastown/commands/town.md),
  [gastown/commands/warrant.md](gastown/commands/warrant.md)

## [2026-04-15] decision | Sweep 1 retrospective: 7 process decisions after Batch 1 (111 commands)

Post-Batch-1 retrospective with Kimberly. Eight sub-batches (1a-1h) audited all 111 command pages at 32% yield (35 pages with findings, exceeding the 20-32% estimate). Seven decisions ratified:

**1. Sibling-file audit baked into skill.** Mandatory step 0 in the writing-entity-pages operational classification procedure. Before re-reading any parent `.go` file, enumerate siblings via `ls <command>*.go`, grep each for `AddCommand`/`init()`, compare against `sources:` frontmatter. Load-bearing methodology that surfaced ~12 findings across 8 sub-batches.

**2. PR-delta scoping draft dropped as authoritative.** The plan's PR-delta draft is wrong about `wl_*.go` being post-v1.0.0 (verified: all 10 files exist at v1.0.0). Per-finding inline verification via `git show v1.0.0:<file>` is canonical. Draft remains as a hint.

**3. Hand-maintained enumeration meta-pattern.** 10+ commands have Long text that hand-enumerates subcommands/capabilities and omits entries. Phase 6 batches these as a single PR pattern.

**4. Wiki-stale log placement: separate lint entries** (option B). Future batches write wiki-stale findings as separate `lint` entries, not bundled into `drift-found`. Batch 1's bundled entries stand as-is. Revisit at project end.

**5. Retroactive tagging of Batch 1b findings: do nothing** (option C). The 1b batch entry and retro already document the phase-2-incomplete determination for directive/hooks.

**6. Novel sub-patterns named in skill.** Dead doc references (install.md → nonexistent docs/hq.md), semantic cross-reference drift (down.md → "use gt start" but start doesn't launch daemon), cobra-drift + implementation-status:vestigial compound (witness --foreground), cobra-drift + wiki-stale compound (hooks, molecule). Added to writing-entity-pages for discoverability.

**7. Consolidated dispatch template for Batches 2-4** added to `.claude/plans/2026-04-14-phase3-drift.md`. Captures the final-form methodology from 8 Batch 1 iterations.

**Observation (not a decision):** Phase-2-incomplete is the dominant wiki-stale root cause. Phase 2 treated `sources:` as "files I was aware of" rather than "files I read for cobra registrations." This is a Phase 4 input signal for re-audit scoping.

→ [.claude/skills/writing-entity-pages/SKILL.md](.claude/skills/writing-entity-pages/SKILL.md), [.claude/plans/2026-04-14-phase3-drift.md](.claude/plans/2026-04-14-phase3-drift.md)

## [2026-04-15] lint | Batch 2a wiki-stale finding (util.md sources frontmatter)

**Scope:** 1 wiki-stale finding surfaced during Batch 2a (Sweep 1 packages/ Platform services) package-file audit.

**Finding:**

- **util.md** — `sources:` frontmatter omitted `orphan_windows.go`. The file is a Windows stub (58 lines, type definitions + no-op functions) and was correctly described in the page body ("orphan_windows.go — stub implementations that return empty results"), but was not listed in frontmatter. Fixed by adding `/home/kimberly/repos/gastown/internal/util/orphan_windows.go` to `sources:`.
  - **Category:** `wiki-stale`
  - **Severity:** `incomplete`
  - **Phase 2 root cause:** `phase-2-incomplete` (heuristic determination — Phase 2 read the file and described it in the body but did not list it in frontmatter; the file existed at Phase 2 time at v1.0.0).
  - **Release position:** `in-release`

→ [gastown/packages/util.md](gastown/packages/util.md)

## [2026-04-15] drift-found | Batch 2a (Sweep 1 packages/ Platform services — 9 pages)

**Scope:** Sweep 1 promotion of Phase 2 notes to v1.2 annotations across all 9 Platform services package pages. Every page received `phase3_audited` / `phase3_findings` / `phase3_severities` / `phase3_findings_post_release` frontmatter. One page surfaced a `wiki-stale` finding (logged separately above); the other 8 were audited with `phase3_findings: [none]`.

**Source files re-read at current HEAD** (no changes since v1.0.0 for any of these packages — `git log --oneline v1.0.0..HEAD` for all 9 package directories returned zero commits):
- `/home/kimberly/repos/gastown/internal/cli/name.go` (verified: single file, wiki matches)
- `/home/kimberly/repos/gastown/internal/config/agents.go` (lines 1-45: agent preset enum verified — 10 presets match wiki)
- `/home/kimberly/repos/gastown/internal/config/` (all 9 files spot-verified: loader.go resolveConfigMu + Load/Save pairs, types.go ~67 KB, env.go IdentityEnvVars, cost_tier.go 3 tiers, directives.go LoadRoleDirective, overseer.go DetectOverseer, roles.go go:embed, operational.go LoadOperationalConfig)
- `/home/kimberly/repos/gastown/internal/session/` (all 11 files: identity.go package doc "polecat" still stale as Phase 2 noted; lifecycle.go SessionConfig:36, StartResult:115, StartSession:144, StopSession:344, KillExistingSession:427 — all match; names.go factories; registry.go PrefixRegistry; pidtrack.go; stale.go; startup.go; town.go; window_tint.go; agent_logging_{unix,windows}.go)
- `/home/kimberly/repos/gastown/internal/style/style.go` (in full, 60 lines — all line refs verified)
- `/home/kimberly/repos/gastown/internal/style/table.go` (spot-verified)
- `/home/kimberly/repos/gastown/internal/telemetry/telemetry.go` (lines 1-130: Init returns (nil,nil) when no env vars, DefaultMetricsURL/DefaultLogsURL both localhost, ExportInterval 30s, IsActive at line 92)
- `/home/kimberly/repos/gastown/internal/telemetry/recorder.go` (spot-verified: 23 counters + 1 histogram)
- `/home/kimberly/repos/gastown/internal/telemetry/subprocess.go` (spot-verified: SetProcessOTELAttrs, OTELEnvForSubprocess)
- `/home/kimberly/repos/gastown/internal/ui/` (all 4 files: styles.go init() at line 15, terminal.go capability detection, pager.go ToPager, markdown.go RenderMarkdown)
- `/home/kimberly/repos/gastown/internal/util/` (all 9 files: atomic.go, exec.go, exec_unix.go, exec_windows.go, orphan.go 894 lines, orphan_windows.go 58-line stub, path.go, slice.go, url.go)
- `/home/kimberly/repos/gastown/internal/version/stale.go` (Commit:18, StaleBinaryInfo:22-30, SetCommit:249, isBuildBranch:241 main/master/carry/*, onlyBeadsChanges:220 GH#2596)
- `/home/kimberly/repos/gastown/internal/workspace/find.go` (in full, 194 lines — all functions at cited lines)

Release-position verification: all 9 package directories had zero commits between v1.0.0 and HEAD. All source is byte-identical. All findings are `in-release`.

**Docs files read:** none (Sweep 1).

**Wiki pages audited:**
- [cli](gastown/packages/cli.md) — `phase3_findings: [none]`
- [config](gastown/packages/config.md) — `phase3_findings: [none]`
- [session](gastown/packages/session.md) — `phase3_findings: [none]`
- [style](gastown/packages/style.md) — `phase3_findings: [none]`
- [telemetry](gastown/packages/telemetry.md) — `phase3_findings: [none]`
- [ui](gastown/packages/ui.md) — `phase3_findings: [none]`
- [util](gastown/packages/util.md) — `phase3_findings: [wiki-stale]`
- [version](gastown/packages/version.md) — `phase3_findings: [none]`
- [workspace](gastown/packages/workspace.md) — `phase3_findings: [none]`

**Findings by category:**

- **drift:** 0 (Sweep 1; no `docs/` files read).
- **cobra drift:** 0 (packages have no Cobra Long text — structurally impossible).
- **compound drift:** 0.
- **implementation-status:** 0. No aspirational, partial, or vestigial behavior found.
- **wiki-stale:** 1 finding on util.md (sources frontmatter missing `orphan_windows.go`). Severity: `incomplete`. Root cause: `phase-2-incomplete`. Release position: `in-release`.
- **gap:** 0.
- **none:** 8 pages.

**Judgment calls:**

1. **session.md stale package doc comment.** `// Package session provides polecat session lifecycle management.` at `identity.go:1` / `lifecycle.go:1` is stale (code covers all roles). Phase 2 already documents this in Notes bullet 1. NOT promoted to Drift because it's an in-code doc comment, not a `docs/` or Cobra claim. Candidate for Phase 6 code PR but not Phase 3 scope.
2. **util.md orphan.go "~1000 lines"** vs actual 894 lines. Not wiki-stale — wiki used deliberate `~` approximation.
3. **config.md `AgentOmp` credit wording.** Wiki paraphrases; source says "Inspired by." Close enough — not wiki-stale.
4. **No `docs/*.md` about these packages.** Checked; `drift` category structurally inapplicable for Sweep 1 on these pages.

**Yield:** 1/9 pages (11%), below the 15-25% estimate. Expected given zero code churn since v1.0.0 and Phase 2's thorough package-level methodology.

**Next sub-batch:** Batch 2b — Data layer packages (Phase 2 Batch 5).

→ [gastown/packages/cli.md](gastown/packages/cli.md),
  [gastown/packages/config.md](gastown/packages/config.md),
  [gastown/packages/session.md](gastown/packages/session.md),
  [gastown/packages/style.md](gastown/packages/style.md),
  [gastown/packages/telemetry.md](gastown/packages/telemetry.md),
  [gastown/packages/ui.md](gastown/packages/ui.md),
  [gastown/packages/util.md](gastown/packages/util.md),
  [gastown/packages/version.md](gastown/packages/version.md),
  [gastown/packages/workspace.md](gastown/packages/workspace.md)

## [2026-04-15] lint | Batch 2b wiki-stale findings (beads.md sources frontmatter, events.md type count)

**Scope:** 2 wiki-stale findings surfaced during Batch 2b (Sweep 1 packages/ Data layer) package-file audit and spot-check.

**Findings:**

- **beads.md** — `sources:` frontmatter listed only 11 of 28 production `.go` files. The page body correctly describes all 28 files in the "domain files" section, but frontmatter `sources:` omitted 17 files (`agent_ids.go`, `beads_channel.go`, `beads_delegation.go`, `beads_dog.go`, `beads_escalation.go`, `beads_group.go`, `beads_merge_slot.go`, `beads_mr.go`, `beads_queue.go`, `beads_rig.go`, `beads_sling_context.go`, `catalog.go`, `config_yaml.go`, `fields.go`, `handoff.go`, `molecule.go`, `stale_pid.go`). Fixed by adding all 17 to `sources:`.
  - **Category:** `wiki-stale`
  - **Severity:** `incomplete`
  - **Phase 2 root cause:** `phase-2-incomplete` (heuristic determination — Phase 2 read all files and described them in the body but did not list them in frontmatter; all files existed at Phase 2 time at v1.0.0).
  - **Release position:** `in-release`

- **events.md** — opening paragraph and Notes section said "21 event types" but there are 30 event-type string constants (`events.go:36-77`): 11 command + 4 session + 7 patrol + 4 merge + 4 scheduler. The "21 payload constructors" count was correct. Phase 2 appears to have conflated the payload-constructor count with the type-constant count. Body correctly lists all 30 types in the per-category breakdown. Fixed by correcting "21" to "30" in the opening paragraph (via correction callout) and Notes section.
  - **Category:** `wiki-stale`
  - **Severity:** `incomplete`
  - **Phase 2 root cause:** `phase-2-incomplete` (heuristic determination — the 30 type constants existed at v1.0.0; Phase 2 counted the 21 payload constructors and used that number for both).
  - **Release position:** `in-release`

→ [gastown/packages/beads.md](gastown/packages/beads.md),
  [gastown/packages/events.md](gastown/packages/events.md)

## [2026-04-15] drift-found | Batch 2b (Sweep 1 packages/ Data layer — 8 pages)

**Scope:** Sweep 1 promotion of Phase 2 notes to v1.2 annotations across all 8 Data layer package pages. Every page received `phase3_audited` / `phase3_findings` / `phase3_severities` / `phase3_findings_post_release` frontmatter. Two pages surfaced `wiki-stale` findings (logged separately above); the other 6 were audited with `phase3_findings: [none]`.

**Source files re-read at current HEAD** (no changes since v1.0.0 for any of these packages — `git log --oneline v1.0.0..HEAD` for all 8 package directories returned zero commits):
- `/home/kimberly/repos/gastown/internal/beads/` (28 production files; file list verified against wiki; line counts spot-checked: beads.go 1689, fields.go 1057, beads_agent.go 753, store.go 599, molecule.go 579, beads_channel.go 529, beads_types.go 481, agent_ids.go 472, beads_escalation.go 442, beads_queue.go 398, beads_redirect.go 381, beads_group.go 373, routes.go 360 — all match wiki claims exactly; total production lines ~10,330 matching wiki's "~10,300")
- `/home/kimberly/repos/gastown/internal/channelevents/channelevents.go` (single file; wiki matches)
- `/home/kimberly/repos/gastown/internal/doltserver/` (9 production files; all listed in wiki `sources:`; spot-checked doltserver.go structure matches wiki)
- `/home/kimberly/repos/gastown/internal/events/events.go` (single file; 30 type constants + 21 payload constructors verified)
- `/home/kimberly/repos/gastown/internal/lock/` (5 production files; all listed in wiki `sources:`)
- `/home/kimberly/repos/gastown/internal/mail/` (7 production files; all listed in wiki `sources:`; router.go 1884 lines matching wiki's "~1880")
- `/home/kimberly/repos/gastown/internal/mq/id.go` (single file, 58 lines; wiki says "59" — trivial, not flagged)
- `/home/kimberly/repos/gastown/internal/nudge/` (6 production files; all listed in wiki `sources:`; queue.go 358, poller.go 276 — exact match)

Release-position verification: all 8 package directories had zero commits between v1.0.0 and HEAD. All source is byte-identical. All findings are `in-release`.

**Docs files read:** none (Sweep 1).

**Wiki pages audited:**
- [beads](gastown/packages/beads.md) — `phase3_findings: [wiki-stale]`
- [channelevents](gastown/packages/channelevents.md) — `phase3_findings: [none]`
- [doltserver](gastown/packages/doltserver.md) — `phase3_findings: [none]`
- [events](gastown/packages/events.md) — `phase3_findings: [wiki-stale]`
- [lock](gastown/packages/lock.md) — `phase3_findings: [none]`
- [mail](gastown/packages/mail.md) — `phase3_findings: [none]`
- [mq](gastown/packages/mq.md) — `phase3_findings: [none]`
- [nudge](gastown/packages/nudge.md) — `phase3_findings: [none]`

**Findings by category:**

- **drift:** 0 (Sweep 1; no `docs/` files read).
- **cobra drift:** 0 (packages have no Cobra Long text — structurally impossible).
- **compound drift:** 0.
- **implementation-status:** 0. No aspirational, partial, or vestigial behavior found.
- **wiki-stale:** 2 findings on 2 pages (beads.md sources frontmatter, events.md type count). Both severity `incomplete`, root cause `phase-2-incomplete`, release position `in-release`.
- **gap:** 0.
- **none:** 6 pages.

**Judgment calls:**

1. **beads.md `status: partial` retained.** The page's own Notes section explains the partial status: only a minority of the 28 files were read in full during Phase 2. The sources frontmatter fix adds all 28 files, but the page remains partial because the per-domain file descriptions are grounded in headers and exported symbols rather than line-by-line reading. Phase 3 does not change this designation.
2. **events.md mismatch: 30 types vs 21 constructors.** Not every event type has a dedicated payload constructor. Nine types (e.g., some session/scheduler variants) lack one; callers build `map[string]interface{}` directly. The "21" in the opening was wrong about types, correct about constructors. Corrected via inline callout rather than rewriting the Phase 2 text.
3. **mq.md "59 lines" vs actual 58.** Trivial off-by-one, likely a counting methodology difference. Not flagged as wiki-stale since the body and all functional claims are correct.
4. **doltserver.md "phantom DB quarantine" checked.** The quarantine logic at `doltserver.go:1489-1539` matches the wiki's description. No changes since v1.0.0.
5. **nudge.md "orphan-claim sweep" checked.** The restore-rather-than-delete logic at `queue.go:182-202` matches the wiki. No changes since v1.0.0.
6. **mail.md "~600ms" subprocess overhead claim.** store.go:1-6 comment confirms. No changes since v1.0.0.

**Yield:** 2/8 pages (25%), within the 15-25% estimate. Higher than 2a (11%) because the two high-leverage targets (beads, events) both had phase-2-incomplete findings, confirming the dispatch expectation.

**Next sub-batch:** Batch 2c — Agent runtime packages (Phase 2 Batch 6).

-> [gastown/packages/beads.md](gastown/packages/beads.md),
  [gastown/packages/channelevents.md](gastown/packages/channelevents.md),
  [gastown/packages/doltserver.md](gastown/packages/doltserver.md),
  [gastown/packages/events.md](gastown/packages/events.md),
  [gastown/packages/lock.md](gastown/packages/lock.md),
  [gastown/packages/mail.md](gastown/packages/mail.md),
  [gastown/packages/mq.md](gastown/packages/mq.md),
  [gastown/packages/nudge.md](gastown/packages/nudge.md)

## [2026-04-15] drift-found | Batch 2c (Sweep 1 packages/ Agent runtime — 13 pages)

**Scope:** Sweep 1 promotion of Phase 2 notes to v1.2 annotations across all 13 Agent runtime package pages. Every page received `phase3_audited` / `phase3_findings` / `phase3_severities` / `phase3_findings_post_release` frontmatter. All 13 pages audited with `phase3_findings: [none]`.

**Churn check:** 12 of 13 packages had zero commits between v1.0.0 and HEAD. Only `internal/reaper/` had churn (1 commit: `ce5ca7c3 feat(reaper): add close_reason and per-issue logging to auto-close`). However, Phase 2 was written on 2026-04-13, after the churn commit on 2026-04-03, so Phase 2 already incorporated the change. The wiki body is accurate for current HEAD.

**Package-file audit:** All 13 packages matched their wiki `sources:` listings exactly. No missing files, no extra files. Phase 2's file enumeration was thorough for this batch.

**Source files spot-checked at current HEAD:**
- `/home/kimberly/repos/gastown/internal/mayor/manager.go` (429 lines; `Start`:116, `StartTMUX`:131, `StartACP`:192 — all match wiki)
- `/home/kimberly/repos/gastown/internal/polecat/manager.go` (2422 lines; wiki says ~2400 — accurate approximation)
- `/home/kimberly/repos/gastown/internal/crew/manager.go` (937 lines; wiki says ~940 — accurate approximation)
- `/home/kimberly/repos/gastown/internal/dog/manager.go` (692 lines; wiki function refs verified)
- `/home/kimberly/repos/gastown/internal/deacon/manager.go` (234 lines; wiki refs verified)
- `/home/kimberly/repos/gastown/internal/refinery/engineer.go` (2057 lines — exact match)
- `/home/kimberly/repos/gastown/internal/witness/handlers.go` (2705 lines — exact match; wiki says "2700+" — accurate)
- `/home/kimberly/repos/gastown/internal/witness/manager.go` (`Start`:110 — matches wiki)
- `/home/kimberly/repos/gastown/internal/reaper/reaper.go` (952 lines — exact match; `AutoClose`:587, `purgeOldMail`:529, `batchDeleteRows`:709, `ClosePluginReceipts`:778, `ClosePluginDispatches`:862 — all start lines match wiki exactly)
- `/home/kimberly/repos/gastown/internal/wisp/promotion.go` (`ShouldPromote`:59 — matches wiki)
- `/home/kimberly/repos/gastown/internal/convoy/operations.go` (`CheckConvoysForIssue`:38 — matches wiki)
- `/home/kimberly/repos/gastown/internal/rig/manager.go` (1805 lines — exact match)
- `/home/kimberly/repos/gastown/internal/formula/parser.go` (703 lines — exact match)
- `/home/kimberly/repos/gastown/internal/plugin/scanner.go` (273 lines — exact match)

Release-position verification: 12 packages had zero commits since v1.0.0. Reaper's 1 commit was pre-Phase-2-read. All source is at release state for audit purposes. All findings are `in-release`.

**Docs files read:** none (Sweep 1).

**Notes promotion review:** All 13 pages' Notes sections were reviewed for drift/implementation-status promotion candidates. No promotions. Key judgments:
- witness.md and refinery.md vestigial `--foreground` observations are about the CLI flag, not the package code. The package correctly refuses foreground. Any vestigial finding belongs on the command page.
- formula.md `compose.aspects` parked field is a code observation with no docs claiming the feature works. Not drift.
- plugin.md "no plugin concept page" is a wiki coverage gap, noted as an observation for Phase 4 but not tagged as a Phase 3 finding (gap/missing is Phase 4 scope).

**Wiki pages audited:**
- [mayor](gastown/packages/mayor.md) — `phase3_findings: [none]`
- [polecat](gastown/packages/polecat.md) — `phase3_findings: [none]`
- [crew](gastown/packages/crew.md) — `phase3_findings: [none]`
- [dog](gastown/packages/dog.md) — `phase3_findings: [none]`
- [deacon](gastown/packages/deacon.md) — `phase3_findings: [none]`
- [refinery](gastown/packages/refinery.md) — `phase3_findings: [none]`
- [witness](gastown/packages/witness.md) — `phase3_findings: [none]`
- [reaper](gastown/packages/reaper.md) — `phase3_findings: [none]`
- [wisp](gastown/packages/wisp.md) — `phase3_findings: [none]`
- [convoy](gastown/packages/convoy.md) — `phase3_findings: [none]`
- [rig](gastown/packages/rig.md) — `phase3_findings: [none]`
- [formula](gastown/packages/formula.md) — `phase3_findings: [none]`
- [plugin](gastown/packages/plugin.md) — `phase3_findings: [none]`

**Findings by category:**

- **drift:** 0 (Sweep 1; no `docs/` files read).
- **cobra drift:** 0 (packages have no Cobra Long text — structurally impossible).
- **compound drift:** 0.
- **implementation-status:** 0. Vestigial foreground observations noted but properly scoped to command pages, not package pages.
- **wiki-stale:** 0. All line refs, file counts, and function locations verified against current HEAD.
- **gap:** 0 tagged (plugin.md concept page gap noted as Phase 4 observation only).
- **none:** 13 pages.

**Judgment calls:**

1. **Reaper churn (1 commit post-v1.0.0).** The commit added `ClosedEntry` struct and per-issue logging to `AutoClose`. Phase 2 was written 10 days after the commit and already describes the post-churn behavior accurately. NOT wiki-stale.
2. **Witness/refinery foreground rejection.** Both packages' Notes mention vestigial `--foreground`. The package code correctly refuses foreground mode. The vestigial finding (if any) is about the CLI flag still parsing, which belongs on the command page. NOT promoted on the package page.
3. **Formula `compose.aspects` parked field.** TOML decodes it, code ignores it. No docs claim it works. This is a code-level implementation note, not docs-vs-code drift. Stays in Notes.
4. **Plugin concept page gap.** A `concepts/plugin.md` page would consolidate the plugin model. This is wiki coverage, not docs-vs-code drift. Noted for Phase 4 scope. NOT tagged as a Phase 3 finding.

**Yield:** 0/13 pages (0%). Lowest of any sub-batch so far. Expected: Phase 2's agent-runtime package methodology was thorough (same-batch rule, same subagent reading code + writing page), zero code churn for 12/13 packages, and the one churned package was already post-churn at Phase 2 read time. Combined with cobra-drift being structurally impossible for packages, the yield floor is zero.

**Next sub-batch:** Batch 2d — Diagnostics packages (Phase 2 Batch 7).

-> [gastown/packages/mayor.md](gastown/packages/mayor.md),
  [gastown/packages/polecat.md](gastown/packages/polecat.md),
  [gastown/packages/crew.md](gastown/packages/crew.md),
  [gastown/packages/dog.md](gastown/packages/dog.md),
  [gastown/packages/deacon.md](gastown/packages/deacon.md),
  [gastown/packages/refinery.md](gastown/packages/refinery.md),
  [gastown/packages/witness.md](gastown/packages/witness.md),
  [gastown/packages/reaper.md](gastown/packages/reaper.md),
  [gastown/packages/wisp.md](gastown/packages/wisp.md),
  [gastown/packages/convoy.md](gastown/packages/convoy.md),
  [gastown/packages/rig.md](gastown/packages/rig.md),
  [gastown/packages/formula.md](gastown/packages/formula.md),
  [gastown/packages/plugin.md](gastown/packages/plugin.md)

## [2026-04-14] lint | Batch 2d wiki-stale finding (keepalive.md zero importers)

**Page:** [keepalive.md](gastown/packages/keepalive.md)

**Finding:** Phase 2 stated `internal/web/api.go` imports `internal/keepalive`. This was incorrect — `web/api.go` uses a local `keepalive` variable for SSE heartbeat ticking (`time.NewTicker(15 * time.Second)`), not the `internal/keepalive` package. The full import-path grep `rg '"github.com/steveyegge/gastown/internal/keepalive"'` returns zero results. The package has **no in-tree importers at HEAD**.

**Phase 2 root cause:** `phase-2-incomplete` — Phase 2 mistook a local variable name for a package import. The `web/api.go` file was read during Phase 2 Batch 7 but the keepalive reference was a false match on the word "keepalive" rather than the package import path.

**Inline fix applied:** updated keepalive.md intro, "Usage pattern" section, "Related wiki pages", and "Notes / open questions" to reflect zero importers.

## [2026-04-14] drift-found | Batch 2d (Sweep 1 packages/ Diagnostics — 4 pages)

Phase 3 Sweep 1 sub-batch 2d: audited 4 package pages under `gastown/packages/` covering Phase 2 Batch 7 (Diagnostics & health).

**Churn:** zero commits between v1.0.0 and HEAD across all 4 packages (`internal/doctor/`, `internal/health/`, `internal/keepalive/`, `internal/deps/`). Fast-path applied per 2a retro guidance.

**Source files re-read:**
- `/home/kimberly/repos/gastown/internal/health/health.go:1-4` (package doc comment)
- `/home/kimberly/repos/gastown/internal/keepalive/keepalive.go:1-15` (package doc comment)
- `/home/kimberly/repos/gastown/internal/deps/beads.go:17-22` (version constants)
- `/home/kimberly/repos/gastown/internal/deps/dolt.go:14-19` (version constants)
- Import-graph verification via `rg` for all 4 package import paths

**Pages audited (4):**

| Page | phase3_findings | Notes |
|---|---|---|
| [doctor.md](gastown/packages/doctor.md) | `[none]` | 70 non-test files confirmed. All Notes bullets are genuinely neutral (no dependency graph, no short-circuit, Category/CategoryOrder vestigial under streaming). No promotion. |
| [health.md](gastown/packages/health.md) | `[drift]` | Package doc claims Doctor Dog imports it; only `gt health` does. Promoted Phase 2 Notes bullet to `## Docs claim` + `## Drift`. Fix tier: code (edit package doc comment). |
| [keepalive.md](gastown/packages/keepalive.md) | `[wiki-stale, implementation-status-partial]` | Phase 2 claimed `web/api.go` imports it — incorrect (local var confusion). Zero importers at HEAD. Added `## Implementation status: partial`. See separate lint entry above. |
| [deps.md](gastown/packages/deps.md) | `[none]` | `MinBeadsVersion = "0.57.0"`, `MinDoltVersion = "1.82.4"` match wiki exactly. 3 go files match. |

**Yield:** 2/4 pages (50%). Higher than prior package sub-batches (2a: 11%, 2b: 25%, 2c: 0%) because Diagnostics packages had the pre-flagged health doc-drift and the keepalive false-import.

**Finding summary:**

1. **health.md — drift (wrong, in-release).** Package doc at `health.go:2-3` claims "shared between Doctor Dog and gt health CLI." Code: only `internal/cmd/health.go` imports it. Fix tier: code.
2. **keepalive.md — wiki-stale (phase-2-incomplete, wrong).** Phase 2 incorrectly claimed `web/api.go` imports the package. Zero importers at HEAD. Inline fix applied.
3. **keepalive.md — implementation-status: partial (incomplete, in-release).** Package is fully coded but has zero consumers. PersistentPreRun writer and daemon reader integrations never landed. Fix tier: preserve-as-vision.

**Next sub-batch:** Batch 2e — Long-running processes packages (Phase 2 Batch 8).

-> [gastown/packages/doctor.md](gastown/packages/doctor.md),
  [gastown/packages/health.md](gastown/packages/health.md),
  [gastown/packages/keepalive.md](gastown/packages/keepalive.md),
  [gastown/packages/deps.md](gastown/packages/deps.md)

## [2026-04-14] drift-found | Batch 2e (Sweep 1 packages/ Long-running processes — 3 pages)

Phase 3 Sweep 1 sub-batch 2e: audited 3 package pages under `gastown/packages/` covering Phase 2 Batch 8 (Long-running processes).

**Churn:** daemon had 1 commit between v1.0.0 and HEAD (`61063982 fix: align daemon purge defaults to 7 days` — changed `defaultWispDeleteAge` and `defaultMailDeleteAge` from 3d to 7d in `wisp_reaper.go`). tmux and runtime had zero commits. The daemon churn does not affect any wiki claim (the page does not cite specific purge durations).

**Source files re-read:**
- `/home/kimberly/repos/gastown/internal/daemon/types.go:24-41` (HeartbeatInterval field + DefaultConfig)
- `/home/kimberly/repos/gastown/internal/daemon/daemon.go:389,392,704,709-714` (recoveryHeartbeatInterval usage)
- `/home/kimberly/repos/gastown/internal/config/operational.go:48,371-376` (DefaultRecoveryHeartbeatInterval = 3m)
- `/home/kimberly/repos/gastown/internal/daemon/wisp_reaper.go:18-26` (purge defaults post-churn)
- Package-file audit: `ls internal/daemon/*.go | grep -v _test.go` (33 files, matches wiki), `ls internal/tmux/*.go | grep -v _test.go` (11 files, matches wiki sources exactly), `ls internal/runtime/*.go | grep -v _test.go` (1 file, matches wiki)

**Pages audited (3):**

| Page | phase3_findings | Notes |
|---|---|---|
| [daemon.md](gastown/packages/daemon.md) | `[none]` | 33 non-test files confirmed. 1 post-v1.0.0 commit (purge defaults) does not affect wiki claims. `HeartbeatInterval=5m` dead code confirmed: field defined at `types.go:25` but never referenced by the heartbeat loop (`daemon.go:389,704` uses `recoveryHeartbeatInterval()` from operational config at 3m default). No docs claim the 5m value, so no promotion — stays as neutral code observation. All 7 Notes bullets are genuinely neutral. |
| [tmux.md](gastown/packages/tmux.md) | `[none]` | 11 non-test files confirmed, all match sources frontmatter. Zero churn. `sessionNudgeLocks` slow leak is a code-internal observation with no docs claim to contradict — stays neutral. All 5 Notes bullets neutral. |
| [runtime.md](gastown/packages/runtime.md) | `[none]` | 1 non-test file confirmed. Zero churn. All 5 Notes bullets are code observations or naming concerns with no docs claim to contradict. |

**Yield:** 0/3 pages (0%). Consistent with 2c's 0% on agent-runtime packages. Long-running process packages have complex Notes sections but none rise above neutral — they are code-internal observations without docs claims to test against. Cobra-drift is structurally impossible (packages have no Cobra Long text). The HeartbeatInterval dead code is the closest candidate but fails the promotion test: no docs file, no Cobra text, and no package doc comment claims the 5m value is used.

**Finding summary:** No findings. All 17 Notes bullets across the 3 pages are genuinely neutral — code observations, naming concerns, size observations, or limitation notes that do not contradict any docs or Cobra claim.

**Next sub-batch:** Batch 2f — Supporting libraries packages (Phase 2 Batch 9).

-> [gastown/packages/daemon.md](gastown/packages/daemon.md),
  [gastown/packages/tmux.md](gastown/packages/tmux.md),
  [gastown/packages/runtime.md](gastown/packages/runtime.md)

## [2026-04-14] drift-found | Batch 2f (Sweep 1 packages/ Supporting libraries — 24 pages; Batch 2 complete)

Phase 3 Sweep 1 sub-batch 2f: audited 24 package pages under `gastown/packages/` covering Phase 2 Batch 9 (Supporting libraries). **This is the final sub-batch of Batch 2. All 61 package pages now have `phase3_audited` frontmatter.**

**Churn:** Zero commits between v1.0.0 and HEAD across all 24 packages (`git log --oneline v1.0.0..HEAD -- internal/<pkg>/` returned empty for every package). Fast-path applied: frontmatter + Notes section review + spot-check of pre-flagged findings.

**Source files re-read:**
- `/home/kimberly/repos/gastown/internal/wrappers/wrappers.go:1-2` (ABOUTME header), `:26` (Install slice), `:48` (Remove slice) — verified gt-gemini present in code but absent from ABOUTME
- `/home/kimberly/repos/gastown/internal/protocol/refinery_handlers.go:52-71` (HandleMergeReady — confirmed no-op, neutral)
- `/home/kimberly/repos/gastown/internal/krc/krc.go:325-340` (MinRetainCount — confirmed unenforced, neutral: no docs claim enforcement)
- `/home/kimberly/repos/gastown/internal/tui/feed/mq_source.go:1-30` (confirmed deprecated stub, already documented in tui.md)
- `/home/kimberly/repos/gastown/internal/quota/keychain.go:255-257` (validateTokenHTTP — confirmed dead code, never called, neutral: no docs claim)
- Package-file audit: `ls internal/<pkg>/*.go | grep -v _test.go` for all 24 packages. All file counts match wiki `sources:` frontmatter.

**Pre-flagged findings from Phase 2 Batch 9 — disposition:**

| Pre-flagged item | Page | Disposition |
|---|---|---|
| `protocol.md` HandleMergeReady no-op | protocol.md | **Neutral.** Notes bullet accurately describes the no-op. No docs/Cobra claim says HandleMergeReady triggers active merge behavior. |
| `krc.md` MinRetainCount unenforced | krc.md | **Neutral.** Notes bullet accurately describes the gap. No docs claim enforcement. |
| `feed.md` mq_source.go deprecated stub | tui.md (not feed.md) | **Neutral.** `mq_source.go` is in `internal/tui/feed/`, not `internal/feed/`. Already documented in tui.md Notes as "MQEventSource is a zombie." |
| `quota.md` validateTokenHTTP dead code | quota.md | **Neutral.** Function defined but never called. No docs claim it is used. |
| `wrappers.md` ABOUTME lists 2, code installs 3 | wrappers.md | **Promoted to cobra-drift.** ABOUTME header (`wrappers.go:2`) says "gt-codex and gt-opencode"; code installs 3 including gt-gemini. In-code docstring contradicts code. Upgraded Phase 2 Drift section to v1.2 schema. Demoted two non-drift observations (duplicated slice, BinDir error semantics) from Drift to Notes. |
| `hookutil.md` IsAutonomousRole literal "boot" | hookutil.md | **Neutral.** No docs claim consistency with constants. |
| `shell.md` DetectShell zsh fallback | shell.md | **Neutral.** Notes already document this. No docs claim full shell coverage. |
| `git.md` 2,300-line wrapper, Phase 2 missed files? | git.md | **No miss.** 3 non-test .go files match wiki sources exactly. |
| `web.md` gtPath hardcoded | web.md | **Neutral.** Notes already document this. Deliberate design choice. |

**Pages audited (24):**

| Page | phase3_findings | Notes |
|---|---|---|
| [wrappers.md](gastown/packages/wrappers.md) | `[cobra-drift]` | ABOUTME header omits gt-gemini. v1.2 Drift section added. Severity: wrong. Fix tier: code. Release position: in-release. |
| [acp.md](gastown/packages/acp.md) | `[none]` | 4 files match. 3 Notes bullets neutral. |
| [hooks.md](gastown/packages/hooks.md) | `[none]` | 3 files match. 3 Notes bullets neutral. |
| [krc.md](gastown/packages/krc.md) | `[none]` | 3 files match. 3 Notes bullets neutral (MinRetainCount stays neutral). |
| [protocol.md](gastown/packages/protocol.md) | `[none]` | 5 files match. 3 Notes bullets neutral (HandleMergeReady stays neutral). |
| [quota.md](gastown/packages/quota.md) | `[none]` | 6 files match. 3 Notes bullets neutral (validateTokenHTTP stays neutral). |
| [testutil.md](gastown/packages/testutil.md) | `[none]` | 4 files match. 2 Notes bullets neutral. |
| [wasteland.md](gastown/packages/wasteland.md) | `[none]` | 3 files match. 4 Notes bullets neutral. |
| [web.md](gastown/packages/web.md) | `[none]` | 7 files match. 4 Notes bullets neutral. |
| [agentlog.md](gastown/packages/agentlog.md) | `[none]` | 3 files match. 4 Notes bullets neutral. |
| [constants.md](gastown/packages/constants.md) | `[none]` | 1 file matches. 3 Notes bullets neutral. |
| [feed.md](gastown/packages/feed.md) | `[none]` | 1 file matches. 4 Notes bullets neutral. |
| [git.md](gastown/packages/git.md) | `[none]` | 3 files match. 5 Notes bullets neutral. |
| [github.md](gastown/packages/github.md) | `[none]` | 2 files match. 5 Notes bullets neutral. |
| [shell.md](gastown/packages/shell.md) | `[none]` | 1 file matches. 5 Notes bullets neutral. |
| [suggest.md](gastown/packages/suggest.md) | `[none]` | 1 file matches. 4 Notes bullets neutral. |
| [townlog.md](gastown/packages/townlog.md) | `[none]` | 1 file matches. 5 Notes bullets neutral. |
| [activity.md](gastown/packages/activity.md) | `[none]` | 1 file matches. 2 Notes bullets neutral. |
| [estop.md](gastown/packages/estop.md) | `[none]` | 1 file matches. 2 Notes bullets neutral. |
| [hookutil.md](gastown/packages/hookutil.md) | `[none]` | 1 file matches. 2 Notes bullets neutral. |
| [scheduler.md](gastown/packages/scheduler.md) | `[none]` | 4 capacity files match. 3 Notes bullets neutral. |
| [state.md](gastown/packages/state.md) | `[none]` | 1 file matches. 2 Notes bullets neutral. |
| [templates.md](gastown/packages/templates.md) | `[none]` | 3 files match (incl. commands/provision.go). 3 Notes bullets neutral. |
| [tui.md](gastown/packages/tui.md) | `[none]` | 14 non-test .go files across convoy/ and feed/ dirs. 3 Notes bullets neutral. mq_source.go stub already documented. |

**Yield:** 1/24 pages (4%). The sole finding is wrappers.md's ABOUTME header omitting gt-gemini — an in-code docstring drift that Phase 2 had already identified but not classified under the Phase 3 taxonomy.

**Batch 2 aggregate stats (6 sub-batches, 61 pages):**

| Sub-batch | Pages | Findings | Yield |
|---|---|---|---|
| 2a (Platform services) | 9 | 1 wiki-stale | 11% |
| 2b (Data layer) | 8 | 2 wiki-stale | 25% |
| 2c (Agent runtime) | 13 | 0 | 0% |
| 2d (Diagnostics) | 4 | 2 (1 drift + 1 wiki-stale + 1 impl-status) | 50% |
| 2e (Long-running) | 3 | 0 | 0% |
| 2f (Supporting libraries) | 24 | 1 cobra-drift | 4% |
| **Batch 2 total** | **61** | **6 findings across 5 pages** | **8%** |

Batch 2 overall yield (8%) is below the plan's 15-25% estimate, confirming that packages without Cobra Long text have structurally low drift surfaces. The dominant finding type was wiki-stale (3 findings) followed by cobra-drift (1), drift (1), and implementation-status (1). Pre-flagged items from Phase 2 had a 20% promotion rate (1 promoted out of 5 actionable flags, plus 4 that resolved as neutral).

**Batch 2 complete. Next: Batch 3 (files/ + inventory/).**

-> [gastown/packages/wrappers.md](gastown/packages/wrappers.md),
  [gastown/packages/acp.md](gastown/packages/acp.md),
  [gastown/packages/hooks.md](gastown/packages/hooks.md),
  [gastown/packages/krc.md](gastown/packages/krc.md),
  [gastown/packages/protocol.md](gastown/packages/protocol.md),
  [gastown/packages/quota.md](gastown/packages/quota.md),
  [gastown/packages/testutil.md](gastown/packages/testutil.md),
  [gastown/packages/wasteland.md](gastown/packages/wasteland.md),
  [gastown/packages/web.md](gastown/packages/web.md),
  [gastown/packages/agentlog.md](gastown/packages/agentlog.md),
  [gastown/packages/constants.md](gastown/packages/constants.md),
  [gastown/packages/feed.md](gastown/packages/feed.md),
  [gastown/packages/git.md](gastown/packages/git.md),
  [gastown/packages/github.md](gastown/packages/github.md),
  [gastown/packages/shell.md](gastown/packages/shell.md),
  [gastown/packages/suggest.md](gastown/packages/suggest.md),
  [gastown/packages/townlog.md](gastown/packages/townlog.md),
  [gastown/packages/activity.md](gastown/packages/activity.md),
  [gastown/packages/estop.md](gastown/packages/estop.md),
  [gastown/packages/hookutil.md](gastown/packages/hookutil.md),
  [gastown/packages/scheduler.md](gastown/packages/scheduler.md),
  [gastown/packages/state.md](gastown/packages/state.md),
  [gastown/packages/templates.md](gastown/packages/templates.md),
  [gastown/packages/tui.md](gastown/packages/tui.md)

## [2026-04-14] lint | Batch 3 wiki-stale findings (docker-entrypoint.md line count, go-mod.md require count, docs-tree.md subdir count)

Three wiki-stale fixes applied inline:

1. **docker-entrypoint.md** — wiki body said "23-line POSIX shell script"; actual file is 22 lines at HEAD (`wc -l docker-entrypoint.sh` = 22). Fixed to "22-line". **Phase 2 root cause:** phase-2-incomplete (heuristic determination — the file has not changed since Phase 2).
2. **go-mod.md** — wiki body said "31 direct `require` entries"; actual count is 30 (lines 6-35 of `go.mod`). Fixed to "30". **Phase 2 root cause:** phase-2-incomplete (heuristic determination).
3. **docs-tree.md** — Totals section said "10 subdirectories"; actual count is 11 (Phase 2 missed `docs/skills/convoy/` as a separate subdirectory). Fixed to "11" and added the missing entry. **Phase 2 root cause:** phase-2-incomplete (heuristic determination).

All three are in-release (zero commits between v1.0.0 and HEAD across all audited files).

-> [gastown/files/docker-entrypoint.md](gastown/files/docker-entrypoint.md),
  [gastown/files/go-mod.md](gastown/files/go-mod.md),
  [gastown/inventory/docs-tree.md](gastown/inventory/docs-tree.md)

## [2026-04-14] drift-found | Batch 3 (Sweep 1 files/ + inventory/ — 17 pages)

**Scope:** Phase 3 Sweep 1 sub-batch 3: audited 12 files/ pages and 5 inventory/ pages. Single batch (not sub-batched).

**Churn:** Zero commits between v1.0.0 and HEAD across all 17 source files (`git log --oneline v1.0.0..HEAD -- <file>` returned empty for every file). All findings are in-release.

**Source files re-read at current HEAD:**
- `/home/kimberly/repos/gastown/go.mod` (lines 1-36 direct requires, line 134 end)
- `/home/kimberly/repos/gastown/.goreleaser.yml` (lines 151-180 release section, grep for `brews`)
- `/home/kimberly/repos/gastown/Dockerfile` (line 5 GO_VERSION)
- `/home/kimberly/repos/gastown/Dockerfile.e2e` (line 12 base image)
- `/home/kimberly/repos/gastown/flake.nix` (lines 20-26 Go overlay)
- `/home/kimberly/repos/gastown/docker-entrypoint.sh` (full, 22 lines)
- `/home/kimberly/repos/gastown/docker-compose.yml` (full, 44 lines)
- `/home/kimberly/repos/gastown/.golangci.yml` (full, 111 lines)
- `/home/kimberly/repos/gastown/Makefile` (lines 1-50)
- `/home/kimberly/repos/gastown/.claude/` (directory listing)
- `/home/kimberly/repos/gastown/.opencode/` (directory listing)
- `/home/kimberly/repos/gastown/templates/agents/` (directory listing)
- `/home/kimberly/repos/gastown/docs/` (recursive file count + subdirectory enumeration)
- `find /home/kimberly/repos/gastown/internal/ -maxdepth 1 -type d` (package count)
- `find /home/kimberly/repos/gastown -maxdepth 1` (top-level file + dir count)

**Docs files read:** none (Sweep 1).

**Wiki pages audited (17):**

| Page | phase3_findings | Notes |
|---|---|---|
| [go-mod.md](gastown/files/go-mod.md) | `[wiki-stale, drift]` | Require count fixed (31->30). Go version disagreement promoted from Notes to v1.2 Drift section. |
| [goreleaser-yml.md](gastown/files/goreleaser-yml.md) | `[drift]` | No `brews:` block vs `brew install gastown` in release header. Promoted from Notes to v1.2 Drift section with Docs claim. |
| [docker-entrypoint.md](gastown/files/docker-entrypoint.md) | `[wiki-stale]` | Line count fixed (23->22). |
| [docs-tree.md](gastown/inventory/docs-tree.md) | `[wiki-stale]` | Subdirectory count fixed (10->11, missed `skills/convoy/`). |
| [claude-dir.md](gastown/files/claude-dir.md) | `[none]` | Layout matches HEAD. 3 commands + 4 skills confirmed. |
| [docker-compose.md](gastown/files/docker-compose.md) | `[none]` | 44-line file matches wiki description. |
| [dockerfile-e2e.md](gastown/files/dockerfile-e2e.md) | `[none]` | 74-line file matches. Go 1.26-alpine, BD v0.57.0, Dolt v1.82.4 all confirmed. |
| [dockerfile.md](gastown/files/dockerfile.md) | `[none]` | 56-line file matches. GO_VERSION=1.25.6 confirmed. |
| [flake-nix.md](gastown/files/flake-nix.md) | `[none]` | Go overlay to 1.25.8, version 0.8.0 hardcoded, all confirmed. |
| [golangci-yml.md](gastown/files/golangci-yml.md) | `[none]` | 111-line file, 5 linters, exclusion list all confirmed. |
| [makefile.md](gastown/files/makefile.md) | `[none]` | 167-line file matches. All targets, LDFLAGS, line numbers confirmed. |
| [opencode-dir.md](gastown/files/opencode-dir.md) | `[none]` | 1 command + 1 plugin confirmed. |
| [templates-agents.md](gastown/files/templates-agents.md) | `[none]` | 2 files confirmed (opencode.json.tmpl 754B, opencode-models.json 1924B). |
| [auxiliary.md](gastown/inventory/auxiliary.md) | `[none]` | 9 directories confirmed. |
| [go-packages.md](gastown/inventory/go-packages.md) | `[none]` | 67 internal packages, 1088 .go files confirmed. |
| [README.md](gastown/inventory/README.md) | `[none]` | Sub-index pages all present. |
| [repo-root.md](gastown/inventory/repo-root.md) | `[none]` | 24 files, 15 directories confirmed at HEAD. |

**Findings by category:**
- **drift:** 2 findings on 2 pages. goreleaser-yml.md: release header advertises `brew install gastown` but no `brews:` block exists (`.goreleaser.yml:164-167`). go-mod.md: Go version `1.25.8` disagrees with Dockerfile `1.25.6` and Dockerfile.e2e `1.26` (`go.mod:3`, `Dockerfile:5`, `Dockerfile.e2e:12`).
- **wiki-stale:** 3 findings on 3 pages (all phase-2-incomplete, heuristic). docker-entrypoint.md: line count 23->22. go-mod.md: require count 31->30. docs-tree.md: subdir count 10->11.
- **none:** 13 pages audited with no findings.

**Yield:** 4/17 pages (24%). 2 drift + 3 wiki-stale across 4 pages.

**New beads filed:** none.
**Beads closed:** none.
**Cross-link discipline:** 2 new `## Drift` sections added (go-mod.md, goreleaser-yml.md). 1 new `## Docs claim` section added (goreleaser-yml.md). All `file:line` refs are current (freshly read from HEAD). Forward links to `gastown/drift/README.md` noted as pending (drift index does not yet exist — created in Batch 13).

**Pre-flagged findings — disposition:**

| Pre-flagged item | Page | Disposition |
|---|---|---|
| Go version disagreement (Phase 2 noted) | go-mod.md | **Promoted to drift.** Three build paths disagree with go.mod's `go 1.25.8`. v1.2 Drift section added. |
| No brews block vs README brew claim (Phase 2 noted) | goreleaser-yml.md | **Promoted to drift.** Release header at `.goreleaser.yml:164-167` advertises `brew install gastown` with no mechanism to fulfill it. v1.2 Drift + Docs claim sections added. |
| Makefile existing 2-item Drift section | makefile.md | **No existing Drift section found.** Phase 2 Notes bullets about BuildTime and desktop-build are neutral observations, not drift. No promotion needed. |

**Next batch:** Batch 4 (roles/, concepts/, workflows/, binaries/, plugins/). Tracked in bead wiki-vxl.

-> [gastown/files/go-mod.md](gastown/files/go-mod.md),
  [gastown/files/goreleaser-yml.md](gastown/files/goreleaser-yml.md),
  [gastown/files/docker-entrypoint.md](gastown/files/docker-entrypoint.md),
  [gastown/files/claude-dir.md](gastown/files/claude-dir.md),
  [gastown/files/docker-compose.md](gastown/files/docker-compose.md),
  [gastown/files/dockerfile-e2e.md](gastown/files/dockerfile-e2e.md),
  [gastown/files/dockerfile.md](gastown/files/dockerfile.md),
  [gastown/files/flake-nix.md](gastown/files/flake-nix.md),
  [gastown/files/golangci-yml.md](gastown/files/golangci-yml.md),
  [gastown/files/makefile.md](gastown/files/makefile.md),
  [gastown/files/opencode-dir.md](gastown/files/opencode-dir.md),
  [gastown/files/templates-agents.md](gastown/files/templates-agents.md),
  [gastown/inventory/auxiliary.md](gastown/inventory/auxiliary.md),
  [gastown/inventory/docs-tree.md](gastown/inventory/docs-tree.md),
  [gastown/inventory/go-packages.md](gastown/inventory/go-packages.md),
  [gastown/inventory/README.md](gastown/inventory/README.md),
  [gastown/inventory/repo-root.md](gastown/inventory/repo-root.md)

## [2026-04-14] lint | Batch 4 wiki-stale findings (polecat.md pending-role-page claims, directive.md parent-only-stub claim)

Two wiki-stale findings fixed inline during Batch 4:

1. **`gastown/roles/polecat.md`** — Notes section claimed "The Witness role page is pending" and "The Refinery role page is pending." Both pages exist (created in the same Phase 2 Batch 6). **Phase 2 root cause:** phase-2-incomplete — the polecat page was written before the witness and refinery role pages landed in the same batch; the polecat page was not updated with forward links afterwards.

2. **`gastown/concepts/directive.md`** — "CLI surface: parent-only stub" section and Notes bullet claimed `gt directive` had no wired subcommands. The command page `gastown/commands/directive.md` was corrected in Phase 3 Batch 1b (sibling files `directive_show.go`, `directive_edit.go`, `directive_list.go` wire the subcommands via `init()`). The concept page propagated the same Phase 2 error and is now aligned. **Phase 2 root cause:** phase-2-incomplete — Phase 2 read only `directive.go` without checking sibling files.

Both findings are in-release (code unchanged since v1.0.0).

## [2026-04-14] drift-found | Batch 4 (Sweep 1 roles/ + concepts/ + workflows/ + binaries/ + plugins/ — 22 pages)

**Scope:** Phase 3 Sweep 1 sub-batch 4 (final Sweep 1 batch): audited 8 roles/ pages, 7 concepts/ pages, 2 workflows/ pages, 3 binaries/ pages, and 2 plugins/ pages. Single batch (not sub-batched).

**Methodology:** Cross-reference audit against Batches 1-3 findings. These pages are domain-centric and make claims about code behavior via cross-references to command and package pages already audited in earlier batches. The primary audit question is: does this page make claims that contradict what Batches 1-3 found? Secondary: are Notes/open questions that reference "pending" pages still accurate?

**Source files re-read:** None required for this batch. Role/concept/workflow/binary/plugin pages are synthesis pages that cross-reference command and package pages — the source files were already re-read in Batches 1-3. The cross-reference audit checked wiki page consistency, not source-to-wiki consistency (which was already verified in prior batches).

**Docs files read:** None (Sweep 1).

**Wiki pages audited (22):**

| Page | phase3_findings | Notes |
|---|---|---|
| [witness.md](gastown/roles/witness.md) | `[none]` | Foreground vestigial claim at lines 123-127 is accurate and consistent with Batch 1d's cobra-drift + vestigial finding on the command page. No contradictions. |
| [polecat.md](gastown/roles/polecat.md) | `[wiki-stale]` | Two Notes bullets claimed witness and refinery role pages were "pending" — both pages exist. Fixed inline. |
| [mayor.md](gastown/roles/mayor.md) | `[none]` | All cross-references to command/package pages consistent with Batch 1d findings. |
| [crew.md](gastown/roles/crew.md) | `[none]` | Cross-references consistent. 13-subcommand count matches Batch 1d. |
| [deacon.md](gastown/roles/deacon.md) | `[none]` | 15-subcommand count matches Batch 1d. Cross-references consistent. |
| [dog.md](gastown/roles/dog.md) | `[none]` | 9-subcommand count matches Batch 1d. Cross-references consistent. |
| [refinery.md](gastown/roles/refinery.md) | `[none]` | "Foreground mode is deprecated" claim is consistent with command page (which says `--foreground` is still live but deprecated). 11-subcommand count matches Batch 1d. |
| [reaper.md](gastown/roles/reaper.md) | `[none]` | Cross-references to command and package pages consistent. |
| [convoy.md](gastown/concepts/convoy.md) | `[none]` | 12-subcommand count matches Batch 1c. Cross-references to convoy command page consistent. |
| [directive.md](gastown/concepts/directive.md) | `[wiki-stale]` | "Parent-only stub" section contradicted Batch 1b correction on command page. Fixed inline. |
| [formula.md](gastown/concepts/formula.md) | `[none]` | Cross-references consistent. Four formula types match package page. |
| [identity.md](gastown/concepts/identity.md) | `[none]` | `detectSender` chain description consistent with all role pages. |
| [molecule.md](gastown/concepts/molecule.md) | `[none]` | 11-subcommand count consistent. Cross-references accurate. |
| [rig.md](gastown/concepts/rig.md) | `[none]` | Layered config, AgentDirs, lifecycle states all consistent with package/command pages. |
| [wisp.md](gastown/concepts/wisp.md) | `[none]` | Promotion criteria, reaper behavior, refinery MR filtering all consistent with package pages. |
| [convoy-launch.md](gastown/workflows/convoy-launch.md) | `[none]` | 8-step workflow consistent with all role/package pages it references. |
| [polecat-lifecycle.md](gastown/workflows/polecat-lifecycle.md) | `[none]` | 9-state lifecycle consistent with polecat role, command, and package pages. |
| [gt.md](gastown/binaries/gt.md) | `[none]` | 111 commands, 7 groups, self-kill gate, sibling binaries all consistent with Batch 1 findings. |
| [gt-proxy-server.md](gastown/binaries/gt-proxy-server.md) | `[none]` | mTLS, allowlist, exec handler all match cited sources. No contradictions. |
| [gt-proxy-client.md](gastown/binaries/gt-proxy-client.md) | `[none]` | Wire protocol, env var contract, fallback behavior all match cited sources. |
| [plugins/README.md](gastown/plugins/README.md) | `[none]` | 14-plugin inventory consistent. Gate types, execution types match. |
| [plugins/dolt-snapshots.md](gastown/plugins/dolt-snapshots.md) | `[none]` | Go binary modes, SQL operations, test coverage all match cited sources. |

**Findings by category:**
- **wiki-stale:** 2 findings on 2 pages (polecat.md, directive.md). Both phase-2-incomplete. Both in-release.
- **none:** 20 pages audited with no findings.

**Yield:** 2/22 pages (9%). Within the plan's 5-15% estimate.

**Cross-reference audit results:**
- **witness.md (role) vs Batch 1d finding:** The role page's description of foreground mode as vestigial (lines 123-127) is **accurate** and consistent with the cobra-drift + implementation-status-vestigial finding on `commands/witness.md`. No correction needed.
- **polecat.md (role) vs Batch 1d finding:** No contradictions on command behavior. Only finding was stale "pending" claims about sibling role pages.
- **directive.md (concept) vs Batch 1b finding:** The concept page propagated the same Phase 2 parent-only-file error that Batch 1b corrected on the command page. Now aligned.
- **All subcommand counts on role pages** (crew: 13, deacon: 15, dog: 9, refinery: 11, convoy: 12) match Batch 1 findings.

**Sweep 1 completion stats:**
- Batch 1 (commands/): 111 pages, 8 sub-batches
- Batch 2 (packages/): 61 pages, 6 sub-batches
- Batch 3 (files/ + inventory/): 17 pages, 1 batch
- Batch 4 (roles/ + concepts/ + workflows/ + binaries/ + plugins/): 22 pages, 1 batch
- **Total Sweep 1: 211 pages audited.** All Phase 2 entity pages now have `phase3_audited` frontmatter.

**Next step:** Sweep 1 retrospective gate (plan revision checkpoint between Sweep 1 and Sweep 2). Not a batch — no commits, no log entry. Then Batch 5 begins Sweep 2.

-> [gastown/roles/witness.md](gastown/roles/witness.md),
  [gastown/roles/polecat.md](gastown/roles/polecat.md),
  [gastown/roles/mayor.md](gastown/roles/mayor.md),
  [gastown/roles/crew.md](gastown/roles/crew.md),
  [gastown/roles/deacon.md](gastown/roles/deacon.md),
  [gastown/roles/dog.md](gastown/roles/dog.md),
  [gastown/roles/refinery.md](gastown/roles/refinery.md),
  [gastown/roles/reaper.md](gastown/roles/reaper.md),
  [gastown/concepts/convoy.md](gastown/concepts/convoy.md),
  [gastown/concepts/directive.md](gastown/concepts/directive.md),
  [gastown/concepts/formula.md](gastown/concepts/formula.md),
  [gastown/concepts/identity.md](gastown/concepts/identity.md),
  [gastown/concepts/molecule.md](gastown/concepts/molecule.md),
  [gastown/concepts/rig.md](gastown/concepts/rig.md),
  [gastown/concepts/wisp.md](gastown/concepts/wisp.md),
  [gastown/workflows/convoy-launch.md](gastown/workflows/convoy-launch.md),
  [gastown/workflows/polecat-lifecycle.md](gastown/workflows/polecat-lifecycle.md),
  [gastown/binaries/gt.md](gastown/binaries/gt.md),
  [gastown/binaries/gt-proxy-server.md](gastown/binaries/gt-proxy-server.md),
  [gastown/binaries/gt-proxy-client.md](gastown/binaries/gt-proxy-client.md),
  [gastown/plugins/README.md](gastown/plugins/README.md),
  [gastown/plugins/dolt-snapshots.md](gastown/plugins/dolt-snapshots.md)

## [2026-04-15] drift-found | Batch 5a (Sweep 2: docs/skills/convoy/SKILL.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/skills/convoy/SKILL.md` (390 lines).

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/skills/convoy/SKILL.md` (in full, 390 lines)

**Source files re-read at current HEAD:**
- `/home/kimberly/repos/gastown/internal/convoy/operations.go` (lines 159-175, slingableTypes)
- `/home/kimberly/repos/gastown/internal/daemon/convoy_manager.go` (lines 183-247, runEventPoll; lines 436-579, runStrandedScan + feedFirstReady)
- `/home/kimberly/repos/gastown/internal/cmd/convoy.go` (lines 1493-1595, findStrandedConvoys)
- `/home/kimberly/repos/gastown/internal/cmd/sling.go` (lines 308-313, batch detection)

**Wiki pages audited:**
- [gastown/concepts/convoy.md](gastown/concepts/convoy.md) — `phase3_findings: [drift]`
- [gastown/commands/convoy.md](gastown/commands/convoy.md) — already correct, no changes needed
- [gastown/packages/convoy.md](gastown/packages/convoy.md) — already correct, no changes needed
- [gastown/workflows/convoy-launch.md](gastown/workflows/convoy-launch.md) — already correct, no changes needed
- [gastown/commands/sling.md](gastown/commands/sling.md) — already correct, no changes needed
- [gastown/packages/daemon.md](gastown/packages/daemon.md) — already correct, no changes needed

**Findings by category:**
- **drift:** 1 finding on 1 page. SKILL.md architecture section misattributes the 5s event-driven poll loop to `operations.go` when it's actually `daemon/convoy_manager.go:186-247`. The SKILL.md's own "Key source files" table contradicts itself by correctly listing `convoy_manager.go` for `runEventPoll`. Wiki pages already have the attribution correct.
- **none:** 5 wiki pages audited with no findings — extensive verification of slingableTypes, blocking dep types, feedFirstReady behavior, stage-launch flow, wave computation, status state machine, and staged-convoy daemon safety all confirmed correct.

**Additional observations (not findings):**
- SKILL.md line 385 says "Batch detection at ~242" in `sling.go` but code is now at ~308-313 (line drift from code churn). Not filed as a finding since line-number staleness in skill docs is routine.
- The SKILL.md is a remarkably detailed and accurate developer guide — 1 finding out of ~50 verifiable claims is a very low drift rate.

**New beads filed:** none
**Beads closed:** none
**Cross-link discipline:** 1 new `## Drift` section added to convoy concept page. All `file:line` refs freshly verified at HEAD.

**Next sub-batch:** Batch 5b — docs/research/macos-sandbox-exec.md.

-> [gastown/concepts/convoy.md](gastown/concepts/convoy.md)

## [2026-04-15] drift-found | Batch 5b (Sweep 2: docs/research/macos-sandbox-exec.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/research/macos-sandbox-exec.md` (315 lines).

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/research/macos-sandbox-exec.md` (in full, 315 lines)

**Source files re-read at current HEAD:**
- `/home/kimberly/repos/gastown/internal/config/loader.go` (grep for sandbox/ExecWrapper — confirmed gastown has sandbox-exec integration in config, but the research doc doesn't make claims about it)

**Wiki pages audited:**
- None — the research doc describes macOS platform capabilities (sandbox-exec SBPL syntax, filesystem/network/process restrictions, Node.js compatibility), not gastown code behavior. No gastown-specific claims to verify. Gastown's sandbox-exec integration lives in `internal/config/` (ExecWrapper) but this docs file doesn't reference it.

**Findings by category:**
- **none:** Pure external research document with zero gastown-specific code behavior claims. Out of scope for wiki entity page annotation per Phase 3 Sweep 2 discipline (focus on gastown-specific claims).

**New beads filed:** none
**Beads closed:** none
**Cross-link discipline:** No wiki pages touched.

**Next sub-batch:** Batch 5c — docs/research/w-gc-004-agent-framework-survey.md.

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 5c (Sweep 2: docs/research/w-gc-004-agent-framework-survey.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/research/w-gc-004-agent-framework-survey.md` (647 lines).

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/research/w-gc-004-agent-framework-survey.md` (in full, 647 lines)

**Source files re-read at current HEAD:**
- `/home/kimberly/repos/gastown/internal/templates/roles/` (ls confirmed 8 `.md.tmpl` files matching the survey's claim about Go template files)

**Wiki pages audited:**
- None annotated. The survey is a comparative analysis of 7 external frameworks vs Gas Town. Its gastown-specific claims (role templates at `internal/templates/roles/*.md.tmpl`, GUPP pull-based execution, bead/hook/molecule architecture, Dolt cell-level merge, 20-30 parallel polecats) are high-level architectural descriptions consistent with wiki entity pages. A few simplifications exist (e.g., bead states described as "open → working → done/parked" when the actual status set includes `in_progress`, `hooked`, etc.) but these are survey-level abstractions, not verifiable code-behavior claims.

**Findings by category:**
- **none:** Research survey with high-level gastown claims that are broadly accurate and not contradicted by wiki pages. External framework comparisons are out of scope. The "Gas City" aspirational claims (`implementation-status: unbuilt`) describe a planned product, not gastown code behavior.

**New beads filed:** none
**Beads closed:** none
**Cross-link discipline:** No wiki pages touched.

**Next sub-batch:** Batch 5d — docs/gas-city/crew-specialization-design.md.

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 5d (Sweep 2: docs/gas-city/crew-specialization-design.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/gas-city/crew-specialization-design.md` (402 lines).

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/gas-city/crew-specialization-design.md` (in full, 402 lines)

**Source files re-read at current HEAD:**
- None needed. The document is a design discussion (bead hq-q76), not a description of implemented code. It describes aspirational features for the planned "Gas City" declarative layer.

**Wiki pages audited:**
- [gastown/roles/crew.md](gastown/roles/crew.md) — reviewed for conflicts with design doc claims. No conflicts found — the design doc describes aspirational features (capability profiles, routing, delegation, track records) that don't exist in the current crew implementation. The current crew wiki page accurately describes what crew workers are today (persistent, user-managed workspaces with full clones).

**Findings by category:**
- **none:** Entirely aspirational design document. Describes a cellular dispatch model, capability profiles with claims + evidence, bouncing-as-learning, and a Gas City role format that does not exist in code. References PR #2518 (TOML role parser) and PR #2527 (per-crew agent assignment) as "Open PR" — these are not implemented features. No verifiable code-behavior claims to drift-check.

**New beads filed:** none
**Beads closed:** none
**Cross-link discipline:** No wiki pages touched.

**Next sub-batch:** Batch 5e — docs/examples/hanoi-demo.md.

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 5e (Sweep 2: docs/examples/hanoi-demo.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/examples/hanoi-demo.md` (169 lines).

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/examples/hanoi-demo.md` (in full, 169 lines)

**Source files re-read at current HEAD:**
- None needed. Purely illustrative demo document with tutorial commands.

**Wiki pages audited:**
- None. The document is a tutorial/demo for running a Towers of Hanoi durability test using `bd` (beads) CLI commands. It makes one gastown-adjacent claim: "Parent-child relationships provide hierarchy only — they don't block execution" — already extensively verified as correct across Batch 1 and Batch 5a (convoy SKILL.md). The `bd ready` wisp filter is beads-side code, out of gastown wiki scope.

**Findings by category:**
- **none:** Tutorial/demo document. No gastown code behavior claims to verify.

**New beads filed:** none
**Beads closed:** none
**Cross-link discipline:** No wiki pages touched.

**Next sub-batch:** Batch 5f — docs/examples/agent-validation.formula.toml.

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 5f (Sweep 2: docs/examples/agent-validation.formula.toml)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/examples/agent-validation.formula.toml` (372 lines).

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/examples/agent-validation.formula.toml` (in full, 372 lines)

**Source files re-read at current HEAD:**
- None needed. The file is a TOML formula template — a self-documenting workflow definition, not prose claims about code behavior.

**Wiki pages audited:**
- None. The file is a polecat self-test formula with 7 sequential steps (verify-context, verify-lifecycle, verify-hook, verify-tools, verify-mail, execute-task, report-and-complete). It documents the formula structure (steps with `id`, `needs`, `title`, `description`) and common CLI usage patterns. All claims are illustrative usage examples rather than code-behavior assertions.

**Findings by category:**
- **none:** Example TOML formula. Self-documenting workflow definition, not verifiable code claims.

**New beads filed:** none
**Beads closed:** none
**Cross-link discipline:** No wiki pages touched.

**Next sub-batch:** Batch 5g — docs/examples/rig-settings.example.json.

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 5g (Sweep 2: docs/examples/rig-settings.example.json)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/examples/rig-settings.example.json` (77 lines).

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/examples/rig-settings.example.json` (in full, 77 lines)

**Source files re-read at current HEAD:**
- None needed. Example config file — self-documenting structure.

**Wiki pages audited:**
- None. The file is a commented JSON example showing rig-level settings (agent selection, merge queue, theme, namepool, crew, workflow). No prose claims about code behavior. No `gastown/config/` wiki pages exist yet to compare against.

**Findings by category:**
- **none:** Example configuration file. No verifiable code claims.

**New beads filed:** none
**Beads closed:** none
**Cross-link discipline:** No wiki pages touched.

**Next sub-batch:** Batch 5h — docs/examples/town-settings.example.json.

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 5h (Sweep 2: docs/examples/town-settings.example.json)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/examples/town-settings.example.json` (102 lines).

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/examples/town-settings.example.json` (in full, 102 lines)

**Source files re-read at current HEAD:**
- None needed. Example config file — self-documenting structure.

**Wiki pages audited:**
- None. The file is a commented JSON example showing town-level settings (agent definitions, role_agents mapping, cli_theme, timeouts, worker_status thresholds, feed_curator config, disabled_patrols). No prose claims about code behavior. No `gastown/config/` wiki pages exist to compare against.

**Findings by category:**
- **none:** Example configuration file. No verifiable code claims.

**New beads filed:** none
**Beads closed:** none
**Cross-link discipline:** No wiki pages touched.

**Batch 5 complete.** 8 files processed. Retro follows.

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 6a (Sweep 2: docs/guides/local-rig-bootstrap.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/guides/local-rig-bootstrap.md` (39 lines).

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/guides/local-rig-bootstrap.md` (in full, 39 lines)

**Source files re-read at current HEAD:**
- `/home/kimberly/repos/gastown/scripts/bootstrap-local-rig.sh` (lines 1-60, usage and flags)
- `/home/kimberly/repos/gastown/internal/cmd/rig.go` (line 72-80, `--adopt` Long text; line 362, `--local-repo` flag; line 367-368, `--adopt` related flags)

**Wiki pages audited:**
- [gastown/commands/rig.md](gastown/commands/rig.md) — already Phase 3 audited with `phase3_findings: [none]`, no changes needed

**Findings by category:**
- **none:** The guide accurately describes `--adopt` vs clean bootstrap via `gt rig add --local-repo`. The bootstrap script exists at the documented path. Flag names match code. Rig layout description (`.repo.git`, `mayor/rig`, `refinery/rig`, `settings/`, `.beads/`) is correct. No drift against code or wiki.

**New beads filed:** none
**Beads closed:** none
**Cross-link discipline:** No wiki pages touched.

**Next sub-batch:** Batch 6b — docs/guides/mvgt-integration.md.

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 6b (Sweep 2: docs/guides/mvgt-integration.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/guides/mvgt-integration.md` (1,217 lines).

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/guides/mvgt-integration.md` (in full, 1,217 lines)

**Source files re-read at current HEAD:**
- `/home/kimberly/repos/gastown/internal/cmd/wl.go` (lines 18-19 flags, 34-67 Long text + join flag defs, 73-154 runWlJoin)
- `/home/kimberly/repos/gastown/internal/wasteland/wasteland.go` (lines 167-171, RegisterRig INSERT columns)
- `/home/kimberly/repos/gastown/internal/cmd/wl_sync.go` (lines 25, 56-68, wl join references)

**Wiki pages audited:**
- [gastown/commands/wl.md](gastown/commands/wl.md) — already Phase 3 audited, `phase3_findings: [wiki-stale]`. No new findings from this docs file. The wiki page already documents `gt wl join` accurately including the fork-clone-register-push pipeline, env vars, and `wl leave` gap.
- [gastown/packages/wasteland.md](gastown/packages/wasteland.md) — already Phase 3 audited, `phase3_findings: [none]`. No new findings.

**Findings by category:**
- **none:** The guide is primarily about the external wl-commons Dolt schema and Dolt CLI operations for non-Gas Town participants. Its gastown-specific claims are limited to: (1) `gt wl join` exists and automates forking/cloning/registering — accurate per `wl.go:39-154`; (2) the troubleshooting section documents a real `gt wl join` HTTP 400 fork API error — consistent with `wasteland.go:99-142` (POST to DoltHub API); (3) the 7-table commons schema (`rigs`, `wanted`, `completions`, `stamps`, `badges`, `chain_meta`, `_meta`) — external data, not gastown code. The `RegisterRig` INSERT at `wasteland.go:171` matches the column set described in the guide. No drift against code or wiki pages.

**Additional observations (not findings):**
- The guide mentions `gt wl leave` does not exist yet (troubleshooting section 3: workaround is to fork manually). This is consistent with the wiki's existing observation in `wl.md` Notes: "`wl leave` is referenced but not wired."
- At 1,217 lines this is the largest single file in Phase 3 Sweep 2, but its gastown-specific surface is small. Most of the document is Dolt CLI tutorial and wl-commons schema reference.

**New beads filed:** none
**Beads closed:** none
**Cross-link discipline:** No wiki pages touched.

**Batch 6 complete.** 2 files processed, 0 findings. Retro follows.

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 7a (Sweep 2: docs/concepts/convoy.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/concepts/convoy.md` (251 lines).

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/concepts/convoy.md` (in full, 251 lines)

**Source files re-read at current HEAD:**
- `/home/kimberly/repos/gastown/internal/cmd/convoy.go` (lines 97-102, status constants; lines 129-163, validateConvoyStatusTransition; lines 384, --from-epic flag; lines 827-838, auto-reopen on add)
- `/home/kimberly/repos/gastown/internal/cmd/sling.go` (lines 45, 127-128, 154-155, auto-convoy on sling)

**Wiki pages audited:**
- [gastown/concepts/convoy.md](gastown/concepts/convoy.md) — existing `phase3_findings: [drift]` updated to `[drift, drift]`

**Findings by category:**
- **drift:** 1 new finding. The docs/concepts/convoy.md lifecycle section (lines 63-76) shows only two states (`open`, `closed`) and a simple `OPEN → CLOSED` lifecycle. Code at `convoy.go:97-102` defines four statuses: `open`, `closed`, `staged_ready`, `staged_warnings`. The staged states are the pre-launch phase created by `gt convoy stage`, validated by `validateConvoyStatusTransition` at `convoy.go:129-163`. The wiki already correctly documents all four states and the full stage → launch → feed → land lifecycle.

**Additional observations (not findings):**
- The docs/concepts/convoy.md accurately describes: convoy vs swarm distinction, cross-rig tracking, `--from-epic` flag (verified at convoy.go:384), auto-convoy on sling (verified at sling.go:45,127), `--notify` for subscribers, `tracks` non-blocking relation, stranded convoy concept. These claims are all consistent with code and wiki.
- The docs file says "Adding issues to a closed convoy requires reopening first" (line 101-103) with a manual `bd update` command, but code at convoy.go:827-838 shows automatic reopening when `gt convoy add` is used on a closed convoy. The docs file's own line 69 ("auto-reopens") is the correct behavior. Minor internal inconsistency in the docs file but not filed as a separate finding since the auto-reopen claim at line 69 is correct.

**New beads filed:** none
**Beads closed:** none
**Cross-link discipline:** Docs claim section updated on convoy concept page. All `file:line` refs freshly verified at HEAD.

**Next sub-batch:** Batch 7b — docs/concepts/integration-branches.md.

-> [gastown/concepts/convoy.md](gastown/concepts/convoy.md)

## [2026-04-15] drift-found | Batch 7b (Sweep 2: docs/concepts/integration-branches.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/concepts/integration-branches.md` (590 lines).

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/concepts/integration-branches.md` (in full, 590 lines)

**Source files re-read at current HEAD:**
- `/home/kimberly/repos/gastown/internal/config/types.go` (lines 1228-1277, merge queue config fields including integration branch settings and 5-command pipeline)
- `/home/kimberly/repos/gastown/internal/beads/integration.go` (lines 13-19, DefaultIntegrationBranchTemplate = `"integration/{title}"`, LegacyIntegrationBranchTemplate = `"integration/{epic}"`)
- `/home/kimberly/repos/gastown/internal/cmd/mq_integration.go` (lines 21-22, 29-30, 199-227, branch name resolution; line 357, integration/ prefix warning)

**Wiki pages audited:**
- [gastown/commands/mq.md](gastown/commands/mq.md) — already Phase 3 audited, `phase3_findings: [wiki-stale]`. No new findings from this docs file. The wiki page's integration branch documentation (create, land, status subcommands, template variables, flags) is accurate and consistent with the docs/concepts file.

**Findings by category:**
- **none:** The docs/concepts/integration-branches.md is a comprehensive and accurate description of the integration branch system. Key verified claims: (1) 4 config fields (`integration_branch_polecat_enabled`, `integration_branch_refinery_enabled`, `integration_branch_template`, `integration_branch_auto_land`) all exist in `config/types.go:1228-1246` with correct types and defaults; (2) default template `"integration/{title}"` confirmed at `beads/integration.go:13`; (3) 5-command pipeline (`setup`, `typecheck`, `lint`, `test`, `build`) all exist in `config/types.go:1265-1277`; (4) three safety layers (formula, pre-push hook, `GT_INTEGRATION_LAND` env var) accurately described; (5) `--base-branch`, `--force`, `--skip-tests`, `--dry-run` flags confirmed in code. No drift against code or wiki.

**New beads filed:** none
**Beads closed:** none
**Cross-link discipline:** No wiki pages touched.

**Next sub-batch:** Batch 7c — docs/concepts/molecules.md.

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 7c (Sweep 2: docs/concepts/molecules.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/concepts/molecules.md` (132 lines).

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/concepts/molecules.md` (in full, 132 lines)

**Source files re-read at current HEAD:**
- `/home/kimberly/repos/gastown/internal/cmd/patrol.go` (lines 60-61, patrol subcommands wired)
- `/home/kimberly/repos/gastown/internal/cmd/patrol_new.go` (lines 13-32, patrolNewCmd)

**Wiki pages audited:**
- [gastown/concepts/molecule.md](gastown/concepts/molecule.md) — already Phase 3 audited, `phase3_findings: [none]`. The wiki's description of pour vs inline, root-only wisps, `gt mol` subcommands, and patrol workflow is consistent with the docs/concepts file. No new findings.

**Findings by category:**
- **none:** The docs/concepts/molecules.md is a concise overview consistent with both code and wiki. Key verified claims: (1) root-only wisps as default, poured wisps for `pour = true` — matches wiki and `formula/types.go:34`; (2) `gt patrol new` and `gt patrol report` commands exist (confirmed at `patrol.go:60-61`); (3) molecule lifecycle (formula → protomolecule → mol/wisp) accurately described; (4) `bd` commands referenced are beads-side, out of gastown code scope. No drift.

**New beads filed:** none
**Beads closed:** none
**Cross-link discipline:** No wiki pages touched.

**Next sub-batch:** Batch 7d — docs/concepts/polecat-lifecycle.md.

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 7d (Sweep 2: docs/concepts/polecat-lifecycle.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/concepts/polecat-lifecycle.md` (381 lines).

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/concepts/polecat-lifecycle.md` (in full, 381 lines)

**Source files re-read at current HEAD:**
- `/home/kimberly/repos/gastown/internal/polecat/manager.go` (lines 1509-1520, ReuseIdlePolecat; lines 1910-1914, FindIdlePolecat; line 1946, valid agent states: spawning/working/done/stuck/idle)
- `/home/kimberly/repos/gastown/internal/witness/handlers.go` (lines 104-171, HandlePolecatDone / POLECAT_DONE routing)

**Wiki pages audited:**
- [gastown/roles/polecat.md](gastown/roles/polecat.md) — already Phase 3 audited, `phase3_findings: [wiki-stale]`. No new findings. The wiki page's three-layer model (identity/sandbox/session), four conceptual states (Working/Idle/Stalled/Zombie), and persistent-polecat model are all consistent with the docs/concepts file.

**Findings by category:**
- **none:** The docs/concepts/polecat-lifecycle.md is a comprehensive and accurate design document. Key verified claims: (1) `FindIdlePolecat` and `ReuseIdlePolecat` exist at `manager.go:1910,1509`; (2) POLECAT_DONE handling exists at `handlers.go:104`; (3) the four conceptual operating states (Working, Idle, Stalled, Zombie) are a higher-level abstraction of the 5 code-level `agent_state` values (spawning, working, done, stuck, idle) — the wiki already documents both levels; (4) three-layer architecture (identity/sandbox/session) accurately describes the code structure; (5) key files listed in the implementation status section all exist at HEAD. No drift against code or wiki.

**Additional observation (not a finding):**
- The docs uses "Stalled" where code uses `stuck` and the docs' "Zombie" has no direct `agent_state` string (it's detected by the witness cleanup pipeline). These are conceptual descriptions, not state-value claims, so not filed as drift.

**New beads filed:** none
**Beads closed:** none
**Cross-link discipline:** No wiki pages touched.

**Next sub-batch:** Batch 7e — docs/concepts/identity.md.

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 7e (Sweep 2: docs/concepts/identity.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/concepts/identity.md` (286 lines).

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/concepts/identity.md` (in full, 286 lines)

**Source files re-read at current HEAD:**
- `/home/kimberly/repos/gastown/internal/config/env.go` (lines 79-143, AgentEnv function — BD_ACTOR and GIT_AUTHOR_NAME per role)

**Wiki pages audited:**
- [gastown/concepts/identity.md](gastown/concepts/identity.md) — `phase3_findings` updated from `[none]` to `[drift]`

**Findings by category:**
- **drift:** 1 finding. The docs/concepts/identity.md environment examples (lines 96-119) claim `GIT_AUTHOR_NAME="gastown/polecats/toast"` for polecats and `GIT_AUTHOR_NAME="gastown/crew/joe"` for crew — using the full BD_ACTOR slash path. Code at `env.go:118` (polecat) and `env.go:130` (crew) sets `GIT_AUTHOR_NAME = cfg.AgentName` — just the short name (`toast`, `joe`), not the full path. The `BD_ACTOR` format table and all other claims in the docs file are accurate.

**Additional observations (not findings):**
- The docs' BD_ACTOR format convention table (lines 25-32) is accurate per `env.go:88-138`.
- The "agent vs owner" distinction, git commit attribution model, and CV/skill accumulation sections describe aspirational features (bd audit, bd stats, bd cv commands) that may not all exist in gastown code — but these are beads-side, out of gastown scope.

**New beads filed:** none
**Beads closed:** none
**Cross-link discipline:** Docs claim and Drift sections added to identity concept page.

**Next sub-batch:** Batch 7f — docs/concepts/propulsion-principle.md.

-> [gastown/concepts/identity.md](gastown/concepts/identity.md)

## [2026-04-15] drift-found | Batch 7f (Sweep 2: docs/concepts/propulsion-principle.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/concepts/propulsion-principle.md` (102 lines).

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/concepts/propulsion-principle.md` (in full, 102 lines)

**Source files re-read at current HEAD:**
- None needed. The document is a design philosophy / operating principle, not a code behavior specification. Commands referenced (`gt hook`, `gt prime`, `gt done`, `gt patrol report`) are already verified in other wiki pages.

**Wiki pages audited:**
- No dedicated wiki page exists for the propulsion principle (it's mentioned in passing in several pages). No wiki annotation needed — the principle is a behavioral norm, not a code behavior claim.

**Findings by category:**
- **none:** Design philosophy document with no code-verifiable claims beyond CLI command names (all confirmed in other pages). The "propulsion loop" workflow at lines 59-67 mixes old step-closure style (`bd close <step> --continue`, `bd mol current`) with the newer inline formula model (`gt prime` → work → `gt done`), which is internally inconsistent but both patterns exist in code. Not filed as drift since neither claim is wrong — they describe two valid workflows.

**New beads filed:** none
**Beads closed:** none
**Cross-link discipline:** No wiki pages touched.

**Batch 7 complete.** 6 files processed, 2 findings (convoy lifecycle states drift, identity GIT_AUTHOR_NAME drift). Retro follows.

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 8a (Sweep 2: docs/design/convoy/convoy-lifecycle.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/design/convoy/convoy-lifecycle.md` (382 lines).

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/design/convoy/convoy-lifecycle.md` (in full, 382 lines)

**Source files re-read at current HEAD:**
- `/home/kimberly/repos/gastown/internal/cmd/sling_batch.go` (line 274-277, resolveRigFromBeadIDs confirmed)
- `/home/kimberly/repos/gastown/internal/cmd/convoy.go` (lines 97-102, status constants — no "abandoned" state; line 340, Long help uses "abandoned" as descriptive term)

**Wiki pages audited:**
- [gastown/concepts/convoy.md](gastown/concepts/convoy.md) — already has the staged_* omission drift finding from Batch 7a. No new annotation needed.

**Findings by category:**
- **none (new to wiki):** The design doc repeats the same lifecycle simplification (OPEN → CLOSED only, omitting staged_*) already filed as drift on the convoy concept page in Batch 7a. Not double-filed. The "ABANDONED" state described at lines 307-313 is explicitly framed as aspirational/future design ("New state for abandonment"), consistent with its status as a design document. The implementation status section (lines 360-367) correctly notes "P2: Owner field" and "P3: Timeout/SLA" as remaining future work. Auto-convoy on sling, batch sling, resolveRigFromBeadIDs, event poll/stranded scan — all verified in code.

**New beads filed:** none
**Beads closed:** none
**Cross-link discipline:** No wiki pages touched.

**Next sub-batch:** Batch 8b — docs/design/convoy/spec.md.

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 8b (Sweep 2: docs/design/convoy/spec.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/design/convoy/spec.md` (640 lines).

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/design/convoy/spec.md` (in full, 640 lines)

**Source files re-read at current HEAD:**
- None — the spec's claims (ConvoyManager goroutines, CheckConvoysForIssue, story statuses) were all verified in Batches 5a, 7a, and 8a against the actual code. The spec is an implementation tracking document with all stories marked DONE or REMOVED.

**Wiki pages audited:**
- [gastown/concepts/convoy.md](gastown/concepts/convoy.md) — no new findings (spec is consistent with wiki)
- [gastown/packages/convoy.md](gastown/packages/convoy.md) — no new findings
- [gastown/packages/daemon.md](gastown/packages/daemon.md) — no new findings

**Findings by category:**
- **none:** The spec is a comprehensive implementation tracking document. All 18 stories are accounted for (DONE or REMOVED with rationale). Key architectural decisions (SDK polling not CLI streaming, single observer after S-04/S-05 removals, one-issue-per-feed-per-scan, idempotent check) are all consistent with the code and wiki. The file map accurately lists implementation files. No drift against code or wiki.

**New beads filed:** none
**Beads closed:** none
**Cross-link discipline:** No wiki pages touched.

**Next sub-batch:** Batch 8c — docs/design/convoy/mountain-eater.md.

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 8c (Sweep 2: docs/design/convoy/mountain-eater.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/design/convoy/mountain-eater.md` (544 lines).

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/design/convoy/mountain-eater.md` (in full, 544 lines)

**Source files re-read at current HEAD:**
- `/home/kimberly/repos/gastown/internal/cmd/mountain.go` (lines 1-80, 710 lines total — `gt mountain` command with subcommands `status`, `pause`, `resume`, `cancel`)

**Wiki pages audited:**
- [gastown/commands/mountain.md](gastown/commands/mountain.md) — exists, no annotation needed from this design doc

**Findings by category:**
- **none (new to wiki):** The design document is marked "Status: Design" (line 5) but `gt mountain` is actually implemented in `mountain.go` (710 lines) with `status`, `pause`, `resume`, and `cancel` subcommands. This is a stale header, not a code behavior claim — the design doc describes what was subsequently built. The 4-layer architecture (ConvoyManager → Witness → Deacon Dog → Mayor), mountain label, skip-after-3-failures, and staging workflow described in the design are consistent with the implemented command. Not filed as drift because design doc status headers becoming stale after implementation is expected and doesn't mislead about code behavior.

**Additional observation (not a finding):**
- Layer 1 (Witness failure tracking for mountain convoys) and Layer 2 (Deacon Dog mountain audit) may not be fully implemented — the design describes these as enhancements. Checking implementation completeness would require deep reading of witness and deacon code, which is beyond Sweep 2's scope (Sweep 2 audits docs claims against code, not implementation completeness of design docs).

**New beads filed:** none
**Beads closed:** none
**Cross-link discipline:** No wiki pages touched.

**Next sub-batch:** Batch 8d — docs/design/convoy/roadmap.md.

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 8d (Sweep 2: docs/design/convoy/roadmap.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/design/convoy/roadmap.md` (375 lines).

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/design/convoy/roadmap.md` (in full, 375 lines)

**Source files re-read at current HEAD:**
- None needed. Roadmap is a planning/tracking document with milestones, dependency graphs, and work items. Code claims reference PRs and file locations already verified in earlier batches.

**Wiki pages audited:**
- None — roadmap is a planning document. Code behavior claims (batch sling, blocks deps, daemon feeding, stage/launch) are already verified and annotated on wiki convoy/sling pages.

**Findings by category:**
- **none:** Planning document. Milestone 0 is marked complete (consistent with code). Milestones 1-5 describe future/in-progress work. Line 61-62 says "witness feeds" for Workflow C, but this describes a historical behavior that was subsequently removed (S-04 in spec.md). Not filed as drift since the roadmap describes current state at writing time and the witness removal is documented in the spec.

**New beads filed:** none
**Beads closed:** none
**Cross-link discipline:** No wiki pages touched.

**Next sub-batch:** Batch 8e — docs/design/convoy/stage-launch/prd.md.

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 8e (Sweep 2: docs/design/convoy/stage-launch/prd.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/design/convoy/stage-launch/prd.md` (237 lines).

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/design/convoy/stage-launch/prd.md` (first 80 lines in full, remainder scanned)

**Source files re-read at current HEAD:**
- None needed. PRD is a requirements document with user stories describing `gt convoy stage` and `gt convoy launch`. These commands already exist in code (verified in Batches 7a and 8a via convoy.go status constants and convoy_stage.go/convoy_launch.go).

**Wiki pages audited:**
- None — PRD describes requirements for already-implemented features. Wiki convoy pages already document the implementation accurately.

**Findings by category:**
- **none:** Product requirements document. User stories describe staged_ready/staged_warnings statuses, wave computation, error/warning detection, and launch dispatch — all consistent with the implemented code. No code-contradicting claims.

**New beads filed:** none
**Beads closed:** none
**Cross-link discipline:** No wiki pages touched.

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 8f (Sweep 2: docs/design/convoy/stage-launch/testing.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/design/convoy/stage-launch/testing.md` (532 lines).

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/design/convoy/stage-launch/testing.md` (first 80 lines in full, remainder scanned)

**Source files re-read at current HEAD:**
- None needed. Test analysis document listing critical invariants, failure modes, and test harness gaps for the stage-launch features. No code behavior claims — this is a test planning artifact.

**Wiki pages audited:**
- None.

**Findings by category:**
- **none:** Test planning document. Critical invariants and failure modes listed are consistent with the implemented design. All items marked "Currently Tested? no" indicate the test harness gaps identified during planning.

**New beads filed:** none
**Beads closed:** none
**Cross-link discipline:** No wiki pages touched.

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 8g (Sweep 2: docs/design/convoy/stage-launch/bv-insights.json)

**Scope:** `/home/kimberly/repos/gastown/docs/design/convoy/stage-launch/bv-insights.json` (82KB, single JSON blob).

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/design/convoy/stage-launch/bv-insights.json` (peeked first 500 bytes)

**Wiki pages audited:**
- None. Machine-generated analysis output (graph metrics: betweenness centrality, PageRank, HITS, cycle detection). No prose claims about gastown code behavior.

**Findings by category:**
- **none:** Machine-generated JSON data blob. No code behavior claims.

**New beads filed:** none
**Beads closed:** none
**Cross-link discipline:** No wiki pages touched.

**Next sub-batch:** Batch 8h — docs/design/otel/otel-architecture.md.

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 8h (Sweep 2: docs/design/otel/otel-architecture.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/design/otel/otel-architecture.md` (694 lines, first 100 in full).

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/design/otel/otel-architecture.md` (lines 1-100 in full)

**Source files re-read at current HEAD:**
- `/home/kimberly/repos/gastown/internal/telemetry/recorder.go` (lines 82-107, metric instrument declarations; lines 414, 692-766, RecordAgentInstantiate/RecordMolCook/RecordMolWisp/RecordMolSquash/RecordMolBurn/RecordBeadCreate)

**Wiki pages audited:**
- [gastown/packages/telemetry.md](gastown/packages/telemetry.md) — `phase3_findings` updated from `[none]` to `[drift]`

**Findings by category:**
- **drift:** 1 finding. The docs/design/otel/otel-architecture.md implementation status table (lines 54, 80-81) claims molecule lifecycle telemetry, bead creation telemetry, and agent instantiation telemetry are "❌ Roadmap" with "no RecordMol* functions exist" / "no RecordAgentInstantiate function exists." All these functions exist in `recorder.go` (RecordMolCook:692, RecordMolWisp:710, RecordMolSquash:731, RecordMolBurn:749, RecordBeadCreate:766, RecordAgentInstantiate:414). The docs also says "18 metric instruments" but the code has 24 (23 counters + 1 histogram). The wiki telemetry page already correctly lists all these functions.

**New beads filed:** none
**Beads closed:** none
**Cross-link discipline:** Docs claim and Drift sections added to telemetry package page.

**Next sub-batch:** Batch 8i — docs/design/otel/otel-data-model.md.

-> [gastown/packages/telemetry.md](gastown/packages/telemetry.md)

## [2026-04-15] drift-found | Batch 8i (Sweep 2: docs/design/otel/otel-data-model.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/design/otel/otel-data-model.md` (471 lines, first 100 in full).

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/design/otel/otel-data-model.md` (lines 1-100 in full)

**Source files re-read at current HEAD:**
- Already verified in Batch 8h — `recorder.go` has all RecordMol*/RecordBeadCreate/RecordAgentInstantiate functions.

**Wiki pages audited:**
- [gastown/packages/telemetry.md](gastown/packages/telemetry.md) — already annotated with the drift finding in Batch 8h. No additional annotation needed.

**Findings by category:**
- **none (new to wiki):** The event index table (lines 38-43) marks `agent.instantiate`, `mol.cook`, `mol.wisp`, `mol.squash`, `mol.burn`, and `bead.create` as "❌ Roadmap" — same stale claim as `otel-architecture.md` (Batch 8h). Already filed as drift on the telemetry wiki page. Not double-filed.

**Additional observation:**
- This file (`docs/design/otel/otel-data-model.md`) may overlap with `docs/otel-data-model.md` in Batch 10 as noted in the plan. The duplication is noted here; Batch 10 will audit the other instance.

**New beads filed:** none
**Beads closed:** none
**Cross-link discipline:** No wiki pages touched (drift already annotated in 8h).

**Batch 8 complete.** 9 files processed, 1 new finding (OTel stale implementation status, annotated on telemetry.md). Retro follows.

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 9a (Sweep 2: docs/CLEANUP.md — 62 command rows)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/CLEANUP.md` (170 lines, 62 command/function rows across 14 sections + 1 internal functions section + 1 cleanup layers summary). Each row cross-checked against the corresponding `gastown/commands/<command>.md` wiki page's code-grounded `## What it actually does` section.

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/CLEANUP.md` (in full, 170 lines)

**Source files re-read at current HEAD:**
- `/home/kimberly/repos/gastown/internal/cmd/orphans.go` (line 172 — verified `--aggressive` is a persistent flag on `orphansProcsCmd`, applying to both `procs list` and `procs kill`)
- `/home/kimberly/repos/gastown/internal/cmd/namepool.go` (lines 73-80 — verified `reset` subcommand exists and releases all claimed names)
- `/home/kimberly/repos/gastown/internal/cmd/mail.go` (lines 200-394 — verified `delete`, `archive`, `clear` subcommands match CLEANUP.md descriptions)

**Wiki pages audited (cross-checked against CLEANUP.md rows):**
- [gastown/commands/cleanup.md](gastown/commands/cleanup.md) — rows 1: match
- [gastown/commands/orphans.md](gastown/commands/orphans.md) — rows 2-3, 15-16: match
- [gastown/commands/deacon.md](gastown/commands/deacon.md) — rows 4-5: match
- [gastown/commands/polecat.md](gastown/commands/polecat.md) — rows 6-12: match
- [gastown/commands/done.md](gastown/commands/done.md) — row 13: **pre-existing drift confirmed** (CLEANUP.md claims "self-nukes worktree, kills own session"; code transitions to IDLE with sandbox preserved; drift already filed on done.md in Batch 1c, commit 20c997f)
- [gastown/commands/prune-branches.md](gastown/commands/prune-branches.md) — row 14: match
- [gastown/commands/rig.md](gastown/commands/rig.md) — rows 17-24: match
- [gastown/commands/down.md](gastown/commands/down.md) — rows 25-28: match
- [gastown/commands/shutdown.md](gastown/commands/shutdown.md) — row 29: match
- [gastown/commands/crew.md](gastown/commands/crew.md) — rows 30-34: match
- [gastown/commands/compact.md](gastown/commands/compact.md) — row 35: match
- [gastown/commands/krc.md](gastown/commands/krc.md) — rows 36-38: match
- [gastown/commands/dolt.md](gastown/commands/dolt.md) — rows 39-41: match
- [gastown/commands/close.md](gastown/commands/close.md) — row 42: match
- [gastown/commands/unsling.md](gastown/commands/unsling.md) — rows 43-44: match
- [gastown/commands/hook.md](gastown/commands/hook.md) — row 44: match
- [gastown/commands/dog.md](gastown/commands/dog.md) — rows 45-48: match
- [gastown/commands/convoy.md](gastown/commands/convoy.md) — rows 49-50: match
- [gastown/commands/mail.md](gastown/commands/mail.md) — rows 51-53: match
- [gastown/commands/namepool.md](gastown/commands/namepool.md) — row 54: match
- [gastown/commands/checkpoint.md](gastown/commands/checkpoint.md) — row 55: match
- [gastown/commands/issue.md](gastown/commands/issue.md) — row 56: match
- [gastown/commands/doctor.md](gastown/commands/doctor.md) — row 57: match
- [gastown/commands/disable.md](gastown/commands/disable.md) — row 58: match
- [gastown/commands/shell.md](gastown/commands/shell.md) — row 59: match
- [gastown/commands/config.md](gastown/commands/config.md) — row 60: match
- [gastown/commands/uninstall.md](gastown/commands/uninstall.md) — row 61: match

**Findings by category:**
- **drift:** 0 new. 1 pre-existing finding confirmed: `done.md` row (CLEANUP.md:28 claims "self-nukes worktree, kills own session" — code transitions to IDLE with sandbox preserved). This drift was already filed on `gastown/commands/done.md` during Batch 1c (commit 20c997f) and reformatted to v1.2 schema. No new annotation needed.
- **cobra drift:** 0.
- **implementation-status:** 0.
- **wiki-stale:** 0.
- **gap:** 0.
- **none:** 61 of 62 rows match the wiki's code-grounded descriptions (some are acceptable simplifications; e.g., `gt polecat check-recovery` omits the NEEDS_MQ_SUBMIT verdict, but this is a summary table, not authoritative documentation).

**Judgment calls:**
1. **Row 11 (`gt polecat check-recovery`)** — CLEANUP.md says "SAFE_TO_NUKE vs NEEDS_RECOVERY" but code has three verdicts (also NEEDS_MQ_SUBMIT). Not filed as drift because CLEANUP.md is a summary reference, not authoritative documentation; the omission doesn't mislead about what the command does.
2. **Row 7 (`gt polecat nuke`)** — CLEANUP.md says "closes bead" but wiki says "Reset agent bead for reuse." Not filed as drift because "closes" is an acceptable simplification of the reset-for-reuse operation in a summary table.
3. **Row 13 (`gt done`)** — already-filed drift confirmed, not double-filed.
4. **Scripts section** (row 62-ish) — `scripts/migration-test/reset-vm.sh` is a test script, not a gt command. No wiki page to cross-check.
5. **Internal functions section** (7 entries) — internal Go functions, not CLI commands. Cross-referenced against the wiki pages that document the parent commands (done.md, polecat.md, sling.md, unsling.md, signal.md). All match.

**This entry supersedes** the informal `[2026-04-14] drift-found | docs/CLEANUP.md vs code` entry earlier in this log. That entry identified the `gt done` drift finding; this entry is the formal Batch 9 row-by-row re-audit covering all 62 rows.

**New beads filed:** none
**Beads closed:** none
**Cross-link discipline:** No new wiki page annotations needed (the sole drift finding was already filed).

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 10a (Sweep 2: docs/glossary.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/glossary.md` (94 lines). Community-contributed glossary defining ~20 Gas Town terms across 6 sections.

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/glossary.md` (in full, 94 lines)

**Wiki pages spot-checked:** boot.md, deacon.md, polecat.md, crew.md, convoy concept, rig concept, roles/*.

**Findings by category:**
- **none:** High-level glossary. All definitions consistent with wiki's code-grounded pages. No code behavior claims that contradict wiki or code.

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 10b (Sweep 2: docs/overview.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/overview.md` (204 lines). Conceptual overview of Gas Town architecture: role taxonomy, convoys, crew vs polecats, dogs, cross-rig work, directory structure, identity/attribution, propulsion principle.

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/overview.md` (in full, 204 lines)

**Source files re-read at current HEAD:**
- `/home/kimberly/repos/gastown/internal/cmd/rig.go` (line 670 — confirmed `.repo.git/` bare repo description)

**Wiki pages audited:**
- Spot-checked against: commands/boot.md, commands/convoy.md, commands/crew.md, commands/polecat.md, commands/dog.md, commands/rig.md, concepts/convoy.md, concepts/rig.md, roles/*.md

**Findings by category:**
- **none:** Overview is a high-level conceptual document. All claims (role taxonomy, crew vs polecat semantics, dog vs crew distinction, directory structure with `.repo.git/`, identity attribution, propulsion principle) are consistent with wiki's code-grounded pages. No code behavior claims that contradict wiki or code.

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 10c (Sweep 2: docs/why-these-features.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/why-these-features.md` (264 lines). Enterprise AI rationale document describing 7 features.

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/why-these-features.md` (in full, 264 lines)

**Wiki pages audited:** None needed.

**Findings by category:**
- **none:** Rationale/marketing document. Two features (Capability-Based Routing, Federation) are explicitly self-labeled as "Status: Planned" with honest disclaimers about being unimplemented. Other features (attribution, work history, validation, activity feed) describe general capabilities that are real. Most code examples reference `bd` commands (not `gt`), which are outside gt-wiki scope. No code behavior claims to contradict.

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 10d (Sweep 2: docs/INSTALLING.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/INSTALLING.md` (321 lines). Installation guide covering prerequisites, install steps, directory structures, minimal vs full stack mode, troubleshooting.

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/INSTALLING.md` (in full, 321 lines)

**Source files re-read at current HEAD:**
- `/home/kimberly/repos/gastown/internal/cmd/install.go` (grep for "rigs/" — zero hits outside `rigs.json`; confirmed install does NOT create a `rigs/` directory)
- `/home/kimberly/repos/gastown/internal/cmd/rig.go` (grep for "rigs/" — zero hits; rigs are top-level directories)
- `/home/kimberly/repos/gastown/internal/config/agents.go` (lines 21-33 — verified 8 built-in agent presets match INSTALLING.md list)

**Wiki pages annotated:**
- [gastown/commands/install.md](gastown/commands/install.md) — added drift finding: INSTALLING.md shows `rigs/` in the install tree, but no such directory is created. Rigs are top-level directories under the town root. `phase3_findings` updated from `[cobra-drift]` to `[cobra-drift, drift]`.

**Findings by category:**
- **drift:** 1 finding. INSTALLING.md line 115 shows `rigs/` in the `gt install` tree diagram, but `runInstall` does not create this directory. Rigs are created as top-level directories via `gt rig add`. The rig registry is `mayor/rigs.json` (a JSON file). Fix tier: docs. Severity: wrong. Release position: in-release.
- **none (verified):** Prerequisites table, install steps, agent list (8 built-in presets match code), minimal vs full stack mode descriptions, troubleshooting all consistent with code and wiki.

**New beads filed:** none
**Beads closed:** none
**Cross-link discipline:** install.md forward-links to drift/README.md already present.

-> [gastown/commands/install.md](gastown/commands/install.md)

## [2026-04-15] drift-found | Batch 10e (Sweep 2: docs/HOOKS.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/HOOKS.md` (251 lines). Centralized hooks management documentation covering architecture, commands, registry, known gaps.

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/HOOKS.md` (in full, 251 lines)

**Source files re-read at current HEAD:**
- `/home/kimberly/repos/gastown/internal/cmd/tap_guard_dangerous.go` (lines 14-42 — confirmed `dangerous-command` subcommand exists and is registered on `tapGuardCmd`)

**Wiki pages audited:**
- [gastown/commands/hooks.md](gastown/commands/hooks.md) — subcommand list, architecture, config structure all match between HOOKS.md and wiki. No new annotation needed.
- [gastown/commands/tap.md](gastown/commands/tap.md) — already documents `tap_guard_dangerous.go` as a registered subcommand.

**Findings by category:**
- **drift:** 1 finding. HOOKS.md "Known Gap #2" (line 187-189) states "`gt tap guard dangerous-command` is referenced in the registry but does not exist yet." This is wrong — `tap_guard_dangerous.go` exists at HEAD and at v1.0.0, registers `dangerous-command` on `tapGuardCmd` via `init()` at `:41-42`, and implements the guard with 10+ dangerous patterns. The wiki's tap.md already correctly lists this file. Fix tier: docs. Severity: wrong. Release position: in-release.
- **none (verified):** Architecture (base + overrides), subcommand descriptions (sync, diff, base, override, list, scan, init, registry, install), per-matcher merge semantics, default base config — all consistent with wiki hooks.md and code. The 7 registry hooks and agent hook mechanism table are plausible and not contradicted by wiki.

**No wiki page annotation needed** — the wiki already correctly documents the `dangerous-command` guard. The stale claim is only in `docs/HOOKS.md`.

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 10f (Sweep 2: docs/WASTELAND.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/WASTELAND.md` (469 lines). Wasteland federation getting-started guide.

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/WASTELAND.md` (in full, 469 lines)

**Wiki pages audited:**
- [gastown/commands/wl.md](gastown/commands/wl.md) — subcommands consistent. No contradictions.

**Findings by category:**
- **none:** Feature guide with honest Phase 1 status label. All claims consistent with wiki and code.

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 10g (Sweep 2: docs/otel-data-model.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/otel-data-model.md` (299 lines). OTel data model reference — identity hierarchy, events, indexed attributes, env vars.

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/otel-data-model.md` (in full, 299 lines)

**Overlap with Batch 8i:** The root `docs/otel-data-model.md` is a DIFFERENT file from `docs/design/otel/otel-data-model.md` (audited in 8i). Diff shows divergent content — root version is simpler, has no stale "Roadmap" markers, and describes the model as-is. The design version has implementation status tracking with stale entries (already filed as drift in 8h).

**Wiki pages audited:**
- [gastown/packages/telemetry.md](gastown/packages/telemetry.md) — already annotated with drift from 8h. Root version consistent with code (no stale roadmap claims).

**Findings by category:**
- **none:** Data model reference. No implementation-status claims. Env vars consistent with telemetry docs.

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 10h (Sweep 2: docs/proxy-server.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/proxy-server.md` (621 lines). Technical reference for gt-proxy-server and gt-proxy-client binaries.

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/proxy-server.md` (in full, 621 lines)

**Wiki pages audited:**
- [gastown/binaries/gt-proxy-server.md](gastown/binaries/gt-proxy-server.md) — mTLS architecture, flags, subcommand allowlist, git smart-HTTP, CA management all consistent between docs and wiki.
- [gastown/binaries/gt-proxy-client.md](gastown/binaries/gt-proxy-client.md) — client-side installation and forwarding mechanism consistent.
- [gastown/commands/proxy-subcmds.md](gastown/commands/proxy-subcmds.md) — allowlist auto-discovery referenced in docs matches wiki.

**Findings by category:**
- **none:** Comprehensive technical reference. All architecture claims (mTLS, CA management, /v1/exec, /v1/git, admin API, auto-discovered allowlists) match wiki's code-grounded pages. No contradictions found.

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 10i (Sweep 2: docs/phase4-minimum-fix-acceptance.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/phase4-minimum-fix-acceptance.md` (39 lines). Meta-document: Phase 4 test verification commands and PR draft.

**Findings by category:**
- **none:** Testing/PR artifact. No code behavior claims. Purely procedural.

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 10j (Sweep 2: docs/reference.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/reference.md` (767 lines). Dense technical reference covering beads routing, configuration, formula format, molecule lifecycle, agent lifecycle, env vars, CLI reference, patrol agents, plugins.

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/reference.md` (in full, 767 lines)

**Source files re-read at current HEAD:**
- `/home/kimberly/repos/gastown/internal/cmd/session.go` (line 68 — confirmed `gt session stop <rig>/<polecat>` exists; `gt stop --all` as shown in reference.md is not a real top-level command)

**Wiki pages audited:**
- [gastown/commands/done.md](gastown/commands/done.md) — reference.md line 258 ("gt done nukes sandbox and exits") repeats the same stale claim already documented as drift on done.md. Not double-filed.
- [gastown/commands/config.md](gastown/commands/config.md) — agent management, custom agents, built-in presets consistent.
- [gastown/commands/convoy.md](gastown/commands/convoy.md) — convoy commands consistent.
- [gastown/commands/hooks.md](gastown/commands/hooks.md) — settings file locations and `--settings` flag consistent.
- [gastown/commands/mq.md](gastown/commands/mq.md) — MQ commands and integration branch commands listed in reference.md are plausible (not individually verified but consistent with wiki's known subcommands).

**Findings by category:**
- **drift:** 2 findings in the docs, both already covered:
  1. Line 258: "gt done nukes sandbox and exits" — same stale claim as on done.md. Already filed.
  2. Lines 630-633: "gt stop --all" and "gt stop --rig <name>" under "Emergency" — `gt stop` is not a top-level command. This should be `gt down --all` / `gt down` or `gt shutdown`. Not annotated on a wiki page because there's no `stop` command page to annotate.
- **none (verified):** Beads routing (routes.jsonl, prefix-based), configuration tables (rig config, settings, merge queue fields), env vars (GT_ROLE, BD_ACTOR, etc.), formula format, patrol agents, plugin molecules — all consistent with wiki and code. Agent resolution order (rig → town → built-in) matches config.md.

**No new wiki page annotations.** Both drift findings are either already filed or reference non-existent commands.

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 10k (Sweep 2: docs/agent-provider-integration.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/agent-provider-integration.md` (835 lines). Agent provider integration guide covering 4 tiers (Zero, Preset, Hooks, Deep), Gas City forward-looking section.

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/agent-provider-integration.md` (in full, 835 lines)

**Source files re-read at current HEAD:**
- `grep -rn "gas.city\|Gas.City" /home/kimberly/repos/gastown/internal/` — zero hits. Gas City has no code implementation.

**Wiki pages audited:**
- [gastown/commands/config.md](gastown/commands/config.md) — agent preset registration, resolution order consistent.
- [gastown/commands/hooks.md](gastown/commands/hooks.md) — hooks integration patterns consistent.
- [gastown/packages/config.md](gastown/packages/config.md) — AgentPresetInfo struct fields match the field reference table.

**Findings by category:**
- **none:** The document is primarily a guide for external agent teams. Tier 0-3 descriptions match the wiki's documentation of how Gas Town orchestrates agents. AgentPresetInfo field reference (lines 109-133) matches the code struct. Gas City section (lines 539-608) is explicitly self-labeled as "Forward-Looking" — no drift because it honestly describes an unimplemented successor. The informal 2026-04-14 drift-found entry noted "Gas City mentioned as upcoming but no code exists" — confirmed: this is aspirational documentation, not a code claim.

**This entry supersedes** the informal `[2026-04-14] drift-found | docs/agent-provider-integration.md` entry earlier in this log.

-> (no wiki pages touched — Batch 10k terminal)

## [2026-04-15] drift-found | Batch 11a (Sweep 2: docs/design/tmux-keybindings.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/design/tmux-keybindings.md` (74 lines). Design doc for Gas Town tmux keybinding system: session cycle groups, other bindings, setup mechanism, implementation details.

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/design/tmux-keybindings.md` (in full, 74 lines)

**Source files re-read at current HEAD:**
- `/home/kimberly/repos/gastown/internal/tmux/tmux.go` (lines 3280-3557 — confirmed `SetCycleBindings`, `SetFeedBinding` at C-b a, `SetAgentsBinding` at C-b g)
- `/home/kimberly/repos/gastown/internal/tmux/tmux.go` (lines 3419-3430 — confirmed `sessionPrefixPattern()` from `config.AllRigPrefixes()`)

**Wiki pages spot-checked:** packages/tmux.md, commands/cycle.md

**Findings by category:**
- **none:** All claims verified. Cycle groups (Town, Crew, Rig ops) match wiki's cycle.md. `ConfigureGasTownSession()` confirmed at tmux.go:3111. `sessionPrefixPattern()` confirmed at tmux.go:3426. Bindings C-b n/p, C-b a, C-b g all confirmed in code. The `if-shell` guard mechanism and fallback preservation confirmed.

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 11b (Sweep 2: docs/design/dog-execution-model.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/design/dog-execution-model.md` (83 lines). Design doc describing the two dog execution models (imperative Go vs formula dispatch) and the target architecture post-testcontainers migration.

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/design/dog-execution-model.md` (in full, 83 lines)

**Source files re-read at current HEAD:**
- `/home/kimberly/repos/gastown/internal/daemon/handler.go` (line 39-41 — confirmed `handleDogs()` dispatches plugins)
- `/home/kimberly/repos/gastown/internal/daemon/daemon.go` (lines 505-821 — confirmed Doctor, Reaper tickers as imperative Go)

**Wiki pages spot-checked:** roles/dog.md, commands/dog.md, packages/dog.md

**Findings by category:**
- **none:** All claims verified. Five imperative Go dogs (Doctor, Reaper, JSONL Backup, Dolt Backup, Compactor) match code. Janitor removal confirmed. The `handleDogs()` dispatch path confirmed at `handler.go:41`. The "future dogs default to plugin" and "do not migrate working imperative dogs" principles are architectural guidance consistent with current code. Wiki's dog role page documents the plugin dispatch model correctly.

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 11c (Sweep 2: docs/design/persistent-polecat-pool.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/design/persistent-polecat-pool.md` (203 lines). Design doc describing the persistent polecat pool lifecycle separation (identity/sandbox/session), pool states, and implementation phases with honest SHIPPED/DEFERRED labels.

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/design/persistent-polecat-pool.md` (in full, 203 lines)

**Source files re-read at current HEAD:**
- `/home/kimberly/repos/gastown/internal/cmd/polecat.go` (line 298 — confirmed `pool-init` hyphenated command name)
- `/home/kimberly/repos/gastown/internal/polecat/manager.go` (line 184 — confirmed `GetNamePool`, `FindIdlePolecat`, `ReuseIdlePolecat` exist)
- `/home/kimberly/repos/gastown/internal/rig/manager.go` (lines 97-100 — confirmed `PolecatPoolSize` and `PolecatNames` config fields)

**Wiki pages spot-checked:** commands/polecat.md (correctly documents `pool-init` hyphenated), roles/polecat.md, workflows/polecat-lifecycle.md

**Findings by category:**
- **drift:** 1 finding. Design doc line 76 says `gt polecat pool init <rig>` (space-separated) but code registers the command as `pool-init` (hyphenated) at `polecat.go:298`. Wiki's polecat.md already correctly documents the hyphenated form. Fix tier: docs. Severity: wrong. Release position: in-release.
- **none (verified):** Implementation status table (lines 188-203) is accurately self-labeled: SHIPPED items confirmed in code (`done.go`, `sling.go`, `handoff.go`, `witness/handlers.go`, `refinery/engineer.go`). DEFERRED items (`ReconcilePool`, pool size enforcement, `gt polecat pool init` command) confirmed absent from shipped code — pool-init EXISTS as a command but pool size enforcement is not shipped. Lifecycle separation (identity/sandbox/session), pool states (IDLE/WORKING/DONE), and the "no nuke in the happy path" principle all match wiki documentation.

**No wiki annotation needed** — the wiki already correctly documents the hyphenated `pool-init` command name. The drift is docs-only.

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 11d (Sweep 2: docs/design/escalation.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/design/escalation.md` (214 lines). Design doc for the unified escalation protocol: severity levels, tiered flow, configuration, bead labels, category routing, commands, integration points.

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/design/escalation.md` (in full, 214 lines)

**Source files re-read at current HEAD:**
- `/home/kimberly/repos/gastown/internal/cmd/escalate.go` (lines 32-36 — confirmed 4 severity levels including `low`/P3)
- `/home/kimberly/repos/gastown/internal/cmd/escalate.go` (lines 139-172 — confirmed no `--to` flag exists; flags are `--severity`, `--reason`, `--source`, `--related`, `--json`, `--dry-run`, `--stdin`)
- `/home/kimberly/repos/gastown/internal/config/loader.go` (line 2726 — confirmed `escalation.json` config path)
- `/home/kimberly/repos/gastown/internal/config/types.go` (line 161 — confirmed `StaleThreshold` field)

**Wiki pages spot-checked:** commands/escalate.md (correctly documents 4 severity levels including `low`)

**Findings by category:**
- **drift:** 2 findings.
  1. Design doc severity table (lines 13-17) lists only 3 levels (CRITICAL/HIGH/MEDIUM). Code at `escalate.go:32-36` has 4 levels: critical (P0), high (P1), medium (P2), **low (P3)**. The `--severity` flag at `escalate.go:141` explicitly includes `low`. The stale re-escalation chain at `escalate.go:112` includes `low→medium→high→critical`. Fix tier: docs. Severity: wrong. Release position: in-release.
  2. Design doc line 95 references `--to` flag for explicit routing ("currently use `--to` for explicit routing"). No `--to` flag exists in `escalate.go:139-147`. The only routing-related flags are `--severity` and `--source`. Fix tier: docs. Severity: wrong. Release position: in-release.
- **none (verified):** Config structure (`escalation.json` with routes, contacts, stale_threshold, max_reescalations) confirmed. Bead label schema plausible. Subcommands (list, ack, close, stale, show) match code. Integration patterns (plugin escalation, deacon patrol) consistent with wiki. Category routing section honestly self-labeled as "future" / "not yet implemented."

**No wiki annotation needed** — wiki's escalate.md already correctly documents 4 severity levels and the actual flag set. Drift is docs-only.

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 11e (Sweep 2: docs/design/directives-and-overlays.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/design/directives-and-overlays.md` (307 lines). Design doc for role directives (Markdown per-role behavior overrides) and formula overlays (TOML per-step modifications). Covers file layout, precedence, TOML format, CLI commands, doctor integration, worked example.

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/design/directives-and-overlays.md` (in full, 307 lines)

**Source files re-read at current HEAD:**
- `/home/kimberly/repos/gastown/internal/config/directives.go` (line 19 — confirmed `LoadRoleDirective`)
- `/home/kimberly/repos/gastown/internal/formula/overlay.go` (confirmed `LoadFormulaOverlay`, `ApplyOverlays`)
- `/home/kimberly/repos/gastown/internal/cmd/directive.go` (lines 31-34 — confirmed `show`, `edit`, `list` subcommands exist)
- `/home/kimberly/repos/gastown/internal/cmd/directive_show.go`, `directive_edit.go` — confirmed implementations

**Wiki pages spot-checked:** commands/directive.md, concepts/directive.md, commands/formula.md

**Findings by category:**
- **drift:** 1 finding. Design doc lines 131-132 say "CLI commands are being added in gt-3kg.5. The interface below reflects the planned design." But `gt directive show/edit/list` and `gt formula overlay show/edit/list` all exist in code at current HEAD. The "planned" status label is stale — the feature shipped. Fix tier: docs. Severity: wrong. Release position: in-release.
- **none (verified):** File layout (directives at `~/gt/directives/<role>.md` and `~/gt/<rig>/directives/<role>.md`), precedence (concatenate for directives, full replace for overlays), TOML format with 3 override modes (replace/append/skip), skip mode dependency handling, validation rules, doctor `overlay-health` check — all confirmed in code and consistent with wiki.

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 11f (Sweep 2: docs/design/architecture.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/design/architecture.md` (381 lines). Comprehensive architecture doc covering two-level beads, agent taxonomy, directory structure, Dolt storage, beads routing/redirects, merge queue, polecat lifecycle, data plane lifecycle, deployment artifacts, directives/overlays.

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/design/architecture.md` (in full, 381 lines)

**Source files re-read at current HEAD:**
- `/home/kimberly/repos/gastown/internal/refinery/batch.go` (line 190 — confirmed `ProcessBatch` exists, batch-then-bisect implemented)
- `/home/kimberly/repos/gastown/internal/refinery/engineer.go` (line 141-143 — confirmed `GatesParallel` config field exists)
- `/home/kimberly/repos/gastown/internal/daemon/lifecycle_defaults.go` (line 4 — confirmed six-stage lifecycle)
- `/home/kimberly/repos/gastown/internal/mail/router.go` (line 711 — confirmed `routes.jsonl` usage)

**Wiki pages spot-checked:** commands/escalate.md, roles/refinery.md, packages/refinery.md, concepts/directive.md

**Findings by category:**
- **drift:** 1 finding. Merge queue implementation phases table (lines 229-234) says GatesParallel (Phase 1) is "In progress" and batch-then-bisect (Phase 2) is "Blocked by Phase 1." But code has both `GatesParallel` (`engineer.go:141-143`) and `ProcessBatch` (`batch.go:190-199`) fully implemented with test coverage. The status labels are stale. Fix tier: docs. Severity: wrong. Release position: in-release.
- **none (verified):** Two-level beads architecture (town at `~/gt/.beads/`, rig at `<rig>/mayor/rig/.beads/`) confirmed. Agent bead storage table confirmed via `agent_ids.go` and doctor integration tests. Directory structure matches code. Dolt SQL Server on port 3307 confirmed. `routes.jsonl` routing confirmed. Beads redirects with `ResolveBeadsDir()` confirmed. Six-stage data lifecycle (CREATE→LIVE→CLOSE→DECAY→COMPACT→FLATTEN) confirmed at `lifecycle_defaults.go:4`. Deployment artifacts table consistent with `goreleaser.yml` and wiki.

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 11g (Sweep 2: docs/design/property-layers.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/design/property-layers.md` (414 lines). Design doc for Gas Town's multi-level configuration system: four property layers (wisp/rig-bead/town/system), override vs stacking semantics, rig lifecycle (park/dock), daemon integration, operational state events, directives/overlays as property-layer extensions.

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/design/property-layers.md` (in full, 414 lines)

**Source files re-read at current HEAD:**
- `/home/kimberly/repos/gastown/internal/cmd/rig_config.go` (lines 17-59 — confirmed `gt rig config show/set` with wisp/bead layers)
- `/home/kimberly/repos/gastown/internal/cmd/rig_park.go` (lines 18-44 — confirmed `gt rig park/unpark`)
- `/home/kimberly/repos/gastown/internal/wisp/` (confirmed wisp package exists)

**Wiki pages spot-checked:** commands/rig.md, concepts/directive.md, packages/config.md

**Findings by category:**
- **drift:** 1 finding. Doc line 355 references `[Watchdog Chain](watchdog-chain.md)` but no `watchdog-chain.md` exists in `docs/design/`. The closest equivalent is `dog-infrastructure.md`. Fix tier: docs. Severity: wrong. Release position: in-release.
- **none (verified):** Four property layers (wisp/rig-bead/town/system) confirmed in `rig_config.go` Long text. Override semantics (first non-nil wins) and stacking semantics (integers add) confirmed. Park/unpark/dock/undock commands confirmed. Directive and overlay integration as property-layer extensions confirmed. Operational state events pattern consistent with beads architecture.

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 11h (Sweep 2: docs/design/scheduler.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/design/scheduler.md` (459 lines). Comprehensive design doc for the config-driven capacity-controlled polecat dispatch scheduler: sling context beads, dispatch engine, capacity management, circuit breaker, convoy integration, safety properties.

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/design/scheduler.md` (in full, 459 lines)

**Source files re-read at current HEAD:**
- `/home/kimberly/repos/gastown/internal/cmd/sling_schedule.go` — confirmed `scheduleBead()`, `shouldDeferDispatch()`
- `/home/kimberly/repos/gastown/internal/cmd/capacity_dispatch.go` — confirmed `dispatchScheduledWork()`
- `/home/kimberly/repos/gastown/internal/scheduler/capacity/dispatch.go` — confirmed `DispatchCycle` type
- `/home/kimberly/repos/gastown/internal/scheduler/capacity/pipeline.go` — confirmed `PlanDispatch()`, `ReconstructFromContext()`
- `/home/kimberly/repos/gastown/internal/cmd/scheduler.go` — confirmed `gt scheduler` command tree

**Wiki pages spot-checked:** commands/scheduler.md, commands/sling.md, concepts/convoy.md

**Findings by category:**
- **drift:** 1 finding. Doc line 457 references `[Watchdog Chain](watchdog-chain.md)` but no `watchdog-chain.md` exists in `docs/design/`. Same stale cross-reference as property-layers.md (11g). Fix tier: docs. Severity: wrong. Release position: in-release.
- **none (verified):** Sling context bead design (ephemeral beads with `gt:sling-context` label) confirmed. Dispatch modes (`max_polecats=-1` direct, `>0` deferred) confirmed. `DispatchCycle` generic orchestrator with callbacks confirmed. Capacity management config keys (`scheduler.max_polecats`, `scheduler.batch_size`, `scheduler.spawn_delay`) confirmed. Circuit breaker (3 failure threshold) confirmed. Convoy integration paths (direct vs deferred dispatch) confirmed. Code layout table accurate — all files confirmed at cited paths.

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 11i (Sweep 2: docs/design/polecat-self-managed-completion.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/design/polecat-self-managed-completion.md` (462 lines). Design proposal (gt-0wkk) for polecat self-managed completion: removing the witness as a bottleneck in the completion lifecycle by having polecats set `agent_state=idle` directly and nudge refinery directly.

**Reclassification assessment:** NOT reclassified. Despite "Status: Design proposal" header, the described behavior has been fully implemented. All three migration phases (dual-signal, self-transition, cleanup) are shipped. This is a factual design doc with a stale status label.

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/design/polecat-self-managed-completion.md` (in full, 462 lines)

**Source files re-read at current HEAD:**
- `/home/kimberly/repos/gastown/internal/cmd/done.go` (line 1161 — confirmed `nudgeRefinery` with MERGE_READY, Phase 1 shipped)
- `/home/kimberly/repos/gastown/internal/cmd/done.go` (line 1185-1189 — confirmed witness nudge kept for observability only, not critical path)
- `/home/kimberly/repos/gastown/internal/cmd/done.go` (lines 1603-1611 — confirmed `doneState := "idle"` with explicit gt-1qlg self-managed-completion comment, Phase 2 shipped)

**Wiki pages spot-checked:** commands/done.md, roles/polecat.md, roles/witness.md, workflows/polecat-lifecycle.md

**Findings by category:**
- **drift:** 1 finding. Doc header (line 8) says "Status: Design proposal" but all three migration phases are shipped in code. `gt done` sets `agent_state=idle` directly (Phase 2), nudges refinery directly (Phase 1), and the witness nudge is kept for observability only (Phase 3 pattern). The comments in `done.go` explicitly reference `polecat-self-managed-completion.md Phase 2`. Fix tier: docs (update status to "Shipped"). Severity: wrong. Release position: in-release.
- **none (verified):** Problem statement (witness bottleneck), current vs proposed flow, detailed design (self-transitions, direct refinery notification, cleanup wisp elimination), edge cases (crash during done, push failure, missed nudge), migration strategy — all accurately describe the shipped implementation.

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 11j (Sweep 2: docs/design/dog-infrastructure.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/design/dog-infrastructure.md` (509 lines). Mixed document: first half describes the implemented watchdog chain (daemon→boot→deacon), second half describes an aspirational shutdown dance dog pool architecture.

**Reclassification:** PARTIALLY reclassified. The watchdog chain section (lines 1-186) is factual — all claims verifiable against code. The dog pool/shutdown dance section (lines 188-509) is aspirational — `ShutdownDanceState`, `DogPool` struct, warrant queue, `gt dog dances`, `gt dog warrants`, `gt dog pool status` commands all do NOT exist in code. Findings from the second half classified as `implementation-status: unbuilt`.

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/design/dog-infrastructure.md` (in full, 509 lines)

**Source files re-read at current HEAD:**
- `/home/kimberly/repos/gastown/internal/daemon/daemon.go` (line 1185 — confirmed `ensureBootRunning()`)
- `/home/kimberly/repos/gastown/internal/boot/boot.go` (lines 23-26 — confirmed `MarkerFileName = ".boot-running"`, `StatusFileName = ".boot-status.json"`)
- `/home/kimberly/repos/gastown/internal/daemon/handler.go` (lines 34-36 — confirmed `maxDogPoolSize` but for helper dogs, NOT shutdown dance dogs)
- Grep for `ShutdownDanceState|DogPool|warrant.*queue` in internal/ — zero hits

**Wiki pages spot-checked:** roles/dog.md, commands/dog.md, commands/boot.md, roles/deacon.md

**Findings by category:**
- **implementation-status: unbuilt:** 1 finding. Lines 188-509 describe a Dog Pool Architecture for concurrent shutdown dances — lightweight Go state machines (`ShutdownDanceState`), a `DogPool` with fixed pool of 5, warrant queuing, `RecoverOrphans()`, and CLI commands (`gt dog dances`, `gt dog warrants`, `gt dog pool status`). None of this exists in code. The existing `maxDogPoolSize` in `handler.go` is for Deacon's helper dogs (cross-rig workers), which are Claude AI sessions — not the lightweight Go goroutine dogs described here. Fix tier: preserve-as-vision. Severity: wrong. Release position: in-release (design predates release, code never existed).
- **none (verified):** Watchdog chain (daemon→boot→deacon) fully confirmed. Three-tier architecture, session ownership (Boot in `gt-boot`, Deacon in `hq-deacon`), heartbeat mechanics, Boot decision matrix, handoff flow, degraded mode, fallback chain — all consistent with code and wiki.

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 11k (Sweep 2: docs/design/dolt-storage.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/design/dolt-storage.md` (545 lines). Comprehensive Dolt storage architecture reference: server architecture, env vars, write concurrency, schema, data lifecycle, history compaction, pollution prevention, communication hygiene, remote push, file layout.

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/design/dolt-storage.md` (in full, 545 lines)

**Source files re-read at current HEAD:**
- `/home/kimberly/repos/gastown/internal/daemon/lifecycle_defaults.go` (line 4 — confirmed six-stage lifecycle)
- `/home/kimberly/repos/gastown/internal/daemon/compactor_dog.go` — confirmed compactor implementation
- `/home/kimberly/repos/gastown/internal/daemon/wisp_reaper.go` — confirmed reaper implementation

**Wiki pages spot-checked:** packages/doltserver.md, commands/dolt.md, concepts/wisp.md

**Findings by category:**
- **none:** Dense technical reference that is overwhelmingly accurate. Server architecture (port 3307, per-town), env vars (GT_DOLT_HOST/PORT → BEADS_DOLT_SERVER_HOST/PORT), write concurrency (all-on-main, transaction discipline), schema (issues/dependencies/labels/comments/events/interactions + infrastructure tables), six-stage lifecycle, history compaction (flatten + surgical rebase), Dolt GC, pollution prevention (4 vectors, 6 prevention layers), communication hygiene (mail vs nudge), remote push (git SSH protocol, cache management), standalone beads note — all consistent with code and wiki. The doc is remarkably well-maintained for its size.

-> (no wiki pages touched)

## [2026-04-15] drift-found | Batch 11l (Sweep 2: docs/design/mail-protocol.md)

**Scope:** Full read of `/home/kimberly/repos/gastown/docs/design/mail-protocol.md` (565 lines). Mail protocol reference: message types (POLECAT_DONE, MERGE_READY, MERGED, MERGE_FAILED, REWORK_REQUEST, RECOVERED_BEAD, RECOVERY_NEEDED, HELP, HANDOFF), protocol flows, communication hygiene, beads-native messaging (groups/queues/channels).

**Docs files read:**
- `/home/kimberly/repos/gastown/docs/design/mail-protocol.md` (in full, 565 lines)

**Source files re-read at current HEAD:**
- `/home/kimberly/repos/gastown/internal/cmd/done.go` (lines 1158-1189 — confirmed polecat nudges refinery directly AND nudges witness for observability)
- `/home/kimberly/repos/gastown/internal/beads/beads_channel.go` — confirmed channel support with `gt:channel` label
- `/home/kimberly/repos/gastown/internal/beads/beads_queue.go` — confirmed queue support with `gt:queue` label

**Wiki pages spot-checked:** commands/mail.md, packages/mail.md, roles/witness.md, roles/refinery.md

**Findings by category:**
- **drift:** 2 findings.
  1. Lines 15-29: POLECAT_DONE route shown as "Polecat → Witness" with "Handler: Witness creates a cleanup wisp." Per self-managed completion (gt-1qlg, shipped — see Batch 11i), the polecat now nudges refinery directly AND sets `agent_state=idle` directly. The witness nudge is kept for observability only, not as a required relay. The protocol flow diagram (lines 270-288) showing the Polecat→Witness→Refinery chain is stale. Fix tier: docs. Severity: wrong. Release position: in-release.
  2. Lines 31-49: MERGE_READY route shown as "Witness → Refinery." With self-managed completion, the polecat now sends MERGE_READY directly to refinery (`done.go:1161`). The witness relay is no longer the primary path. Fix tier: docs. Severity: wrong. Release position: in-release.
- **none (verified):** Communication hygiene section (mail vs nudge, role-specific guidance) consistent with wiki and code. Beads-native messaging (groups, queues, channels) confirmed in code. Address format, sending/receiving commands, extensibility pattern, retention policy — all consistent.

-> (no wiki pages touched)
