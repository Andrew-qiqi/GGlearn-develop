# Phase 10: Dead Task Cleanup and Hosted Task Surface Alignment - Context

**Gathered:** 2026-04-10
**Status:** Ready for planning
**Source:** PRD Express Path (`docs/discuss/phases/10-dead-task-cleanup-and-hosted-task-surface-alignment-brief.md`)

<domain>
## Phase Boundary

Phase 10 is a task-surface cleanup and hosted-semantic alignment phase.

This phase should:

- remove expired task residue, especially `evaluate_note`
- define one explicit active task truth across code, docs, and tests
- stop treating hosted regenerate restrictions as unexplained legacy if/else
- align `Platform API` and `My API` so both expose the same supported learning tasks
- add clear hosted action and pricing semantics for card-level regenerate flows

This phase must not become:

- another model-capability phase
- a payment-system redesign
- a broad frontend redesign
- a task taxonomy rewrite across the whole product

</domain>

<decisions>
## Implementation Decisions

### Locked dead-task cleanup direction
- `evaluate_note` is a dead task and must be fully removed.
- Do not leave type residue, routing residue, test residue, or docs residue for `evaluate_note`.

### Locked hosted task-surface direction
- `regenerate_chunk` and `regenerate_followup` must no longer be blocked in hosted mode.
- `Platform API` and `My API` should expose the same current learning tasks.
- Task-surface differences belong to product policy and billing semantics, not model-capability logic.

### Locked hosted action semantics
- Runtime task names remain:
  - `regenerate_chunk`
  - `regenerate_followup`
- Hosted action names do not need to mirror task names exactly.
- Both regenerate tasks should map to one hosted action:
  - `card_regenerate`
- `card_regenerate` has fixed pricing:
  - `1 credit`

### Locked scope restraint
- This phase should not redesign the whole regenerate UX.
- Hidden-help / credits-hover UI reorganization may be acknowledged, but only minimal guardrail/UI work should land here unless directly required by the hosted alignment.

### the agent's Discretion
- The exact place where the repo records the canonical active task list is open, as long as there is one clear truth.
- The exact wording for hosted UI copy and credit prompts is open, as long as it clearly explains card-level regenerate charging.
- The exact internal helper/function names for task-to-hosted-action mapping are open.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Product scope and locked decisions
- `docs/discuss/project-brief.md` - Project-level product direction and hosted/BYOK coexistence.
- `docs/discuss/phases/09-model-capability-registry-and-parameter-hardening-brief.md` - Upstream phase boundary so task-surface work stays separate from capability logic.
- `docs/discuss/phases/10-dead-task-cleanup-and-hosted-task-surface-alignment-brief.md` - Phase-specific objective and locked decisions.
- `docs/discuss/phases/06-login-hosted-access-and-credit-brief.md` - Original hosted credits/product decisions that introduced the current task-surface mismatch.

### Planning truth
- `.planning/PROJECT.md`
- `.planning/ROADMAP.md`
- `.planning/REQUIREMENTS.md`
- `.planning/STATE.md`

### Existing implementation anchors
- `SlideTutor-AI/api/lib/generateService.ts` - Current unsupported hosted task blocking and task routing.
- `SlideTutor-AI/api/lib/platformAccess/types.ts` - Current hosted action and unsupported-action type truth.
- `SlideTutor-AI/api/lib/platformAccess/pricing.ts` - Hosted pricing map and unsupported-action rejection.
- `SlideTutor-AI/api/lib/platformAccess/pricing.test.ts` - Current pricing/unsupported-action coverage.
- `SlideTutor-AI/src/lib/platformAccess/pricing.ts` - Frontend pricing display truth.
- `SlideTutor-AI/src/hooks/useChunkRegenerate.ts` - Frontend hosted blocking for chunk regenerate.
- `SlideTutor-AI/src/hooks/useFollowUp.ts` - Frontend hosted blocking for regenerate follow-up.
- `SlideTutor-AI/src/hooks/useSlideAnalysis.ts` - Hosted analyze flow for context on task/action semantics.

### Existing docs
- `docs/backend/api-design.md`
- `docs/frontend/data-flow.md`
- `docs/backend/platform-model-configuration.md`
- `docs/changelog/CHANGELOG_TECH.md`

</canonical_refs>

<specifics>
## Specific Ideas

- Current hosted regenerate blocking is legacy product scope, not model capability.
- Because the front-end may later hide or regroup regenerate entry points in a hover/help area, hosted action naming should follow object semantics rather than a generic `regenerate` verb.
- `card_regenerate` is the current chosen hosted action umbrella for:
  - `regenerate_chunk`
  - `regenerate_followup`
- Billing should stay simple in this phase: both regenerate tasks cost `1 credit`.

</specifics>

<deferred>
## Deferred Ideas

- broader frontend menu / hover-help redesign
- future regenerate actions for quiz or full analyze refresh
- broader task taxonomy cleanup beyond current dead-task removal and hosted alignment

</deferred>

---

*Phase: 10-dead-task-cleanup-and-hosted-task-surface-alignment*
*Context gathered: 2026-04-10 via PRD Express Path*
