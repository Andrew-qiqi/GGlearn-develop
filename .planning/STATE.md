# Project State

## Project Reference

See: `.planning/PROJECT.md`

**Core Value:** Teacher-like PDF tutoring with controllable model access
**Current Focus:** Execute the newly planned Phase 07 operational-fit work now that Clerk, Volcengine parser, and ZPAY are live enough to validate real China-user/operator friction.

## Current Position

**Current Phase:** 07
**Current Phase Name:** China-User Operational Fit
**Total Phases:** 7
**Completed Phases:** 6
**Current Plan:** Phase 07 planned
**Total Plans in Phase:** 3
**Status:** Ready to execute Phase 07
**Last Activity:** 2026-04-06
**Last Activity Description:** Closed Phase 06 with live ZPAY recharge verification and created Phase 07 planning artifacts for China-user operational-fit hardening
**Progress:** Foundational runtime, BYOK, parser guardrails, hosted access, Clerk auth, and live ZPAY recharge are in place; the next structured step is to execute the three Phase 07 plans

## Performance Metrics

- Total completed phases: 6
- Total completed plans: 11
- Latest completed phase: 06 Accounts and Platform-Hosted APIs

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

### Pending Todos

- Execute 07-01 to normalize region/provider access failures and add low-noise access guidance.
- Execute 07-02 to add observability parity and a China-operator smoke checklist for auth, parser, credits, and recharge.
- Execute 07-03 to create the operational-fit report and explicit decision gate for parser BYOK / MinerU.

### Blockers/Concerns

- `CN-03` is still real: the product must not assume Gemini API availability for China-based users.
- Credits/recharge/parser routes do not yet have the same request-level observability parity as `/api/generate`.
- Clerk build-time public key vs Worker runtime secret remains an operator footgun if not documented and smoke-tested together.

## Session Continuity

**Last session:** 2026-04-06
**Stopped At:** Phase 07 planning completed; next resume point is `gsd-execute-phase 07`
**Resume file:** None
