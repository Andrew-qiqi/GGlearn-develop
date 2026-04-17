# BYOK Custom Capability Probe Design

**Date:** 2026-04-12

## Goal

Reduce false negatives in `Custom OpenAI-compatible` capability checks and stop overstating probe failures as missing single capabilities when the current implementation is actually testing a composite GGlearn runtime contract.

## Problem Summary

The current custom probe has three core issues:

1. It spends network time on low-value checks such as pure text generation, which adds latency without materially improving decision quality.
2. The final probe stage currently tests `image + stream + json_schema` together, but failures are surfaced as `MODEL_CAPABILITY_CHECK_VISION_UNSUPPORTED`, which overstates what is actually known.
3. Current Cloudflare / Worker observability only shows route-level success and total duration, not which probe stage failed or what the upstream provider returned.

This leads to a misleading user experience: a model can be known-good in the built-in registry, yet the custom path can still fail with a stage-4 error that reads like "vision unsupported" even when the failure may actually be caused by the multimodal structured runtime contract.

## Agreed Design Direction

### 1. Remove low-value probe stages

Delete these standalone custom probe stages:

- pure text generation
- pure streaming-only generation

Reason:

- both capabilities are now common enough that they do not justify an extra request by themselves
- both will still be implicitly exercised inside the higher-value structured and multimodal stages
- route wall time is already dominated by upstream latency, so every probe request matters

### 2. Keep only high-value staged probes

The custom probe should become a 2-stage contract check:

1. `structured runtime contract`
   - text
   - stream
   - `response_format = json_schema`
2. `multimodal runtime contract`
   - image
   - stream
   - `response_format = json_schema`

This keeps the probe aligned with the actual GGlearn runtime contract instead of drifting into synthetic toy checks.

### 3. Correct the failure semantics

The second stage must no longer report `VISION_UNSUPPORTED` by default.

Recommended replacement:

- `MODEL_CAPABILITY_CHECK_MULTIMODAL_RUNTIME_CONTRACT_UNSUPPORTED`

Meaning:

- the endpoint cannot satisfy the multimodal structured streaming contract GGlearn currently requires

It must **not** imply:

- the model lacks raw vision capability
- the endpoint cannot parse any image at all

Only a dedicated `image-only` probe would justify a true `VISION_UNSUPPORTED` conclusion. That probe is intentionally out of scope for this first correction pass.

### 4. Align probe image input with real runtime input

The probe should stop using a synthetic 1x1 PNG placeholder and instead use the same image shape the runtime normally sends:

- `data:image/jpeg;base64,...`

The objective is not to perfectly simulate a full PDF page, but to stop introducing an avoidable input-shape mismatch between probe traffic and real explain traffic.

### 5. Add minimum viable observability

Before or alongside the probe refactor, add stage-level probe logs that record:

- `appRequestId`
- `cfRayId` when available
- `probeStage`
- `modelId`
- `baseURLHost`
- `includeImage`
- `responseFormatEnabled`
- `upstreamStatus`
- `upstreamErrorSnippet`
- `stageDurationMs`

This is the minimum needed to distinguish:

- real incompatibility
- false negatives caused by input shape or provider quirks
- failures caused by `json_schema`
- failures caused by multimodal structured combinations

## Scope

### In scope

- custom BYOK capability probe stage design
- custom BYOK error-code semantics
- custom BYOK settings UI copy for probe failures
- stage-level Worker observability for capability checks

### Out of scope

- changing built-in model registry truth
- adding a full image-only capability matrix
- changing the runtime generation contract itself
- changing parser behavior

## Proposed Delivery Order

### Phase A: Observability

Add stage-level logging without changing admission decisions.

Success criterion:

- a failed custom capability check can be mapped to a specific probe stage and upstream error summary

### Phase B: Probe simplification

Remove:

- text-only probe
- stream-only probe

Keep:

- structured runtime contract
- multimodal runtime contract

Success criterion:

- fewer probe round trips
- no loss of meaningful contract coverage

### Phase C: Semantic correction

Replace current stage-2 failure naming and UI copy so the product no longer claims "vision unsupported" when only the multimodal structured contract has been shown to fail.

Success criterion:

- failure messages match what the system actually knows

### Phase D: Input-shape alignment

Use JPEG data URL probe input to better match the real runtime path.

Success criterion:

- reduced avoidable divergence between probe and explain input shape

## Validation Strategy

Validation should include:

- unit tests for probe stage routing
- unit tests for new error-code mapping
- settings UI tests for updated incompatibility copy
- a local or staged manual check confirming stage-level logs appear with probe failures

## Open Question

If provider evidence later shows that some endpoints support raw image input but reject `image + stream + json_schema`, we may want an optional future `image-only` diagnostic probe.

That is intentionally deferred until after observability is in place.

## Execution Clarifications

### Logging interface

The Worker route should pass request correlation metadata into the probe layer:

- `appRequestId`
- `cfRayId`

The probe layer should accept an optional logger callback for testability and default to structured `console.info` logging in runtime execution.

### Stage-1 error semantics after simplification

When Phase B lands and the probe is reduced to:

1. `structured runtime contract`
2. `multimodal runtime contract`

stage 1 should continue to preserve useful distinctions where possible:

- failures traceable to streaming should remain `MODEL_CAPABILITY_CHECK_STREAMING_UNSUPPORTED`
- failures traceable to structured JSON output should remain `MODEL_CAPABILITY_CHECK_STRUCTURED_OUTPUT_UNSUPPORTED`

The simplification removes low-value standalone requests, not the semantic distinction between those two failure modes.

### Legacy error-code compatibility

Stored frontend state may still contain `MODEL_CAPABILITY_CHECK_VISION_UNSUPPORTED`.

Until that cached state naturally ages out, the UI must map both:

- `MODEL_CAPABILITY_CHECK_VISION_UNSUPPORTED`
- `MODEL_CAPABILITY_CHECK_MULTIMODAL_RUNTIME_CONTRACT_UNSUPPORTED`

to the same corrected multimodal-contract wording.

### JPEG probe fixture expectations

When Phase D lands, the replacement fixture should be:

- a stable embedded JPEG data URL
- larger than a toy 1x1 placeholder, with explicit safety headroom above provider minimums
- current target: `32x32`
- deterministic and safe for test snapshots

The goal is not realism for its own sake, but reducing avoidable divergence from the runtime `image/jpeg` shape used by explain requests.
