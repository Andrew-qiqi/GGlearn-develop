# BYOK Capability Check

Last updated: 2026-04-12

## Overview

This note records the current BYOK capability-check contract for `My API`, especially the `Custom OpenAI-compatible` path.

Key files:

- [models.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/src/config/models.ts)
- [apiClient.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/src/lib/api/apiClient.ts)
- [modelCapabilityProbe.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/api/lib/modelCapabilityProbe.ts)
- [generateService.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/api/lib/generateService.ts)

## Current frontend contract

The frontend persists capability state as:

- `pending`
- `checking`
- `usable`
- `unusable`
- `stale`

For `Custom OpenAI-compatible`, the selected catalog model remains the sentinel:

```ts
{
  providerId: 'openai-compatible',
  endpointPreset: 'custom',
  modelId: 'custom-openai-model'
}
```

The real runtime tuple is stored separately:

- `customApiKey`
- `customBaseURL`
- `customModelId`

## Probe route

Capability checks go through:

- `POST /api/model-capability-check`

The worker route forwards:

- `providerId`
- `modelId`
- `endpointPreset`
- normalized BYOK access payload

## Current custom OpenAI-compatible outcomes

For custom OpenAI-compatible models, probe results now split into three broad classes:

### `usable`

Returned when the custom probe succeeds and SlideTutor can treat the current endpoint/model as eligible for the current product contract.

### `unusable`

Returned for clear non-retryable failures:

- `MODEL_CAPABILITY_CHECK_AUTH_FAILED`
  - current API key was rejected
- `MODEL_CAPABILITY_CHECK_UNSUPPORTED`
  - current endpoint/model failed the structured runtime contract, but the failure could not be confidently attributed to streaming or structured-output support
- `MODEL_CAPABILITY_CHECK_STREAMING_UNSUPPORTED`
  - current endpoint/model appears unable to satisfy the streaming part of the structured runtime contract
- `MODEL_CAPABILITY_CHECK_STRUCTURED_OUTPUT_UNSUPPORTED`
  - current endpoint/model appears unable to satisfy the structured JSON-output part of the structured runtime contract
- `MODEL_CAPABILITY_CHECK_MULTIMODAL_RUNTIME_CONTRACT_UNSUPPORTED`
  - current endpoint/model failed the final `image + stream + structured-output` SlideTutor runtime contract probe

Legacy note:

- older cached frontend state may still contain `MODEL_CAPABILITY_CHECK_VISION_UNSUPPORTED`
- the UI should treat that legacy code as equivalent to the new multimodal-contract failure instead of as literal proof that raw vision capability is missing

For unknown or unverified custom runtime model ids, `capabilitySummary` is now allowed to be `null` on failure.
This avoids leaking the backend registry fallback (`all false`) as if it were real probe evidence for the current custom endpoint/model.

### `pending`

Still used for low-confidence or retryable failures:

- `MODEL_CAPABILITY_CHECK_FAILED`
  - transient network/provider failure
  - incomplete probe evidence

When the probe does not have trustworthy per-capability evidence, `capabilitySummary` should remain `null` instead of pretending the current custom model failed every hard constraint individually.

## Runtime admission

For custom OpenAI-compatible BYOK generation:

- a successful custom capability check can now produce a usable saved result
- the frontend can pass the saved capability summary back into runtime access payloads
- backend generate preflight may accept custom BYOK requests when a usable capability summary is supplied

This is different from the older behavior where custom BYOK remained effectively stuck behind `unverified` / `pending`.

## Current custom probe shape

For `Custom OpenAI-compatible`, the backend now probes progressively with two high-value runtime-contract checks:

1. streaming structured-output completion
2. streaming structured-output completion with image input

This preserves the real SlideTutor runtime contract while removing lower-value standalone text / streaming checks.

Current observability now also logs stage-level probe events with request correlation and upstream failure summaries so route-level `200` logs no longer hide which internal stage failed.

The current probe image fixture also uses a stable embedded `data:image/jpeg;base64,...` input so the capability check matches the runtime image shape more closely than the earlier PNG placeholder.
The fixture now keeps explicit size headroom above small-provider minimums instead of targeting the smallest possible passing dimensions.

For providers such as DashScope that enforce JSON-mode prompt rules, the probe prompt now explicitly includes the word `JSON` so the structured runtime contract check does not fail on prompt-shape grounds alone.

## UI expectations

The settings panel should now behave like this:

- `checking`: show explicit loading
- `usable`: show ready state
- `unusable`: show specific auth/unsupported messaging
- `pending` with `MODEL_CAPABILITY_CHECK_FAILED`: show retryable failure messaging
- expose a manual `Retry Check` action after non-checking states

## Related boundaries

- `Platform API` still must not allow `custom` OpenAI-compatible runtime execution
- built-in `Qwen` / `Doubao` capability truth still comes from the shared capability registry
