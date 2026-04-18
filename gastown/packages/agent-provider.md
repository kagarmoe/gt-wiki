---
title: internal/agent/provider
type: package
status: verified
topic: gastown
created: 2026-04-16
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/agent/provider/acp.go
  - /home/kimberly/repos/gastown/internal/agent/provider/provider.go
tags: [package, agent, json-rpc, acp, provider, mcp, llm]
phase3_audited: 2026-04-16
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
phase8_audited: 2026-04-17
phase8_findings: [none]
---

# internal/agent/provider

JSON-RPC 2.0 provider types for LLM agent communication. This package
defines the complete type system for the Agent Client Protocol (ACP)
transport layer: roles, messages, content blocks, tools, and the
request/response framing that agents use to communicate with LLM
backends. It also provides a `LocalProvider` implementation for
in-process tool execution.

**Go package path:** `github.com/steveyegge/gastown/internal/agent/provider`
**File count:** 2 non-test Go files (acp.go 516 lines, provider.go
283 lines; 799 total).
**Imports (notable):** stdlib only (`context`, `encoding/json`, `fmt`,
`sync`, `time`). No internal gastown dependencies.
**Imported by:** No external importers at current HEAD; the package is
newly introduced infrastructure for the ACP protocol layer.

## What it actually does

### Core types (acp.go)

The type system follows the MCP (Model Context Protocol) / JSON-RPC 2.0
conventions:

**Roles and content types** (`acp.go:14-34`):

- `Role` string type with constants `RoleUser`, `RoleAssistant`,
  `RoleSystem`.
- `ContentType` string type with constants `ContentTypeText`,
  `ContentTypeToolUse`, `ContentTypeToolResult`, `ContentTypeImage`.

**Wire protocol** (`acp.go:36-55`):

- `JSONRPCRequest` -- standard JSON-RPC 2.0 request with `jsonrpc`,
  `id`, `method`, `params` fields (`acp.go:36-41`).
- `JSONRPCResponse` -- standard response with `result` or `error`
  (`acp.go:43-48`).
- `JSONRPCError` -- structured error with `code`, `message`, `data`
  (`acp.go:50-54`). Standard error codes defined at `acp.go:478-484`:
  `ParseError` (-32700), `InvalidRequest` (-32600), `MethodNotFound`
  (-32601), `InvalidParams` (-32602), `InternalError` (-32603).

**Content blocks** (`acp.go:56-68`):

- `ContentBlock` -- polymorphic content unit carrying text, tool-use
  invocations, tool results, or images. Uses `Type` discriminator field.
  Image support via embedded `ImageSource` struct (`acp.go:70-74`).

**Messages** (`acp.go:76-79`):

- `Message` -- a role + list of `ContentBlock` items. Constructor helpers
  at `acp.go:273-306`: `NewUserMessage`, `NewUserMessageWithContent`,
  `NewAssistantMessage`, `NewAssistantMessageWithContent`,
  `NewSystemMessage`.

**Tools** (`acp.go:81-92`):

- `Tool` -- name, description, and `InputSchema` (JSON Schema object
  with properties, required fields, and overflow `Additional` map).
- `InputSchema` has custom `MarshalJSON`/`UnmarshalJSON` at
  `acp.go:94-138` that merge the `Additional` overflow map into the
  base fields, allowing arbitrary extra schema properties to round-trip.

**Initialize handshake** (`acp.go:141-181`):

- `InitializeParams` / `InitializeResult` -- client and server
  capabilities, protocol version (`"2024-11-05"`), server info, and
  optional instructions string.
- `ClientCapabilities` / `ServerCapabilities` with `Tools` and
  `Resources` capability flags.

**Tool call/result types** (`acp.go:183-224`):

- `CallToolParams` / `CallToolResult` -- tool invocation request and
  response.
- `CreateMessageParams` / `CreateMessageResult` -- LLM message creation
  with model, temperature, max tokens, system prompts.
- `Usage` -- token usage tracking (input, output, total).

**SimpleMessage bridge** (`acp.go:308-350`):

- `SimpleMessage` struct for plain from/to/subject/body messaging.
- `TranslateSimpleMessage` and `TranslateMessageToSimple` -- convert
  between `SimpleMessage` and the rich `Message` type.

**Factory functions** (`acp.go:352-476`):

- `ToolFromDefinition` -- builds a `Tool` from name, description, and a
  schema map.
- `NewInitializeRequest`, `NewInitializeResponse`,
  `NewListToolsRequest`, `NewListToolsResponse`,
  `NewCallToolRequest`, `NewCallToolResponse`,
  `NewErrorResponse` -- convenience constructors for JSON-RPC
  request/response pairs for each ACP method.

**Parse helpers** (`acp.go:486-516`):

- `ParseRequest`, `ParseResponse` -- decode JSON bytes to typed
  request/response, validating the `jsonrpc: "2.0"` version field.
- `JSONRPCRequest.ParseParams(v any)` -- unmarshal the `params`
  field into a typed struct.

### Provider interface and implementation (provider.go)

**`ACPProvider` interface** (`provider.go:31-40`):

Defines the contract for LLM communication providers:
- `Initialize`, `ListTools`, `CallTool`, `CreateMessage` -- the four
  ACP protocol methods.
- `GetStatus` -- returns `AgentStatus` (state, session ID, agent name).
- `OnToolCall`, `OnSessionStart` -- callback registration.
- `Close` -- teardown.

**`ProviderState`** (`provider.go:10-18`): lifecycle FSM with states
`disconnected`, `connecting`, `ready`, `busy`, `error`.

**`BaseProvider`** (`provider.go:49-116`): thread-safe base
implementation with mutex-protected state, tool list management
(`AddTool`, `RemoveTool`), callback storage, and status reporting.

**`LocalProvider`** (`provider.go:118-181`): embeds `BaseProvider`,
adds an `instructions` field. Key behaviors:
- `Initialize` (`provider.go:130-152`) -- transitions to `StateReady`,
  returns protocol version `"2024-11-05"`, fires `sessionStart` callback.
- `CallTool` (`provider.go:154-172`) -- delegates to the registered
  `toolCallback`; returns an error content block if no callback is
  registered.
- `CreateMessage` (`provider.go:174-176`) -- returns "not supported"
  error (local providers don't create LLM messages).

**Message utility functions** (`provider.go:183-284`):

- `TranslateGastownMessage` -- converts from/to/subject/body into a
  `Message`.
- `ExtractToolCalls`, `ExtractToolResults`, `ExtractTextContent` --
  iterate over `ContentBlock` slices to extract typed data.
- `MessagesToJSON` / `MessagesFromJSON` -- JSON serialization helpers.
- `RequestToJSON` / `ResponseToJSON` / `RequestFromJSON` /
  `ResponseFromJSON` -- JSON-RPC wire-format helpers.
- `NewInitializedNotification` -- builds the `notifications/initialized`
  notification (no ID, per JSON-RPC notification convention).
- `IsNotification` -- checks whether a request is a notification
  (ID == nil).

## Related

- [internal/acp](acp.md) -- the ACP proxy that uses provider types
  for LLM communication.
- [internal/agent](agent.md) -- parent package providing shared agent
  types and `StateManager[T]`.
- [internal/protocol](protocol.md) -- higher-level protocol
  definitions that may consume provider types.
- [go-packages inventory](../inventory/go-packages.md)

## Notes / open questions

- The package has zero importers at current HEAD, suggesting it is
  newly extracted infrastructure. The type system closely mirrors the
  MCP specification (protocol version `"2024-11-05"`), indicating it
  is designed to replace or supplement the existing ACP transport layer
  in `internal/acp`.
- `CreateMessage` is unimplemented on `LocalProvider` -- this method
  is only meaningful for remote providers that can invoke an LLM.
- The `SimpleMessage` bridge type at `acp.go:308-317` includes fields
  (`ID`, `From`, `To`, `Subject`, `Priority`, `Type`) that overlap
  with Gas Town's mail system, suggesting this is the glue layer for
  translating mail-like messages into ACP-compatible `Message` objects.
- `InputSchema.Additional` is a catch-all map for non-standard JSON
  Schema properties. The custom marshal/unmarshal logic ensures these
  survive a round-trip without loss -- important for tool schemas that
  include vendor-specific extensions.
