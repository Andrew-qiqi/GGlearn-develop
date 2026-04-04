# Phase 03: Minimal Cloudflare Migration - Context

**Gathered:** 2026-04-04
**Status:** Ready for planning
**Source:** PRD Express Path (`docs/discuss/phases/03-minimal-cloudflare-migration-brief.md`)

<domain>
## Phase Boundary

This phase defines and prepares a minimal Cloudflare migration that moves the first public user path off the current Vercel-first assumption without turning migration into a full infrastructure rewrite.

The phase must migrate the public frontend path and the first-release user critical-path APIs onto Cloudflare so later BYOK-first work does not continue to grow on a Vercel-first base.

The phase is explicitly not a bundled commercialization phase. It does not include login, payments, hosted-model productization, or parser-provider abstraction beyond what is strictly needed to preserve the first-release learning flow.

</domain>

<decisions>
## Implementation Decisions

### Platform direction
- Cloudflare is the intended primary runtime direction for the next public product stage.
- The first migration is allowed to be minimal, but it must still de-risk the later BYOK-first launch.
- Short-term dual-platform transition is acceptable only as a temporary bridge, not as a steady-state operating model.

### Critical-path migration scope
- The first migration should prioritize moving the full first-release user critical path to Cloudflare.
- The critical path includes the public frontend path, `/api/generate`, `/api/get-token`, `/api/parse`, and any API required for a normal first-release PDF tutoring workflow.
- Non-critical-path endpoints may be deferred only if they do not affect the first-release learning flow.

### Runtime adaptation boundary
- `/api/generate` may receive a small, intentional server/runtime adaptation refactor during migration.
- That refactor exists only to make the runtime and streaming path compatible with Cloudflare.
- Do not use this phase to modify mature teaching business logic.
- Do not change teaching prompt intent, explanation behavior, structured artifact contracts, or frontend consumption contracts in this phase.

### Support-layer migration scope
- Support layers required by the current public runtime should be migrated with the first Cloudflare move rather than left as long-lived Vercel/Cloudflare split responsibilities.
- This includes the currently required authentication, rate limiting, environment-variable handling, proxy/IP treatment, logging/observability, and existing notification or email support that the current public runtime depends on.
- This does not include future login, billing, subscriptions, or hosted-model product systems.

### Scope protection
- Do not bundle Phase 04 BYOK-first implementation into this phase.
- Do not bundle Phase 05 parser abstraction into this phase except where parser behavior must continue to work for the critical path.
- Do not quietly expand this phase into a general platform rewrite.

### the agent's Discretion
- The exact Cloudflare runtime topology may be chosen during planning if it respects the locked boundaries above.
- The exact implementation pattern for streaming compatibility on Cloudflare may be chosen during research and planning.
- The specific internal adapter-layer refactor shape is left to the planner as long as it stays within runtime/platform concerns.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Product and phase scope
- `docs/discuss/project-brief.md` - Project-level locked product direction and sequencing decisions.
- `docs/discuss/phases/03-minimal-cloudflare-migration-brief.md` - Phase-specific migration boundary and locked decisions.
- `docs/discuss/phases/04-byok-first-access-layer-brief.md` - Downstream phase that this migration must de-risk.
- `docs/discuss/phases/05-parser-bootstrap-and-provider-abstraction-brief.md` - Parser follow-on phase that should not be accidentally absorbed here.

### GSD planning state
- `.planning/PROJECT.md` - Active project definition for GSD.
- `.planning/ROADMAP.md` - Phase ordering and requirements mapping.
- `.planning/REQUIREMENTS.md` - DEP-01, DEP-02, DEP-03 requirement definitions.
- `.planning/STATE.md` - Current planning state and recent decisions.

### Existing architecture and runtime
- `docs/architecture/deployment.md` - Current deployment assumptions and runtime environment documentation.
- `docs/backend/api-design.md` - Existing API surface and critical endpoints.
- `docs/security/token-authentication.md` - Existing token-based protection flow.
- `docs/frontend/architecture.md` - Frontend runtime contract that should not be disturbed.
- `docs/frontend/data-flow.md` - Artifact-first data flow and provider routing expectations.

### Code anchors
- `SlideTutor-AI/api/generate.ts` - Current critical-path generation endpoint and runtime coupling.
- `SlideTutor-AI/api/lib/structuredOutputConfig.ts` - Structured output contract layer that must remain stable.
- `SlideTutor-AI/src/config/models.ts` - Provider family assumptions that later BYOK work depends on.

</canonical_refs>

<specifics>
## Specific Ideas

- The migration should prefer “single-platform first-release user flow” over “partial split runtime with ongoing cross-platform glue.”
- If tradeoffs arise, protect future BYOK-first implementation velocity over preserving old Vercel convenience.
- Research should pay special attention to streaming behavior, request handling, and runtime differences that could tempt accidental changes to teaching logic.

</specifics>

<deferred>
## Deferred Ideas

- Login and identity systems.
- Billing, subscriptions, or credits.
- Hosted-model user product flows.
- Full parser-provider abstraction.
- China-mainland-specific deep infrastructure changes beyond what is needed to preserve first-release viability.

</deferred>

---

*Phase: 03-minimal-cloudflare-migration*
*Context gathered: 2026-04-04 via PRD Express Path*
