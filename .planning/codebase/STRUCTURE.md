# Codebase Structure

**Analysis Date:** 2026-02-21

## Directory Layout

```
opencode/
├── packages/                  # All source packages (Bun workspace)
│   ├── opencode/              # Core backend: CLI, server, session, agents, tools, providers
│   ├── app/                   # SolidJS web app (shared between web and desktop)
│   ├── desktop/               # Tauri v2 desktop app wrapping packages/app
│   ├── ui/                    # Reusable SolidJS UI component library
│   ├── sdk/js/                # Auto-generated TypeScript SDK client from OpenAPI
│   ├── plugin/                # Plugin type definitions and hook interfaces (types only)
│   ├── util/                  # Shared utilities (error, slug, identifier, lazy, etc.)
│   ├── enterprise/            # SolidStart app for enterprise features (share pages)
│   ├── web/                   # Astro-based marketing/docs website
│   ├── console/               # Admin console (multi-package: app, core, function, mail, resource)
│   ├── docs/                  # Mintlify documentation content
│   ├── function/              # Serverless API functions
│   ├── script/                # Build/release scripts
│   ├── containers/            # Docker configs for CI/CD
│   ├── identity/              # Brand assets (SVG logos, PNGs)
│   ├── slack/                 # Slack integration
│   └── extensions/            # Editor extensions
├── infra/                     # SST infrastructure definitions (Cloudflare)
│   ├── app.ts                 # App infrastructure
│   ├── console.ts             # Console infrastructure
│   ├── enterprise.ts          # Enterprise infrastructure
│   ├── secret.ts              # Secret definitions
│   └── stage.ts               # Stage configuration
├── sdks/                      # External SDK definitions
├── specs/                     # OpenAPI specs
├── script/                    # Root-level build/release scripts
├── patches/                   # Bun patch files for dependencies
├── nix/                       # Nix build configurations
├── github/                    # GitHub-related configs
├── .opencode/                 # Project-level opencode configuration
├── .github/                   # GitHub Actions workflows
├── .husky/                    # Git hooks
├── sst.config.ts              # SST root config
├── turbo.json                 # Turborepo task configuration
├── package.json               # Root workspace definition
├── tsconfig.json              # Root TypeScript config
├── bunfig.toml                # Bun configuration
├── opencode.json              # Project opencode config
└── bun.lock                   # Bun lockfile
```

## Directory Purposes

**`packages/opencode/`** (Core Backend):

- Purpose: The heart of the application — CLI, HTTP server, AI session orchestration, tools, providers
- Contains: TypeScript source organized by domain namespace
- Key structure:
  ```
  packages/opencode/
  ├── src/
  │   ├── index.ts              # CLI entry point (yargs)
  │   ├── cli/
  │   │   ├── cmd/              # 21 CLI command files
  │   │   │   ├── tui/          # TUI app (SolidJS terminal UI)
  │   │   │   ├── run.ts        # Non-interactive run command
  │   │   │   ├── serve.ts      # HTTP server command
  │   │   │   ├── web.ts        # Browser UI command
  │   │   │   ├── auth.ts       # Authentication command
  │   │   │   ├── agent.ts      # Agent management
  │   │   │   ├── mcp.ts        # MCP server management
  │   │   │   ├── acp.ts        # Add-commit-push workflow
  │   │   │   └── ...
  │   │   ├── bootstrap.ts      # CLI bootstrap
  │   │   ├── ui.ts             # CLI UI utilities
  │   │   └── error.ts          # CLI error formatting
  │   ├── server/
  │   │   ├── server.ts         # Hono HTTP server (623 lines)
  │   │   ├── event.ts          # Server-level bus events
  │   │   ├── error.ts          # Server error helpers
  │   │   ├── mdns.ts           # mDNS discovery
  │   │   └── routes/           # 12 route modules
  │   │       ├── session.ts    # Session CRUD + chat endpoints
  │   │       ├── config.ts     # Config read/write endpoints
  │   │       ├── provider.ts   # Provider/model listing
  │   │       ├── file.ts       # File operations
  │   │       ├── mcp.ts        # MCP server management
  │   │       ├── pty.ts        # PTY/terminal endpoints
  │   │       ├── tui.ts        # TUI-specific endpoints
  │   │       ├── project.ts    # Project operations
  │   │       ├── question.ts   # User question/approval endpoints
  │   │       ├── permission.ts # Permission management
  │   │       ├── global.ts     # Global state endpoints
  │   │       └── experimental.ts # Experimental features
  │   ├── session/
  │   │   ├── index.ts          # Session CRUD (873 lines)
  │   │   ├── prompt.ts         # Chat orchestration (1959 lines)
  │   │   ├── processor.ts      # Stream processing (421 lines)
  │   │   ├── llm.ts            # AI SDK wrapper (279 lines)
  │   │   ├── session.sql.ts    # DB schema (session, message, part tables)
  │   │   ├── message-v2.ts     # Message type definitions
  │   │   ├── system.ts         # System prompt builder
  │   │   ├── instruction.ts    # Instruction prompt
  │   │   ├── compaction.ts     # Message compaction
  │   │   ├── retry.ts          # Retry logic
  │   │   ├── revert.ts         # Session revert
  │   │   ├── summary.ts        # Summary generation
  │   │   ├── status.ts         # Session status tracking
  │   │   └── prompt/           # Prompt template .txt files
  │   ├── agent/
  │   │   ├── agent.ts          # Agent definitions (339 lines)
  │   │   ├── generate.txt      # Generation prompt
  │   │   └── prompt/           # Agent-specific prompt .txt files
  │   ├── tool/
  │   │   ├── tool.ts           # Tool.define() abstraction (89 lines)
  │   │   ├── registry.ts       # ToolRegistry (171 lines)
  │   │   ├── truncation.ts     # Output truncation
  │   │   ├── bash.ts + bash.txt
  │   │   ├── read.ts + read.txt
  │   │   ├── edit.ts + edit.txt
  │   │   ├── write.ts + write.txt
  │   │   ├── glob.ts + glob.txt
  │   │   ├── grep.ts + grep.txt
  │   │   ├── task.ts + task.txt
  │   │   ├── webfetch.ts + webfetch.txt
  │   │   ├── websearch.ts + websearch.txt
  │   │   ├── codesearch.ts + codesearch.txt
  │   │   ├── todo.ts + todo.txt
  │   │   ├── question.ts + question.txt
  │   │   ├── skill.ts + skill.txt
  │   │   ├── apply_patch.ts + apply_patch.txt
  │   │   ├── batch.ts + batch.txt
  │   │   ├── lsp.ts + lsp.txt
  │   │   ├── plan.ts + plan.txt
  │   │   └── invalid.ts
  │   ├── provider/
  │   │   ├── provider.ts       # Provider system (1338 lines)
  │   │   ├── transform.ts      # Provider-specific transforms
  │   │   ├── models.ts         # models.dev integration
  │   │   └── sdk/              # Custom SDK wrappers (e.g., copilot)
  │   ├── config/
  │   │   ├── config.ts         # Config system (1493 lines)
  │   │   └── markdown.ts       # Config from markdown files
  │   ├── storage/
  │   │   ├── db.ts             # Database layer (142 lines)
  │   │   ├── storage.ts        # File storage (220 lines)
  │   │   ├── schema.ts         # Schema barrel export
  │   │   └── json-migration.ts # JSON→SQLite migration
  │   ├── bus/
  │   │   ├── index.ts          # Per-instance event bus (105 lines)
  │   │   ├── bus-event.ts      # Event definition (43 lines)
  │   │   └── global.ts         # Cross-instance EventEmitter (10 lines)
  │   ├── project/
  │   │   ├── instance.ts       # Instance context (114 lines)
  │   │   ├── project.sql.ts    # Project DB schema
  │   │   ├── project.ts        # Project operations
  │   │   ├── state.ts          # State management for Instance.state()
  │   │   ├── vcs.ts            # Git/VCS operations
  │   │   └── bootstrap.ts      # Instance bootstrap
  │   ├── permission/
  │   │   ├── next.ts           # PermissionNext rule system
  │   │   └── permission.sql.ts # Permission DB schema
  │   ├── plugin/
  │   │   ├── index.ts          # Plugin loader (143 lines)
  │   │   ├── codex.ts          # OpenAI Codex auth plugin
  │   │   └── copilot.ts        # GitHub Copilot auth plugin
  │   ├── mcp/                  # Model Context Protocol integration
  │   ├── lsp/                  # LSP client/server
  │   ├── skill/                # Skill discovery from .opencode/skills/
  │   ├── auth/                 # Authentication system
  │   ├── env/                  # Environment variable handling
  │   ├── flag/                 # Feature flags
  │   ├── format/               # Output formatting
  │   ├── global/               # XDG paths (data, config, cache)
  │   ├── id/                   # ID generation (ULID-based)
  │   ├── installation/         # Version/installation info
  │   ├── shell/                # Shell environment
  │   ├── snapshot/             # File snapshot/restore
  │   ├── worktree/             # Git worktree management
  │   ├── acp/                  # Add-commit-push workflow
  │   ├── command/              # Custom command system
  │   ├── control/              # Control flow utilities
  │   ├── file/                 # File time tracking
  │   ├── ide/                  # IDE integration
  │   ├── patch/                # Patch application
  │   ├── pty/                  # PTY/terminal management
  │   ├── question/             # User question/approval flow
  │   ├── scheduler/            # Task scheduling
  │   ├── share/                # Session sharing
  │   ├── sql.d.ts              # SQL type declarations
  │   └── util/
  │       ├── context.ts        # AsyncLocalStorage wrapper (25 lines)
  │       ├── lazy.ts           # Lazy singleton pattern
  │       ├── fn.ts             # Zod-validated function wrapper
  │       ├── log.ts            # Logging system
  │       ├── filesystem.ts     # File system helpers
  │       ├── glob.ts           # Glob utilities
  │       ├── iife.ts           # Immediately invoked function expression helper
  │       ├── defer.ts          # Deferred promise
  │       └── proxied.ts        # Proxy utilities
  ├── migration/                # Drizzle SQL migrations
  ├── drizzle.config.ts         # Drizzle Kit config
  └── package.json
  ```

```

**`packages/app/`** (Web Application):
- Purpose: SolidJS single-page application shared between browser and desktop
- Contains: Pages, context providers, components, hooks, i18n, utilities
- Key structure:
```

packages/app/src/
├── app.tsx # App shell with provider tree (171 lines)
├── entry.tsx # App entry point
├── index.ts # Public exports
├── index.css # Global styles
├── context/ # 35 SolidJS context providers
│ ├── server.tsx # Server connection management
│ ├── global-sdk.tsx # SDK client provider
│ ├── global-sync.tsx # SSE event sync
│ ├── sync.tsx # Data sync utilities
│ ├── settings.tsx # User settings
│ ├── models.tsx # Model listing
│ ├── command.tsx # Command palette
│ ├── layout.tsx # Layout state
│ ├── prompt.tsx # Prompt input state
│ ├── file.tsx # File browser state
│ ├── terminal.tsx # Terminal state
│ ├── permission.tsx # Permission state
│ ├── notification.tsx # Notifications
│ ├── highlights.tsx # Syntax highlighting
│ ├── language.tsx # i18n
│ ├── comments.tsx # Comment threads
│ ├── platform.tsx # Platform capabilities
│ ├── local.tsx # Local storage
│ └── ...
├── components/ # 36 component files
│ ├── session/ # Session-related components
│ ├── prompt-input/ # Prompt input sub-components
│ ├── server/ # Server-related components
│ ├── titlebar.tsx # App title bar
│ ├── dialog-settings.tsx # Settings dialog
│ ├── dialog-select-model.tsx
│ └── ...
├── pages/
│ ├── home.tsx # Home page
│ ├── session.tsx # Session page
│ ├── layout.tsx # Page layout
│ ├── directory-layout.tsx # Directory-scoped layout
│ └── error.tsx # Error page
├── hooks/ # Custom SolidJS hooks
├── addons/ # App addons
├── i18n/ # Internationalization
└── utils/ # Frontend utilities

```

**`packages/desktop/`** (Desktop Application):
- Purpose: Tauri v2 desktop app wrapping `packages/app` with native capabilities
- Contains: Tauri entry point, native bindings, updater, CLI integration
- Key files:
- `packages/desktop/src/index.tsx` (509 lines) — Main entry, server management, deep links, WSL support
- `packages/desktop/src/entry.tsx` — Router entry point
- `packages/desktop/src/bindings.ts` — Tauri API bindings
- `packages/desktop/src/cli.ts` — CLI argument handling
- `packages/desktop/src/updater.ts` — Auto-update logic
- `packages/desktop/src/menu.ts` — Native menu
- `packages/desktop/src-tauri/` — Rust backend

**`packages/ui/`** (Component Library):
- Purpose: 50+ reusable SolidJS UI components
- Contains: Components with CSS modules pattern (`.tsx` + `.css` pairs)
- Key components: `button`, `dialog`, `diff`, `code`, `markdown`, `toast`, `tabs`, `select`, `text-field`, `spinner`, `accordion`, `context-menu`, `dropdown-menu`, `popover`, `hover-card`, `list`, `checkbox`, `switch`, `radio-group`, `progress`, `scroll-view`, `resize-handle`, `session-turn`, `session-review`, `message-part`, `message-nav`, `dock-surface`, `dock-prompt`
- Also contains: `context/` directory with shared UI contexts (dialog, diff, code, marked, i18n, theme)

**`packages/sdk/js/`** (TypeScript SDK):
- Purpose: Auto-generated client SDK from OpenAPI spec
- Contains: Generated API client, type definitions
- Generation: Run `./packages/sdk/js/script/build.ts` to regenerate
- Used by: `packages/app`, `packages/plugin`, `packages/enterprise`

**`packages/plugin/`** (Plugin Types):
- Purpose: Type definitions for the plugin hook system (no runtime code)
- Key file: `packages/plugin/src/index.ts` (234 lines) — defines `Plugin`, `Hooks`, `PluginInput`, `ToolDefinition`, `AuthHook`, and all hook types

**`packages/util/`** (Shared Utilities):
- Purpose: Framework-agnostic utility functions shared across packages
- Key files: `error.ts` (NamedError), `slug.ts`, `identifier.ts`, `lazy.ts`

## Key File Locations

**Entry Points:**
- `packages/opencode/src/index.ts`: CLI entry point
- `packages/opencode/src/server/server.ts`: HTTP server
- `packages/app/src/app.tsx`: Web app shell
- `packages/desktop/src/index.tsx`: Desktop app entry
- `sst.config.ts`: Infrastructure entry

**Configuration:**
- `opencode.json`: Project-level opencode config
- `package.json`: Root workspace + scripts
- `turbo.json`: Turborepo task pipeline
- `tsconfig.json`: Root TypeScript config
- `bunfig.toml`: Bun settings
- `packages/opencode/drizzle.config.ts`: Drizzle ORM config

**Core Logic:**
- `packages/opencode/src/session/prompt.ts`: Chat orchestration (largest file, 1959 lines)
- `packages/opencode/src/provider/provider.ts`: Provider system (1338 lines)
- `packages/opencode/src/config/config.ts`: Config system (1493 lines)
- `packages/opencode/src/session/index.ts`: Session CRUD (873 lines)

**Database Schemas:**
- `packages/opencode/src/session/session.sql.ts`: Session, Message, Part tables
- `packages/opencode/src/project/project.sql.ts`: Project table
- `packages/opencode/src/permission/permission.sql.ts`: Permission table
- `packages/opencode/src/tool/todo.sql.ts`: Todo table

**Migrations:**
- `packages/opencode/migration/`: Drizzle SQL migrations (each in `<timestamp>_<slug>/migration.sql`)

## Naming Conventions

**Files:**
- Domain modules: `kebab-case.ts` (e.g., `bus-event.ts`, `message-v2.ts`, `json-migration.ts`)
- Database schemas: `<entity>.sql.ts` (e.g., `session.sql.ts`, `project.sql.ts`)
- Tool implementations: `<tool-name>.ts` + `<tool-name>.txt` (e.g., `bash.ts` + `bash.txt`)
- Prompt templates: `<name>.txt` imported as string constants
- Tests: `<name>.test.ts` (co-located with source)
- UI components: `kebab-case.tsx` + `kebab-case.css` (CSS modules)

**Directories:**
- Domain-based: `session/`, `agent/`, `tool/`, `provider/`, `config/`, `storage/`, `bus/`
- Single-word preferred: `auth/`, `env/`, `flag/`, `mcp/`, `lsp/`, `shell/`
- Plural for collections: `routes/`, `components/`, `context/`, `hooks/`, `pages/`

**TypeScript Patterns:**
- Namespaces: PascalCase matching the domain (e.g., `export namespace Session`, `export namespace Bus`)
- Functions within namespaces: camelCase (e.g., `Session.create()`, `Bus.publish()`)
- Zod schemas as types: `export const Info = z.object({...})` followed by `export type Info = z.infer<typeof Info>`
- State initialization: `const state = Instance.state(() => { ... })`
- Singletons: `const Client = lazy(() => { ... })`

## Where to Add New Code

**New Tool:**
- Implementation: `packages/opencode/src/tool/<name>.ts`
- Description: `packages/opencode/src/tool/<name>.txt`
- Registration: Add import to `packages/opencode/src/tool/registry.ts` and include in `all()` array
- Pattern: Use `Tool.define(id, init)` — see `packages/opencode/src/tool/glob.ts` for a simple example

**New Agent:**
- Built-in: Add to the `result` record in `packages/opencode/src/agent/agent.ts`
- Custom (user-facing): Create `.opencode/agents/<name>.md` markdown file
- Pattern: Define `name`, `description`, `mode`, `permission` ruleset, optional `model` and `prompt`

**New Server Route:**
- Route file: `packages/opencode/src/server/routes/<name>.ts`
- Registration: Import and mount in `packages/opencode/src/server/server.ts`
- Pattern: Use `describeRoute` + `validator` from `hono-openapi` for typed routes with OpenAPI docs

**New CLI Command:**
- Command file: `packages/opencode/src/cli/cmd/<name>.ts`
- Registration: Import and add `.command()` in `packages/opencode/src/index.ts`
- Pattern: Export a yargs `CommandModule`

**New Database Table:**
- Schema: `packages/opencode/src/<domain>/<entity>.sql.ts`
- Export: Add to `packages/opencode/src/storage/schema.ts` barrel
- Migration: Run `bun run db generate --name <slug>` from `packages/opencode/`
- Pattern: Use snake_case columns, `<entity>_id` for foreign keys, Drizzle ORM

**New UI Component:**
- Component: `packages/ui/src/components/<name>.tsx`
- Styles: `packages/ui/src/components/<name>.css` (CSS modules)
- Export: Add to `packages/ui/package.json` exports map
- Pattern: SolidJS functional component with CSS module imports

**New App Context Provider:**
- Provider: `packages/app/src/context/<name>.tsx`
- Mount: Add to provider tree in `packages/app/src/app.tsx` (in appropriate group: base, shell, or session)
- Pattern: SolidJS context with `createContext` + `createStore` + provider component

**New App Component:**
- Component: `packages/app/src/components/<name>.tsx`
- Pattern: SolidJS component using contexts from `packages/app/src/context/`

**New Plugin:**
- Types: Follow `packages/plugin/src/index.ts` hook interfaces
- Runtime: Publish to npm or use `file://` path in `opencode.json` `plugin` array
- Pattern: Export a function matching `Plugin` type that returns `Hooks`

**New Bus Event:**
- Define: `BusEvent.define("domain.event.name", zodSchema)` in the relevant domain module
- Publish: `Bus.publish(EventDef, properties)` after relevant state changes
- Subscribe: `Bus.subscribe(EventDef, callback)` where needed

**New Utility:**
- Shared (cross-package): `packages/util/src/<name>.ts`
- Backend-only: `packages/opencode/src/util/<name>.ts`
- Frontend-only: `packages/app/src/utils/<name>.ts`

## Special Directories

**`.opencode/`** (Project Configuration):
- Purpose: Project-level customization — agents, commands, skills, tools, plugins, config
- Generated: No (user-created)
- Committed: Yes (project-specific)
- Subdirs: `agents/`, `commands/`, `skills/`, `tools/`, `plugins/`

**`packages/opencode/migration/`** (Database Migrations):
- Purpose: Drizzle SQL migrations for SQLite schema changes
- Generated: Yes (by `bun run db generate --name <slug>`)
- Committed: Yes
- Structure: `<timestamp>_<slug>/migration.sql` + `snapshot.json`

**`packages/sdk/js/src/`** (Generated SDK):
- Purpose: Auto-generated TypeScript SDK from OpenAPI spec
- Generated: Yes (by `./packages/sdk/js/script/build.ts`)
- Committed: Yes

**`infra/`** (Infrastructure):
- Purpose: SST (Serverless Stack) infrastructure definitions for Cloudflare deployment
- Generated: No
- Committed: Yes

**`.planning/`** (GSD Planning):
- Purpose: Planning documents for GSD workflow
- Generated: By GSD agents
- Committed: Optional

---

*Structure analysis: 2026-02-21*
```
