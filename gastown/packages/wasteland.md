---
title: internal/wasteland
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-15
sources:
  - /home/kimberly/repos/gastown/internal/wasteland/wasteland.go
  - /home/kimberly/repos/gastown/internal/wasteland/spider.go
  - /home/kimberly/repos/gastown/internal/wasteland/trust.go
tags: [package, wasteland, federation, dolthub, reputation, trust]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
---

# internal/wasteland

Implementation of the Wasteland — a federation of Gas Towns over
DoltHub. Each rig has a sovereign fork of a shared commons database.
This package handles the join workflow (fork → clone → register →
push), fraud detection over the stamps graph (the Spider Protocol),
and trust-tier escalation from Drifter → Registered → Contributor →
War Chief.

**Go package path:** `github.com/steveyegge/gastown/internal/wasteland`
**File count:** 3 go files, 6 test files
**Imports (notable):** stdlib (`bytes`, `encoding/csv`,
`encoding/json`, `math`, `net/http`, `os/exec`, `regexp`), plus
[`internal/util`](util.md) for
`SetDetachedProcessGroup`. All Dolt interaction happens via the
`dolt` CLI subprocess rather than a Go client — the package does not
import the dolt Go driver.
**Imported by (notable):** [`gt wl`](../commands/wl.md) — the entire
wasteland command tree (join/fetch/contribute/stamp/spider/tier etc.)
calls into this package. No long-running service consumes it directly;
DoltHub is the data plane.

## What it actually does

### Config & join workflow (`wasteland.go`)

- `ErrNotJoined` (`wasteland.go:27`) — sentinel wrapped into a "run
  `gt wl join <upstream>`" message when `LoadConfig` can't find the
  file.
- `Config` (`wasteland.go:30-48`) — per-town record of the wasteland
  the rig joined: `Upstream` (e.g., `"steveyegge/wl-commons"`),
  `ForkOrg`, `ForkDB`, absolute `LocalDir`, `RigHandle`, `JoinedAt`.
  Stored at `<townRoot>/mayor/wasteland.json`
  (`wasteland.go:51-79`).
- `ParseUpstream(upstream)` (`wasteland.go:89-95`) — splits
  `"org/database"` into two non-empty parts or errors.
- `ForkDoltHubRepo(fromOrg, fromDB, toOrg, token)`
  (`wasteland.go:99-142`) — POST to
  `https://www.dolthub.com/api/v1alpha1/fork` with a bearer token.
  Treats "already exists" as success. The API base is declared as a
  `var` (`wasteland.go:83`) so tests can redirect it.
- `CloneLocally(org, db, targetDir)` (`wasteland.go:146-165`) —
  shells out to `dolt clone` against the Dolt remote API at
  `https://doltremoteapi.dolthub.com/...`. Idempotent: skips if
  `<targetDir>/.dolt` already exists.
- `RegisterRig(localDir, handle, dolthubOrg, displayName,
  ownerEmail, gtVersion)` (`wasteland.go:169-212`) — runs
  `dolt sql -q "INSERT INTO rigs ... ON DUPLICATE KEY UPDATE ..."`
  with SQL-escaped fields, then `dolt add .` and
  `dolt commit -m "Register rig: <handle>"`. "Nothing to commit"
  output is treated as success (already registered).
- `PushToOrigin(localDir)` (`wasteland.go:215-224`) — `dolt push
  origin main`, bubbling stderr on failure.
- `AddUpstreamRemote(localDir, upstreamOrg, upstreamDB)`
  (`wasteland.go:227-255`) — adds `upstream` remote if not already
  present; tolerates "already exists".
- `WastelandDir(townRoot)` (`wasteland.go:258-260`) returns
  `<townRoot>/.wasteland`. `LocalCloneDir(townRoot, org, db)`
  (`wasteland.go:263-265`) returns `<.wasteland>/<org>/<db>`.
- `DoltHubAPI` / `DoltCLI` / `ConfigStore` interfaces
  (`wasteland.go:273-290`) abstract the three external I/O surfaces
  so the join orchestrator is unit-testable.
- `Service` / `NewService` (`wasteland.go:293-404`) — orchestrator
  carrying the three interfaces plus an optional `OnProgress`
  callback for UI updates during `Join`.
- `Service.Join(upstream, forkOrg, token, handle, displayName,
  ownerEmail, gtVersion, townRoot)` (`wasteland.go:302-362`) —
  orchestrates the workflow in order: parse upstream → check existing
  config (reject if joining to a different upstream) → fork commons
  → clone fork locally → add upstream remote → register rig → push
  → save config. Each step emits a progress message.

### Spider Protocol — fraud detection (`spider.go`)

- `FraudSignalKind` (`spider.go:32-47`) — four kinds:
  `SignalCollusion`, `SignalRubberStamp`,
  `SignalConfidenceInflation`, `SignalSelfLoop`.
- `FraudSignal` (`spider.go:50-58`) — detected anomaly: kind, rigs
  involved, severity `Score` in [0, 1], human-readable `Detail`,
  raw `Evidence` string, and `SampleSize`.
- `SpiderConfig` (`spider.go:61-85`) — threshold knobs:
  `MinStampsForCollusion`, `CollusionRatioThreshold`,
  `RubberStampMinCount`, `ConfidenceFloor`, `ConfidenceMinStamps`.
- `DefaultSpiderConfig()` (`spider.go:89-97`) — production defaults
  calibrated for "50-200 active rigs": min 3 mutual stamps for
  collusion, 0.5 ratio threshold, min 5 identical stamps for rubber
  stamp, 0.95 confidence floor.
- `collusionQuery(cfg)` (`spider.go:105-125`) — self-join on stamps
  that flags `(A, B)` pairs where ≥threshold of A's stamps go to B
  **and** B reciprocally stamps A. The `HAVING` clause uses three
  computed columns with nested subqueries for rig totals.
- `rubberStampQuery(cfg)` (`spider.go:130-144`) — groups stamps by
  `(author, JSON_EXTRACT(valence, '$'))` to find validators that
  emit identical valence JSON across many stamps — suggests the
  validator isn't actually reviewing.
- `confidenceInflationQuery(cfg)` (`spider.go:149-167`) — flags
  authors whose average confidence is above the floor with many
  stamps. The narrow spread (always near-max) is what makes this
  suspicious, not high confidence per se.
- `selfLoopQuery()` (`spider.go:173-189`) — detects pairs `(A, B)`
  where both directions have ≥2 stamps and the loop accounts for
  most of both rigs' activity.
- `RunSpiderDetection(doltPath, forkDir, cfg)`
  (`spider.go:200-244`) — the top-level entry point. Runs each
  detector independently; partial results are preferred to none.
  Returns an empty slice (not an error) when nothing is found.
- `scoreCollusion` / `scoreRubberStamp` / `scoreConfidenceInflation` /
  `scoreSelfLoop` (`spider.go:249-300`) — convert per-row query
  output into a severity score. Confidence inflation maps
  [0.95, 1.0] → [0.5, 1.0]; self-loop scoring is based on symmetry
  (`min(forward, reverse) / max(forward, reverse)`).
- `runDoltQuery(doltPath, forkDir, query)` (`spider.go:328-349`)
  shells out via `dolt sql -r csv -q <query>`, captures stderr for
  error messages, and returns result rows (minus the header).

### Trust tier escalation (`trust.go`)

- `TrustTier` (`trust.go:29-36`) — four levels: `TierDrifter` (0),
  `TierRegistered` (1), `TierContributor` (2), `TierWarChief` (3).
  `String()` returns the human name.
- `validHandle` (`trust.go:39`) — allowlist regex
  `^[a-zA-Z0-9_-]+$` used to prevent SQL injection in the handle
  parameter to the ad-hoc dolt queries.
- `TierRequirements` (`trust.go:60-83`) — criteria: min completions,
  min stamps, min avg quality (extracted from `valence.quality`),
  min distinct validators (Spider Protocol's first line of defense
  against collusion), min time in current tier.
- `DefaultTierRequirements()` (`trust.go:92-117`) — the production
  escalation schedule:
  - Registered: automatic on join.
  - Contributor: 3 completions, 3 stamps, 2 validators, 7 days.
  - War Chief: 10 completions, 15 stamps, avg quality ≥3.5, 5
    validators, 30 days.
- `RigTrustProfile` (`trust.go:122-130`) — captured metrics for one
  rig: handle, current tier, when they reached it, completion count,
  stamp count, avg quality, distinct validators.
- `EscalationResult` (`trust.go:133-140`) — `Eligible`, `NextTier`,
  and `Reasons` listing either "all criteria met" or every unmet
  criterion with its current-vs-required comparison.
- `EvaluateEscalation(profile, requirements)` (`trust.go:147-217`)
  — pure function: looks up requirements for `profile.CurrentTier+1`,
  checks each criterion, collects failure strings. Every field must
  pass — partial matches don't count.
- `LoadRigTrustProfile(doltPath, forkDir, handle)`
  (`trust.go:225-278`) — queries the local dolt fork: current
  `trust_level` and `registered_at` from `rigs`, validated
  completion count from `completions INNER JOIN stamps`, and
  stamp-count / avg-quality / distinct-validators from `stamps
  WHERE subject = handle`. Rejects handles that don't match the
  validHandle regex.
- `runTrustQuery(doltPath, forkDir, query)` (`trust.go:283-303`)
  — the shared dolt CSV helper, analogous to spider's
  `runDoltQuery`.

## Related wiki pages

- [gt](../binaries/gt.md) — binary containing the `gt wl` command tree.
- [gt wl](../commands/wl.md) — user-facing wasteland CLI
  (join, fetch, stamp, spider, tier, etc.).
- [internal/doltserver](doltserver.md) — local Dolt SQL server
  managed by gt. Wasteland's dolt calls are subprocess shells out to
  the `dolt` binary, not calls into this Go package.
- [internal/util](util.md) — provides `SetDetachedProcessGroup`
  used on every dolt subprocess.
- [go-packages inventory](../inventory/go-packages.md).

## Notes / open questions

- All SQL in this package is constructed with `fmt.Sprintf` and
  manual escaping (`escapeSQLString` in `wasteland.go:268-271`,
  `validHandle` guard in `trust.go:230-232`). There is no prepared
  statement path because every call shells out to `dolt sql -q`
  rather than using a database/sql driver.
- The comment at `wasteland.go:8` references
  `~/hop/docs/wasteland/design.md` as the full design doc — that
  path is not in-repo and won't survive a fresh clone; the design is
  effectively only captured in code comments for LLM readers.
- `RunSpiderDetection` returns `nil, err` **only** if ALL detectors
  failed (`spider.go:239-241`). Individual detector failures during
  a run are swallowed silently and only surface if every detector
  fails.
- `LoadRigTrustProfile` approximates `TierSince` with the rig's
  `registered_at` timestamp because the schema doesn't (yet) track
  when each tier was reached (`trust.go:243-249`). A newly promoted
  War Chief will see their War Chief time-in-tier counted from
  join time, which is wrong but conservative — it will never
  under-gate the next escalation.
