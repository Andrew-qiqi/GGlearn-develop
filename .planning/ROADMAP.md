# Roadmap: SlideTutor AI

## Phases
- [x] **Phase 1: Environment & Core Stability** - Secure API keys and establish regression safety.
- [x] **Phase 2: Data Persistence Migration** - Transition from LocalStorage to IndexedDB for robust data handling.
- [x] **Phase 3: Minimal Cloudflare Migration** - Establish the next deployment base before commercial feature work.
- [x] **Phase 4: BYOK-First Access Layer** - Make user-supplied model APIs the first public product path.
- [x] **Phase 5: Parser Bootstrap and Provider Abstraction** - Finish parser guardrails by replacing the Azure-backed platform parser path with Volcengine and cleaning legacy Azure runtime paths.
- [x] **Phase 6: Accounts and Platform-Hosted APIs** - Build on the existing Clerk + credits baseline to finish hosted access and replace mock payment with ZPAY.
- [x] **Phase 7: China-User Operational Fit** - Re-check the real bottlenecks for China-based users and operators before deeper infra commitments.
- [x] **Phase 8: Parser Reliability and LlamaParse BYOK** - Stabilize the live parser path, remove misleading parser quota behavior, and add a dedicated parser BYOK path for `My API`.
- [ ] **Phase 9: Model Capability Registry and Parameter Hardening** - Centralize model capability truth, harden provider-parameter generation, and stabilize structured-output execution.
- [x] **Phase 10: Dead Task Cleanup and Hosted Task Surface Alignment** - Remove dead task residue and align hosted task availability with the true active task surface. (completed 2026-04-10)

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
- [x] 05-03-PLAN.md - Replace the Azure-backed platform parser with Volcengine, preserve the block contract, and remove legacy Azure runtime paths. (COMPLETED)

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
**Completed**: 2026-04-06

### Phase 7: China-User Operational Fit
**Goal**: Validate that the product can actually be used and operated reliably in the target geography before deeper commitments.
**Depends on**: Phase 3, Phase 4
**Requirements**: CN-01, CN-02, CN-03
**Success Criteria** (what must be TRUE):
  1. The real bottlenecks for China-based users are verified instead of assumed.
  2. Decisions about deeper mainland-specific infrastructure are made only after access, provider, and payment realities are rechecked.
  3. The operator has a clearer path for support, costs, and reliability management.
**Plans**: 3 plans
- [x] 07-01-PLAN.md - Normalize China-user access errors and make `My API` / `Platform API` assumptions explicit without forcing recommendations. (COMPLETED)
- [x] 07-02-PLAN.md - Add operator-grade observability and a China-operator smoke checklist for auth, parser, credits, and recharge. (COMPLETED)
- [x] 07-03-PLAN.md - Create the operational-fit report and decision gate that determines whether parser BYOK / MinerU should stay deferred. (COMPLETED)
**Completed**: 2026-04-07

### Phase 8: Parser Reliability and LlamaParse BYOK
**Goal**: Remove misleading parser quota behavior from the live product, harden the Volcengine platform parser path, and add `LlamaParse` as the first dedicated parser BYOK provider for `My API` without breaking degraded fallback.
**Depends on**: Phase 4, Phase 5, Phase 7
**Requirements**: PARSE-05, PARSE-06, PARSE-07
**Success Criteria** (what must be TRUE):
  1. `Platform API` long-document analysis no longer fails because of user-visible parser quota semantics, and parser-related errors are classified by their real source.
  2. `My API` no longer borrows the platform parser; users can configure `LlamaParse`, and if no parser is configured the existing no-parser degraded analysis still works.
  3. Parser routing, provider adapters, and settings boundaries remain simple and modular: platform parser, BYOK parser, and degraded fallback are clearly separated.
**Plans**: 3 plans
- [x] 08-01-PLAN.md - Remove misleading parser quota semantics, stop `My API` from borrowing the platform parser, and normalize user-facing parser errors. (COMPLETED)
- [x] 08-02-PLAN.md - Add optional parser BYOK settings for `My API` and implement the first `LlamaParse` adapter without breaking degraded fallback. (COMPLETED)
- [x] 08-03-PLAN.md - Harden the remaining Volcengine platform parser path, separate route vs parser failures, and sync the final parser ownership docs. (COMPLETED)
**Completed**: 2026-04-09

## Next Work

Current recommendation: Phase 09 is now defined and planned; execute the model-capability hardening work next, then follow with Phase 10 task-surface cleanup and hosted alignment.

### Immediate Task List
- [x] Complete the Cloudflare-first migration.
- [x] Complete the BYOK-first access layer.
- [x] Land the parser quota/degraded/settings baseline.
- [x] Land the Clerk + hosted credits local baseline.
- [x] Lock the hosted product decisions: starter credits, success-only charging, 1 RMB = 30 credits, and ZPAY direction.
- [x] Re-enter GSD on Phase 05 and finish the remaining parser provider replacement to Volcengine.
- [x] Re-enter GSD on Phase 06 to replace the mock payment adapter with ZPAY and finish hosted-access hardening.
- [x] Execute Phase 7 plans to validate the real China-user operational bottlenecks after live Clerk + ZPAY rollout.
- [x] Record the live operational-fit findings in the operator report and checklist docs.
- [x] Plan and execute Phase 08 to remove misleading parser limits, harden the Volcengine platform parser path, and add `LlamaParse` for `My API`.
- [x] Define the next separate phase or milestone for Gemini/model-configuration cleanup after parser stability work lands.
- [x] Execute Phase 09 to centralize model capability truth, harden provider parameters, and stabilize structured-output behavior. (completed 2026-04-10)
- [ ] Execute Phase 10 to remove dead task residue and align hosted task availability semantics.

## Progress Table

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Environment & Core Stability | 2/2 | Completed | 2026-03-27 |
| 2. Data Persistence Migration | 2/2 | Completed | 2026-03-26 |
| 3. Minimal Cloudflare Migration | 3/3 | Completed | 2026-04-04 |
| 4. BYOK-First Access Layer | 2/2 | Completed | 2026-04-04 |
| 5. Parser Bootstrap and Provider Abstraction | 3/3 | Completed | 2026-04-06 |
| 6. Accounts and Platform-Hosted APIs | 0/0 | Completed | 2026-04-06 |
| 7. China-User Operational Fit | 3/3 | Completed | 2026-04-07 |
| 8. Parser Reliability and LlamaParse BYOK | 3/3 | Completed | 2026-04-09 |
| 9. Model Capability Registry and Parameter Hardening | 0/3 | Planned | — |
| 10. Dead Task Cleanup and Hosted Task Surface Alignment | 2/2 | Complete   | 2026-04-10 |

### Phase 8: Parser Reliability and LlamaParse BYOK

**Goal**: Remove misleading parser quota behavior from the live product, harden the Volcengine platform parser path, and add `LlamaParse` as the first dedicated parser BYOK provider for `My API` without breaking degraded fallback.
**Depends on**: Phase 4, Phase 5, Phase 7
**Requirements**: PARSE-05, PARSE-06, PARSE-07
**Success Criteria** (what must be TRUE):
  1. `Platform API` long-document analysis no longer fails because of user-visible parser quota semantics, and parser-related errors are classified by their real source.
  2. `My API` no longer borrows the platform parser; users can configure `LlamaParse`, and if no parser is configured the existing no-parser degraded analysis still works.
  3. Parser routing, provider adapters, and settings boundaries remain simple and modular: platform parser, BYOK parser, and degraded fallback are clearly separated.
**Plans:** 3 plans

Plans:
- [x] 08-01-PLAN.md - Remove misleading parser quota semantics, stop `My API` from borrowing the platform parser, and normalize user-facing parser errors.
- [x] 08-02-PLAN.md - Add optional parser BYOK settings for `My API` and implement the first `LlamaParse` adapter without breaking degraded fallback.
- [x] 08-03-PLAN.md - Harden the remaining Volcengine platform parser path, separate route vs parser failures, and sync the final parser ownership docs.

**Completed:** 2026-04-09

### Phase 9: Model Capability Registry and Parameter Hardening

**Goal**: Establish one backend-owned model capability truth, harden provider-parameter generation, and remove fragile model-setting failures such as unsupported Gemini thinking controls and truncated structured-output responses.
**Depends on**: Phase 4, Phase 6, Phase 8
**Requirements**: MODEL-01, MODEL-02, MODEL-03
**Success Criteria** (what must be TRUE):
  1. The product has one global model hard-constraint baseline for active tasks, and models that fail it are rejected before normal execution instead of failing mid-task.
  2. Provider-specific runtime parameters are generated from model capability truth rather than scattered task-specific conditionals, so unsupported settings such as Gemini `thinkingLevel` on incompatible models no longer explode at runtime.
  3. Structured-output flows, especially `distill`, no longer frequently fail because of token-budget and parameter misconfiguration, while `quickExplain` / Focus mode quality remains stable.
**Plans**: 0 plans

Plans:
- [ ] TBD (run /gsd:plan-phase 9 to break down)

### Phase 10: Dead Task Cleanup and Hosted Task Surface Alignment

**Goal**: Remove expired task residue, define one active task truth, and align hosted task availability so `Platform API` and `My API` expose the same supported learning actions with explicit hosted semantics.
**Depends on**: Phase 6, Phase 9
**Requirements**: TASK-01, HOST-03
**Success Criteria** (what must be TRUE):
  1. `evaluate_note` and any other expired task residue are removed from code, types, docs, and tests.
  2. The active task surface is explicit and consistent across backend routing, pricing, frontend guardrails, and docs.
  3. `regenerate_chunk` and `regenerate_followup` are no longer inconsistently blocked in hosted mode and are both mapped to hosted action `card_regenerate = 1 credit`.
**Plans**: 0 plans

Plans:
- [ ] TBD (run /gsd:plan-phase 10 to break down)
