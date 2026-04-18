---
title: rig (concept)
type: concept
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/rig/types.go
  - /home/kimberly/repos/gastown/internal/rig/config.go
  - /home/kimberly/repos/gastown/internal/rig/manager.go
  - /home/kimberly/repos/gastown/internal/cmd/rig.go
tags: [concept, rig, workspace, container, layered-config, polecats, crew, refinery, witness, mayor]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
phase8_audited: 2026-04-17
phase8_findings: [cross-ref-only]
detail_depth: {synthesis: 2, relationships: 2}
---

# rig

A **rig** is Gas Town's workspace abstraction. It is a
directory on disk that contains one cloned source repository,
the per-project agents that watch it (a Witness, a Refinery, a
Mayor clone), its own pool of workers (polecats, crew), its own
rig-level beads database, its own settings, plugins, and patrol
molecules. A rig is the unit of "one project being worked on by
Gas Town." A town has many rigs; a rig has one repo.

Put differently: a rig is to a town what a project is to an
IDE workspace. The **town** is the installation
(`~/gt/` / `<townRoot>`) that holds all the rigs, plus the
town-level Mayor, Deacon, and town beads. The **rig** is one
project inside that installation.

## Why rigs exist

A plain git checkout isn't enough to run Gas Town workers
against a project. Running polecats concurrently requires
shared git state (so one polecat's branch is visible to the
Refinery without round-tripping the remote), a beads database
scoped to the project (so issues from one project don't mix
with another), per-project patrol state for the Witness, a
Mayor clone that can stay parked on the default branch without
conflicting with the Refinery, and a scaffold for per-project
plugins and formulas.

The rig abstraction packages all of that into a single
directory with a conventional layout. When the operator runs
`gt rig add <name> <git-url>`, gastown creates that layout
atomically:

- `<rigRoot>/config.json` — rig configuration
- `<rigRoot>/.repo.git/` — shared bare clone (source of truth
  for refinery + polecats)
- `<rigRoot>/refinery/rig/` — refinery worktree (default branch)
- `<rigRoot>/mayor/rig/` — mayor's *separate* regular clone
  (NOT a worktree, so it can park on main without colliding
  with refinery's own default-branch worktree)
- `<rigRoot>/witness/` — empty directory (witness has no clone)
- `<rigRoot>/polecats/` — polecat worktrees (scaffolded empty)
- `<rigRoot>/crew/` — crew worktrees (scaffolded empty + README)
- `<rigRoot>/.beads/` — rig-level beads database OR a redirect
  file pointing at `mayor/rig/.beads/` (when the source repo
  tracks its own `.beads/`)
- `<rigRoot>/plugins/` — rig-level plugins
- `<rigRoot>/settings/` — rig-level settings overrides
- `<rigRoot>/.gitignore` — defensive gitignore covering all
  GT runtime directories in case someone `git init`s the
  container by accident

The canonical declaration of the agent dirs is `rig.AgentDirs`
in `/home/kimberly/repos/gastown/internal/rig/types.go:48-54`:
`polecats`, `crew`, `refinery/rig`, `witness`, `mayor/rig`.

## Rig vs town: the load-bearing distinction

The **town** is the enclosing installation. `~/gt/` (or
whatever `$GT_TOWN_ROOT` points at) is a town. Inside the town:

- `~/gt/.beads/` — **town beads** (mail, coordination,
  town-level agents: Mayor, Deacon, role beads).
- `~/gt/mayor/` — Mayor's home. Contains `town.json`
  (the town marker), `rigs.json` (the registry of all rigs),
  and the Mayor's working state.
- `~/gt/deacon/` — Deacon's home + dogs.
- `~/gt/settings/config.json` — town-level settings.
- `~/gt/plugins/` — town-level plugins (shared across rigs).
- `~/gt/<rig1>/`, `~/gt/<rig2>/`, ... — each registered rig
  as a sibling directory inside the town.

**Town-level agents are Mayor and Deacon.** They are shared
across all rigs in the town and live in town beads.
**Rig-level agents are Witness and Refinery.** They are created
once per rig, scoped to that rig, and live in rig beads with
the rig's prefix
(`/home/kimberly/repos/gastown/internal/rig/manager.go:1125-1140`).
The Mayor has a per-rig clone at `<rig>/mayor/rig/`, but the
Mayor *identity* is town-level, not rig-level.

Mail routing, bead routing, and agent dispatch all depend on
this split. A message addressed to `mayor/` goes to the
town-level Mayor; a message addressed to `<rig>/witness` goes
to the rig-level Witness. See [identity](identity.md).

## Layered config

A rig's "effective config" is resolved through a four-layer
chain defined in
`/home/kimberly/repos/gastown/internal/rig/config.go:59-87`
(`Rig.GetConfigWithSource`). First non-nil value wins:

1. **`SourceWisp`** — local wisp layer
   (`.beads-wisp/config/<rig>.json`). Can also explicitly
   *block* a key, returning `SourceBlocked`.
2. **`SourceBead`** — labels on the rig identity bead. Walks
   the bead's labels looking for `<key>:<value>` format.
3. **`SourceTown`** — town defaults. Currently a placeholder
   (`config.go:76-78`: "For now, we skip directly to system
   defaults").
4. **`SourceSystem`** — compiled-in defaults in `SystemDefaults`
   (`config.go:33-42`).

Current system defaults include `status: "operational"`,
`auto_restart: true`, `max_polecats: 10`, `dnd: false`, and
`default_formula: "mol-polecat-work"`. The `gt rig config
show` CLI surfaces the effective value *and* its source, so an
operator can tell whether a setting is default, town-wide, or
rig-specific.

**Stacking keys** are a special case: `priority_adjustment` is
the only one listed (`config.go:46-48`). For stacking keys,
the layers **sum** instead of override — a bead-layer +1 plus a
wisp-layer +2 produces an effective +3. Override semantics
apply to all other keys.

The package-level realization is covered in
[`internal/rig` package](../packages/rig.md).

## Rig lifecycle states

A rig moves through several operational states visible on
`gt rig list` and `gt rig menu` as LED glyphs:

- **Operational** — the default, healthy running state. Patrols
  can be started; polecats can be spawned; crew can work.
- **Parked** — explicitly set non-operational via `gt rig
  park`. The rig exists but no new work is dispatched.
- **Docked** — explicitly non-operational via `gt rig dock`.
  Similar to parked with a different connotation ("out for
  maintenance").
- **Unreachable / unhealthy** — runtime-derived conditions,
  e.g. missing `.repo.git/`, bad `metadata.json` identity,
  Dolt unhealthy. These are detected by `gt doctor` and the
  rig's identity-verification pass at creation time
  (gas-tc4, `manager.go:871-909`).

The axes are independent: you can park a rig without stopping
its Witness and Refinery; you can stop all its patrols without
parking it. See [`gt rig`](../commands/rig.md) for the full
lifecycle command inventory.

## Rig structure: what the filesystem looks like

```
<townRoot>/
├── mayor/                  # Town-level Mayor
│   ├── town.json           # Town marker
│   └── rigs.json           # Registry of all rigs
├── deacon/                 # Town-level Deacon + dogs
├── .beads/                 # Town beads (mail, coord, town agents)
├── settings/config.json    # Town settings
├── plugins/                # Town-level plugins
├── <rig1>/                 # First rig
│   ├── config.json         # Rig config (beads prefix, default branch, ...)
│   ├── .repo.git/          # Shared bare clone
│   ├── refinery/rig/       # Refinery worktree (default branch)
│   ├── mayor/rig/          # Mayor's clone (separate from bare)
│   ├── witness/            # Witness dir (no clone)
│   ├── polecats/           # Polecat worktrees
│   │   ├── furiosa/
│   │   └── obsidian/
│   ├── crew/               # Crew worktrees + README.md
│   │   └── alice/
│   ├── .beads/             # Rig beads (OR redirect file)
│   ├── plugins/            # Rig-level plugins
│   ├── settings/           # Rig settings overrides
│   ├── formula-overlays/   # Rig-level formula step overrides
│   ├── directives/         # Rig-level role directives
│   └── .gitignore          # Defensive gitignore for GT dirs
└── <rig2>/
    └── ...
```

The layout is created by `rig.Manager.AddRig`
(`manager.go:301-866`) and is stable enough that other agents
can assume it. `rig.AgentDirs` is the programmatic list.

## How the concept is realized in the code

- **`internal/rig` package** — the implementation. See
  [`packages/rig.md`](../packages/rig.md) for the detailed
  walkthrough. Owns the `Rig` struct, the layered config,
  the `Manager` that creates/discovers/lists rigs, overlay
  file handling, and setup-hook execution.
- **`internal/cmd/rig*.go`** — the `gt rig` CLI surface:
  `rig.go`, `rig_config.go`, `rig_detect.go`, `rig_dock.go`,
  `rig_helpers.go`, `rig_park.go`, `rig_quick_add.go`,
  `rig_settings.go`. See [`gt rig`](../commands/rig.md).
- **`internal/mayor`** — the town-level Mayor that holds
  `rigs.json` (the canonical rig registry) and keeps a clone
  at `<rig>/mayor/rig/` for each.
- **`internal/polecat`** — polecats live under
  `<rig>/polecats/`. The polecat `Manager` is bound to a
  `*rig.Rig`; every polecat operation runs in a rig's
  namespace.
- **`internal/witness`** — per-rig watchdog; lives in
  `<rig>/witness/` with no clone.
- **`internal/refinery`** — per-rig merge queue processor;
  worktree at `<rig>/refinery/rig/`.
- **`internal/crew`** — user-managed persistent workspaces
  under `<rig>/crew/<name>/`.
- **`internal/plugin`** — the scanner is bound to `townRoot +
  rigNames` and walks both `<townRoot>/plugins/` and
  `<rig>/plugins/` per rig.
- **`internal/formula`** — patrol molecules and formulas are
  loaded from `.beads/formulas/` and embedded defaults; rig
  overlays live at `<rig>/formula-overlays/`.
- **`internal/config`** — `config.RigEntry` and
  `config.RigsConfig` are the town-level registry shapes.
  Directives (see [directive concept](directive.md)) resolve
  from town-level and rig-level directory trees.

## Rig-level vs town-level agents

| agent     | scope    | where it lives                | beads prefix         |
|-----------|----------|-------------------------------|----------------------|
| Mayor     | town     | `<town>/mayor/`               | town beads (`hq-`)   |
| Deacon    | town     | `<town>/deacon/`              | town beads (`hq-`)   |
| Witness   | rig      | `<rig>/witness/`              | rig beads (rig prefix) |
| Refinery  | rig      | `<rig>/refinery/rig/`         | rig beads (rig prefix) |
| Polecat   | rig      | `<rig>/polecats/<name>/`      | rig beads            |
| Crew      | rig      | `<rig>/crew/<name>/`          | rig beads            |
| Dog       | town     | `<town>/deacon/dogs/<name>/`  | town beads           |

Mayor's per-rig clone at `<rig>/mayor/rig/` is a working
directory, not an agent instance — the Mayor identity is
singular per town. This is the layout-vs-identity distinction
the codebase is careful about.

## Relationships with other concepts

- **town** — the enclosing installation. Many rigs per town.
- **[molecule](molecule.md)** — molecules run in a rig
  context (via `GT_RIG` + workspace detection). Patrol
  molecules are seeded per-rig by `AddRig →
  seedPatrolMolecules`.
- **[formula](formula.md)** — formulas are loaded from
  `.beads/formulas/` within a rig. Rig-level
  `formula-overlays/` let an operator customize step
  content without forking the formula.
- **[directive](directive.md)** — town-level and rig-level
  `directives/<role>.md` files concatenate to produce the
  effective directive for a role.
- **[identity](identity.md)** — an agent's identity chain
  always includes `GT_RIG` unless the agent is town-level
  (Mayor, Deacon).
- **[convoy](convoy.md)** — convoys dispatch work to polecats
  in specific rigs; `rigForIssue` maps an issue to its
  target rig via prefix routing.
- **[wisp](wisp.md)** — wisp-layer config
  (`.beads-wisp/config/<rig>.json`) is layer 1 of the rig
  config resolution chain.

## Related wiki pages

- [`internal/rig` package](../packages/rig.md) — the
  implementation walkthrough.
- [`gt rig`](../commands/rig.md) — the 20-subcommand CLI.
- [`internal/polecat`](../packages/polecat.md) — polecats
  live inside rigs.
- [`internal/witness`](../packages/witness.md) — per-rig
  watchdog.
- [`internal/refinery`](../packages/refinery.md) — per-rig
  merge queue processor.
- [`internal/mayor`](../packages/mayor.md) — town-level
  Mayor; manages the rigs registry.
- [`internal/crew`](../packages/crew.md) — persistent
  rig-scoped workspaces.
- [`internal/plugin`](../packages/plugin.md) — plugin
  scanner walks town + per-rig plugin dirs.
- [`internal/formula`](../packages/formula.md) — formulas
  and rig-level overlays.
- [molecule concept](molecule.md), [formula concept](formula.md),
  [identity concept](identity.md), [directive concept](directive.md).

## Notes / open questions

- **`town` is not yet a concept page.** The town is the
  enclosing installation and is referenced everywhere but
  doesn't have its own concept narrative yet. A future
  `concepts/town.md` would capture the town-level Mayor,
  Deacon, beads, and mail routing as a single concept.
- **The `mayor/rig/` clone is structurally confusing.** It's
  a clone owned by the Mayor agent but created during rig
  add. It exists to let the Mayor read the project without
  colliding with the Refinery's default-branch worktree. A
  reader seeing the path for the first time might assume it's
  a Mayor identity — it isn't.
- **Rig status LEDs have three axes (patrol, operational,
  data).** They're rendered as a single glyph, which can
  hide the distinction between "parked" and "witness down."
  Worth a future page explaining the LED legend.
- **`SourceTown` is implemented but not populated.** The
  placeholder comment at `config.go:76-78` has been in the
  tree long enough to be part of the normal config
  experience.
- **"Rig" is both a domain noun and a Go package name.** The
  concept is the workspace abstraction; the package is the
  implementation. This page is the concept; [`packages/rig.md`](../packages/rig.md)
  is the package. A third entity — `gt rig` CLI — is covered by
  [`commands/rig.md`](../commands/rig.md).
