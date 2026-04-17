---
title: internal/rig
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
sources:
  - /home/kimberly/repos/gastown/internal/rig/types.go
  - /home/kimberly/repos/gastown/internal/rig/config.go
  - /home/kimberly/repos/gastown/internal/rig/manager.go
  - /home/kimberly/repos/gastown/internal/rig/overlay.go
  - /home/kimberly/repos/gastown/internal/rig/setuphooks.go
tags: [package, agent-runtime, rig, workspace, layered-config, bare-repo, worktree, patrol-molecules, plugins]
---

# internal/rig

The workspace-manager package: creates rigs (clone + scaffold +
agent beads + patrol molecules + plugin dirs), loads them from
disk, discovers the polecats/crew inside them, resolves per-rig
config through a four-layer override chain, applies overlay
files and setup hooks to newly-created worktrees, and owns the
`config.json` / rigs-in-town-config shape. Every agent-runtime
package that cares about "which rig am I in?" ultimately touches
a `*rig.Rig` from here.

**Go package path:** `github.com/steveyegge/gastown/internal/rig`
**File count:** 5 non-test `.go` files. `manager.go` dominates
at 1805 lines; `config.go` is 219 lines; `overlay.go` is 306;
`setuphooks.go` is 130; `types.go` is 98.
**Concept:** [rig concept](../concepts/rig.md) — what a rig IS
in Gas Town's mental model.
**CLI command:** [`gt rig`](../commands/rig.md) — 20+
subcommands across 8 sibling files in `internal/cmd/rig*.go`.
**Imports (notable):** [`internal/beads`](beads.md),
[`internal/config`](config.md), `internal/constants`,
[`internal/doltserver`](doltserver.md), `internal/git`,
`internal/hooks`, `internal/style`, `internal/templates/commands`,
[`internal/util`](util.md), [`internal/wisp`](wisp.md).
**Imported by:** every `gt rig` subcommand
(`internal/cmd/rig*.go`), the
[`polecat` package](polecat.md) (`NewManager` binds a `*rig.Rig`),
[`witness`](witness.md) (per-rig patrol attach), and
the `internal/plugin` scanner indirectly via `rigNames`.

## What it actually does

Five files. Three concerns: the `Rig` data shape, the layered
config-resolution API, and the `Manager` that creates / discovers
/ deletes rigs. Two more concerns handled in small files: overlay
file copy + gitignore maintenance (`overlay.go`) and
executable-script setup hooks (`setuphooks.go`).

### types.go — the `Rig` data shape and agent dirs

Source: `/home/kimberly/repos/gastown/internal/rig/types.go`.

- `Rig` struct (`types.go:9-44`) — `Name`, `Path`, `GitURL`,
  `PushURL`, `LocalRepo`, `Config *config.BeadsConfig`,
  `Polecats []string`, `Crew []string`, `HasWitness`,
  `HasRefinery`, `HasMayor`. The boolean flags are
  filesystem-derived: `loadRig` (`manager.go:178-243`) checks
  the conventional directories under the rig path and sets them.
- `AgentDirs` (`types.go:48-54`) — the canonical list of agent
  directories a rig container holds:
  `polecats`, `crew`, `refinery/rig`, `witness`, `mayor/rig`.
  Note the comment: **"witness doesn't have a /rig subdirectory
  (no clone needed)"**. The Witness is the only agent that
  watches the rig from outside any clone.
- `RigSummary` (`types.go:57-63`) — the compact shape
  (`PolecatCount`, `CrewCount`, `HasWitness`, `HasRefinery`) used
  by status listings.
- `BeadsPath() string` (`types.go:86-88`) — returns the rig root
  path. The comment is important: **the rig root's `.beads/`
  contains either a local beads database OR a `redirect` file
  pointing at `mayor/rig/.beads/`** (when the project tracks its
  own `.beads/` in git). `beads.ResolveBeadsDir` follows the
  redirect. This is how gastown supports both "we own the beads"
  and "the project we wrap already owns the beads" cases without
  a flag.
- `DefaultBranch() string` (`types.go:92-98`) — loads
  `config.json` and returns its `default_branch`, falling back to
  `"main"`. Used anywhere a rig-relative branch target is needed.

### config.go — layered config resolution

Source: `/home/kimberly/repos/gastown/internal/rig/config.go`.

This is the `*rig.Rig.GetConfig()` API — a four-layer override
chain that agents use to ask "what is this rig's value for
`max_polecats`/`default_formula`/`auto_restart`/..." without
caring where the value came from.

**Layers** (`config.go:16-23`), in priority order (first win):

1. **`SourceWisp`** — local wisp layer, `wisp.NewConfig(townRoot,
   rigName)`. Reads from `.beads-wisp/config/<rig>.json`. Can
   also *explicitly block* a key (`SourceBlocked`).
2. **`SourceBead`** — labels on the rig identity bead.
   `getBeadLabel` (`config.go:168-196`) reads the rig bead via
   `beads.NewWithBeadsDir` and walks its labels looking for
   `<key>:<value>` format. This is how operators can flag a rig
   via `bd label` without touching files.
3. **`SourceTown`** — town defaults. Currently a placeholder
   (`config.go:76-78`): the code comment says "Town defaults for
   operational state would typically be in
   `~/gt/settings/config.json`. For now, we skip directly to
   system defaults."
4. **`SourceSystem`** — compiled-in `SystemDefaults` map
   (`config.go:33-42`). Current defaults:
   - `status: "operational"`
   - `auto_restart: true`
   - `auto_start_on_up: false`
   - `max_polecats: 10`
   - `priority_adjustment: 0`
   - `dnd: false`
   - `polecat_branch_template: ""`
   - `default_formula: "mol-polecat-work"`

`GetConfigWithSource` (`config.go:59-87`) returns a
`ConfigResult{Value, Source}` so callers can tell where a value
came from, which is surfaced by `gt rig config show`.

**Stacking keys** (`config.go:46-48`): `priority_adjustment` is
the only one. For stacking keys, `GetIntConfig` sums layers
instead of picking one (`config.go:111-148`). This lets a rig's
town-level +1 and wisp-level +2 stack into a total +3 instead of
one overriding the other.

Typed accessors:

- `GetBoolConfig(key)` (`config.go:91-106`) — returns `false` for
  nil/blocked/non-bool; also accepts `"true"`/`"1"`/`"yes"` from
  bead labels (which are always strings).
- `GetIntConfig(key)` (`config.go:111-148`) — stacking-aware;
  blocked returns 0.
- `GetStringConfig(key)` (`config.go:152-164`) — empty string for
  blocked or non-string.
- `toInt(v)` (`config.go:199-219`) — coerces `int`, `int64`,
  `float64`, or parseable `string` into an `int`; anything else
  is 0.

This layer is what the [`rig`](../concepts/rig.md) concept
page calls "a rig's effective config." The layers are the
concept; this file is the mechanism.

### manager.go — the rig lifecycle engine

Source: `/home/kimberly/repos/gastown/internal/rig/manager.go`.

~1800 lines. Owns everything about creating, registering,
loading, listing, and removing rigs.

**Sentinel errors** (`manager.go:27-30`): `ErrRigNotFound`,
`ErrRigExists`.

**Reserved names** (`manager.go:35`): `"hq"` — rejected because
`EnsureMetadata` and Dolt routing special-case it as the
town-level beads alias.

**`RigConfig` struct** (`manager.go:84-101`) — the persistent
`config.json` shape: `Type`, `Version`, `Name`, `GitURL`,
`PushURL`, `UpstreamURL`, `LocalRepo`, `DefaultBranch`,
`CreatedAt`, `Beads` (with `Prefix`), `PolecatPoolSize`,
`PolecatNames`. `CurrentRigConfigVersion = 1`
(`manager.go:109`). `warnDeprecatedRigConfigKeys`
(`manager.go:940-953`) warns on now-removed `merge_queue.*`
keys — a useful precedent for retiring fields.

**Manager** (`manager.go:112-125`) — bound to `townRoot`,
`*config.RigsConfig`, and `*git.Git`. `NewManager` is trivial.

**Key public methods:**

- `DiscoverRigs() ([]*Rig, error)` (`manager.go:129-142`) —
  iterates `config.Rigs` and calls `loadRig` per entry. Failed
  loads are logged to stderr and skipped, so one broken rig
  doesn't block the list.
- `GetRig(name)` / `RigExists(name)` (`manager.go:144-158`) —
  single-rig lookups.
- `UsedNamepoolThemes(fallback)` (`manager.go:163-175`) — scans
  every rig's `settings/config.json` for an explicit namepool
  style, falling back to the supplied function. Used by
  `polecat.ThemeForRigAvoiding` to pick a non-colliding theme
  for a new rig.
- `loadRig(name, entry)` (`manager.go:178-243`) — filesystem-
  derived rig load: reads the rig dir, scans `polecats/` and
  `crew/` for subdirs (non-dotfile), checks for `witness/`,
  `refinery/rig/`, `mayor/rig/` to set the `Has*` booleans.
  ZFC-style: the filesystem is the source of truth, not a
  cached list.
- `AddRig(opts AddRigOptions)` (`manager.go:301-866`) — the
  heavyweight constructor. ~560 lines. Covered below.
- `RegisterRig(opts RegisterRigOptions)`
  (`manager.go:1430-1595`) — the "adopt existing directory"
  sibling: no clone, reuses the on-disk config and remotes.
  Auto-detects git URL from `origin` via `detectGitURL`
  (`manager.go:1641-1655`) and push URL from existing remotes
  (`detectPushURL` / `detectPushURLFrom`,
  `manager.go:1599-1634`).
- `InitBeads(rigPath, prefix, rigName)`
  (`manager.go:963-1098`) — runs `bd init` (or falls back to
  `beads.EnsureConfigYAML` if bd is missing), applies the
  `types.custom`/`issue_prefix` bd config, and drops the orphan
  `beads_<prefix>` database if it differs from `<rigName>`
  (gt-sv1h). Filters `BEADS_DIR` out of the environment and
  re-adds it explicitly so bd can't find a parent directory's
  `.beads/`. Bridges `GT_DOLT_PORT` / `GT_DOLT_HOST` into
  `BEADS_DOLT_PORT` / `BEADS_DOLT_SERVER_HOST` so bd
  subprocesses find the right Dolt server during tests or
  remote setups.
- `initAgentBeads(rigPath, rigName, prefix)`
  (`manager.go:1110-1165`) — creates the rig-level Witness and
  Refinery agent beads in rig beads. **Town-level agents
  (Mayor, Deacon) are NOT created here** — they're created by
  `gt install` in town beads (comment at `manager.go:1103`).
- `seedPatrolMolecules(rigPath)` / `seedPatrolMoleculesManually`
  (`manager.go:1667-1723`) — runs `bd mol seed --patrol` to
  seed the Deacon / Witness / Refinery patrol molecule
  prototypes. Falls back to creating them one at a time if
  `--patrol` isn't supported.
- `createPluginDirectories(rigPath)` (`manager.go:1728-1805`) —
  creates `~/gt/plugins/` (shared) and `<rig>/plugins/`
  (rig-specific), writes the town-level README, and appends a
  defensive `.gitignore` to the rig container covering all the
  GT infrastructure directories (`.beads/`, `.claude/`,
  `.runtime/`, `crew/`, `daemon/`, `mayor/`, `polecats/`,
  `refinery/`, `settings/`, `witness/`, plus `config.json`,
  `state.json`, `AGENTS.md`). The rig container isn't a git
  repo, but this protects against accidental `git init`.
- `RemoveRig(name)` (`manager.go:1397-1405`) — **config only**,
  does NOT delete files. The operator has to nuke the
  directory themselves.
- `ListRigNames()` (`manager.go:1657-1663`) — names from the
  town config.
- `verifyRigIdentity(rigPath, rigName)` (`manager.go:871-909`) —
  gas-tc4: checks `metadata.json` `dolt_database` matches the
  expected rig name. If mismatched, attempts `EnsureMetadata`
  repair and prints a clear warning. Caught early in the rig
  creation path so polecats don't hang in retry loops on a
  mis-named database.

**`AddRig` — the creation pipeline** (`manager.go:301-866`):

The shape of this function is the operational definition of a
rig. In order:

1. **Validate name** (`manager.go:306-321`): reject hyphens,
   dots, spaces, path separators. Suggest an underscored
   alternative. Reject reserved names (`hq`).
2. **Pre-flight Dolt** (`manager.go:323-331`): refuse to
   proceed if the Dolt server isn't running, because beads init
   requires it. Test seam: `SkipDoltCheck`.
3. **Directory collision** (`manager.go:336-338`): if the
   target path already exists, tell the user to use
   `--adopt`.
4. **Derive beads prefix** (`manager.go:340-352`):
   `deriveBeadsPrefix` is a small grammar (`manager.go:1202-1242`)
   that turns `gastown` → `gt`, `my-project` → `mp`,
   `foo` → `foo`. Splits on `-`/`_`, then tries camelCase and
   compound-word splits (`splitCamelCase`,
   `splitCompoundWord`) for single-word names, and falls back
   to the first 2-3 chars. Prefix collisions across rigs are
   caught by `beads.CheckPrefixAvailable`.
5. **Local repo resolution** (`manager.go:354-357`):
   `resolveLocalRepo` (`manager.go:259-288`) validates that a
   `--local-repo` path is a real git repo whose `origin`
   matches the target URL, so `git clone --reference` can use
   it as an object cache.
6. **Create rig dir** with a deferred cleanup function
   (`manager.go:360-371`) — rig directory is removed on any
   failure before `success = true`.
7. **Save `config.json`** (`manager.go:374-389`).
8. **Clone bare repo** (`manager.go:395-427`): creates
   `<rig>/.repo.git` as the shared source of truth for
   refinery + polecat worktrees. Variants:
   `CloneBarePartialWithReferenceAndBranch` (both filter + local
   ref), `CloneBarePartialWithBranch` (filter only),
   `CloneBareWithReferenceAndBranch` (local ref only),
   `CloneBareWithBranch` (plain). Each higher tier falls back
   to a lower tier on failure.
9. **Empty-repo detection** (`manager.go:433-437`): fail fast
   with a clear diagnostic instead of hitting an opaque
   checkout error later.
10. **Configure push URL and upstream** on the bare repo
    (`manager.go:441-454`).
11. **Resolve default branch** (`manager.go:457-482`) from
    `opts.DefaultBranch` or `bareGit.DefaultBranch()` (the
    latter reads `HEAD` from the bare clone, which points at
    the remote default). Fetches the branch explicitly if the
    shallow clone didn't include it.
12. **Mayor clone** (`manager.go:489-532`): `mayor/rig/` is a
    *separate* regular clone (NOT a worktree). The comment
    makes the rationale explicit: **"Mayor doesn't need to see
    polecat branches — that's refinery's job. This also allows
    mayor to stay on the default branch without conflicting
    with refinery."** Uses `--reference` to the bare repo to
    avoid a second remote download (GH#1059).
13. **Tracked-beads detection** (`manager.go:536-607`): if the
    source repo ships a tracked `.beads/` directory, detect
    its prefix from `config.yaml` (`detectBeadsPrefixFromConfig`,
    `manager.go:1366-1395`), refuse a `--prefix` mismatch,
    re-save the rig config with the detected prefix, and run
    `bd init` against the `mayor/rig` directory if the Dolt
    database doesn't exist yet. Writes `bd config set
    types.custom` and `issue_prefix` explicitly.
14. **Rig-level database + beads init** (`manager.go:613-666`):
    creates the Dolt server-side database via
    `doltserver.InitRig`, runs `m.InitBeads`, calls
    `doltserver.EnsureMetadata` so metadata.json has
    `dolt_mode=server` + `dolt_database=<rigName>`, drops
    orphan `beads_<prefix>` databases, and re-sets
    `issue_prefix` on the correct database.
15. **DoltHub remote** (`manager.go:671-683`): when
    `DOLTHUB_TOKEN` + `DOLTHUB_ORG` are set, auto-create the
    DoltHub remote for the rig's beads database and do an
    initial push. Non-fatal.
16. **`PRIME.md` provisioning** (`manager.go:690-693`) — writes
    the fallback Gas Town context that any bd-prime
    consumer can read.
17. **Refinery worktree** (`manager.go:698-720`): created via
    `bareGit.WorktreeAddExisting` pointing at the default
    branch — shared bare repo means refinery can see polecat
    branches without round-tripping the remote. Sets up the
    beads redirect and copies overlay files (see `overlay.go`).
18. **Crew dir + README** (`manager.go:727-752`).
19. **Witness dir** (`manager.go:754-757`) — empty. Witness
    hooks are installed later by
    `witness/manager.go:Start() → EnsureSettingsForRole`.
20. **Polecats dir + agent settings scaffold**
    (`manager.go:765-791`): uses the town's `default_agent`
    preset to install hooks and commands ahead of the first
    polecat session, so session startup has a valid settings
    file to load.
21. **Routes append** (`manager.go:796-809`): writes a
    `beads.Route{Prefix, Path}` into `routes.jsonl` BEFORE
    creating agent beads, so `ResolveRoutingTarget` doesn't
    log "no route found" warnings (gh#1424).
22. **Rig settings directory** (`manager.go:812-815`) — a bare
    `settings/` dir is created (no seeded content).
23. **Agent beads** (`manager.go:825-828`) — `initAgentBeads`
    creates the rig-level Witness and Refinery agent beads.
24. **Patrol molecules** (`manager.go:831-834`) —
    `seedPatrolMolecules`.
25. **Plugin dirs** (`manager.go:837-840`) —
    `createPluginDirectories` (town-level + rig-level + the
    protective `.gitignore`).
26. **Town config registration** (`manager.go:843-852`).
27. **Identity verification** (`manager.go:857-862`) —
    `verifyRigIdentity` (gas-tc4).
28. `success = true` (`manager.go:864`) — the deferred cleanup
    is disarmed and the rig is loaded via `loadRig`.

Order matters: routes.jsonl before agent beads, bd init before
EnsureMetadata before issue_prefix, DoltHub remote only after
the server-side DB exists, etc. Changing the sequence is a risk
vector because each step assumes the prior ones succeeded.

### overlay.go — overlay files + gitignore maintenance

Source: `/home/kimberly/repos/gastown/internal/rig/overlay.go`.

Three concerns, mostly unrelated to each other but sharing a
"files that need to appear in a new worktree" theme:

- `gasTownIgnorePatterns()` (`overlay.go:14-24`) — canonical
  list of patterns every Gas Town worktree's `.gitignore` must
  contain: `.runtime/`, `.claude/`, `.logs/`, `__pycache__/`,
  `state.json`, `CLAUDE.md`, `CLAUDE.local.md`.
  **`.beads/` is deliberately NOT here** — beads manages its
  own `.beads/.gitignore` and adding `.beads/` to the tracked
  gitignore breaks bd sync. See comment at `overlay.go:83-91`
  — this has regressed three times (PR#753/#891/#966) so a
  regression guard lives in `overlay_test.go`.
- `CopyOverlay(rigPath, destPath)` (`overlay.go:41-72`) — copies
  files (not subdirs) from `<rigPath>/.runtime/overlay/` into
  `destPath`. Used by `AddRig` to seed the refinery worktree
  with any overlay `.env` / config files the operator dropped
  in. Individual file failures are warnings, not fatal.
- `EnsureGitignorePatterns(worktreePath)` (`overlay.go:76-145`)
  — appends any missing `gasTownIgnorePatterns()` entries to
  the worktree's tracked `.gitignore`. Uses
  `matchesGitignorePattern` (`overlay.go:247-270`) which
  understands that a broader pattern (`.claude/`) covers
  narrower ones (`.claude/commands/`) — prevents duplicate
  entries when a project already ignores a parent directory.
- `EnsureLocalExcludePatterns(worktreePath)`
  (`overlay.go:164-224`) — writes the ignore patterns to the
  *per-worktree* `.git/info/exclude` file instead of the
  tracked `.gitignore`. Also includes `.beads/`
  (`gasTownLocalExcludePatterns`, `overlay.go:152-159`) —
  safe here because `.git/info/exclude` is never committed.
  Defense-in-depth for `gas-7vg`.
- `copyFilePreserveMode(src, dst)` (`overlay.go:273-306`) —
  preserves permissions and explicitly checks `Close()` so
  full-disk errors surface reliably.

### setuphooks.go — per-worktree setup scripts

Source: `/home/kimberly/repos/gastown/internal/rig/setuphooks.go`.

- `RunSetupHooks(rigPath, worktreePath)` (`setuphooks.go:41-97`)
  — executes every executable file in
  `<rigPath>/.runtime/setup-hooks/` in alphabetical order, with
  the worktree as cwd. Non-executable files are skipped with a
  warning. Hook failures are warnings, not errors.
- `runHook(hookPath, worktreePath)` (`setuphooks.go:107-130`) —
  runs a single hook with a 60-second `hookTimeout`
  (`setuphooks.go:105`). Injects `GT_WORKTREE_PATH` and
  `GT_RIG_PATH`.

The hook model is stable and uses filename-ordering (`01-…`,
`02-…`, `99-…`). Setup-hooks and plugin.md plugins are distinct
systems: setup-hooks run once when a worktree is created;
plugins run during Deacon patrol cycles. See
[`internal/plugin`](plugin.md).

## Notable design choices

- **Rig = container, not a git repo.** The rig root is a
  directory, not a checkout. `.repo.git/` is a shared bare
  clone; `refinery/rig/` is a worktree against that bare clone;
  `mayor/rig/` is a separate regular clone (for conflict-free
  default-branch parking); `crew/<name>/` and `polecats/<name>/`
  are also worktrees or clones managed by their own packages.
  The rig root itself never has tracked files.
- **Four-layer config with explicit source tracking.** The
  `ConfigResult{Value, Source}` shape lets `gt rig config show`
  display not just the effective value but where it came from,
  which is the difference between "this is the default" and
  "someone set this at the bead-label layer and you'll need to
  unset it there."
- **`SourceBlocked` is a distinct state.** A wisp-layer block
  returns `Value: nil, Source: SourceBlocked` — the caller can
  distinguish "no value found" from "value explicitly
  suppressed," which matters for auditability.
- **Filesystem-derived `Polecats`/`Crew`.** `loadRig` scans
  the directories every time; the list isn't cached on the
  `Rig` struct. Adding or removing a polecat from disk is
  immediately reflected without any refresh step.
- **ZFC identity verification.** `verifyRigIdentity` catches
  `metadata.json` drift early (gas-tc4). Without this check,
  a rig with a wrong `dolt_database` value would silently work
  for read operations but fail to spawn polecats, and the
  failure mode would surface ~3 minutes later in the polecat
  Dolt retry loop.
- **Patrol molecules and plugin directories are rig-creation
  responsibilities.** `AddRig` calls `seedPatrolMolecules` and
  `createPluginDirectories` so a freshly-created rig has a
  working Deacon loop *without* requiring a separate
  post-install step.
- **Local reference clones.** `--local-repo` + `git clone
  --reference` avoids a redundant remote download by sharing
  objects with an already-cloned local copy. `AddRig` validates
  the reference's `origin` matches the target URL so you can't
  accidentally cross-wire object stores.
- **The reserved name list is small.** `"hq"` only. The
  prefix-derivation grammar is a much richer namespace check:
  `deriveBeadsPrefix` + `beads.CheckPrefixAvailable` stops you
  from creating `widgets` and `web-services` side by side
  because both derive to `ws`.

## Related wiki pages

- [rig concept](../concepts/rig.md) — what a rig IS in Gas
  Town's mental model. This package is the implementation.
- [`gt rig`](../commands/rig.md) — the 20-subcommand CLI.
- [`internal/polecat`](polecat.md) — `polecat.NewManager(r, g,
  t)` binds a `*rig.Rig`; polecats live under `<rig>/polecats/`.
- [`internal/crew`](crew.md) — crew workspaces live under
  `<rig>/crew/`.
- [`internal/mayor`](mayor.md) — the mayor clone at
  `<rig>/mayor/rig/` is managed by the mayor package; `AddRig`
  creates the directory.
- [`internal/refinery`](refinery.md) — refinery worktree at
  `<rig>/refinery/rig/`.
- [`internal/witness`](witness.md) — witness directory at
  `<rig>/witness/` (no clone).
- [`internal/config`](config.md) — `RigsConfig`, `RigEntry`,
  and the town-level rigs map that `Manager` consumes.
- [`internal/beads`](beads.md) — `ResolveBeadsDir`,
  `CheckPrefixAvailable`, `AppendRoute`, `ProvisionPrimeMD`,
  `SetupRedirect`, `RigBeadIDWithPrefix`,
  `WitnessBeadIDWithPrefix`, `RefineryBeadIDWithPrefix`.
- [`internal/doltserver`](doltserver.md) — `IsRunning`,
  `InitRig`, `EnsureMetadata`, `RemoveDatabase`,
  `DoltHubToken`, `DoltHubOrg`, `RigDatabaseDir`,
  `SetupDoltHubRemote`.
- [`internal/workspace`](workspace.md) — `FindFromCwdOrError`
  is the entrypoint agents use to find the town root before
  asking `Manager` for a specific rig.
- [`internal/wisp`](wisp.md) — `wisp.NewConfig(townRoot,
  rigName)` is layer 1 of `GetConfigWithSource`.
- [`internal/formula`](formula.md) — patrol molecule templates
  come from formulas. `AddRig → seedPatrolMolecules` seeds
  them into the rig's beads.
- [`internal/plugin`](plugin.md) — plugin discovery needs the
  rig names from `ListRigNames()`.
- [`gt install`](../commands/install.md) — the town-level
  creator that Mayor/Deacon beads come from. Rig creation
  assumes the town already exists.
- [`gt doctor`](../commands/doctor.md) — consumes
  `verifyRigIdentity` failures and other health checks.

## Notes / open questions

- **`manager.go` at ~1800 lines is heavy for one file.** The
  `AddRig` pipeline alone is ~560 lines and composes steps that
  would read more clearly as named helpers. A split by concern
  (clone, beads, agents, plugins, verification) would help
  without changing semantics.
- **Town defaults layer is a placeholder.** `SourceTown` is
  listed in the enum but `GetConfigWithSource` skips it with a
  `// For now, we skip directly to system defaults` comment
  (`config.go:76-78`). Adding town-level defaults is a planned
  change but would affect how operators override the
  compiled-in `SystemDefaults`.
- **`RemoveRig` doesn't delete files.** Operators may expect
  it to. The gap is filled by `gt rig remove`'s confirmation
  prompt which handles on-disk removal separately (see
  [`gt rig`](../commands/rig.md)).
- **Overlay-file copy is non-recursive.** `CopyOverlay` only
  copies files at the overlay root. If the overlay directory
  grows a structure, the copier needs to change. Worth a flag
  or a recursive option.
- **Setup hooks don't cascade.** Each hook is run independently.
  Hooks have no way to signal "stop running subsequent hooks"
  short of `exit`. Since failures are warnings, a broken hook
  doesn't block later hooks from running in a possibly-broken
  state.
- **`deriveBeadsPrefix` collisions are silent-ish.** Two rigs
  whose names derive to the same prefix will be rejected by
  `CheckPrefixAvailable`, but the error message points at the
  bead layer, not at the prefix derivation. Worth a more
  self-describing error.
- **`AddRig` has no atomic-rollback for steps that succeed
  remotely.** The `cleanup` func removes the rig directory on
  local failure, but if `InitRig` creates a Dolt database and
  then bd init fails mid-way, the orphan Dolt DB persists. The
  explicit `RemoveDatabase` calls (`manager.go:590-591` and
  `1071-1075`) handle the known case; more exotic failures
  require `gt doctor --fix` or manual cleanup.
