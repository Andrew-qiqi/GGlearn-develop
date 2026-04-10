---
gsd_state_version: 1.0
milestone: v1.0
milestone_name: milestone
current_phase: 09
current_phase_name: model-capability-registry-and-parameter-hardening
current_plan: 1
status: executing
stopped_at: Phase 09 added to roadmap and waiting for concrete plans generated from the approved phase brief
last_updated: "2026-04-10T09:26:19.239Z"
last_activity: 2026-04-10
progress:
  total_phases: 10
  completed_phases: 3
  total_plans: 19
  completed_plans: 11
---

# Project State

## Project Reference

See: `.planning/PROJECT.md`

**Core Value:** Teacher-like PDF tutoring with controllable model access
**Current Focus:** Phase 09 — model-capability-registry-and-parameter-hardening

## Current Position

Phase: 09 (model-capability-registry-and-parameter-hardening) — EXECUTING
Plan: 1 of 3
**Current Phase:** 09
**Current Phase Name:** model-capability-registry-and-parameter-hardening
**Total Phases:** 10
**Completed Phases:** 8
**Current Plan:** 1
**Total Plans in Phase:** 3
**Status:** Executing Phase 09
**Last Activity:** 2026-04-10
**Last Activity Description:** Phase 09 execution started
**Progress:** Foundational runtime, BYOK, parser guardrails, hosted access, Clerk auth, live ZPAY recharge, and the Phase 07 operational-fit evidence are in place; the current focus is centralizing model capability truth, hardening provider parameters, fixing structured-output truncation, and then cleaning the hosted task surface

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
- Phase 09 should establish one backend-owned model capability truth instead of separate `My API` / `Platform API` admission logic.
- `thinking` is a soft capability only and must not remain a hard admission constraint.
- `native structured output` is a hard constraint for the current artifact-dependent active tasks.
- `distill` truncation and unsupported provider parameters are part of the same parameter-hardening problem and should be solved without degrading Focus mode quality.
- Phase 10 should remove `evaluate_note` and align `regenerate_*` support across hosted and BYOK paths.
- Phase 10 should map `regenerate_chunk` and `regenerate_followup` to one hosted action: `card_regenerate = 1 credit`.

### Pending Todos

- Run deployed/manual verification for a longer `Platform API` slide analysis and confirm no parser daily-limit UX remains.
- Run deployed/manual verification for `My API` with `LlamaParse` configured and then removed, confirming the degraded no-parser fallback still works.
- Review and refine the three Phase 09 draft plans before execution.
- Decide whether the front-end hidden help / credits-hover reveal for card regenerate belongs in Phase 10 or a later UI cleanup step.

### Blockers/Concerns

- Volcengine upstream rate/queue behavior is now classified more accurately, but live capacity behavior still needs continued observation in deployed traffic.
- The next phase must normalize provider-specific model parameters, including Gemini thinking controls, without collapsing the now-clean parser boundaries.
- The roadmap and requirements were missing formal entries for the new model-capability and task-surface phases; this is now being repaired during planning setup.

### Roadmap Evolution

- Phase 8 executed and completed: Parser Reliability and LlamaParse BYOK.
- Phase 9 added: Model Capability Registry and Parameter Hardening.
- Phase 10 added: Dead Task Cleanup and Hosted Task Surface Alignment.

## Session Continuity

**Last session:** 2026-04-09
**Stopped At:** Phase 09 added to roadmap and waiting for concrete plans generated from the approved phase brief
**Resume file:** .planning/ROADMAP.md
