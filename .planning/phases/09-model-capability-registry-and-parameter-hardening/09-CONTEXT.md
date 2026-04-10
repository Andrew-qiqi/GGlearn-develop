# Phase 09: Model Capability Registry and Parameter Hardening - Context

**Gathered:** 2026-04-10
**Status:** Ready for planning
**Source:** PRD Express Path (`docs/discuss/phases/09-model-capability-registry-and-parameter-hardening-brief.md`)

<domain>
## Phase Boundary

Phase 09 is a model-runtime stability and capability-governance phase, not a new provider-expansion phase.

This phase should:

- establish one backend-owned source of truth for model capability, hard constraints, and soft constraints
- stop deciding provider parameters directly from task branches alone
- make Gemini and OpenAI-compatible runtime parameters model-aware
- prevent unsupported provider settings such as invalid Gemini thinking controls from reaching live requests
- define a reliable admission path for BYOK models, including capability probe and usable/unusable state
- harden structured-output execution so `distill` and related tasks stop failing because of parameter-budget mistakes or incomplete JSON
- preserve `quickExplain` / Focus mode quality while fixing `distill` truncation

This phase must not become:

- a product-surface redesign of `My API` vs `Platform API`
- a dead-task cleanup phase
- a hosted pricing/credits expansion phase
- a new provider-family integration phase
- a prompt-style rewrite for teaching behavior

</domain>

<decisions>
## Implementation Decisions

### Locked active-task baseline
- The current capability baseline is the union of these active tasks only:
  - `explain`
  - `distill`
  - `followup`
  - `regenerate_chunk`
  - `regenerate_followup`
  - `generate_questions`
  - `evaluate_answers`
- `evaluate_note` is not part of the active capability baseline.
- Dead-task cleanup happens in the following phase, not here.

### Locked model-capability posture
- There is exactly one global model capability truth.
- Do not maintain separate model-admission standards for `My API` and `Platform API`.
- Model capability and product policy must stay separate:
  - model capability answers whether a model can technically satisfy the product's active-task baseline
  - product policy answers which access mode exposes which task
- Any model that fails any global hard constraint is directly considered unavailable.
- Do not allow “task A works, task B fails later” as the normal admission model.

### Locked hard vs soft constraint direction
- `native structured output` is a hard constraint for the current artifact-dependent product.
- streaming remains a hard constraint under the current frontend contract.
- `thinking` is a soft constraint only and must not remain a hard admission gate.
- Unsupported provider-specific runtime knobs must be handled by model-aware parameter generation, not by letting provider errors surface late.

### Locked BYOK posture
- Users must not manually fill a capability table.
- custom BYOK models must enter a system-managed state such as usable / unusable / pending verification.
- custom OpenAI-compatible models that do not support `native structured output` are directly unusable.
- There is no experimental compatibility mode for unsupported structured-output models.

### Locked probe and preflight direction
- Real provider capability probing should not happen on every normal generation request.
- Normal requests may do lightweight local preflight decisions against cached/known capability truth.
- Real provider probes should happen after saving configuration.
- A successful normal generation request can be treated as an implicit health check for the current saved model configuration.
- Do not introduce time-based forced rechecks in the first version.
- If normal generation returns a clear capability-mismatch, model-unreachable, or configuration-invalid signal, mark the model as needing recheck.

### Locked structured-output stability direction
- `maxOutputTokens` must not remain the primary abuse-control mechanism for critical structured tasks.
- Critical structured tasks must have enough output budget to return complete valid JSON.
- `distill` truncation is in scope for this phase and must be solved as part of model/parameter hardening.
- The preferred mitigation order for `distill` is:
  - raise output budget first
  - slim only packaging information from `distill` input second
  - improve failure attribution and diagnostics last
- If `distill` input is slimmed, remove packaging information first.
- Do not degrade `quickExplain` / Focus mode quality by crudely shrinking teaching content.
- Do not introduce task-specific automatic retry in the first version.

### the agent's Discretion
- The exact schema and storage location of the capability registry is open.
- The exact field names for hard/soft constraints, provider feature flags, and model status are open.
- The exact provider-error normalization table is open if it yields stable product-facing error semantics.
- The exact structured-output error taxonomy and logging detail are open, as long as they make parameter failures, capability failures, and output-truncation failures easier to distinguish.
- The exact invalidation rules after runtime failures are open, as long as they are based on stable error signatures and do not silently bypass hard constraints.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Product scope and locked decisions
- `docs/discuss/project-brief.md` - Current product posture and long-running boundaries.
- `docs/discuss/phases/09-model-capability-registry-and-parameter-hardening-brief.md` - Phase-specific objective, scope, and locked decisions.
- `docs/discuss/phases/10-dead-task-cleanup-and-hosted-task-surface-alignment-brief.md` - Follow-up cleanup phase boundary so Phase 09 does not absorb Phase 10 work.

### Planning truth
- `.planning/PROJECT.md` - Active project direction and current milestone focus.
- `.planning/ROADMAP.md` - Phase 09 / 10 goals, dependencies, and success criteria.
- `.planning/REQUIREMENTS.md` - `MODEL-01`, `MODEL-02`, `MODEL-03`, plus adjacent hosted/task-surface requirements.
- `.planning/STATE.md` - Current state, decisions, and blockers.

### Existing implementation anchors
- `SlideTutor-AI/src/config/models.ts` - Current frontend-visible model list and defaults.
- `SlideTutor-AI/api/lib/env.ts` - Provider/access resolution and current platform/BYOK parsing of model inputs.
- `SlideTutor-AI/api/lib/structuredOutputConfig.ts` - Current structured-task policy, schema definitions, Gemini thinking config, and output-token caps.
- `SlideTutor-AI/api/lib/generateService.ts` - Runtime request orchestration and provider execution entrypoint.
- `SlideTutor-AI/src/hooks/useSlideAnalysis.ts` - `explain -> distill` pipeline and current handling of invalid `distill` JSON.
- `SlideTutor-AI/src/hooks/useFollowUp.ts` - Follow-up and regenerate request assembly.
- `SlideTutor-AI/src/hooks/useChunkRegenerate.ts` - regenerate chunk flow on the frontend.
- `SlideTutor-AI/src/hooks/useQuiz.ts` - quiz generation/evaluation task usage.
- `SlideTutor-AI/src/lib/ai/artifacts.ts` - artifact parsing and current `distill` / explanation formatting used as task input.
- `SlideTutor-AI/src/lib/ai/prompts.ts` - task prompts and structured-output contracts.

### Existing docs
- `docs/backend/platform-model-configuration.md` - Existing developer guidance for model configuration; some content will be superseded by backend capability truth.
- `docs/backend/api-design.md` - Current API/task-surface documentation.
- `docs/frontend/data-flow.md` - Current structured-output flow and hosted-task descriptions.
- `docs/changelog/CHANGELOG_TECH.md` - Historical change log that will need synchronization after hardening lands.

### External provider references
- `https://ai.google.dev/gemini-api/docs/thinking` - Gemini thinking controls and concepts.
- `https://ai.google.dev/gemini-api/docs/structured-output` - Gemini structured-output contract.
- `https://platform.openai.com/docs/guides/structured-outputs` - OpenAI structured-output reference.

</canonical_refs>

<specifics>
## Specific Ideas

- The current bug originated because `distill` and `regenerate_chunk` hardcode Gemini `thinkingLevel` and output caps without model-aware branching.
- Current frontend model visibility and backend runtime capability are too loosely coupled; new models can appear selectable before runtime compatibility is truly known.
- `distill` currently receives a full explanation string that includes packaging data such as visual-focus boxes and Socratic probe text, even though `quickExplain` and context-memory generation may not need all of that formatting.
- `distill` failures are currently too easy to swallow or surface only as low-signal parse errors, which weakens confidence in structured-output reliability.
- This phase should leave Phase 10 with a cleaner boundary: capability truth here, dead-task cleanup and hosted task-surface alignment there.

</specifics>

<deferred>
## Deferred Ideas

- task-surface cleanup for `evaluate_note`
- hosted billing semantics for regenerate actions
- broader task taxonomy or naming cleanup
- provider quality/ranking heuristics beyond hard/soft capability modeling
- automatic frontend model visibility generation from the capability registry

</deferred>

---

*Phase: 09-model-capability-registry-and-parameter-hardening*
*Context gathered: 2026-04-10 via PRD Express Path*
