# Roadmap: SlideTutor AI

## Phases
- [x] **Phase 1: Environment & Core Stability** - Secure API keys and establish regression safety.
- [x] **Phase 2: Data Persistence Migration** - Transition from LocalStorage to IndexedDB for robust data handling.
- [x] **Phase 3: Minimal Cloudflare Migration** - Establish the next deployment base before commercial feature work.
- [x] **Phase 4: BYOK-First Access Layer** - Make user-supplied model APIs the first public product path.
- [ ] **Phase 5: Parser Bootstrap and Provider Abstraction** - Finish parser guardrails by replacing the Azure-backed platform parser path with Volcengine and cleaning legacy Azure runtime paths.
- [ ] **Phase 6: Accounts and Platform-Hosted APIs** - Build on the existing Clerk + credits baseline to finish hosted access and replace mock payment with ZPAY.
- [ ] **Phase 7: China-User Operational Fit** - Re-check the real bottlenecks for China-based users and operators before deeper infra commitments.

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
- [x] 01-01-PLAN.md - Environment security and regression baseline. (COMPLETED)
- [x] 01-02-PLAN.md - Fix serverless rate limiting gap. (COMPLETED)

### Phase 2: Data Persistence Migration
**Goal**: Implement a robust local storage solution to prevent data loss.
**Depends on**: Phase 1
**Requirements**: DATA-01, DATA-02
**Success Criteria** (what must be TRUE):
  1. User data is successfully migrated from LocalStorage to IndexedDB without loss.
  2. Application state is preserved across sessions and browser restarts.
  3. Subsequent updates do not break compatibility with existing IndexedDB data.
**Plans**: 2 plans
- [x] 02-01-PLAN.md - Database schema update and migration infrastructure. (COMPLETED)
- [x] 02-02-PLAN.md - Store and component integration. (COMPLETED)

### Phase 3: Minimal Cloudflare Migration
**Goal**: Move the deployment base off the current Vercel-first assumption without turning migration into a full-system rewrite.
**Depends on**: Phase 2
**Requirements**: DEP-01, DEP-02, DEP-03
**Success Criteria** (what must be TRUE):
  1. The public frontend path and the core `/api/generate` flow run on the new Cloudflare-oriented base.
  2. Streaming reliability, environment-variable handling, and core request protection still work after the move.
  3. Migration scope stays minimal: hosted accounts, payments, and parser abstraction are not forced into the same phase.
**Plans**: 3 plans
- [x] 03-01-PLAN.md - Establish the Cloudflare Worker shell, route-scoped runtime helpers, and Worker foundation tests. (COMPLETED)
- [x] 03-02-PLAN.md - Move `/api/get-token`, `/api/parse`, and `/api/generate` onto Worker handlers without changing teaching contracts. (COMPLETED)
- [x] 03-03-PLAN.md - Finish the single-Worker cutover, remove Vercel-first runtime assumptions, and resolve feedback explicitly. (COMPLETED)

### Phase 4: BYOK-First Access Layer
**Goal**: Ship the first public user path around user-supplied model APIs.
**Depends on**: Phase 3
**Requirements**: BYOK-01, BYOK-02, BYOK-03
**Success Criteria** (what must be TRUE):
  1. A user can configure their own model access and start using the product without platform-hosted inference.
  2. OpenAI-compatible BYOK is supported through one clean adapter path.
  3. Gemini remains functional through its separate adapter without breaking structured-output guarantees.
**Plans**: 2 plans
- [x] 04-01-PLAN.md - Persist BYOK access state, expose settings UI, and attach normalized access metadata from the frontend. (COMPLETED)
- [x] 04-02-PLAN.md - Resolve BYOK access on the backend, preserve migration-safe fallback paths, and update docs. (COMPLETED)

### Phase 5: Parser Bootstrap and Provider Abstraction
**Goal**: Keep parser cost platform-funded for early growth while making parser choice and cost an explicit architectural concern, and finish the provider transition away from Azure.
**Depends on**: Phase 3
**Requirements**: PARSE-01, PARSE-02, PARSE-03, PARSE-04
**Success Criteria** (what must be TRUE):
  1. Early users can rely on platform-managed parsing with low setup friction.
  2. Parser usage is observable and guardrailed instead of treated as a hidden free dependency.
  3. The codebase is prepared for alternative parser providers later.
  4. The platform-managed parser runtime no longer depends on Azure as the live default provider.
**Plans**: 3 plans
- [x] 05-01-PLAN.md - Add D1-backed parser quota truth, a shared parser access layer, and downgrade metadata. (COMPLETED OUTSIDE GSD)
- [x] 05-02-PLAN.md - Add Settings quota visibility, downgraded-analysis warning UX, and supporting docs/tests. (COMPLETED OUTSIDE GSD)
- [ ] 05-03-PLAN.md - Replace the Azure-backed platform parser with Volcengine, preserve the block contract, and remove legacy Azure runtime paths. (NEXT)

### Phase 6: Accounts and Platform-Hosted APIs
**Goal**: Introduce login and paid hosted-model access as a second product track, using the existing Clerk + credits baseline as the starting point.
**Depends on**: Phase 3, Phase 4
**Requirements**: ACCT-01, HOST-01, HOST-02
**Success Criteria** (what must be TRUE):
  1. User identity, entitlement, and hosted access can be managed consistently.
  2. Hosted-model access coexists cleanly with BYOK instead of replacing it.
  3. The launch mode for hosted access is intentionally scoped: waitlist, invite-only, or direct paid rollout.
  4. Mock payment is replaced by ZPAY without turning the product into a full billing platform.
**Plans**: 0 plans

### Phase 7: China-User Operational Fit
**Goal**: Validate that the product can actually be used and operated reliably in the target geography before deeper commitments.
**Depends on**: Phase 3, Phase 4
**Requirements**: CN-01, CN-02
**Success Criteria** (what must be TRUE):
  1. The real bottlenecks for China-based users are verified instead of assumed.
  2. Decisions about deeper mainland-specific infrastructure are made only after access, provider, and payment realities are rechecked.
  3. The operator has a clearer path for support, costs, and reliability management.
**Plans**: 0 plans

## Next Work

Current recommendation: work the next milestone in this order.

### Immediate Task List
- [x] Complete the Cloudflare-first migration.
- [x] Complete the BYOK-first access layer.
- [x] Land the parser quota/degraded/settings baseline.
- [x] Land the Clerk + hosted credits local baseline.
- [x] Lock the hosted product decisions: starter credits, success-only charging, 1 RMB = 30 credits, and ZPAY direction.
- [ ] Re-enter GSD on Phase 05 and plan only the remaining parser provider replacement to Volcengine.
- [ ] After Phase 05 is stable, re-enter GSD on Phase 06 to replace the mock payment adapter with ZPAY and finish hosted-access hardening.
- [ ] Keep parser BYOK and MinerU discussion deferred until after the platform-managed parser path is clean.

## Progress Table

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Environment & Core Stability | 2/2 | Completed | 2026-03-27 |
| 2. Data Persistence Migration | 2/2 | Completed | 2026-03-26 |
| 3. Minimal Cloudflare Migration | 3/3 | Completed | 2026-04-04 |
| 4. BYOK-First Access Layer | 2/2 | Completed | 2026-04-04 |
| 5. Parser Bootstrap and Provider Abstraction | 2/3 | In Progress | - |
| 6. Accounts and Platform-Hosted APIs | 0/0 | Planned | - |
| 7. China-User Operational Fit | 0/0 | Planned | - |
