---
phase: 01-fix-desktop-copilot-quota-bug
plan: 01
subsystem: copilot-auth-plugin
tags: [copilot, x-initiator, subagent, desktop, tauri]
dependency-graph:
  requires: []
  provides:
    - "Reliable x-initiator: agent header for subagent sessions in all contexts"
  affects: []
tech-stack:
  added: []
  patterns:
    - "Direct module access over SDK HTTP client for intra-process data"
key-files:
  created: []
  modified:
    - packages/opencode/src/plugin/copilot.ts
decisions:
  - id: "use-direct-session-access"
    description: "Replace sdk.session.get() with direct Session.get() import to avoid SDK HTTP client failure modes in Desktop"
    rationale: "The SDK client does an HTTP round-trip through Server.App().fetch() which can silently fail in Desktop (Tauri) context. Direct Session.get() accesses the database directly with no intermediary."
metrics:
  duration: "~3 minutes"
  completed: "2026-02-21"
---

# Phase 1 Plan 1: Investigate and Fix x-initiator Header for Desktop Subagents Summary

**One-liner:** Direct Session.get() replaces SDK HTTP client for subagent parentID check, fixing silent failures in Desktop that caused Copilot quota consumption.

## What Was Done

### Root Cause Analysis

Traced the `x-initiator` header flow through three mechanisms:

1. **Fetch interceptor** (`copilot.ts:63-139`): Body-based `isAgent` detection parses request body to check if the last message role is not `"user"`. For fresh subagent sessions the last message IS `role: "user"` (the task prompt), so this yields `isAgent: false`. This is correct — the fetch interceptor is a fallback, and `init?.headers` spread on line 123 allows `chat.headers` hook to override it.

2. **chat.headers hook** (`copilot.ts:304-325`): This is the subagent-specific mechanism. It calls `sdk.session.get()` which routes through `createOpencodeClient` → `Server.App().fetch()` (an HTTP round-trip through the SDK client). The `.catch(() => undefined)` on line 321 silently swallowed all errors. In Desktop (Tauri) context, this SDK HTTP call fails, so `x-initiator: agent` is never set.

3. **Header propagation** (`llm.ts:208-223`): Headers from `Plugin.trigger("chat.headers", ...)` are correctly spread into `streamText()` call. The AI SDK passes them through `combineHeaders()` to the fetch interceptor's `init?.headers`. This chain works — the bug was upstream in mechanism 2.

**Root cause:** `sdk.session.get()` in `chat.headers` hook uses the SDK HTTP client which silently fails in Desktop context, so `x-initiator: agent` is never set for subagent sessions.

### Fix Applied

Replaced `sdk.session.get()` with direct `Session.get(incoming.sessionID)`:

- **Before:** `sdk.session.get({ path: { id }, query: { directory } })` → HTTP round-trip through SDK client → `.catch(() => undefined)` silently swallows failures
- **After:** `Session.get(incoming.sessionID)` → Direct database access via the Session module → No HTTP intermediary, no failure modes from SDK client configuration

Also removed the now-unused `const sdk = input.client` variable.

## Changes

| File                                      | Change                                                                                                                              |
| ----------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| `packages/opencode/src/plugin/copilot.ts` | Added `Session` import, replaced `sdk.session.get()` with `Session.get()`, removed unused `sdk` variable, added explanatory comment |

## Verification

- `bun test` from `packages/opencode`: **1135 pass, 5 skip, 1 pre-existing fail** (unrelated file permissions test)
- Pre-existing failure confirmed by running same test on unmodified code
- Grep for `x-initiator` confirms both mechanisms (fetch interceptor body-based + chat.headers hook) intact
- No temporary debug code remains
- Code comment documents why direct access is used

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 2 - Missing Critical] Removed dead code (`const sdk = input.client`)**

- **Found during:** Task 2
- **Issue:** After replacing `sdk.session.get()` with `Session.get()`, the `sdk` variable was unused dead code
- **Fix:** Removed `const sdk = input.client` line
- **Files modified:** `packages/opencode/src/plugin/copilot.ts`
- **Commit:** 5d97cdffd

## Commits

| Hash        | Message                                                                                |
| ----------- | -------------------------------------------------------------------------------------- |
| `5d97cdffd` | `fix(copilot): use direct session access for x-initiator header in subagent detection` |
