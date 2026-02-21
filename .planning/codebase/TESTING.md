# Testing Patterns

**Analysis Date:** 2026-02-21

## Test Framework

**Runner:**

- Bun's built-in test runner (`bun:test`) for unit/integration tests
- Playwright for E2E tests (packages/app only)
- No Jest or Vitest

**Assertion Library:**

- `bun:test` built-in `expect` (Jest-compatible API)
- Playwright's `expect` for E2E assertions

**Run Commands:**

```bash
# packages/opencode - unit/integration tests
bun test --timeout 30000            # Run all tests (from packages/opencode/)
bun test test/tool/read.test.ts     # Run specific test file
bun test --watch                    # Watch mode (if supported)

# packages/app - unit tests
bun test:unit                       # Run unit tests (from packages/app/)
bun test:unit:watch                 # Watch mode

# packages/app - E2E tests
bun test:e2e                        # Run all E2E tests (from packages/app/)
bun test:e2e -- app/home.spec.ts    # Run specific E2E test file
bun test:e2e -- -g "test title"     # Run single test by title
bun test:e2e:ui                     # Interactive Playwright UI
bun test:e2e:local                  # E2E with local server setup
bun test:e2e:report                 # View HTML report

# CRITICAL: Do NOT run tests from repo root
# Root package.json has: "test": "echo 'do not run tests from root' && exit 1"
# Root bunfig.toml has: root = "./do-not-run-tests-from-root"
```

**Turbo Integration:**

- `turbo.json` defines `opencode#test` and `@opencode-ai/app#test` tasks
- Both depend on `^build` (build dependencies first)

## Test File Organization

**Location:**

- `packages/opencode`: Separate `test/` directory mirroring `src/` structure
- `packages/app` unit tests: Co-located with source files (`src/**/*.test.ts`)
- `packages/app` E2E tests: Separate `e2e/` directory organized by feature

**Naming:**

- Unit/integration tests: `<name>.test.ts`
- E2E tests: `<feature-name>.spec.ts`

**Structure:**

```
packages/opencode/
├── test/
│   ├── preload.ts               # Test environment setup (env vars, logging)
│   ├── fixture/
│   │   └── fixture.ts           # tmpdir helper with auto-cleanup
│   ├── AGENTS.md                # Test fixture documentation
│   ├── tool/
│   │   ├── fixtures/            # Test data files (JSON, images)
│   │   ├── read.test.ts
│   │   ├── bash.test.ts
│   │   ├── edit.test.ts
│   │   └── ...
│   ├── session/
│   │   ├── session.test.ts
│   │   ├── compaction.test.ts
│   │   └── ...
│   ├── util/
│   │   ├── glob.test.ts
│   │   ├── lazy.test.ts
│   │   └── ...
│   ├── provider/
│   │   ├── provider.test.ts
│   │   └── copilot/
│   │       └── ...
│   └── ...

packages/app/
├── happydom.ts                  # DOM environment setup (preload)
├── src/
│   ├── context/
│   │   ├── global-sync.test.ts          # Co-located tests
│   │   └── global-sync/
│   │       ├── event-reducer.test.ts
│   │       ├── child-store.test.ts
│   │       └── session-trim.test.ts
│   └── components/
│       ├── prompt-input/
│       │   ├── submit.test.ts
│       │   ├── placeholder.test.ts
│       │   └── history.test.ts
│       └── file-tree.test.ts
├── e2e/
│   ├── fixtures.ts              # Custom Playwright fixtures
│   ├── actions.ts               # Reusable page actions
│   ├── selectors.ts             # DOM selectors
│   ├── utils.ts                 # Test utilities
│   ├── AGENTS.md                # E2E testing guide
│   ├── prompt/
│   │   ├── prompt.spec.ts
│   │   └── prompt-multiline.spec.ts
│   ├── session/
│   │   └── session.spec.ts
│   ├── sidebar/
│   │   └── sidebar.spec.ts
│   └── ...
```

## Test Structure

**Suite Organization (packages/opencode):**

```typescript
import { describe, expect, test } from "bun:test"
import path from "path"
import { Instance } from "../../src/project/instance"
import { tmpdir } from "../fixture/fixture"

// Define a reusable context object for tool tests
const ctx = {
  sessionID: "test",
  messageID: "",
  callID: "",
  agent: "build",
  abort: AbortSignal.any([]),
  messages: [],
  metadata: () => {},
  ask: async () => {},
}

describe("module.feature", () => {
  test("descriptive test name", async () => {
    await using tmp = await tmpdir({ git: true })
    await Instance.provide({
      directory: tmp.path,
      fn: async () => {
        // Test actual implementation
        const result = await SomeModule.doSomething()
        expect(result).toBe(expected)
      },
    })
  })
})
```

**Suite Organization (packages/app unit tests):**

```typescript
import { describe, expect, test } from "bun:test"
import { createStore } from "solid-js/store"
import type { State } from "./types"
import { applyDirectoryEvent } from "./event-reducer"

// Factory functions for test data
const rootSession = (input: { id: string; parentID?: string }) =>
  ({ id: input.id, parentID: input.parentID, time: { created: 1, updated: 1 } }) as Session

describe("applyDirectoryEvent", () => {
  test("inserts root sessions in sorted order", () => {
    const [store, setStore] = createStore(baseState({ session: [...] }))
    applyDirectoryEvent({ event: {...}, store, setStore, push() {}, directory: "/tmp", loadLsp() {} })
    expect(store.session.map((x) => x.id)).toEqual(["a", "b"])
  })
})
```

**E2E Test Structure (packages/app):**

```typescript
import { test, expect } from "../fixtures" // NOT @playwright/test
import { promptSelector } from "../selectors"
import { withSession } from "../actions"

test("can send a prompt and receive a reply", async ({ page, sdk, gotoSession }) => {
  test.setTimeout(120_000)
  await gotoSession()

  const prompt = page.locator(promptSelector)
  await prompt.click()
  await page.keyboard.type("Reply with exactly: test")
  await page.keyboard.press("Enter")

  await expect(page).toHaveURL(/\/session\/[^/?#]+/, { timeout: 30_000 })
})
```

**Patterns:**

- `describe` blocks use `"module.feature"` or `"featureName"` naming
- `test` blocks use descriptive lowercase sentences
- `describe.each` and `test.each` for parameterized tests:

```typescript
const cases: [string, boolean][] = [
  [".env", true],
  [".env.local", true],
  [".env.example", false],
]
test.each(cases)("%s asks=%s", async (filename, shouldAsk) => { ... })
```

## Test Environment Setup

**packages/opencode preload (`test/preload.ts`):**

- Sets XDG env vars to temp directories for isolation
- Clears provider API key env vars (ANTHROPIC_API_KEY, OPENAI_API_KEY, etc.)
- Initializes `Log.init({ print: false })` to suppress logs
- Creates cache version file to prevent cache clearing
- Loaded automatically by Bun test runner (configured in bunfig.toml)

**packages/app preload (`happydom.ts`):**

- Registers Happy DOM global registrator for browser API simulation
- Mocks `HTMLCanvasElement.prototype.getContext` for 2D canvas
- Loaded via: `bun test --preload ./happydom.ts ./src`

## Instance.provide Pattern

Most tests in `packages/opencode` wrap test logic in `Instance.provide()`:

```typescript
await Instance.provide({
  directory: tmp.path, // or projectRoot
  fn: async () => {
    // All code inside has access to Instance.directory, Instance.worktree
    // Tools, sessions, and state are scoped to this directory
    const tool = await SomeTool.init()
    const result = await tool.execute(params, ctx)
    expect(result.output).toContain("expected")
  },
})
```

This creates a project-scoped execution context. Required because modules use `Instance.state()` for per-project state.

## Temporary Directory Fixture

The `tmpdir()` function from `test/fixture/fixture.ts` is the primary test setup utility:

```typescript
import { tmpdir } from "../fixture/fixture"

// Basic temp directory
await using tmp = await tmpdir()

// With git repo initialized
await using tmp = await tmpdir({ git: true })

// With config file
await using tmp = await tmpdir({
  config: { model: "test/model", username: "testuser" },
})

// With custom file setup
await using tmp = await tmpdir({
  init: async (dir) => {
    await Bun.write(path.join(dir, "test.txt"), "hello world")
    return "extra data" // accessible as tmp.extra
  },
})
```

**Key features:**

- Uses `await using` (TC39 explicit resource management) for automatic cleanup
- Creates directories in system temp with prefix `opencode-test-`
- Returns `{ path: string, extra: T }` where `extra` is the `init` return value
- Supports `dispose` callback for custom cleanup
- Paths are `realpath` resolved and null-byte sanitized

## Mocking

**Framework:** `bun:test` built-in `mock` module

**Approach: Avoid mocks as much as possible.** Test actual implementations.

**When mocking IS used (packages/app unit tests):**

```typescript
import { beforeAll, mock } from "bun:test"

beforeAll(async () => {
  mock.module("@solidjs/router", () => ({
    useNavigate: () => () => undefined,
    useParams: () => ({}),
  }))

  mock.module("@opencode-ai/sdk/v2/client", () => ({
    createOpencodeClient: (input: { directory: string }) => clientFor(input.directory),
  }))
})
```

**What to Mock:**

- Router/navigation in component tests
- SDK client calls in component tests
- External modules that require browser context unavailable in test

**What NOT to Mock:**

- Core business logic modules
- Database operations (use real SQLite)
- File system operations (use `tmpdir()` for real filesystem)
- Tool implementations (test actual execution)

**Permission Testing Pattern (no mocks):**
Instead of mocking, inject a test `ask` function into the context:

```typescript
const requests: Array<Omit<PermissionNext.Request, "id" | "sessionID" | "tool">> = []
const testCtx = {
  ...ctx,
  ask: async (req: Omit<PermissionNext.Request, "id" | "sessionID" | "tool">) => {
    requests.push(req)
  },
}
await tool.execute(params, testCtx)
expect(requests.find((r) => r.permission === "external_directory")).toBeDefined()
```

## Fixtures and Factories

**Test Data (packages/opencode):**

- Static fixtures in `test/tool/fixtures/`: JSON files, images
- `tmpdir()` with `init` callback creates dynamic test environments
- Shared context object (`ctx`) for tool test execution

**Factory Functions (packages/app):**

```typescript
const rootSession = (input: { id: string; parentID?: string; archived?: number }) =>
  ({
    id: input.id,
    parentID: input.parentID,
    time: { created: 1, updated: 1, archived: input.archived },
  }) as Session

const userMessage = (id: string, sessionID: string) =>
  ({
    id,
    sessionID,
    role: "user",
    time: { created: 1 },
    agent: "assistant",
    model: { providerID: "openai", modelID: "gpt" },
  }) as Message

const baseState = (input: Partial<State> = {}) =>
  ({ status: "complete", session: [], message: {}, part: {}, ...input }) as State
```

**E2E Fixtures (packages/app/e2e):**

- Custom Playwright fixtures in `e2e/fixtures.ts` extending `base.test`
- `sdk` fixture: OpenCode SDK client for API calls
- `gotoSession(sessionID?)` fixture: Navigate to session page
- `withProject(callback)` fixture: Create and clean up temp project
- Selectors in `e2e/selectors.ts`: `data-component`, `data-action` attributes
- Actions in `e2e/actions.ts`: `openPalette()`, `openSettings()`, `withSession()`

## Coverage

**Requirements:** None enforced (no coverage thresholds configured)

**View Coverage:**

```bash
bun test --coverage     # From packages/opencode/
```

## Test Types

**Unit Tests:**

- Located in `packages/opencode/test/util/`, `packages/app/src/**/*.test.ts`
- Test pure functions, utilities, state reducers
- No external dependencies, fast execution
- Example: `lazy.test.ts`, `glob.test.ts`, `format.test.ts`, `event-reducer.test.ts`

**Integration Tests:**

- Located in `packages/opencode/test/tool/`, `test/session/`, `test/server/`
- Test real tool execution, session creation, server routes
- Use `tmpdir()` + `Instance.provide()` for realistic project setup
- Use real SQLite database, real filesystem
- Example: `read.test.ts`, `bash.test.ts`, `session.test.ts`

**E2E Tests:**

- Located in `packages/app/e2e/`
- Playwright with Chromium
- Test full user workflows against running dev server
- Config in `packages/app/playwright.config.ts`:
  - Timeout: 60s per test, 10s per assertion
  - Retries: 2 in CI, 0 locally
  - Traces: on first retry
  - Screenshots: only on failure
  - Video: retain on failure
  - Reporter: HTML + line
- Use `data-component` and `data-action` selectors (never CSS classes)

## Common Patterns

**Async Testing:**

```typescript
test("async with Instance.provide", async () => {
  await using tmp = await tmpdir({ git: true })
  await Instance.provide({
    directory: tmp.path,
    fn: async () => {
      const result = await someAsyncOp()
      expect(result).toBeDefined()
    },
  })
})
```

**Error Testing:**

```typescript
test("throws on invalid input", async () => {
  await Instance.provide({
    directory: tmp.path,
    fn: async () => {
      const tool = await ReadTool.init()
      await expect(tool.execute({ filePath: path.join(tmp.path, "short.txt"), offset: 4 }, ctx)).rejects.toThrow(
        "Offset 4 is out of range",
      )
    },
  })
})
```

**Event Testing:**

```typescript
test("emits event when session is created", async () => {
  await Instance.provide({
    directory: projectRoot,
    fn: async () => {
      let eventReceived = false
      const unsub = Bus.subscribe(Session.Event.Created, (event) => {
        eventReceived = true
      })

      const session = await Session.create({})
      await new Promise((resolve) => setTimeout(resolve, 100))

      unsub()
      expect(eventReceived).toBe(true)
      await Session.remove(session.id)
    },
  })
})
```

**Parameterized Testing:**

```typescript
describe.each(["build", "plan"])("agent=%s", (agentName) => {
  test.each(cases)("%s asks=%s", async (filename, shouldAsk) => {
    // Test with each combination
  })
})
```

**SolidJS Store Testing (packages/app):**

```typescript
test("updates store correctly", () => {
  const [store, setStore] = createStore(baseState({ session: [...] }))

  applyDirectoryEvent({
    event: { type: "session.created", properties: { info: rootSession({ id: "a" }) } },
    store,
    setStore,
    push() {},
    directory: "/tmp",
    loadLsp() {},
  })

  expect(store.session.map((x) => x.id)).toEqual(["a", "b"])
})
```

**E2E Poll Pattern (for async LLM responses):**

```typescript
await expect
  .poll(
    async () => {
      const messages = await sdk.session.messages({ sessionID, limit: 50 }).then((r) => r.data ?? [])
      return messages
        .filter((m) => m.info.role === "assistant")
        .flatMap((m) => m.parts)
        .filter((p) => p.type === "text")
        .map((p) => p.text)
        .join("\n")
    },
    { timeout: 90_000 },
  )
  .toContain(token)
```

---

_Testing analysis: 2026-02-21_
