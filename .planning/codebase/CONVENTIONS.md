# Coding Conventions

**Analysis Date:** 2026-02-21

## Naming Patterns

**Files:**

- Use lowercase with hyphens for multi-word files: `message-v2.ts`, `bus-event.ts`, `json-migration.ts`
- Drizzle schema files use `.sql.ts` suffix: `session.sql.ts`, `schema.sql.ts`, `control.sql.ts`
- Prompt/description text files use `.txt` extension imported as default: `read.txt`, `compaction.txt`
- Single-word filenames preferred: `lazy.ts`, `glob.ts`, `context.ts`, `iife.ts`
- Test files use `.test.ts` suffix, E2E files use `.spec.ts` suffix

**Functions:**

- Prefer single-word camelCase names: `lazy()`, `iife()`, `publish()`, `subscribe()`
- Only use multi-word when necessary for clarity: `formatDuration()`, `assertExternalDirectory()`

**Variables:**

- Prefer single-word names: `log`, `state`, `result`, `match`
- Use `const` exclusively; avoid `let` unless reassignment is unavoidable
- Inline values used only once:

```typescript
// Good
const journal = await Bun.file(path.join(dir, "journal.json")).json()

// Bad
const journalPath = path.join(dir, "journal.json")
const journal = await Bun.file(journalPath).json()
```

**Types:**

- Use TypeScript `namespace` pattern to group related types, schemas, and functions together
- Zod schemas double as type definitions using `z.infer`:

```typescript
export namespace Agent {
  export const Info = z.object({
    name: z.string(),
    description: z.string().optional(),
    // ...
  })
  export type Info = z.infer<typeof Info>
}
```

- Namespace-scoped `const` for internal state, loggers, etc.
- `type` keyword for local type aliases (e.g., `type SessionRow = typeof SessionTable.$inferSelect`)

**Identifiers (IDs):**

- Prefixed IDs with underscore separator: `ses_`, `msg_`, `prt_`, `per_`, `que_`, `pty_`, `tool_`
- Generated via `Identifier.ascending()` or `Identifier.descending()` from `@/id/id`

## Code Style

**Formatting:**

- Prettier with config in root `package.json`:
  - `semi: false` (no semicolons)
  - `printWidth: 120`
- `.editorconfig`: 2-space indent, UTF-8, LF line endings, trailing newline
- `.prettierignore` excludes generated files: `sst-env.d.ts`, `packages/desktop/src/bindings.ts`

**Linting:**

- No ESLint in main packages (only `sdks/vscode/eslint.config.mjs` for VS Code extension)
- Pre-push hook runs `bun typecheck` via Husky (`.husky/pre-push`)
- Turbo handles `typecheck` task across packages

**General Principles:**

- Avoid `try`/`catch` where possible
- Avoid using the `any` type
- Prefer Bun APIs: `Bun.file()`, `Bun.write()`, `$` shell tag
- Rely on type inference; avoid explicit type annotations unless necessary for exports or clarity
- Prefer functional array methods (`flatMap`, `filter`, `map`) over `for` loops; use type guards on `filter`
- Keep things in one function unless composable or reusable

## Namespace Module Pattern

The codebase uses a distinctive TypeScript `namespace` pattern as the primary module organization:

```typescript
// packages/opencode/src/bus/index.ts
export namespace Bus {
  const log = Log.create({ service: "bus" })

  const state = Instance.state(() => {
    // per-instance state initialization
  })

  export async function publish<T>(def: T, properties: z.output<T>) {
    // ...
  }

  export function subscribe<T>(def: T, callback: (event: T) => void) {
    // ...
  }
}
```

**Key characteristics:**

- Namespaces group related exports, internal state, and private helpers
- Namespaces act as singletons with instance-scoped state via `Instance.state()`
- Zod schemas are defined as namespace members with matching `type` re-exports
- Private functions/constants live inside namespace but are not exported

**When to use this pattern:**

- Core domain modules: `Session`, `Bus`, `Config`, `Provider`, `Agent`, `Tool`
- Utility modules: `Context`, `Identifier`, `Glob`
- Every new module in `packages/opencode/src/` should follow this pattern

## Destructuring

Avoid unnecessary destructuring. Use dot notation to preserve context:

```typescript
// Good
obj.a
obj.b

// Bad
const { a, b } = obj
```

## Control Flow

Avoid `else` statements. Prefer early returns:

```typescript
// Good
function foo() {
  if (condition) return 1
  return 2
}

// Bad
function foo() {
  if (condition) return 1
  else return 2
}
```

Use ternaries instead of `let` + conditional assignment:

```typescript
// Good
const foo = condition ? 1 : 2

// Bad
let foo
if (condition) foo = 1
else foo = 2
```

## Import Organization

**Order (observed in source files):**

1. External npm packages (`zod`, `path`, `hono`, `ai`, `remeda`, etc.)
2. Workspace packages (`@opencode-ai/util/...`, `@opencode-ai/sdk/...`)
3. Path-aliased internal imports (`@/bus`, `@/util/log`, `@/permission/next`)
4. Relative imports (`../config/config`, `./bus-event`)

**Path Aliases (packages/opencode):**

- `@/*` maps to `./src/*` (configured in `packages/opencode/tsconfig.json`)
- `@tui/*` maps to `./src/cli/cmd/tui/*`
- Use `@/` for cross-directory imports within the same package
- Use relative imports for same-directory or parent-directory imports

**Path Aliases (packages/app):**

- `@/` maps to `./src/`
- Used for context, components, and utilities

**Workspace References:**

- Use `workspace:*` for inter-package dependencies
- Catalog versions in root `package.json` for shared dependencies

**Import Style:**

- Named imports preferred: `import { Session } from "../../session"`
- Default imports for text files: `import DESCRIPTION from "./read.txt"`
- `import z from "zod"` (default import for Zod, not `{ z }`)
- `import type` for type-only imports: `import type { Provider } from "@/provider/provider"`

## Error Handling

**Patterns:**

- Throw `Error` with descriptive messages directly:

```typescript
throw new Error(`File not found: ${filepath}`)
throw new Error(`ID ${given} does not start with ${prefixes[prefix]}`)
```

- Use `NamedError` from `@opencode-ai/util/error` for domain errors
- Use `NotFoundError` from `@/storage/db` for database lookups
- Avoid `try`/`catch` unless strictly necessary (e.g., `lazy.ts` reset on init failure)
- Chain errors with `{ cause: error }` when rethrowing:

```typescript
throw new Error(`The ${id} tool was called with invalid arguments: ${error}`, { cause: error })
```

- In promise chains, use `.catch(() => [])` or `.catch(() => {})` for non-critical failures
- Zod validation errors get formatted via optional `formatValidationError` on tool definitions

## Logging

**Framework:** Custom `Log` utility from `@/util/log`

**Patterns:**

- Create scoped loggers: `const log = Log.create({ service: "session" })`
- Use structured logging: `log.info("publishing", { type: def.type })`
- Initialize logging in tests: `Log.init({ print: false })`
- Log levels: DEBUG, INFO, etc.

## Comments

**When to Comment:**

- JSDoc `/** ... */` only for exported functions that need clarification
- Inline comments for non-obvious logic: `// Non-git projects set worktree to "/" which would match ANY absolute path.`
- `// Good` / `// Bad` patterns in AGENTS.md for convention documentation

**JSDoc/TSDoc:**

- Minimal usage; type system provides documentation
- Used on select exported utilities: `/** Extract timestamp from an ascending ID. Does not work with descending IDs. */`

## Function Design

**Size:** Keep functions compact; one screen of code preferred. Break out only if reusable.

**Parameters:** Use object parameters for functions with 3+ arguments or optional params:

```typescript
async function list(input: { directory?: string; roots?: boolean; start?: number; search?: string; limit?: number })
```

**Return Values:** Return typed objects; let TypeScript infer return types. Use explicit return types only for exported functions when clarity is needed.

## Module Design

**Exports:** Primarily via namespace members (`export namespace Foo { export function bar() {} }`)

**Barrel Files:** Modules use `index.ts` as the entry point within directories. The namespace is exported from `index.ts`:

```
src/bus/
  index.ts       # exports namespace Bus
  bus-event.ts   # exports namespace BusEvent
  global.ts      # exports GlobalBus
```

**Module File Layout:**

- Domain logic in `index.ts` or `<module>.ts`
- SQL schema in `<module>.sql.ts`
- Related types colocated with their logic
- Utility functions in dedicated files within `src/util/`

## Schema Definitions (Drizzle)

Use snake_case for field names so column names don't need to be redefined as strings:

```typescript
// Good
const table = sqliteTable("session", {
  id: text().primaryKey(),
  project_id: text().notNull(),
  created_at: integer().notNull(),
})

// Bad
const table = sqliteTable("session", {
  id: text("id").primaryKey(),
  projectID: text("project_id").notNull(),
  createdAt: integer("created_at").notNull(),
})
```

**Database conventions:**

- Schema files: `src/**/*.sql.ts`
- Tables and columns: snake_case
- Join columns: `<entity>_id` (e.g., `project_id`, `session_id`, `message_id`)
- Indexes: `<table>_<column>_idx` (e.g., `session_project_idx`, `message_session_idx`)
- Shared timestamp columns via `Timestamps` from `@/storage/schema.sql`:

```typescript
export const Timestamps = {
  time_created: integer()
    .notNull()
    .$default(() => Date.now()),
  time_updated: integer()
    .notNull()
    .$onUpdate(() => Date.now()),
}
```

- Migrations generated via: `bun run db generate --name <slug>`

## Zod Schema Conventions

- Use `z.object()` for schemas, attach `.meta({ ref: "Name" })` for OpenAPI refs
- Coerce numbers in API validators: `z.coerce.number()`
- Use `.meta({ description: "..." })` for query param documentation
- Define schemas inside namespaces alongside the matching `type` alias:

```typescript
export const Info = z
  .object({
    id: z.string(),
    name: z.string(),
  })
  .meta({ ref: "Agent" })
export type Info = z.infer<typeof Info>
```

## State Management

**Backend (packages/opencode):**

- `Instance.state()` creates per-project-directory scoped singletons
- `AsyncLocalStorage` via `Context.create()` for request-scoped context
- State disposal handled via `Instance.dispose()` and `State.dispose()`

**Frontend (packages/app):**

- SolidJS `createStore` preferred over multiple `createSignal` calls
- Reactive stores for global state management
- WebSocket events drive store updates via event reducers

## API Route Conventions

- Hono framework with `hono-openapi` for route documentation
- `describeRoute()` + `validator()` for type-safe request handling
- Routes defined as `lazy()` wrapped Hono instances
- Response schemas use `resolver()` with Zod schemas
- Error responses defined in `src/server/error.ts` and reused via `errors(400, 404)`

---

_Convention analysis: 2026-02-21_
