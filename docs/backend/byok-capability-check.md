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
  - current endpoint/model failed even the baseline text-generation compatibility probe
- `MODEL_CAPABILITY_CHECK_STREAMING_UNSUPPORTED`
  - current endpoint/model cannot stream chat-completion output the way SlideTutor requires
- `MODEL_CAPABILITY_CHECK_STRUCTURED_OUTPUT_UNSUPPORTED`
  - current endpoint/model cannot produce the structured JSON output SlideTutor requires
- `MODEL_CAPABILITY_CHECK_VISION_UNSUPPORTED`
  - current endpoint/model rejects image input under the SlideTutor runtime contract

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

For `Custom OpenAI-compatible`, the backend now probes progressively instead of with one opaque mixed request:

1. text completion
2. streaming text completion
3. streaming structured-output completion
4. streaming structured-output completion with image input

This preserves the real SlideTutor runtime contract while making unsupported failures more diagnosable.

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
