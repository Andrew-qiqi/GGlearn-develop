# Project State

## Project Reference

See: `.planning/PROJECT.md`

**Core Value:** Teacher-like PDF tutoring with controllable model access
**Current Focus:** Re-enter GSD on the remaining Phase 05 parser provider work: swap the platform parser from Azure to Volcengine and clean legacy Azure paths before continuing Phase 06

## Current Position

**Current Phase:** 05
**Current Phase Name:** Parser Bootstrap and Provider Abstraction
**Total Phases:** 7
**Completed Phases:** 4
**Current Plan:** Not started
**Total Plans in Phase:** 3
**Status:** Ready to plan Phase 05 remaining work
**Last Activity:** 2026-04-05
**Last Activity Description:** Verified the local Clerk + hosted credits happy path, then re-synced briefs and planning files so GSD can resume from the remaining parser-provider work instead of mixing parser and payment scopes
**Progress:** Foundational runtime, BYOK, parser guardrails, and hosted-access baseline are in place; the next structured planning step is Phase 05-03

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

- Plan Phase 05-03: replace Azure-backed platform parsing with Volcengine.
- Remove legacy Azure direct parser usage from the runtime path.
- Defer the remaining Phase 06 hosted-access hardening and ZPAY integration until Phase 05 is stable.

### Blockers/Concerns

- The parser quota and downgrade baseline exists, but the live provider path is still not cleanly off Azure.
- Hosted payment is still on a mock adapter and must not be mixed into the current parser cleanup.

## Session Continuity

**Last session:** 2026-04-05
**Stopped At:** Project brief, Phase 05 brief, and Phase 06 brief were re-synced; next resume point is `gsd-plan-phase 05` using the updated Phase 05 brief
**Resume file:** None
