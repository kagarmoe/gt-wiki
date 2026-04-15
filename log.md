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
