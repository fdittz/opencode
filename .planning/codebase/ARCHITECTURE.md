# Architecture

**Analysis Date:** 2026-02-21

## Pattern Overview

**Overall:** Monorepo with a namespace-oriented backend core, provider-heavy SolidJS frontend, and Hono HTTP + SSE server bridging them.

**Key Characteristics:**

- TypeScript `namespace` as the primary code organization unit — every module exports a namespace (e.g., `Session`, `Bus`, `Provider`, `Agent`, `Config`, `Tool`)
- Zod-first schema definitions with `.meta({ ref })` for OpenAPI generation
- `AsyncLocalStorage`-based context system (`Context.create<T>()`) for request-scoped state
- `Instance.state()` for per-project-directory singletons with lifecycle management
- Event-driven communication via `Bus` (per-instance pub/sub) and `GlobalBus` (cross-instance `EventEmitter`)
- SQLite + Drizzle ORM with transactional context and deferred side effects

## Layers

**CLI Layer:**

- Purpose: Parse commands, bootstrap the runtime, invoke server/session logic
- Location: `packages/opencode/src/cli/`
- Contains: yargs command definitions, TUI app (SolidJS terminal UI)
- Entry point: `packages/opencode/src/index.ts` (203 lines, yargs CLI with ~21 commands)
- Key commands: `run` (non-interactive), `serve` (HTTP server), `web` (browser UI), `acp` (add-commit-push), `agent`, `auth`, `mcp`, `tui/attach`, `tui/thread`
- Depends on: Server, Session, Config, Instance

**Server Layer:**

- Purpose: HTTP API for all clients (web app, desktop, TUI, SDK)
- Location: `packages/opencode/src/server/server.ts` (623 lines)
- Contains: Hono app with CORS, basic auth, SSE streaming, OpenAPI spec generation, 12 route modules
- Routes: `packages/opencode/src/server/routes/` — `session.ts`, `config.ts`, `provider.ts`, `file.ts`, `mcp.ts`, `pty.ts`, `tui.ts`, `project.ts`, `question.ts`, `permission.ts`, `global.ts`, `experimental.ts`
- Depends on: Instance, Bus, Config, Session, Provider, Agent, MCP, LSP
- Used by: SDK client (`packages/sdk/js`), web app (`packages/app`), desktop app (`packages/desktop`)

**Session Layer (Core Orchestration):**

- Purpose: Manages conversations, LLM interactions, tool execution, and message storage
- Location: `packages/opencode/src/session/`
- Contains: Session CRUD, prompt orchestration, stream processing, LLM wrapper, compaction, retry, revert, summary, status
- Key files:
  - `packages/opencode/src/session/index.ts` (873 lines) — Session CRUD, message/part management, events
  - `packages/opencode/src/session/prompt.ts` (1959 lines) — Orchestrates the full chat loop: builds system prompt, registers tools, manages abort, calls LLM, processes results
  - `packages/opencode/src/session/processor.ts` (421 lines) — Processes LLM stream events (text, reasoning, tool calls), handles doom loop detection, retry, compaction triggers
  - `packages/opencode/src/session/llm.ts` (279 lines) — Wraps Vercel AI SDK `streamText()`, applies provider transforms, plugin hooks
- Depends on: Agent, Provider, Tool, Bus, Database, Config, Instance, Plugin, MCP, LSP, Permission

**Agent Layer:**

- Purpose: Define named agents with permissions, prompts, and model overrides
- Location: `packages/opencode/src/agent/agent.ts` (339 lines)
- Contains: Built-in agents (build, plan, general, explore, compaction, title, summary) and custom agent loading from `.opencode/agents/` markdown files
- Key agents:
  - `build` — Default agent, full tool access, permission-gated
  - `plan` — Read-only agent, edit tools denied except to plan files
  - `general` — Subagent for parallel multi-step tasks
  - `explore` — Subagent for codebase exploration (read-only tools)
  - `compaction`, `title`, `summary` — Internal utility agents (hidden)
- Depends on: Config, Provider, Instance, Permission, Plugin, Skill

**Tool Layer:**

- Purpose: Define and register tools available to agents during LLM interactions
- Location: `packages/opencode/src/tool/`
- Contains: 20+ built-in tools, each as a `.ts` file with a companion `.txt` prompt description file
- Key abstraction: `Tool.define(id, init)` in `packages/opencode/src/tool/tool.ts` (89 lines) — validates parameters with Zod, auto-truncates output
- Registry: `packages/opencode/src/tool/registry.ts` (171 lines) — collects built-in tools, custom tools from `.opencode/tools/`, and plugin tools; filters by model capabilities
- Built-in tools: `bash`, `read`, `edit`, `write`, `glob`, `grep`, `task`, `webfetch`, `websearch`, `codesearch`, `todo`, `question`, `skill`, `apply_patch`, `batch`, `lsp`, `plan_enter`, `plan_exit`, `invalid`
- Depends on: Agent, Config, Instance, Plugin, Permission

**Provider Layer:**

- Purpose: Wrap 20+ AI SDK providers, manage model discovery, authentication, and configuration
- Location: `packages/opencode/src/provider/provider.ts` (1338 lines)
- Contains: Provider factory with bundled AI SDK providers, model catalog from `models.dev`, custom provider support, auth integration
- Bundled providers: Anthropic, OpenAI, Azure, Google (Gemini + Vertex), Bedrock, OpenRouter, xAI, Mistral, Groq, DeepInfra, Cerebras, Cohere, Gateway, TogetherAI, Perplexity, Vercel, GitLab, GitHub Copilot
- Depends on: Config, Auth, Instance, Plugin, ModelsDev

**Config Layer:**

- Purpose: Multi-layered configuration with precedence resolution
- Location: `packages/opencode/src/config/config.ts` (1493 lines)
- Contains: Config loading, merging, JSONC parsing, env/file interpolation, config markdown support
- Precedence (low → high): remote `.well-known/opencode` → global `~/.config/opencode/opencode.json{,c}` → custom (`OPENCODE_CONFIG`) → project `opencode.json{,c}` → `.opencode/` directories → inline (`OPENCODE_CONFIG_CONTENT`) → managed (`/etc/opencode`)
- Depends on: Instance, Global, Auth, Plugin, Flag

**Storage Layer:**

- Purpose: SQLite database and file storage
- Location: `packages/opencode/src/storage/`
- Key files:
  - `packages/opencode/src/storage/db.ts` (142 lines) — Database singleton via `lazy()`, Drizzle ORM on `bun:sqlite`, WAL mode, migration runner
  - `packages/opencode/src/storage/storage.ts` (220 lines) — File-based storage for session artifacts
  - `packages/opencode/src/storage/schema.ts` — Re-exports all `*.sql.ts` schemas
- Pattern: `Database.use(callback)` for transactional access; `Database.effect(fn)` for deferred side effects (e.g., bus events published after commit)
- Depends on: Global, Context

**Event Bus Layer:**

- Purpose: Decouple components via pub/sub events, stream events to clients via SSE
- Location: `packages/opencode/src/bus/`
- Key files:
  - `packages/opencode/src/bus/index.ts` (105 lines) — Per-instance event bus using `Instance.state()`, supports typed subscriptions and wildcard
  - `packages/opencode/src/bus/bus-event.ts` (43 lines) — `BusEvent.define(type, zodSchema)` for typed event definitions with OpenAPI integration
  - `packages/opencode/src/bus/global.ts` (10 lines) — `GlobalBus` as a Node.js `EventEmitter` for cross-instance communication
- Pattern: `Bus.publish(EventDef, properties)` → local subscribers + `GlobalBus.emit("event", ...)` → SSE `/event` endpoint → all connected clients
- Depends on: Instance, Log

**Permission Layer:**

- Purpose: Control tool access per agent with pattern-based rules
- Location: `packages/opencode/src/permission/`
- Contains: `PermissionNext` — rule-based system with `allow`/`ask`/`deny` per tool, with glob-pattern matching for file paths and external directories
- Pattern: Agent permissions are merged from defaults → agent-specific → user config overrides
- Depends on: Config

**Plugin Layer:**

- Purpose: Extend opencode with hooks, custom tools, and auth providers
- Location (runtime): `packages/opencode/src/plugin/index.ts` (143 lines) — loads internal plugins + npm plugins + config plugins
- Location (types): `packages/plugin/src/index.ts` (234 lines) — hook type definitions
- Hook points: `chat.message`, `tool.execute.before`, `tool.execute.after`, `tool.definition`, `shell.env`, `auth.provider`, `model.filter`, `provider.create`, and more
- Built-in internal plugins: `CodexAuthPlugin`, `CopilotAuthPlugin`, `GitlabAuthPlugin`
- Depends on: Config, Instance, Server, Bus, SDK

**Frontend Layer (Web/Desktop):**

- Purpose: SolidJS web application shared between browser and desktop
- Location: `packages/app/` (shared SolidJS app), `packages/desktop/` (Tauri v2 wrapper), `packages/ui/` (component library)
- Pattern: Provider-heavy architecture — 15+ SolidJS context providers in nested tree
- Provider groups in `packages/app/src/app.tsx`:
  - Base: `MetaProvider` → `ThemeProvider` → `LanguageProvider` → `DialogProvider` → `MarkedProvider` → `DiffComponentProvider` → `CodeComponentProvider`
  - Shell: `SettingsProvider` → `PermissionProvider` → `LayoutProvider` → `NotificationProvider` → `ModelsProvider` → `CommandProvider` → `HighlightsProvider`
  - Session: `TerminalProvider` → `FileProvider` → `PromptProvider` → `CommentsProvider`
  - Server: `ServerProvider` → `GlobalSDKProvider` → `GlobalSyncProvider`
- Routes: `/` (home), `/:dir/session/:id?` (session)
- Communication: Uses `@opencode-ai/sdk` HTTP client + SSE for real-time updates
- Desktop: `packages/desktop/` wraps the app with Tauri v2 (Rust backend at `packages/desktop/src-tauri/`) for native clipboard, file dialogs, notifications, auto-updater, deep links, WSL support
- Component library: `packages/ui/src/components/` — 50+ reusable SolidJS components with CSS modules (`.tsx` + `.css` pairs)
- Depends on: SDK (`packages/sdk/js`)

## Data Flow

**Chat Message Flow:**

1. Client sends user message via HTTP POST to `/session/:id/message` (or SDK call)
2. `SessionRoutes` handler calls `Session.chat()` which delegates to `SessionPrompt.chat()`
3. `SessionPrompt.chat()` builds system prompt (agent prompt + instructions + context), collects tools from `ToolRegistry`, creates `LLM.StreamInput`
4. `SessionProcessor.create()` creates a processor that calls `LLM.stream()` in a loop
5. `LLM.stream()` wraps Vercel AI SDK `streamText()` with provider transforms and plugin hooks
6. `SessionProcessor` iterates the stream, creating `MessageV2` parts (text, reasoning, tool calls) and persisting them via `Session.updatePart()`
7. Each part update triggers `Bus.publish(Session.Event.PartUpdated, ...)` → `GlobalBus` → SSE → client
8. For tool calls: tool is executed, result stored as tool result part, loop continues for next LLM turn
9. On completion: assistant message finalized, summary/title generated asynchronously

**SSE Event Flow:**

1. Client connects to `/event` SSE endpoint
2. Server subscribes to `GlobalBus` "event" channel
3. All `Bus.publish()` calls emit to `GlobalBus` with directory context
4. SSE stream filters events by client's directory context and streams as JSON

**State Management:**

- Backend: `Instance.state()` creates per-directory singletons managed by `State.create()`. Disposal cleans up all state for an instance.
- Backend: `Context.create<T>()` wraps `AsyncLocalStorage` for request-scoped data (Instance context, Database transactions)
- Frontend: SolidJS context providers + `createStore` for reactive state. `GlobalSyncProvider` syncs server SSE events into local stores.

## Key Abstractions

**Instance:**

- Purpose: Per-project-directory execution context — provides `directory`, `worktree`, and `project` via `AsyncLocalStorage`
- Location: `packages/opencode/src/project/instance.ts` (114 lines)
- Pattern: `Instance.provide({ directory, fn })` wraps execution in context; `Instance.state(init, dispose?)` creates directory-scoped singletons
- Used by: Nearly every module (Bus, Session, Config, ToolRegistry, Agent, Plugin, etc.)

**Session:**

- Purpose: Represents a conversation with messages, parts, and metadata
- Location: `packages/opencode/src/session/index.ts`
- Pattern: Namespace with CRUD functions (`create`, `get`, `list`, `remove`, `rename`, `archive`) plus message/part management
- Database: `SessionTable`, `MessageTable`, `PartTable` in `packages/opencode/src/session/session.sql.ts`

**Tool:**

- Purpose: Represents an executable capability available to agents
- Location: `packages/opencode/src/tool/tool.ts`
- Pattern: `Tool.define(id, init)` where `init` returns `{ description, parameters, execute }`. Parameters validated with Zod. Output auto-truncated.
- Each tool has a `.txt` description file read at init time for the LLM prompt

**BusEvent:**

- Purpose: Typed event definitions for the pub/sub system
- Location: `packages/opencode/src/bus/bus-event.ts`
- Pattern: `BusEvent.define("type.name", zodSchema)` creates a typed event. Published via `Bus.publish(def, properties)`. Events auto-registered for OpenAPI spec generation.

**Database:**

- Purpose: Transactional database access with deferred effects
- Location: `packages/opencode/src/storage/db.ts`
- Pattern: `Database.use(callback)` for read/write; `Database.transaction(callback)` for explicit transactions; `Database.effect(fn)` queues side effects (like bus events) to run after transaction commits

## Entry Points

**CLI:**

- Location: `packages/opencode/src/index.ts`
- Triggers: `opencode <command>` from terminal
- Responsibilities: Parse args, init logging, run JSON→SQLite migration if needed, dispatch to command handler

**HTTP Server:**

- Location: `packages/opencode/src/server/server.ts`
- Triggers: `opencode serve` or auto-started by `opencode web`/`opencode dev`
- Responsibilities: Serve REST API, SSE events, OpenAPI spec, proxy TUI websockets

**TUI:**

- Location: `packages/opencode/src/cli/cmd/tui/`
- Triggers: `opencode` (default command via `attach`), or `opencode thread`
- Responsibilities: Terminal-based SolidJS UI with ink-like rendering

**Web App:**

- Location: `packages/app/src/app.tsx`
- Triggers: Browser navigation to the app URL
- Responsibilities: SolidJS SPA with routing, provider tree, SDK communication

**Desktop App:**

- Location: `packages/desktop/src/index.tsx` (509 lines)
- Triggers: Native app launch
- Responsibilities: Tauri v2 shell wrapping `packages/app`, manages server lifecycle, native integrations

## Error Handling

**Strategy:** Named errors with structured data, propagated through namespaced error classes

**Patterns:**

- `NamedError.create(name, zodSchema)` in `packages/util/src/error.ts` — creates typed error classes with `.toObject()` serialization
- Server returns `NamedError` as JSON with appropriate HTTP status codes (404 for `NotFoundError`, 400 for model/worktree errors, 500 otherwise)
- `Database.effect()` defers side effects to avoid publishing events from failed transactions
- CLI catches all errors in top-level try/catch, logs them, and formats for display via `FormatError()`
- Tools validate parameters with Zod and throw descriptive errors on validation failure
- `SessionProcessor` has doom loop detection (threshold of 3 consecutive same-tool failures)

## Cross-Cutting Concerns

**Logging:**

- `Log.create({ service })` creates service-scoped loggers
- Logs to file at `Global.Path.data` (XDG data directory)
- Supports levels: DEBUG, INFO, WARN, ERROR
- `log.time(label)` for timing measurements
- Initialization in CLI middleware (`Log.init()`)

**Validation:**

- Zod schemas for all data types, tool parameters, config, events
- `fn()` wrapper in `packages/opencode/src/util/fn.ts` validates function inputs with Zod before execution
- Tool parameters validated in `Tool.define()` before execution

**Authentication:**

- Multi-provider auth system in `packages/opencode/src/auth/`
- Plugin-based auth hooks for OAuth and API key flows
- Env var resolution for provider API keys
- Built-in auth plugins: Codex (OpenAI), Copilot (GitHub), GitLab

**Configuration:**

- 7-layer precedence (remote → global → custom → project → .opencode → inline → managed)
- JSONC support, env/file interpolation in values
- Config directories (`.opencode/`) for agents, commands, skills, tools, plugins
- `Config.get()` returns merged config; `Config.state` is per-instance

---

_Architecture analysis: 2026-02-21_
