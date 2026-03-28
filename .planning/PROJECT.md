# Project: SlideTutor AI

## Context
SlideTutor AI is an AI-powered assistant specifically designed for slide-based learning (PDF slides). After multiple iterations, the project is now entering a high-refinement phase. The focus is shifting from feature-bloat to "True Needs" (真需求) — polishing the core experience to be comfortable, natural, and immersive for students.

## What This Is
A high-quality, tasteful, and functionally accurate slide learning assistant that integrates reading and AI explanation seamlessly. It aims to solve the disjointed experience found in current AI tools by providing a deeply immersive and educationally optimized environment.

## Core Value
**Immersive Learning Excellence**: Providing a distraction-free, smooth, and educationally sound reading experience where the AI explanation feels like a natural extension of the content, not a separate task.

## Requirements

### Validated
- ✓ PDF Slide Rendering and Navigation (existing)
- ✓ AI Explanation Integration (existing)
- ✓ Basic UI/UX Framework (existing)
- ✓ **API Security** (SEC-01): Environment variables secured and validated. (Validated in Phase 1)
- ✓ **Stability Guard** (STAB-01, STAB-02): Core logic clean; regression baseline established. (Validated in Phase 1)
- ✓ **Data Persistence** (DATA-01, DATA-02): Migrated from fragile localStorage to robust IndexedDB. (Validated in Phase 2)

### Active
- [ ] **Smoothing Interaction**: Fix "jittery" or overly aggressive animations that distract from reading.
- [ ] **UI Harmony**: Refine UI placement and visual aesthetics for better ergonomics and comfort.
- [ ] **UX Polish**: Improve "unhandy" interactions that feel unnatural to the user.
- [ ] **Immersive Visuals**: Enhance the "Immersive Reading" mode to be truly distraction-free.

### Out of Scope
- [ ] Feature Bloat: No "pseudo-needs" or redundant functions added just for the sake of it.
- [ ] Breaking Changes: Any change that compromises the existing "clean" logical framework.
- [ ] Public Cloud Storage: (Hypothesis) Keep data local-first unless explicit sync is requested.

## Key Decisions
| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Focus on "True Needs" | Avoid pseudo-feature bloat to maintain high quality. | — Active |
| Immersive Visuals First | User prioritizes "Visual Immersion" as the primary aesthetic goal. | — Active |
| Fine-Grained Iteration | "Fine" granularity chosen to allow surgical refinement of interactions. | — Active |
| Data-Safety First | Prevent data loss from user error; ensure backward compatibility. | — Active |
| API Security Priority | Mandatory protection of expensive/sensitive AI service keys. | — Active |

## Evolution
This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd:transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd:complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: March 27, 2026 after Phase 1 and Phase 2 completion*
