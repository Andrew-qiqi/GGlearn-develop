---
phase: 07
slug: china-user-operational-fit
status: ready_for_execution
nyquist_compliant: true
wave_0_complete: true
created: 2026-04-06
updated: 2026-04-06
---

# Phase 07 Validation Strategy

> Per-phase validation contract for China-user operational-fit hardening.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | `vitest` + `@cloudflare/vitest-pool-workers` |
| **App config** | `SlideTutor-AI/vite.config.ts` |
| **Worker config** | `SlideTutor-AI/vitest.worker.config.ts` |
| **Quick run command** | `npm test -- api/lib/generateService.platform.test.ts src/lib/api/apiClient.test.ts src/components/SettingsModal.test.tsx src/components/settings/PlatformApiSection.test.tsx` |
| **Worker run command** | `npm run test:workers -- test/workers/credits-balance.worker.test.ts test/workers/recharge.worker.test.ts test/workers/security-observability.worker.test.ts` |
| **Full phase command** | `npm test -- api/lib/generateService.platform.test.ts src/lib/api/apiClient.test.ts src/components/SettingsModal.test.tsx src/components/settings/PlatformApiSection.test.tsx && npm run test:workers -- test/workers/credits-balance.worker.test.ts test/workers/recharge.worker.test.ts test/workers/security-observability.worker.test.ts && npm run lint` |

---

## Per-Task Verification Map

| Task ID | Plan | Requirement | Test Type | Automated Command | Status |
|---------|------|-------------|-----------|-------------------|--------|
| 07-01-01 | 01 | CN-01, CN-03 | service + client unit | `npm test -- api/lib/generateService.platform.test.ts src/lib/api/apiClient.test.ts` | pending |
| 07-01-02 | 01 | CN-01, CN-03 | component | `npm test -- src/components/SettingsModal.test.tsx src/components/settings/PlatformApiSection.test.tsx` | pending |
| 07-02-01 | 02 | CN-01, CN-02 | worker route + observability | `npm run test:workers -- test/workers/credits-balance.worker.test.ts test/workers/recharge.worker.test.ts test/workers/security-observability.worker.test.ts` | pending |
| 07-02-02 | 02 | CN-02 | static docs contract | `rg -n "APP_URL|requestId|payment-webhook|Clerk|Volcengine|ZPAY" docs/architecture/deployment.md docs/backend/api-design.md docs/operations/china-operator-checklist.md` | pending |
| 07-03-01 | 03 | CN-01, CN-02, CN-03 | static docs contract | `rg -n "My API|Platform API|parser BYOK|MinerU|must-fix|defer|evidence" docs/operations/china-operational-fit-report.md docs/operations/README.md docs/README.md docs/user_guide/README.md` | pending |

---

## Wave 0 Requirements

- [x] Existing `vitest` and Worker test infrastructure already cover the code paths in scope.
- [ ] `SlideTutor-AI/src/components/settings/PlatformApiSection.test.tsx` should be added for the new low-noise access guidance.
- [ ] `SlideTutor-AI/test/workers/security-observability.worker.test.ts` should be extended to cover operational route logging, not just `/api/generate`.

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| China-based user can recognize a region/provider-blocked model failure without being auto-switched into another provider | CN-01, CN-03 | depends on real provider/region behavior and UX wording | From a China-based access path, trigger a known unsupported provider scenario, confirm the UI message is explicit and the app does not silently switch providers. |
| Clerk build key + runtime secret are both configured correctly in a deployed environment | CN-02 | requires deploy/build settings, not just local tests | Deploy with real envs, confirm sign-in works, balance loads, and the app does not fall back to `My API` because of missing public Clerk build config. |
| Real parser + recharge support chain is debuggable from logs | CN-02 | needs deployed logs and real callbacks | Run one parse, one balance fetch, one recharge intent, and one real or replayed ZPAY callback; confirm request IDs and route logs are usable for support. |

---

## Validation Sign-Off

- [x] Automated tests exist for service, client, component, and Worker layers in scope
- [x] Worker-route observability is treated as first-class validation, not a docs-only promise
- [x] Docs-only deliverables have explicit grep-verifiable contracts
- [x] No new framework or watch-mode dependency is required
- [x] `nyquist_compliant: true` set in frontmatter

**Approval:** ready for execution
