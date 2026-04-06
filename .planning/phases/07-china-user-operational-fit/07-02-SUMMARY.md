---
phase: 07-china-user-operational-fit
plan: 02
subsystem: operations
tags: [observability, request-id, parser, credits, zpay, docs]
requires:
  - phase: 07-china-user-operational-fit
    provides: phase context, operational-fit scope, and route targets
provides:
  - Request-id parity across parser, credits, recharge, and payment routes
  - Structured Worker log parity for the operational support chain
  - China-operator smoke checklist covering Clerk, Volcengine, D1, and ZPAY
affects: [07-03, deployment, api, support]
key-files:
  created:
    - docs/operations/china-operator-checklist.md
  modified:
    - SlideTutor-AI/src/worker/lib/observability.ts
    - SlideTutor-AI/src/worker/routes/parse.ts
    - SlideTutor-AI/src/worker/routes/parser-usage.ts
    - SlideTutor-AI/src/worker/routes/credits-balance.ts
    - SlideTutor-AI/src/worker/routes/recharge-intent.ts
    - SlideTutor-AI/src/worker/routes/payment-webhook.ts
    - SlideTutor-AI/test/workers/security-observability.worker.test.ts
    - SlideTutor-AI/test/workers/credits-balance.worker.test.ts
    - SlideTutor-AI/test/workers/recharge.worker.test.ts
    - docs/architecture/deployment.md
    - docs/backend/api-design.md
key-decisions:
  - "Kept the request-log expansion low-noise by extending the existing Worker observer instead of adding a separate logging system."
  - "Added `requestId` only to JSON error responses so successful route contracts stay stable, especially the plain-text ZPAY acknowledgement."
patterns-established:
  - "Pattern 1: non-streaming operational routes use one shared observed-route helper."
  - "Pattern 2: operator docs point to request-id lookup rather than ad hoc console digging."
requirements-completed: [CN-01, CN-02]
completed: 2026-04-06
---

# Phase 07: Plan 02 Summary

**Request-id parity and operator-grade observability for the China-facing support chain**

## Accomplishments

- Extended Worker observability so `/api/parse`, `/api/parser-usage`, `/api/credits/balance`, `/api/recharge-intent`, and `/api/payment-webhook` now log `requestId`, `path`, `status`, `durationMs`, `method`, and low-sensitivity metadata.
- Added `requestId` to JSON error responses on those operational routes while preserving the plain-text `success` body required by valid ZPAY callbacks.
- Added a repeatable operator checklist that documents the Clerk build/runtime split, Volcengine parser setup, D1 bindings, ZPAY `APP_URL` coupling, live recharge verification, replay-safe callback verification, and request-id-based log lookup.
- Synced deployment and API docs so support flow is explicit instead of implied.

## Verification

- `npm run test:workers -- test/workers/credits-balance.worker.test.ts test/workers/recharge.worker.test.ts test/workers/security-observability.worker.test.ts`
- `rg -n "createRequestId|createRequestObserver|createObservedRoute|requestId" SlideTutor-AI/src/worker/routes/parse.ts SlideTutor-AI/src/worker/routes/parser-usage.ts SlideTutor-AI/src/worker/routes/credits-balance.ts SlideTutor-AI/src/worker/routes/recharge-intent.ts SlideTutor-AI/src/worker/routes/payment-webhook.ts`
- `rg -n "APP_URL|requestId|Clerk|Volcengine|ZPAY|payment-webhook" docs/operations/china-operator-checklist.md docs/architecture/deployment.md docs/backend/api-design.md`

## Notes

- Valid payment callbacks still do not expose `requestId` in the public response because ZPAY requires the exact plain-text `success` acknowledgement.
- The operator should use the webhook log entry plus callback timestamp when diagnosing successful callback paths.

---
*Phase: 07-china-user-operational-fit*
*Completed: 2026-04-06*
