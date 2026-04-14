---
title: internal/config
type: package
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/config/agents.go
  - /home/kimberly/repos/gastown/internal/config/loader.go
  - /home/kimberly/repos/gastown/internal/config/env.go
  - /home/kimberly/repos/gastown/internal/config/types.go
  - /home/kimberly/repos/gastown/internal/config/operational.go
  - /home/kimberly/repos/gastown/internal/config/cost_tier.go
  - /home/kimberly/repos/gastown/internal/config/directives.go
  - /home/kimberly/repos/gastown/internal/config/overseer.go
  - /home/kimberly/repos/gastown/internal/config/roles.go
tags: [package, platform-service, config, settings, env-vars, agents, roles, town-settings, cost-tier]
---

# internal/config

The single largest `internal/` package in the codebase. Central
configuration layer for Gas Town: defines all persisted config shapes
(town, rigs, mayor, settings, operational thresholds, messaging,
accounts, daemon patrols, escalation, merge queues, runtime config,
agent registry, overseer identity, namepools, themes, role definitions),
loads and validates them from disk, resolves which agent runtime
(Claude / Gemini / Codex / Cursor / Auggie / Amp / OpenCode / Copilot /
Pi / Omp) should run for a given role and rig, builds the environment
variable map and shell startup command that actually launches each
agent, and provides the cost-tier system for bulk model selection.

**Go package path:** `github.com/steveyegge/gastown/internal/config`
**File count:** 9 non-test go files plus an embedded `roles/*.toml`
directory. 12 test files. `loader.go` alone is ~92 KB and `types.go`
~67 KB, making this the largest surface area in the whole repo.
**Imports (notable):** stdlib (`encoding/json`, `os`, `os/exec`,
`path/filepath`, `sync`, `runtime`, `sort`, `strings`, `strconv`,
`embed`, `time`), `github.com/BurntSushi/toml` (for role TOML),
`github.com/steveyegge/gastown/internal/constants`,
`github.com/steveyegge/gastown/internal/scheduler/capacity`.
**Imported by (notable):** [`gt`](../binaries/gt.md) root command
(`/home/kimberly/repos/gastown/internal/cmd/root.go:14`),
[`internal/workspace`](workspace.md) (`workspace.GetTownName` calls
`config.LoadTownConfig`), [`internal/session`](session.md) (session
package consumes `config.RuntimeConfig`, `config.AgentEnvConfig`,
`config.BuildStartupCommand*`), and practically every user-facing
command that needs to read or modify persistent config — far too many
to enumerate individually, but prominent hits include
[`gt config`](../commands/config.md), [`gt agents`](../commands/agents.md),
[`gt costs`](../commands/costs.md), [`gt rigs`](../commands/rigs.md),
[`gt mayor`](../commands/mayor.md), [`gt overseer`](../commands/overseer.md),
[`gt init`](../commands/init.md),
[`gt escalate`](../commands/escalate.md), and the whole messaging /
mail surface.

## What it actually does

Nine non-test files, each with a clear functional focus. This page
summarizes the responsibilities of each file and enumerates the public
API. Because of the sheer volume, drill-down details per type/function
have been kept terse; see `file:line` refs when you need the body.

### agents.go — agent runtime registry

Source: `/home/kimberly/repos/gastown/internal/config/agents.go`.

**Purpose:** single source of truth for "which LLM CLIs can Gas Town
drive, and how". A global registry of `AgentPresetInfo` entries keyed
by `AgentPreset` strings. Each preset describes the command to invoke,
its session-resume behaviour, process names, ACP (Agent Client
Protocol) config, and session-ID env var. Designed so that **adding a
new agent = adding a `builtinPresets` entry** (plus an optional hook
installer) with no `switch agent.Name {}` statements leaking outside
this file.

**Built-in preset enum** (`agents.go:14-39`): `AgentClaude`,
`AgentGemini`, `AgentCodex`, `AgentCursor`, `AgentAuggie`, `AgentAmp`,
`AgentOpenCode`, `AgentCopilot`, `AgentPi`, `AgentOmp`. Comment on
`AgentOmp` credits the `github.com/ProbabilityEngineer/pi-mono` Gas
Town integration.

**Public API:**

- `AgentPreset`, `AgentPresetInfo`, `ACPConfig`, `NonInteractiveConfig`,
  `AgentRegistry` types (`agents.go:14-210`).
- `LoadAgentRegistry(path)` / `LoadRigAgentRegistry(path)` and the
  corresponding path helpers `DefaultAgentRegistryPath(townRoot)`,
  `DefaultRigAgentRegistryPath(rigPath)`, `RigAgentRegistryPath(rigPath)`
  (`agents.go:497-529`).
- `GetAgentPreset(name)`, `GetAgentPresetByName(name)`,
  `ListAgentPresets()`, `DefaultAgentPreset()`, `IsKnownPreset(name)`
  (`agents.go:531-755`) — lookups against the global registry.
- `RuntimeConfigFromPreset(preset) *RuntimeConfig` (`agents.go:567-600`) —
  materialises a `RuntimeConfig` from a preset + defaults.
- `BuildResumeCommand(agentName, sessionID) string`,
  `SupportsSessionResume(agentName) bool`,
  `GetSessionIDEnvVar(agentName) string`,
  `GetProcessNames(agentName)`, `ResolveProcessNames(agentName, command)`
  (`agents.go:602-745`) — session-resume and process-matching helpers.
- `SaveAgentRegistry(path, registry)`, `NewExampleAgentRegistry()`
  (`agents.go:756-791`).
- ACP (Agent Client Protocol) helpers: `ResolveACPConfig`,
  `SupportsACP`, `GetACPConfig`, `GetACPCommand`, `GetACPArgs`,
  `RuntimeConfigSupportsACP`, `GetACPConfigFromRuntime`
  (`agents.go:813-935`).
- Test hooks: `ResetRegistryForTesting()`,
  `RegisterAgentForTesting(name, info)` (`agents.go:793-810`).

### loader.go — the load/save/resolve bulk

Source: `/home/kimberly/repos/gastown/internal/config/loader.go`
(~92 KB, the largest file in the package).

**Purpose:** every `Load*` / `Save*` function for every persisted
config file in Gas Town, plus all the agent-resolution and
startup-command-builder logic that translates "this role in this rig"
into a concrete shell command line to spawn in a tmux pane.

**Guarantees / concurrency:** `resolveConfigMu` (package-level
`sync.Mutex`, `loader.go:20-22`) serialises `ResolveRoleAgentConfig`
and `ResolveAgentConfig` so concurrent lookups for different rigs
can't corrupt the shared agent registry.

**Error sentinels** (`loader.go:25-36`): `ErrNotFound`,
`ErrInvalidVersion`, `ErrInvalidType`, `ErrMissingField`.

**Load/save pairs** (names are self-describing):

- `LoadTownConfig` / `SaveTownConfig` (`loader.go:40-82`) —
  `mayor/town.json` (town identity).
- `LoadRigsConfig` / `SaveRigsConfig` (`loader.go:84-151`).
- `LoadRigConfig` / `SaveRigConfig` (`loader.go:153-265`).
- `LoadRepoSettings` / `LoadRigSettings` / `SaveRigSettings`
  (`loader.go:293-447`).
- `LoadMayorConfig` / `SaveMayorConfig` (`loader.go:448-501`) —
  `mayor/config.json` (behavioural config, distinct from identity).
- `LoadDaemonPatrolConfig` / `SaveDaemonPatrolConfig` /
  `EnsureDaemonPatrolConfig` / `AddRigToDaemonPatrols` /
  `RemoveRigFromDaemonPatrols` (`loader.go:511-775`).
- `LoadAccountsConfig` / `SaveAccountsConfig` / `NewAccountsConfig` /
  `ResolveAccountConfigDir` (`loader.go:777-922`).
- `LoadMessagingConfig` / `SaveMessagingConfig` /
  `LoadOrCreateMessagingConfig` / `MessagingConfigPath`
  (`loader.go:924-1046`).
- `LoadOrCreateTownSettings` / `SaveTownSettings` / `TownSettingsPath`
  (`loader.go:1048-1110`).
- `LoadEscalationConfig` / `LoadOrCreateEscalationConfig` /
  `SaveEscalationConfig` / `EscalationConfigPath`
  (`loader.go:2725-2770`).

**Factories / defaults:** `NewRigConfig`, `NewRigSettings`,
`NewMayorConfig` (`loader.go:267-510`).

**Merge-queue helpers:** `MergeSettingsCommand(repo, local
*MergeQueueConfig)` (`loader.go:314-377`) — layers repo-level and
local merge-queue config with local taking precedence.

**Agent resolution (the heart of the file):**

- `ResolveAgentConfig(townRoot, rigPath) *RuntimeConfig`
  (`loader.go:1112-1165`).
- `ResolveAgentConfigWithOverride(townRoot, rigPath, agentOverride)
  (*RuntimeConfig, string, error)` (`loader.go:1167-1268`).
- `ValidateAgentConfig(agentName, townSettings, rigSettings) error`
  (`loader.go:1270-1322`).
- `ResolveRoleAgentConfig(role, townRoot, rigPath) *RuntimeConfig`
  (`loader.go:1324-1354`) — role-level override lookup.
- `ResolveWorkerAgentConfig(workerName, townRoot, rigPath)
  *RuntimeConfig` (`loader.go:1356-1403`) — worker-specific lookup.
- `IsResolvedAgentClaude(rc *RuntimeConfig) bool`
  (`loader.go:1405-1469`) — post-resolve type check; several call
  sites fork Claude-vs-other behaviour.
- `RoleSettingsDir(role, rigPath) string`
  (`loader.go:1471-1510ish`) — where a role's per-rig overrides live.
- `ResolveRoleAgentName(role, townRoot, rigPath) (agentName string,
  isRoleSpecific bool)` (`loader.go:1670-1713`).
- `ResolveAgentConfigByName(name, townRoot, rigPath) *RuntimeConfig`
  (`loader.go:1715-1743`).
- `HasExplicitRoleAgent(role, townRoot, rigPath) bool`
  (`loader.go:1745-...`).
- `GetRuntimeCommand(rigPath) string`,
  `GetRuntimeCommandWithAgentOverride(rigPath, agentOverride)
  (string, error)`, `GetRuntimeCommandWithPrompt` / `...AgentOverride`
  (`loader.go:1994-2103`).

**Startup command builders (the thing that finally produces the
shell one-liner that spawns a pane):**

- `BuildStartupCommand(envVars, rigPath, prompt) string`
  (`loader.go:2138-2310`).
- `BuildStartupCommandWithAgentOverride(envVars, rigPath, prompt,
  agentOverride) (string, error)` (`loader.go:2365-2541`).
- `BuildStartupCommandFromConfig(cfg AgentEnvConfig, rigPath, prompt,
  agentOverride)` (`loader.go:2543-2551`).
- `BuildAgentStartupCommand` / `BuildAgentStartupCommandWithAgentOverride`
  — role/rig-level wrappers (`loader.go:2552-2573`).
- `BuildPolecatStartupCommand` / `BuildPolecatStartupCommandWithAgentOverride`
  (`loader.go:2575-2607`).
- `BuildCrewStartupCommand` / `BuildCrewStartupCommandWithAgentOverride`
  (`loader.go:2608-2654`).
- `SanitizeAgentEnv(resolvedEnv, callerEnv map[string]string)`
  (`loader.go:2312-2335`) — strips identity env vars (see `env.go`)
  from resolved config before they leak back into caller scope. See
  GH#3006 referenced in `env.go`.
- `PrependEnv(command, envVars) string` (`loader.go:2337-2363`) —
  builds the leading `FOO=bar BAZ=qux <command>` portion.
- `ExpectedPaneCommands(rc *RuntimeConfig) []string`
  (`loader.go:2656-2667`) — list of process names a pane is expected
  to contain, used by liveness checks.

**Misc helpers:**

- `GetDefaultFormula(rigPath) string`,
  `GetRigPrefix(townRoot, rigName) string`,
  `AllRigPrefixes(townRoot) []string`, `ExtractSimpleRole(gtRole)`
  (`loader.go:2669-2723`, `2105-2136`).

### env.go — agent environment variable generation

Source: `/home/kimberly/repos/gastown/internal/config/env.go`.

**Purpose:** builds the `map[string]string` of environment variables
every agent session should run under. Single source of truth —
comment explicitly says so at `env.go:26`.

**Public API:**

- `IdentityEnvVars` — package-level `[]string` listing the vars that
  must NOT leak across process/session boundaries
  (`env.go:20-23`): `GT_ROLE`, `GT_RIG`, `GT_CREW`, `GT_POLECAT`,
  `GT_DOG_NAME`, `GT_SESSION`, `GT_AGENT`, `BD_ACTOR`,
  `GIT_AUTHOR_NAME`, `BEADS_AGENT_NAME`. Referenced by daemon
  sanitisation, tmux global cleanup, and prime session env repair.
  GH#3006.
- `AgentEnvConfig` struct (`env.go:28-77`) — input to `AgentEnv`.
  Fields include `Role`, `Rig`, `AgentName`, `TownRoot`,
  `RuntimeConfigDir`, `SessionIDEnv`, `Agent` (agent override), plus
  more.
- `AgentEnv(cfg AgentEnvConfig) map[string]string` (`env.go:79-502`)
  — the main generator. Produces the env map.
- `AgentEnvSimple(role, rig, agentName) map[string]string`
  (`env.go:503-512`) — convenience wrapper.
- `ShellQuote(s string) string` (`env.go:514-545`) — POSIX single-
  quote escaping for embedding values in shell command strings.
- `ExportPrefix(env map[string]string) string`
  (`env.go:547-574`) — renders the env map as `export A=... B=...;`
  shell prefix.
- `BuildStartupCommandWithEnv(env, agentCmd, prompt) string`
  (`env.go:576-585`).
- `MergeEnv(maps ...) map[string]string`,
  `FilterEnv(env, keys...) map[string]string`,
  `WithoutEnv(env, keys...) map[string]string` (`env.go:587-623`)
  — map utility primitives.
- `EnvForExecCommand(env) []string`, `EnvToSlice(env) []string`
  (`env.go:625-645`) — conversions to `exec.Cmd.Env`'s `[]string`
  format.
- `ClaudeConfigDir() (string, error)` (`env.go:647-...`) — resolves
  `CLAUDE_CONFIG_DIR`.

### types.go — persisted config shapes

Source: `/home/kimberly/repos/gastown/internal/config/types.go`
(~67 KB; essentially a catalogue of every JSON/TOML config shape Gas
Town reads or writes).

**Top-level town identity and behaviour:**

- `TownConfig` (`types.go:17-25`) — `mayor/town.json`: `Type`,
  `Version`, `Name`, `Owner`, `PublicName`, `CreatedAt`.
- `MayorConfig` (`types.go:28-36`) — `mayor/config.json`: behaviour,
  theme, daemon, deacon, default crew name.
- `TownSettings` (`types.go:42-118`) — `settings/config.json`: the
  big behaviour-flag config. `CurrentTownSettingsVersion = 1` at
  `types.go:40`. Includes `CLITheme` field (read by
  [`internal/ui`](ui.md)'s `InitTheme`).
- `NewTownSettings() *TownSettings` (`types.go:119-128`).
- `OperationalConfig` + nested `*Thresholds` types
  (`types.go:208-483`): `SessionThresholds`, `NudgeThresholds`,
  `DaemonThresholds`, `DeaconThresholds`, `PolecatThresholds`,
  `DoltThresholds`, `MailThresholds`, `WebThresholds`,
  `WitnessThresholds`. `DefaultOperationalConfig()` at
  `types.go:484-487`.
- `WebTimeoutsConfig`, `WorkerStatusConfig`, `FeedCuratorConfig`,
  `ConvoyConfig`, `DaemonConfig`, `DaemonPatrolConfig`,
  `HeartbeatConfig`, `PatrolConfig`, `DeaconConfig`, `RigsConfig`,
  `RigEntry`, `BeadsConfig`, `RigConfig`, `WorkflowConfig`,
  `RigSettings`, `CrewConfig`.
- `RuntimeConfig` (`types.go:692-750`) and its sub-types
  `RuntimeSessionConfig`, `RuntimeHooksConfig`, `RuntimeTmuxConfig`,
  `RuntimeInstructionsConfig` (`types.go:752-797`),
  `DefaultRuntimeConfig()` (`types.go:799-...`).
- Theme/tint: `ThemeConfig`, `CustomTheme`, `TownThemeConfig`,
  `WindowTint`, `BuiltinRoleThemes() map[string]string`
  (`types.go:1139-1220`).
- Merge queue: `MergeQueueConfig`, `DefaultMergeQueueConfig()`
  (`types.go:1221-1408`).
- Namepool: `NamepoolConfig`, `DefaultNamepoolConfig()`
  (`types.go:1410-1432`).
- Accounts / quotas: `AccountsConfig`, `Account`, `QuotaState`,
  `AccountQuotaStatus`, `AccountQuotaState`,
  `DefaultAccountsConfigDir() (string, error)`
  (`types.go:1434-1502`).
- Messaging: `MessagingConfig`, `QueueConfig`, `AnnounceConfig`,
  `NewMessagingConfig()` (`types.go:1504-1563`).
- Escalation: `EscalationConfig`, `EscalationContacts`,
  `ValidSeverities()`, `IsValidSeverity(severity)`,
  `NextSeverity(severity)`, `NewEscalationConfig()`
  (`types.go:1566-1665`).
- `ParseDurationOrDefault(s string, fallback time.Duration)
  time.Duration` (`types.go:496-506`) — shared helper used across
  threshold parsing.

### operational.go — compiled-in operational defaults

Source: `/home/kimberly/repos/gastown/internal/config/operational.go`.

Compiled-in `const` defaults for every `*Thresholds` type, grouped by
consumer (session, nudge, daemon, deacon, polecat, dolt, mail, web,
witness). Each one was previously a hardcoded `const` scattered
across the codebase before being pulled into this single file.
`LoadOperationalConfig(townRoot) *OperationalConfig`
(`operational.go:119-...`) materialises a config, layering any
per-town overrides on top of the compiled defaults.

### cost_tier.go — bulk model selection tiers

Source: `/home/kimberly/repos/gastown/internal/config/cost_tier.go`.

Three predefined cost tiers that map role → Claude model variant in
bulk, so users can switch the whole town from opus-everywhere to
sonnet/haiku without editing per-role settings.

**Public API:**

- `CostTier` string type; constants `TierStandard`, `TierEconomy`,
  `TierBudget` (`cost_tier.go:9-18`).
- `ValidCostTiers() []string`, `IsValidTier(tier string) bool`
  (`cost_tier.go:21-33`).
- `TierManagedRoles` package-level slice (`cost_tier.go:38`):
  `["mayor", "deacon", "witness", "refinery", "polecat", "crew",
  "boot", "dog"]`. These are the only roles `ApplyCostTier` touches;
  custom user roles are preserved. Boot and dog are utility roles
  that always use haiku even on the standard tier.
- `CostTierRoleAgents(tier) map[string]string` (`cost_tier.go:46-89`)
  — role → agent-name mapping for a tier.
- `CostTierAgents(tier) map[string]*RuntimeConfig`
  (`cost_tier.go:91-124`) — richer mapping including full
  RuntimeConfig.
- `ApplyCostTier(settings *TownSettings, tier CostTier) error`
  (`cost_tier.go:126-173`) — mutates a `TownSettings` in place.
- `GetCurrentTier(settings *TownSettings) string`
  (`cost_tier.go:175-210`) — reverse lookup, "which tier matches
  this settings file?"
- `TierDescription(tier CostTier) string`,
  `FormatTierRoleTable(tier CostTier) string` — user-facing strings
  (`cost_tier.go:212-...`).

### directives.go — role directive loading

Source: `/home/kimberly/repos/gastown/internal/config/directives.go`.

Tiny file. One public function:

- `LoadRoleDirective(role, townRoot, rigName) string`
  (`directives.go:19-42`) — reads `<townRoot>/directives/<role>.md`
  (town-level) and `<townRoot>/<rigName>/directives/<role>.md` (rig-
  level), concatenates them with rig-level appearing last (so it
  wins), returns empty string if neither exists. Invalid or
  unreadable paths are treated as absent — no error propagation.

### overseer.go — human operator identity

Source: `/home/kimberly/repos/gastown/internal/config/overseer.go`.

Loads and manages `mayor/overseer.json`: the human who operates the
town, distinct from AI agents (see the
[identity concept page](../concepts/identity.md) for the agent side
of the same system). Fields: `Name`, `Email`, `Username`, `Source`
(how identity was detected).

**Public API:**

- `OverseerConfig` struct (`overseer.go:15-24`),
  `CurrentOverseerVersion = 1` (`overseer.go:27`).
- `OverseerConfigPath(townRoot) string` (`overseer.go:28-30`).
- `LoadOverseerConfig(path) (*OverseerConfig, error)`
  (`overseer.go:33-53`).
- `SaveOverseerConfig(path, config) error` (`overseer.go:55-99`).
- `DetectOverseer(townRoot) (*OverseerConfig, error)`
  (`overseer.go:101-222`) — auto-detection from git config, env
  vars, OS user.
- `LoadOrDetectOverseer(townRoot) (*OverseerConfig, error)`
  (`overseer.go:224-...`) — load if exists, else detect and save.

### roles.go — role definitions from embedded TOML

Source: `/home/kimberly/repos/gastown/internal/config/roles.go`.

Role definitions are packaged as `roles/*.toml` files embedded via
`//go:embed roles/*.toml` (`roles.go:14-15`). This replaces an earlier
"role bead" system with config files.

**Public API:**

- `RoleDefinition` struct (`roles.go:20-41`): `Role`, `Scope`
  (`"town"` or `"rig"`), `Session` (tmux pattern + workdir), `Env`,
  `Health`, `Nudge` (initial prompt), `PromptTemplate`.
- `RoleSessionConfig`, `RoleHealthConfig`, `Duration` helper types
  (`roles.go:44-108`).
- `AllRoles()`, `TownRoles()`, `RigRoles() []string`
  (`roles.go:109-140`).
- `LoadRoleDefinition(townRoot, rigPath, roleName)
  (*RoleDefinition, error)` (`roles.go:141-275`) — merges embedded
  default with any on-disk override under
  `<townRoot>/directives/` or `<rigPath>/directives/`.
- `ExpandPattern(pattern, townRoot, rig, name, role, prefix) string`
  (`roles.go:277-...`) — substitutes `{rig}`, `{name}`, `{role}`,
  etc. into the session pattern.

### Usage pattern (across the whole package)

Typical command flow:

1. `workspace.FindFromCwdOrError()` → `townRoot`.
2. `config.LoadTownConfig(townRoot/mayor/town.json)` → town identity.
3. `config.LoadOrCreateTownSettings(...)` → behaviour settings.
4. `config.LoadOrCreateEscalationConfig(...)` if escalation-relevant.
5. For agent launches: `config.ResolveAgentConfig(...)` →
   `*RuntimeConfig`; `config.AgentEnv(AgentEnvConfig{...})` → env
   map; `config.BuildStartupCommandWithAgentOverride(...)` → shell
   one-liner; that string becomes the tmux pane command.

## Related wiki pages

- [gt](../binaries/gt.md) — main binary; imports config from root
  command init.
- [internal/workspace](workspace.md) — upstream consumer that calls
  `config.LoadTownConfig` via `GetTownName`.
- [internal/session](session.md) — downstream consumer of
  `RuntimeConfig`, `AgentEnvConfig`, and the `BuildStartupCommand*`
  family.
- [gt config](../commands/config.md), [gt agents](../commands/agents.md),
  [gt costs](../commands/costs.md), [gt rigs](../commands/rigs.md),
  [gt mayor](../commands/mayor.md), [gt overseer](../commands/overseer.md)
  — key user-facing consumers of the corresponding loader functions.
- [go-mod](../files/go-mod.md) — dependency context; BurntSushi/toml
  and scheduler/capacity cross the module boundary from here.
- [go-packages inventory](../inventory/go-packages.md).

## Notes / open questions

- `types.go` is enormous (~67 KB). A future split by feature
  (`types_messaging.go`, `types_merge.go`, `types_daemon.go`, …)
  would reduce merge conflicts significantly. Neutral observation.
- There are two overlapping "build startup command" APIs: the
  `BuildStartupCommand*` family in `loader.go` and the `env.go`
  helpers (`ExportPrefix`, `BuildStartupCommandWithEnv`). Callers
  fork between them based on whether they already have a resolved
  env map or want the loader to resolve one. Worth a decision note
  in a future pass if the split intent isn't obvious.
- `IdentityEnvVars` at `env.go:20-23` is the canonical list for
  session-hygiene cleanup. Anything grepping for "identity leak"
  should start here.
- `ResolveAgentConfig`'s mutex (`resolveConfigMu`) is a global lock;
  under heavy concurrent lookups across many rigs this could
  serialise more than necessary. Not observed as a bottleneck.
- The embedded `roles/*.toml` directory (under
  `internal/config/roles/`) was not itself read for this page — only
  the Go code that consumes it. A future ingest should read the TOML
  files and file them under `gastown/config/` entity pages for each
  role.
