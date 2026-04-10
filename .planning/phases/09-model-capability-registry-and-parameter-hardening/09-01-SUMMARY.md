# 09-01 Summary

## Outcome

- Added one backend-owned capability registry for the active SlideTutor task baseline.
- Added capability-aware preflight in `generateService` so unknown, unverified, and globally ineligible models are rejected before provider execution.
- Kept structured-output task definitions aligned with the centralized active-task truth instead of leaving that contract implicit in runtime config only.

## Key Changes

- `SlideTutor-AI/api/lib/modelCapabilities.ts`
  - introduced `ACTIVE_MODEL_TASKS`, global hard/soft constraints, and model capability profiles
  - added registry coverage for current built-in Gemini and OpenAI-compatible models
  - established `unknown` / `unverified` / eligible vs ineligible capability states
- `SlideTutor-AI/api/lib/modelCapabilities.test.ts`
  - locked the exact active task set
  - verified `thinking` stays soft while structured output, streaming, image input, and text generation remain hard constraints
  - covered hard-constraint failure behavior
- `SlideTutor-AI/api/lib/generateService.ts`
  - added capability preflight before provider access resolution and stream creation
  - returns stable `MODEL_CAPABILITY_UNKNOWN`, `MODEL_CAPABILITY_UNVERIFIED`, and `MODEL_NOT_ELIGIBLE` errors
- `SlideTutor-AI/api/lib/generateService.platform.test.ts`
  - added preflight coverage for unknown, unverified, ineligible, and eligible model paths
- `SlideTutor-AI/api/lib/structuredOutputConfig.ts`
  - tied structured-task detection back to the centralized active-task baseline

## Verification

- `npm test -- api/lib/modelCapabilities.test.ts api/lib/generateService.platform.test.ts api/lib/geminiGenerationConfig.test.ts api/lib/openaiCompatibleGenerationConfig.test.ts`
- `npm run lint`

## Notes

- This summary covers Phase 09 Plan 01 only. BYOK save-time capability probing and capability-aware provider parameter generation remain for `09-02` and `09-03`.
