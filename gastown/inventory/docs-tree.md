---
title: gastown docs/ tree inventory
type: note
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/docs/
tags: [inventory, enumeration, docs, batch-12]
---

# gastown `docs/` tree inventory

Every file under `/home/kimberly/repos/gastown/docs/` with line count,
topic extract, and path, as of 2026-04-11. Neutral enumeration — no
summaries of content, no claim about audience.

**Capture method:** `find /home/kimberly/repos/gastown/docs -type f` +
`wc -l` per file for line counts (Batch 1) +
first-paragraph / first-H1 extraction for topic (Batch 12). Topics are
factual extracts, not content summaries — they match the file's
stated subject, not our interpretation of it.

**Total:** 60 files.

## docs/ (root level — 12 files)

| lines | topic                                                          | path                                          |
|------:|----------------------------------------------------------------|-----------------------------------------------|
|   835 | Agent Provider Integration Guide                               | `docs/agent-provider-integration.md`          |
|   170 | Gastown/Beads Cleanup Commands Reference                       | `docs/CLEANUP.md`                             |
|   251 | Gas Town Hooks Management                                      | `docs/HOOKS.md`                               |
|   321 | Installing Gas Town                                            | `docs/INSTALLING.md`                          |
|   469 | Getting Started with the Wasteland                             | `docs/WASTELAND.md`                           |
|    94 | Gas Town Glossary                                              | `docs/glossary.md`                            |
|   299 | Gastown OTel Data Model                                        | `docs/otel-data-model.md`                     |
|   204 | Understanding Gas Town (conceptual overview)                   | `docs/overview.md`                            |
|    39 | Phase 4 Minimum Fix — acceptance commands and PR draft         | `docs/phase4-minimum-fix-acceptance.md`       |
|   621 | gt-proxy-server and gt-proxy-client                            | `docs/proxy-server.md`                        |
|   767 | Gas Town Reference (technical internals)                       | `docs/reference.md`                           |
|   264 | Why These Features? (enterprise AI rationale)                  | `docs/why-these-features.md`                  |

## docs/concepts/ (6 files)

| lines | topic                                                          | path                                          |
|------:|----------------------------------------------------------------|-----------------------------------------------|
|   251 | Convoys — primary unit for tracking batched work across rigs   | `docs/concepts/convoy.md`                     |
|   286 | Agent Identity and Attribution                                 | `docs/concepts/identity.md`                   |
|   590 | Integration Branches — group epic work, land as a unit         | `docs/concepts/integration-branches.md`       |
|   132 | Molecules — workflow templates for multi-step work             | `docs/concepts/molecules.md`                  |
|   381 | Polecat Lifecycle — three-layer architecture of polecat workers| `docs/concepts/polecat-lifecycle.md`          |
|   102 | The Propulsion Principle — if it's on your hook, YOU RUN IT    | `docs/concepts/propulsion-principle.md`       |

## docs/design/ (root level — 23 files)

| lines | topic                                                          | path                                                 |
|------:|----------------------------------------------------------------|------------------------------------------------------|
|   853 | Agent API Touch-Point Inventory                                | `docs/design/agent-api-inventory.md`                 |
|   381 | Gas Town Architecture (two-level beads)                        | `docs/design/architecture.md`                        |
|   307 | Role Directives and Formula Overlays                           | `docs/design/directives-and-overlays.md`             |
|    83 | Dog Execution Model: Imperative vs Formula Dispatch            | `docs/design/dog-execution-model.md`                 |
|   509 | Dog Infrastructure: Watchdog Chain & Pool Architecture         | `docs/design/dog-infrastructure.md`                  |
|   545 | Dolt Storage Architecture                                      | `docs/design/dolt-storage.md`                        |
|   214 | Gas Town Escalation Protocol                                   | `docs/design/escalation.md`                          |
|   299 | Factory Worker API — GT↔agent runtime boundary                 | `docs/design/factory-worker-api.md`                  |
|   209 | Federation Architecture — multi-workspace coordination         | `docs/design/federation.md`                          |
|   249 | Formula Resolution Architecture                                | `docs/design/formula-resolution.md`                  |
|   640 | Ledger Export Triggers — Operational→Ledger plane transitions  | `docs/design/ledger-export-triggers.md`              |
|   565 | Gas Town Mail Protocol                                         | `docs/design/mail-protocol.md`                       |
|   635 | Model-Aware Molecule Constraints                               | `docs/design/model-aware-molecules.md`               |
|   475 | Mol Mall Design — marketplace for Gas Town formulas            | `docs/design/mol-mall-design.md`                     |
|   203 | Persistent Polecat Pool (gt-lpop)                              | `docs/design/persistent-polecat-pool.md`             |
|   276 | Plugin System Design                                           | `docs/design/plugin-system.md`                       |
|   664 | Polecat Lifecycle and Patrol Coordination                      | `docs/design/polecat-lifecycle-patrol.md`            |
|   462 | Polecat Self-Managed Completion                                | `docs/design/polecat-self-managed-completion.md`     |
|   414 | Property Layers: Multi-Level Configuration                     | `docs/design/property-layers.md`                     |
|   662 | Sandboxed Polecat Execution (exitbox + daytona)                | `docs/design/sandboxed-polecat-execution.md`         |
|   459 | Scheduler Architecture — config-driven dispatch                | `docs/design/scheduler.md`                           |
|    74 | Tmux Keybindings (Gas Town overrides)                          | `docs/design/tmux-keybindings.md`                    |
|   985 | Witness AT Team Lead: Implementation Spec                      | `docs/design/witness-at-team-lead.md`                |

## docs/design/convoy/ (4 files)

| lines | topic                                                          | path                                          |
|------:|----------------------------------------------------------------|-----------------------------------------------|
|   382 | Convoy Lifecycle Design — actively converge on completion      | `docs/design/convoy/convoy-lifecycle.md`      |
|   544 | The Mountain-Eater: Autonomous Epic Grinding                   | `docs/design/convoy/mountain-eater.md`        |
|   375 | Convoy Stability Roadmap                                       | `docs/design/convoy/roadmap.md`               |
|   640 | Convoy Manager Specification                                   | `docs/design/convoy/spec.md`                  |

## docs/design/convoy/stage-launch/ (3 files)

| lines | topic                                                                    | path                                                          |
|------:|--------------------------------------------------------------------------|---------------------------------------------------------------|
|     1 | Convoy stage-launch business-value insights data (single-line JSON blob) | `docs/design/convoy/stage-launch/bv-insights.json`            |
|   237 | PRD: Convoy Stage & Launch (`gt convoy stage`, `gt convoy launch`)       | `docs/design/convoy/stage-launch/prd.md`                      |
|   532 | Test Analysis: Convoy Stage & Launch                                     | `docs/design/convoy/stage-launch/testing.md`                  |

## docs/design/otel/ (2 files)

| lines | topic                                                          | path                                          |
|------:|----------------------------------------------------------------|-----------------------------------------------|
|   694 | OpenTelemetry Architecture (OTLP, backend-agnostic)            | `docs/design/otel/otel-architecture.md`       |
|   471 | OpenTelemetry Data Model — full event schema                   | `docs/design/otel/otel-data-model.md`         |

## docs/examples/ (4 files)

| lines | topic                                                          | path                                                |
|------:|----------------------------------------------------------------|-----------------------------------------------------|
|   372 | Polecat self-test formula for Gas Town integration validation  | `docs/examples/agent-validation.formula.toml`       |
|   169 | Towers of Hanoi Demo — durability proof for long workflows     | `docs/examples/hanoi-demo.md`                       |
|    77 | Rig-level settings example (`~/gt/<rig>/settings/config.json`) | `docs/examples/rig-settings.example.json`           |
|   102 | Town-level settings example (`~/gt/settings/config.json`)      | `docs/examples/town-settings.example.json`          |

## docs/gas-city/ (1 file)

| lines | topic                                                          | path                                          |
|------:|----------------------------------------------------------------|-----------------------------------------------|
|   402 | Crew Specialization and Capability-Based Dispatch              | `docs/gas-city/crew-specialization-design.md` |

## docs/guides/ (2 files)

| lines | topic                                                                   | path                                          |
|------:|-------------------------------------------------------------------------|-----------------------------------------------|
|    39 | Local Rig Bootstrap — NightRider-style clean bootstrap over --adopt     | `docs/guides/local-rig-bootstrap.md`          |
|  1217 | MVGT Integration Guide — Wasteland federation via Dolt without Gas Town | `docs/guides/mvgt-integration.md`             |

## docs/research/ (2 files)

| lines | topic                                                                  | path                                                    |
|------:|------------------------------------------------------------------------|---------------------------------------------------------|
|   315 | macOS sandbox-exec Research Report (gt-6qt)                            | `docs/research/macos-sandbox-exec.md`                   |
|   647 | Survey: Agent Orchestration Frameworks vs Gas City (w-gc-004)          | `docs/research/w-gc-004-agent-framework-survey.md`      |

## docs/skills/ (1 file)

| lines | topic                                                          | path                                      |
|------:|----------------------------------------------------------------|-------------------------------------------|
|   390 | Gastown Convoy System — definitive skill guide for convoy work | `docs/skills/convoy/SKILL.md`             |

## Totals

- **60 files total** under `docs/`.
- **10 subdirectories** (`concepts`, `design`, `design/convoy`,
  `design/convoy/stage-launch`, `design/otel`, `examples`, `gas-city`,
  `guides`, `research`, `skills`).
- **Line count total:** ~23,600 lines of documentation (sum of all
  line counts above).
- **Largest single file:** `docs/guides/mvgt-integration.md` at 1,217
  lines.
- **Second largest:** `docs/design/witness-at-team-lead.md` at 985
  lines.
- **Smallest substantive file:** `docs/guides/local-rig-bootstrap.md`
  at 39 lines (tied with `docs/phase4-minimum-fix-acceptance.md`).
- **One-byte stub:** `docs/design/convoy/stage-launch/bv-insights.json`
  at 1 line (single-line JSON blob with business-value insights data).
