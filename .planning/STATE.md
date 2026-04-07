# Project State

## Project Reference

See: `.planning/PROJECT.md`

**Core Value:** Teacher-like PDF tutoring with controllable model access
**Current Focus:** Close Phase 07 cleanly and choose the next milestone from live evidence instead of speculative infrastructure expansion.

## Current Position

**Current Phase:** Phase 07 completed
**Current Phase Name:** China-User Operational Fit
**Total Phases:** 7
**Completed Phases:** 7
**Current Plan:** None active
**Total Plans in Phase:** 3
**Status:** Awaiting next milestone definition
**Last Activity:** 2026-04-07
**Last Activity Description:** Recorded live Phase 07 validation after successful production login, recharge, and parser verification, then closed the phase
**Progress:** Foundational runtime, BYOK, parser guardrails, hosted access, Clerk auth, live ZPAY recharge, and the Phase 07 operational-fit evidence are all in place; the next structured step is milestone selection rather than deeper infrastructure work

## Performance Metrics

- Total completed phases: 7
- Total completed plans: 15
- Latest completed phase: 07 China-User Operational Fit

## Accumulated Context

### Decisions

- The first public version should be BYOK-first.
- Minimal Cloudflare migration should happen before the next commercialization-critical features.
- Platform-hosted APIs remain a secondary track for paid users.
- Early document parsing can be platform-funded to reduce activation friction.
- Platform-managed parser should move to Volcengine.
- Future parser BYOK, if it happens, should prefer China-friendly options such as MinerU, but it is not the next implementation target.
- Hosted payment direction is ZPAY.
- Real 1 RMB ZPAY recharge has already been verified in production-like flow and credited 30.
- Phase 07 should not recommend or auto-switch providers for the user; it should make real access constraints and fallbacks explicit.
- `/docs` is the canonical long-term record; `.planning` should stay focused on active GSD work.
- Current evidence supports keeping parser BYOK and `MinerU` deferred.

### Pending Todos

- Review the completed Phase 07 report before defining the next milestone.
- Keep watching for repeated live evidence before reopening parser BYOK / `MinerU`.
- Capture exact production deploy hashes during future operational-fit validation runs.

### Blockers/Concerns

- No blocking operator or user-path failure is currently confirmed in the latest validation round.
- Volcengine service activation may not be instantaneous immediately after manual enablement; operators should allow for propagation before classifying it as a persistent outage.
- More live usage evidence is still needed before changing provider, parser, or payment scope.

## Session Continuity

**Last session:** 2026-04-07
**Stopped At:** Phase 07 completed; next resume point is next-milestone selection
**Resume file:** None
