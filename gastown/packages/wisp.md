---
title: internal/wisp
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-15
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
sources:
  - /home/kimberly/repos/gastown/internal/wisp/types.go
  - /home/kimberly/repos/gastown/internal/wisp/io.go
  - /home/kimberly/repos/gastown/internal/wisp/promotion.go
  - /home/kimberly/repos/gastown/internal/wisp/config.go
tags: [package, data, wisp, ephemeral-bead, ttl, compaction, beads-wisp-config]
---

# internal/wisp

Low-level utilities for the `.beads` and `.beads-wisp` directories
plus the promotion-criteria predicate for wisp compaction. The
package is deliberately small: it holds the directory constants,
atomic-JSON helpers, a rig-scoped key-value config store that lives
under `.beads-wisp/config/<rig>.json`, and the pure promotion
predicate that decides which wisps are worth keeping during
compaction.

**NOTE:** This internal/wisp package is NOT where the "wisp = bead"
domain concept is implemented. The domain concept — a wisp is an
ephemeral bead stored in the `wisps` SQL table, compactable by TTL,
created for rate-limits / scheduled work / molecule steps /
stranded-convoy feeds — is implemented in the [`beads` package](beads.md),
the [`doltserver` package](doltserver.md), and the beads SDK. See
[`wisp` concept](../concepts/wisp.md) for the domain story. This
package is just utilities that support that story (plus some legacy
`.beads` directory helpers whose original purpose — hook files —
has been deprecated).

**Go package path:** `github.com/steveyegge/gastown/internal/wisp`
**File count:** 4 non-test `.go` files — `types.go`, `io.go`,
`promotion.go`, `config.go`.
**Role:** NONE. Wisps are not an agent role — see
[`wisp` concept](../concepts/wisp.md) for why.
**CLI command:** NONE directly. `.beads-wisp/config/*.json` is
written by `gt wl` ([`gt wl`](../commands/wl.md) — wisp-config list)
and read by several other commands.
**Imports (notable):** [`internal/util`](util.md) for
`AtomicWriteJSON`, `EnsureDirAndWriteJSON`, `RemoveFromSlice`.
Stdlib only otherwise.
**Imported by (notable):** the [`doctor` package](go-packages.md)
(`wisp_check.go`, `misclassified_wisp_check.go`), various CLI
paths that need `WispDir` / `WispConfigDir` constants, test code,
and the promotion-compaction path that sweeps wisps using
`ShouldPromote` as the filter.

## What it actually does

Four small files, four orthogonal concerns.

### `types.go` — directory constant

Nine lines. Declares:

- `WispDir = ".beads"` (`types.go:9`).

The package doc comment is honest about the history: "This package
was originally for 'hook files' but those are now deprecated in
favor of pinned beads. The remaining utilities help with directory
management for the beads system." The name `WispDir = ".beads"` is
a misnomer — the directory is for beads infrastructure, not wisps
specifically. The name is preserved for compatibility; any renaming
has to chase down every `wisp.WispDir` call site.

### `io.go` — atomic JSON helpers

- `EnsureDir(root string) (string, error)` (`io.go:12-18`) — creates
  `<root>/.beads/` with 0755 and returns the path. Used by
  callers that want the directory to exist before writing.
- `WispPath(root, filename string) string` (`io.go:20-23`) — returns
  `<root>/.beads/<filename>`. Pure path join.
- `writeJSON(path, v interface{}) error` (`io.go:25-32`) — unexported
  wrapper around `util.AtomicWriteJSON`. Exists so other files in
  the package can share the same path for atomic writes.

### `promotion.go` — the compaction predicate

This is the load-bearing file for the wisp lifecycle. When the
reaper or compaction pass is deciding whether to drop a wisp, it
consults `ShouldPromote` — if any criterion is met, the wisp should
be PROMOTED to a permanent bead instead of being dropped. The
criteria are "this wisp has proven value and should survive".

- `KeepLabel = "gt:keep"` (`promotion.go:10`) — the explicit
  preservation label.
- `WispCandidate` struct (`promotion.go:14-29`) — the evaluation
  input: `CommentCount`, `NonWispRefCount`, `Labels`, `Age`, `TTL`.
  Populated from a `beads.Issue` by the caller.
- `HasComments(c) bool` (`promotion.go:33-35`) — someone discussed
  this wisp; it proved worth preserving.
- `IsReferenced(c) bool` (`promotion.go:39-41`) — a non-wisp bead
  links to this wisp; preserving it keeps that link meaningful.
- `HasKeepLabel(c) bool` (`promotion.go:44-51`) — the
  `gt:keep` label is present; an operator explicitly said "keep".
- `ShouldPromote(c) bool` (`promotion.go:59-61`) — the OR of all
  four criteria. Any one triggers promotion.
- `isPastTTL(c) bool` (`promotion.go:65-67`) — the fourth
  criterion: if the wisp is open past its TTL, something is stuck
  and needs attention — promoting it brings it to light rather
  than silently dropping.

The `ShouldPromote` doc comment enumerates the four criteria
verbatim:

1. Has comments — someone discussed it.
2. Is referenced — linked from a non-wisp bead.
3. Has keep label — explicitly flagged.
4. Open past TTL — something is stuck and needs attention.

These are the "wisp proved worth preserving" criteria. Everything
else gets compacted.

### `config.go` — rig-scoped key-value config

The `.beads-wisp/config/<rig>.json` storage layer for transient
local settings that should NEVER be synced via git. The
`.beads-wisp` directory is explicitly excluded from git (comment
at `config.go:15-16`). This is NOT the same as the wisp lifecycle
— it's a separate feature that happens to share the package because
both relate to the `.beads-wisp` directory.

- `WispConfigDir = ".beads-wisp"` (`config.go:16`).
- `ConfigSubdir = "config"` (`config.go:19`).
- `ConfigFile` struct (`config.go:23-27`) — JSON shape: `Rig`,
  `Values map[string]interface{}`, `Blocked []string`. Blocked
  keys return nil on Get and cannot be Set — "null value" semantics.
- `Config` struct (`config.go:31-36`) — the instance, holds a
  mutex, the town root, the rig name, and the computed file path.
- `NewConfig(townRoot, rigName) *Config` (`config.go:41-48`) —
  computes `<townRoot>/.beads-wisp/config/<rig>.json`.
- `Config.ConfigPath() string` (`config.go:51-53`).
- `Config.load() (*ConfigFile, error)` (`config.go:57-84`) —
  returns an empty `ConfigFile` if the file doesn't exist, so
  the caller never has to check for non-existence. Also
  initialises `Values` / `Blocked` maps after unmarshal so nil
  maps never escape.
- `Config.save(*ConfigFile) error` (`config.go:87-93`) — uses
  `util.EnsureDirAndWriteJSON` for atomic write + parent-dir
  creation.
- `Config.Get(key) interface{}` (`config.go:97-112`) — blocked
  keys return nil even if they have a value.
- `Config.GetString(key) string` (`config.go:116-122`),
  `Config.GetBool(key) bool` (`config.go:126-132`) — typed
  accessors that return the type's zero value on blocked keys or
  type mismatches.
- `Config.Set(key, value) error` (`config.go:136-152`) —
  silently ignores set on blocked keys. "Block takes precedence."
- `Config.Block(key) error` (`config.go:156-174`) — marks a key
  as null-value. Removes from `Values`, adds to `Blocked`.
- `Config.Unset(key) error` (`config.go:177-193`) — removes from
  BOTH lists. Unset != Block.
- `Config.IsBlocked(key) bool` (`config.go:196-206`).
- `Config.Keys() []string` (`config.go:219-246`) — all keys, set
  and blocked.
- `Config.All() map[string]interface{}` (`config.go:249-263`) —
  all non-blocked values.
- `Config.BlockedKeys() []string` (`config.go:266-278`).
- `Config.Clear() error` (`config.go:281-291`) — full reset.

Thread safety: `Config.mu sync.RWMutex`. Reads (`Get`, `Keys`,
`All`, etc.) take a read lock; writes (`Set`, `Block`, `Unset`,
`Clear`) take a write lock. Every method re-loads the file under
its lock rather than caching — this is a correctness-over-speed
trade since configs are small.

### Notable design choices

- **`.beads` and `.beads-wisp` are two different directories with
  two different lifetimes.** `.beads` is the persistent beads
  store (misleadingly named via `WispDir`); `.beads-wisp` is
  transient rig-scoped config. Merging them would lose
  git-sync-ability.
- **The package is NOT the domain wisp.** This is the biggest
  drift trap in the whole wiki. Developers assume "internal/wisp"
  implements the wisp bead type; it doesn't. The `wisps` SQL
  table, the TTL mechanism, the compaction sweep, and the
  "wisps are merge-requests too" insight all live in
  [`internal/beads`](beads.md) and
  [`internal/doltserver`](doltserver.md). See
  [`wisp` concept](../concepts/wisp.md).
- **Promotion predicate is pure.** `ShouldPromote` has zero side
  effects — you feed in a struct, you get a bool. The compaction
  sweep is responsible for populating `WispCandidate` from a
  beads issue and acting on the result. This keeps the predicate
  testable and lets callers swap in their own population logic.
- **"Promote" vs "preserve" semantics.** The code uses "promote"
  meaning "elevate to Level 1 / permanent bead". Semantically
  this is the same as "keep" — a wisp that promotes is no longer
  a wisp. The asymmetry is intentional: compaction drops; the
  only way to survive a sweep is to graduate out of wisp-hood.
- **Blocked vs Unset.** Block is persistent (a subsequent Set
  does nothing); Unset is temporary (a subsequent Set works).
  This distinction exists because the config is a key-value
  store with an explicit "never override this" marker.
- **Per-rig scoping.** `Config` is always bound to a rig name —
  there's no town-wide wisp config. Rigs are the natural unit of
  "local, non-synced config" because they correspond to work
  areas.

## Related wiki pages

- [`wisp` concept](../concepts/wisp.md) — the DOMAIN explanation
  of what a wisp is. **Required reading before this package page.**
- [`internal/beads`](beads.md) — where the `wisps` SQL table,
  `ListMergeRequests`, and wisp lifecycle actually live.
- [`internal/doltserver`](doltserver.md) — `wisps_migrate.go`
  owns the migration from the old hook-file world to the
  wisps-table world.
- [`reaper` package](reaper.md) — runs the compaction sweep that
  calls into `ShouldPromote`.
- [`refinery` package](refinery.md) — queries the wisps table via
  `ListMergeRequests`; MRs ARE wisps.
- [`witness` package](witness.md) — creates cleanup wisps via
  `createCleanupWisp`.
- [`gt wl`](../commands/wl.md) — the CLI that drives the
  `.beads-wisp/config/<rig>.json` store.
- [`gt reaper`](../commands/reaper.md) — the cleanup CLI the
  reaper package exposes.
- [`internal/util`](util.md) — `AtomicWriteJSON`,
  `EnsureDirAndWriteJSON`, `RemoveFromSlice`.
- [`doctor` package](go-packages.md) —
  `wisp_check.go` and `misclassified_wisp_check.go` are the
  health-check paths that consume wisp metadata.

## Notes / open questions

- **Name collision is painful.** `internal/wisp` does not
  implement wisps. A rename to `internal/beadsdirs` or
  `internal/localcfg` would clarify, but `wisp.WispDir` and
  `wisp.Config` are called from many places.
- **`.beads` directory is a misnomer.** It was originally the
  hook-file directory for wisps; hook files are gone; the name
  remains. The doc comment on `types.go:1-6` acknowledges this.
- **No cache in `Config`.** Every Get re-reads the file under the
  lock. This is fine for small configs but means a busy hot path
  would hit the disk often. Benchmarks or caching would be a
  future optimisation.
- **`Keys()` deduplication** relies on a linear scan at
  `config.go:234-245`. Fine for small configs; quadratic for
  large ones. A set-based rewrite is trivial but unnecessary
  today.
- **No schema version on `ConfigFile`.** If the shape changes the
  package has no way to detect old files. Since the files are
  ignored by git, this is low-risk — operators can delete and
  regenerate.
