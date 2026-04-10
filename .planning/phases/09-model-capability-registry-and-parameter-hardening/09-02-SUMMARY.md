# 09-02 Summary

## Outcome

- Added one backend-owned BYOK capability probe path instead of trusting manually entered model metadata.
- Persisted capability-check status alongside model/access settings without making the frontend the source of truth.
- Added save-time and stale first-use capability checks so normal generation avoids repeated live-provider probing.

## Key Changes

- `SlideTutor-AI/api/lib/modelCapabilityProbe.ts`
  - introduced a normalized probe contract with `usable` / `unusable` status and machine-readable capability facts
  - keeps backend ownership of structured-output, streaming, vision, and thinking eligibility checks
- `SlideTutor-AI/src/worker/routes/model-capability-check.ts`
  - added a dedicated `POST /api/model-capability-check` route for settings-time and first-use verification
- `SlideTutor-AI/src/config/models.ts`
  - added persisted `modelCapabilityCheck` metadata shape including `status`, `checkedAt`, `lastErrorCode`, `capabilitySummary`, and `selection`
- `SlideTutor-AI/src/store/uiStore.ts`
  - persists capability-check state separately from raw BYOK credentials
  - marks saved readiness `stale` when model selection or BYOK access changes
- `SlideTutor-AI/src/lib/api/apiClient.ts`
  - added `checkModelCapability(...)`
  - runs one stale first-use check before BYOK generation requests
- `SlideTutor-AI/src/components/SettingsModal.tsx`
  - surfaces `Checking model compatibility`, `Model is ready`, and unusable states in the BYOK settings flow

## Verification

- `npm test -- api/lib/modelCapabilityProbe.test.ts src/store/uiStore.test.ts src/lib/api/apiClient.test.ts src/components/SettingsModal.test.tsx`

## Notes

- This summary covers Phase 09 Plan 02 only. Runtime parameter hardening and `distill` truncation handling are captured in `09-03`.
