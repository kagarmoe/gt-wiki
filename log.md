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
