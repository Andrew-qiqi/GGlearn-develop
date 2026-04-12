# BYOK Custom Capability Probe Phase A/C Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add stage-level observability to custom BYOK capability checks and correct the final-stage error semantics so the product no longer claims unsupported vision when only the multimodal structured runtime contract has been shown to fail.

**Architecture:** Keep the current probe mechanics intact for this first pass. Instrument the existing custom probe stages with lightweight structured logs and replace the misleading final-stage error code and UI copy without yet deleting probe stages or changing image input shape.

**Tech Stack:** TypeScript, Cloudflare Worker, OpenAI SDK, React, Vitest, existing observability helpers

---

## File Map

- Modify: `SlideTutor-AI/api/lib/modelCapabilityProbe.ts`
  - add stage-aware logging hooks and rename the final-stage failure code
- Modify: `SlideTutor-AI/api/lib/modelCapabilityProbe.test.ts`
  - lock the new final-stage error code and stage-log behavior
- Modify: `SlideTutor-AI/src/components/SettingsModal.tsx`
  - map the new final-stage error code to corrected UI copy
- Modify: `SlideTutor-AI/src/components/SettingsModal.test.tsx`
  - cover the updated incompatibility message
- Modify: `SlideTutor-AI/src/lib/i18n/index.ts`
  - replace the misleading “image input unsupported” wording with multimodal-contract wording
- Modify: `docs/backend/byok-capability-check.md`
  - document the corrected semantics and stage logging
- Modify: `docs/changelog/CHANGELOG_TECH.md`
  - record the Phase A/C rollout

## Task 1: Lock the new semantics in tests

- [ ] Add failing tests in `SlideTutor-AI/api/lib/modelCapabilityProbe.test.ts` for:
  - final-stage failure returning `MODEL_CAPABILITY_CHECK_MULTIMODAL_RUNTIME_CONTRACT_UNSUPPORTED`
  - custom probe stage logs including stage name and upstream status/error snippet
- [ ] Add a failing UI test in `SlideTutor-AI/src/components/SettingsModal.test.tsx` for the corrected multimodal-contract message
- [ ] Run the targeted tests and verify they fail for the expected reasons

## Task 2: Instrument custom probe stages

- [ ] Add a small stage-log helper in `SlideTutor-AI/api/lib/modelCapabilityProbe.ts`
- [ ] Emit logs for each custom stage with:
  - `probeStage`
  - `modelId`
  - `baseURLHost`
  - `includeImage`
  - `responseFormatEnabled`
  - `upstreamStatus`
  - `upstreamErrorSnippet`
  - `stageDurationMs`
- [ ] Keep the current admission decisions unchanged except for the renamed final-stage error code

## Task 3: Correct the product semantics

- [ ] Replace `MODEL_CAPABILITY_CHECK_VISION_UNSUPPORTED` in the custom final stage with `MODEL_CAPABILITY_CHECK_MULTIMODAL_RUNTIME_CONTRACT_UNSUPPORTED`
- [ ] Update Settings copy so the UI says the endpoint cannot satisfy SlideTutor’s multimodal structured runtime contract
- [ ] Keep the existing `STRUCTURED_OUTPUT_UNSUPPORTED` and `STREAMING_UNSUPPORTED` semantics unchanged

## Task 4: Verify and document

- [ ] Run:
  - `cd SlideTutor-AI && npm run lint`
  - `cd SlideTutor-AI && npm test -- api/lib/modelCapabilityProbe.test.ts src/components/SettingsModal.test.tsx`
  - `cd SlideTutor-AI && npm test -- api/lib/modelCapabilities.test.ts src/lib/api/apiClient.test.ts`
- [ ] Update backend capability-check docs
- [ ] Update technical changelog
