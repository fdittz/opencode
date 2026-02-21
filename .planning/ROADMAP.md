# Roadmap: OpenCode Fork

## Overview

Fix the desktop subagent Copilot quota bug. CLI subagents correctly avoid consuming Copilot quota, but desktop subagents do not. This single-phase roadmap investigates the root cause in the Copilot auth plugin fetch interceptor and fixes the divergent behavior between CLI and Desktop (Tauri) execution paths.

## Phases

**Phase Numbering:**

- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

- [ ] **Phase 1: Fix Desktop Copilot Quota Bug** - Investigate and fix subagents consuming Copilot quota in Desktop but not CLI

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
   **Plans**: TBD

Plans:

- [ ] 01-01: Investigate divergence and implement fix

## Progress

| Phase                            | Plans Complete | Status      | Completed |
| -------------------------------- | -------------- | ----------- | --------- |
| 1. Fix Desktop Copilot Quota Bug | 0/TBD          | Not started | -         |
