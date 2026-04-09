# 08-03 Summary

## Outcome

- Hardened the remaining `Volcengine` platform parser path around upstream throttling vs general unavailability.
- Separated Worker route throttling, platform parser failures, and BYOK parser failures into explicit error classes.
- Synced the main backend/frontend parser docs to the final Phase 08 ownership model.

## Key Changes

- `SlideTutor-AI/api/lib/parser/volcengineProvider.ts`
  - added explicit `VolcengineParserError`
  - now normalizes upstream throttling into `PLATFORM_PARSER_RATE_LIMITED`
  - now normalizes general parser failures into `PLATFORM_PARSER_UNAVAILABLE`
- `SlideTutor-AI/api/lib/parser/accessService.ts`
  - propagates platform parser rate-limited vs unavailable outcomes separately
- `SlideTutor-AI/api/lib/generateService.ts`
  - hosted analyze now distinguishes `PLATFORM_PARSER_RATE_LIMITED` from `PLATFORM_PARSER_UNAVAILABLE`
- `SlideTutor-AI/src/worker/routes/generate.ts`
  - Worker route throttling now returns `ROUTE_RATE_LIMITED`
- `docs/backend/api-design.md`
- `docs/frontend/data-flow.md`
- `docs/backend/platform-model-configuration.md`
- `docs/changelog/CHANGELOG_TECH.md`
  - updated to reflect final parser ownership and error taxonomy

## Verification

- `npm test -- api/lib/parser/volcengineProvider.test.ts api/parserAccess.test.ts api/lib/generateService.platform.test.ts api/lib/parser/llamaparseProvider.test.ts src/lib/api/apiClient.test.ts src/components/SettingsModal.test.tsx src/store/uiStore.test.ts src/hooks/useSlideAnalysis.test.ts src/hooks/useFollowUp.test.ts`
- `npm run test:workers -- test/workers/parse-route.worker.test.ts test/workers/generate-stream.worker.test.ts test/workers/security-observability.worker.test.ts`
- `npm run lint`
- `rg -n "Volcengine|LlamaParse|Platform API|My API|degraded|ROUTE_RATE_LIMITED|PLATFORM_PARSER|BYOK_PARSER" docs/backend/api-design.md docs/frontend/data-flow.md docs/backend/platform-model-configuration.md docs/changelog/CHANGELOG_TECH.md`

## Notes

- `docs/frontend/data-flow.md` had a pre-existing local modification; updates were applied surgically rather than rewriting the file wholesale.
