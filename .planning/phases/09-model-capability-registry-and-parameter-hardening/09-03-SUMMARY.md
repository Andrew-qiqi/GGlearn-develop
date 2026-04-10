# 09-03 Summary

## Outcome

- Replaced task-only provider parameter branching with capability-aware generation config.
- Hardened `distill` so unsupported Gemini thinking controls no longer leak into incompatible models and structured-output truncation now surfaces as a stable backend failure.
- Slimmed `distill` input by removing packaging-only explanation lines while keeping Focus mode quality on the main explanation artifact unchanged.

## Key Changes

- `SlideTutor-AI/api/lib/structuredOutputConfig.ts`
  - generation config builders now accept resolved model capability profiles
  - Gemini only emits `thinkingConfig.thinkingLevel` when the chosen model actually supports thinking
  - `distill` output budget was raised to `4096` for Gemini and OpenAI-compatible structured-output calls
- `SlideTutor-AI/api/lib/generateService.ts`
  - passes resolved capability profiles into provider config builders
  - buffers Gemini `distill` responses, converts `MAX_TOKENS` into `STRUCTURED_OUTPUT_TRUNCATED`, and rejects invalid final structured JSON instead of silently continuing
- `SlideTutor-AI/api/lib/geminiStreamDiagnostics.ts`
  - added explicit helpers for `MAX_TOKENS` detection and final JSON completeness checks
- `SlideTutor-AI/src/lib/ai/artifacts.ts`
  - added `formatExplanationArtifactForDistill(...)` so `distill` receives teaching content without `Visual Focus Box` or `Socratic Probe` packaging
- `SlideTutor-AI/src/hooks/useSlideAnalysis.ts`
  - now sends the distill-safe explanation formatter into the `distill` request while keeping the full explanation artifact for main reading flows
- `SlideTutor-AI/src/lib/api/apiClient.ts`
  - runtime capability/configuration failures now mark the saved BYOK capability status `stale` for recheck on the next eligible request

## Verification

- `npm test -- api/lib/geminiGenerationConfig.test.ts api/lib/openaiCompatibleGenerationConfig.test.ts api/lib/generateService.platform.test.ts src/lib/ai/artifacts.test.ts src/hooks/useSlideAnalysis.test.ts`
- `npm test -- api/lib/modelCapabilityProbe.test.ts src/store/uiStore.test.ts src/lib/api/apiClient.test.ts src/components/SettingsModal.test.tsx`
- `npm run lint`

## Notes

- `STRUCTURED_OUTPUT_TRUNCATED` intentionally does not mark BYOK capability state stale by itself because it reflects output-budget exhaustion rather than a model-eligibility regression.
