# Codebase Concerns

**Analysis Date:** 2026-02-21

## Tech Debt

**Pervasive `any` Type Usage (~142 occurrences across core package):**

- Issue: The `any` type is used extensively across the codebase, defeating TypeScript's type safety. Highest-density files include `src/provider/provider.ts` (16), `src/util/log.ts` (11), `src/session/prompt.ts` (8), `src/plugin/copilot.ts` (8), and `src/storage/json-migration.ts` (6).
- Files:
  - `packages/opencode/src/provider/provider.ts` — `BUNDLED_PROVIDERS` typed as `Record<string, (options: any) => SDK>`, `CustomModelLoader` typed `(sdk: any, modelID: string, options?: Record<string, any>) => Promise<any>`, and every `getModel` function parameter is `sdk: any`
  - `packages/opencode/src/plugin/copilot.ts` — body parsing uses `(msg: any)`, `(part: any)`, `(item: any)`, `(nested: any)` in deeply nested array checks
  - `packages/opencode/src/storage/json-migration.ts` — `insert(values: any[], table: any, label: string)`, migration value arrays typed as `any[]`
  - `packages/opencode/src/util/log.ts` — logger interface uses `any` for all message params
  - `packages/opencode/src/util/rpc.ts` — RPC methods typed as `(input: any) => any`
  - `packages/opencode/src/bus/index.ts` — `Subscription = (event: any) => void`, `subscribeAll(callback: (event: any) => void)`
  - `packages/opencode/src/util/eventloop.ts` — `(process as any)._getActiveHandles()` used 4 times, relies on Node internals
- Impact: Type errors silently pass through. Refactoring is risky without type guards. The `any` in provider/plugin code means SDK method signatures are not verified at compile time.
- Fix approach: Introduce proper generics for provider SDK types. Add typed interfaces for Copilot API request/response bodies. Type the bus event system with discriminated unions. Replace `any[]` migration arrays with typed schemas.

**20 TypeScript Suppressions (`@ts-ignore` / `@ts-expect-error`):**

- Issue: Type system bypassed in 20 locations across core source files. Some are documented workarounds for upstream bugs; others are typing challenges that were abandoned.
- Files:
  - `packages/opencode/src/plugin/index.ts:115` — `// @ts-expect-error if you feel adventurous, please fix the typing, make sure to bump the try-counter if you give up. try-counter: 2`
  - `packages/opencode/src/plugin/index.ts:131` — plugin not moved to SDK v2
  - `packages/opencode/src/server/server.ts:44` — global monkey-patch to suppress AI SDK warnings
  - `packages/opencode/src/session/prompt.ts:49` — duplicate AI SDK warning suppression
  - `packages/opencode/src/provider/provider.ts:108` — `@ts-ignore (TODO: kill this code so we dont have to maintain it)`
  - `packages/opencode/src/provider/provider.ts:800,806,1108` — various type mismatches
  - `packages/opencode/src/provider/models.ts:12,91` — suppressed type checks
  - `packages/opencode/src/lsp/client.ts:47-48` — stream type incompatibility
  - `packages/opencode/src/ui/components/diff-ssr.tsx:251` — accessing private property
- Impact: Compile-time safety holes. Upstream library changes could break these silently.
- Fix approach: Address plugin typing (try-counter: 3). Move plugin system to SDK v2 to remove `index.ts:131`. Create proper type wrappers for LSP stream types. Track Bun bug fixes for `bun/issues/19936` and `bun/issues/16682`.

**Env Module vs `process.env` Split:**

- Issue: `Env` module (`packages/opencode/src/env/index.ts`) creates a shallow copy of `process.env` per instance for test isolation. However, multiple providers bypass it with direct `process.env` access because `Env.set` only updates the copy, not `process.env`.
- Files:
  - `packages/opencode/src/env/index.ts` — shallow copy design
  - `packages/opencode/src/provider/provider.ts:229-236` — `process.env.AWS_BEARER_TOKEN_BEDROCK = auth.key` with TODO comment
  - `packages/opencode/src/provider/provider.ts:434-440` — `process.env.AICORE_SERVICE_KEY = auth.key` with TODO comment
  - `packages/opencode/src/flag/flag.ts` — 20+ direct `process.env[]` accesses
- Impact: Environment mutations leak across instances. The `Env` abstraction is partially bypassed, creating inconsistent behavior between test and production. Credentials written to `process.env` are visible to all code paths.
- Fix approach: Clarify Env API scope (test-only vs runtime). Either make Env.set update process.env, or make all provider code go through the Env abstraction. Consolidate flag.ts to use Env where appropriate.

**Deprecated Config Fields Still Active:**

- Issue: Multiple deprecated configuration fields remain in the schema and require migration logic.
- Files:
  - `packages/opencode/src/config/config.ts:695` — `tools` field deprecated in favor of `permission`
  - `packages/opencode/src/config/config.ts:717` — `maxSteps` deprecated in favor of `steps`
  - `packages/opencode/src/config/config.ts:1046` — old `share` field
  - `packages/opencode/src/config/config.ts:1079` — `mode` deprecated in favor of `agent`
  - `packages/opencode/src/config/config.ts:1165` — `layout` deprecated (always stretch)
  - `packages/opencode/src/config/config.ts:208` — migration code for `mode` to `agent`
  - `packages/opencode/src/session/status.ts:35,67` — deprecated status fields
- Impact: Config schema bloat. Migration code adds complexity. Users may still reference deprecated fields causing confusion.
- Fix approach: Set a deprecation timeline. Remove deprecated fields in a major version bump. Emit warnings when deprecated fields are used.

**Pre-release Dependencies:**

- Issue: Core database layer relies on beta releases of Drizzle ORM.
- Files:
  - `packages/opencode/package.json:47` — `"drizzle-kit": "1.0.0-beta.12-a5629fb"`
  - `packages/opencode/package.json:48` — `"drizzle-orm": "1.0.0-beta.12-a5629fb"`
  - `packages/opencode/package.json:78` — `"@clack/prompts": "1.0.0-alpha.1"`
- Impact: Beta dependencies may have breaking changes, missing features, or unfixed bugs. The specific commit hash pin (`a5629fb`) suggests a custom fork or snapshot, making upgrades harder.
- Fix approach: Track Drizzle ORM 1.0 stable release. Pin to stable once available. Evaluate `@clack/prompts` alternatives if alpha stalls.

**Commented-Out Code and Abandoned Features:**

- Issue: Several blocks of commented-out code remain, indicating incomplete features or deferred decisions.
- Files:
  - `packages/opencode/src/plugin/copilot.ts:43-55` — Messages API code commented out, awaiting higher rate limits
  - `packages/opencode/src/permission/next.ts:227-229` — Permission ruleset persistence disabled, awaiting UI
  - `packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-chat-language-model.ts:374` — lost type safety on Chunk, marked MUST FIX
  - `packages/opencode/src/app/src/components/settings-commands.tsx:5` — placeholder, not implemented
  - `packages/opencode/src/app/src/components/settings-agents.tsx:5` — placeholder, not implemented
  - `packages/opencode/src/app/src/components/settings-mcp.tsx:5` — placeholder, not implemented
- Impact: Commented code misleads contributors. Missing permission persistence means "always allow" rules are lost on restart. Placeholder settings pages are non-functional.
- Fix approach: Remove dead commented code. Implement permission persistence with simple file storage. Complete settings pages or remove from navigation.

## Known Bugs

**Filesystem.contains Symlink Bypass:**

- Symptoms: Symlinks inside the project directory can escape the project sandbox. On Windows, cross-drive paths bypass the check entirely.
- Files:
  - `packages/opencode/src/file/index.ts:499-500` — TODO comment documenting the issue in `read()`
  - `packages/opencode/src/file/index.ts:575-576` — Same issue in `list()`
- Trigger: Create a symlink inside the project pointing to a file outside the project directory. The `read()` and `list()` functions will follow it.
- Workaround: None documented.

**Context Overflow Not Handled:**

- Symptoms: When LLM context overflows, the error is caught but the TODO indicates handling is incomplete.
- Files:
  - `packages/opencode/src/session/processor.ts:357` — `// TODO: Handle context overflow error`
- Trigger: Send enough context to exceed the model's token limit during a session.
- Workaround: Compaction should trigger before overflow, but edge cases may slip through.

## Security Considerations

**Path Traversal (Symlink Escape):**

- Risk: The `Filesystem.contains` check is lexical-only, meaning symlinks and Windows cross-drive paths can escape the project sandbox. An LLM-directed file read/write could access arbitrary filesystem locations.
- Files:
  - `packages/opencode/src/file/index.ts:499-500` — `read()` path check
  - `packages/opencode/src/file/index.ts:575-576` — `list()` path check
- Current mitigation: Lexical path prefix check via `Instance.containsPath()`
- Recommendations: Use `fs.realpath()` canonicalization before the containment check. On Windows, normalize drive letters. Add integration tests with symlinks.

**Credentials in process.env:**

- Risk: Provider authentication keys are written directly to `process.env`, making them visible to child processes and any code that reads the environment.
- Files:
  - `packages/opencode/src/provider/provider.ts:235` — `process.env.AWS_BEARER_TOKEN_BEDROCK = auth.key`
  - `packages/opencode/src/provider/provider.ts:440` — `process.env.AICORE_SERVICE_KEY = auth.key`
- Current mitigation: None. Keys persist in process environment.
- Recommendations: Pass credentials through SDK constructor options rather than environment variables. If env vars are required by upstream SDKs, clear them after SDK initialization.

**Global AI SDK Warning Suppression:**

- Risk: `globalThis.AI_SDK_LOG_WARNINGS = false` suppresses all AI SDK warnings, including potential security or deprecation warnings.
- Files:
  - `packages/opencode/src/server/server.ts:44` — sets global
  - `packages/opencode/src/session/prompt.ts:49-50` — duplicate set
- Current mitigation: None.
- Recommendations: Route AI SDK warnings through the log system instead of suppressing them entirely.

**LSP Binary Downloads Without Integrity Check:**

- Risk: LSP server binaries (zls, lua-language-server, clangd, etc.) are downloaded from GitHub releases without verifying checksums or signatures.
- Files:
  - `packages/opencode/src/lsp/server.ts:640-680` — zls download, `(await releaseResponse.json()) as any`
  - `packages/opencode/src/lsp/server.ts:1432` — lua-language-server download, same `as any` pattern
- Current mitigation: HTTPS is used for downloads.
- Recommendations: Verify SHA256 checksums against release manifests. Pin known-good release versions. Add signature verification where available.

## Performance Bottlenecks

**Oversized Source Files:**

- Problem: Several files exceed 1000 lines, concentrating complex logic in single modules. This impacts IDE performance, makes code review harder, and increases merge conflict risk.
- Files:
  - `packages/opencode/src/lsp/server.ts` — 2,057 lines (LSP server definitions for 15+ languages)
  - `packages/opencode/src/session/prompt.ts` — 1,959 lines (prompt construction, command execution, subtask handling)
  - `packages/opencode/src/acp/agent.ts` — 1,698 lines (agent protocol implementation)
  - `packages/opencode/src/cli/cmd/github.ts` — 1,631 lines (GitHub integration)
  - `packages/opencode/src/config/config.ts` — 1,493 lines (config schema, parsing, validation)
  - `packages/opencode/src/provider/provider.ts` — 1,338 lines (provider initialization for 20+ providers)
  - `packages/opencode/src/cli/cmd/tui/routes/session/index.tsx` — 2,201 lines (TUI session view)
  - `packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx` — 1,155 lines (TUI prompt component)
- Cause: New providers, LSP servers, and features get appended to existing files instead of extracted into separate modules.
- Improvement path: Extract each LSP server definition to its own file under `src/lsp/servers/`. Split `provider.ts` into per-provider loader files. Break `prompt.ts` into `prompt/loop.ts`, `prompt/tools.ts`, `prompt/commands.ts`. Extract `config.ts` schema into `config/schema.ts`.

**Generated SDK Files:**

- Problem: Large generated files inflate the codebase.
- Files:
  - `packages/sdk/js/src/v2/gen/types.gen.ts` — 5,149 lines
  - `packages/web/src/components/icons/index.tsx` — 4,454 lines
  - `packages/sdk/js/src/gen/types.gen.ts` — 3,904 lines
  - `packages/sdk/js/src/v2/gen/sdk.gen.ts` — 3,366 lines
- Cause: Generated code, not manually written.
- Improvement path: These are generated and acceptable. Consider code-splitting if bundle size becomes an issue.

## Fragile Areas

**Copilot Plugin Fetch Interceptor:**

- Files: `packages/opencode/src/plugin/copilot.ts:60-140`
- Why fragile: The `fetch` wrapper manually parses JSON bodies to detect vision/agent requests, then rewrites headers. It checks nested arrays 3-4 levels deep with `(part: any)` casts. Different code paths for Completions API, Responses API, and Messages API.
- Safe modification: Add comprehensive tests for each API format. Type the request body interfaces. Test with actual Copilot API responses.
- Test coverage: No dedicated tests for the fetch interceptor logic. `packages/opencode/test/provider/copilot/` tests exist but focus on the model layer.

**Provider Custom Loader System:**

- Files: `packages/opencode/src/provider/provider.ts:87-530`
- Why fragile: 20+ provider loaders in a single record, each with different authentication patterns (env vars, OAuth, API keys, bearer tokens, profiles). The `CUSTOM_LOADERS` and `BUNDLED_PROVIDERS` registries use `any`-typed SDK parameters. Adding a new provider requires understanding the full loader lifecycle.
- Safe modification: Test each provider loader in isolation. Add TypeScript interfaces for loader return types. Document the loader contract.
- Test coverage: `packages/opencode/test/provider/provider.test.ts` (2,220 lines) provides good coverage, but individual loader edge cases may not all be exercised.

**Session Prompt Loop:**

- Files: `packages/opencode/src/session/prompt.ts:240-700` (approx.)
- Why fragile: The main `loop()` function orchestrates LLM calls, tool execution, subtask spawning, compaction, structured output, and error handling. It manages complex state transitions with abort controllers, snapshots, and nested async operations.
- Safe modification: Avoid changing the loop flow without understanding all exit paths. Test with abort scenarios. Be cautious with the `tasks` array and `needsCompaction` flag interactions.
- Test coverage: `packages/opencode/test/session/prompt.test.ts` exists but the loop's complexity means edge cases around abort + compaction + subtask interactions may not be fully covered.

**Plugin Typing System:**

- Files: `packages/opencode/src/plugin/index.ts:110-121`
- Why fragile: The plugin trigger function has a documented "try-counter: 2" for fixing the typing, meaning multiple engineers have attempted and failed to type it correctly. The `@ts-expect-error` is intentional and the function relies on `any` casts.
- Safe modification: Do not attempt to change the typing without allocating significant time. The current approach works at runtime.
- Test coverage: No dedicated typing tests. Runtime behavior tested indirectly through plugin integration.

**Token Cost Calculation:**

- Files: `packages/opencode/src/session/index.ts:780-845`
- Why fragile: Cost calculation uses provider-specific metadata paths (`input.metadata?.["anthropic"]?.["cacheCreationInputTokens"]`, `input.metadata?.["bedrock"]?.["usage"]?.["cacheWriteInputTokens"]`). Each provider reports cache tokens differently. The `excludesCachedTokens` flag handles Anthropic/Bedrock differently from other providers. Reasoning tokens are charged at output rates as a TODO workaround.
- Safe modification: Add unit tests per provider's token reporting format. Centralize provider-specific metadata extraction.
- Test coverage: Not covered by dedicated cost calculation tests.

## Scaling Limits

**LSP Server Binary Management:**

- Current capacity: Downloads and manages binaries for 15+ language servers.
- Limit: Each language server has its own download/install/version logic duplicated in `packages/opencode/src/lsp/server.ts`. Adding more servers linearly increases file size and maintenance burden.
- Scaling path: Extract a common binary download/install abstraction. Define language server configs declaratively.

**Single SQLite Database:**

- Current capacity: All sessions, messages, parts, projects, permissions stored in one SQLite file.
- Limit: SQLite write serialization. Very large session histories may slow queries.
- Scaling path: Current design is appropriate for a local tool. If multi-user or cloud deployment is needed, the Drizzle ORM abstraction would allow swapping to PostgreSQL.

## Dependencies at Risk

**Drizzle ORM Beta:**

- Risk: Pinned to `1.0.0-beta.12-a5629fb`, a commit-specific beta. Breaking changes between betas are expected. The commit hash pin suggests patching around specific issues.
- Impact: Database schema management, migrations, and all queries depend on this.
- Migration plan: Upgrade to stable 1.0 when released. Monitor Drizzle changelog for breaking changes. The schema uses standard SQL types, so migration to another ORM is feasible if needed.

**AI SDK Global Suppression:**

- Risk: The `globalThis.AI_SDK_LOG_WARNINGS = false` hack depends on an internal implementation detail of `@vercel/ai`. If the AI SDK changes how it checks this flag, warnings could start appearing in stdout (breaking TUI output).
- Impact: TUI rendering could be corrupted by unexpected stdout writes.
- Migration plan: Track the referenced GitHub issue. Request an official API for warning configuration.

## Missing Critical Features

**Permission Rules Not Persisted:**

- Problem: When users select "always allow" for a permission, the rule is stored in-memory only.
- Blocks: Permission rules are lost on restart. Users must re-approve the same operations each session.
- Files: `packages/opencode/src/permission/next.ts:227-229` — persistence code commented out with `// TODO: we don't save the permission ruleset to disk yet until there's UI to manage it`

**Settings Pages Incomplete:**

- Problem: Three settings pages in the desktop app are stubs with placeholder text.
- Blocks: Users cannot configure commands, agents, or MCP through the desktop UI.
- Files:
  - `packages/app/src/components/settings-commands.tsx:5`
  - `packages/app/src/components/settings-agents.tsx:5`
  - `packages/app/src/components/settings-mcp.tsx:5`

## Test Coverage Gaps

**Session/Prompt Core Logic:**

- What's not tested: The full `SessionPrompt.loop()` with all its branches (abort during compaction, structured output fallback, subtask error propagation, retry with compaction).
- Files: `packages/opencode/src/session/prompt.ts` (1,959 lines)
- Risk: The most critical code path in the application. Regressions could cause infinite loops, data loss, or cost overruns.
- Priority: High

**PTY Management:**

- What's not tested: Only one test file (`packages/opencode/test/pty/pty-output-isolation.test.ts`). No tests for session lifecycle, timeout handling, or concurrent PTY access. `packages/opencode/src/pty/index.ts` has 8 empty catch blocks.
- Files: `packages/opencode/src/pty/index.ts` (400+ lines)
- Risk: PTY bugs cause hanging processes, orphaned shells, or corrupted output.
- Priority: High

**Error Swallowing (~59 silent catch blocks):**

- What's not tested: Approximately 7 empty `catch {}` blocks and 52 `.catch(() => {})` patterns silently discard errors across the codebase. These are not tested because they are explicitly designed to be silent, but any one of them could mask a real failure.
- Files: Distributed across `packages/opencode/src/` — heaviest in `src/pty/index.ts` (8 empty catches), `src/lsp/server.ts`, `src/util/filesystem.ts`, `src/file/watcher.ts`
- Risk: Failures in file operations, LSP communication, share syncing, and clipboard access are silently ignored. Debugging production issues becomes difficult.
- Priority: Medium — many of these are intentionally best-effort, but each should be audited for whether at minimum a debug log should be emitted.

**Provider Loaders (Individual):**

- What's not tested: While `packages/opencode/test/provider/provider.test.ts` covers the system broadly, individual provider authentication edge cases (AWS bearer tokens vs profiles vs container credentials, Azure cognitive services, SAP AI Core) lack isolated tests.
- Files: `packages/opencode/src/provider/provider.ts:119-530`
- Risk: Authentication regressions for specific cloud providers go unnoticed until user reports.
- Priority: Medium

**Worktree Operations:**

- What's not tested: `packages/opencode/src/worktree/index.ts` has fire-and-forget `void start().catch()` patterns and setTimeout-based init. Only one test file exists (`test/project/worktree-remove.test.ts`).
- Files: `packages/opencode/src/worktree/index.ts` (460+ lines)
- Risk: Race conditions in worktree creation/deletion. Orphaned git worktrees on crash.
- Priority: Medium

**~40 Source Modules Without Corresponding Tests:**

- What's not tested: Core modules including `src/session/processor.ts`, `src/session/summary.ts`, `src/share/share-next.ts`, `src/auth/index.ts`, `src/server/routes/*.ts`, `src/worktree/index.ts`, `src/util/rpc.ts`, `src/util/log.ts`, and most CLI commands.
- Files: See full list in exploration — approximately 40 non-trivial source files in `packages/opencode/src/` have no corresponding test file.
- Risk: Changes to untested modules rely entirely on integration testing and manual QA.
- Priority: Medium

---

_Concerns audit: 2026-02-21_
