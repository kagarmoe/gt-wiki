---
title: gt hooks
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/cmd/hooks.go
  - /home/kimberly/repos/gastown/internal/cmd/hooks_base.go
  - /home/kimberly/repos/gastown/internal/cmd/hooks_override.go
  - /home/kimberly/repos/gastown/internal/cmd/hooks_sync.go
  - /home/kimberly/repos/gastown/internal/cmd/hooks_diff.go
  - /home/kimberly/repos/gastown/internal/cmd/hooks_list.go
  - /home/kimberly/repos/gastown/internal/cmd/hooks_scan.go
  - /home/kimberly/repos/gastown/internal/cmd/hooks_registry.go
  - /home/kimberly/repos/gastown/internal/cmd/hooks_install.go
  - /home/kimberly/repos/gastown/internal/cmd/hooks_init.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, configuration, hooks, claude-code]
phase3_audited: 2026-04-15
phase3_findings: [wiki-stale, cobra-drift]
phase3_severities: [wrong]
phase3_findings_post_release: false
phase5_audience: user
phase8_audited: 2026-04-17
phase8_findings: [none]
detail_depth: {params: 2, data_flow: 1, errors: 1, side_effects: 1}
---

# gt hooks

Parent command for centralized management of Claude Code hooks
(`.claude/settings.json`) across the Gas Town workspace with a shared
base config and per-role/per-rig overrides. `hooks.go` itself declares
only the parent, but every subcommand its `Long` text advertises —
`base`, `override`, `sync`, `diff`, `list`, `scan`, `registry`,
`install` — plus a ninth subcommand `init` that the `Long` text does
**not** mention — is wired up in a dedicated sibling file
(`hooks_base.go`, `hooks_override.go`, etc.), each of which calls
`hooksCmd.AddCommand(...)` from its own `init()` (Phase 3 Batch 1b
wiki-stale correction: the Phase 2 body claimed the subcommands were
unwired).

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupConfig` ("Configuration")
**Polecat-safe:** no (no `AnnotationPolecatSafe` on the cobra.Command at
`/home/kimberly/repos/gastown/internal/cmd/hooks.go:7-40`)
**Beads-exempt:** no (not in `beadsExemptCommands` on `root.go:44-77` —
note that `hook` singular **is** exempt (`root.go:54`), but `hooks`
plural is a different command and is not listed)
**Branch-check-exempt:** no (not in `branchCheckExemptCommands` on
`root.go:81-91`)

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/hooks.go:7-44`
plus sibling `hooks_*.go` files.

### Invocation

```
gt hooks <subcommand>
```

### Behavior

Parent `hooksCmd` is declared in `hooks.go:7-40` and enforces
`RunE: requireSubcommand` (`hooks.go:39`) — without a subcommand,
returns an error and prints help. `hooks.go`'s own `init()` only
registers the parent on the root (`hooks.go:42-44`):

```go
func init() {
    rootCmd.AddCommand(hooksCmd)
}
```

Each subcommand lives in its own sibling file and calls
`hooksCmd.AddCommand(...)` from that file's `init()`:

- `hooks_base.go:32` — `hooksBaseCmd`
- `hooks_override.go:38` — `hooksOverrideCmd`
- `hooks_sync.go:42` — `hooksSyncCmd`
- `hooks_diff.go:34` — `hooksDiffCmd`
- `hooks_list.go:32` — `hooksListCmd`
- `hooks_scan.go:41` — `hooksScanCmd`
- `hooks_registry.go:51` — `hooksRegistryCmd`
- `hooks_install.go:41` — `hooksInstallCmd`
- `hooks_init.go:32` — `hooksInitCmd`

Nine subcommands total. Go init ordering wires them all at startup.

### Subcommands

Eight of the nine subcommands appear in the parent `hooksCmd.Long`
text at `hooks.go:17-25`:

- `base` — edit the shared base hook config (`hooks_base.go`)
- `override` — edit overrides for a role or rig (`hooks_override.go`)
- `sync` — regenerate all `.claude/settings.json` files (`hooks_sync.go`)
- `diff` — show what `sync` would change (`hooks_diff.go`)
- `list` — show all managed `settings.json` locations (`hooks_list.go`)
- `scan` — scan workspace for existing hooks (`hooks_scan.go`)
- `registry` — list hooks from the registry (`hooks_registry.go`)
- `install` — install a hook from the registry (`hooks_install.go`)

A ninth subcommand, `init`, is wired in `hooks_init.go:31-34` and has
its own `Long` text at `hooks_init.go:17-27` describing its purpose
(bootstrap the hooks base config by analyzing existing
`settings.json` files) and the `--dry-run` flag. **It is not listed
in the parent `hooksCmd.Long`** — see `## Drift` below.

### Flags

None declared on the parent command itself; each subcommand
declares its own flags in its sibling file's `init()`.

### Config structure (per Long text)

`hooks.go:27-31`:

- Base: `~/.gt/hooks-base.json`
- Overrides: `~/.gt/hooks-overrides/<target>.json`
- Merge strategy: base → role → rig+role (more specific wins)

This differs from the `~/.local/state/gastown/` and
`~/.config/gastown/` state paths used by `enable`/`disable`/`uninstall`
(see [uninstall.md](uninstall.md)). The hooks system has its own
`~/.gt/` namespace.

## Docs claim

### Source
- `/home/kimberly/repos/gastown/internal/cmd/hooks.go:11-38` — Cobra `Long` text on the parent `hooksCmd`.

### Verbatim

> Manage Claude Code hooks across the Gas Town workspace.
>
> Provides centralized hook configuration with a base config and
> per-role/per-rig overrides. Changes are propagated to all workers
> via the sync command.
>
> Subcommands:
>   base       Edit the shared base hook config
>   override   Edit overrides for a role or rig
>   sync       Regenerate all .claude/settings.json files
>   diff       Show what sync would change
>   list       Show all managed settings.json locations
>   scan       Scan workspace for existing hooks
>   registry   List hooks from the registry
>   install    Install a hook from the registry
>
> Config structure:
>   Base:      ~/.gt/hooks-base.json
>   Overrides: ~/.gt/hooks-overrides/<target>.json
>
> Merge strategy: base → role → rig+role (more specific wins)
>
> Examples:
>   gt hooks sync           # Regenerate all settings.json files
>   gt hooks diff           # Preview what sync would change
>   gt hooks base           # Edit the shared base config
>   gt hooks override crew  # Edit overrides for all crew workers
>   gt hooks list           # Show managed locations and sync status

## Drift

See forward-link: [../drift/README.md](../drift/README.md).

### Parent `Long` text enumerates 8 subcommands; `hooks init` is wired but unlisted

- **Claim source:** Cobra `Long` text at `/home/kimberly/repos/gastown/internal/cmd/hooks.go:17-25`.
- **Docs claim:** the `Subcommands:` block names exactly eight subcommands — `base`, `override`, `sync`, `diff`, `list`, `scan`, `registry`, `install`. A user reading `gt hooks --help` would reasonably conclude those are the complete subcommand set.
- **Code does:** `hooks_init.go:14-29` defines a ninth user-facing subcommand `hooksInitCmd` with `Use: "init"`, `Short: "Bootstrap base config from existing settings.json files"`, a multi-line `Long` describing the bootstrap flow, and a `--dry-run` flag. `hooks_init.go:31-34` registers it on `hooksCmd` via `hooksCmd.AddCommand(hooksInitCmd)`. `runHooksInit` at `hooks_init.go:36-150` implements the full bootstrap (scan targets → intersect common hooks → compute overrides → write base + override files). This is a first-class subcommand, not a helper or test fixture; the cobra-generated `Available Commands:` block when running `gt hooks` bare does list it, but the hand-maintained `Subcommands:` section of the parent `Long` text does not.
- **Category:** `cobra drift`
- **Severity:** `wrong`
- **Fix tier:** `code` — add an `init` entry to the `Subcommands:` block in `hooksCmd.Long` at `hooks.go:17-25`, mirroring the wording of `hooks_init.go:16` (`Bootstrap base config from existing settings.json files`). A matching `gt hooks init` example in the `Examples:` block at `hooks.go:33-38` is optional but would improve symmetry with the other listed subcommands.
- **Release position:** `in-release` (parent `Long` text byte-identical at `v1.0.0:internal/cmd/hooks.go:17-25`; `hooks_init.go` already present at v1.0.0, same registration).

### Phase 2 wiki body incorrectly claimed subcommand wiring was missing (wiki-stale)

- **Category:** `wiki-stale`
- **Severity:** `wrong`
- **Phase 2 claim (removed):** the Phase 2 page body stated "`hooks.go` only declares the parent. All eight advertised subcommands must be registered somewhere else under `internal/cmd/` (or not at all)" and listed a follow-up "grep for `hooksCmd.AddCommand`."
- **Current reality (2026-04-15, gastown HEAD `9f962c4a`):** all nine subcommands — the eight advertised plus the unlisted `init` — are wired on `hooksCmd` by their respective sibling `hooks_*.go` files via per-file `init()` blocks. Confirmed via `grep -rn "hooksCmd.AddCommand" internal/cmd/`. Every one of the sibling files existed at `v1.0.0`, so this was stale at Phase 2 time — not churn-induced drift.
- **Fix tier:** `wiki` — already fixed inline in `## What it actually does` above (the "Behavior" section now enumerates the sibling-file registrations; the "Subcommands" section now reflects that they are all wired).

## Failure modes

No failure modes identified in `hooks.go` itself. The parent command is a pure `requireSubcommand` dispatcher with no error paths of its own. Failure modes for the nine subcommands (`base`, `override`, `sync`, `diff`, `list`, `scan`, `registry`, `install`, `init`) live in their respective sibling files (`hooks_base.go`, etc.) and would be audited on per-subcommand pages if those pages existed. The parent file at `hooks.go:7-44` has no assumptions, no cleanup, and no suppressed errors.

## Related

- [gt](../binaries/gt.md) — parent binary.
- [README.md](README.md) — command tree index.
- [doctor.md](doctor.md) — `doctor` runs a hooks-sync health check
  (`hooks sync` drift) and a hook-attachment check — see the
  "Hooks sync" and "Hook attachment" check blocks at
  `doctor.go:259-265`.
- [prime.md](prime.md) — `gt prime` is invoked from a Claude Code
  `SessionStart` hook; `hooks sync` regenerates the `settings.json`
  files that wire that hook up.
- [upgrade.md](upgrade.md) — post-install migration; likely the
  consumer that invokes `hooks sync` automatically after upgrade.
- [config.md](config.md) — sibling Configuration-group command for
  town settings. Hooks config is explicitly a separate system
  (`~/.gt/hooks-*.json`) from town settings (`settings/config.json`).

## Outgoing calls

### Subprocess invocations
| Called binary | Command | Flags | Flag source | `file:line` |
|---|---|---|---|---|
| `$EDITOR` (or `vi`) | `<basePath>` | — | `EDITOR` env var, fallback `vi` | `hooks_base.go:66` |
| `$EDITOR` (or `vi`) | `<overridePath>` | — | `EDITOR` env var, fallback `vi` | `hooks_override.go:78` |

### Config file writes
| Target | Operation | Value | Purpose | `file:line` |
|---|---|---|---|---|
| `settings.json` (sync target) | `os.WriteFile` | marshaled hooks settings | sync hooks into Claude Code settings | `hooks_sync.go:305` |
| `settings.json` (install target) | `os.WriteFile` | marshaled hooks settings | install hooks into Claude Code settings | `hooks_install.go:299` |

## Notes / open questions

- **Phase 3 Batch 1b wiki-stale correction (2026-04-15):** → promoted to `## Drift` — see the `wiki-stale` finding above. The Phase 2 claim that subcommands were unwired has been replaced with the actual sibling-file registrations.
- **`hook` vs `hooks`.** `root.go:54` exempts `hook` from the beads
  check, which is described as "Hook signal handlers must be fast,
  handle beads internally". That exempt-map entry is for a different
  command — likely a signal-handler command registered elsewhere, not
  this `hooks` management command. Two separate command trees.
- `~/.gt/hooks-base.json` is a new path not in the Layer-(c) file
  layer yet; add `files/hooks-base-json.md` when the shape is mapped.
- `registry` and `install` subcommands imply a hooks registry — worth
  its own `concepts/hook-registry.md` page when source is mapped.
