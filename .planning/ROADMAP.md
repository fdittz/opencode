# Roadmap: OpenCode Fork

## Overview

Fix the desktop subagent Copilot quota bug and add comprehensive test coverage. CLI subagents correctly avoid consuming Copilot quota, but desktop subagents do not. Phase 1 investigates and fixes the root cause. Phase 2 adds dedicated tests for the Copilot auth plugin mechanisms that were identified as fragile and untested.

## Phases

**Phase Numbering:**

- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

- [x] **Phase 1: Fix Desktop Copilot Quota Bug** - Investigate and fix subagents consuming Copilot quota in Desktop but not CLI ✓
- [ ] **Phase 2: Test Copilot Desktop Fix** - Add dedicated tests for the fetch interceptor, chat.headers hook, and header propagation chain

## Phase Details

### Phase 1: Fix Desktop Copilot Quota Bug

**Goal**: Subagents in the desktop app use the same Copilot API path as CLI, so they don't consume GitHub Copilot quota
**Depends on**: Nothing (first phase)
**Requirements**: BUG-01, BUG-02
**Success Criteria** (what must be TRUE):

1. Running a subagent (Task tool) in the desktop app does NOT increment Copilot quota usage — verified by checking Copilot usage dashboard before and after
2. Running the same subagent prompt in CLI and Desktop produces equivalent API request headers (auth, model routing, agent markers) — the fetch interceptor in `copilot.ts` applies identically in both contexts
3. The root cause is documented in a code comment or commit message explaining WHY desktop behaved differently from CLI
4. Regular (non-subagent) Copilot requests in desktop continue working correctly — no regression
   **Plans:** 1 plan

Plans:

- [x] 01-01-PLAN.md — Investigate x-initiator header flow and fix subagent quota bug ✓

### Phase 2: Test Copilot Desktop Fix

**Goal**: Comprehensive test coverage for the Copilot auth plugin fetch interceptor, chat.headers subagent detection hook, and the header propagation chain that ensures x-initiator overrides work correctly
**Depends on**: Phase 1 (fix must exist to test)
**Requirements**: HARD-01
**Success Criteria** (what must be TRUE):

1. Fetch interceptor body-based detection tested for all three API formats (completions, responses, messages)
2. chat.headers hook tested for subagent detection (parentID → x-initiator: agent) with real Session data
3. Header propagation chain tested — chat.headers override beats body-based detection
4. Vision detection tested for all API formats
5. Error handling tested (invalid bodies, nonexistent sessions)
6. All new tests pass, no regressions in existing suite (baseline: 1135 pass, 5 skip, 1 pre-existing fail)
   **Plans:** 3 plans

Plans:

- [ ] 02-01-PLAN.md — Fetch interceptor unit tests (body detection, headers, vision)
- [ ] 02-02-PLAN.md — chat.headers hook tests (subagent detection, error handling)
- [ ] 02-03-PLAN.md — Header propagation integration tests (override chain)

## Progress

| Phase                            | Plans Complete | Status   | Completed  |
| -------------------------------- | -------------- | -------- | ---------- |
| 1. Fix Desktop Copilot Quota Bug | 1/1            | Complete | 2026-02-21 |
| 2. Test Copilot Desktop Fix      | 0/3            | Planned  | —          |
