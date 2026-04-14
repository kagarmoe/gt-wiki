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
