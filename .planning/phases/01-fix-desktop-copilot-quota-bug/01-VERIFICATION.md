---
phase: 01-fix-desktop-copilot-quota-bug
verified: 2026-02-21T15:30:00Z
status: passed
score: 4/4 must-haves verified
human_verification:
  - test: "Run a subagent (Task tool) in the Desktop app and check Copilot usage dashboard"
    expected: "Copilot quota usage does NOT increment for subagent requests"
    why_human: "Requires running the Desktop (Tauri) app and checking external Copilot usage dashboard"
  - test: "Run the same subagent prompt in CLI and Desktop, compare API request headers"
    expected: "Both produce x-initiator: agent for subagent sessions"
    why_human: "Requires running both CLI and Desktop side-by-side with network inspection"
  - test: "Send a regular (non-subagent) chat message in Desktop with Copilot"
    expected: "Request uses body-based x-initiator detection (user for user-initiated, agent for tool continuations)"
    why_human: "Requires manual Desktop app interaction and network inspection"
---

# Phase 1: Fix Desktop Copilot Quota Bug — Verification Report

**Phase Goal:** Subagents in the desktop app use the same Copilot API path as CLI, so they don't consume GitHub Copilot quota
**Verified:** 2026-02-21
**Status:** ✅ passed
**Re-verification:** No — initial verification

## Goal Achievement

### Observable Truths

| #   | Truth                                                                                                 | Status     | Evidence                                                                                                                                                                                                                                                                                                                                                          |
| --- | ----------------------------------------------------------------------------------------------------- | ---------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | Subagent sessions in Desktop send `x-initiator: agent` header to Copilot API                          | ✓ VERIFIED | `chat.headers` hook (copilot.ts:304-318) uses `Session.get()` (direct DB access, no HTTP round-trip) to check `parentID`, sets `output.headers["x-initiator"] = "agent"`. This bypasses the SDK HTTP client that silently failed in Tauri.                                                                                                                        |
| 2   | Subagent sessions in CLI continue to send `x-initiator: agent` (no regression)                        | ✓ VERIFIED | Same code path applies to CLI — `Session.get()` is a database call that works identically in both contexts. The `.catch(() => undefined)` gracefully handles any edge cases. No context-specific branching.                                                                                                                                                       |
| 3   | Regular (non-subagent) sessions in Desktop send `x-initiator` based on body detection (no regression) | ✓ VERIFIED | Fetch interceptor (copilot.ts:68-119) still performs body-based `isAgent` detection and sets `x-initiator` on line 122. For non-subagent sessions, `chat.headers` hook returns early at line 315 (`!session.parentID`), so `x-initiator` is NOT overridden — body-based detection remains the sole mechanism.                                                     |
| 4   | Root cause is documented in code comment explaining the fix                                           | ✓ VERIFIED | Lines 312-313: `"// Direct Session access avoids SDK HTTP client round-trip which can // silently fail in Desktop (Tauri) context, causing subagent sessions // to miss the x-initiator: agent header and consume Copilot quota."` Commit message also documents the fix: `fix(copilot): use direct session access for x-initiator header in subagent detection`. |

**Score:** 4/4 truths verified

### Required Artifacts

| Artifact                                  | Expected                                                   | Exists        | Substantive                                              | Wired                                                                          | Status     |
| ----------------------------------------- | ---------------------------------------------------------- | ------------- | -------------------------------------------------------- | ------------------------------------------------------------------------------ | ---------- |
| `packages/opencode/src/plugin/copilot.ts` | Copilot auth plugin with correct x-initiator for subagents | ✓ (320 lines) | ✓ (320 lines, full implementation, no stubs in fix area) | ✓ (imported by plugin system, `chat.headers` hook invoked by `llm.ts:133-145`) | ✓ VERIFIED |

### Key Link Verification

| From                                      | To                          | Via                                       | Status  | Details                                                                                                                                                                                                                                                                                           |
| ----------------------------------------- | --------------------------- | ----------------------------------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `copilot.ts` chat.headers hook (line 314) | Session parentID check      | `Session.get(incoming.sessionID)`         | ✓ WIRED | Direct import `import { Session } from "../session"` (line 3). `Session.get()` does direct DB query (session/index.ts:335-338), returns object with `parentID` field (session/index.ts:67). `.catch(() => undefined)` handles errors gracefully.                                                  |
| `copilot.ts` chat.headers hook (line 317) | Copilot API request headers | `output.headers["x-initiator"] = "agent"` | ✓ WIRED | `llm.ts:133-145` calls `Plugin.trigger("chat.headers", ...)` with `sessionID`. Result `headers` spread LAST into `streamText()` at `llm.ts:222` (`...headers`), ensuring chat.headers values override any prior headers.                                                                          |
| `copilot.ts` fetch interceptor (line 122) | Copilot API request headers | Body-based `isAgent` → `x-initiator`      | ✓ WIRED | Fetch interceptor at line 68-119 parses request body to detect `isAgent`. Sets `"x-initiator": isAgent ? "agent" : "user"` (line 122) BEFORE `...(init?.headers)` spread (line 123), so `chat.headers` values from streamText headers override fetch interceptor's body-based value when present. |

### Requirements Coverage

| Requirement                                          | Status      | Notes                                                                                                  |
| ---------------------------------------------------- | ----------- | ------------------------------------------------------------------------------------------------------ |
| BUG-01: Subagents consume Copilot quota in Desktop   | ✓ SATISFIED | Root cause fixed: `sdk.session.get()` replaced with direct `Session.get()` that works in Tauri context |
| BUG-02: CLI and Desktop subagent behavior divergence | ✓ SATISFIED | Both now use identical code path (direct `Session.get()`) with no context-specific branching           |

### Anti-Patterns Found

| File         | Line  | Pattern                                      | Severity | Impact                                                           |
| ------------ | ----- | -------------------------------------------- | -------- | ---------------------------------------------------------------- |
| `copilot.ts` | 43-44 | TODO comments about messages API rate limits | ℹ️ Info  | Pre-existing, unrelated to this fix — about future API migration |
| `copilot.ts` | 176   | `return undefined` in validate callback      | ℹ️ Info  | Legitimate — Zod/prompt validation pattern (undefined = valid)   |

No blocker or warning-level anti-patterns found in the fix area.

### Human Verification Required

### 1. Desktop Subagent Quota Test

**Test:** Run a subagent (Task tool) in the Desktop app targeting a Copilot model. Check Copilot usage dashboard before and after.
**Expected:** Copilot quota usage does NOT increment for the subagent request (or increments at the agent/non-quota rate).
**Why human:** Requires running the Desktop (Tauri) app and checking the external GitHub Copilot usage dashboard. Cannot verify programmatically.

### 2. CLI/Desktop Header Parity Test

**Test:** Run the same subagent prompt in CLI and Desktop with network inspection enabled.
**Expected:** Both contexts produce `x-initiator: agent` header on Copilot API requests for subagent sessions.
**Why human:** Requires running both CLI and Desktop side-by-side with network traffic capture or proxy.

### 3. Non-Subagent Regression Test

**Test:** Send a regular chat message (not via Task tool) in Desktop using a Copilot model.
**Expected:** Request uses body-based `x-initiator` detection — `user` for initial messages, `agent` for tool-continuation turns.
**Why human:** Requires manual Desktop app interaction and network inspection to verify correct header values.

### Gaps Summary

No gaps found. All four must-have truths are verified at the code level:

1. **Session access fix is correct:** `Session.get()` (direct DB call) replaces `sdk.session.get()` (HTTP round-trip through SDK client). The SDK client used `Server.App().fetch()` which can silently fail in Tauri, causing `.catch(() => undefined)` to swallow the error and skip the `x-initiator: agent` header.

2. **Header propagation chain is intact:** `chat.headers` hook → `Plugin.trigger()` result → `...headers` spread last in `streamText()` call → arrives as `init?.headers` in fetch interceptor → spread after body-based value, correctly overriding it.

3. **No regression risk:** Non-subagent sessions hit the early return at line 315 (`!session.parentID`), so their `x-initiator` continues to be set exclusively by the fetch interceptor's body-based detection.

4. **Clean implementation:** No debug logging remains, unused `sdk` variable removed, explanatory comment added, commit message documents the fix clearly.

---

_Verified: 2026-02-21_
_Verifier: OpenCode (gsd-verifier)_
