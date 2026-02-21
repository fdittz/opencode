# Technology Stack

**Analysis Date:** 2026-02-21

## Languages

**Primary:**

- TypeScript 5.8 - All packages, CLI, web apps, workers, SDK
- TypeScript 7.0 (`tsgo`) - Used for type-checking only via `turbo typecheck`

**Secondary:**

- Rust - Desktop app backend (`packages/desktop/src-tauri/Cargo.toml`), Tauri v2 shell
- Nix - Dev environment and package builds (`flake.nix`)

## Runtime

**Environment:**

- Bun 1.2+ (primary runtime for CLI, dev, build, tests)
- Cloudflare Workers (production API, auth, console, enterprise, log processing)
- Node.js (VS Code extension runtime)
- Tauri v2 (desktop app, Rust runtime wrapping web view)

**Package Manager:**

- Bun (workspace protocol, `bun install`)
- Lockfile: `bun.lock` (present, committed)
- Cargo (Rust deps for desktop, `packages/desktop/src-tauri/Cargo.lock`)

## Frameworks

**Core:**

- Hono 4.x - HTTP server for CLI (`packages/opencode/src/server/server.ts`), Cloudflare Workers (`packages/function/src/api.ts`)
- SolidJS 1.9 - Web app (`packages/app`), TUI (`packages/opencode/src/cli/cmd/tui/`), shared UI (`packages/ui`)
- SolidStart 1.1 - Console app (`packages/console/app`), Enterprise app (`packages/enterprise`)
- Astro 5.x + Starlight 0.34 - Documentation site (`packages/web`)
- Tauri 2.5 - Desktop app wrapper (`packages/desktop`)

**AI/LLM:**

- Vercel AI SDK v5 (`ai` package) - Unified LLM interface across 20+ providers
- `@ai-sdk/*` provider packages - One per AI provider (see Integrations)
- `@modelcontextprotocol/sdk` 1.12 - MCP client for tool servers
- `@agentclientprotocol/sdk` - ACP protocol support

**Testing:**

- Bun test runner (`bun test`) - Unit/integration tests in `packages/opencode`
- Playwright 1.52 - E2E tests for web app (`packages/app/playwright.config.ts`)

**Build/Dev:**

- Turborepo 2.5 - Monorepo task orchestration (`turbo.json`)
- Bun.build - CLI binary compilation with SolidJS plugin (`packages/opencode/script/build.ts`)
- Vite 6.x - Web app dev/build (`packages/app/vite.config.ts`)
- esbuild - VS Code extension bundling (`sdks/vscode`)
- SST v3 (Ion) - Infrastructure-as-code, Cloudflare home (`sst.config.ts`)
- `@hey-api/openapi-ts` - SDK generation from OpenAPI spec (`packages/sdk/js`)

## Key Dependencies

**Critical:**

- `ai` (Vercel AI SDK v5) - Core LLM abstraction, streaming, tool calls. Used in `packages/opencode/src/provider/provider.ts`
- `drizzle-orm` 0.44 - Database ORM for SQLite (CLI) and MySQL (console)
- `hono` 4.x - HTTP framework for server and workers
- `zod` 4.x - Schema validation throughout (config, tools, API, plugins)
- `@openauthjs/openauth` - Authentication for console/cloud services
- `yargs` 17.x - CLI argument parsing (`packages/opencode/src/index.ts`)

**Infrastructure:**

- `@cloudflare/workers-types` - Worker type definitions
- `wrangler` - Cloudflare Workers CLI (dev/deploy)
- `@planetscale/database` - PlanetScale MySQL driver for console
- `drizzle-kit` - Database migration tooling
- `stripe` 18.x - Payment/subscription management (`packages/console/core`)

**UI/TUI:**

- `@opentui/solid` 0.0.6 - Terminal UI framework for SolidJS (`packages/opencode/src/cli/cmd/tui/`)
- `@kobalte/core` 0.13 - Accessible SolidJS UI primitives (`packages/ui`)
- `tailwindcss` 4.x - CSS utility framework (`packages/app`, `packages/ui`)
- `tree-sitter` + language grammars - Code syntax parsing in TUI

**Protocol/Communication:**

- `@modelcontextprotocol/sdk` 1.12 - MCP client (`packages/opencode/src/mcp/index.ts`)
- `vscode-jsonrpc` - LSP client communication (`packages/opencode/src/lsp/`)
- `jose` 6.x - JWT creation for GitHub App auth (`packages/function/src/api.ts`)
- `@octokit/rest` + `@octokit/graphql` - GitHub API (`packages/function`)

**Desktop:**

- `tauri-apps/api` 2.x - Tauri JS bridge (`packages/desktop`)
- `tauri` 2.5 + `tauri-build` - Rust framework (`packages/desktop/src-tauri/Cargo.toml`)
- `tauri-plugin-shell` 2.2 - Shell command execution from desktop app
- `tauri-plugin-deep-link` 2.2 - Custom URL scheme handling
- `tauri-plugin-single-instance` 2.2 - Single instance enforcement

## Configuration

**Environment:**

- SST secrets for cloud services (Stripe keys, GitHub App credentials, Discord tokens, etc.) defined in `infra/console.ts` and `infra/app.ts`
- Local CLI config via `opencode.json` or `opencode.jsonc` in project root
- `.env` files for local development (not committed)
- Nix flake provides reproducible dev environment with Bun, Wrangler, SST, Turborepo

**Build:**

- `turbo.json` - Task pipeline (typecheck, build, dev per package)
- `tsconfig.json` (root) - Extends `@tsconfig/bun`, strict mode, path aliases
- `packages/opencode/tsconfig.json` - JSX preserve, paths: `@opentui/solid`
- `packages/app/vite.config.ts` - SolidJS plugin, Tailwind, proxy to CLI server
- `packages/web/astro.config.mjs` - Starlight theme, Cloudflare adapter, Tailwind
- `sst.config.ts` - SST v3 with Cloudflare home, Stripe provider, PlanetScale provider
- `packages/opencode/drizzle.config.ts` - SQLite dialect, `src/**/*.sql.ts` schema pattern
- `packages/console/core/drizzle.config.ts` - MySQL dialect, PlanetScale

**Formatting/Linting:**

- Prettier - No semicolons, 120 print width (`.prettierrc` at root)
- Husky - Git hooks (`.husky/`)
- No ESLint detected

## Platform Requirements

**Development:**

- Bun 1.2+ (required, primary runtime)
- Nix (optional but recommended, provides all tooling via `flake.nix`)
- Node.js 22+ (for VS Code extension development)
- Rust toolchain (for desktop app development only)
- Wrangler CLI (for Cloudflare Workers development)

**Production - CLI:**

- Standalone Bun binary (no runtime dependency)
- 11 platform targets: linux-x64 (glibc/musl, baseline/avx2), linux-arm64 (glibc/musl), darwin-x64, darwin-arm64, win32-x64 (baseline/avx2), win32-arm64
- Build script: `packages/opencode/script/build.ts`
- Binary includes embedded SQLite migrations and models.dev snapshot

**Production - Cloud:**

- Cloudflare Workers (API, Auth, Console, Enterprise, Functions, Log Processor)
- Cloudflare Pages/Static Sites (web app, docs)
- Cloudflare R2 (file storage)
- Cloudflare KV (auth sessions, config)
- Cloudflare Durable Objects (sync server in `packages/function`)
- PlanetScale MySQL (console database)

---

_Stack analysis: 2026-02-21_
