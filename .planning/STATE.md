# Project State

## Project Reference

See: `.planning/PROJECT.md`

**Core Value:** Teacher-like PDF tutoring with controllable model access
**Current Focus:** Close out the now-complete Phase 08 work in GSD state and prepare the next separate model-configuration phase or milestone.

## Current Position

**Current Phase:** Phase 08 completed
**Current Phase Name:** Parser Reliability and LlamaParse BYOK
**Total Phases:** 8
**Completed Phases:** 8
**Current Plan:** All Phase 08 plans completed
**Total Plans in Phase:** 3
**Status:** Phase 08 code, docs, and local verification are complete; the next structured step is milestone closeout or a new model-configuration phase.
**Last Activity:** 2026-04-09
**Last Activity Description:** Completed Phase 08 parser cleanup, Volcengine hardening, `LlamaParse` BYOK integration, and the supporting docs/tests
**Progress:** Foundational runtime, BYOK, parser guardrails, hosted access, Clerk auth, live ZPAY recharge, and the Phase 07 operational-fit evidence are in place; Phase 08 parser reliability and parser BYOK are now complete, and model cleanup remains intentionally separated into the next phase or milestone

## Performance Metrics

- Total completed phases: 8
- Total completed plans: 18
- Latest completed phase: 08 Parser Reliability and LlamaParse BYOK

## Accumulated Context

### Decisions

- The first public version should be BYOK-first.
- Minimal Cloudflare migration should happen before the next commercialization-critical features.
- Platform-hosted APIs remain a secondary track for paid users.
- Early document parsing can be platform-funded to reduce activation friction.
- Platform-managed parser stays on `Volcengine`, and `Platform API` users should not configure their own parser.
- `My API` should support parser BYOK now, starting with `LlamaParse`.
- If `My API` has no parser configured, the product should keep the current no-parser degraded analysis path.
- User-visible parser quotas and parser trial semantics should be removed; parser protections should remain internal infrastructure guardrails only.
- Phase 08 landed as three plans: parser quota removal, `LlamaParse` BYOK wiring, and parser error-taxonomy hardening.
- Route throttling, platform parser failures, and BYOK parser failures now use separate error classes instead of one misleading generic rate-limit message.
- Hosted payment direction is ZPAY.
- Real 1 RMB ZPAY recharge has already been verified in production-like flow and credited 30.
- Phase 07 should not recommend or auto-switch providers for the user; it should make real access constraints and fallbacks explicit.
- `/docs` is the canonical long-term record; `.planning` should stay focused on active GSD work.
- Parser stabilization must land before the separate model-configuration cleanup phase.

### Pending Todos

- Run deployed/manual verification for a longer `Platform API` slide analysis and confirm no parser daily-limit UX remains.
- Run deployed/manual verification for `My API` with `LlamaParse` configured and then removed, confirming the degraded no-parser fallback still works.
- Define the separate model-configuration cleanup phase or next milestone now that Phase 08 is complete.

### Blockers/Concerns

- Volcengine upstream rate/queue behavior is now classified more accurately, but live capacity behavior still needs continued observation in deployed traffic.
- The next phase must normalize provider-specific model parameters, including Gemini thinking controls, without collapsing the now-clean parser boundaries.

### Roadmap Evolution

- Phase 8 executed and completed: Parser Reliability and LlamaParse BYOK.

## Session Continuity

**Last session:** 2026-04-09
**Stopped At:** Phase 08 implementation and local verification complete; next resume point is milestone closeout or the next model-configuration phase definition
**Resume file:** .planning/ROADMAP.md
