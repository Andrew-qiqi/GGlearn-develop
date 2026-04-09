# Phase 08: Parser Reliability and LlamaParse BYOK - Context

**Gathered:** 2026-04-09
**Status:** Ready for planning
**Source:** PRD Express Path (`docs/discuss/phases/08-parser-reliability-and-llamaparse-byok-brief.md`)

<domain>
## Phase Boundary

Phase 08 is a parser-system cleanup and stabilization phase, not a broad infrastructure reset.

This phase should:

- harden the existing `Volcengine` parser path used by `Platform API`
- remove user-visible parser quota and parser trial semantics that now conflict with the product direction
- stop `My API` from implicitly borrowing the platform parser
- add a dedicated parser BYOK path for `My API`, with `LlamaParse` as the first provider
- preserve the existing no-parser degraded analysis path when a `My API` user does not configure a parser
- keep parser/provider boundaries simple so platform parser, BYOK parser, and degraded fallback do not bleed into each other

This phase must not become:

- a platform parser provider replacement away from `Volcengine`
- a multi-provider parser marketplace phase
- a self-hosted parser phase
- a model-configuration cleanup phase
- a full Worker routing or queueing rewrite

</domain>

<decisions>
## Implementation Decisions

### Locked parser product posture
- `Platform API` keeps a platform-managed parser path and remains locked to `Volcengine`.
- `Platform API` users should not configure their own parser.
- `My API` now supports parser BYOK through a dedicated parser configuration path.
- The first `My API` parser BYOK provider is `LlamaParse`.
- If a `My API` user has no parser configured, the existing no-parser degraded analysis path must continue to work.

### Locked parser quota and routing rules
- Remove user-visible parser daily quota semantics and parser trial semantics.
- Do not keep or reintroduce the old BYOK parser trial / shared-platform-parser behavior.
- Parser protection may remain as internal infrastructure safeguards only; it should not surface as product quota UX.
- Parser-related failures should be classified by source instead of being merged into one misleading generic `429` story.

### Locked code-structure direction
- Keep the current explain pipeline shape centered on normalized parser output consumed as `LayoutBlock[]` or a compatible minimal structure.
- Add `LlamaParse` through a modular provider/adapter path rather than hardwiring provider-specific logic into the old platform parser access flow.
- Keep parser routing boundaries explicit:
  - platform parser for `Platform API`
  - parser BYOK for `My API`
  - degraded no-parser fallback when no parser is configured

### Phase sequencing rules
- Parser stabilization and parser BYOK happen in this phase.
- Model configuration cleanup, including Gemini thinking-parameter normalization, stays in the next separate phase.

### the agent's Discretion
- The exact internal shape of parser error categories may be chosen pragmatically if the categories clearly separate platform parser unavailability, route-level limiting, and upstream provider limiting/failure.
- The exact `LlamaParse` result normalization may be chosen pragmatically if it preserves the current explain-chain contract and avoids forcing a whole-pipeline rewrite.
- The minimal frontend settings UX for parser BYOK may be chosen pragmatically if the boundary stays low-friction and explicit.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Product scope and locked decisions
- `docs/discuss/project-brief.md` - Project-level product posture and commercial boundaries; treat stale parser-BYOK notes as superseded by this phase brief and context.
- `docs/discuss/phases/08-parser-reliability-and-llamaparse-byok-brief.md` - Phase-specific objective, scope, and locked parser decisions.

### Planning truth
- `.planning/PROJECT.md` - Current project-level parser direction and phase sequencing.
- `.planning/ROADMAP.md` - Phase 08 goal, requirements, and success criteria.
- `.planning/REQUIREMENTS.md` - `PARSE-05`, `PARSE-06`, `PARSE-07`.
- `.planning/STATE.md` - Current state, decisions, and active concerns.

### Existing parser implementation anchors
- `SlideTutor-AI/api/lib/generateService.ts` - Current explain-chain parser preflight and BYOK/platform access interaction.
- `SlideTutor-AI/api/lib/parser/accessService.ts` - Current parser access policy layer.
- `SlideTutor-AI/api/lib/parser/usageStore.ts` - Existing parser quota truth and storage logic that now needs cleanup.
- `SlideTutor-AI/api/lib/parser/volcengineProvider.ts` - Current `Volcengine` parser provider contract.
- `SlideTutor-AI/src/worker/routes/generate.ts` - Shared route-level limiting and generation entrypoint.
- `SlideTutor-AI/src/hooks/useSlideAnalysis.ts` - Frontend analysis path and parser/rate-limit copy.
- `SlideTutor-AI/src/hooks/useFollowUp.ts` - Follow-up path and shared user-facing error handling.
- `SlideTutor-AI/src/components/SettingsModal.tsx` - Existing settings surface that may host parser BYOK configuration.
- `SlideTutor-AI/src/store/uiStore.ts` - Existing settings state.
- `SlideTutor-AI/src/config/models.ts` - Access-mode settings adjacency and current provider configuration shape.

### Architecture and route contracts
- `docs/backend/api-design.md` - Backend route contracts and parser-related API expectations.
- `docs/backend/platform-model-configuration.md` - Adjacent model-configuration context; relevant only to preserve separation from the next phase.

### Supporting references
- `tmp_files/volcengine_document_parse_intellgence/2.md` - Local Volcengine parser reference snapshot.
- `tmp_files/volcengine_document_parse_intellgence/3.md` - Local Volcengine parser reference snapshot.
- `https://developers.llamaindex.ai/python/cloud/llamaparse/api-v2-guide/` - `LlamaParse` API shape.
- `https://developers.llamaindex.ai/python/cloud/general/rate_limits/` - `LlamaParse` rate-limit reference.
- `https://developers.llamaindex.ai/python/cloud/general/pricing/` - `LlamaParse` pricing reference.

</canonical_refs>

<specifics>
## Specific Ideas

- The current user-facing `15 RPM` story is misleading; the system mixes product quota behavior, route-level limiting, and provider-side parser failures into one UX surface.
- The old parser D1 quota/trial baseline was useful as a guardrail during an earlier stage, but it now hurts the live experience and should not remain a product behavior.
- `My API` parser BYOK should be implemented now, not deferred, and the chosen first provider is `LlamaParse`, not `MinerU`.
- `LlamaParse` is acceptable on the `My API` path even if its latency model differs from `Volcengine`, because `My API` users explicitly opt into their own parser setup.
- The parser phase should improve maintainability, not add a large provider-selection framework.

</specifics>

<deferred>
## Deferred Ideas

- additional parser BYOK providers beyond `LlamaParse`
- `LiteParse`, `Docling`, or self-hosted parser infrastructure
- replacing the platform parser provider away from `Volcengine`
- model configuration stability work, including Gemini thinking-parameter cleanup

</deferred>

---

*Phase: 08-parser-reliability-and-llamaparse-byok*
*Context gathered: 2026-04-09 via PRD Express Path*
