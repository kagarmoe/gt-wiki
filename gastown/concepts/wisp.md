---
title: wisp (concept)
type: concept
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/wisp/promotion.go
  - /home/kimberly/repos/gastown/internal/refinery/manager.go
  - /home/kimberly/repos/gastown/internal/reaper/reaper.go
  - /home/kimberly/repos/gastown/internal/doltserver/wisps_migrate.go
  - /home/kimberly/repos/gastown/internal/witness/handlers.go
tags: [concept, wisp, ephemeral-bead, ttl, compaction, beads]
---

# wisp

A **wisp** is an ephemeral bead. Where a permanent bead is a
durable unit of work or record that survives indefinitely, a
wisp is a bead that exists for a bounded lifetime and is either
compacted away when its TTL expires or **promoted** to a
permanent bead if something proves it worth keeping. Wisps live
in a separate SQL table (`wisps`) from permanent beads
(`issues`) and are subject to a distinct reaper-driven cleanup
cycle. Merge requests, rate-limit notices, scheduled work items,
molecule steps, stranded-convoy feeds, and cleanup receipts are
all wisps. Permanent beads — epics, features, bugs, decisions,
agent identity beads — are not.

## Origin and purpose

Gas Town's bead ecosystem started with a single table: `issues`.
Every bead — including one-off messages, ephemeral tracking
records, transient merge state, and agent chatter — was a
permanent row. This caused three problems that compounded:

1. **Unbounded growth.** The `issues` table accumulated rows at
   a rate operators could not sustain. Query latency climbed.
2. **Noise in enumerations.** `bd ready`, `bd list`, and
   similar queries returned thousands of rows that no human or
   agent cared about. Every list was a haystack.
3. **Unclear lifetimes.** A bead that represented a real piece
   of work and a bead that represented "rate-limit warning
   sent at 14:02" looked identical in the schema. Cleanup had
   to guess.

The wisp concept solves all three problems with one move:
separate the two lifetimes into two tables. Ephemeral beads go
into `wisps`; permanent beads go into `issues`. The `wisps`
table has a TTL-based reaper; the `issues` table does not. A
wisp whose TTL expires is closed (and eventually purged)
unless the `ShouldPromote` predicate says otherwise.

**Beads v0.59** is the schema version where the `wisps` table
became authoritative (`internal/doltserver/wisps_migrate.go`
owns the migration from the old single-table world). Before
v0.59 the split didn't exist; after v0.59 it's a fact of the
schema.

## What distinguishes a wisp from a permanent bead

Explicit answer for future readers:

1. **Storage table.** Wisps live in the `wisps` SQL table;
   permanent beads live in `issues`. This is the defining
   structural difference. A bead is a wisp if and only if it
   is in `wisps`.
2. **TTL and compaction.** Wisps have an associated TTL. The
   reaper
   (`/home/kimberly/repos/gastown/internal/reaper/reaper.go:315-426`)
   closes wisps whose `created_at` is older than `--max-age`.
   Permanent beads are not subject to the reaper's close
   sweep.
3. **Promotion criteria.** A wisp can escape compaction by
   being **promoted** to Level 1 (permanent bead). The
   predicate is `ShouldPromote` in
   `/home/kimberly/repos/gastown/internal/wisp/promotion.go:59-61`.
   Four criteria, any one triggers promotion:
   - Has comments (`HasComments`) — someone discussed it.
   - Is referenced (`IsReferenced`) — a permanent bead links
     to it.
   - Has the `gt:keep` label (`HasKeepLabel`) — explicit
     operator flag.
   - Open past TTL (`isPastTTL`) — something is stuck and
     needs attention.
4. **Purge eligibility.** Closed wisps past `--purge-age` are
   *deleted* (`Purge` in reaper.go:428). Permanent beads are
   never deleted by the standard cleanup pipeline.
5. **Role in enumerations.** Wisps do NOT appear in
   `bd ready` or `bd list` by default. Agents see only
   permanent work unless they explicitly query the `wisps`
   table. This is the "noise suppression" benefit.
6. **What they represent.** A wisp is a record of *something
   happened, briefly, and may not matter later*. A permanent
   bead is a record of *a unit of work or decision that
   matters indefinitely*.

Examples of wisps: merge requests, agent rate-limit notices,
cleanup receipts the Witness creates after a polecat crash,
stranded-convoy feed markers, molecule step states, scheduled
work items. Examples of permanent beads: epics, features,
bugs, decisions, agent identity beads, retrospectives.

## Origin of "wisp" as a name

A wisp is a small, ephemeral flicker — the metaphor captures
both the "may exist only briefly" and "might still be
important" properties. The name is consistent with Gas Town's
other ephemeral-adjacent concepts (smoke, flame, gas).

## How it's realized in the code

The wisp concept is realized across several packages:

- **`internal/doltserver/wisps_migrate.go`** owns the schema
  migration from the old single-table world to the `wisps` +
  `issues` split. This is where the concept becomes a fact of
  the database.
- **The beads SDK** (`github.com/steveyegge/beads`) provides
  the `Wisp` / `ListWisps` / `ListMergeRequests` primitives
  that query the `wisps` table. Code in
  [`internal/beads`](../packages/beads.md) wraps the SDK.
- **`internal/wisp/promotion.go`** holds the `ShouldPromote`
  predicate — the pure function the reaper consults when
  deciding whether a wisp is worth saving. See
  [`internal/wisp` package](../packages/wisp.md) (note: this
  internal package is NOT where the wisp concept is
  implemented — it's a utility package for
  `.beads` / `.beads-wisp` directory management).
- **`internal/reaper/reaper.go`** is the cleanup engine. The
  `Scan`, `Reap`, `Purge`, `AutoClose` functions
  (`reaper.go:228+`) are where the wisp lifecycle is actually
  enforced.
- **`internal/refinery/manager.go:329-381`** (`Manager.Queue`)
  queries the `wisps` table via `beads.ListMergeRequests` —
  merge requests ARE wisps. The filter-by-rig at
  `manager.go:351-355` exists because "wisps are shared across
  all rigs (gh#2718)".
- **`internal/witness/handlers.go`** creates cleanup wisps via
  `createCleanupWisp` / `createSwarmWisp` and looks them up via
  `findCleanupWisp` / `findAnyCleanupWisp` /
  `findAllCleanupWisps`. Cleanup wisps are the primary
  externalised artifact of witness patrol work.
- **`internal/doctor/wisp_check.go`** and
  `internal/doctor/misclassified_wisp_check.go` are the
  health-check paths that verify wisps are correctly
  classified and not leaking into `issues`.

## Lifecycle / state

A wisp moves through the following states during its
lifetime:

1. **Created.** An agent or command inserts a row into the
   `wisps` table via the beads SDK. The wisp starts with
   `status = 'open'` (or `hooked` / `in_progress` depending
   on its category) and `created_at = now`.
2. **Alive.** The wisp exists and is visible to any code that
   queries `wisps`. Agents consulting the wisps table may
   read it, act on it, close it, comment on it, or link a
   permanent bead to it.
3. **Eligible for compaction.** The wisp's `created_at` crosses
   the `--max-age` threshold (reaper default 24h). The reaper
   scan counts it; the reaper reap may close it.
4. **Promotion check.** If the reaper is about to close the
   wisp, `ShouldPromote` is consulted. If any of the four
   criteria fires, the wisp is promoted to a permanent bead
   (moves to `issues`) instead of being compacted.
5. **Closed.** The reaper (or a manual close) sets the wisp's
   `status = 'closed'` and `closed_at = now`. The wisp still
   exists as a row but is no longer active.
6. **Purgeable.** After `--purge-age` (default 7d) past
   `closed_at`, the wisp is eligible for deletion. The
   reaper purge path DELETEs the row. This is irreversible.

A wisp can also be closed directly by an agent (e.g. a
Witness closing a cleanup wisp it created once the cleanup
is verified) without going through the TTL path.

## Relationships with other concepts

- **convoy** — a convoy's tracked issues are permanent
  beads, but convoy-related stranded-feed wisps and the
  molecule step wisps for `mol-convoy-feed` are wisps. See
  [`convoy` concept](convoy.md).
- **molecule** (pending Sub C) — molecule steps are wisps.
  The molecule itself is typically a permanent bead; its
  ephemeral step state is wisp-backed.
- **rate-limit notices** — purely wisps. They matter at the
  moment the rate limit fires and are compacted away.
- **merge requests** — wisps. The refinery's queue is a
  query against the `wisps` table.
- **cleanup receipts** — wisps. The Witness's
  `createCleanupWisp` creates them; the reaper eventually
  compacts them (unless someone commented, in which case
  `HasComments` triggers promotion).

## Related wiki pages

- [internal/beads](../packages/beads.md) — SDK wrapper,
  `ListMergeRequests`, `ListWisps`, `ParseMRFields`.
- [internal/wisp](../packages/wisp.md) — utility package
  (NOT where the concept is implemented).
- [internal/reaper](../packages/reaper.md) — the cleanup
  engine.
- [internal/refinery](../packages/refinery.md) — queries the
  wisps table for merge requests.
- [internal/witness](../packages/witness.md) — creates
  cleanup wisps.
- [internal/doltserver](../packages/doltserver.md) — owns the
  `wisps_migrate.go` schema migration.
- [convoy concept](convoy.md) — related concept that uses
  wisps for ephemeral tracking state.
- [reaper role](../roles/reaper.md) — the cleanup persona
  that enforces wisp TTLs.
- [refinery role](../roles/refinery.md) — the primary consumer
  of merge-request wisps.
- [witness role](../roles/witness.md) — the primary producer
  of cleanup wisps.
- [gt reaper](../commands/reaper.md) — the CLI for the
  cleanup cycle.
- [gt wl](../commands/wl.md) — wisp-config list (the
  `.beads-wisp/config/<rig>.json` CLI, distinct from the
  wisp table).

## Notes / open questions

- **`internal/wisp` is a name collision.** The Go package
  `internal/wisp` does NOT implement the wisp concept — it
  holds `.beads` / `.beads-wisp` directory utilities and the
  `ShouldPromote` predicate. The concept lives in
  `internal/beads` + `internal/doltserver` + the SDK. The
  [`internal/wisp` package page](../packages/wisp.md) warns
  about this collision explicitly.
- **Promotion moves the row across tables.** Once promoted,
  a wisp is no longer a wisp — it's an `issues` row. The
  promotion path is where the ephemeral/permanent boundary
  is crossed. The code path that does the physical move is
  not in this wiki yet (pending a read of the SDK).
- **TTL defaults are CLI-driven.** `--max-age=24h`,
  `--purge-age=168h`, `--mail-age=168h`, `--stale-age=720h`
  are the `gt reaper` defaults but are per-invocation
  parameters, not per-wisp-type. Future work may introduce
  per-type TTLs (different defaults for merge requests vs
  cleanup wisps vs rate-limit notices).
- **Merge requests are wisps but long-lived ones.** MRs may
  stay in the queue for hours while gates run. The reaper's
  default `--max-age=24h` is long enough that active MRs
  don't get reaped, but a stuck-queue scenario could see
  them pass the TTL. The MR-bead-specific reap semantics
  are worth a future drift check.
- **Agent beads are NOT wisps.** `issue_type='agent'` is
  explicitly excluded from `Reap`
  (`reaper.go:322-325`). Agent identity beads have
  persistent identity even when they share the `wisps`
  table with ephemeral content.
