---
title: gastown topic landing
type: note
status: stub
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/
tags: [landing, index]
---

# gastown

## Mission (current phase)

This topic **maps the gastown codebase in the wiki** so that every part
is represented at inventory level and every load-bearing part has a
full entity page that **accurately describes what the code actually
does**, read off the source and cited with `file:line` references. See
the plan at `.claude/plans/2026-04-11-gastown-map.md` for the design,
the 12-layer traversal, the load-bearing test, and the cross-linking
discipline.

**Deliverable this phase:** a wiki that answers "what does gastown's
`<component>` actually do?" with code-grounded pages. Nothing else.

**Explicitly out of scope this phase:**

- Drift analysis, gap analysis against upstream docs, audience
  classification, upstream documentation rewrites — all deferred to
  later phases.
- Reading `docs/` content beyond inventory rows — phase 3.
- Edits to anything outside `~/repos/wiki/`. `/home/kimberly/repos/gastown/`
  is read-only reference.

## Working model

- Read from `/home/kimberly/repos/gastown/` (source of truth, never
  modified).
- Produce wiki pages under this topic following the schema's entity
  template (frontmatter + "What it actually does" + `file:line`
  source refs + "Notes / open questions").
- Link every entity page to every adjacent entity per the cross-linking
  discipline in the plan — forward links, back-references, alias
  coverage, `grep`-verified before commit. This is what makes the wiki
  answer multi-hop questions later.
- Track follow-up investigation work in **beads** — `bd list -l gastown`
  for the live queue. Agent-task conventions in the "Task tracking"
  section of [../CLAUDE.md](../CLAUDE.md).

## Coordinates

- **Source under investigation:** `/home/kimberly/repos/gastown/`
- **Upstream:** https://github.com/gastownhall/gastown
- **Go module path declared in `go.mod`:** `github.com/steveyegge/gastown`
  (the `gastownhall` repo self-identifies under the `steveyegge`
  namespace, so `go install` must use the `steveyegge` path)
- **Separate runtime workspace:** `~/gt/` — the Docker-containerized
  deployment of `gt`. Not part of the wiki; independent from this
  mapping effort.

## Scope of the map

gastown is a multi-agent orchestration system for AI coding runtimes,
managed by the `gt` CLI. The map will cover, in order (see plan for
the 12 layers):

- Build & packaging (Makefile, Dockerfiles, flake.nix, .goreleaser.yml, go.mod)
- Binaries & entry points (`cmd/gt`, `cmd/gt-proxy-server`, `cmd/gt-proxy-client`)
- The command layer (all ~107 top-level commands in `internal/cmd/`)
- Platform services (`cli`, `config`, `session`, `workspace`, `style`, `ui`, `telemetry`, `version`, `util`)
- Data layer (`beads`, `doltserver`, `mail`, `nudge`, `events`, etc.)
- Agent runtime — packages + first-class domain entities: roles
  (mayor, polecat, crew, dog, deacon, refinery, witness, wisp, reaper),
  concepts (convoy, rig, molecule, formula, directive, identity,
  polecat-lifecycle), services, workflows
- Diagnostics & health (`doctor`, `health`, `keepalive`, `deps`)
- Long-running processes (`daemon`, `tmux`, `runtime`)
- Supporting libraries
- Plugins (`plugins/*`)
- Auxiliary trees (`scripts/`, `templates/`, `gt-model-eval/`, `npm-package/`, `.github/`, `.githooks/`, `.claude/`, `.opencode/`, `.runtime/`)
- Documentation tree (`docs/`) — inventory only this phase

## Sub-index

### Binaries

- [gt](binaries/gt.md) — main CLI; entry point, ldflags, startup sequence, command groups, exempt-command maps
- [gt-proxy-server](binaries/gt-proxy-server.md) — long-lived host-side mTLS proxy server for polecat containers
- [gt-proxy-client](binaries/gt-proxy-client.md) — container-side shim that forwards `gt`/`bd` calls to the proxy server (or execs the real binary)

### Commands

- [gt command tree](commands/README.md) — inventory of all 111 top-level `rootCmd.AddCommand` registrations + 495 total `cobra.Command` definitions
- **Diagnostics group** (22 commands, fully mapped) — see the `entity page` column of [commands/README.md](commands/README.md) for individual links: [activity](commands/activity.md), [audit](commands/audit.md), [checkpoint](commands/checkpoint.md), [costs](commands/costs.md), [dashboard](commands/dashboard.md), [doctor](commands/doctor.md), [feed](commands/feed.md), [heartbeat](commands/heartbeat.md), [info](commands/info.md), [log](commands/log.md), [metrics](commands/metrics.md), [patrol](commands/patrol.md), [prime](commands/prime.md), [repair](commands/repair.md), [seance](commands/seance.md), [stale](commands/stale.md), [status](commands/status.md), [thanks](commands/thanks.md), [upgrade](commands/upgrade.md), [version](commands/version.md), [vitals](commands/vitals.md), [whoami](commands/whoami.md)
- **Configuration group** (11 commands, fully mapped): [account](commands/account.md), [config](commands/config.md), [directive](commands/directive.md), [disable](commands/disable.md), [enable](commands/enable.md), [hooks](commands/hooks.md), [issue](commands/issue.md), [plugin](commands/plugin.md), [shell](commands/shell.md), [theme](commands/theme.md), [uninstall](commands/uninstall.md)
- **Work Management group** (26 commands, fully mapped): [assign](commands/assign.md), [bead](commands/bead.md), [cat](commands/cat.md), [changelog](commands/changelog.md), [cleanup](commands/cleanup.md), [close](commands/close.md), [compact](commands/compact.md), [convoy](commands/convoy.md), [done](commands/done.md), [formula](commands/formula.md), [handoff](commands/handoff.md), [hook](commands/hook.md), [molecule](commands/molecule.md), [mountain](commands/mountain.md), [mq](commands/mq.md), [orphans](commands/orphans.md), [prune-branches](commands/prune-branches.md), [ready](commands/ready.md), [release](commands/release.md), [resume](commands/resume.md), [scheduler](commands/scheduler.md), [sling](commands/sling.md), [synthesis](commands/synthesis.md), [trail](commands/trail.md), [unsling](commands/unsling.md), [wl](commands/wl.md)
- **Agent Management group** (12 commands, fully mapped): [agents](commands/agents.md), [boot](commands/boot.md), [callbacks](commands/callbacks.md), [deacon](commands/deacon.md), [dog](commands/dog.md), [mayor](commands/mayor.md), [polecat](commands/polecat.md), [refinery](commands/refinery.md), [role](commands/role.md), [session](commands/session.md), [signal](commands/signal.md), [witness](commands/witness.md)
- **Communication group** (7 commands, fully mapped): [broadcast](commands/broadcast.md), [dnd](commands/dnd.md), [escalate](commands/escalate.md), [mail](commands/mail.md), [notify](commands/notify.md), [nudge](commands/nudge.md), [peek](commands/peek.md)

### Files

- [Makefile](files/makefile.md) — canonical build recipe; produces `gt`, `gt-proxy-server`, `gt-proxy-client` with the `BuiltProperly` ldflag
- [Dockerfile](files/dockerfile.md) — primary container image for `gt`
- [Dockerfile.e2e](files/dockerfile-e2e.md) — end-to-end test container
- [docker-compose.yml](files/docker-compose.md) — Compose spec
- [docker-entrypoint.sh](files/docker-entrypoint.md) — container entrypoint shell script
- [flake.nix](files/flake-nix.md) — Nix flake (dev shell, packages)
- [.goreleaser.yml](files/goreleaser-yml.md) — GoReleaser release-pipeline build config
- [.golangci.yml](files/golangci-yml.md) — golangci-lint configuration
- [go.mod](files/go-mod.md) — Go module manifest

### Inventory

- [inventory/README.md](inventory/README.md) — neutral A-level enumeration of repo contents
  - [repo-root](inventory/repo-root.md) — top-level files and directories at `/home/kimberly/repos/gastown/`
  - [docs-tree](inventory/docs-tree.md) — every file under `docs/` with line counts
  - [go-packages](inventory/go-packages.md) — every Go package under `cmd/`, `internal/`, `plugins/` with file counts
