---
title: identity (concept)
type: concept
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/cmd/mail_identity.go
  - /home/kimberly/repos/gastown/internal/config/directives.go
  - /home/kimberly/repos/gastown/internal/polecat/session_manager.go
  - /home/kimberly/repos/gastown/internal/polecat/manager.go
tags: [concept, identity, agent, detectsender, env-vars, agent-bead, session-name, whoami]
---

# identity

An **identity** in Gas Town is what answers the question
"who am I?" for an agent running a `gt` command. The answer
is a structured address of the form
`<prefix>/<role>` or `<prefix>/<name>` — for example
`mayor/`, `deacon/`, `gastown/witness`, `gastown/refinery`,
`gastown/polecats/furiosa`, `gastown/crew/alice`, or
`deacon/dogs/rover`. The address is load-bearing: mail
routing, bead authorship, role dispatch, prime-time
directive loading, and session naming all consult it.

Identity is **resolved**, not assigned. There is no single
"identity file" for an agent. Instead, identity is
computed from a chain of signals at the moment a gt
command runs, with explicit environment variables as the
primary channel and filesystem-based fallbacks for edge
cases.

## The identity chain

The canonical resolver is `detectSender` in
`/home/kimberly/repos/gastown/internal/cmd/mail_identity.go:88-98`.
Priority order:

1. **`GT_ROLE` env var** — if set, build the address from
   the role and related env vars
   (`detectSenderFromRole`, `:107-156`).
2. **No `GT_ROLE`** — fall back to cwd-based detection
   (`detectSenderFromCwd`, `:159-227`), which walks the
   directory tree looking for an agent path pattern or a
   `.gt-agent` identity file.
3. **No match** — return `"overseer"` (the human at the
   terminal).

Every Gas Town agent runs in a tmux session with
`GT_ROLE` set at spawn, so the env-var path is the normal
path. The cwd fallback is for debugging sessions where a
human runs `gt` from inside an agent's working directory
without the env set.

## Building an address from `GT_ROLE` + context

`detectSenderFromRole(role)` (`mail_identity.go:107-156`)
is a switch over the role name:

- **`GT_ROLE` contains `/`** — already a full address,
  return it verbatim.
- **`mayor`** — return `"mayor/"` (town-level, no rig).
- **`deacon`** — return `"deacon/"` (town-level, no rig).
- **`polecat`** — return `"<GT_RIG>/<GT_POLECAT>"` if both
  are set; otherwise fall back to cwd detection.
- **`crew`** — return `"<GT_RIG>/crew/<GT_CREW>"` if both
  are set; otherwise cwd.
- **`witness`** — return `"<GT_RIG>/witness"` if `GT_RIG`
  is set; otherwise cwd.
- **`refinery`** — return `"<GT_RIG>/refinery"` if
  `GT_RIG` is set; otherwise cwd.
- **`dog`** — return `"deacon/dogs/<GT_DOG_NAME>"` if the
  dog name is set; otherwise cwd.
- **Unknown role** — fall back to cwd detection.

The env vars involved:

- **`GT_ROLE`** — the role class (mayor, deacon, polecat,
  crew, witness, refinery, dog) or a full pre-built
  address.
- **`GT_RIG`** — the rig name for rig-scoped agents
  (every role except mayor and deacon).
- **`GT_POLECAT`** — the polecat's themed name (e.g.
  `furiosa`).
- **`GT_CREW`** — the crew worker's name.
- **`GT_DOG_NAME`** — the dog's themed name (e.g.
  `rover`).
- **`GT_TOWN_ROOT` / `GT_ROOT`** — the town root, used by
  `findMailWorkDir` and other path resolvers.

These are set by the session manager when it spawns the
agent's tmux session. See
`/home/kimberly/repos/gastown/internal/polecat/session_manager.go`
for the polecat side of the spawn
(`SessionStartOptions`, `Start(polecat, opts)`).

## Cwd-based fallback

`detectSenderFromCwd` (`mail_identity.go:159-227`) walks
the current working directory looking for recognizable
patterns:

1. **`.gt-agent` identity file.** Walks up from cwd looking
   for a `.gt-agent` JSON file
   (`detectSenderFromAgentFile`, `:235-255`). The file
   shape is `{role, rig, name}`
   (`agentIdentityFile`, `:229-233`). When present, the
   role/rig/name are assembled via `identityFromAgentFile`
   (`:257-290`). This is the preferred path because it's
   explicit metadata rather than inferred from path
   components.
2. **`/polecats/` in cwd** — extract rig + polecat name
   from `<rigPath>/polecats/<name>/...`.
3. **`/deacon/dogs/` in cwd** — extract dog name.
4. **`/crew/` in cwd** — extract rig + crew name.
5. **`/refinery` in cwd** — extract rig, return
   `<rig>/refinery`.
6. **`/witness` in cwd** — extract rig, return
   `<rig>/witness`.
7. **`/mayor` in cwd** — return `"mayor"`.
8. **No match** — return `"overseer"`.

The order is important: `.gt-agent` wins over path
parsing so nested agent directories (e.g., a witness
clone inside a rig) don't get mis-parsed.

## `BEADS_ACTOR` vs identity

The `BEADS_ACTOR` environment variable is a related but
distinct concept: it's what the beads SDK stamps on
newly-created beads as the author. In a Gas Town session,
`BEADS_ACTOR` and `detectSender()` typically agree because
the session manager sets both. But they can diverge:

- **`BEADS_ACTOR`** is a single env var that the beads SDK
  reads directly. Nothing stops a script from setting
  `BEADS_ACTOR=alice` before calling `bd create`.
- **`detectSender()`** is gastown's higher-level resolver.
  It uses `GT_ROLE` + related env vars, not
  `BEADS_ACTOR`, and it has the cwd fallback chain.

For the common case (polecat in its worktree, witness in
its watch loop, refinery processing the merge queue),
they agree. For edge cases (a polecat debugging from its
own worktree with `BEADS_ACTOR` overridden, a script
impersonating another agent for a one-off write), they
may not. The code comments in `mail_identity.go:85-87`
are explicit that cwd-based detection is "also tried to
support running commands from agent directories without
`GT_ROLE` set (e.g., debugging sessions)."

## Agent beads — the durable identity anchor

An identity has a matching **agent bead** in the beads
database. The agent bead is a row with
`issue_type = 'agent'` that carries the agent's
persistent state: its role, rig, current hook bead,
current state (idle / working / stuck / done), and a CV
chain of completed assignments.

Agent bead IDs follow a structured format. From the
rig package
(`/home/kimberly/repos/gastown/internal/rig/manager.go:1129-1139`):

- Witness: `<prefix>-<rig>-witness` (e.g.,
  `gt-gastown-witness`)
- Refinery: `<prefix>-<rig>-refinery`
- Mayor: town-level, created by `gt install`
- Deacon: town-level, created by `gt install`

Polecats have their own naming: the agent bead ID encodes
the rig prefix, role, and themed name. Crew workers
similar. All of them are created via
`beads.CreateAgentBead` with the matching `RoleType`,
`Rig`, `AgentState`, and `HookBead` fields.

Agent beads are **NOT wisps**. Even though they share the
`wisps` table in some schemas, `issue_type='agent'` is
explicitly excluded from the reaper's close sweep
(`internal/reaper/reaper.go:322-325`) so they have
persistent identity regardless of age.

## Session name

The tmux session running an agent has a name derived from
identity. For polecats, the session name is built by
`session.PolecatSessionName(prefix, polecat)` (called from
`SessionManager.SessionName(polecat)` in
`polecat/session_manager.go:111-122`). The shape is
typically `<rig-prefix>-<role>-<polecat-name>` or
`<rig-prefix>-<polecat-name>`.

The validation code at
`session_manager.go:127-145` checks for a known
double-prefix bug where a session name might end up with
two copies of the rig prefix (e.g.,
`gt-gastown_manager-gastown_manager-142`); it logs but
does not fail, so malformed sessions can still be cleaned
up.

Session names are how the Witness identifies which tmux
session belongs to which polecat, how `gt attach` finds a
specific agent, and how heartbeat files
(`<townRoot>/.runtime/heartbeats/<sessionName>.json`) are
keyed.

## The identity chain in practice

For a polecat named `furiosa` in the `gastown` rig:

1. **Session spawn** — the session manager sets
   `GT_ROLE=polecat`, `GT_RIG=gastown`,
   `GT_POLECAT=furiosa`, `BEADS_ACTOR=gastown/polecats/furiosa`,
   and starts tmux with a session name like
   `gastown-polecat-furiosa`.
2. **Session begins** — the polecat runs `gt prime`, which
   calls `detectSender()` → `detectSenderFromRole("polecat")`
   → `"gastown/furiosa"`.
3. **Prime loads directive** — `LoadRoleDirective("polecat",
   townRoot, "gastown")` concatenates
   `<town>/directives/polecat.md` and
   `<town>/gastown/directives/polecat.md` (if either
   exists) into the agent's prompt. See
   [directive concept](directive.md).
4. **Work begins** — the polecat calls `gt mol current` to
   see what it should work on. The current-molecule lookup
   is identity-scoped, keyed by the polecat's agent bead.
5. **Every bd write** carries the identity as `BEADS_ACTOR`
   so the audit trail points back to this polecat.
6. **Mail lookups** use `detectSender()` to determine
   which inbox to check (`<rig>/polecats/furiosa`).
7. **On completion**, `gt done` uses the same identity to
   find the polecat's agent bead, transition its state,
   and write a CV chain entry.

Every step of the polecat's operational day consults the
identity. It is the backbone of routing, not a
nice-to-have.

## Identity vs actor

Subtle but load-bearing:

- **Identity** (from `detectSender()`) is the
  address-shaped answer to "who is running this command in
  this agent's operational context?"
- **Actor** (from `BEADS_ACTOR`) is what gets stamped on
  bead writes as the author. Usually the same as the
  identity, but it's a separate env var with its own
  history in the env.

In a healthy session: `detectSender()` and `BEADS_ACTOR`
agree. The session manager sets both at spawn time. In a
debugging session or a script: they may diverge. The
bead's `created_by` field ends up whatever `BEADS_ACTOR`
said at write time; the mail routing and role dispatch
use `detectSender()`. An operator debugging a
routing issue should check both and notice if they
disagree.

## Relationships with other concepts

- **[rig](rig.md)** — the rig is the middle component of
  most identities (`<rig>/polecats/<name>`, `<rig>/witness`).
  Town-level agents (Mayor, Deacon) have no rig in their
  address.
- **[directive](directive.md)** — directives are loaded by
  role + rig, both of which come from identity.
- **[polecat role](../roles/polecat.md)** — polecat
  identity is the richest case: role + rig + themed name
  + agent bead + CV chain.
- **[mayor role](../roles/mayor.md) / [deacon
  role](../roles/deacon.md)** — town-level singletons; no
  rig component; no per-instance name.
- **[wisp](wisp.md)** — agent beads are NOT wisps even
  though they share the same table; identity beads are
  explicitly protected from compaction.
- **[molecule](molecule.md)** — `gt mol current` is
  identity-scoped: it asks "what is this specific agent
  currently working on?"

## How it's realized in the code

- **`internal/cmd/mail_identity.go`** —
  `detectSender`, `detectSenderFromRole`,
  `detectSenderFromCwd`, `detectSenderFromAgentFile`,
  `identityFromAgentFile`, `findMailWorkDir`,
  `findLocalBeadsDir`.
- **`internal/cmd/whoami.go`** — the `gt whoami` CLI that
  prints the detected identity.
- **`internal/cmd/commit.go` / `internal/cmd/handoff.go`**
  — commit author and handoff recipient resolution.
- **`internal/config/roles/`** — the canonical role
  TOML files consulted for role behavior.
- **`internal/polecat/session_manager.go`** — sets
  `GT_POLECAT` / `GT_RIG` / `GT_ROLE` when starting a
  polecat session.
- **`internal/beads`** — agent bead CRUD
  (`CreateAgentBead`, `ResetAgentBeadForReuse`,
  `WitnessBeadIDWithPrefix`, `RefineryBeadIDWithPrefix`,
  `RigBeadIDWithPrefix`).
- **`internal/session`** — `PolecatSessionName` and
  session lifecycle.

## Related wiki pages

- [`gt whoami`](../commands/whoami.md) — the CLI that
  prints the detected identity.
- [rig concept](rig.md)
- [directive concept](directive.md)
- [polecat role](../roles/polecat.md)
- [mayor role](../roles/mayor.md)
- [deacon role](../roles/deacon.md)
- [witness role](../roles/witness.md)
- [refinery role](../roles/refinery.md)
- [crew role](../roles/crew.md)
- [dog role](../roles/dog.md)
- [`internal/polecat`](../packages/polecat.md)
- [`internal/config`](../packages/config.md)
- [`internal/beads`](../packages/beads.md)

## Notes / open questions

- **`.gt-agent` files are the explicit form.** The comment
  at `mail_identity.go:165-167` calls cwd path-parsing
  "brittle" and nudges toward `.gt-agent` as the
  preferred mechanism. A lint pass could check that every
  agent directory has a `.gt-agent` file.
- **`detectSenderFallback` is mentioned in tests.** The
  test file `escalate_test.go:352-394` tests a function
  called `detectSenderFallback` — not present in the
  mapped sources. Probably a refactor target or a
  function name collision.
- **`BEADS_ACTOR` is not read by `detectSender()`.**
  Worth noting plainly: these are two channels that are
  usually coherent because the same spawn path sets
  both, but they are not coupled at resolution time. A
  future refactor could unify them, but the current
  shape is pragmatic.
- **The `overseer` fallback is the "human at the
  terminal" identity.** There is no richer identity
  available for a human operator running `gt` directly.
  Mail to `overseer` lands in the town beads with that
  address; `detectSender()` returns it by default when
  no role context is available.
- **Double-prefix sessions.** The validation in
  `session_manager.go:127-145` catches cases where the
  rig prefix is duplicated in the session name. It logs
  but doesn't refuse, so the bug can persist until
  something reaps the session. Worth a doctor check.
- **`GT_ROLE` may be a full address.** The resolver
  handles both `GT_ROLE=polecat` (simple) and
  `GT_ROLE=gastown/polecats/furiosa` (full address)
  shapes. When it contains a `/`, it's returned as-is.
  This is how certain agents pre-compute their own
  identity at spawn time and pass it through the env.
- **Role name casing is inconsistent.** The constants
  use `RolePolecat`, `RoleCrew`, etc., but the `"dog"`
  role is a raw string literal in `detectSenderFromRole`
  (`:146`). Worth a constants pass.
