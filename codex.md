# Codex Source Code Analysis

## Abstraction Layer Design

### Layer Overview

```
┌─────────────────────────────────────────┐
│           Layer 1: Frontend             │
│  TUI (tui/)   ←→   CLI (cli/)           │
└───────────────────┬─────────────────────┘
                    │ JSON-RPC / App Server Protocol
┌───────────────────▼─────────────────────┐
│        Layer 2: App Server Bridge       │
│  app-server / app-server-protocol       │
└───────────────────┬─────────────────────┘
                    │ Op / Event (SQ/EQ pattern)
┌───────────────────▼─────────────────────┐
│        Layer 3: Session / Thread        │
│  core: Codex  ←→  CodexThread           │
└──────┬──────────┬──────────┬────────────┘
       │          │          │
┌──────▼───┐ ┌───▼────┐ ┌───▼────────────┐
│ Layer 4a │ │  4b    │ │      4c        │
│ Model    │ │ Tools  │ │  Extensions    │
│ Provider │ │        │ │  (ext/*)       │
└──────────┘ └────────┘ └────────────────┘
```

### Layer 1 — Frontend

Two frontends, both consume the same layer below:
- **`tui/src/main.rs`** — the interactive terminal UI (ratatui-based)
- **`cli/src/main.rs`** — non-interactive CLI (`codex login`, `codex mcp`, etc.)

They talk to the agent via the App Server Protocol (JSON-RPC over a Unix socket or in-process channel).

### Layer 2 — App Server Bridge

**Crates:** `app-server`, `app-server-protocol`, `app-server-transport`

Translates between client JSON-RPC messages and the agent's internal `Op`/`Event` types. This boundary allows any frontend (TUI, IDE plugins, web app) to drive the same agent.

### Layer 3 — Session / Thread (core design)

**Crates:** `core`, `protocol`

The heart: a **SQ/EQ (Submission Queue / Event Queue) pattern** — stated explicitly in `protocol/src/protocol.rs:1`.

Key types in `protocol/src/protocol.rs`:

| Type | Line | Role |
|---|---|---|
| `Op` | 447 | Commands sent **into** the agent (user input, exec approval, interrupt…) |
| `EventMsg` | 1133 | Events flowing **out** (turn started, agent message, exec begin/end…) |
| `Submission` | 128 | Wraps an `Op` with a correlation ID |

**`Codex` struct** (`core/src/session/mod.rs:372`) holds:
- `tx_sub: Sender<Submission>` — the input queue
- `rx_event: Receiver<Event>` — the output queue

**`CodexThread`** (`core/src/codex_thread.rs:145`) wraps `Codex` and is the handle given to the app server. Its public API is essentially `.submit(op)` and `.wait_until_terminated()`.

This queue pair is the **central abstraction boundary** — everything above only sees `Op`/`Event`, everything below only sees internal session state.

### Layer 4a — Model Provider

**Crates:** `model-provider`, `model-provider-info`

`ModelProvider` trait (`model-provider/src/provider.rs:83`) abstracts over AI backends:

```rust
pub trait ModelProvider: fmt::Debug + Send + Sync {
    fn info(&self) -> &ModelProviderInfo;
    fn capabilities(&self) -> ProviderCapabilities;
    // ...
}
```

Concrete implementations: OpenAI Responses API, AWS Bedrock, Ollama, LM Studio. Core never calls HTTP directly — always through this trait.

### Layer 4b — Tool System

**Crates:** `tools`, `ext/extension-api`

`ToolExecutor` trait (`tools/src/tool_executor.rs:44`) is the interface every tool implements:

```rust
pub trait ToolExecutor<Invocation>: Send + Sync {
    fn tool_name(&self) -> ToolName;
    fn spec(&self) -> ToolSpec;         // what the model sees (serialized JSON)
    fn exposure(&self) -> ToolExposure; // Direct / Deferred / Hidden
    async fn handle(&self, invocation: Invocation)
        -> Result<Box<dyn ToolOutput>, FunctionCallError>;
}
```

`ToolSpec` (`tools/src/tool_spec.rs`) serializes into the OpenAI Responses API tool list. Tools can be `Direct` (always visible), `Deferred` (discoverable via `tool_search`), or `Hidden`.

### Layer 4c — Extension System

**Crates:** `ext/extension-api`, all `ext/*` crates

Extensions implement one or more **contributor traits** and register them in `ExtensionRegistry` (`ext/extension-api/src/registry.rs:143`). Core iterates the registry at lifecycle points — it has zero direct dependency on any extension.

| Contributor Trait | When called |
|---|---|
| `ContextContributor` | Before each turn — injects extra prompt fragments |
| `TurnInputContributor` | Before each turn — adds model-visible input items |
| `TurnLifecycleContributor` | On turn start/stop/abort/error |
| `ThreadLifecycleContributor` | On thread start/resume/idle/stop |
| `ToolContributor` | Registers additional tools |
| `ToolLifecycleContributor` | Before/after each tool call |
| `ApprovalReviewContributor` | Reviews tool calls (e.g. the `guardian` ext) |
| `ConfigContributor` | Notified when thread config changes |
| `TokenUsageContributor` | Observes token usage per turn |

### Design Principles

1. **Queue-pair isolation** — `Op`/`Event` is the single boundary between frontend and agent internals. Neither side calls into the other directly.
2. **Contributor pattern over inheritance** — Extensions register typed callbacks. No `dyn Extension` god trait.
3. **`ToolExecutor` decouples spec from runtime** — The model sees `ToolSpec` (JSON). The executor handles invocations. Bound together by the trait.
4. **`ModelProvider` is a seam** — Swapping backends is a trait swap, not `if provider == "..."` branches.
5. **`protocol` crate is dependency-free** — Only contains types. `core`, `tui`, and `app-server` all depend on it; it depends on nothing in the workspace — the stable shared vocabulary.

---


## Overview of all components of codex

---

## Core Agent

| Crate | Description |
|---|---|
| `core` | Heart of the agent — session, exec policy, MCP, shell, sandboxing, tools |
| `core-api` | Public API types for `core` |
| `core-plugins` | Plugin integration layer for core |
| `core-skills` | Built-in skills wired into the core agent |

---

## User Interface

| Crate | Description |
|---|---|
| `tui` | Full terminal UI — chat, markdown rendering, key bindings, streaming |
| `cli` | CLI entry point — parses `codex chat`, `codex login`, `codex mcp`, etc. |

---

## App Server (TUI ↔ Agent bridge)

| Crate | Description |
|---|---|
| `app-server` | Server that bridges TUI frontend and agent backend |
| `app-server-protocol` | Message types for app server communication |
| `app-server-transport` | Transport layer (how messages are sent) |
| `app-server-daemon` | Runs app-server as a background daemon |
| `app-server-client` | Client to connect to the app server |
| `app-server-test-client` | Test helpers for app server |

---

## Execution & Sandboxing

| Crate | Description |
|---|---|
| `exec` | Runs shell commands for the agent |
| `exec-server` | Daemon that handles exec requests in sandboxed mode |
| `execpolicy` | Decides whether a command is allowed to run |
| `execpolicy-legacy` | Old exec policy logic (kept for compatibility) |
| `sandboxing` | Sandbox orchestration logic |
| `linux-sandbox` | Linux-specific sandbox using namespaces/seccomp |
| `bwrap` | Wraps commands with `bubblewrap` (unprivileged containers) |
| `process-hardening` | Applies seccomp/landlock restrictions to processes |
| `shell-command` | Represents a parsed shell command |
| `shell-escalation` | Handles privilege escalation scenarios |

---

## Model & Backend

| Crate | Description |
|---|---|
| `model-provider` | Abstracts over different model providers (OpenAI, Ollama, etc.) |
| `model-provider-info` | Metadata about models (names, capabilities, limits) |
| `models-manager` | Downloads/manages model files |
| `backend-client` | HTTP client for talking to the AI backend |
| `codex-api` | Codex-specific API types |
| `codex-backend-openapi-models` | Generated OpenAPI model types for the backend |
| `codex-client` | High-level client to drive a Codex session |
| `responses-api-proxy` | Proxies requests to the OpenAI Responses API |
| `ollama` | Ollama model provider integration |
| `lmstudio` | LM Studio model provider integration |
| `aws-auth` | AWS Bedrock authentication support |
| `realtime-webrtc` | WebRTC transport for realtime voice/streaming |

---

## MCP (Model Context Protocol)

| Crate | Description |
|---|---|
| `codex-mcp` | Core MCP integration |
| `mcp-server` | Exposes Codex as an MCP server (for IDE plugins etc.) |
| `rmcp-client` | Client for connecting to remote MCP servers |

---

## Memory & State

| Crate | Description |
|---|---|
| `memories/read` | Reads agent memories from disk |
| `memories/write` | Writes agent memories to disk |
| `state` | Persistent app state (SQLite via sqlx) |
| `thread-store` | Stores conversation threads |
| `keyring-store` | Secure credential storage |
| `agent-graph-store` | Stores agent execution graphs |

---

## Extensions (`ext/`)

| Crate | Description |
|---|---|
| `ext/extension-api` | API surface all extensions must implement |
| `ext/goal` | "Goal" extension — sets and tracks session goals |
| `ext/guardian` | Safety guardian — reviews and blocks unsafe actions |
| `ext/image-generation` | Image generation tool extension |
| `ext/memories` | Memory extension (reads/writes memories as a tool) |
| `ext/skills` | Extension that exposes skills as tools |
| `ext/web-search` | Web search extension |

---

## Config & Auth

| Crate | Description |
|---|---|
| `config` | Reads/writes `~/.codex/config.toml` |
| `cloud-config` | Cloud-specific configuration |
| `login` | Handles `codex login` flow |
| `secrets` | Manages API keys and secrets |
| `install-context` | Detects how/where Codex was installed |
| `features` | Feature flags and rollout gating |
| `rollout` | Rollout logic (gradual feature enablement) |
| `rollout-trace` | Tracing/logging for rollout decisions |

---

## Skills & Hooks

| Crate | Description |
|---|---|
| `skills` | Slash-command / skill definitions |
| `core-skills` | Skills built into the core agent |
| `hooks` | Executes user-configured hooks |
| `plugin` | Plugin loading/management |
| `core-plugins` | Plugins bundled with core |

---

## Tools

| Crate | Description |
|---|---|
| `tools` | Tool definitions (file edit, shell, etc.) used by the agent |
| `apply-patch` | Applies unified diff patches to files |
| `file-search` | File search tool (BM25 + fuzzy) |
| `file-system` | File read/write tool implementation |
| `file-watcher` | Watches filesystem for changes |
| `git-utils` | Git helpers (diff, status, etc.) |
| `context-fragments` | Chunks of context injected into prompts |
| `prompts` | System prompt construction |

---

## Auth & Cloud

| Crate | Description |
|---|---|
| `analytics` | Usage analytics/telemetry |
| `otel` | OpenTelemetry tracing setup |
| `feedback` | Sends user feedback |
| `cloud-tasks` | Cloud task queue integration |
| `cloud-tasks-client` | Client for cloud tasks |
| `cloud-tasks-mock-client` | Mock client for testing |
| `connectors` | Integrations with external services |
| `external-agent-sessions` | Manages sessions with external agents |
| `external-agent-migration` | Migrates external agent configs |
| `agent-identity` | Agent identity/signing |
| `collaboration-mode-templates` | Templates for collaborative agent modes |
| `network-proxy` | HTTP proxy support |

---

## Infrastructure & Utils

| Crate | Description |
|---|---|
| `protocol` | Core message/event types shared across crates |
| `async-utils` | Async helpers |
| `uds` | Unix domain socket utilities |
| `stdio-to-uds` | Bridges stdio to a Unix socket |
| `terminal-detection` | Detects terminal capabilities |
| `ansi-escape` | ANSI escape code parsing/generation |
| `arg0` | Manages `argv[0]` (for multi-call binaries) |
| `code-mode` | "Code mode" behavior logic |
| `response-debug-context` | Debug info attached to API responses |
| `message-history` | Conversation message history management |
| `context-fragments` | Reusable prompt fragments |

---

## Utils (`utils/`)

| Crate | Description |
|---|---|
| `utils/absolute-path` | Resolves absolute paths |
| `utils/approval-presets` | Preset approval rules for tools |
| `utils/cache` | Generic caching primitives |
| `utils/cargo-bin` | Locates Cargo-built binaries |
| `utils/cli` | CLI display helpers |
| `utils/elapsed` | Human-readable elapsed time |
| `utils/fuzzy-match` | Fuzzy string matching |
| `utils/home-dir` | Cross-platform home directory |
| `utils/image` | Image processing helpers |
| `utils/json-to-toml` | Converts JSON to TOML |
| `utils/oss` | Open-source license utilities |
| `utils/output-truncation` | Truncates long tool output |
| `utils/path-utils` | Path manipulation helpers |
| `utils/plugins` | Plugin loading helpers |
| `utils/pty` | PTY (pseudo-terminal) helpers |
| `utils/readiness` | Health/readiness checks |
| `utils/rustls-provider` | TLS provider setup |
| `utils/sandbox-summary` | Summarizes sandbox state |
| `utils/sleep-inhibitor` | Prevents system sleep during runs |
| `utils/stream-parser` | Parses streaming API responses |
| `utils/string` | String manipulation helpers |
| `utils/template` | Text template rendering |

---

## Experimental / Other

| Crate | Description |
|---|---|
| `v8-poc` | Proof-of-concept V8 JS runtime integration |
| `codex-experimental-api-macros` | Procedural macros for experimental APIs |
| `thread-manager-sample` | Sample/demo for the thread manager |
| `test-binary-support` | Helpers for test binaries |


## Layer 2

### Data input from UI (Layer 1) to App Server (Layer 2)

The protocol is **JSON-RPC over WebSocket or Unix socket**. Every message is either a **Request** (expects a response) or a **Notification** (fire-and-forget).

#### The two core requests

**`thread/start`** — open a new conversation (`app-server-protocol/src/protocol/v2/thread.rs:95`)

```
ThreadStartParams {
  model, model_provider    // which AI model to use
  cwd                      // working directory
  approval_policy          // how to approve tool calls (auto/manual)
  sandbox                  // sandbox mode
  base_instructions        // system prompt override
  developer_instructions
  personality              // agent persona
  ...
}
```

**`turn/start`** — send a user message and trigger an agent turn (`app-server-protocol/src/protocol/v2/turn.rs:66`)

```
TurnStartParams {
  thread_id            // which conversation this belongs to
  input: Vec<UserInput>  // the actual content (see below)
  model                // optional per-turn override
  cwd                  // optional working directory override
  approval_policy
  sandbox_policy
  ...
}
```

#### `UserInput` — what a message can contain (`turn.rs:270`)

```
UserInput =
  | Text { text: String, text_elements }   // plain text message
  | Image { url }                          // image from URL
  | LocalImage { path }                    // image from local file
  | Skill { name, path }                   // a /slash-command skill
  | Mention { name, path }                 // an @mention of a file/symbol
```

A single turn's `input` is a `Vec<UserInput>` — so one message can mix text, images, and file mentions.

#### All protocol messages — defined in `app-server-protocol/src/protocol/common.rs`

Four macro invocations register every message type in one file:

| Macro | Line | Direction | Description |
|---|---|---|---|
| `client_request_definitions!` | 450 | UI → App Server | All requests the UI sends (expects a response) |
| `server_request_definitions!` | 1371 | App Server → UI | Requests backend sends to UI (UI must respond) |
| `server_notification_definitions!` | 1519 | App Server → UI | Fire-and-forget events pushed to UI |
| `client_notification_definitions!` | 1615 | UI → App Server | Fire-and-forget from UI (only: `Initialized`) |

---

##### `client_request_definitions!` — UI → App Server (line 450)

| Group | Methods |
|---|---|
| Thread lifecycle | `thread/start`, `thread/resume`, `thread/fork`, `thread/archive`, `thread/unarchive`, `thread/unsubscribe`, `thread/name/set`, `thread/compact/start`, `thread/rollback`, `thread/list`, `thread/search`, `thread/read`, `thread/inject_items` |
| Turn | `turn/start`, `turn/steer`, `turn/interrupt` |
| Thread settings | `thread/settings/update`, `thread/goal/set/get/clear`, `thread/metadata/update`, `thread/memoryMode/set` |
| Shell / command | `thread/shellCommand`, `command/exec`, `command/exec/write`, `command/exec/terminate`, `command/exec/resize` |
| Process (experimental) | `process/spawn`, `process/writeStdin`, `process/kill`, `process/resizePty` |
| File system | `fs/readFile`, `fs/writeFile`, `fs/createDirectory`, `fs/getMetadata`, `fs/readDirectory`, `fs/remove`, `fs/copy`, `fs/watch`, `fs/unwatch` |
| MCP | `mcpServer/oauth/login`, `config/mcpServer/reload`, `mcpServerStatus/list`, `mcpServer/resource/read`, `mcpServer/tool/call` |
| Config | `config/read`, `config/value/write`, `config/batchWrite`, `configRequirements/read` |
| Skills / Hooks | `skills/list`, `skills/extraRoots/set`, `skills/config/write`, `hooks/list` |
| Plugins / Marketplace | `plugin/list/install/uninstall/read`, `marketplace/add/remove/upgrade`, `plugin/share/*` |
| Account / Auth | `account/login/start`, `account/login/cancel`, `account/logout`, `account/read`, `account/rateLimits/read`, `account/usage/read` |
| Models | `model/list`, `modelProvider/capabilities/read` |
| Remote control | `remoteControl/enable/disable/status/read`, `remoteControl/pairing/*`, `remoteControl/client/*` |
| Realtime (voice) | `thread/realtime/start/stop/appendAudio/appendText/listVoices` |
| Other | `review/start`, `feedback/upload`, `fuzzyFileSearch`, `memory/reset`, `environment/add` |

---

##### `server_request_definitions!` — App Server → UI (line 1371)

Requests the backend sends to the UI that require a response (approval dialogs, auth):

| Method | Purpose |
|---|---|
| `item/commandExecution/requestApproval` | Ask user to approve a shell command |
| `item/fileChange/requestApproval` | Ask user to approve a file change |
| `item/tool/requestUserInput` | Ask user for input for a tool call |
| `mcpServer/elicitation/request` | Ask user for MCP elicitation input |
| `item/permissions/requestApproval` | Ask user to approve additional permissions |
| `item/tool/call` | Execute a dynamic tool on the client side |
| `account/chatgptAuthTokens/refresh` | Ask UI to refresh auth tokens |
| `attestation/generate` | Request a fresh attestation from the client |

---

##### `server_notification_definitions!` — App Server → UI (line 1519)

Fire-and-forget events pushed to the UI (streaming output, status changes):

| Group | Examples |
|---|---|
| Turn lifecycle | `turn/started`, `turn/completed`, `turn/diff/updated`, `turn/plan/updated` |
| Thread lifecycle | `thread/started`, `thread/status/changed`, `thread/closed`, `thread/archived` |
| Streaming content | `item/agentMessage/delta`, `item/reasoning/textDelta`, `item/reasoning/summaryTextDelta` |
| Tool execution | `item/started`, `item/completed`, `item/commandExecution/outputDelta`, `item/commandExecution/terminalInteraction` |
| File changes | `item/fileChange/outputDelta`, `item/fileChange/patchUpdated`, `fs/changed` |
| MCP | `item/mcpToolCall/progress`, `mcpServer/oauthLogin/completed`, `mcpServer/startupStatus/updated` |
| Account | `account/updated`, `account/rateLimits/updated`, `account/login/completed` |
| Realtime / voice | `thread/realtime/started`, `thread/realtime/transcript/delta`, `thread/realtime/outputAudio/delta` |
| Misc | `error`, `warning`, `configWarning`, `model/rerouted`, `thread/compacted` |

---

#### Data flow summary

```
User types in TUI
      ↓
JSON-RPC over Unix socket:
  { "method": "turn/start", "params": { "thread_id": "...", "input": [{"type":"text","text":"..."}] } }
      ↓
app-server MessageProcessor → TurnRequestProcessor
      ↓
codex-core executes the agent turn
```


### Full lifecycle: startup → request → shutdown

**Phase 1 — Startup** (`app-server/src/lib.rs:420` — `run_main_with_transport_options`)

**Step 1 — Config loading** (`lib.rs:439–509`)

```
cli_config_overrides.parse_overrides()       ← CLI -c key=val flags
ConfigManager::new(codex_home, overrides)    ← creates the config loader (no IO yet)
config_manager.load_latest_config()          ← first load: to discover cloud config loaders
  → AuthManager::shared_from_config()        ← sets up auth from the config
  → config_manager.replace_cloud_config_bundle_loader()
config_manager.load_latest_config()          ← second load: full config with cloud layers merged
```

If config fails and `--strict-config` is set → hard error. Otherwise falls back to defaults.

**Step 2 — SQLite state DB** (`lib.rs:534`)

```
rollout_state_db::try_init(&config)
  → opens (or creates) ~/.codex/state.db
  → used by: thread store, rollout flags, memory, log_db
```

**Step 3 — Personality migration** (`lib.rs:544`)

One-time migration: if the user's old config had an inline personality, moves it to the new format. Skipped if already migrated or no sessions exist.

**Step 4 — Exec policy + config warnings** (`lib.rs:584–615`)

Validates the exec policy rules from config. Any parsing errors become `ConfigWarningNotification` messages queued to be sent to the client after it connects.

**Step 5 — Tracing / logging setup** (`lib.rs:622–651`)

```
tracing_subscriber::registry()
  .with(stderr fmt layer)          ← RUST_LOG-controlled stderr output
  .with(feedback layer)            ← attaches logs to feedback reports
  .with(log_db layer)              ← writes TRACE+ logs to SQLite
  .with(otel logger/tracing layer) ← sends to OpenTelemetry if configured
```

**Step 6 — Async channels** (`lib.rs:431–435`)

```
transport_event_tx/rx   ← all inbound events (connections opened/closed, messages)
outgoing_tx/rx          ← all outbound messages (responses, notifications)
outbound_control_tx/rx  ← control events for managing outbound connection state
```

**Step 7 — Transport listeners** (`lib.rs:668–699`)

Exactly one transport is started based on the `--listen` flag:

```
Stdio       → start_stdio_connection()         single client, exits when stdin closes
UnixSocket  → start_control_socket_acceptor()  default for TUI + IDE plugins
WebSocket   → start_websocket_acceptor()       remote clients, requires auth if non-loopback
Off         → no listener (only valid with remote control)
```

Plus always: `start_remote_control()` — starts the cloud relay connection if `--remote-control` is set.

**Step 8 — MessageProcessor construction** (`lib.rs:810`, `message_processor.rs:281`)

This is the heaviest init step. Creates every request processor and wires them together:

```
ThreadManager::new()           ← core's thread registry (owns all live sessions)
  ├─ PluginsManager::new()     ← loads installed plugins
  ├─ McpManager::new()         ← manages MCP server connections
  ├─ SkillsManager::new()      ← loads /slash-command skills from disk
  └─ build_models_manager()    ← creates ModelProvider + ModelsManager
       └─ create_model_provider(config.model_provider)
            → OpenAI / Bedrock / Ollama / LMStudio provider

thread_store_from_config()     ← LocalThreadStore (disk) or InMemoryThreadStore
SkillsWatcher::new()           ← starts filesystem watcher for skills directory changes

InitializeRequestProcessor     ← handles the Initialize handshake
ThreadRequestProcessor         ← thread/start, thread/resume, thread/fork, …
TurnRequestProcessor           ← turn/start, turn/steer, turn/interrupt
ConfigRequestProcessor         ← config/read, config/value/write, …
FsRequestProcessor             ← fs/readFile, fs/writeFile, …
McpRequestProcessor            ← mcpServer/tool/call, mcpServer/resource/read, …
AccountRequestProcessor        ← account/login, account/read, …
CommandExecRequestProcessor    ← command/exec, command/exec/write, …
… (15+ processors total)

plugin_startup_tasks           ← if enabled, warms up plugin processes now
```

**Step 9 — Spawn event loop tasks** (`lib.rs:745–798`, `lib.rs:800–1070`)

Two Tokio tasks are spawned and run concurrently for the lifetime of the process. `tokio::spawn` schedules each task immediately and returns a `JoinHandle` — both tasks run in parallel, communicating only through async channels. `run_main_with_transport_options` `.await`s both handles at the end and does not return until both exit.

```
transport ──► transport_event_rx ──► processor_handle ──► processor.process_request()
                                            │                        │
                                            │                        └──► outgoing_tx
                                            │
                                            └──► outbound_control_tx
                                                          │
                                                 outbound_handle
                                                          │
                                             outgoing_rx ─┘
                                                          │
                                                          ▼
                                               route_outgoing_envelope() ──► client
```

**`outbound_handle`** (`lib.rs:745`) — connection registry + message delivery

Owns a `HashMap<ConnectionId, OutboundConnectionState>` and Listen two channels forever `outbound_control_rx` and `outgoing_rx`.

```rust
loop {
    tokio::select! {
        biased;
        event = outbound_control_rx.recv() => {
            // ...
        }
        envelope = outgoing_rx.recv() => {
            //  ...
        }
    }
}

```

**outbound_control_rx** — manages which connections exist                        
   
Sent by processor_handle when the transport reports a connection opened, closed, or when a graceful restart needs to kick everyone off. It's the lifecycle channel — "here's a new client", "that client left", "disconnect everyone".

**outgoing_rx** — carries messages to send to clients

Sent by MessageProcessor (and anywhere that holds an OutgoingMessageSender) whenever a response, notification, or event needs to go out. Every JSON-RPC response, streaming agent message, tool approval dialog, etc. flows through this channel.

The reason they're two separate channels instead of one: connection management must be processed before message delivery (biased; enforces this). If a connection closes and a message for it arrives simultaneously, you want to remove the dead connection first — otherwise you'd try to write to a closed writer.

```
outbound_control_rx:
  Opened        → insert connection into registry (stores writer + disconnect sender)
  Closed        → remove connection from registry
  DisconnectAll → kick all connections (graceful restart)

outgoing_rx:
  envelope → route_outgoing_envelope() → write to the correct client
```

No business logic — purely a router that delivers messages to the right connection.

**`processor_handle`** (`lib.rs:800`) — main event loop (the brain)

Owns a `HashMap<ConnectionId, ConnectionState>` and loops on a `tokio::select!` with 5 branches:

**Loop head** — checked every iteration before `select!`:
```
shutdown_state.update(running_turn_count, connections.len())
  → ShutdownAction::Finish: cancel transport token, send DisconnectAll, break
```

**transport_event channel**: `mpsc` (multi-producer, single-consumer). Each transport accept loop holds a cloned `transport_event_tx` and pushes `ConnectionOpened`, `ConnectionClosed`, and `IncomingMessage` events as raw I/O happens. `processor_handle` is the sole consumer on `transport_event_rx`. When all senders are dropped, `recv()` returns `None` and the loop breaks.

The UI (TUI/IDE/web) is the ultimate origin of these events, but it never touches the channel directly. The transport accept loop is the translator — it reads raw bytes from the socket, parses them into `JSONRPCMessage`, wraps that in a `TransportEvent`, and sends it into `transport_event_tx`. `processor_handle` never sees bytes, only typed events.

```
UI (TUI/IDE/web)
  │  bytes over socket/stdio/websocket
  ▼
transport accept loop  (start_stdio_connection, start_control_socket_acceptor, etc.)
  │  parses bytes → JSONRPCMessage, wraps in TransportEvent
  ▼
transport_event_tx.send(TransportEvent::IncomingMessage { ... })
  │
  ▼
processor_handle  (transport_event_rx.recv())
```

| Sender | Line |
|---|---|
| `start_stdio_connection()` | `lib.rs:682` |
| `start_control_socket_acceptor()` | `lib.rs:691` |
| `start_websocket_acceptor()` | `lib.rs:700` |
| `start_remote_control()` | `lib.rs:737` |


**Branch 1 — `transport_event_rx`** (main inbound path):
```
ConnectionOpened  → register in connections map
                    → send OutboundControlEvent::Opened to outbound_handle
ConnectionClosed  → remove from connections map
                    → send OutboundControlEvent::Closed to outbound_handle
                    → processor.connection_closed()
                    → if shutdown_when_no_connections && empty: break
IncomingMessage:
  Request      → processor.process_request()
                 if first request that caused initialization:
                   send_initialize_notifications_to_connection()
                   send RemoteControlStatusChanged notification
                   processor.connection_initialized()
                   set outbound_initialized = true
  Response     → processor.process_response()   (client replied to a server request, e.g. approval dialog)
  Notification → processor.process_notification()
  Error        → processor.process_error()
```

The first message from a new connection must be `Initialize` — all other requests are rejected until it arrives (`lib.rs:961`).

MessageProcessor (line 810, Arc::new(MessageProcessor::new(...))) is the central dispatcher — it owns all the *RequestProcessor structs and routes every incoming JSON-RPC request to the right one.

But it's worth being precise about the division of responsibility:

- processor_handle — decides when to call the processor (connection lifecycle, initialization gating, shutdown logic)
- MessageProcessor — decides what to do with a request (deserialize → dispatch → call the right sub-processor)

So processor_handle is the event loop shell, and MessageProcessor is the business logic core inside it. The handle calls processor.process_request() and processor.process_response() etc., but all the routing and handling lives inside MessageProcessor.


**Branch 2 — `shutdown_signal`** (graceful restart, only when enabled and not forced):
```
→ shutdown_state.on_signal(signal, connections.len(), running_turn_count)
  starts drain countdown
```

**Branch 3 — `running_turn_count_rx.changed()`** (only active during graceful drain):
```
→ re-evaluates shutdown_state at loop head on next iteration
```

**Branch 4 — `remote_control_status_rx.changed()`**:
```
→ broadcast RemoteControlStatusChanged to all connections
```

**Branch 5 — `thread_created_rx`**:
```
→ new CodexThread was created anywhere
→ try_attach_thread_listener(thread_id, initialized_connection_ids)
   so all initialized connections receive events from the new thread
```

**After loop exits** (`lib.rs:1067`):
```
if !forced:
  rpc_gate.shutdown()            — drain in-flight requests on all connections
  processor.drain_background_tasks()
  processor.shutdown_threads()   — stop all agent threads in core
```


**Phase 3 — Request routing** (`app-server/src/message_processor.rs`)
Once the `run_main_with_transport_options` sets up,
```rust
let _ = processor_handle.await;  // lib.rs:1083           
let _ = outbound_handle.await;   // lib.rs:1084                           
```

These two handle will wait for client connecting and sending messages.

The sequence of real activity after setup:

1. client connects 
    → transport accept loop sends ConnectionOpened 
    → processor_handle registers it

2. client sends Initialize
    → processor_handle → processor.process_request() → InitializeRequestProcessor
    → connection marked as initialized
    → server sends initialize notifications back

3. client sends ThreadStart
    → ThreadManager creates a CodexThread
    → agent session starts running

4. client sends TurnStart
    → TurnRequestProcessor submits Op::UserInput → CodexThread
    → agent starts processing, emits EventMsg back
    → app-server converts EventMsg → ServerNotification → outgoing_tx → outbound_handle → client

5. ... continues until client disconnects or shutdown signal

All calls the `processor.process_request()`

**Phase 4 — Request handle** (`app-server/src/message_processor.rs:528`)

**`process_request`** (`message_processor.rs:528`):
1. Build `RequestContext` (span, trace context, request ID)
2. Deserialize `JSONRPCRequest` → `ClientRequest` enum
3. Call `handle_client_request()` — send error back if it fails

**`handle_client_request`** (`message_processor.rs:756`):
- `ClientRequest::Initialize` → `initialize_processor.initialize()`, return early
- Everything else → `dispatch_initialized_client_request()`

**`dispatch_initialized_client_request`** (`message_processor.rs:801`):
- Reject if not yet initialized
- Reject if experimental API required but not enabled
- Check `serialization_scope()`:
  - Has scope (e.g. same `thread_id`) → enqueue in `request_serialization_queues` (serialized, no races)
  - No scope → `tokio::spawn` directly (runs concurrently)
- Either path calls `handle_initialized_client_request()`

**`handle_initialized_client_request`** (`message_processor.rs:864`) — the big `match`:

`handle_initialized_client_request` is the core of Layer 2 — it's the single point where every client request gets routed to the right sub-processor.

```
ClientRequest variant       → sub-processor
─────────────────────────────────────────────
Initialize                  → (panic, handled above)
ConfigRead/Write/…          → config_processor
FsReadFile/WriteFile/…      → fs_processor
ThreadStart/Resume/Fork/…   → thread_processor
TurnStart/Steer/Interrupt   → turn_processor
McpServer/…                 → mcp_processor
Account/…                   → account_processor
RemoteControl/…             → remote_control_processor
… (one arm per ClientRequest variant)
```

The key design: `dispatch_initialized_client_request` is where **concurrency is controlled**. Requests without a `serialization_scope` run in parallel in their own spawned tasks. Requests with a scope (e.g. two `TurnStart` on the same thread) are serialized through a queue so they never race each other.

**Minimum required request sequence to get the agent talking:**

```
1. Initialize    — must be first; everything else rejected until complete
                   negotiates capabilities, sets experimental API flags

2. ThreadStart   — creates a CodexThread (agent session)
   (or ThreadResume / ThreadFork to reopen an existing thread)

3. TurnStart     — submits user input; kicks the agent into action
```

**Optional requests valid at any point after Initialize:**

```
TurnSteer        — add more input mid-turn while agent is still running
TurnInterrupt    — interrupt the running turn
ConfigRead/Write — read or change config
FsReadFile/…     — filesystem operations
McpServer/…      — MCP tool calls
Account/…        — auth / login
```

#### ThreadStart request handle (`app_server/src/request_processors/thread_processor.rs`)

`thread_start` (line 367) → thin wrapper → **`thread_start_inner`** (line 802):
- Destructures `ThreadStartParams` (model, cwd, approval_policy, sandbox, instructions, etc.)
- Builds `ConfigOverrides` from params
- Spawns **`thread_start_task`** as a background task — returns `Ok(())` immediately without waiting

**`thread_start_task`** (line 943) — the real work, runs in background:

```
1. config_manager.load_with_overrides()
      → merge ThreadStartParams overrides into base Config

2. project trust resolution
      → if cwd is set and not yet trusted, persist trust level to disk, reload config

3. thread_manager.start_thread_with_options()
      → creates CodexThread  ← agent session comes alive here (crosses into Layer 3)

4. ensure_conversation_listener()
      → attaches event listener to the new thread for this connection
      → all EventMsg from core now flow back as ServerNotifications to the client

5. thread_watch_manager.upsert_thread_silently()
      → registers thread in the thread list (for thread/list queries)

6. outgoing.send_response(request_id, ThreadStartResponse)
      → sends JSON-RPC response back to client

7. outgoing.send_notification(ThreadStartedNotification)
      → broadcasts thread/started to all initialized connections
```

Step 3 is the boundary crossing — `thread_manager.start_thread_with_options()` is where `CodexThread` is created and Layer 3 begins running.

#### `start_thread_with_options` → `Codex::spawn` (`core/src/thread_manager.rs`)

`start_thread_with_options` (line 583) → thin wrapper → **`start_thread_with_options_and_fork_source`** (line 591) → **`spawn_thread_with_source`** (line 1255):

```
spawn_thread_with_source:

1. if InitialHistory::Resumed and thread already running → return existing thread early

2. resolve_environment_selections()
      → map environment names to actual env vars

3. Codex::spawn(CodexSpawnArgs { config, auth_manager, models_manager,
                                  mcp_manager, extensions, conversation_history, … })
      → creates the Codex session and its internal async task
      → returns CodexSpawnOk { codex, thread_id }
      ← THIS is where the agent's internal event loop starts running

4. finalize_thread_spawn(codex, thread_id)
      → codex.next_event().await  ← waits for the first EventMsg
      → must be EventMsg::SessionConfigured (or error)
      → CodexThread::new(codex, session_configured, …)  ← wraps Codex in CodexThread
      → inserts into threads HashMap
      → returns NewThread { thread_id, thread, session_configured }
```

**`Codex::spawn`** is where Layer 3 truly starts — it creates the internal `Submission`/`Event` queues, wires up all the managers (MCP, skills, plugins, extensions), and spawns the `submission_loop` Tokio task that processes all future `Op` commands.

**`finalize_thread_spawn`** (line 1350) waits for `EventMsg::SessionConfigured` as a synchronization point — it proves the session is ready before returning the `CodexThread` handle to the caller. If any other event arrives first, it's a fatal error.


#### `Codex::spawn` internals (`core/src/session/mod.rs:475:spawn_internal()`)

`Codex::spawn` is the single call that brings the entire core to life and is exactly where the entire core sets up:

```
1. create channels  (mod.rs:505)
      (tx_sub, rx_sub) ← Submission Queue — inbound Ops from app-server
      (tx_event, rx_event) ← Event Queue — outbound EventMsgs to app-server

2. load exec policy  (mod.rs:519)
      ExecPolicyManager::load() ← reads shell command allow/deny rules from config
      (guardian sessions get a default policy — cannot be overridden by caller)

3. resolve model  (mod.rs:548)
      models_manager.get_default_model() ← picks the model to use for this thread

4. resolve base instructions  (mod.rs:564)
      priority: config.base_instructions → conversation_history → model default

5. build SessionConfiguration  (mod.rs:591)
      assembles everything into one struct:
      model, approval_policy, cwd, workspace_roots, personality,
      dynamic_tools, environments, session_source, …

6. Session::new(session_configuration, …)  (mod.rs:627)
      creates the Session:
        - loads conversation history
        - sets up MCP servers
        - wires tools, extensions, shell environment
        - emits EventMsg::SessionConfigured as the first event

7. tokio::spawn(submission_loop(session, rx_sub))  (mod.rs:659)
      ← agent's main loop starts as a background Tokio task
      ← runs until Op::Shutdown is received

8. return Codex { tx_sub, rx_event, session, … }  (mod.rs:664)
      ← the handle that CodexThread wraps
      ← app-server submits Ops via tx_sub, reads EventMsgs via rx_event
```

After `Codex::spawn` returns, the agent is alive and waiting for its first `Op`. The `submission_loop` task (step 7) is already running concurrently, ready to process commands.


#### TurnStart request handle

TODO.

## Layer 3

### Lifecycle 

**`CodexThread` is the unit of one conversation.** One is created per chat thread, lives for its duration, and is dropped when the thread is closed.

#### Struct layout (`core/src/codex_thread.rs:145`)

```rust
pub struct CodexThread {
    codex: Codex,                          // owns queues + Arc<Session>
    session_source: SessionSource,
    session_configured: SessionConfiguredEvent,
    rollout_path: Option<PathBuf>,
    out_of_band_elicitation_count: Mutex<u64>,
}
```

`Codex` inside it holds:

```
Codex
 ├─ tx_sub        → pushes Ops into submission_loop
 ├─ rx_event      → reads EventMsgs back out
 └─ session       → Arc<Session>  (history, tools, MCP, exec policy, config…)
```

#### Birth — triggered by app-server

```
TUI sends "initialize" JSON-RPC request
          │
          ▼
app-server/request_processors/thread_processor.rs:1061
    thread_manager.start_thread_with_options(...)
          │
          ▼
core/src/thread_manager.rs:1219   spawn_thread()
          │
          ▼
core/src/session/mod.rs:475       Codex::spawn_internal()
          │
          ├─ 1. (tx_sub, rx_sub) channel created          line 505
          ├─ 2. (tx_event, rx_event) channel created      line 506
          ├─ 3. Session::new(...)                         line 627
          └─ 4. tokio::spawn { submission_loop(rx_sub) }  line 659
          │
          ▼  returns
core/src/thread_manager.rs:1370   CodexThread::new(codex)
          │
          ▼
app-server stores Arc<CodexThread> in ThreadStateManager
```

#### Runtime

- app-server calls `CodexThread::submit(op)` → pushes onto `tx_sub`
- `submission_loop` receives from `rx_sub`, dispatches to handler functions
- handlers emit `EventMsg` via `tx_event` → app-server reads from `rx_event` → forwards to TUI

#### Death

- app-server calls `Op::Shutdown` → `submission_loop` exits, session teardown runs
- or the channel closes (client disconnects) → same teardown path

#### Session Start — `Session::new` (`core/src/session/session.rs:471`)

`Session::new` is called from inside `Codex::spawn_internal` (step 3 of the birth sequence). It is an `async fn` that runs **sequentially** through init phases. The steps, in order:

**Phase 1 — parallel async setup** (`tokio::join!` at line 650)

Four futures run concurrently to minimise startup latency:

```
tokio::join!(
    thread_persistence_fut,      // create or resume LiveThread in ThreadStore (line 532)
    state_db_fut,                // open SQLite StateDb for rollout/state (line 594)
    auth_and_mcp_fut,            // fetch auth token + resolve MCP server list (line 614)
    plugin_and_skill_warmup_fut, // warm plugin/skill caches from config (line 632)
)
```

**Phase 2 — build `SessionConfiguration`**

- Resolves `thread_id`: new `Uuid` for fresh threads, or preserved ID when resuming (`line 510`)
- Resolves `window_generation`: counts prior compaction events in history (`line 516`)
- Resolves shell: user override → zsh-fork feature → `default_user_shell()` (`line 817`)
- Starts `ShellSnapshot` watcher if feature enabled (`line 838`)

**Phase 3 — emit `SessionConfigured` event** (`line 1073`)

Before MCP is even started, the session immediately fires `EventMsg::SessionConfigured` to the TUI. This lets the UI render the thread header (model, cwd, approval policy, initial history messages) without waiting for MCP startup. Also emits any queued `DeprecationNotice` and `Warning` events.

**Phase 4 — start MCP** (`line 1153`)

```rust
let (mcp_connection_manager, cancel_token) = McpConnectionManager::new(
    &mcp_servers,          // resolved in phase 1
    ...,
    tx_event.clone(),      // MCP startup progress → McpStartupUpdate events to TUI
    ...,
).await;
// replaces the placeholder McpConnectionManager::new_uninitialized_*
*sess.services.mcp_connection_manager.write().await = mcp_connection_manager;
```

If any **required** MCP servers fail to connect, `Session::new` returns `Err` and the whole thread creation fails (`line 1202`).

**Phase 5 — prewarm model connection** (`line 1211`)

```rust
sess.schedule_startup_prewarm(base_instructions).await;
```

Fires off a background task that opens a WebSocket/HTTP connection to the model API before the user even types, so the first turn has lower latency.

**Phase 6 — record initial history** (`line 1222`)

```rust
sess.record_initial_history(initial_history).await;
```

For resumed threads: replays the stored rollout items into `SessionState.history` (the `ContextManager`). For new threads: history starts empty.

**Phase 7 — queue session-start hook** (`line 1224`)

```rust
state.queue_pending_session_start_source(session_start_source);
// Startup | Resume | Clear
```

The hook fires on the first `Op::UserInput` turn, not during init — so hooks don't block session creation.

**Summary timeline**

```
Session::new()
  │
  ├─ [parallel] thread_persistence + state_db + auth/MCP resolve + skill warmup
  │
  ├─ resolve thread_id, shell, window_generation
  │
  ├─ emit EventMsg::SessionConfigured  ← TUI can render immediately
  │
  ├─ McpConnectionManager::new()       ← connect to MCP servers (may emit McpStartupUpdate)
  │   └─ if required servers fail → return Err (thread creation aborted)
  │
  ├─ schedule_startup_prewarm()        ← background model connection warmup
  │
  ├─ record_initial_history()          ← replay history into ContextManager
  │
  └─ queue session-start hook          ← fires on first turn, not now
         │
         ▼
  return Arc<Session>  → back to Codex::spawn_internal → submission_loop starts
```

### Receive and Handle Op

After `Session::new` returns, `submission_loop` is already running (started by `tokio::spawn` in `Codex::spawn_internal`). It idles on `rx_sub.recv()` until app-server pushes an `Op`.

#### Overview: `Op::UserInput` three-stage flow

Every `Op::UserInput` goes through three stages: **prepare** → **route** → **execute**.

```
submission_loop (handlers.rs:738) receives Op::UserInput
      │
      ▼
handlers.rs:88   user_input_or_turn()
      │
      ▼
handlers.rs:194  user_input_or_turn_inner()
      │
      │ ── STAGE 1: PREPARE ──────────────────────────────────────────────
      │
      ├─ new_turn_with_sub_id(sub_id, updates)        (turn_context.rs:580)
      │     applies ThreadSettingsOverrides → allocates Arc<TurnContext>
      │     (no task created yet — pure config snapshot)
      │
      │ ── STAGE 2: ROUTE ────────────────────────────────────────────────
      │
      ├─ steer_input(items, additional_context, …)    (mod.rs:3229)
      │     optimistic attempt — ALWAYS called first
      │       Ok(turn_id)        → input injected into running turn (STAGE 3a)
      │       Err(NoActiveTurn)  → no turn running, fall through (STAGE 3b)
      │       Err(other)         → bad input, emit Error event, return
      │
      │ ── STAGE 3a: INJECT (turn already running) ───────────────────────
      │
      ├─ [steer_input Ok] → input pushed into TurnState.pending_input
      │     no new task spawned; running RegularTask picks it up mid-execution
      │
      │ ── STAGE 3b: SPAWN (no turn running) ────────────────────────────
      │
      └─ [steer_input Err(NoActiveTurn)]
               │
               ▼
         spawn_task(turn_context, input, RegularTask)   (tasks/mod.rs:305)
               │
               ├─ abort_all_tasks(Replaced)   ← cancel any stale task
               ├─ clear_connector_selection()
               └─ start_task(turn_context, input, task)   (tasks/mod.rs:316)
                        │
                        ├─ timing: mark_turn_started(), set_turn_started_at_unix_ms()
                        ├─ snapshot: total_token_usage() at turn start
                        ├─ drain Session.input_queue → TurnState.pending_input
                        ├─ emit_turn_start_lifecycle()
                        ├─ build OTel task_span ("turn")
                        │
                        └─ tokio::spawn (task_span) {
                                │
                                ▼
                           task.run(session_ctx, ctx, task_input, cancel_token)
                                │                   ↑ SessionTask trait
                                │
                                ▼
                           flush_rollout()           ← persist transcript
                                │
                                ▼
                           on_task_finished()        ← uniform lifecycle completion
                                │                      (emits TurnComplete, clears ActiveTurn)
                                ▼
                           done.notify_waiters()     ← unblocks any awaiter
                           RunningTask.handle stored in ActiveTurn
                        }
```

`RegularTask::run` is the `SessionTask` implementation for normal agent turns:

```
RegularTask::run()   (tasks/regular.rs:36)
      │
      ├─ emit EventMsg::TurnStarted          ← TUI shows spinner
      │
      ├─ consume_startup_prewarm()           ← take pre-opened WebSocket (first turn only)
      │   → Ready(client_session) | Unavailable | Cancelled
      │
      └─ loop {                              ← outer steer loop
               run_turn(sess, ctx, input, prewarmed_session, …)
                    │
                    ├─ run_pending_session_start_hooks()  ← FIRST TURN ONLY
                    ├─ run_pre_sampling_compact()
                    ├─ build_skills_and_plugins()
                    ├─ run_hooks_and_record_inputs()
                    │
                    └─ loop {               ← inner model ↔ tool call loop
                            send request to ModelClient
                            ├─ tool_call  → execute → feed result → next iteration
                            ├─ text resp  → emit AgentMessage, break
                            └─ poll TurnState.pending_input
                       }
               │
               if !has_pending_input() → return last_agent_message
               else: next_input = Vec::new(), loop again
         }
```

After the outer loop exits, control returns to the `tokio::spawn` closure in `start_task`, which runs `flush_rollout` → `on_task_finished` → `done.notify_waiters`.

#### the tool call loop `core/src/turn:125:run_turn()` (heart of codex)

```rust
/// Takes initial turn input and runs a loop where, at each sampling request,
/// the model replies with either:
///
/// - requested function calls
/// - an assistant message
///
/// While it is possible for the model to return multiple of these items in a
/// single sampling request, in practice, we generally one item per sampling request:
///
/// - If the model requests a function call, we execute it and send the output
///   back to the model in the next sampling request.
/// - If the model sends only an assistant message, we record it in the
///   conversation history and consider the turn complete.
///
pub(crate) async fn run_turn(
    sess: Arc<Session>,
    turn_context: Arc<TurnContext>,
    turn_extension_data: Arc<codex_extension_api::ExtensionData>, // used to add extensions(plugins, connectors)
    input: Vec<TurnInput>,
    prewarmed_client_session: Option<ModelClientSession>, // used to make a requset to LLM.
    cancellation_token: CancellationToken,  //used to interrupt.
) -> Option<String> {
}
```


---

#### `run_turn` — pre-loop setup (`turn.rs:157–193`)

**0. `run_pre_sampling_compact` runs before the first model call of a turn. It has two sub-steps:**

Step 1: `maybe_run_previous_model_inline_compact` (line 785)

Handles the case where the user switched to a smaller model mid-conversation. If the previous turn used a different model and the existing history is now too large for the new model's context window, it compacts using the previous model (not the current one) before handing off — so the compaction summary is generated by a model that can still fit the full history. If there was no previous turn, it's a no-op.

Step 2: `auto_compact_token_status` + `run_auto_compact` (line 765)

Checks if the current history already exceeds the configured compaction budget (either Total token count or BodyAfterPrefix window). If token_limit_reached, it runs compaction right now with CompactionPhase::PreTurn before any new user input is recorded.

The error path aborts the turn: `emit_turn_error_lifecycle` + return `None`.

After compaction, `run_turn` runs six setup steps before entering the model loop:

**1. `record_context_updates_and_set_reference_context_item`** (`session/mod.rs:3005`)

Injects system-level environment state into conversation history so the model sees its current context (cwd, config, tool list, etc.):
- **First turn** (`reference_context_item` is `None`) → `build_initial_context` — full environment snapshot.
- **Subsequent turns** → `build_settings_update_items` — only the *diff* from the previous baseline (minimises token overhead).

Also persists a `TurnContextItem` to the rollout log unconditionally, keeping the resume/replay diff baseline aligned even when no model-visible diff was emitted.

**2. `build_skills_and_plugins`** (`turn.rs:431`)

Returns `(injection_items, explicitly_enabled_connectors)`. Scans user input for `UserInput::Skill` and `UserInput::Mention` variants:
- Skills → loads the `SKILL.md` file, converts to a `ResponseItem` for injection.
- Plugin `@mentions` → queries MCP connection manager for that plugin's tools, converts to guidance items.

`explicitly_enabled_connectors` records which connector IDs were explicitly mentioned — used by the tool router to decide which MCP tools are visible this turn.

**3. `run_pending_session_start_hooks`**

Fires session-start hooks exactly once (first turn only). If any hook returns `should_stop`, `run_turn` returns `None` immediately.

**4. `can_drain_pending_input = input.is_empty()`** (line 166)

A loop-control flag. Starts `false` when there is fresh user input (so the first sampling request uses it, not pending steered input). Becomes `true` after the first successful sampling round. Reset to `false` after auto-compact so tool continuation resumes before any steer is drained.

**5. `run_hooks_and_record_inputs`** (`turn.rs:399`)

For each `TurnInput` item, runs pre-input hooks (`inspect_pending_input`). Each hook can block the item (`should_stop = true`). Accepted items are written into `SessionState.history` via `record_pending_input`. Returns `true` if all items were blocked and no user input survived → `run_turn` returns `None`.

**6. Merge connector selection, record previous turn settings, inject `injection_items`** (lines 171–183)

- `merge_connector_selection` — writes the explicitly-mentioned connectors into session state for the tool router.
- `set_previous_turn_settings` — stores the current model slug + `realtime_active` as `PreviousTurnSettings` so the *next* turn's `maybe_run_previous_model_inline_compact` can detect a model switch.
- Appends skill/plugin `injection_items` to `SessionState.history` so they become model-visible context.
- `track_turn_resolved_config_analytics` — emits observability for the resolved turn config (no side effects).

**Local state before the loop** (lines 185–193):

```rust
let mut last_agent_message: Option<String> = None;
let mut stop_hook_active = false;   // true once a stop hook injects a continuation prompt
let turn_diff_tracker = Arc::new(Mutex::new(
    TurnDiffTracker::with_environment_display_roots(…)
));  // accumulates all file/patch diffs across tool calls; reported in TurnComplete
```

After these six steps, the `loop {` begins.

---

#### `run_turn` — the model ↔ tool call loop (`turn.rs:202`)

**Top of each iteration — drain pending input (lines 206–214)**

```rust
let pending_input = if can_drain_pending_input {
    sess.input_queue.get_pending_input(&sess.active_turn).await
} else { Vec::new() };
run_hooks_and_record_inputs(&sess, &turn_context, &pending_input).await
```

If `can_drain_pending_input` is true, pulls any steered input that `steer_input` pushed while the model was running, runs it through pre-input hooks, and writes accepted items into history. If all items are blocked, `break` (turn ends cleanly). On the first iteration this always yields `Vec::new()` because `can_drain_pending_input` starts `false` when there is fresh user input.

**Build the sampling request (lines 217–221)**

```rust
let sampling_request_input = sess.clone_history().await
    .for_prompt(&turn_context.model_info.input_modalities);
```

Clones the full conversation history and filters it to only the content types the current model supports (e.g. drops images if the model has no vision). This is the complete prompt that goes to the model — everything accumulated so far including context items, injected skills/plugins, tool results, etc.

**`turn_metadata_header` (lines 223–226)**

Reads the current sticky routing token from `TurnMetadataState` and formats it as an HTTP header value. Sent with every request so the server can pin the API call to the same backend instance as the previous call within this turn.

**`run_sampling_request` (lines 227–237)**

The actual model API call. `client_session` is passed by `&mut` so the same WebSocket connection and sticky routing state is reused across all iterations of the loop. Returns `SamplingRequestResult { needs_follow_up, last_agent_message }`. Inside this call is where tool dispatch, approval gating, and event emission happen — the model responds with a tool call, the tool runs, its result is appended to history, and `needs_follow_up = true` is returned.

**`Ok` branch — post-sampling decisions (lines 239–338)**

After a successful response:

1. **`can_drain_pending_input = true`** (line 244) — unlocks mid-turn steer drain from now on.

2. **`needs_follow_up = model_needs_follow_up || has_pending_input`** (line 246) — either the model called a tool that needs a result, or the user typed something while the model was running.

3. **Token limit check (lines 273–291)** — if `token_limit_reached && needs_follow_up` (context window is full but we still have work to do): runs `run_auto_compact(CompactionPhase::MidTurn, InitialContextInjection::BeforeLastUserMessage)` to compress history. After compaction, resets `can_drain_pending_input = !model_needs_follow_up` (if the model was mid-tool-call, defer steered input until tool continuation finishes), then `continue`.

4. **`!needs_follow_up` — turn is finishing (lines 293–337):**
   - `run_turn_stop_hooks` — gives registered stop hooks a chance to inspect the final agent message. If a hook returns `should_block`, `build_hook_prompt_message` builds a continuation prompt from the hook's fragments, injects it into history, sets `stop_hook_active = true`, and `continue` — the model gets one more round driven by the hook. If the hook gave no continuation prompt, emit a `Warning` and fall through.
   - If hook returns `should_stop` → `break`.
   - `run_legacy_after_agent_hook` — old hook API; returns `true` to abort → `return None`.
   - Otherwise `break` — turn is done.

5. **`needs_follow_up` — keep looping (line 338):** `continue` directly — tool result is already in history from inside `run_sampling_request`.

**Error branches (lines 340–377)**

| Error | Action |
|---|---|
| `TurnAborted` | `break` — clean exit; the abort event was already emitted by the cancellation path |
| `InvalidImageRequest` | Try `history.replace_last_turn_images("Invalid image")` → `continue` to retry. If sanitization fails, emit `EventMsg::Error` → `break` |
| Any other error | Emit `EventMsg::Error`, `break` — conversation continues (user can send next message) |

**After the loop (line 381)**

```rust
last_agent_message
```

Returns the last text the model produced (`Option<String>`), or `None` if the turn was aborted/errored before any text response. Propagates back through `RegularTask::run` → `start_task` closure → `on_task_finished`.

---


#### `run_sampling_request` — retry wrapper (`turn.rs:974`)

`run_sampling_request` is a retry loop wrapping `try_run_sampling_request`. Before entering the loop it builds the shared infrastructure for all attempts:

- **`built_tools()`** — assembles a `ToolRouter` from all sources: built-in tools, MCP tools, extension-contributed tools. The router holds both the specs (what the model sees) and the executors (what runs when called).
- **`get_base_instructions()`** — fetches the current system prompt text.
- **`ToolCallRuntime::new(router, sess, turn_context, turn_diff_tracker)`** — wraps the router with execution context; this is what actually dispatches tool calls during the stream.
- **`code_mode_service.start_turn_worker(...)`** — if `ToolMode::CodeMode`, spawns a dispatch worker for nested tool calls from within code cells (RAII: drops when `run_sampling_request` returns, shutting the worker down).

Then the retry loop:

```
loop {
    prompt_input = initial_input.take()
                   OR clone_history().for_prompt()   ← on retries, rebuild from history

    build_prompt(prompt_input, router, turn_context, base_instructions)

    try_run_sampling_request(…)
      → Ok(result)                   return Ok
      → Err(ContextWindowExceeded)   set_total_tokens_full(), return immediately
      → Err(UsageLimitReached)       update_rate_limits(), return immediately
      → Err(retryable)               handle_retryable_response_stream_error()
                                       inc retries, check max_retries
                                       reset client_session WebSocket state
                                       continue
}
```

On retries the `initial_input` is exhausted, so the prompt is rebuilt from the current history — this ensures any tool results recorded during a partially-completed attempt are included.

---

#### `try_run_sampling_request` — the streaming event loop (`turn.rs:1750`)

This is the innermost function that actually talks to the model. It opens one HTTP/WebSocket stream and drives an event loop over it.

**Setup (lines 1761–1807)**

- **`feedback_tags!`** — attaches structured tags (model, approval_policy, sandbox_policy, effort, auth_mode, features) to the current OTel span.
- **`inference_trace`** — trace context for the rollout log (durable conversation transcript), scoped to `sub_id` + model + provider.
- **`sampling_timing_guard`** — starts a timing measurement on `turn_timing_state`. Dropped after the stream loop (line 2188) to record total model response duration.
- **`client_session.stream(...)`** — opens the API stream. Cancellable via `.or_cancel(&cancellation_token)`. Returns a `Stream<Item = CodexResult<ResponseEvent>>`.

Local state for the event loop:

| Variable | Purpose |
|---|---|
| `in_flight: FuturesOrdered` | Concurrent tool execution futures; collected after the stream ends |
| `needs_follow_up` | Accumulates `true` if any item requires a follow-up request |
| `last_agent_message` | The last text message the model produced |
| `active_item` | Currently-streaming item — set on `OutputItemAdded`, cleared on `OutputItemDone` |
| `active_tool_argument_diff_consumer` | Streaming argument diff handler for `CustomToolCall` items |
| `active_item_is_streaming_to_client` | Whether `active_item` is being forwarded to the TUI in real time |
| `assistant_message_stream_parsers` | Parses raw text deltas — strips plan-mode XML tags, handles citations |
| `plan_mode_state` | Buffers agent messages in plan mode until plan approval |
| `defer_streamed_turn_items_for_contributors` | If any `TurnItemContributor` extension is registered, hold items until `OutputItemDone` |

**Event loop (line 1808) — normal turn event sequence**

A normal turn (user text → model reasons → model replies with text + one tool call → tool runs → model gives final answer) produces this sequence:

**`ResponseEvent::Created`** (line 1850)
```rust
ResponseEvent::Created => {}
```
Pure acknowledgement that the server accepted the request. No state change, no event emitted.

---

**`ResponseEvent::OutputItemAdded(Reasoning)`** (line 1944)

`Reasoning` is not a `CustomToolCall` or `FunctionCall` → no diff consumer set. `handle_non_tool_response_item` (`stream_events_utils.rs:517`) matches `ResponseItem::Reasoning` → `parse_turn_item` builds `TurnItem::Reasoning` → `finalize_turn_item` with `TurnItemContributorPolicy::Skip` (extensions don't run yet at `Added` time). Returns `Some(TurnItem::Reasoning)`.

Since it's not an `AgentMessage`, the text-seeding branch is skipped. `stream_item_to_client = !defer_streamed_turn_items_for_contributors`. If no `TurnItemContributor` extensions are registered, `emit_turn_item_started` fires immediately → `EventMsg::TurnItemStarted` to TUI. `active_item = Some(TurnItem::Reasoning)`.

---

**`ResponseEvent::ReasoningSummaryDelta × N`** (line 2129)

```rust
let event = ReasoningContentDeltaEvent { item_id: active.id(), delta, summary_index };
sess.send_event(&turn_context, EventMsg::ReasoningContentDelta(event)).await;
```
Each reasoning chunk forwarded directly to TUI as `ReasoningContentDelta`. `active_item` must be set — if not, `error_or_panic`.

---

**`ResponseEvent::OutputItemDone(Reasoning)`** (line 1851)

Diff consumer finalised (none here — no-op). `active_item.take()` recovers the `TurnItem::Reasoning`. Text flush is skipped (not `AgentMessage`). Falls through to `handle_output_item_done` (`stream_events_utils.rs:405`):
- `ToolRouter::build_tool_call(item)` → `Ok(None)` (not a tool call).
- `finalize_non_tool_response_item` with `TurnItemContributorPolicy::Run` — extensions run this time.
- `emit_turn_item_completed` → `EventMsg::TurnItemCompleted` to TUI.
- `record_completed_response_item` → appended to `SessionState.history`.
- `output.last_agent_message = None`.

---

**`ResponseEvent::OutputItemAdded(AgentMessage)`** (line 1944)

`handle_non_tool_response_item` returns `Some(TurnItem::AgentMessage)`. AgentMessage-specific branch fires:
```rust
let mut seeded = assistant_message_stream_parsers.seed_item_text(&item_id, &raw_text);
// strips hidden XML/plan-mode markup from initial prefix
agent_message.content = [Text { text: seeded.visible_text }];
sess.emit_turn_item_started(&turn_context, &turn_item).await;
// → EventMsg::TurnItemStarted to TUI with the stripped initial text
```
`active_item = Some(TurnItem::AgentMessage)`, `active_item_is_streaming_to_client = true`.

---

**`ResponseEvent::OutputTextDelta × N`** (line 2079)

```rust
let parsed = assistant_message_stream_parsers.parse_delta(&item_id, &delta);
emit_streamed_assistant_text_delta(&sess, &turn_context, …, &item_id, parsed).await;
```
Each chunk is fed through `parse_delta` — strips hidden markup, parses plan-mode XML tags on the fly, returns `ParsedAssistantTextDelta { visible_text }`. Emits `EventMsg::AgentMessageContentDelta` to TUI. User sees text appear token by token.

---

**`ResponseEvent::OutputItemDone(AgentMessage)`** (line 1851)

```rust
flush_assistant_text_segments_for_item(…).await;  // flush any buffered partial markup
```
Then `handle_output_item_done`:
- `ToolRouter::build_tool_call` → `Ok(None)`.
- `finalize_non_tool_response_item` with `TurnItemContributorPolicy::Run` → extensions run, hidden markup stripped on the full text, memory citation parsed.
- `emit_turn_item_completed` → `EventMsg::TurnItemCompleted` to TUI.
- `record_completed_response_item` → history.
- `output.last_agent_message = Some(stripped_text)`.

---

**`ResponseEvent::OutputItemAdded(FunctionCall)`** (line 1944)

```rust
} else if matches!(&item, ResponseItem::FunctionCall { .. }) {
    active_tool_argument_diff_consumer = None;  // clear any previous consumer
}
```
`handle_non_tool_response_item` → `FunctionCall` falls through to `_ => None`. Returns `None`. So `active_item` stays `None`, `active_item_is_streaming_to_client = false`. Tool calls are **not** streamed to the TUI incrementally.

---

**`ResponseEvent::ToolCallInputDelta × N`** (line 2111)

For a plain `FunctionCall`, `active_tool_argument_diff_consumer` is `None` → `continue`. No-op. (Only `CustomToolCall` tools that opted into streaming argument diffs get events here.)

---

**`ResponseEvent::OutputItemDone(FunctionCall)`** (line 1851)

`handle_output_item_done`:
- `ToolRouter::build_tool_call(item)` → `Ok(Some(ToolCall { tool_name, call_id, payload }))`.
- Logs: `tracing::info!("ToolCall: {} {}", call.tool_name, payload_preview)`.
- `accept_mailbox_delivery_for_current_turn` — marks any pending inter-agent mailbox mail as accepted.
- `record_completed_response_item` → tool call item appended to history **immediately** (before result arrives).
- Builds a lazy future:
  ```rust
  let tool_future = Box::pin(ctx.tool_runtime.clone().handle_tool_call(call, cancellation_token));
  ```
  Not yet polled — pushed into `in_flight: FuturesOrdered`.
- `output.needs_follow_up = true`, `output.tool_future = Some(tool_future)`.

Back in event loop: `in_flight.push_back(tool_future)`, `needs_follow_up = true`.

---

**`ResponseEvent::RateLimits(snapshot)`** (line 2045)

```rust
sess.record_rate_limits_info(snapshot).await;
should_emit_token_count = true;
```
Stores rate limit snapshot internally. Sets flag to defer `TokenCount` event emission until after tool execution — avoids sending a progress event while the turn is blocked on a tool.

---

**`ResponseEvent::Completed { token_usage, end_turn }`** (line 2055)

```rust
flush_assistant_text_segments_all(…).await;
sess.record_token_usage_info(&turn_context, token_usage.as_ref()).await;
should_emit_token_count = true;
should_emit_turn_diff = true;
if let Some(false) = end_turn { needs_follow_up = true; }
break Ok(SamplingRequestResult { needs_follow_up, last_agent_message });
```
Stream closed. Breaks the event loop with `needs_follow_up = true` (set by the `FunctionCall` done step).

---

**After the loop: `drain_in_flight()`** (turn.rs:1716)

```rust
while let Some(res) = in_flight.next().await {
    Ok(response_input) => {
        sess.record_conversation_items(&turn_context, &[response_input.into()]).await;
        mark_thread_memory_mode_polluted_if_external_context(…).await;
    }
}
```

**This is where the tool actually executes.** `in_flight.next()` polls `handle_tool_call` (`parallel.rs:63`), which `tokio::spawn`s the tool under a `RwLock` (parallel tools take `read()`, exclusive tools take `write()`), dispatches through `router.dispatch_tool_call_with_terminal_outcome`, and returns a `ResponseInputItem` (the tool result). The result is converted to a `ResponseItem` and appended to `SessionState.history`.

Multiple tool calls from the same response run concurrently but are collected in `FuturesOrdered` — results arrive in issue order.

---

**`send_token_count_event()`** (line 2206)

Now safe to emit `EventMsg::TokenCount` — tool execution is complete, the turn is no longer blocked.

**Cancellation check** (line 2214): `if cancellation_token.is_cancelled() → Err(TurnAborted)`. Checked *after* tool drain so token usage recorded from the completed response is always persisted.

**`TurnDiff`** (line 2218): `turn_diff_tracker.get_unified_diff()` → `EventMsg::TurnDiff` — unified diff of all file changes made by tool calls this iteration.

Returns `Ok(SamplingRequestResult { needs_follow_up: true })` → back in `run_turn` → `continue` → second iteration sends updated history (now including the tool result) to the model for the final text response, which repeats the `OutputItemAdded(AgentMessage)` → deltas → `OutputItemDone` → `Completed` sequence with `needs_follow_up = false` → `break`.

**After the loop (lines 2188–2229)**

```
drop(sampling_timing_guard)          ← record model response duration

flush_assistant_text_segments_all()  ← flush any remaining buffered text (error/abort paths)

if in_flight is non-empty:
    begin_tool_blocking()            ← start timing how long we wait on tools
    drain_in_flight()                ← await all concurrent tool futures (in order)
    drop tool_blocking_timing_guard

if should_emit_token_count:
    send_token_count_event()         ← emitted AFTER tools resolve, so clients don't see
                                       progress events while turn is paused on user input

if cancellation_token.is_cancelled():
    return Err(TurnAborted)          ← checked AFTER tool drain so token usage is persisted

if should_emit_turn_diff:
    turn_diff_tracker.get_unified_diff() → emit TurnDiff event

return outcome
```

Key design: **tool futures are collected during the stream and awaited after it ends**. Multiple tool calls from one model response run concurrently via `FuturesOrdered` but results are presented in order. Cancellation is checked after draining — token usage from the completed response is always persisted even when the user interrupts.

---

#### Stage 1: `new_turn_with_sub_id` — prepare the turn snapshot (`turn_context.rs:580`)

`new_turn_with_sub_id` does **not** create a task. It applies the `ThreadSettingsOverrides` from the incoming `Op` to the current `SessionConfiguration`, writes the updated config back into `SessionState`, and allocates an `Arc<TurnContext>` — a frozen read-only snapshot of everything the turn needs to make decisions: model, cwd, approval policy, reasoning effort, tool schemas, personality, etc.

This snapshot is then passed into `spawn_task` (or `steer_input`) as the authoritative config for the turn. All mutable fields inside `TurnContext` (`turn_metadata_state`, `turn_timing_state`, `extension_data`) are `Arc`-wrapped so the running task can write into them without needing the full snapshot to be mutable.

---

#### Stage 2: `steer_input` — optimistic routing (`mod.rs:3229`)

`steer_input` is **always** called first, even when there is no turn running. It is an optimistic attempt: if a `Regular` turn is active it can inject the new input directly into the running task's `TurnInputQueue` without spawning anything new. If there is no active turn (or the active task is not a `Regular` task) it returns `Err(NoActiveTurn)` and the caller falls through to `spawn_task`.

This design avoids the race condition of checking "is a turn running?" and then acting — instead the check and the enqueue happen atomically under the `active_turn` mutex lock.

---

#### Stage 3a: Input injection into a running turn

When `steer_input` succeeds, the new `TurnInput` items are pushed into `TurnState.pending_input` (the `TurnInputQueue`). No new task is spawned. The already-running `RegularTask` checks `has_pending_input` after each `run_turn` call and loops again if there is work queued. Inside `run_turn`, the inner model loop also polls `pending_input` after each model response, so mid-turn injected input can be picked up without waiting for the outer loop boundary.

---

#### Stage 3b: `spawn_task` + `start_task` — launching a new turn (`tasks/mod.rs:305`)

`spawn_task` is the public entry point when no turn is running. It does three things:

1. **`abort_all_tasks(Replaced)`** — cancels any stale task (normally a no-op on a fresh turn, but guards against races where an old task is still winding down).
2. **`clear_connector_selection()`** — resets any connector/MCP selection from the previous turn.
3. **`start_task(turn_context, input, task)`** — the real work.

`start_task` (`tasks/mod.rs:316`) is the bookkeeping layer before the task actually runs:

- Records turn start timing (`mark_turn_started`, `set_turn_started_at_unix_ms`).
- Snapshots total token usage at turn start (so per-turn deltas can be computed later).
- Drains `Session.input_queue` into `TurnState.pending_input` — this is the **two-level input queue transfer** (see below).
- Emits the turn-start lifecycle event (`emit_turn_start_lifecycle`).
- Builds the OTel `"turn"` span with model/thread/sub_id attributes.
- Calls `tokio::spawn` with the task closure. Inside the closure the actual `task.run(...)` executes, then `flush_rollout`, then `on_task_finished`, then `done.notify_waiters`.
- Stores the resulting `JoinHandle` as `RunningTask` inside `ActiveTurn`.

The post-run sequence (`flush_rollout` → `on_task_finished`) is always emitted from the `start_task` spawn site rather than from inside each `SessionTask` implementation. This keeps the lifecycle identical for all task types (regular turns, compact, review, etc.) without each needing to duplicate it.

---

#### Two-level input queue

User input goes through two staging layers before the model sees it:

```
Op::UserInput arrives
      │
      ▼
Session.input_queue  (session-level staging)
      │   holds items until a turn starts
      │   also manages inter-agent mailbox (watch channel for trigger_turn)
      │
      ▼ drained by start_task via get_pending_input() + extend_pending_input_for_turn_state()
      │
TurnState.pending_input  (TurnInputQueue — turn-level buffer)
      │   owned by the running task
      │   also receives items from steer_input() mid-turn
      │
      ▼
run_turn() / inner model loop polls this
```

The split exists because input can arrive before a turn starts (queued at session level) or while a turn is running (injected directly into the turn-level buffer by `steer_input`). `start_task` bridges them by draining the session queue into the turn queue at turn-start time.

---

#### Prewarm — pre-opening the WebSocket connection (`session_startup_prewarm.rs`)

For WebSocket-backed model providers, `Session::new` schedules a background prewarm task immediately after construction:

```
Session::new
      │
      └─ schedule_startup_prewarm(base_instructions)   (session_startup_prewarm.rs:174)
               │  only runs if responses_websocket_enabled()
               │
               └─ tokio::spawn → schedule_startup_prewarm_inner()
                        │
                        ├─ allocate dummy TurnContext (with INITIAL_SUBMIT_ID)
                        ├─ built_tools()              ← assemble tool list
                        ├─ build_prompt(Vec::new(), …) ← system prompt + empty user turn
                        └─ client_session.prewarm_websocket(prompt, model_info, …)
                               │
                               └─ opens WebSocket, sends system prompt + tool list
                                  (cache-prefills the model connection)
```

On the **first regular turn**, `RegularTask::run` calls `consume_startup_prewarm_for_regular_turn`, which awaits the background handle with a timeout:

- **`Ready(client_session)`** — prewarm succeeded; the already-open WebSocket connection is used for the first API call (avoids connection setup latency and benefits from the cached prompt prefix).
- **`Unavailable`** — timed out, failed, or not scheduled; a fresh connection is opened normally.
- **`Cancelled`** — the session was cancelled before the first turn; returns `None` from `RegularTask::run`.

Subsequent turns always open fresh connections — prewarm is a first-turn-only optimisation.

---

#### `SessionTask` trait — the execution boundary (`tasks/mod.rs:~250`)

`SessionTask` is the trait that all turn implementations satisfy:

```rust
pub(crate) trait SessionTask: Send + Sync + 'static {
    fn kind(&self) -> TaskKind;
    fn span_name(&self) -> &'static str;
    async fn run(
        self: Arc<Self>,
        session: Arc<SessionTaskContext>,
        ctx: Arc<TurnContext>,
        input: Vec<TurnInput>,
        cancellation_token: CancellationToken,
    ) -> Option<String>;   // last agent message text, if any
}
```

`SessionTaskContext` wraps `Arc<Session>` and the turn's `ExtensionData`, giving the task access to session services without exposing the raw `Session` directly. The `Option<String>` return value is the last agent message text — passed to `on_task_finished` for lifecycle bookkeeping.

Implementors include `RegularTask` (normal agent turns), `CompactTask` (context compression), and `ReviewTask` (code review).

---

#### `Op::UserInput` fields (`protocol/src/protocol.rs:472`)

```rust
UserInput {
    items: Vec<UserInput>,                              // the actual user input (see below)
    environments: Option<Vec<TurnEnvironmentSelection>>, // turn-scoped cwd overrides
    final_output_json_schema: Option<Value>,            // constrain model's final response format
    responsesapi_client_metadata: Option<HashMap<String, String>>, // forwarded to Responses API
    additional_context: BTreeMap<String, AdditionalContextEntry>,  // IDE/tool injected context
    thread_settings: ThreadSettingsOverrides,           // config changes applied before turn
}
```

**`UserInput` variants** (`protocol/src/user_input.rs:15`):

```rust
pub enum UserInput {
    Text {
        text: String,
        text_elements: Vec<TextElement>, // byte-range spans for rich UI markers (image placeholders etc.)
    },
    Image {
        image_url: String,               // pre-encoded data URI
        detail: Option<ImageDetail>,
    },
    LocalImage {
        path: PathBuf,                   // converted to base64 data URI at serialization time
        detail: Option<ImageDetail>,
    },
    Skill {
        name: String,
        path: PathBuf,                   // path to SKILL.md
    },
    Mention {
        name: String,
        path: String,                    // e.g. "app://<connector-id>" or "plugin://<name>@<marketplace>"
    },
}
```

**`ThreadSettingsOverrides`** (`protocol/src/protocol.rs:372`) — all fields `Option`, only `Some` are applied:

```rust
pub struct ThreadSettingsOverrides {
    cwd: Option<AbsolutePathBuf>,
    workspace_roots: Option<Vec<AbsolutePathBuf>>,
    profile_workspace_roots: Option<Vec<AbsolutePathBuf>>,
    approval_policy: Option<AskForApproval>,        // change command approval mode
    approvers_reviewer: Option<ApprovalsReviewer>,
    sandbox_policy: Option<SandboxPolicy>,
    permission_profile: Option<PermissionProfile>,
    active_permission_profile: Option<ActivePermissionProfile>,
    windows_sandbox_level: Option<WindowsSandboxLevel>,
    model: Option<String>,                          // switch model mid-conversation
    effort: Option<Option<ReasoningEffortConfig>>,
    summary: Option<ReasoningSummaryConfig>,
    service_tier: Option<Option<String>>,
    collaboration_mode: Option<CollaborationMode>,
    personality: Option<Personality>,
}
```

**`AdditionalContextEntry`** (`protocol/src/protocol.rs:438`):

```rust
pub struct AdditionalContextEntry {
    pub value: String,
    pub kind: AdditionalContextKind,  // Untrusted | Application
}
```

`Untrusted` (e.g. web page content) vs `Application` (e.g. IDE file context) — kept separate so the model can treat them differently in the prompt.

**`TurnEnvironmentSelection`** (`protocol/src/protocol.rs:110`):

```rust
pub struct TurnEnvironmentSelection {
    pub environment_id: String,    // which stored environment to activate
    pub cwd: AbsolutePathBuf,      // its working directory
}
```

Allows a turn to target a specific filesystem environment (Docker container, remote path) without permanently changing the session's default cwd.

#### `TurnInput` — the task input payload (`core/src/session/input_queue.rs:13`)

`task_input: Vec<TurnInput>` is what `spawn_task` (and `steer_input`) actually pass to the running task. It is assembled in `user_input_or_turn_inner` before `spawn_task` is called:

```rust
pub(crate) enum TurnInput {
    UserInput {
        content: Vec<UserInput>,     // the user's actual message (Text / Image / Skill / Mention)
        client_id: Option<String>,   // client-side message ID for dedup/tracking
    },
    ResponseItem(ResponseItem),      // additional context entries, already converted to API items
}
```

Assembly (`handlers.rs:261`):
```
additional_context_input           // AdditionalContextEntry values merged from SessionState
  .map(ResponseItem::from)         // → converted to ResponseItem (API-visible items)
  .map(TurnInput::ResponseItem)    // → wrapped as TurnInput
  + TurnInput::UserInput { content: items, client_id }   // ← appended last
```

So the final `Vec<TurnInput>` is always:
```
[ TurnInput::ResponseItem, …, TurnInput::UserInput { … } ]
  ↑ injected IDE/tool context                 ↑ the user's message
```

The task processes them in order — context items first, then the user message — before making the first model call.

`TurnInputQueue` (`input_queue.rs:23`) is the turn-local buffer inside `TurnState` where `steer_input` pushes mid-turn input. The running `run_turn` loop polls it after each model response to pick up any new input injected while it was executing.

`new_turn_with_sub_id` allocates one `Arc<TurnContext>` per turn. It is a **read-only** snapshot — the running task reads from it without locking `SessionState`. The mutable fields (`turn_metadata_state`, `turn_timing_state`, `extension_data`) are `Arc`-wrapped shared state that the task writes into during execution.

```rust
pub struct TurnContext {
    pub(crate) sub_id: String,               // correlation ID for all emitted events
    pub(crate) model_info: ModelInfo,         // model name, capabilities, context window
    pub(crate) provider: SharedModelProvider, // which backend (OpenAI / Bedrock / Ollama…)
    pub(crate) tool_mode: ToolMode,
    pub(crate) reasoning_effort: Option<ReasoningEffortConfig>,
    pub(crate) reasoning_summary: ReasoningSummaryConfig,
    pub(crate) collaboration_mode: CollaborationMode,
    pub(crate) personality: Option<Personality>,
    pub(crate) approval_policy: Constrained<AskForApproval>,
    pub(crate) permission_profile: PermissionProfile,
    pub(crate) environments: ResolvedTurnEnvironments, // resolved env vars + cwd per environment
    pub(crate) developer_instructions: Option<String>,
    pub(crate) user_instructions: Option<String>,
    pub(crate) compact_prompt: Option<String>,
    pub(crate) final_output_json_schema: Option<Value>,
    pub(crate) dynamic_tools: Vec<DynamicToolSpec>,
    pub(crate) truncation_policy: TruncationPolicy,
    pub(crate) session_source: SessionSource,
    pub(crate) features: ManagedFeatures,
    // mutable shared state (written during turn execution):
    pub(crate) turn_metadata_state: Arc<TurnMetadataState>,
    pub(crate) turn_timing_state: Arc<TurnTimingState>,
    pub(crate) extension_data: Arc<ExtensionData>,
    pub(crate) turn_skills: TurnSkillsContext,
    // …
}
```

#### Other `Op`s (non-turn)

Most are handled entirely within `submission_loop` without spawning a task:

| Op | Handler | What it does |
|---|---|---|
| `Op::Interrupt` | `interrupt()` :68 | Cancels the active turn's `CancellationToken` |
| `Op::ExecApproval` | `exec_approval()` :406 | Sends decision into a pending oneshot channel the exec handler is waiting on |
| `Op::PatchApproval` | `patch_approval()` :448 | Same pattern for patch approvals |
| `Op::Compact` | `compact()` :487 | Spawns a `CompactTask` (replaces turn task) |
| `Op::ThreadRollback` | `thread_rollback()` :494 | Drops last N turns from `SessionState.history` |
| `Op::Shutdown` | `shutdown()` :655 | Tears down services, returns `true` → loop exits |

### Data flowing between Layer 2 (app-server) and Layer 3 (core)                                                 

The communication is defined in `protocol/src/protocol.rs`. It's **bidirectional** — layer 2 sends `Op` commands down, and layer 3 fires `EventMsg` events back up.

#### Layer 2 → Layer 3: `Op` enum (`protocol/src/protocol.rs:447`)

All the things app-server can tell core to do:

| Op variant | Meaning |
|---|---|
| `UserInput { items, environments, thread_settings, ... }` | User typed a message — main trigger for an agent turn |
| `ThreadSettings { thread_settings }` | Change model/config without starting a turn |
| `Interrupt` | Abort the current running turn |
| `CleanBackgroundTerminals` | Kill all background shell processes |
| `ExecApproval { id, decision }` | User approved/denied a shell command |
| `PatchApproval { id, decision }` | User approved/denied a file patch |
| `ResolveElicitation { ... }` | User answered an MCP tool's question |
| `UserInputAnswer { id, response }` | User answered an agent's `request_user_input` call |
| `RequestPermissionsResponse { id, response }` | User granted/denied runtime permissions |
| `DynamicToolResponse { id, response }` | Response to a dynamic tool call |
| `Review { review_request }` | Trigger a code review by the agent |
| `InterAgentCommunication { communication }` | Inter-agent message (multi-agent scenarios) |
| `RefreshMcpServers { config }` | Reload MCP tool servers |
| `ReloadUserConfig` | Hot-reload config changes |
| `Compact` | Ask agent to summarize/compress context |
| `ThreadRollback { num_turns }` | Undo the last N user turns |
| `SetThreadMemoryMode { mode }` | Toggle memory generation for this thread |
| `Shutdown` | Shut down the codex instance |
| `RealtimeConversationStart/Audio/Text/Close` | Voice/realtime streaming ops |

#### Layer 3 → Layer 2: `EventMsg` enum (`protocol/src/protocol.rs:1133`)

Core fires these events back to app-server (which forwards them to the TUI/client):

| Category | Events |
|---|---|
| **Turn lifecycle** | `TurnStarted`, `TurnComplete`, `TurnAborted` |
| **Agent output** | `AgentMessage`, `AgentReasoning`, `UserMessage` |
| **Shell execution** | `ExecCommandBegin`, `ExecCommandOutputDelta`, `ExecCommandEnd` |
| **Approval requests** | `ExecApprovalRequest`, `ApplyPatchApprovalRequest`, `RequestPermissions`, `RequestUserInput` |
| **File patching** | `PatchApplyBegin`, `PatchApplyUpdated`, `PatchApplyEnd` |
| **MCP tools** | `McpToolCallBegin`, `McpToolCallEnd`, `McpStartupUpdate`, `McpStartupComplete` |
| **Context mgmt** | `ContextCompacted`, `ThreadRolledBack`, `ThreadSettingsApplied` |
| **Errors/warnings** | `Error`, `Warning`, `StreamError`, `DeprecationNotice` |
| **Token tracking** | `TokenCount`, `TurnDiff` |
| **Guardian** | `GuardianAssessment`, `GuardianWarning` |
| **Web/image tools** | `WebSearchBegin/End`, `ImageGenerationBegin/End` |
| **Realtime voice** | `RealtimeConversationStarted`, `RealtimeConversationRealtime`, `RealtimeConversationClosed` |
| **Shutdown** | `ShutdownComplete` |

### Op dispatch — `submission_loop`

`core/src/session/handlers.rs:738` — a single `while` loop consuming `Submission` values:

```
app-server:  CodexThread::submit(op)
                    │  wrapped as  Submission { id, op }
                    ▼
handlers.rs:738  submission_loop(rx_sub)
    while let Ok(sub) = rx_sub.recv().await {
        match sub.op {
            Op::UserInput              → user_input_or_turn()       :88
            Op::ThreadSettings         → update_thread_settings()   :104
            Op::Interrupt              → interrupt()                :68
            Op::ExecApproval           → exec_approval()            :406
            Op::PatchApproval          → patch_approval()           :448
            Op::ResolveElicitation     → resolve_elicitation()      :364
            Op::UserInputAnswer        → request_user_input_response()
            Op::RequestPermissionsResponse → request_permissions_response()
            Op::DynamicToolResponse    → dynamic_tool_response()
            Op::Compact                → compact()                  :487
            Op::ThreadRollback         → thread_rollback()          :494
            Op::RefreshMcpServers      → refresh_mcp_servers()      :478
            Op::ReloadUserConfig       → reload_user_config()       :483
            Op::RunUserShellCommand    → run_user_shell_command()   :337
            Op::Review                 → review()                   :702
            Op::Shutdown               → shutdown()  [exits loop]   :655
            Op::RealtimeConversation*  → handle_realtime_*()
        }
    }
```

### `Op::UserInput` — the main agent turn path

```
handlers.rs:88   user_input_or_turn()
      │
      ▼
handlers.rs:194  user_input_or_turn_inner()
      │
      ├─ sess.new_turn_with_sub_id()    ← allocates a TurnContext
      ├─ sess.steer_input()             ← if a turn is already running,
      │                                   queue input mid-turn ("steering")
      └─ sess.spawn_task(RegularTask)   ← start a new turn
               │
               ▼
tasks/mod.rs:305  spawn_task()
      │
      ├─ abort_all_tasks()              ← cancel any currently running turn
      └─ start_task()                   ← tokio::spawn
               │
               ▼
tasks/regular.rs:36  RegularTask::run()
      │
      ├─ emit EventMsg::TurnStarted     ← TUI shows "thinking..."
      └─ loop {
               │
               ▼
         session/turn.rs:135  run_turn()
               │
               ├─ run_pre_sampling_compact()      ← compress context if near limit
               ├─ build_skills_and_plugins()       ← assemble tool list + injections
               ├─ run_hooks_and_record_inputs()    ← fire pre-turn hooks
               └─ loop {                           ← model ↔ tool call loop
                     call ModelProvider
                     if tool_call  → execute tool → feed result back to model
                     if text only  → emit AgentMessage, break
                  }
         → emit EventMsg::TurnComplete
```

### Key files for Layer 3 internals

| File | Line | What it contains |
|---|---|---|
| `core/src/session/handlers.rs` | 738 | `submission_loop` — the `Op` dispatch switch |
| `core/src/session/handlers.rs` | 194 | `user_input_or_turn_inner` — `UserInput` handler |
| `core/src/tasks/mod.rs` | 305 | `spawn_task` — aborts old turn, spawns new tokio task |
| `core/src/tasks/regular.rs` | 36 | `RegularTask::run` — emits `TurnStarted`, calls `run_turn` |
| `core/src/session/turn.rs` | 135 | `run_turn` — the model ↔ tool loop |





## Harness

### Session Management

`Session` is composed of three distinct sub-objects. Each has a clear ownership and mutability role.

#### 1. `Session` — top-level runtime object (`core/src/session/session.rs:21`)

```rust
pub(crate) struct Session {
    pub(crate) thread_id: ThreadId,
    pub(crate) installation_id: String,
    pub(super) tx_event: Sender<Event>,
    pub(super) agent_status: watch::Sender<AgentStatus>,
    pub(super) out_of_band_elicitation_paused: watch::Sender<bool>,
    pub(super) state: Mutex<SessionState>,          // mutable per-turn state
    pub(super) managed_network_proxy_refresh_lock: Semaphore,
    pub(super) features: ManagedFeatures,           // invariant for session lifetime
    pub(super) multi_agent_version: OnceLock<MultiAgentVersion>,
    pub(super) pending_mcp_server_refresh_config: Mutex<Option<McpServerRefreshConfig>>,
    pub(crate) conversation: Arc<RealtimeConversationManager>,
    pub(crate) active_turn: Mutex<Option<ActiveTurn>>, // at-most-one-turn invariant
    pub(crate) input_queue: InputQueue,             // mid-turn steer input
    pub(crate) guardian_review_session: GuardianReviewSessionManager,
    pub(crate) services: SessionServices,           // singleton services
    pub(super) next_internal_sub_id: AtomicU64,
}
```

#### 2. `SessionState` — mutable conversation state (`core/src/state/session.rs:24`, behind `Mutex`)

Changes turn by turn:

```rust
pub(crate) struct SessionState {
    pub(crate) session_configuration: SessionConfiguration, // model, policy, cwd, sandbox…
    pub(crate) history: ContextManager,             // full conversation history → sent to model
    pub(crate) latest_rate_limits: Option<RateLimitSnapshot>,
    pub(crate) server_reasoning_included: bool,
    pub(crate) mcp_dependency_prompted: HashSet<String>,
    pub(crate) additional_context: AdditionalContextStore,
    previous_turn_settings: Option<PreviousTurnSettings>, // model/realtime from last turn
    auto_compact_window: AutoCompactWindow,          // token accounting for auto-compaction
    pub(crate) startup_prewarm: Option<SessionStartupPrewarmHandle>,
    pub(crate) active_connector_selection: HashSet<String>,
    pub(crate) pending_session_start_sources: VecDeque<codex_hooks::SessionStartSource>,
    granted_permissions_by_environment_id: HashMap<String, AdditionalPermissionProfile>,
    next_turn_is_first: bool,
}
```

`SessionConfiguration` (`core/src/session/session.rs:45`) is the full snapshot of per-session settings. It is cloned and mutated by `new_turn_with_sub_id` when `ThreadSettingsOverrides` arrive, then written back into `SessionState`:

```rust
pub(crate) struct SessionConfiguration {
    pub(super) provider: ModelProviderInfo,          // "openai", "openrouter", …
    pub(super) collaboration_mode: CollaborationMode, // model + reasoning effort
    pub(super) model_reasoning_summary: Option<ReasoningSummaryConfig>,
    pub(super) service_tier: Option<String>,
    pub(super) developer_instructions: Option<String>,
    pub(super) user_instructions: Option<LoadedAgentsMd>,
    pub(super) personality: Option<Personality>,
    pub(super) base_instructions: String,            // system prompt text
    pub(super) compact_prompt: Option<String>,
    pub(super) approval_policy: Constrained<AskForApproval>,
    pub(super) approvals_reviewer: ApprovalsReviewer,
    pub(super) permission_profile_state: PermissionProfileState,
    pub(super) windows_sandbox_level: WindowsSandboxLevel,
    pub(super) cwd: AbsolutePathBuf,                 // session working directory
    pub(super) workspace_roots: Vec<AbsolutePathBuf>,
    pub(super) codex_home: AbsolutePathBuf,
    pub(super) thread_name: Option<String>,
    pub(super) environments: Vec<TurnEnvironmentSelection>,
    pub(super) session_source: SessionSource,        // cli | vscode | exec | mcp | …
    pub(super) forked_from_thread_id: Option<ThreadId>,
    pub(super) parent_thread_id: Option<ThreadId>,
    pub(super) dynamic_tools: Vec<DynamicToolSpec>,
    pub(super) inherited_shell_snapshot: Option<Arc<ShellSnapshot>>,
    pub(super) user_shell_override: Option<shell::Shell>,
    // ...
}
```

#### 4. `ActiveTurn` — the running-turn handle (`core/src/state/turn.rs:29`)

`Session.active_turn: Mutex<Option<ActiveTurn>>` enforces the **at-most-one-turn** invariant. `None` = idle, `Some` = a task is running.

```rust
pub(crate) struct ActiveTurn {
    pub(crate) task: Option<RunningTask>,
    pub(crate) turn_state: Arc<Mutex<TurnState>>,
}
```

**`RunningTask`** (`turn.rs:71`) — the live tokio task handle:

```rust
pub(crate) struct RunningTask {
    pub(crate) done: Arc<Notify>,
    pub(crate) kind: TaskKind,              // Regular | Review | Compact
    pub(crate) task: Arc<dyn AnySessionTask>,
    pub(crate) cancellation_token: CancellationToken,
    pub(crate) handle: AbortOnDropHandle<()>, // dropping this kills the task
    pub(crate) turn_context: Arc<TurnContext>,
    pub(crate) turn_extension_data: Arc<ExtensionData>,
    pub(crate) _timer: Option<codex_otel::Timer>,
}
```

**`TurnState`** (`turn.rs:85`) — mutable state for this one turn only, dropped when the turn ends:

```rust
pub(crate) struct TurnState {
    pending_approvals: HashMap<String, oneshot::Sender<ReviewDecision>>,
    pending_request_permissions: HashMap<String, PendingRequestPermissions>,
    pending_user_input: HashMap<String, oneshot::Sender<RequestUserInputResponse>>,
    pending_elicitations: HashMap<(String, RequestId), oneshot::Sender<ElicitationResponse>>,
    pending_dynamic_tools: HashMap<String, oneshot::Sender<DynamicToolResponse>>,
    pub(crate) pending_input: TurnInputQueue,   // steer_input pushes here
    mailbox_delivery_phase: MailboxDeliveryPhase, // CurrentTurn | NextTurn
    granted_permissions_by_environment_id: HashMap<String, AdditionalPermissionProfile>,
    strict_auto_review_enabled: bool,
    pub(crate) tool_calls: u64,
    pub(crate) has_memory_citation: bool,
    pub(crate) token_usage_at_turn_start: TokenUsage,
}
```

All pending approval/input `oneshot` channels live in `TurnState`. When `spawn_task` replaces the active turn (via `AbortOnDropHandle`), all outstanding waiters are dropped and their channels close — automatic cleanup with no explicit teardown code.

#### 3. `SessionServices` — singleton services (`core/src/state/service.rs:41`, immutable after init)

All long-lived infrastructure shared across turns:

```rust
pub(crate) struct SessionServices {
    pub(crate) mcp_connection_manager: Arc<RwLock<McpConnectionManager>>,
    pub(crate) mcp_startup_cancellation_token: Mutex<CancellationToken>,
    pub(crate) unified_exec_manager: UnifiedExecProcessManager, // runs shell commands
    pub(crate) analytics_events_client: AnalyticsEventsClient,
    pub(crate) hooks: ArcSwap<Hooks>,               // hot-reloadable
    pub(crate) user_shell: Arc<crate::shell::Shell>,
    pub(crate) exec_policy: Arc<ExecPolicyManager>, // which commands are allowed
    pub(crate) auth_manager: Arc<AuthManager>,
    pub(crate) models_manager: SharedModelsManager,
    pub(crate) session_telemetry: SessionTelemetry,
    pub(crate) tool_approvals: Mutex<ApprovalStore>,
    pub(crate) guardian_rejections: Mutex<HashMap<String, GuardianRejection>>,
    pub(crate) guardian_rejection_circuit_breaker: Mutex<GuardianRejectionCircuitBreaker>,
    pub(crate) skills_manager: Arc<SkillsManager>,
    pub(crate) plugins_manager: Arc<PluginsManager>,
    pub(crate) mcp_manager: Arc<McpManager>,
    pub(crate) extensions: Arc<ExtensionRegistry<crate::config::Config>>,
    pub(crate) session_extension_data: ExtensionData,
    pub(crate) thread_extension_data: ExtensionData,
    pub(crate) agent_control: AgentControl,
    pub(crate) network_proxy: ArcSwapOption<StartedNetworkProxy>, // hot-reloadable
    pub(crate) network_approval: Arc<NetworkApprovalService>,
    pub(crate) state_db: Option<StateDbHandle>,
    pub(crate) thread_store: Arc<dyn ThreadStore>,  // persists history to disk
    pub(crate) model_client: ModelClient,           // HTTP client, cached across turns
    pub(crate) code_mode_service: CodeModeService,
    pub(crate) environment_manager: Arc<EnvironmentManager>,
}
```

#### Design decisions

| Decision | Why |
|---|---|
| `SessionState` behind `Mutex` | Multiple async tasks (turns, approvals, interrupts) all read/write conversation state concurrently |
| `SessionServices` not behind a lock | Initialized once, never mutated — `ArcSwap` only for the two hot-reloadable fields (`hooks`, `network_proxy`) |
| `active_turn: Mutex<Option<ActiveTurn>>` | Enforces at-most-one-running-turn — `spawn_task` aborts the old one before inserting a new one |
| `input_queue` separate from `SessionState` | Lets the TUI "steer" (inject input mid-turn) without blocking on the state lock |


### Prompt build

Every sampling call to the model is driven by a `Prompt` struct
(`core/src/client_common.rs:18`).  Understanding prompt build means tracing
how that struct is populated.

---

#### The `Prompt` struct

```rust
// core/src/client_common.rs:18
pub struct Prompt {
    /// Conversation context input items.
    pub input: Vec<ResponseItem>,

    /// Tools available to the model, including additional tools sourced from
    /// external MCP servers.
    pub(crate) tools: Vec<ToolSpec>,

    /// Whether parallel tool calls are permitted for this prompt.
    pub(crate) parallel_tool_calls: bool,

    pub base_instructions: BaseInstructions,

    /// Optionally specify the personality of the model.
    pub personality: Option<Personality>,

    /// Optional the output schema for the model's response.
    pub output_schema: Option<Value>,

    /// Whether the Responses API should strictly validate `output_schema`.
    pub output_schema_strict: bool,
}
```

Four concerns packed into one struct:

| Field | What it carries |
|---|---|
| `input` | Full conversation history (user messages, assistant replies, tool calls/outputs, skill injections) |
| `tools` | All tool specs visible to the model (built-in + MCP + extension tools) |
| `base_instructions` | System/instructions text (maps to the `instructions` field in the Responses API) |
| `parallel_tool_calls` / `output_schema*` | Per-model call-site settings |

---

#### Where `build_prompt` is called

```
run_turn (turn.rs:135)
 └─ run_sampling_request (turn.rs:974)
     retry loop {
       └─ build_prompt (turn.rs:1013)          ← constructs the Prompt
       └─ try_run_sampling_request (turn.rs:1019)
           └─ client_session.stream(prompt, ...)  ← sends it to the model
     }
```

`build_prompt` is inside a retry loop — on each retry the history is
re-snapshotted via `sess.clone_history().for_prompt()` (`turn.rs:1009`) so
the `Prompt` always reflects the latest state of `ContextManager`.

`build_prompt` at `turn.rs:945`:

```rust
pub(crate) fn build_prompt(
    input: Vec<ResponseItem>,
    router: &ToolRouter,
    turn_context: &TurnContext,
    base_instructions: BaseInstructions,
) -> Prompt {
    Prompt {
        input,
        tools: router.model_visible_specs(),
        parallel_tool_calls: turn_context.model_info.supports_parallel_tool_calls,
        base_instructions,
        personality: turn_context.personality,
        output_schema: turn_context.final_output_json_schema.clone(),
        output_schema_strict: !crate::guardian::is_guardian_reviewer_source(
            &turn_context.session_source,
        ),
    }
}
```

---

#### Part 1 — `base_instructions` (the system prompt)

`BaseInstructions` is a simple newtype:

```rust
// protocol/src/models.rs:920
pub struct BaseInstructions {
    pub text: String,
}
```

The `text` value flows through this chain:

```
models catalog (ModelInfo.base_instructions)
  └─ ModelInfo::get_model_instructions(personality)
      ├─ if model_messages.instructions_template exists
      │     replace PERSONALITY_PLACEHOLDER with personality snippet
      └─ else: return ModelInfo.base_instructions as-is

Session::new (session.rs:564):
  base_instructions = config.base_instructions          // CLI/API override?
    .or_else(|| history.get_base_instructions())        // resumed thread?
    .unwrap_or_else(|| model_info.get_model_instructions(config.personality))
                                                        // ← default path

Stored in SessionConfiguration.base_instructions (session.rs:64)

Session::get_base_instructions (session/mod.rs:1172):
    lock state → return BaseInstructions { text: state.session_configuration.base_instructions.clone() }

run_sampling_request (turn.rs:986):
    let base_instructions = sess.get_base_instructions().await;
    ...
    let prompt = build_prompt(input, router, turn_context, base_instructions);
```

The raw instruction text ultimately comes from:
- **Primary**: the model's entry in the models catalog JSON (`ModelInfo.base_instructions`) — a static string baked into the binary
- **Override**: `config.instructions` / `config.model_instructions_file` in `config.toml` or via CLI (`config/mod.rs:3246`)
- **Resume**: `history.get_base_instructions()` — the instructions string stored in the rollout when the thread was originally created

The default base instructions for all models are embedded via `include_str!`:
```rust
// protocol/src/models.rs:918
pub const BASE_INSTRUCTIONS_DEFAULT: &str = include_str!("prompts/base_instructions/default.md");
```

---

#### Part 2 — `input` (the conversation history)

The `input` field is built from `ContextManager` — the session's append-only conversation log:

```
run_turn (turn.rs:217):
    let sampling_request_input: Vec<ResponseItem> = {
        sess.clone_history()
            .await
            .for_prompt(&turn_context.model_info.input_modalities)
    };
```

`ContextManager::for_prompt` (`context_manager/history.rs:119`):

```rust
pub(crate) fn for_prompt(mut self, input_modalities: &[InputModality]) -> Vec<ResponseItem> {
    self.normalize_history(input_modalities);
    self.items
}
```

`normalize_history` enforces three invariants before the items go to the model:
1. Every function/tool call has a corresponding output (`ensure_call_outputs_present`)
2. Every output has a corresponding call (`remove_orphan_outputs`)
3. Images are stripped when the model does not support them (`strip_images_when_unsupported`)

`ContextManager.items` is a `Vec<ResponseItem>` ordered oldest → newest.
Items are appended by `record_items()` which also applies a `TruncationPolicy`
to tool output payloads (caps oversized outputs before they enter the history).

**What goes into `items`?**  Only API-visible items — `is_api_message` filters:

```
✓  user messages (role ≠ "system")
✓  assistant messages
✓  function/tool calls
✓  function/tool outputs (truncated by TruncationPolicy)
✓  local shell calls
✓  reasoning items
✓  web search calls / image generation calls
✓  Compaction / ContextCompaction items

✗  system-role messages  (kept separately in base_instructions)
✗  CompactionTrigger     (internal marker, never sent to the model)
✗  Other
```

---

#### Part 3 — skill / plugin injection items

Before any sampling call, `build_skills_and_plugins` (`turn.rs:431`) prepares
extra `ResponseItem` values that are **prepended to the history** for this turn:

```rust
// turn.rs:573
let mut injection_items: Vec<ResponseItem> = /* skill_items + plugin_items + extension_items */;
// turn.rs:583
injection_items.extend(plugin_items);
injection_items.extend(extension_injection_items);
```

These injection items are then committed to history before the first sampling
call (`turn.rs:178–181`):

```rust
for response_item in injection_items {
    sess.record_conversation_items(&turn_context, std::slice::from_ref(&response_item))
        .await;
}
```

Sources of injection items:

| Source | What it adds |
|---|---|
| `build_skill_injections` | Skill prompt text (from `@skill` mentions or active skills) |
| `build_plugin_injections` | Plugin guidance items (from `plugin://` mentions) |
| `build_extension_turn_input_items` | Contributions from registered extension `TurnInputContributor`s |

---

#### Part 4 — `tools` (the tool list)

`router.model_visible_specs()` returns all tool specs the model can invoke.
The `ToolRouter` is built by `built_tools` (`turn.rs:1077`):

```
built_tools (turn.rs:1077):
  1. read McpConnectionManager (RwLock) → list all MCP tools
  2. load plugins (plugins_manager.plugins_for_config)
  3. determine accessible connectors (app tools)
  4. run tool-suggest for discoverable tools
  5. build_mcp_tool_exposure → partitions MCP tools into direct vs deferred
  6. ToolRouter::from_turn_context(turn_context, ToolRouterParams { mcp_tools, deferred_mcp_tools, discoverable_tools, extension_tool_executors, dynamic_tools })
```

`ToolRouter` merges:
- Built-in tools (shell execution, apply_patch, web_search, etc. — wired by `TurnContext`)
- MCP server tools (from `McpConnectionManager`)
- Extension-contributed tool executors
- Dynamically-registered tools (`dynamic_tools`)

---

#### Full prompt assembly sequence (per sampling call)

```
run_turn
  1. run_pre_sampling_compact          ← compact if near token limit
  2. record_context_updates_and_set_reference_context_item
  3. build_skills_and_plugins          ← assemble injection_items + connector IDs
  4. run_pending_session_start_hooks   ← fire session-start hooks (first turn only)
  5. run_hooks_and_record_inputs       ← run pre-turn hooks; record user input → history
  6. record injection_items → history  ← skill/plugin/extension items now in ContextManager
  loop {
    7. sess.clone_history().for_prompt() → sampling_request_input
    8. run_sampling_request
        a. built_tools()               ← build ToolRouter (MCP + built-in + extensions)
        b. sess.get_base_instructions()← lock SessionState → read base_instructions text
        c. build_prompt(input, router, turn_context, base_instructions)
             Prompt {
               input:               history items (normalized, images stripped if needed)
               tools:               router.model_visible_specs()
               base_instructions:   { text: model's system prompt string }
               parallel_tool_calls: model_info.supports_parallel_tool_calls
               personality:         turn_context.personality
               output_schema:       turn_context.final_output_json_schema
               output_schema_strict: !is_guardian_reviewer
             }
        d. client_session.stream(prompt, model_info, ...) → ResponseStream
        e. loop over stream events → handle tool calls → append outputs to history
           → if model returns text: emit EventMsg::AgentMessage, break
  }
```


### Tool System

The model can't execute code by itself. When it wants to run a shell command, read a file, or call an MCP server, it returns a structured `FunctionCall` item in the response stream — essentially: "I want to call tool X with these arguments." Something on the process side has to receive that call, execute the actual work, and send back a `FunctionCallOutput` so the model can continue.

That "something" is a handler. The tool system's job is to:
1. Tell the model what tools exist and what their argument schemas are (outbound: `tools` array in the HTTP request)
2. Route each `FunctionCall` the model returns to the right piece of code (inbound: dispatch)
3. Run the tool and inject the result back into the conversation (outbound: `FunctionCallOutput` in the next request)

The handler is the unit that owns all three for one specific tool.

#### Step 1 — What is a handler?

A handler is a plain Rust struct that implements two traits: `ToolExecutor<ToolInvocation>` and `CoreToolRuntime`.

`ToolExecutor` is the base trait (`tools/src/tool_executor.rs:44`). Every handler must provide:
- `tool_name()` — returns the canonical name the model uses to call this tool (e.g. `"shell_command"`)
- `spec()` — returns a `ToolSpec` that will be serialized to JSON and sent to the model
- `handle(invocation)` — async function that actually executes the tool and returns its output

`CoreToolRuntime` (`core/src/tools/registry.rs:46`) extends `ToolExecutor` with optional hooks:
- `matches_kind(payload)` — rejects incompatible payload shapes before dispatch
- `waits_for_runtime_cancellation()` — lets the handler finish teardown on cancellation
- `pre_tool_use_payload / post_tool_use_payload` — expose hook-facing input/output (for Claude Code hooks)
- `with_updated_hook_input` — re-injects a hook-rewritten argument back into the invocation
- `create_diff_consumer` — streams partial tool arguments as they arrive

A concrete example — `ShellCommandHandler` (`core/src/tools/handlers/shell/shell_command.rs`):

```rust
pub struct ShellCommandHandler {
    backend: ShellCommandBackend,
    options: ShellCommandHandlerOptions,   // allow_login_shell, exec_permission_approvals_enabled
}

impl ToolExecutor<ToolInvocation> for ShellCommandHandler {
    fn tool_name(&self) -> ToolName { ToolName::plain("shell_command") }

    fn spec(&self) -> ToolSpec {
        create_shell_command_tool(CommandToolOptions { … })  // returns ToolSpec::Function(…)
    }

    async fn handle(&self, invocation: ToolInvocation) -> Result<Box<dyn ToolOutput>, …> {
        // deserialize ToolPayload::Function { arguments } into ShellCommandToolCallParams
        // call run_exec_like(…) → ShellRuntime → OS process
    }
}

impl CoreToolRuntime for ShellCommandHandler {
    fn waits_for_runtime_cancellation(&self) -> bool { true }
    fn pre_tool_use_payload(&self, …) → Option<PreToolUsePayload> { … }
    fn post_tool_use_payload(&self, …) → Option<PostToolUsePayload> { … }
    fn with_updated_hook_input(&self, …) { … }
}
```

`spec()` returns `ToolSpec::Function(ResponsesApiTool { name, description, strict, parameters: JsonSchema })` — this is the JSON schema the model sees.

There are ~20 built-in handlers: `ShellCommandHandler`, `ExecCommandHandler`, `ApplyPatchHandler`, `McpHandler`, `ToolSearchHandler`, `SpawnAgentHandler`, `PlanHandler`, `ViewImageHandler`, `RequestPermissionsHandler`, etc.

---

#### Step 2 — How handlers are instantiated each turn (`core/src/tools/spec_plan.rs:151`)

Every call to `run_sampling_request` (`core/src/session/turn.rs:974`) starts by calling `built_tools(sess, turn_context, …)` (`turn.rs:1077`). `built_tools` fetches live MCP tool info, plugin/discoverable tool lists, and then calls `ToolRouter::from_turn_context(turn_context, ToolRouterParams { … })` (`router.rs:48`), which calls `build_tool_router(turn_context, params)` (`spec_plan.rs:151`). This is where all handlers are created and filtered.

`build_tool_router` calls `build_tool_specs_and_registry` which:
1. Constructs a `PlannedTools` builder (default-empty)
2. Calls `add_tool_sources` which calls sub-functions to conditionally instantiate handlers:
   - `add_shell_tools` — creates `ShellCommandHandler` or `ExecCommandHandler` based on `ConfigShellToolType` from `TurnContext`
   - `add_mcp_resource_tools`, `add_core_utility_tools`, `add_collaboration_tools`, `add_mcp_runtime_tools`, `add_extension_tools`, `add_dynamic_tools`
   - `hosted_model_tool_specs` — adds `ToolSpec::WebSearch` / `ToolSpec::ImageGeneration` directly (these are server-side hosted tools, no Rust handler)
3. Calls `append_tool_search_executor` — adds `ToolSearchHandler` only if any deferred tools exist
4. Calls `prepend_code_mode_executors` — adds `CodeModeExecuteHandler` and `CodeModeWaitHandler` if tool_mode is CodeMode
5. Calls `build_model_visible_specs_and_registry(turn_context, planned_tools)` (`spec_plan.rs:189`) — splits the collected handlers into two outputs:
   - `model_visible_specs: Vec<ToolSpec>` — specs from `Direct`/`DirectModelOnly` handlers + `hosted_specs`; namespaces merged and sorted
   - `ToolRegistry` — flat `HashMap<ToolName, Arc<dyn CoreToolRuntime>>` of every handler regardless of exposure

`PlannedTools` is a simple builder (`spec_plan.rs:98`):
```rust
struct PlannedTools {
    runtimes: Vec<Arc<dyn CoreToolRuntime>>,  // handlers that can be dispatched
    hosted_specs: Vec<ToolSpec>,               // server-side tools (no local handler)
}
impl PlannedTools {
    fn add<T: CoreToolRuntime>(&mut self, handler: T) { … }           // Direct exposure
    fn add_with_exposure<T>(&mut self, handler: T, exp: ToolExposure) { … }
    fn add_dispatch_only<T>(&mut self, handler: T) { … }              // Hidden exposure
    fn add_hosted_spec(&mut self, spec: ToolSpec) { … }
}
```

A key design choice: `add_shell_tools` at `spec_plan.rs:582` shows both co-existence and overrides:
```rust
ConfigShellToolType::UnifiedExec => {
    planned_tools.add(ExecCommandHandler::new(…));      // Direct — model sees this
    planned_tools.add(WriteStdinHandler);
    planned_tools.add_dispatch_only(ShellCommandHandler::new(…));  // Hidden — dispatch only
}
ConfigShellToolType::Default | … => {
    planned_tools.add(ShellCommandHandler::new(…));     // Direct — model sees this
}
```

The old `shell_command` tool is kept dispatch-only during the `UnifiedExec` rollout so that already-in-flight model calls that reference `shell_command` can still be dispatched, while the model only sees `exec` in its tool list.

---

#### Step 3 — `ToolExposure`: which specs reach the model

Every handler has an exposure, set either by its own `exposure()` impl or by wrapping it in `ExposureOverride` (`registry.rs:247`).

Four variants (`tools/src/tool_executor.rs:9`):
- `Direct` — included in the initial `tools` array sent to the model; also accessible as a nested code-mode tool
- `DirectModelOnly` — included in the initial `tools` array, but excluded from code-mode nesting
- `Deferred` — omitted from the initial `tools` array; discoverable only via `tool_search`
- `Hidden` — registered for dispatch only, never sent to the model

`build_model_visible_specs_and_registry` (`spec_plan.rs:189`) splits the handlers:
```
for each runtime in planned_tools.runtimes:
    if exposure.is_direct() (i.e. Direct or DirectModelOnly):
        push runtime.spec() → model_visible_specs
    always:
        register into ToolRegistry (HashMap<ToolName, Arc<dyn CoreToolRuntime>>)
```
Then `hosted_specs` (server-side tools) are appended to `model_visible_specs` directly.

`ExposureOverride` is a decorator that wraps a handler and returns a different exposure without touching the handler's code. Used extensively in `add_collaboration_tools` and `add_extension_tools`.

---

#### Step 4 — `ToolRouter`: the per-turn gateway (Core)

`build_tool_router` returns `ToolRouter { registry, model_visible_specs }` (`core/src/tools/router.rs:34`).

```rust
pub struct ToolRouter {
    registry: ToolRegistry, // ALL handlers (Direct + Deferred + Hidden)
    model_visible_specs: Vec<ToolSpec>, //// only Direct/DirectModelOnly + hosted specs
}
```

`ToolRouter` has two responsibilities:
1. **Outbound (before the model call):** `model_visible_specs()` returns the pre-computed `Vec<ToolSpec>` that goes into the prompt
2. **Inbound (after the model responds):** `build_tool_call(item)` converts a `ResponseItem` from the stream into a typed `ToolCall` that the registry can dispatch

`build_tool_call` handles three stream item shapes:
- `FunctionCall { name, call_id, arguments }` → `ToolCall::Function`
- `CustomToolCall { name, call_id, parameters }` → `ToolCall::Function` (code-mode tools)
- `ToolSearchCall { call_id, query }` → `ToolCall::ToolSearch`

---

#### Step 5 — From `ToolSpec` to wire JSON

`Prompt` is the per-request value type (`client_common.rs:18`):
```rust
pub struct Prompt {
    pub input: Vec<ResponseItem>,          // conversation history
    pub tools: Vec<ToolSpec>,              // model_visible_specs from ToolRouter
    pub parallel_tool_calls: bool,
    pub base_instructions: Option<…>,
    …
}
```

`ModelClientSession::build_request()` (`client.rs:763`) converts `Prompt` to the wire struct:
```rust
let tools = create_tools_json_for_responses_api(&prompt.tools)?;
// create_tools_json_for_responses_api = serde_json::to_value(t) per ToolSpec
// ToolSpec has #[serde(tag = "type")] so it injects the "type" discriminant automatically

ResponsesApiRequest {
    model, instructions, input,
    tools,                          // Vec<serde_json::Value>
    tool_choice: "auto",
    stream: true, …
}
```

Wire result:
```json
{
  "tools": [
    {"type": "function", "name": "shell_command", "description": "…", "strict": false, "parameters": {…}},
    {"type": "web_search", "external_web_access": true}
  ]
}
```

---

#### Step 6 — Connecting back to `Op::UserInput` (the turn flow)

This is how the tool system connects to the `Op::UserInput` flow analyzed earlier:

```
Op::UserInput → run_turn() → run_sampling_request()
                                  │
                                  ├─ build_tool_router()        ← Step 2 above: all handlers instantiated
                                  │    returns ToolRouter { registry, model_visible_specs }
                                  │
                                  ├─ build_prompt(tool_router, …)
                                  │    Prompt { tools: tool_router.model_visible_specs(), … }
                                  │
                                  ├─ client_session.stream(prompt, …) → HTTP request with "tools":[…]
                                  │
                                  └─ loop over stream events:
                                        handle_output_item_done(item, tool_router, …)
                                              │
                                              ├─ tool_router.build_tool_call(item)
                                              │    ResponseItem → ToolCall
                                              │
                                              ├─ tool_router.registry.dispatch_any_with_terminal_outcome(tool_call, …)
                                              │    lookup handler by ToolName
                                              │    run pre_tool_use hook
                                              │    handler.handle(ToolInvocation { session, turn, payload, … })
                                              │    run post_tool_use hook
                                              │    emit EventMsg::ToolCallOutput
                                              │
                                              └─ append FunctionCallOutput to history
                                                   → next iteration sends updated history to model
```

`ToolInvocation` (`core/src/tools/context.rs`) is the value passed to every `handle()` call:
```rust
pub struct ToolInvocation {
    pub session: Arc<Session>,
    pub turn: Arc<TurnContext>,
    pub cancellation_token: CancellationToken,
    pub tracker: SharedTurnDiffTracker,
    pub call_id: String,
    pub tool_name: ToolName,
    pub payload: ToolPayload,     // Function { arguments: String } or ToolSearch { query }
}
```

The handler gets everything it needs from `ToolInvocation`: the session for auth/config, the turn for the current environment and approval policy, the `call_id` to correlate the response, and the raw JSON arguments to deserialize into its own params struct.


#### Tool Execution

**`ResponseEvent` sequence for a tool call**

For a single `FunctionCall` tool call, the stream delivers these events in order:

```
Created                                    // response object opened

OutputItemAdded(FunctionCall { … })        // tool call item starts
                                           //   → active_tool_argument_diff_consumer = None
                                           //   → handle_non_tool_response_item → None (no-op)

ToolCallInputDelta { call_id, delta }      // argument JSON streamed incrementally
ToolCallInputDelta { call_id, delta }      // (0..N times)
  → active_tool_argument_diff_consumer is None (set to None on OutputItemAdded for FunctionCall)
  → first line of handler: `let Some(...) = consumer.as_mut() else { continue }`
  → every delta hits `continue` — all deltas silently skipped
  → arguments are only consumed once, complete, from OutputItemDone

OutputItemDone(FunctionCall { … })         // tool call item complete, full arguments available
  → ToolRouter::build_tool_call() → ToolCall { tool_name, call_id, payload }
  → tool_runtime.handle_tool_call() spawned as Tokio task → pushed to in_flight
  → needs_follow_up = true

Completed { end_turn, token_usage }        // model done for this round
  → break loop
  → drain_in_flight(): await each tool future
      → FunctionCallOutput appended to history
```

For a `CustomToolCall` (code-mode tool), the same sequence applies but `OutputItemAdded` sets up a `ToolArgumentDiffConsumer`, so `ToolCallInputDelta` events are forwarded to the consumer for incremental rendering instead of being dropped.

**`FunctionCall` vs `CustomToolCall`**

| | `FunctionCall` | `CustomToolCall` |
|---|---|---|
| Who calls it | Model using standard tool use | Model using code-mode (`execute` tool) |
| Arguments format | Plain JSON string, complete on `OutputItemDone` | Structured diffs streamed via `ToolCallInputDelta` |
| Diff consumer | `None` — deltas silently skipped | Set up on `OutputItemAdded`, fed each delta |
| `ToolPayload` | `Function { arguments: String }` | `Function { arguments: String }` (same, assembled from diffs) |
| UI streaming | Arguments not streamed to UI | Consumer can emit `EventMsg` per delta (e.g. partial code block) |
| `OutputItemDone` | `build_tool_call` → dispatch | Same |

**The sampling loop — `try_run_sampling_request` (`turn.rs:1752`)**


This is the innermost loop that drives both text output and tool calls. It runs after `build_prompt` and `client_session.stream()` open the HTTP stream:

```
stream = client_session.stream(prompt, …)       // open HTTP SSE stream
in_flight = FuturesOrdered::new()               // pending tool futures

loop over stream events:
    ResponseEvent::OutputItemAdded(item)  → start streaming text delta to client
    ResponseEvent::OutputItemDone(item)   → handle_output_item_done()
        tool call  → spawn tool_future → push to in_flight
        text/image → emit EventMsg to client
    ResponseEvent::Completed { end_turn } → flush text, record token usage
        end_turn == false → needs_follow_up = true
        break
    ToolCallInputDelta { call_id, delta } -> ...

// after stream closes — collect all tool results
drain_in_flight(in_flight)               // await each future, append FunctionCallOutput to history
```

If `needs_follow_up == true` (tool calls were made, or model set `end_turn=false`), `run_sampling_request` loops back: it rebuilds the prompt with updated history and calls `try_run_sampling_request` again.

Tool futures run concurrently with the stream — they are spawned as Tokio tasks the moment `OutputItemDone` arrives, not after the stream closes. `drain_in_flight` just awaits them in order after the stream ends.

When the model returns a tool call in the stream, the following sequence runs.


**ToolCallRuntime (parallel.rs:31) is the execution engine for a turn.**
```Rust
pub(crate) struct ToolCallRuntime {
    router: Arc<ToolRouter>,
    session: Arc<Session>,
    turn_context: Arc<TurnContext>,
    tracker: SharedTurnDiffTracker,
    parallel_execution: Arc<RwLock<()>>,
}
```
It is created once per `run_sampling_request` call (`turn.rs:988`) and cloned cheaply (all fields are Arc) into each `handle_output_item_done` context. Every tool call that fires within the same sampling round shares the same `ToolCallRuntime` instance — that's how the `parallel_execution` lock is shared across concurrent calls.


**Step 1 — Stream item arrives: `handle_output_item_done` (`stream_events_utils.rs:405`)**

```rust
match ToolRouter::build_tool_call(item) {
    Ok(Some(call)) => {
        // spawn a future on ToolCallRuntime
        let tool_future = ctx.tool_runtime.handle_tool_call(call, cancellation_token);
        output.tool_future = Some(tool_future);
        output.needs_follow_up = true;
    }
    Ok(None) => { /* text/reasoning item — emit EventMsg, not a tool call */ }
}
```

`ToolRouter::build_tool_call` converts the raw `ResponseItem` into a typed `ToolCall { tool_name, call_id, payload }`. Three item shapes are handled:
- `FunctionCall` → `ToolPayload::Function { arguments: String }`
- `CustomToolCall` (code-mode) → `ToolPayload::Function { arguments }`
- `ToolSearchCall` → `ToolPayload::ToolSearch { query }`

**Step 2 — Parallel execution guard: `ToolCallRuntime::handle_tool_call_with_source` (`parallel.rs:82`)**

The tool call is spawned as a Tokio task. Before dispatching, it acquires a lock:
- `supports_parallel_tool_calls() == true` → read lock (multiple can run concurrently)
- `supports_parallel_tool_calls() == false` → write lock (exclusive, serialized)

The task also sets up a `terminal_outcome_reached: AtomicBool` flag used to coordinate cancellation: if the tool finishes before a cancellation arrives, the completed result wins; if cancellation arrives first and `waits_for_runtime_cancellation() == true`, the runtime is allowed to finish teardown before the aborted response is returned.

**Step 3 — Build `ToolInvocation` and enter the registry: `router.rs:195`**

```rust
let invocation = ToolInvocation {
    session, turn, cancellation_token,
    tracker, call_id, tool_name, source, payload,
};
registry.dispatch_any_with_terminal_outcome(invocation, terminal_outcome_reached).await
```

**Step 4 — Dispatch in `ToolRegistry::dispatch_any_with_terminal_outcome` (`registry.rs:406`)**

The full sequence inside this function:

```
1. increment active_turn.tool_calls counter
2. lookup handler: self.tool(&tool_name)  →  Arc<dyn CoreToolRuntime>
   (None → FunctionCallError::RespondToModel "unsupported tool")
3. tool.telemetry_tags(&invocation)       → tags for OTel span
4. tool.matches_kind(&invocation.payload) → payload shape guard
   (mismatch → FunctionCallError::Fatal)
5. notify_tool_start()                    → ToolLifecycleContributor::on_tool_start
6. tool.pre_tool_use_payload()            → PreToolUsePayload (hook input)
   run_pre_tool_use_hooks()
     Blocked  → return Err (tool never runs)
     Continue { updated_input: Some } → tool.with_updated_hook_input() rewrites invocation
     Continue { updated_input: None } → proceed unchanged
7. handle_any_tool(tool, invocation)      → tool.handle(invocation) → Box<dyn ToolOutput>
                                            + tool.post_tool_use_payload()
8. run_post_tool_use_hooks()              → hook can inject feedback_message or stop execution
   if feedback_message → replace tool output visible to model
9. notify_tool_finish()                   → ToolLifecycleContributor::on_tool_finish
10. return AnyToolResult { call_id, payload, result, post_tool_use_payload }
```

**Step 5 — Result back to the turn loop**

`handle_tool_call` (parallel.rs:63) maps `AnyToolResult` to `ResponseInputItem` (e.g. `FunctionCallOutput`) and returns it. The turn loop in `try_run_sampling_request` collects all in-flight tool futures via `drain_in_flight` (`turn.rs:1716`), appends every `ResponseInputItem` to the conversation history, then sends the updated history in the next model request.

**NOTE: A `ToolExecutor` implementation is the complete definition of a tool!**


#### Overview

Three concerns, three types:

**1. What does the tool look like to the model?** → `ToolSpec` (`codex_tools`)
A `#[serde(tag="type")]` enum that serializes directly to Responses API wire format. It is pure data — no behavior. A handler produces one via `spec()`.

**2. What does the tool look like to the process?** → `CoreToolRuntime` (`codex_core`)
A trait object (`Arc<dyn CoreToolRuntime>`) that owns the execution logic. It extends the public `ToolExecutor` trait with core-internal concerns: hooks, cancellation, argument diffs. Concrete structs (`ShellCommandHandler`, `McpHandler`, …) implement this.

**3. How are they assembled per turn?** → `ToolRouter` (`codex_core`)
Built once at the start of every turn by `built_tools`. Holds two things:
- `model_visible_specs: Vec<ToolSpec>` — the outbound advertising list, pre-filtered by `ToolExposure`
- `ToolRegistry: HashMap<ToolName, Arc<dyn CoreToolRuntime>>` — the inbound dispatch table, contains ALL handlers including Hidden ones the model never sees

`ToolExposure` is the single knob that controls which side a handler appears on:

```
Direct          → both sides  (model sees it, registry has it)
DirectModelOnly → both sides  (model sees it, but excluded from code-mode nesting)
Deferred        → registry only  (hidden until model calls tool_search)
Hidden          → registry only  (model never knows it exists)
```

`PlannedTools` is the temporary builder that accumulates handlers during `built_tools` before the split into `model_visible_specs` + `ToolRegistry`. It is discarded after `ToolRouter` is constructed.

The key abstraction boundary: `ToolSpec` and `ToolExecutor` live in the public `codex_tools` crate. `CoreToolRuntime`, `ToolRegistry`, and `ToolRouter` live in `codex_core`. Extensions and MCP adapters only need to implement `ToolExecutor`; core wraps them into `CoreToolRuntime` via `ExtensionToolAdapter` / `McpHandler`.

**`ToolExecutor<Invocation>`** (`tools/src/tool_executor.rs:44`) is the public trait every tool must implement:

```rust
pub trait ToolExecutor<Invocation>: Send + Sync {
    fn tool_name(&self) -> ToolName;
    fn spec(&self) -> ToolSpec;
    fn exposure(&self) -> ToolExposure { ToolExposure::Direct }
    fn supports_parallel_tool_calls(&self) -> bool { false }
    async fn handle(&self, invocation: Invocation) -> Result<Box<dyn ToolOutput>, FunctionCallError>;
}
```

Three things to note:
- **Generic over `Invocation`** — core uses `ToolExecutor<ToolInvocation>`, extensions use `ToolExecutor<ExtensionToolCall>`. Same trait, different call-site types. `ExtensionToolAdapter` bridges the two: it wraps `Arc<dyn ToolExecutor<ExtensionToolCall>>` and adapts it to `ToolExecutor<ToolInvocation>` so core treats all tools uniformly.
- **`CoreToolRuntime` has it as a supertrait** — you cannot implement `CoreToolRuntime` without also implementing `ToolExecutor<ToolInvocation>`. Every handler in the registry implements both.
- **Extensions only implement `ToolExecutor`** — they never see `CoreToolRuntime`. Core wraps them via `ExtensionToolAdapter` to add hook/diff/cancellation behavior with sensible defaults.


#### Tool Call Concurrency Control

Three levels of concurrency control, each serving a different concern:

**Level 1 — Parallel lock (`ToolCallRuntime.parallel_execution: Arc<RwLock<()>>`, `parallel.rs:93`)**

Shared across all tool calls in the same turn. Controls whether calls run concurrently:



```rust
pub trait ToolExecutor<Invocation>: Send + Sync {
    fn supports_parallel_tool_calls(&self) -> bool { false }
    ...
}

supports_parallel_tool_calls() == true  // read lock   → multiple tools run simultaneously
supports_parallel_tool_calls() == false // write lock  → exclusive, blocks all other tools
```

`ShellCommandHandler` returns `true` — shell commands can run in parallel with each other. Most other handlers return `false`. If the model calls a parallel tool and a non-parallel tool simultaneously, the non-parallel tool waits for the write lock until the parallel one releases its read lock.

**Level 2 — `terminal_outcome_reached: AtomicBool` per call (`parallel.rs:100`)**

A flag shared between the dispatch task and the cancellation path for a single tool call. Whoever calls `swap(true, AcqRel)` first owns the terminal outcome:
- Dispatch finishes before cancel → dispatch owns it → completed result returned
- Cancel fires before dispatch finishes → cancellation owns it → aborted result returned
- Both race → `swap` is atomic — exactly one wins, double-reporting is impossible

**Level 3 — Double-lock for turn counter (`registry.rs:434`)**

```rust
session.active_turn.lock()            // outer: guards existence of active turn
    active_turn.turn_state.lock()     // inner: guards mutable counters (tool_calls++)
```

Two separate locks because the outer one is coarse (held briefly, many readers) and the inner one is fine-grained (only for mutation). The `#[expect(clippy::await_holding_invalid_type)]` attribute suppresses Clippy's warning about holding a mutex guard across `.await` — accepted intentionally to keep the counter increment atomic.


### sandbox