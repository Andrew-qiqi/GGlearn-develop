# 08-02 Summary

## Outcome

- Added optional parser BYOK settings for `My API`, starting with `LlamaParse`.
- Implemented a dedicated `LlamaParse` adapter with upload, bounded polling, timeout handling, and `LayoutBlock[]` normalization.
- Wired `My API` explain requests to use `LlamaParse` only when parser config is present, while preserving the degraded no-parser fallback when it is absent.

## Key Changes

- `SlideTutor-AI/src/config/models.ts`
  - added persisted parser settings:
    - `parser.providerId: 'none' | 'llamaparse'`
    - `parser.apiKey`
- `SlideTutor-AI/src/components/SettingsModal.tsx`
  - added the optional parser block for `My API`
  - exposed `LlamaParse` as the first parser provider
- `SlideTutor-AI/src/lib/api/apiClient.ts`
  - now includes parser BYOK in `access` only when configured
- `SlideTutor-AI/api/lib/env.ts`
  - extended BYOK access payload typing with `parser`
- `SlideTutor-AI/api/lib/parser/provider.ts`
  - extended provider input so parser BYOK config can flow into adapters
- `SlideTutor-AI/api/lib/parser/llamaparseProvider.ts`
  - implemented official cloud upload + polling flow
  - added timeout and failure error classes
  - normalized structured outputs into the existing explain-chain block shape
- `SlideTutor-AI/api/lib/generateService.ts`
  - calls `LlamaParse` only for BYOK requests with parser config
  - emits `BYOK_PARSER_FAILED` and `BYOK_PARSER_TIMEOUT`

## Verification

- `npm test -- src/store/uiStore.test.ts src/components/SettingsModal.test.tsx src/lib/api/apiClient.test.ts api/lib/parser/llamaparseProvider.test.ts api/lib/generateService.platform.test.ts`
- `npm run lint`

## Notes

- `My API` without parser config intentionally does not fail and does not fall back to the platform parser.
