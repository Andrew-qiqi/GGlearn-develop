# Requirements: SlideTutor AI

## Active Product-Direction Requirements

### Deployment and Platform Base
- [ ] **DEP-01**: Establish a minimal Cloudflare-oriented deployment path for the public app.
- [ ] **DEP-02**: Preserve core streaming generation behavior after the migration.
- [ ] **DEP-03**: Preserve request protection, environment-variable handling, and operational observability during the migration.

### BYOK-First Access
- [ ] **BYOK-01**: The first public version must support user-supplied model APIs as the primary access path.
- [ ] **BYOK-02**: OpenAI-compatible user endpoints should use one shared adapter path rather than per-provider feature forks.
- [ ] **BYOK-03**: Gemini must remain supported through a separate adapter without breaking structured-output behavior.

### Parser Strategy
- [ ] **PARSE-01**: Early users should be able to use platform-managed document parsing with low setup friction.
- [ ] **PARSE-02**: Parser usage and cost must become observable and controllable instead of relying on accidental free quota.
- [ ] **PARSE-03**: Parser providers must be abstracted so Azure is no longer the only implicit path.
- [ ] **PARSE-04**: Users who do not provide their own parser access should be subject to explicit platform-managed parsing limits.

### Accounts and Hosted Access
- [ ] **ACCT-01**: The system must be able to add login and user identity without undoing the BYOK-first launch path.
- [ ] **HOST-01**: Platform-hosted model access must coexist with BYOK rather than replace it.
- [ ] **HOST-02**: Hosted access must have an intentionally scoped launch mode and cost boundary.

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

## Locked Commercial Decisions
- [x] Early BYOK is fully free.
- [x] Platform-funded parsing remains available in the early public version, but users who do not provide their own parser access will have explicit request limits.

## Out of Scope for the Next Milestone
- [ ] Full commercialization stack in one implementation phase.
- [ ] Full parser BYOK in the first public release.
- [ ] Deep mainland-specific infrastructure before validating actual bottlenecks.
- [ ] Product changes that weaken teaching quality, context continuity, or structured-output reliability.

---
## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| DEP-01 | Phase 3 | Pending |
| DEP-02 | Phase 3 | Pending |
| DEP-03 | Phase 3 | Pending |
| BYOK-01 | Phase 4 | Pending |
| BYOK-02 | Phase 4 | Pending |
| BYOK-03 | Phase 4 | Pending |
| PARSE-01 | Phase 5 | Pending |
| PARSE-02 | Phase 5 | Pending |
| PARSE-03 | Phase 5 | Pending |
| ACCT-01 | Phase 6 | Pending |
| HOST-01 | Phase 6 | Pending |
| HOST-02 | Phase 6 | Pending |
| CN-01 | Phase 7 | Pending |
| CN-02 | Phase 7 | Pending |
| DATA-01 | Phase 2 | Completed |
| DATA-02 | Phase 2 | Completed |
| SEC-01 | Phase 1 | Completed |
| STAB-01 | Phase 1 | Completed |
| STAB-02 | Phase 1 | Completed |
