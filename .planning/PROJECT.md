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

### Active
- [ ] **Minimal Cloudflare Migration**: establish a non-Vercel-first deployment base before building the next commercial features.
- [ ] **BYOK-First Public Launch**: make user-supplied model APIs the primary entry path for the first public version.
- [ ] **OpenAI-Compatible BYOK Architecture**: support user-supplied OpenAI-compatible endpoints cleanly while keeping Gemini on its own adapter.
- [ ] **Parser Bootstrap Strategy**: keep document parsing platform-funded in the early user-acquisition stage, but turn it into an explicit costed subsystem.
- [ ] **Parser Provider Abstraction**: remove the hidden Azure-default assumption and prepare for alternative parsers later.
- [ ] **Account and Hosted API Foundation**: prepare login, entitlement, and hosted-API architecture as a follow-on track after the Cloudflare base and BYOK path are stable.
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
*Last updated: April 4, 2026 after project-direction reset for Cloudflare, BYOK, and commercialization sequencing*
