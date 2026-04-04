---
phase: 03-minimal-cloudflare-migration
plan: 02
subsystem: worker-api
tags: [cloudflare, workers, generate, parse, token-auth, streaming]
requires:
  - phase: 03-minimal-cloudflare-migration
    plan: 01
    provides: Worker shell, route-scoped env helpers, and Worker test infrastructure
provides:
  - Worker-native `/api/get-token`, `/api/parse`, and `/api/generate` critical-path routing
  - Shared route-scoped Azure parse and generation service helpers reusable across Express and Worker
  - Worker text-stream adapter plus DEP-03 coverage for token auth, origin checks, rate limiting, and request logs
affects: [deployment, api, worker, testing]
tech-stack:
  added: []
  patterns: [shared-service-extraction, worker-stream-response, route-scoped-capability-validation]
key-files:
  created:
    - SlideTutor-AI/api/lib/azureParse.ts
    - SlideTutor-AI/api/lib/generateService.ts
    - SlideTutor-AI/src/worker/lib/streams.ts
    - SlideTutor-AI/src/worker/routes/generate.ts
    - SlideTutor-AI/test/workers/generate-stream.worker.test.ts
  modified:
    - SlideTutor-AI/api/generate.ts
    - SlideTutor-AI/src/worker/index.ts
    - SlideTutor-AI/src/worker/routes/parse.ts
    - SlideTutor-AI/test/workers/security-observability.worker.test.ts
key-decisions:
  - "Kept teaching prompt construction, artifact contracts, and frontend response consumption unchanged by extracting runtime-neutral services instead of rewriting business logic."
  - "Used a Worker-native `ReadableStream` wrapper around the shared async iterable generation path so frontend hooks still receive ordered plain-text chunks."
  - "Applied origin checks, optional token auth, rate limiting, and structured request logs directly inside Worker routes rather than relying on Express middleware."
patterns-established:
  - "Pattern 1: critical-path provider logic lives in shared helpers under `api/lib/*`, while runtime shells stay thin."
  - "Pattern 2: Worker routes convert async iterable output to `Response` streams without changing frontend contracts."
requirements-completed: [DEP-02, DEP-03]
duration: 1h 20m
completed: 2026-04-04
---

# Phase 03: Plan 02 Summary

**Worker-native critical-path APIs with shared generation services, preserved text streaming, and DEP-03 protections**

## Performance

- **Duration:** 1h 20m
- **Completed:** 2026-04-04
- **Tasks:** 2
- **Files modified:** 10

## Accomplishments

- Restored and completed the shared backend extraction by adding `SlideTutor-AI/api/lib/azureParse.ts` and `SlideTutor-AI/api/lib/generateService.ts`.
- Kept `SlideTutor-AI/api/generate.ts` as a thin Express shell around shared services, so the old runtime path still builds while the Worker path reuses the same generation logic.
- Implemented Worker-native `/api/generate` in `SlideTutor-AI/src/worker/routes/generate.ts`, including optional token auth, origin protection, IP-aware rate limiting, structured request logs, and plain-text streaming.
- Switched Worker `/api/parse` to reuse the extracted Azure helper instead of importing from the Express route module.
- Added `SlideTutor-AI/src/worker/lib/streams.ts` so async provider output can be exposed as Worker `ReadableStream` responses without changing frontend hook behavior.
- Added Worker coverage for `/api/generate` success streaming, token-auth rejection, unauthorized origin rejection, rate-limit rejection, and request-log emission.

## Verification

- `npm run lint`
- `npm test -- api/security.test.ts src/hooks/useSlideAnalysis.test.ts src/hooks/useChunkRegenerate.test.ts`
- `npm run test:workers`
- `npm run build:cf`

## Files Created/Modified

- `SlideTutor-AI/api/lib/azureParse.ts` - extracted Azure Document Intelligence layout analysis with route-scoped env reads.
- `SlideTutor-AI/api/lib/generateService.ts` - extracted shared generation pipeline, moderation gate, provider dispatch, and text-stream creation.
- `SlideTutor-AI/src/worker/lib/streams.ts` - converts async iterables into Worker-native text responses.
- `SlideTutor-AI/src/worker/routes/generate.ts` - Worker `/api/generate` with DEP-03 protections and structured logging.
- `SlideTutor-AI/src/worker/routes/parse.ts` - now calls the shared Azure parser helper directly.
- `SlideTutor-AI/src/worker/index.ts` - routes `/api/generate` alongside `/api/get-token` and `/api/parse`.
- `SlideTutor-AI/test/workers/generate-stream.worker.test.ts` - verifies ordered text streaming and token-auth rejection.
- `SlideTutor-AI/test/workers/security-observability.worker.test.ts` - expanded with generate-route origin, rate-limit, and log assertions.

## Decisions Made

- Shared service extraction was chosen over duplicating provider logic inside the Worker route, reducing migration risk and protecting mature tutoring behavior.
- The Worker route logs final success on stream completion and logs failures on early rejection or stream errors so Cloudflare-side observability stays structured.

## Deviations from Plan

### Auto-fixed Issues

**1. Recovered a broken intermediate refactor state before continuing**

- **Found during:** initial test execution
- **Issue:** `api/generate.ts` had already been rewritten to import `azureParse` and `generateService`, but those files were missing, which broke Worker imports.
- **Fix:** recreated the missing shared helper modules and updated Worker `/api/parse` to import the extracted Azure helper directly.
- **Files modified:** `SlideTutor-AI/api/lib/azureParse.ts`, `SlideTutor-AI/api/lib/generateService.ts`, `SlideTutor-AI/src/worker/routes/parse.ts`
- **Verification:** targeted Worker tests went red first, then passed after the extraction was restored.

---

**Total deviations:** 1 auto-fixed
**Impact on plan:** Restored planned architecture and allowed the rest of Plan 02 to complete without changing scope.

## Issues Encountered

- The in-progress refactor had left missing shared modules behind, which caused Worker tests to fail on import before route behavior could be exercised.
- Worker log assertions needed to read the latest matching log entry because multiple generate-route tests can emit structured logs within the same suite.

## User Setup Required

None in this step. Actual Cloudflare secret and binding provisioning can wait until deployment/cutover work.

## Next Phase Readiness

- Plan 03 can now treat `/api/get-token`, `/api/parse`, and `/api/generate` as Worker-native critical-path routes.
- Feedback delivery and Cloudflare-compatible malicious-alert notification transport still remain for the next cutover-focused work.

---
*Phase: 03-minimal-cloudflare-migration*
*Completed: 2026-04-04*
