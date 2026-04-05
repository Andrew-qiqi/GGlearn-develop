---
phase: 05
slug: parser-bootstrap-and-provider-abstraction
status: ready_for_signoff
nyquist_compliant: true
wave_0_complete: true
created: 2026-04-05
updated: 2026-04-06
---

# Phase 05 Validation Strategy

> Updated after the remaining 05-03 Volcengine provider cutover work.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | `vitest` |
| **App config** | `SlideTutor-AI/vite.config.ts` |
| **Worker config** | `SlideTutor-AI/vitest.worker.config.ts` |
| **Quick run command** | `npm test -- api/lib/parser/volcengineProvider.test.ts api/parserAccess.test.ts api/security.test.ts` |
| **Worker run command** | `npm run test:workers -- test/workers/parse-route.worker.test.ts test/workers/security-observability.worker.test.ts` |
| **Full phase command** | `npm test -- api/lib/parser/volcengineProvider.test.ts api/parserAccess.test.ts api/security.test.ts && npm run test:workers -- test/workers/parse-route.worker.test.ts test/workers/security-observability.worker.test.ts && npm run lint` |

---

## Per-Task Verification Map

| Task ID | Plan | Requirement | Test Type | Automated Command | Status |
|---------|------|-------------|-----------|-------------------|--------|
| 05-03-01 | 03 | PARSE-01, PARSE-03 | provider/env unit | `npm test -- api/lib/parser/volcengineProvider.test.ts api/security.test.ts` | green |
| 05-03-02 | 03 | PARSE-01, PARSE-03, PARSE-04 | service/runtime | `npm test -- api/parserAccess.test.ts` | green |
| 05-03-03 | 03 | PARSE-01, PARSE-02, PARSE-04 | worker regression | `npm run test:workers -- test/workers/parse-route.worker.test.ts test/workers/security-observability.worker.test.ts` | green |
| 05-03-04 | 03 | PARSE-01, PARSE-02, PARSE-03, PARSE-04 | repo type safety | `npm run lint` | green |

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Settings still shows `Document Parsing` usage only inside AI Settings | PARSE-02 | placement and copy are UX-sensitive | Open Settings on the AI tab, confirm the usage row still reads `Document Parsing` and no provider name leaks into the UI. |
| A real successful analyze still yields normal teaching output with no visible provider details | PARSE-01, PARSE-03 | requires live parser credentials and browser flow | Run one analyze with valid Volcengine parser secrets, confirm the tutoring result looks normal and no provider copy is shown. |
| A real degraded analyze still shows `Low accuracy` and does not deduct on failure | PARSE-01, PARSE-04 | needs live quota / failure simulation | Disable parser secrets or simulate provider failure, run analyze, confirm `Low accuracy` appears and teaching still completes. |

---

## Validation Sign-Off

- [x] Volcengine provider request + normalization coverage exists
- [x] Live parser default is verified through service-level tests
- [x] Worker parse route regressions stay green
- [x] Old Azure runtime files are removed from the live code path
- [x] `npm run lint`

**Approval:** ready for signoff
