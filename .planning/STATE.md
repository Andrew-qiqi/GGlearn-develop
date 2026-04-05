# Project State

## Project Reference

See: `.planning/PROJECT.md`

**Core Value:** Teacher-like PDF tutoring with controllable model access
**Current Focus:** Move into Phase 06 planning now that the Phase 05 Volcengine parser cutover is complete

## Current Position

**Current Phase:** 06
**Current Phase Name:** Accounts and Platform-Hosted APIs
**Total Phases:** 7
**Completed Phases:** 5
**Current Plan:** Not started
**Total Plans in Phase:** 0
**Status:** Ready to plan Phase 06
**Last Activity:** 2026-04-06
**Last Activity Description:** Completed Phase 05-03 by cutting the live parser runtime over to Volcengine, removing Azure parser runtime remnants, and syncing docs/tests/planning artifacts
**Progress:** Foundational runtime, BYOK, parser guardrails, and hosted-access baseline are in place; the next structured planning step is Phase 06

## Performance Metrics

- Total completed phases: 4
- Total completed plans: 11
- Latest completed phase: 04 BYOK-First Access Layer

## Accumulated Context

### Decisions

- The first public version should be BYOK-first.
- Minimal Cloudflare migration should happen before the next commercialization-critical features.
- Platform-hosted APIs remain a secondary track for paid users.
- Early document parsing can be platform-funded to reduce activation friction.
- Platform-managed parser should move to Volcengine.
- Future parser BYOK, if it happens, should prefer China-friendly options such as MinerU, but it is not the next implementation target.
- Hosted payment direction is ZPAY.
- `/docs` is the canonical long-term record; `.planning` should stay focused on active GSD work.

### Pending Todos

- Plan Phase 06 with the current Clerk + credits baseline as the starting point.
- Replace the mock payment adapter with ZPAY without building unnecessary billing surfaces.
- Keep parser BYOK deferred while hosted access and payment are stabilized.

### Blockers/Concerns

- Hosted payment is still on a mock adapter and must be replaced carefully with ZPAY.
- A live Volcengine AK/SK smoke test is still needed outside local unit coverage.

## Session Continuity

**Last session:** 2026-04-06
**Stopped At:** Phase 05-03 completed with code, docs, and validation synced; next resume point is `gsd-plan-phase 06 --prd docs/discuss/phases/06-login-hosted-access-and-credit-brief.md`
**Resume file:** None
