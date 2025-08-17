## MCP Client Primitives Support Specification

**Audience:** engineers implementing or extending an MCP client

**Goal:** add complete support for all MCP feature primitives reflected in the client matrix on the MCP clients page: **Resources**, **Prompts**, **Tools**, **Discovery**, **Sampling**, **Roots**, **Elicitation**. The client must interoperate with MCP servers across transports and protocol versions using capability negotiation and JSON‑RPC 2.0.

References:
- Clients matrix: [modelcontextprotocol.io/clients](https://modelcontextprotocol.io/clients)
- About and overview: [modelcontextprotocol.io](https://modelcontextprotocol.io)
- LLMs page (index surface listing primitives): [modelcontextprotocol.io/llms-full.txt](https://modelcontextprotocol.io/llms-full.txt)
- Specification (source of truth; use latest date version): [spec.modelcontextprotocol.io](https://spec.modelcontextprotocol.io)

### Scope and non-goals
- In scope: wire protocol lifecycle, capability negotiation, client feature handlers, server feature consumers, user-consent UX hooks, configuration, logging, and test strategy.
- Out of scope: authoring new servers; model provider integrations beyond an abstraction required for Sampling.

---

## 1) Architecture and Lifecycle

- **Transport abstraction**: support at least stdio process, WebSocket, and HTTP/SSE where feasible. Transports must provide a bidirectional JSON-RPC 2.0 channel.
- **JSON-RPC 2.0**: implement request, response, and notification handling, unique IDs, error envelopes, and cancellation.
- **Version negotiation**: on `initialize`, offer supported protocol date(s); select the highest common. Persist chosen version per session.
- **Capability negotiation**:
  - Client advertises: `sampling`, `roots` (listChanged), and any client notifications it can handle (logging, progress, etc.).
  - Server advertises: `resources` (subscribe/listChanged), `prompts` (listChanged), `tools` (listChanged), and optional logging.
- **Lifecycle sequencing**:
  1) Client sends `initialize` (protocolVersion, capabilities, clientInfo).
  2) Server responds (protocolVersion, capabilities, serverInfo).
  3) Client sends `initialized` notification; normal traffic may begin.
  4) Shutdown: client requests orderly shutdown; close transport after acknowledgments/timeouts.

Resilience requirements:
- Exponential backoff reconnect for non-stdio transports; session re-init on reconnect.
- Idempotent request dispatch; deduplicate notifications by `(method, params, sessionId, seq)` if provided.
- Bounded queues and timeouts per request; surface actionable errors with remediation hints.

---

## 2) Feature Primitives

Below, each primitive lists responsibilities, API surfaces, state model, user-consent requirements, and example flows. Method names follow the MCP spec; consult the latest schema for exact names/shapes.

### 2.1 Resources (server feature consumed by client)

What it is: server-exposed context items identified by URIs (text or binary) that the client can discover, read, and subscribe to updates.

- **Client responsibilities**
  - Read-only listing of available resources and/or resource templates.
  - Fetch resource content for inclusion in LLM context or display.
  - Subscribe/unsubscribe to resource updates when server supports it.
  - Handle `resources` list-changed and per-resource update notifications.
  - Respect security boundaries (see Roots) when resolving file:// URIs.

- **Typical RPC interactions**
  - List resources: request current catalog with descriptions and URIs.
  - Read resource: request content by URI; support text, JSON, and binary (base64) payloads.
  - Subscribe/unsubscribe (optional): opt in to change notifications.
  - Notifications: handle list changes and resource updates to invalidate caches.

- **State and UX**
  - Maintain a resource index per server with freshness timestamps.
  - Provide UI to preview, select, and pin resources for chat/session use.
  - Enforce max-size for auto-includes; stream or chunk large resources.

- **Security**
  - Never read or transmit local files unless within an approved Root and the user explicitly selects or authorizes inclusion.

### 2.2 Prompts (server feature consumed by client)

What it is: server-provided prompt templates or workflows to seed/model conversations or tools.

- **Client responsibilities**
  - List available prompts with metadata; fetch the selected prompt/template.
  - Render input parameters (if any) to the user, validate against schema, and materialize the final prompt.
  - Watch for list-changed notifications to refresh prompt catalogs.

- **Typical RPC interactions**
  - List prompts: get catalog (id, name, description, input schema).
  - Get/apply prompt: resolve template into concrete content given user inputs.

- **UX**
  - Parameter form auto-generation from JSON Schema.
  - Preview final prompt text before use in sampling or tool calls.

### 2.3 Tools (server feature consumed by client)

What it is: callable functions/actions exposed by servers with typed arguments and structured results.

- **Client responsibilities**
  - Discover tool list and watch for changes.
  - Invoke tools with validated arguments; display progress and stream outputs if supported.
  - Surface errors with full error.data details; preserve call trace for debugging.

- **Typical RPC interactions**
  - List tools: get name, description, argument schema, result schema, and safety annotations.
  - Call tool: send `name` and `arguments` (validated locally first); handle success or error.

- **Consent & safety**
  - Gate each call behind explicit user consent unless pre-approved by policy.
  - Show concise risk summary derived from tool metadata (e.g., file write, network access).

### 2.4 Discovery (host/client capability)

What it is: mechanisms to locate and configure MCP servers without manual wiring.

- **Client responsibilities**
  - Support multiple discovery sources: project config (e.g., `.mcp.json`), workspace settings, OS-level registries, PATH executables, and well-known ports/URLs.
  - Implement pluggable resolvers for stdio process launch, WebSocket URLs, and HTTP/SSE endpoints.
  - Validate discovered servers with allowlists/denylists and present a clear consent screen before first connect.

- **Caching & health**
  - Cache discovered entries per workspace with last-seen and health state.
  - Provide a one-click re-scan and per-entry connect/test buttons.

### 2.5 Sampling (client feature provided to server)

What it is: the server asks the client to perform LLM sampling using the user’s configured model stack, under user control.

- **Client responsibilities**
  - Implement the sampling request handler that receives messages, model preferences/hints, tool availability, and token/stop settings.
  - Route to the user’s selected model/provider; support streaming deltas and cancellation.
  - Enforce consent: show the exact prompt and filters; allow redaction/edits before sending to the LLM; require approval to return outputs.
  - Optionally support tool-use during sampling when using an agentic model; confine tool access to those explicitly enabled.

- **UX & safety**
  - Prompt review screen: system + user content, model, max tokens, temperature.
  - Output review: redact or decline before returning to server.
  - Audit log of prompts/outputs tied to sampling request IDs.

### 2.6 Roots (client feature provided to server)

What it is: file-system boundaries that define where servers may operate.

- **Client responsibilities**
  - Maintain a list of `file://` root URIs for the current workspace(s), each with a friendly name.
  - Send the current roots on initialization and emit list-changed notifications when roots change.
  - Enforce that client-initiated file reads are constrained to these roots unless user grants temporary overrides.

- **UX & policy**
  - Roots management UI with add/remove and scope (workspace, project, folder).
  - Per-server overrides with clear warnings when expanding scope.

### 2.7 Elicitation (client feature provided to server)

What it is: structured requests for user input mid-workflow, defined by a message and an input schema.

- **Client responsibilities**
  - Render elicitation prompts with auto-generated forms from JSON Schema (types, enums, defaults, required fields).
  - Validate user input and return structured responses; support cancellation/decline with reasons.
  - Persist recent answers per server to enable quick reuse where appropriate.

- **Accessibility & UX**
  - Keyboard-only and screen-reader friendly forms; show concise context of why input is requested.

---

## 3) Cross-cutting Concerns

### 3.1 Security, Privacy, and Trust
- **Explicit consent** for: tool calls, resource access beyond auto-includes, sampling prompts, and root expansions.
- **Data minimization**: redact secrets in prompts; hash or mask file paths when feasible.
- **Policy engine**: per-server and per-tool rules (allow, deny, prompt) with scope (once, session, always) and audit trails.
- **Sandboxing**: isolate servers (process/user); no implicit credential sharing; environment var allowlist.

### 3.2 Reliability
- Request timeouts and retry policy; idempotency keys for tool calls where supported.
- Backpressure on streaming; enforce max concurrent in-flight requests per server.
- Comprehensive error mapping from JSON-RPC error codes to user-facing messages.

### 3.3 Performance
- Lazy discovery and lazy listing; cache catalogs with TTLs and etags/versions.
- Chunked/streaming reads for large resources; incremental rendering of outputs.

### 3.4 Telemetry & Logging
- Structured logs for lifecycle, RPC traffic (sizes, timings), errors, and consent decisions.
- Optional debug mode to mirror raw JSON-RPC (with redactions) for support cases.

### 3.5 Configuration
- Global settings: default model, consent defaults, max token limits, max resource size.
- Workspace overrides: roots, allowed servers/tools, discovery resolvers.

---

## 4) API Surface Summary (high-level)

Note: use the official spec/SDK types for exact names and payloads; below lists common patterns to implement.

- Lifecycle
  - Request: initialize → Response; Notification: initialized
  - Health/ping (optional): ping/pong or transport keepalive
  - Shutdown/close workflows

- Resources (client → server)
  - List available resources (with optional cursors/pagination)
  - Read resource by URI (support text/binary, media type)
  - Subscribe/unsubscribe to updates (if advertised)
  - Notifications handled: resources list changed; resource updated

- Prompts (client → server)
  - List prompts
  - Resolve/apply prompt with parameters → concrete content
  - Notifications handled: prompts list changed

- Tools (client → server)
  - List tools (+ schemas)
  - Call tool with validated args → result or error
  - Notifications handled: tools list changed

- Sampling (server → client)
  - Create sampling request with messages and model preferences
  - Stream deltas and support cancellation
  - Return final message(s) and usage metadata

- Roots (client → server)
  - Provide current roots list
  - Notification: roots list changed on updates

- Elicitation (server → client)
  - Request input with message + JSON Schema
  - Return validated structured input or decline

---

## 5) UX Requirements (consent-first)
- One-screen consent for first-time server connection describing capabilities and risks.
- Per-action consent (tools, sampling) with remember choices per policy.
- Resource picker with previews and size indicators.
- Prompt parameter forms; preview of materialized prompts before use.
- Elicitation forms with context and validation errors inline.

---

## 6) Testing & Compliance
- **Protocol conformance**: round-trip tests for each method; golden fixtures per protocol version.
- **Interoperability**: run against a matrix of popular servers; include discovery-only and tools-only servers.
- **Security tests**: roots escaping attempts, oversized payloads, prompt redaction, consent bypass attempts.
- **Load tests**: concurrent tool calls, large resource reads, long-running sampling streams.
- **Chaos**: dropped notifications, duplicate IDs, delayed responses.

---

## 7) Implementation Plan (milestones)
1) Core runtime: transport(s), JSON-RPC, lifecycle, error model, logging.
2) Server features: Resources, Prompts, Tools (list/read/call + notifications).
3) Client features: Roots (list + listChanged), Sampling handler (stream + cancel), Elicitation UI flow.
4) Discovery resolvers and consent UIs.
5) Security hardening: policy engine, sandbox options.
6) Interop test suite and docs.

---

## 8) Example JSON-RPC Exchanges (illustrative)

Initialization (client → server, then server → client, then client notification):

```json
{ "jsonrpc": "2.0", "id": 1, "method": "initialize", "params": {
  "protocolVersion": "latest",
  "capabilities": { "sampling": {}, "roots": { "listChanged": true } },
  "clientInfo": { "name": "OurMCPClient", "version": "X.Y.Z" }
}}
```
```json
{ "jsonrpc": "2.0", "id": 1, "result": {
  "protocolVersion": "latest",
  "capabilities": {
    "resources": { "subscribe": true, "listChanged": true },
    "prompts": { "listChanged": true },
    "tools": { "listChanged": true }
  },
  "serverInfo": { "name": "ExampleServer", "version": "A.B.C" }
}}
```
```json
{ "jsonrpc": "2.0", "method": "notifications/initialized" }
```

Tools: list and call (client → server):

```json
{ "jsonrpc": "2.0", "id": 2, "method": "tools/list" }
```
```json
{ "jsonrpc": "2.0", "id": 3, "method": "tools/call", "params": {
  "name": "search_issues",
  "arguments": { "repo": "org/repo", "query": "label:bug is:open" }
}}
```

Resources: list and read (client → server):

```json
{ "jsonrpc": "2.0", "id": 4, "method": "resources/list" }
```
```json
{ "jsonrpc": "2.0", "id": 5, "method": "resources/read", "params": {
  "uri": "file:///workspace/README.md"
}}
```

Prompts: list and apply (client → server):

```json
{ "jsonrpc": "2.0", "id": 6, "method": "prompts/list" }
```
```json
{ "jsonrpc": "2.0", "id": 7, "method": "prompts/apply", "params": {
  "id": "bug_report",
  "input": { "severity": "high", "repro": true }
}}
```

Sampling request (server → client):

```json
{ "jsonrpc": "2.0", "id": 8, "method": "sampling/create", "params": {
  "messages": [ { "role": "user", "content": "Summarize the attached docs" } ],
  "modelPreferences": { "hints": [{ "name": "preferred-model" }] },
  "maxTokens": 800
}}
```

Roots update (client notification → server):

```json
{ "jsonrpc": "2.0", "method": "roots/list_changed", "params": {
  "roots": [ { "uri": "file:///workspace", "name": "Workspace" } ]
}}
```

Elicitation (server → client) and response:

```json
{ "jsonrpc": "2.0", "id": 9, "method": "elicitation/requestInput", "params": {
  "message": "Please confirm deletion of 3 files",
  "schema": { "type": "object", "properties": { "confirm": { "type": "boolean" } }, "required": ["confirm"] }
}}
```
```json
{ "jsonrpc": "2.0", "id": 9, "result": { "confirm": true } }
```

Note: verify exact method names and payloads against the current spec/SDK before release.

---

## 9) Deliverables and Acceptance
- All primitives implemented and toggled on in capability negotiation.
- Interop tested against a representative set of servers listed on the clients page.
- Consent-first UX flows for tools, sampling, resource inclusion, and roots changes.
- Documentation for configuration, discovery, and security posture.
- Telemetry and diagnostics sufficient for support.

---

## 10) Future-proofing
- Track date-based protocol releases; ship compatibility shims where feasible.
- Centralize schema-derived types to avoid drift; prefer the official SDK where possible.
- Feature flags for experimental primitives and transports.

