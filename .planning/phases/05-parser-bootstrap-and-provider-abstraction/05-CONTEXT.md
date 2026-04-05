# Phase 05: Parser Bootstrap and Provider Abstraction - Context

**Gathered:** 2026-04-06
**Status:** Ready for planning
**Source:** Updated phase brief (`docs/discuss/phases/05-parser-bootstrap-and-provider-abstraction-brief.md`)

<domain>
## Phase Boundary

Phase 05 is no longer about introducing parser quota and degraded fallback from zero. That baseline already exists in the codebase. The remaining Phase 05 work is to finish the provider transition so the platform-managed parser runtime no longer depends on Azure as the live default implementation.

This phase should:

- keep platform-managed `Document Parsing` as the default early-user experience
- preserve the current D1-backed parser usage truth, `10/day` rule, and successful-only counting
- preserve the existing degraded-analysis behavior and Settings visibility already shipped
- replace the live platform parser provider from Azure to Volcengine
- remove or neutralize legacy Azure direct parser paths so the runtime has one parser truth

This phase must not expand into parser BYOK, parser provider selection UI, payment integration, or hosted-access hardening.

</domain>

<decisions>
## Implementation Decisions

### Current product posture
- Early public BYOK remains fully free.
- Platform-managed `Document Parsing` remains available by default.
- Parser quota, degraded fallback, and parser usage visibility are already part of the baseline.

### Locked provider direction
- The platform-managed parser provider for the live path must be `Volcengine`.
- Azure should no longer remain the implicit runtime default after this phase.
- Product-facing UI and copy should continue to say `Document Parsing`; users should not see provider names.

### Contract and compatibility
- Existing frontend-facing `LayoutBlock[]` expectations must remain stable in this phase.
- `/api/parse` and integrated explain parsing should continue to return the same effective block shape to downstream consumers.
- Successful parser calls still count against usage; failed or skipped calls do not.
- Existing degraded-analysis semantics, including `Low accuracy`, must keep working.

### Scope control
- Parser BYOK is not part of this phase.
- MinerU is a possible future BYOK-friendly parser candidate for China users, but it is explicitly deferred.
- ZPAY, hosted access, and other Phase 06 concerns are out of scope here.
- Mature teaching logic must not be reworked as part of the provider swap.

### the agent's Discretion
- The exact Volcengine-to-`LayoutBlock[]` normalization layer can be designed pragmatically if the output contract stays stable.
- The exact module split for provider normalization can be chosen if later provider additions do not require re-cutting the main chain.
- If a thin compatibility adapter is helpful during the migration, it is acceptable as long as the live platform path is clearly Volcengine-backed.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Product and phase scope
- `docs/discuss/project-brief.md` - Current project-level sequencing and locked decisions.
- `docs/discuss/phases/05-parser-bootstrap-and-provider-abstraction-brief.md` - Updated phase-specific scope for Volcengine replacement.
- `docs/discuss/phases/06-login-hosted-access-and-credit-brief.md` - Confirms Phase 06 work stays separate.

### GSD planning state
- `.planning/PROJECT.md` - Active project definition for GSD.
- `.planning/ROADMAP.md` - Shows 05-03 as the next plan.
- `.planning/REQUIREMENTS.md` - `PARSE-01` through `PARSE-04`.
- `.planning/STATE.md` - Current focus and resume point.

### Existing implementation anchors
- `SlideTutor-AI/api/lib/parser/accessService.ts` - Current shared parser access entry point.
- `SlideTutor-AI/api/lib/parser/azureProvider.ts` - Current provider implementation that should cease being the live default.
- `SlideTutor-AI/api/lib/azureParse.ts` - Legacy Azure-specific parsing implementation.
- `SlideTutor-AI/api/generate.ts` - Legacy direct parse path still using Azure.
- `SlideTutor-AI/src/worker/routes/parse.ts` - Current direct parse route integration.
- `SlideTutor-AI/api/lib/generateService.ts` - Integrated explain path already using parser access semantics.
- `SlideTutor-AI/wrangler.jsonc` - Current parser usage and credits migration bindings.

### Volcengine parser references
- `tmp_files/volcengine_document_parse_intellgence/2.md` - Pricing and product overview copied from official docs.
- `tmp_files/volcengine_document_parse_intellgence/3.md` - Request, response, and field details for `OCRPdf`.

### Existing docs likely to update
- `SlideTutor-AI/README.md`
- `docs/backend/api-design.md`
- `docs/frontend/architecture.md`
- `docs/frontend/data-flow.md`
- `docs/changelog/CHANGELOG_TECH.md`

</canonical_refs>

<specifics>
## Specific Ideas

- Prefer a direct provider swap inside the existing parser access boundary rather than a second parallel parser path.
- Map Volcengine `textblocks[].text`, `label`, and `norm_box` into the existing `LayoutBlock[]` contract.
- Because the current product analyzes slide pages, the first implementation should prefer a single-page image path if it satisfies the API constraints, instead of adding TOS upload complexity prematurely.
- Clean up `api/generate.ts` Azure direct usage so the codebase no longer has two platform parser truths.

</specifics>

<deferred>
## Deferred Ideas

- parser BYOK
- parser provider picker UI
- MinerU adapter
- TOS upload flow unless the implementation proves the image-base64 path is insufficient
- payment or hosted-access work

</deferred>

---

*Phase: 05-parser-bootstrap-and-provider-abstraction*
*Context gathered: 2026-04-06 from updated phase brief and current code state*
