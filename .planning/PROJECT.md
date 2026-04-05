# Project: SlideTutor AI

## Context
SlideTutor AI is a PDF-centered AI learning assistant for slide-based study. After multiple refinement iterations, the project is now moving from internal experimentation toward a real public product path. The focus is shifting from isolated UX polish to sustainable deployment, controllable provider choices, and a commercialization-ready architecture for China-based users.

## What This Is
A high-quality slide learning assistant that turns static lecture PDFs into a guided, teacher-like learning experience. The product should preserve strong explanation quality and continuity while giving users clear control over model access and future paid service paths.

## Core Value
**Teacher-like PDF Tutoring with Controllable Access**: Preserve a strong, continuous learning experience while supporting both user-supplied APIs and future platform-hosted access in a maintainable architecture.

## Requirements

### Validated
- [x] PDF slide rendering and navigation.
- [x] AI explanation integration.
- [x] Basic UI and interaction foundation.
- [x] **API Security** (SEC-01): environment-variable handling and protected generation access exist.
- [x] **Stability Guard** (STAB-01, STAB-02): core logic guardrails and a regression baseline exist.
- [x] **Data Persistence** (DATA-01, DATA-02): local persistence migrated to IndexedDB with compatibility handling.
- [x] **Minimal Cloudflare Migration**: the public app and core APIs no longer depend on a Vercel-first runtime assumption.
- [x] **BYOK-First Public Launch Base**: user-supplied model APIs are now the primary public product path.
- [x] **OpenAI-Compatible BYOK Architecture**: user-supplied OpenAI-compatible endpoints use one shared adapter path while Gemini remains separate.
- [x] **Parser Guardrail Baseline**: parser quota truth, degraded fallback, and settings visibility exist.
- [x] **Hosted Access Baseline**: Clerk auth wiring and D1-backed hosted credits foundations exist.

### Active
- [x] **Parser Provider Replacement**: finish the platform parser transition from Azure to Volcengine and remove legacy Azure runtime dependencies.
- [x] **Parser Provider Abstraction Completion**: make parser abstraction real enough that Azure is no longer the implicit live default.
- [ ] **Account and Hosted API Completion**: finish hosted access on top of the existing Clerk + credits baseline.
- [ ] **Payment Provider Integration**: replace mock payment with ZPAY without turning the product into a full billing platform.
- [ ] **China-User Operational Fit**: re-check practical reliability bottlenecks for China-based users across model access, parser access, streaming, and payment.

### Out of Scope
- [ ] Full commercialization stack in one phase.
- [ ] Full parser BYOK in the first public version.
- [ ] Large infrastructure rewrite before proving the minimal Cloudflare path.
- [ ] Feature bloat that dilutes the core tutoring experience.

## Key Decisions
| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Structured-output first | Preserve stable contracts across providers and tasks. | Active |
| BYOK is mandatory | Needed for sustainable early usage and user control. | Locked |
| First public version is BYOK-first | Avoid coupling first launch to hosted inference and payments. | Locked |
| Minimal Cloudflare migration before next major features | Reduce rework across auth, payments, streaming, and provider access. | Locked |
| Platform-hosted APIs remain a parallel long-term track | Needed for less technical paid users, but not first-launch primary path. | Locked |
| Parser cost can be platform-funded early | Reduce setup friction and improve activation during user acquisition. | Locked |
| Parser providers must become an explicit abstraction | Azure free quota exhaustion proves the current hidden default is not durable. | Locked |
| Platform-managed parser should use Volcengine | Lower expected parsing cost and better fit for the current narrow block-extraction need. | Locked |
| Future parser BYOK is deferred | Avoid mixing provider-choice UX into the current parser cleanup phase. | Locked |
| Hosted access uses credits, not subscription-first packaging | Validate willingness to pay with minimal product complexity. | Locked |
| ZPAY is the payment direction | Better fit for the current China-heavy operator context. | Locked |

## Evolution
This document evolves at phase transitions and milestone boundaries.

**After each phase transition**:
1. Review which active requirements moved to validated.
2. Review whether any out-of-scope items should move into active scope.
3. Log new project-level decisions that affect future phases.
4. Re-check whether the core value still matches the real product direction.

**After each milestone**:
1. Review the deployment path and cost assumptions.
2. Re-check whether the current public entry path is still BYOK-first.
3. Audit parser strategy, user friction, and hosted-service readiness.
4. Update the document so roadmap and requirements stay aligned.

---
*Last updated: April 5, 2026 after re-syncing project direction, parser/provider scope, and hosted-access decisions before returning to GSD*
