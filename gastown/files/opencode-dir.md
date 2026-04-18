---
title: .opencode/ directory
type: file
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-15
sources:
  - /home/kimberly/repos/gastown/.opencode/
  - /home/kimberly/repos/gastown/.opencode/commands/handoff.md
  - /home/kimberly/repos/gastown/.opencode/plugins/gastown.js
tags: [file, agent-facing, opencode]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase8_audited: 2026-04-17
phase8_findings: [none]
detail_depth: {params: 2, data_flow: 2, errors: 1, side_effects: 1}
---

# .opencode/

Repo-local [opencode.ai](https://opencode.ai) agent configuration. Contains
two load-bearing sub-trees that extend the OpenCode TUI agent when it's
invoked inside the `gastown` repository: slash commands (`commands/`) and
plugins (`plugins/`).

## What it actually is

OpenCode auto-discovers two conventions per project:

- `.opencode/commands/*.md` — slash-command markdown files (same convention
  as `.claude/commands/` but simpler — no `allowed-tools` gate;
  `$ARGUMENTS` placeholder interpolation).
- `.opencode/plugins/*.js` — JavaScript plugins (ES module export) that
  hook into OpenCode's event bus and experimental chat hooks.

This gastown tree contributes one command (`/handoff`) and one plugin
(`gastown.js`) that integrates Gas Town's `gt prime` / `gt mail` /
`gt costs` loop into OpenCode's session lifecycle.

## Layout

```
.opencode/
├── commands/
│   └── handoff.md      /handoff [message] — cycle session via gt handoff
└── plugins/
    └── gastown.js      Session hooks — gt prime injection + cost accounting
```

Only two files total — the OpenCode surface is much smaller than the
Claude Code surface at [claude-dir.md](claude-dir.md).

## Slash command: /handoff

`.opencode/commands/handoff.md` (581 B)

```markdown
---
description: Hand off to fresh session, work continues from hook
---

Hand off to a fresh session.

User's handoff message (if any): $ARGUMENTS

Execute these steps in order:

1. If user provided a message, run the handoff command with a subject and message.
   Example: `gt handoff -s "HANDOFF: Session cycling" -m "USER_MESSAGE_HERE"`

2. If no message was provided, run the handoff command:
   `gt handoff`

Note: The new session will auto-prime via the SessionStart hook and find your handoff mail.
End watch. A new session takes over, picking up any molecule on the hook.
```

- Minimal frontmatter — only `description` (no `allowed-tools`, unlike
  the Claude Code surface).
- `$ARGUMENTS` is OpenCode's substitution for the user's argv.
- Depends on the OpenCode SessionStart hook (installed by the
  [../packages/runtime.md](../packages/runtime.md) OpenCode provider)
  firing `gt prime` in the next session — which is exactly what
  `gastown.js` provides. The two halves are co-dependent.

See also: [../commands/handoff.md](../commands/handoff.md) for the
underlying `gt handoff` subcommand and [../commands/prime.md](../commands/prime.md)
for the prime flow.

## Plugin: gastown.js

`.opencode/plugins/gastown.js` (2720 B). JavaScript ES module exporting
`GasTown` — the plugin registration function. This is the
**load-bearing integration point** between OpenCode sessions and the
Gas Town multi-agent runtime.

### Shape

```js
export const GasTown = async ({ $, directory }) => {
  const role = (process.env.GT_ROLE || "").toLowerCase();
  const autonomousRoles = new Set(["polecat", "witness", "refinery", "deacon"]);
  let didInit = false;
  let primePromise = null;

  // ... helpers ...

  return {
    event: async ({ event }) => { ... },
    "experimental.chat.system.transform": async (input, output) => { ... },
    "experimental.session.compacting": async ({ sessionID }, output) => { ... },
  };
};
```

### What it actually does

1. **Reads `GT_ROLE`** from the environment to decide if the agent is an
   autonomous role (polecat/witness/refinery/deacon) vs an interactive
   one (crew/mayor/unknown).

2. **`session.created` event** — If not yet initialized, kicks off a
   `loadPrime()` promise **early**, before the first chat turn, so the
   system-transform hook can `await` it later.

3. **`session.compacted` event** — Re-runs `loadPrime()` to replace the
   stale prime context with fresh output after OpenCode compacts the
   conversation.

4. **`session.deleted` event** — Records session cost via
   `gt costs record --session <sessionID>` (catches errors silently).

5. **`experimental.chat.system.transform` hook** — The system-prompt
   injection point. Awaits the `primePromise`, pushes the captured
   context onto `output.system`. If the loaded context is empty, resets
   `primePromise` to `null` so the next transform retries instead of
   pushing empty forever.

6. **`experimental.session.compacting` hook** — Before compaction, pushes
   a note into the compaction context reminding the (post-compact) agent
   to run `gt prime` and `gt hook`, with the current role baked in.

### loadPrime() — what gets captured

```js
const loadPrime = async () => {
  let context = await captureRun("gt prime");
  if (autonomousRoles.has(role)) {
    const mail = await captureRun("gt mail check --inject");
    if (mail) {
      context += "\n" + mail;
    }
  }
  return context;
};
```

- **Always runs `gt prime`** and captures stdout (via
  `$`/bin/sh -lc ${cmd}`.cwd(directory).text()` — Bun-style `$` template).
- **Autonomous roles additionally run `gt mail check --inject`** and
  concatenate any pending injectable mail into the prime context.
- **Errors are caught, logged, and return empty string** — the plugin
  never throws to OpenCode.
- **Historical note** (from the source comment): a `session-started`
  nudge to the Deacon was previously here but was removed because it
  interrupted the Deacon's await-signal backoff. The Deacon now wakes
  on beads activity instead.

### Why it's load-bearing

Without this plugin, an OpenCode agent starting inside a Gas Town rig
would:

- Not know its `GT_ROLE`, rig, agent bead identity, or formula step
  (all of which `gt prime` injects into the system prompt).
- Not see injected mail that should activate it (autonomous roles only).
- Not record cost on session exit → budget tracking gone.
- Not get re-primed after compaction → context drift.

In other words, **`gastown.js` is the OpenCode equivalent of the
SessionStart hook that the Claude Code runtime uses**. It's the runtime
glue that makes OpenCode-backed roles viable alongside Claude Code roles.

## How it's installed

The plugin is installed per-project by simply existing at
`.opencode/plugins/gastown.js`. OpenCode loads all `.js` files in
`.opencode/plugins/` at session-start and registers their default
exports.

This is distinct from the agent-runtime templates consumed by
[../packages/runtime.md](../packages/runtime.md) — those generate
`opencode.json` files *per agent worktree* with model settings
(see [templates-agents.md](templates-agents.md)). The plugin here lives
at the repo level and applies to all OpenCode sessions that run inside
the gastown checkout.

## Relationship to .claude/

| Surface | Claude Code | OpenCode |
|---|---|---|
| slash command registration | `.claude/commands/*.md` with `allowed-tools` gate | `.opencode/commands/*.md` simpler frontmatter |
| skill system | `.claude/skills/<name>/SKILL.md` | no direct equivalent |
| session hook integration | SessionStart hook (managed by [../packages/runtime.md](../packages/runtime.md) + [../packages/hooks.md](../packages/hooks.md)) | `.opencode/plugins/gastown.js` — ES module plugin |
| commands contributed by gastown | 3 (backup, patrol, reaper) | 1 (handoff) |
| plugins contributed by gastown | 0 | 1 (gastown.js) |

The asymmetry (Claude Code gets patrol/backup/reaper slash commands,
OpenCode does not) suggests the OpenCode surface is younger or less
invested in. OpenCode roles that need to run patrols today presumably
invoke `gt patrol` directly.

## Notes / open questions

- The plugin assumes `$` is the Bun/OpenCode shell template tag. Is
  this documented in OpenCode's public plugin API or is it internal?
- `experimental.chat.system.transform` and `experimental.session.compacting`
  are `experimental.*` hook names — they may change without warning.
- Cost recording uses `gt costs record --session <id>` — grep
  `gastown/internal/cmd/` for the definition. Related page:
  [../commands/costs.md](../commands/costs.md).

## Related pages

- [claude-dir.md](claude-dir.md) — sibling `.claude/` tree (Claude Code
  surface)
- [templates-agents.md](templates-agents.md) — opencode agent-runtime
  templates (different layer)
- [../binaries/gt.md](../binaries/gt.md) — the main CLI, invoked by every
  step of this plugin
- [../commands/prime.md](../commands/prime.md) — the `gt prime` output
  that gets captured and injected
- [../commands/handoff.md](../commands/handoff.md) — the `gt handoff`
  subcommand called by `/handoff`
- [../commands/mail.md](../commands/mail.md) — `gt mail check --inject`
- [../commands/costs.md](../commands/costs.md) — `gt costs record`
- [../packages/runtime.md](../packages/runtime.md) — OpenCode provider
- [../roles/polecat.md](../roles/polecat.md), [../roles/witness.md](../roles/witness.md), [../roles/deacon.md](../roles/deacon.md), [../roles/refinery.md](../roles/refinery.md) — the autonomous roles that get `gt mail check --inject` in addition to prime
- [../roles/crew.md](../roles/crew.md)
- [../inventory/auxiliary.md](../inventory/auxiliary.md) — A-level inventory
