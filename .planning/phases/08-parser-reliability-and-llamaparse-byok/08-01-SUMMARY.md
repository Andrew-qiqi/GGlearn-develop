# 08-01 Summary

## Outcome

- Removed the old user-visible parser quota semantics from the live parser path.
- Stopped `My API` explain requests from borrowing the platform parser before parser BYOK is configured.
- Replaced misleading generic `429` parser/quota copy with code-driven user messaging.

## Key Changes

- `SlideTutor-AI/api/lib/parser/usageStore.ts`
  - kept usage tracking only as internal counting
  - removed the daily user allowance contract (`remaining`, `limit`)
- `SlideTutor-AI/api/lib/parser/accessService.ts`
  - removed parser quota gating from platform parsing
  - made usage tracking optional instead of parser-critical
  - simplified direct parse responses to parser success vs parser unavailable
- `SlideTutor-AI/api/lib/generateService.ts`
  - made parser preflight access-mode-aware
  - `Platform API` still uses the platform parser
  - `My API` without parser config now stays on the no-parser degraded path
- `SlideTutor-AI/src/components/SettingsModal.tsx`
  - removed parser-usage quota UI
- `SlideTutor-AI/src/lib/api/apiClient.ts`
  - removed the public parser-usage client helper
  - added centralized code-driven error message mapping
- `SlideTutor-AI/src/hooks/useSlideAnalysis.ts`
- `SlideTutor-AI/src/hooks/useFollowUp.ts`
  - switched away from hard-coded `15 RPM` / quota copy

## Verification

- `npm test -- api/parserAccess.test.ts api/parserUsage.test.ts api/lib/generateService.platform.test.ts src/lib/api/apiClient.test.ts src/components/SettingsModal.test.tsx src/hooks/useSlideAnalysis.test.ts src/hooks/useFollowUp.test.ts`
- `npm run lint`

## Notes

- The platform parser usage route still exists for operator/debug use, but it is no longer part of the user-facing product contract.
