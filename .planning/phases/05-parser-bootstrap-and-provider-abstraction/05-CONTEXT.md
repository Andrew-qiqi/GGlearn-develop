# Phase 05: Parser Bootstrap and Provider Abstraction - Context

**Gathered:** 2026-04-05
**Status:** Ready for planning
**Source:** PRD Express Path (`docs/discuss/phases/05-parser-bootstrap-and-provider-abstraction-brief.md`)

<domain>
## Phase Boundary

This phase makes document parsing an explicit, controlled product capability instead of a hidden Azure-backed default. Early users should still be able to use platform-managed `Document Parsing` with very low setup friction, but parser cost and usage must now be observable, limited, and intentionally degraded when the free platform path is unavailable.

The phase should introduce only the minimum backend and UI changes needed to support platform-funded parsing with daily limits, provider abstraction, and graceful fallback. It must not turn into parser commercialization, parser BYOK, login-based quota sync, or a large anti-abuse system.

</domain>

<decisions>
## Implementation Decisions

### Product posture
- Early public BYOK remains fully free.
- The platform continues to provide `Document Parsing` by default during the early public stage.
- Users should not be forced to configure their own parser provider in this phase.

### Parser limit policy
- Platform-funded parsing is limited to `10` successful page-level parsing operations per natural day.
- Quota deduction is based on a real successful platform parser call, not on button clicks or failed attempts.
- When the parser limit is exhausted, AI analysis must continue, but it should run without document parsing and therefore with lower precision.

### User experience
- Quota visibility should live in the Settings entry, not in the main analysis flow.
- The settings display should show an exact numeric format such as `7/10`.
- Only analyses that are actually downgraded due to parser unavailability should show a lightweight warning.
- The warning text should be `Low accuracy`.
- The hover detail should read: `Document parsing is unavailable for this analysis, so precision may be lower.`
- Product-facing copy should say `Document Parsing` and should not expose `Azure`.

### Service truth and storage
- Quota truth must be enforced server-side, not in local storage.
- The minimal implementation should use Cloudflare `D1` as the source of truth for anonymous parser quota tracking.
- The anonymous identity key for this phase should be `ip_hash + date_key`.
- IP hashing must use a dedicated `USAGE_HASH_SECRET`, not `API_TOKEN_SECRET` or another existing secret.
- Because current user volume is very small, the first version should avoid complex anti-abuse, multi-device reconciliation, or large-scale fallback design.

### Architecture boundary
- This phase must introduce a clean parser abstraction boundary so Azure is no longer the only implicit implementation path.
- Azure may remain the internal first provider implementation for now, but it must sit behind that abstraction.
- Mature teaching business logic must not be changed in this phase.

### the agent's Discretion
- The exact D1 schema, helper names, and service layering may be chosen during planning if they preserve the locked product behavior above.
- The exact placement and styling of the settings quota row may be chosen during planning if it stays lightweight and non-anxious.
- The parser abstraction can be implemented with the smallest viable interface as long as later provider replacement does not require re-cutting the main business path.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Product and phase scope
- `docs/discuss/project-brief.md` - Project-level direction, sequencing, and commercialization boundaries.
- `docs/discuss/phases/04-byok-first-access-layer-brief.md` - Upstream BYOK-first decisions that must remain intact.
- `docs/discuss/phases/05-parser-bootstrap-and-provider-abstraction-brief.md` - Locked phase-specific decisions and UX wording.

### GSD planning state
- `.planning/PROJECT.md` - Active project definition for GSD.
- `.planning/ROADMAP.md` - Phase ordering and requirement mapping.
- `.planning/REQUIREMENTS.md` - `PARSE-01`, `PARSE-02`, `PARSE-03`, and `PARSE-04`.
- `.planning/STATE.md` - Current planning history and recent product direction.

### Existing architecture and UX
- `docs/backend/api-design.md` - Existing API surface and current parsing/generation flow.
- `docs/frontend/architecture.md` - Frontend structure that the settings integration must respect.
- `docs/frontend/data-flow.md` - Data flow expectations around parsing and AI analysis.
- `docs/architecture/deployment.md` - Cloudflare runtime context and deployment base.

### Code anchors
- `SlideTutor-AI/api/lib/azureParse.ts` - Current parser implementation that should move behind an abstraction boundary.
- `SlideTutor-AI/api/lib/env.ts` - Existing env access patterns that will likely need new secrets and capability checks.
- `SlideTutor-AI/src/components/SettingsModal.tsx` - Current settings surface where parser quota should be displayed.
- `SlideTutor-AI/src/store/tutorStore.ts` - Current state flow that may need downgraded-analysis metadata.
- `SlideTutor-AI/wrangler.jsonc` - Cloudflare runtime config and future D1 binding surface.

</canonical_refs>

<specifics>
## Specific Ideas

- Prefer one minimal parser access layer plus one minimal quota service rather than scattering quota checks through multiple handlers.
- Protect the user reading experience: quota is visible when the user looks for it, but not pushed aggressively during normal study flow.
- Make sure the downgraded-analysis path is easy to perceive but not framed like an error state.
- Keep the first D1-backed quota system intentionally small and evolvable into future login-based identity.

</specifics>

<deferred>
## Deferred Ideas

- Parser BYOK.
- Multiple parser providers with a formal provider picker UI.
- Login-based quota sync across devices.
- Paid parser packs, subscriptions, or commercialization rules.
- Complex anti-abuse logic beyond the very small current-user scenario.

</deferred>

---

*Phase: 05-parser-bootstrap-and-provider-abstraction*
*Context gathered: 2026-04-05 via PRD Express Path*
