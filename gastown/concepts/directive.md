---
title: directive (concept)
type: concept
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/config/directives.go
  - /home/kimberly/repos/gastown/internal/cmd/directive.go
tags: [concept, directive, role-override, prime-time, town-level, rig-level, overlay, markdown]
phase3_audited: 2026-04-14
phase3_findings: [wiki-stale]
phase3_severities: [wrong]
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
phase8_audited: 2026-04-17
phase8_findings: [cross-ref-only]
detail_depth: {synthesis: 2, relationships: 2}
---

# directive

A **directive** is an operator-provided role override: a
markdown file that adjusts how a specific role
(Mayor, Deacon, Witness, Refinery, Polecat, Crew, Dog)
behaves in a particular town or rig without touching the
role's baked-in definition. Directives are loaded and
injected into the agent's prompt at **prime time** —
when `gt prime` runs during session start or compaction —
so the agent carries the directive's instructions for the
duration of that session.

Directives are **declarative**: they are text files, not
code. An operator drops `<townRoot>/directives/polecat.md`
into place and every polecat in every rig under that town
gets the extra instructions next time they prime. Drop
`<townRoot>/<rig>/directives/refinery.md` and only that
rig's Refinery sees the override.

The mechanism is intentionally simple (string
concatenation of town + rig markdown files) and intentionally
invisible at the CLI surface (no subcommand is wired up —
see Notes). The load-bearing code is a single 41-line
file: `internal/config/directives.go`.

## Why directives exist

Gas Town agents have canonical role definitions baked into
`internal/config/roles/<role>.toml` — what the Mayor does,
what a polecat does, what a Refinery does, etc. These
definitions are authoritative but inflexible: changing a
role for one rig requires a code change.

Directives are the escape hatch. An operator running a
specific project might need the Refinery to enforce an
extra pre-merge gate, or a Witness to tolerate longer
heartbeat gaps, or polecats to follow a specific
commit-message convention for this rig only. Instead of
editing the role's TOML in the gastown repo, the operator
drops a markdown file in the appropriate directives
directory. The role's prompt at prime time picks it up and
layers it on top of the canonical definition.

## File layout

Defined by the `LoadRoleDirective` resolver in
`/home/kimberly/repos/gastown/internal/config/directives.go:19-41`:

- **Town-level:** `<townRoot>/directives/<role>.md`
  — applies to every rig's copy of that role.
- **Rig-level:** `<townRoot>/<rigName>/directives/<role>.md`
  — applies only to agents running in that specific rig.

Only the role name in the filename matters. Content is
free-form markdown — whatever text the operator wants the
agent to read at prime time.

## Resolution: concatenate, not override

Unlike [formula overlays](formula.md) (where rig-level
fully replaces town-level) and unlike [rig config](rig.md)
layers (where the first non-nil layer wins), directives
use **concatenation**: both files are loaded, both contents
are included, town first and rig last. From
`directives.go:19-41`:

```go
// If both exist, they are concatenated (town first, then rig)
// separated by a newline, giving rig-level content the last word.
```

Rig-level content gets "the last word" in the sense that it
appears later in the prompt, after the town-level content.
An LLM reading a prompt top-to-bottom typically weights
later content more heavily, so appending the rig directive
after the town one creates an implicit precedence. But both
bodies are in the context — nothing is removed. An
operator writing a rig directive cannot delete a town
directive's instruction; they can only add to it or
contradict it and hope the LLM listens.

The resolver returns an empty string if neither file
exists or if both are empty after trimming whitespace.
Invalid or unreadable paths are treated as absent: no
error, no content. This is a "best-effort, never block"
mechanism.

## When directives are loaded

The Long text on `gt directive`
(`internal/cmd/directive.go:23-28`) advertises that
directives are "injected at prime time." The consumer is
`gt prime` — the command every agent runs at session
start (and after compaction, clear, or a new session) to
load its identity and context. The prime path calls
`LoadRoleDirective(role, townRoot, rigName)` for the
current role and appends the returned text to the agent's
prompt.

This means:

- **Session-scoped.** A directive takes effect the next
  time the agent primes. Existing sessions don't pick it
  up mid-flight.
- **No enforcement.** The directive is instructions to
  the LLM, not a runtime constraint. The agent is free
  to interpret or ignore them, just as it is with any
  other prompt content.
- **Identity-aware.** The role and rig in the resolver
  call come from the agent's identity (see
  [identity concept](identity.md)) — the same `GT_ROLE` +
  `GT_RIG` pair that `detectSender` and session naming
  use.

## CLI surface

The `gt directive` command
(`/home/kimberly/repos/gastown/internal/cmd/directive.go`)
wires up `directiveCmd` as a cobra parent with
`RunE: requireSubcommand`. The three subcommands —
`show <role>`, `edit <role>`, `list` — are wired in
sibling files (`directive_show.go`, `directive_edit.go`,
`directive_list.go`), each registering itself on
`directiveCmd` in its own `init()`. Both the config-loader
side (`config/directives.go`) and the CLI management side
are implemented.

See the [`gt directive` command page](../commands/directive.md)
for the full CLI analysis. (Phase 3 Batch 4 wiki-stale
correction: the Phase 2 concept page incorrectly described
the CLI as a "parent-only stub" with no wired subcommands.
The command page was corrected in Phase 3 Batch 1b; this
concept page is now aligned.)

## How it's realized in the code

- **`internal/config/directives.go`** — 41 lines. The
  entire directive loader. `LoadRoleDirective(role,
  townRoot, rigName)` is the one exported function.
  Reads `<townRoot>/directives/<role>.md` and
  `<townRoot>/<rigName>/directives/<role>.md`, trims,
  concatenates, returns.
- **`internal/cmd/directive.go`** — 41 lines. The cobra
  parent command stub. Advertises subcommands in Long
  text but only registers `requireSubcommand`. No
  implementation for `show`/`edit`/`list` in this file.
- **Prime path** — the consumer of `LoadRoleDirective`.
  `gt prime` collects the caller's role + rig from the
  environment and asks for the matching directive text
  to inject into the session prompt.
- **`internal/config/roles/`** — the canonical role TOML
  files (`mayor.toml`, `deacon.toml`, `polecat.toml`,
  `refinery.toml`, `witness.toml`, `crew.toml`, `dog.toml`).
  Directives layer on top of these at runtime; they do
  NOT modify the TOML files themselves.

## Relationships with other concepts

- **[identity](identity.md)** — the role and rig used
  for directive lookup come from the agent's identity.
  No identity, no directive.
- **[rig](rig.md)** — rig-level directives live under
  `<townRoot>/<rigName>/directives/`. A rig scaffolds
  this directory during creation (or lazily on first
  write).
- **[role](../roles/polecat.md)** — every role has a
  canonical definition; directives are role-scoped
  overlays. A directive for `polecat.md` applies to
  every polecat; one for `refinery.md` applies to the
  Refinery only.
- **[formula](formula.md)** — formulas define workflows,
  not role behavior. A directive can tell a role "don't
  start molecules on weekends"; a formula defines what
  a molecule actually does. They compose orthogonally.
- **[molecule](molecule.md)** — molecules run *under* a
  role. A role whose directive has changed runs the
  same molecules with different implicit guidance.

## Related wiki pages

- [`gt directive`](../commands/directive.md) — the
  parent-only CLI stub.
- [`gt prime`](../commands/prime.md) — the consumer.
- [`gt config`](../commands/config.md) — the sibling
  Configuration command; operates on town/rig settings
  JSON, not markdown overlays.
- [identity concept](identity.md)
- [rig concept](rig.md)
- [role pages](../roles/) — polecat, mayor, deacon,
  witness, refinery, crew, dog, reaper.
- [`internal/config` package](../packages/config.md) —
  where `directives.go` lives.

## Notes / open questions

- ~~**Missing CLI wiring.**~~ Phase 3 Batch 4 wiki-stale
  correction: the `show`, `edit`, `list` subcommands ARE
  wired via sibling files (`directive_show.go`,
  `directive_edit.go`, `directive_list.go`), each adding
  itself to `directiveCmd` in its own `init()`. This was
  confirmed in Phase 3 Batch 1b on the command page; the
  concept page's original claim was wrong at Phase 2 time.
  **Phase 2 root cause:** phase-2-incomplete — Phase 2
  read only `directive.go` in isolation without checking
  sibling files for `AddCommand` calls.
- **No runtime enforcement.** Directives are prompt
  instructions; the LLM may or may not follow them. A
  stricter system would enforce directive-declared
  constraints via runtime checks (e.g., a
  `max_concurrent: 1` directive could be enforced by the
  dispatcher), but the current shape is pure prompt
  layering.
- **No versioning.** A directive file has no metadata —
  no author, no timestamp, no version. Agents pick up
  whatever is on disk at prime time. If an operator
  updates a directive mid-session, existing sessions
  don't notice; new sessions pick it up.
- **Concatenate vs override asymmetry.** The choice to
  concatenate instead of override means you can't write a
  rig directive that "cancels" a town directive. The
  town's instructions are always in the prompt. This is
  deliberate: an operator editing a rig directive doesn't
  necessarily see the town directive and shouldn't
  silently invalidate it. But it means rig directives
  can only add or contradict.
- **`directive edit` lives in the Long text's promise.**
  The advertised `edit <role>` subcommand would open
  `$EDITOR` on the resolved directive file, creating it
  if needed. Implementing this is straightforward — it
  just hasn't happened in the mapped source.
- **`LoadRoleDirective` does not log missing files.** A
  typo in the filename (`witnes.md` instead of
  `witness.md`) silently produces an empty directive.
  No warning is printed. Worth a lint pass.
- **Rig-level directories don't exist by default.**
  `internal/rig/manager.go:AddRig` doesn't create
  `<rig>/directives/` during rig setup. The directory is
  created lazily by whoever first writes a file there.
  A lint check could catch typos in directory names the
  same way.
