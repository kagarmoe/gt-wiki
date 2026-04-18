---
title: templates/agents/ directory
type: file
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-15
sources:
  - /home/kimberly/repos/gastown/templates/agents/
  - /home/kimberly/repos/gastown/templates/agents/opencode.json.tmpl
  - /home/kimberly/repos/gastown/templates/agents/opencode-models.json
tags: [file, agent-facing, templates]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase8_audited: 2026-04-17
phase8_findings: [none]
---

# templates/agents/

Agent-runtime templates: per-agent JSON scaffolds that `gt` uses to
generate concrete agent-configuration files for a given provider and
model. Consumed by [../packages/templates.md](../packages/templates.md)
(which provides the embedded template loader) and surfaced at
agent-spawn time through [../commands/prime.md](../commands/prime.md)
and the boot/spawn paths.

## What it actually is

Two files, both OpenCode-specific as of this snapshot:

```
templates/agents/
├── opencode.json.tmpl     Go text/template for opencode-backed agent config
└── opencode-models.json   Model-preset reference (delay + descriptions)
```

Both are **reference / scaffolding data**, not runtime code. The gastown
runtime reads them, substitutes variables, and writes the result into
`~/gt/<town>/rigs/<rig>/<agent>/...` (or wherever
[../packages/runtime.md](../packages/runtime.md) decides to place per-agent
config).

## opencode.json.tmpl — agent-template-v1 manifest

754 bytes. Go `text/template` syntax (`{{.Field}}` with `default` pipelines).

```json
{
  "$schema": "https://gastown.dev/schemas/agent-template-v1.json",
  "_models_api": "https://models.dev/api.json",
  "_note": "LLMs: fetch available models from _models_api to get current provider/model options",
  "name": "opencode-{{.Model}}",
  "description": "OpenCode agent using {{.Provider}}/{{.Model}}",
  "command": "{{.OpenCodePath | default \"opencode\"}}",
  "args": ["-m", "{{.Provider}}/{{.Model}}"],
  "non_interactive": {
    "subcommand": "run",
    "output_flag": "--format json"
  },
  "runtime": {
    "provider": "opencode",
    "tmux": {
      "ready_prompt_prefix": "",
      "ready_delay_ms": {{.ReadyDelayMs | default 8000}},
      "process_names": ["opencode", "node"]
    }
  },
  "hooks": {
    "provider": "opencode"
  }
}
```

### Template variables

| variable | default | purpose |
|---|---|---|
| `.Provider` | — (required) | The model provider key (e.g. `openai`, `anthropic`, `google`, `xai`, `github-copilot`, `opencode`). |
| `.Model` | — (required) | The specific model name (e.g. `gpt-5.2`, `claude-opus-4-6`). |
| `.OpenCodePath` | `"opencode"` | Path to the opencode binary. Override when running from a specific install. |
| `.ReadyDelayMs` | `8000` | Milliseconds to wait before considering the tmux-hosted opencode "ready". See note on delay-based detection below. |

### What gets generated

A concrete `agents.json` entry (part of the gastown agent registry). Key
fields after substitution:

- **`name`** — e.g. `opencode-gpt-5.2-codex`
- **`command`** — the executable to run
- **`args`** — forwarded to opencode (`-m <provider>/<model>`)
- **`non_interactive.subcommand`** — `run` (opencode's headless mode)
- **`non_interactive.output_flag`** — `--format json`
- **`runtime.provider`** — hard-coded `"opencode"` (picks the
  opencode-specific runtime hook installer in
  [../packages/runtime.md](../packages/runtime.md))
- **`runtime.tmux.ready_prompt_prefix`** — **empty string** (deliberately;
  see below)
- **`runtime.tmux.ready_delay_ms`** — `{{.ReadyDelayMs | default 8000}}`
- **`runtime.tmux.process_names`** — `["opencode", "node"]` (for
  process-group / keepalive detection)
- **`hooks.provider`** — `"opencode"` (same effect on
  [../packages/runtime.md](../packages/runtime.md)'s hook installer
  dispatch)

### Why delay-based ready detection

The OpenCode TUI uses Unicode box-drawing characters that break the
gastown tmux "ready prompt" matcher (which greps for a prefix string in
the terminal buffer). Because of this, the template sets
`ready_prompt_prefix` to empty and instead relies on `ready_delay_ms` —
the gastown spawn logic sleeps that many milliseconds and then assumes
the TUI is ready. This is explicitly documented in the companion
`opencode-models.json`:

> why_delay_based: "OpenCode TUI uses box-drawing characters that break
> prompt prefix matching. Delay-based detection is required."

Per-model delay values come from `opencode-models.json` (below).

## opencode-models.json — model presets

1924 bytes. Not a template — a static JSON reference that lists
recommended `ready_delay_ms` values per provider/model, plus usage notes.

### Shape

```json
{
  "version": 1,
  "description": "OpenCode model presets with recommended delay settings for gastown integration",
  "usage": "Use ready_delay_ms values when configuring runtime.tmux.ready_delay_ms in agents.json",
  "models_api": "https://models.dev/api.json",
  "models_api_note": "LLMs should fetch current model list from models_api. The 'models' below are fallback examples with gastown-specific delay recommendations.",
  "models": { ... },
  "notes": { ... }
}
```

### Models table (as of this snapshot)

| provider | model | ready_delay_ms | description |
|---|---|---:|---|
| `openai` | `gpt-5.2` | 5000 | GPT-5.2 chat model |
| `openai` | `gpt-5.2-codex` | 8000 | GPT-5.2 Codex for code tasks |
| `openai` | `codex-1` | 10000 | Codex-1 extended context |
| `google` | `gemini-3-pro-high` | 6000 | Gemini 3 Pro High quality |
| `xai` | `grok-code-fast-1` | 4000 | Grok Code Fast |
| `github-copilot` | `gpt-5.2-codex` | 8000 | GitHub Copilot GPT-5.2 Codex |
| `opencode` | `glm-4.7-free` | 15000 | GLM 4.7 Free tier (may timeout, needs longer delay) |
| `opencode` | `minimax-m2.1-free` | 10000 | MiniMax M2.1 Free tier |
| `opencode` | `big-pickle` | 12000 | Big Pickle experimental |

### Notes block

- **why_delay_based**: see quote above.
- **free_tier_warning**: "Free tier models may have longer cold start
  times. Increase delay if timeout errors occur."
- **default_delay**: `8000` (matches the template default).

### Important: the list is a fallback

The `_note` field at the top explicitly tells LLMs to fetch live data
from `https://models.dev/api.json` rather than trust this file. The
embedded list is a gastown-specific **recommendation layer** (delay
values tuned for tmux + opencode startup time) layered over whatever
models are currently available upstream.

## Prime-time integration

These templates are not loaded at `gt prime` time directly. They're
loaded at **agent-spawn time** via the template system in
[../packages/templates.md](../packages/templates.md):

1. Operator runs something like `gt rig add --polecat-agent <name>` or
   the equivalent rig-config write.
2. The agent name (e.g. `opencode-gpt-5.2`) maps through
   `opencode-models.json` to pick `ready_delay_ms`.
3. `opencode.json.tmpl` is instantiated with `{Provider, Model,
   OpenCodePath, ReadyDelayMs}` and written as an `agents.json` entry.
4. When that agent spawns, [../packages/runtime.md](../packages/runtime.md)
   picks the `runtime.provider: opencode` path, which sets up the
   OpenCode-specific hook installer (`hooks.provider: opencode`).
5. The spawned OpenCode session loads [.opencode/plugins/gastown.js](opencode-dir.md)
   from the repo, which runs `gt prime` at session.created and injects
   role context into the system prompt.

The templates here are thus **upstream of** `gt prime` — they define
*which binary* gets spawned and *how* it's detected as ready, not what
context it receives. The context injection belongs to
`.opencode/plugins/gastown.js`.

## Role → template mapping

This set of templates is **provider-scoped, not role-scoped**. There is
one `opencode.json.tmpl` that works for any role (polecat / witness /
refinery / crew / deacon) as long as `GT_ROLE` is set in the environment
before the opencode process starts. Per-role behaviour is injected later
by:

- Environment (`GT_ROLE`, `GT_RIG`)
- `gt prime` output (role-specific CLAUDE.md templates from
  [../packages/templates.md](../packages/templates.md) — **different set
  of templates**: `embedded` role templates vs this repo-level
  agent-runtime template)
- Per-repo agent surface
  ([.opencode/plugins/gastown.js](opencode-dir.md) or
  [.claude/](claude-dir.md) slash commands)

There is no corresponding `claude.json.tmpl` here because Claude Code's
config model is different — Claude Code is configured via hook settings
installed by [../packages/hooks.md](../packages/hooks.md) at a different
layer, not via a top-level agent manifest.

## Notes / open questions

- The schema URL `https://gastown.dev/schemas/agent-template-v1.json` is
  referenced but not yet verified to exist or match the template shape.
- The model list in `opencode-models.json` is explicitly marked as
  fallback — worth grepping code to find where `models.dev/api.json` is
  actually fetched (if anywhere).
- No per-role *override* mechanism is visible in this directory. If
  e.g. polecat needs `ready_delay_ms: 20000` but crew needs 5000, the
  override must live elsewhere (rig settings? agent registry?). TODO
  to trace.
- Only OpenCode is represented. Claude Code, Codex, Gemini, Copilot
  agents are assembled elsewhere (cross-ref
  [../packages/runtime.md](../packages/runtime.md) which names all four).
- `version: 1` on `opencode-models.json` implies a schema version,
  but there's no v2 to compare against yet.

## Related pages

- [claude-dir.md](claude-dir.md) — the **per-repo** Claude Code surface
  (slash commands + skills)
- [opencode-dir.md](opencode-dir.md) — the **per-repo** OpenCode surface
  (slash commands + plugins); consumes the config that this directory
  generates
- [../packages/templates.md](../packages/templates.md) — embedded template
  loader + role CLAUDE.md templates (different set)
- [../packages/runtime.md](../packages/runtime.md) — agent-runtime
  provider dispatch (consumes `runtime.provider` and `hooks.provider`
  from the generated `agents.json`)
- [../packages/hooks.md](../packages/hooks.md) — hook settings installer
- [../commands/prime.md](../commands/prime.md) — downstream consumer
- [../commands/rig.md](../commands/rig.md) — `gt rig add` sets
  `role_agents.*`
- [../binaries/gt.md](../binaries/gt.md) — main CLI
- [../roles/polecat.md](../roles/polecat.md), [../roles/witness.md](../roles/witness.md), [../roles/deacon.md](../roles/deacon.md), [../roles/refinery.md](../roles/refinery.md), [../roles/crew.md](../roles/crew.md), [../roles/mayor.md](../roles/mayor.md), [../roles/dog.md](../roles/dog.md) — any of these roles can be backed by an opencode agent generated from this template
- [../inventory/auxiliary.md](../inventory/auxiliary.md) — A-level inventory
