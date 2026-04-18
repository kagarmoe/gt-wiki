---
title: reaper (role)
type: role
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/reaper/
  - /home/kimberly/repos/gastown/internal/cmd/reaper.go
tags: [role, agent, reaper, cleanup, dog-driven, dolt, wisps, ttl]
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

# reaper (role)

The Reaper is Gas Town's wisp and issue cleanup persona — the
LLM role (in practice, a Dog executing the `mol-dog-reaper`
formula) that sweeps Dolt databases for stale wisps, stale
issues, and expired mail, and either closes or purges them
based on eligibility. Unlike the Mayor, Witness, or Refinery,
the Reaper is NOT a long-running tmux agent. It is a role
performed on demand by a Dog, and the work it does is
deterministic enough that the Go code in
[`internal/reaper`](../packages/reaper.md) could run headless
if necessary.

**Also known as:** the cleanup sweeper, the grim reaper,
`mol-dog-reaper`.

## Purpose

Gas Town's data plane is Dolt, and Dolt's `wisps` table
accumulates ephemeral beads at a steady rate. Without a
disciplined cleanup story, the table grows without bound,
query latency climbs, and the town becomes harder to operate.

The Reaper exists so that:

- Stale open wisps (past their TTL) are closed mechanically
  rather than festering as open issues forever.
- Closed wisps past a purge age are deleted — the database
  reclaims their rows so scans stay fast.
- Old `gt:message`-labelled mail is purged on the same
  schedule.
- Long-stale OPEN issues (past `staleAge`, excluding P0/P1,
  epics, and issues with active dependencies) are auto-closed
  with a reason.
- Dangling parent references and other schema-level anomalies
  get surfaced to operators via the `Anomalies` field on scan
  results.

Without a Reaper, operators would have to periodically run
`gt dolt cleanup` by hand against every database, inspect
candidate counts manually, and decide what to drop. The Reaper
makes the sweep deterministic, observable, and safe enough to
run on a schedule.

## Behavior / responsibilities

The Reaper performs one full cleanup cycle per invocation.
From [`gt reaper`](../commands/reaper.md), the invocation
surface is:

- `gt reaper databases` — list databases available for reaping.
- `gt reaper scan` — count reap / purge / auto-close / mail
  candidates without modifying anything.
- `gt reaper reap` — close stale wisps past `--max-age` whose
  parent molecule is already closed or missing.
- `gt reaper purge` — delete closed wisps past `--purge-age`
  and closed `gt:message` mail past `--mail-age`. Irreversible.
- `gt reaper auto-close` — close open issues past `--stale-age`,
  excluding P0/P1, epics, and issues with active dependencies.
- `gt reaper run` — the full cycle: scan → reap → purge →
  auto-close → report.

Responsibilities per cycle:

- Discover every production database on the Dolt server via
  `DiscoverDatabases`, filtering out `information_schema`,
  `mysql`, and test-pollution prefixes (`testdb_`, `beads_t`,
  `beads_pt`, `doctest_`).
- For each database with the reaper schema (checked via
  `HasReaperSchema`), run the SQL pipeline: `Scan` → `Reap`
  → `Purge` → `AutoClose`.
- Respect `--dry-run` at every step. Every mutating function
  takes a `dryRun bool`.
- Accumulate `Anomaly` records from dangling parent references
  and report them per database.
- Aggregate totals across databases and emit a per-run report.
- Alert if total open wisps exceed `500` (hard-coded in the CLI
  at `internal/cmd/reaper.go:231-233`).

The Reaper does NOT:

- Decide what eligibility thresholds to use — the Dog / daemon
  supplies them as flags.
- Make individual close decisions outside the SQL filters —
  the filters ARE the decision.
- Run in a long-lived loop — each invocation is one cycle.
- Touch beads that live on a different Dolt instance — it
  tolerates `isTableNotFound` and silently skips.

## Lifecycle

The Reaper has no lifecycle of its own. It is invoked by a
Dog running `mol-dog-reaper` (the standard path) or by an
operator running `gt reaper run` directly (the manual escape
hatch). In both cases:

- **Invocation.** The Dog (or operator) runs `gt reaper run`
  — the full cycle — or one of the single-purpose subcommands
  (`gt reaper scan` for a read-only pass, `gt reaper reap` /
  `gt reaper purge` / `gt reaper auto-close` for targeted
  operations).
- **Execution.** The Go code in
  [`internal/reaper`](../packages/reaper.md) connects to Dolt
  on `127.0.0.1:3307` (overridable via `GT_DOLT_HOST` /
  `GT_DOLT_PORT`), discovers databases, opens each in turn,
  runs the SQL pipeline, collects results, and prints a
  report.
- **Completion.** The Dog process exits. There is no state to
  persist — the next cycle is entirely driven by fresh Dolt
  reads.
- **Scheduling.** `mol-dog-reaper` is invoked on a schedule
  via the daemon's patrol pathway — see
  [`gt patrol`](../commands/patrol.md) and the Deacon's
  `dog`-dispatch path
  ([`dog` package](../packages/dog.md)).

There is no "Reaper session" in tmux. There is no Reaper
agent identity in the agent registry (aside from the Dog
that executes the formula). The Reaper is an action, not an
always-on persona.

## Interactions with other roles

- **Dog** — the [`dog` role](dog.md) is the actual executor.
  The Dog receives `mol-dog-reaper` from its Deacon-driven
  queue, dispatches to the reaper CLI subcommands, collects
  results, and exits. The Reaper's "persona" is inherited
  from the Dog that runs it.
- **Deacon** — the [`deacon` role](deacon.md) is the scheduler
  and dispatch origin. It owns the dog kennel and decides
  when to run the next reaper cycle. Per the CLI note at
  `/home/kimberly/repos/gastown/internal/cmd/reaper.go:408-409`:
  "when Dog dispatch is unavailable. Normally the daemon
  dispatches a Dog to execute the mol-dog-reaper formula."
- **Witness** — the [`witness` role](witness.md) creates many
  of the wisps the Reaper eventually cleans (cleanup wisps,
  zombie-detection wisps, orphan-bead reset wisps). Witness
  and Reaper never talk directly — they communicate through
  the `wisps` table state.
- **Refinery** — the [`refinery` role](refinery.md) creates
  merge-request wisps. After merge (or rejection), those
  wisps become closed and eventually qualify for purge. The
  Refinery does not coordinate with the Reaper; the TTL is
  the coordination mechanism.
- **Polecats and Crew** — [`polecat` role](polecat.md) and
  [`crew` role](crew.md) do not interact with the Reaper
  directly. Their work creates beads (some of which become
  wisps); the Reaper cleans up the wisps long after.
- **Mayor** — the [`mayor` role](mayor.md) is the escalation
  target if the Reaper (via the Dog / Deacon chain) detects a
  critical anomaly. Alert threshold violations (e.g.
  `totalOpen > 500`) go to stderr and bubble upward through
  the Dog.

## Runtime substrate

The Reaper has no runtime substrate of its own. It is a Go
library ([`internal/reaper`](../packages/reaper.md)) wrapped
by CLI subcommands, invoked as a subprocess of whatever Dog
is executing `mol-dog-reaper`. The Dog runs in its own
short-lived process and exits when the formula is done.

The only "state" the Reaper reads is:

- The Dolt server at the configured host/port.
- The CLI flags (durations, dry-run, host, port, json).

The only "state" the Reaper writes is:

- Closes on stale wisps (UPDATE `wisps SET status='closed'`).
- Deletes on purged rows (DELETE from `wisps`, related
  auxiliary tables, and `issues` for old mail).
- Closes on stale open issues (UPDATE `issues SET
  status='closed'`).

No filesystem state. No PID files. No heartbeat. This makes
the Reaper trivially crash-safe — a killed cycle leaves the
database in a consistent state (all SQL is either transactional
per batch or idempotent).

The Go implementation is in
[`internal/reaper`](../packages/reaper.md).

## Related wiki pages

- [internal/reaper](../packages/reaper.md) — Go package
  implementation.
- [gt reaper](../commands/reaper.md) — CLI entry.
- [wisp concept](../concepts/wisp.md) — what the Reaper
  cleans up and WHY wisps exist as a distinct category.
- [dog role](dog.md) — the executor.
- [deacon role](deacon.md) — the scheduler.
- [mayor role](mayor.md) — the escalation target for
  anomalies.
- [witness role](witness.md) — upstream producer of cleanup
  wisps.
- [refinery role](refinery.md) — upstream producer of merge-
  request wisps.
- [`doltserver` package](../packages/doltserver.md) — the
  Dolt server the Reaper talks to.
- [`gt patrol`](../commands/patrol.md) — the patrol
  scheduling path.
- [`gt dolt`](../commands/dolt.md) — manual Dolt maintenance
  surface; the Reaper's complement for cleanup the Reaper
  doesn't cover (orphan databases, server restarts).

## Notes / open questions

- **The Reaper is not an agent in the tmux sense.** It has no
  session, no heartbeat, no persistent identity. Calling it
  a "role" is a stretch — it is more accurately a
  deterministic cleanup action performed by a Dog. The role
  page exists because the Dog running `mol-dog-reaper`
  effectively adopts a reaper persona for the duration of the
  cycle, and operators talk about "the reaper" as if it were
  a thing.
- **Eligibility decisions are in SQL, not in Go.** The
  `parentExcludeJoin` and `where` clauses ARE the policy. A
  future refactor could move policy into a pluggable
  predicate, but the SQL approach is fast and co-located with
  the data.
- **The Reaper never runs GC on agent beads.** The
  `issue_type != 'agent'` exclusion in `Reap` protects
  persistent agent identity beads explicitly.
- **Mail purging shares the cycle but not the schema.** The
  `issues` / `labels` table queries for `gt:message`-
  labelled mail tolerate `isTableNotFound` so the Reaper
  works even when beads lives on a separate Dolt instance.
  Operators on split-server setups will see the mail count
  silently returning zero.
- **Alert threshold `500` is hard-coded in the CLI.** The
  Reaper library itself never warns; the warning lives in
  `internal/cmd/reaper.go`. Moving the threshold to the
  library would make it configurable via the same flag set.
- **No schema version guard.** If the `wisps` table shape
  changes, the Reaper breaks silently. Coordinated upgrades
  are the operator's responsibility.
