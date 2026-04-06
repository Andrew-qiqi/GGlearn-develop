# Requirements: SlideTutor AI

## Active Product-Direction Requirements

### Deployment and Platform Base
- [x] **DEP-01**: Establish a minimal Cloudflare-oriented deployment path for the public app.
- [x] **DEP-02**: Preserve core streaming generation behavior after the migration.
- [x] **DEP-03**: Preserve request protection, environment-variable handling, and operational observability during the migration.

### BYOK-First Access
- [x] **BYOK-01**: The first public version must support user-supplied model APIs as the primary access path.
- [x] **BYOK-02**: OpenAI-compatible user endpoints should use one shared adapter path rather than per-provider feature forks.
- [x] **BYOK-03**: Gemini must remain supported through a separate adapter without breaking structured-output behavior.

### Parser Strategy
- [x] **PARSE-01**: Early users should be able to use platform-managed document parsing with low setup friction.
- [x] **PARSE-02**: Parser usage and cost must become observable and controllable instead of relying on accidental free quota.
- [x] **PARSE-03**: Parser providers must be abstracted so Azure is no longer the only implicit path.
- [x] **PARSE-04**: Users who do not provide their own parser access should be subject to explicit platform-managed parsing limits.

### Accounts and Hosted Access
- [x] **ACCT-01**: The system must be able to add login and user identity without undoing the BYOK-first launch path.
- [x] **HOST-01**: Platform-hosted model access must coexist with BYOK rather than replace it.
- [x] **HOST-02**: Hosted access must have an intentionally scoped launch mode and cost boundary.

### China-User Operational Fit
- [ ] **CN-01**: Re-check actual reliability bottlenecks for China-based users across model access, parser access, and streaming.
- [ ] **CN-02**: Re-check operator-side realities for payments, support, and deployment convenience before deeper infra commitments.
- [ ] **CN-03**: China-based requests must avoid assuming Gemini API availability, because Gemini can reject requests with `User location is not supported for the API use`.

## Validated Requirements
- [x] **DATA-01**: IndexedDB-based local persistence replaced the old fragile localStorage path.
- [x] **DATA-02**: Persistence compatibility and migration safety were added for existing data.
- [x] **SEC-01**: API security baseline exists for protected generation access.
- [x] **STAB-01**: Core logic guardrails were established for safer iteration.
- [x] **STAB-02**: A regression baseline exists for core functionality.

## Open Commercial Questions
- [ ] Should hosted APIs launch as waitlist-only, invite-only, or direct paid access?
- [ ] How long should parser cost stay platform-funded before introducing stricter usage controls?
- [ ] When Phase 06 resumes, what is the minimum ZPAY webhook/order model that keeps credits accounting safe without overbuilding billing?

## Locked Commercial Decisions
- [x] Early BYOK is fully free.
- [x] Platform-funded parsing remains available in the early public version, but users who do not provide their own parser access will have explicit request limits.
- [x] New users receive a one-time non-expiring `10 credits` starter allowance.
- [x] Hosted pricing weights are fixed for the current product version: `Analyze = 3`, `Follow-up = 1`, `Quiz generate = 1`, `Quiz answer analysis = 1`.
- [x] Hosted usage is deducted only after success.
- [x] Recharge stays free-form with a minimum of `1 RMB`, fixed at `1 RMB = 30 credits`, and credits do not expire.
- [x] ZPAY is the chosen payment direction for hosted recharge.

## Out of Scope for the Next Milestone
- [ ] Full commercialization stack in one implementation phase.
- [ ] Full parser BYOK in the first public release.
- [ ] Deep mainland-specific infrastructure before validating actual bottlenecks.
- [ ] Product changes that weaken teaching quality, context continuity, or structured-output reliability.
- [ ] Mixing parser provider replacement and ZPAY integration into one GSD planning pass.

---
## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| DEP-01 | Phase 3 | Completed |
| DEP-02 | Phase 3 | Completed |
| DEP-03 | Phase 3 | Completed |
| BYOK-01 | Phase 4 | Completed |
| BYOK-02 | Phase 4 | Completed |
| BYOK-03 | Phase 4 | Completed |
| PARSE-01 | Phase 5 | Completed |
| PARSE-02 | Phase 5 | Completed |
| PARSE-03 | Phase 5 | Completed |
| PARSE-04 | Phase 5 | Completed |
| ACCT-01 | Phase 6 | Completed |
| HOST-01 | Phase 6 | Completed |
| HOST-02 | Phase 6 | Completed |
| CN-01 | Phase 7 | Pending |
| CN-02 | Phase 7 | Pending |
| CN-03 | Phase 7 | Pending |
| DATA-01 | Phase 2 | Completed |
| DATA-02 | Phase 2 | Completed |
| SEC-01 | Phase 1 | Completed |
| STAB-01 | Phase 1 | Completed |
| STAB-02 | Phase 1 | Completed |
