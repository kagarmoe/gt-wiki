---
title: .claude/ directory
type: file
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-15
sources:
  - /home/kimberly/repos/gastown/.claude/
  - /home/kimberly/repos/gastown/.claude/commands/
  - /home/kimberly/repos/gastown/.claude/skills/
tags: [file, agent-facing, claude-code]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase8_audited: 2026-04-17
phase8_findings: [none]
detail_depth: {params: 2, data_flow: 2, errors: 1, side_effects: 1}
---

# .claude/

Repo-local Claude Code agent configuration. Contains two load-bearing
sub-trees that extend the Claude Code agent's runtime surface when it
runs inside the `gastown` repository: **slash commands** (`commands/`)
and **skills** (`skills/`).

## What it actually is

Claude Code (the Anthropic official CLI) auto-discovers two conventions
in any project directory it's invoked from:

- `.claude/commands/*.md` — each markdown file registers a slash command
  named after the filename (e.g. `backup.md` → `/backup`). The frontmatter
  `description`, `allowed-tools`, and `argument-hint` gate what the agent
  can run.
- `.claude/skills/<name>/SKILL.md` (or `skill.md`) — each subdirectory is
  a named skill with its own frontmatter (`name`, `description`,
  `allowed-tools`, `version`, `author`). The agent loads skill
  descriptions at startup and invokes them when the user's request
  matches.

This gastown tree contributes both: 3 slash commands wrapping the
autonomous-patrol operations, plus 4 skills supporting crew workflows and
GitHub triage.

## Layout

```
.claude/
├── commands/
│   ├── backup.md        /backup — Dolt backup cycle
│   ├── patrol.md        /patrol — run one patrol cycle
│   └── reaper.md        /reaper — wisp + stale-issue reaper
└── skills/
    ├── crew-commit/
    │   └── SKILL.md     canonical crew commit workflow
    ├── ghi-list/
    │   └── SKILL.md     formatted `gh issue list` table
    ├── pr-list/
    │   └── SKILL.md     formatted `gh pr list` table
    └── pr-sheriff/
        └── skill.md     PR triage workflow (note: lowercase)
```

Note the inconsistent filename casing: three skills use `SKILL.md`, one
(`pr-sheriff`) uses `skill.md`. Claude Code's loader is case-insensitive
on most filesystems but this is a latent portability bug on case-sensitive
deployments.

## Slash commands

### /backup — `.claude/commands/backup.md` (3250 B)

Runs the same cycle as `mol-dog-backup` / the `dolt-backup` plugin
([../plugins/README.md](../plugins/README.md)), but directly without Dog
dispatch.

- **Argument hint:** `[--skip-offsite]`
- **Allowed tools:** `gt dolt status`, `dolt backup`, `dolt log`, `rsync`,
  `du`, `cat`, `mkdir`, `timeout`, `gt escalate`, `ls`
- **Databases:** auto-discovered — dirs under `~/gt/.dolt-data/` that have
  a `<name>-backup` remote. Expected: `hq`, `beads`, `gastown`.
- **Backup dir:** `~/gt/.dolt-backup/`
- **Offsite:** `~/Library/Mobile Documents/com~apple~CloudDocs/gt-dolt-backup`
  (iCloud Drive on macOS; skipped with `--skip-offsite`).
- **Hang detection:** 30s timeout on `dolt sql ... SELECT 1` ping; on hang
  runs `gt escalate -s CRITICAL` and stops.
- **Change detection:** compares current `dolt log -n 1 --oneline` HEAD
  against `~/gt/.dolt-backup/<name>/.last-backup-hash`.
- **Sync command:** `timeout 120 dolt backup sync <name>-backup` per DB.

### /patrol — `.claude/commands/patrol.md` (4804 B)

Runs one patrol cycle for the current agent role.

- **Argument hint:** `[witness|deacon|refinery]`
- **Allowed tools:** `gt patrol`, `gt hook`, `gt mail`, `gt nudge`,
  `gt peek`, `gt escalate`, `gt dolt status`, `bd`, `gt mol`
- **Role detection:** reads `$GT_ROLE`, maps `*/witness` → witness,
  `*/deacon` (or `*/deacon/*`) → deacon, `*/refinery` → refinery.
  Explicit argument wins.
- **Entry point:** `gt patrol new --role <role>` (creates a hooked wisp
  with formula steps; resumes existing hooked patrol if one is already
  running).
- **Witness patrol steps** (9): inbox-check → process-cleanups →
  check-refinery → survey-workers → check-timer-gates → check-swarm →
  patrol-cleanup → context-check → loop-or-exit.
- **Deacon patrol steps** (14): inbox-check → trigger-pending-spawns →
  gate-evaluation → dispatch-gated-molecules → check-convoy-completion →
  health-scan → zombie-scan → plugin-run → dog-pool-maintenance →
  orphan-check → session-gc → patrol-cleanup → context-check → loop-or-exit.
- **Refinery patrol steps** (10): inbox-check → queue-scan →
  process-branch → run-tests → handle-failures → merge-push → notify →
  cleanup → context-check → loop-or-exit.
- **Cycle completion:** each cycle ends with `gt patrol report --summary ...
  --steps ...` which closes the current patrol wisp and spawns the next.

### /reaper — `.claude/commands/reaper.md` (3281 B)

Runs the wisp reaper directly. Scan → reap → purge → auto-close stale
issues across all production Dolt databases.

- **Argument hint:** `[--dry-run]`
- **Allowed tools:** `gt reaper`, `gt escalate`, `gt dolt status`
- **Defaults:** max_age=24h (reap), purge_age=72h (purge), stale_issue_age=168h
  (auto-close), mail_delete_age=72h, alert_threshold=500 open wisps,
  dolt_port=3307
- **Auto-close safety:** never touches P0/P1 issues, epics, or issues with
  active dependencies.
- **Protocol:** scan/reap count mismatch is NORMAL (witness closes wisps
  concurrently) — the command explicitly tells the agent not to escalate
  on mismatch, only on actual errors.
- **Related:** package [../packages/reaper.md](../packages/reaper.md),
  role [../roles/reaper.md](../roles/reaper.md).

## Skills

### crew-commit — `.claude/skills/crew-commit/SKILL.md`

- **Description:** Canonical commit workflow for Gas Town crew members:
  pre-flight checks, branch creation, `gt commit` with agent identity,
  push, and PR creation.
- **Allowed tools:** `git *`, `gt *`, `gh *`
- **Version:** 1.0.0
- **8 steps:** pre-flight (fetch + rebase) → feature branch
  (`<type>/<description>`) → submodule warning → stage → commit (uses
  `gt commit` not `git commit` — `gt commit` sets agent identity based on
  `GT_ROLE`) → push → `gh pr create` → notify.
- **Branch types:** `feat/`, `fix/`, `refactor/`, `docs/`, `chore/`,
  `test/`. Crew-specific prefix: `crew/<name>/description`.
- **Guardrail:** "NEVER commit directly to main. All crew work goes
  through branches and pull requests. The Refinery handles merges to main."

### ghi-list — `.claude/skills/ghi-list/SKILL.md`

- **Description:** List GitHub issues in a formatted ASCII table.
- **Allowed tools:** `gh issue list:*`
- **Output format:** Unicode box-drawing table with columns Issue /
  Assignee / Labels / Title / State.

### pr-list — `.claude/skills/pr-list/SKILL.md`

- **Description:** List GitHub PRs in a formatted ASCII table. Supports
  filters like `--state`, `--author`, `--label`.
- **Allowed tools:** `gh pr list:*`
- **Filter default:** strips PRs with `reviewDecision=CHANGES_REQUESTED`
  unless `--all-reviews` is passed.
- **Output format:** Unicode box-drawing table with columns PR / Author /
  Title / State.

### pr-sheriff — `.claude/skills/pr-sheriff/skill.md`

- **Description:** PR Sheriff workflow — triage PRs into easy-wins and
  crew assignments. Prints recommendations inline, does NOT post to GitHub.
- **Allowed tools:** `gh pr *`, `git *`, `gt *`, `bd *`, `cat *`
- **Version:** 2.0.0
- **Delegates to:** `mol-pr-sheriff-patrol` formula at
  `$GT_ROOT/.beads/formulas/mol-pr-sheriff-patrol.formula.toml`.
- **Repo scope:** this rig (`gastown/crew/max`) is responsible for
  `steveyegge/gastown` only. The beads repo is handled by `beads/crew/emma`.
- **Config:** `$GT_ROOT/.beads/pr-sheriff-config.json` — crew mappings,
  contributor trust tiers, bot auto-merge rules.
- **Formula steps (9):** load-config → discover-prs → triage-batch →
  merge-easy-wins → dispatch-crew-reviews → dispatch-deep-reviews →
  collect-results → interactive-review → summarize.
- **Contributor tiers:** `bot-trusted` (auto-merge if CI green),
  `community` (normal triage), `firewalled` (always NEEDS-HUMAN, deep
  review required).
- **Decision tree:** Draft → SKIP; firewalled → NEEDS-HUMAN; Dependabot
  patch bump + green → EASY-WIN; <50 lines obvious fix → EASY-WIN;
  security/architecture/API → NEEDS-HUMAN; multi-concern → NEEDS-HUMAN;
  100+ lines new feature → NEEDS-CREW or NEEDS-HUMAN; else NEEDS-CREW.
- **Dispatching:** uses **ephemeral beads (wisps)** (`bd new ... --wisp-type
  patrol`) for fix-merge dispatch to keep orchestration noise out of the
  permanent Dolt ledger.

## How they're consumed

The Claude Code agent reads `.claude/commands/` and `.claude/skills/`
at startup from the current working directory. The slash command
registration is a Claude Code feature, not a gastown feature — the
gastown project merely ships markdown that the agent discovers.

**This is pure convention over configuration**: there is no code in
`internal/` that parses these files. They're consumed by the external
Claude Code binary. The `allowed-tools` frontmatter acts as a permissions
contract between the repo and the agent.

## Relationship to other surfaces

| Surface | Audience | Notes |
|---|---|---|
| `.claude/commands/` | Claude Code agent | Slash commands loaded per-repo |
| `.claude/skills/` | Claude Code agent | Skills loaded per-repo |
| [.opencode/](opencode-dir.md) | OpenCode agent | Parallel tree — one command + one plugin |
| [templates/agents/](templates-agents.md) | Gas Town runtime | Per-role runtime templates (distinct layer — runtime-level agent templating) |
| [../packages/hooks.md](../packages/hooks.md) | Claude Code agent | Hook settings installer (consumed by `scripts/guards/context-budget-guard.sh`) |
| [../packages/templates.md](../packages/templates.md) | Gas Town runtime | Embedded role CLAUDE.md templates — distinct from this per-repo set |

## Notes / open questions

- The `/patrol` slash command duplicates logic that lives in the role
  formulas (`mol-witness-patrol`, `mol-deacon-patrol`,
  `mol-refinery-patrol`). The formulas are the source of truth; this
  command is a prose rendition for the agent. Drift risk.
- The `/backup` command is a prose rendition of what the `dolt-backup`
  plugin does declaratively — see [../plugins/README.md](../plugins/README.md).
- Filename casing is inconsistent: `SKILL.md` × 3 vs `skill.md` × 1. Not
  yet verified whether Claude Code loader is case-sensitive or not.
- Skill versioning (`version: 1.0.0`, `2.0.0`) exists in frontmatter but
  there's no changelog of what changed between versions.

## Related pages

- [opencode-dir.md](opencode-dir.md) — sibling `.opencode/` tree
- [templates-agents.md](templates-agents.md) — per-role agent templates
  (runtime layer)
- [../binaries/gt.md](../binaries/gt.md) — the main CLI; all slash
  commands shell out to `gt`
- [../commands/patrol.md](../commands/patrol.md) — the `gt patrol` subcommand
- [../commands/reaper.md](../commands/reaper.md) — the `gt reaper` subcommand
- [../packages/templates.md](../packages/templates.md) — embedded role
  templates (different layer)
- [../packages/runtime.md](../packages/runtime.md) — agent-runtime hook
  installers
- [../commands/prime.md](../commands/prime.md) — consumed by OpenCode plugin
- [../roles/polecat.md](../roles/polecat.md), [../roles/witness.md](../roles/witness.md), [../roles/deacon.md](../roles/deacon.md), [../roles/refinery.md](../roles/refinery.md), [../roles/crew.md](../roles/crew.md)
- [../inventory/auxiliary.md](../inventory/auxiliary.md) — A-level inventory
