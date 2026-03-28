# Roadmap: SlideTutor AI Refinement

## Phases
- [x] **Phase 1: Environment & Core Stability** - Secure API keys and establish regression safety.
- [x] **Phase 2: Data Persistence Migration** - Transition from LocalStorage to IndexedDB for robust data handling.

## Phase Details

### Phase 1: Environment & Core Stability
**Goal**: Secure the application environment and ensure a baseline for stable iteration.
**Depends on**: Nothing
**Requirements**: SEC-01, STAB-01, STAB-02
**Success Criteria** (what must be TRUE):
  1. API keys are handled via secure environment variables and not exposed in client-side code.
  2. A regression baseline exists for core PDF rendering and AI explanation features.
  3. Core functional logic remains concise and clean after security hardening.
**Plans**: 2 plans
- [x] 01-01-PLAN.md — Environment security and regression baseline. (COMPLETED)
- [x] 01-02-PLAN.md — Fix serverless rate limiting gap. (COMPLETED)

### Phase 2: Data Persistence Migration
**Goal**: Implement a robust local storage solution to prevent data loss.
**Depends on**: Phase 1
**Requirements**: DATA-01, DATA-02
**Success Criteria** (what must be TRUE):
  1. User data is successfully migrated from LocalStorage to IndexedDB without loss.
  2. Application state (current slide, history) is preserved across sessions and browser restarts.
  3. Subsequent updates do not break compatibility with existing IndexedDB data.
**Plans**: 2 plans
- [x] 02-01-PLAN.md — Database schema update and migration infrastructure. (COMPLETED)
- [x] 02-02-PLAN.md — Store and component integration. (COMPLETED)

## Next Work

No future phases are currently planned.
Define the next milestone or add a new phase when ready.

## Progress Table

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Environment & Core Stability | 2/2 | Completed | 2026-03-27 |
| 2. Data Persistence Migration | 2/2 | Completed | 2026-03-26 |
