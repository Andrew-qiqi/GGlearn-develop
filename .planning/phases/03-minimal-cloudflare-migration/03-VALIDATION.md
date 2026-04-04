---
phase: 03
slug: minimal-cloudflare-migration
status: draft
nyquist_compliant: true
wave_0_complete: false
created: 2026-04-04
---

# Phase 03 Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | `vitest` |
| **Config file** | `SlideTutor-AI/vite.config.ts` |
| **Quick run command** | `npm run test:workers -- test/workers/security-observability.worker.test.ts` |
| **Full suite command** | `npm test && npm run test:workers` |
| **Estimated runtime** | ~120 seconds |

---

## Sampling Rate

- **After every task commit:** Run `npm run test:workers -- test/workers/security-observability.worker.test.ts`
- **After every plan wave:** Run `npm test && npm run test:workers`
- **Before `$gsd-verify-work`:** Full suite must be green
- **Max feedback latency:** 120 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 03-01-01 | 01 | 1 | DEP-01 | worker integration | `npm run test:workers -- test/workers/spa-routing.worker.test.ts` | No - Wave 0 | pending |
| 03-01-02 | 01 | 1 | DEP-03 | worker integration | `npm run test:workers -- test/workers/security-observability.worker.test.ts` | No - Wave 0 | pending |
| 03-02-01 | 02 | 2 | DEP-03 | unit/integration | `npm test -- api/security.test.ts` | Yes | pending |
| 03-02-02 | 02 | 2 | DEP-02, DEP-03 | worker integration | `npm run test:workers -- test/workers/generate-stream.worker.test.ts test/workers/security-observability.worker.test.ts` | No - Wave 0 | pending |
| 03-03-01 | 03 | 3 | DEP-01 | build/integration | `npm run build` | Yes | pending |
| 03-03-02 | 03 | 3 | DEP-01, DEP-03 | worker integration | `npm run test:workers -- test/workers/spa-routing.worker.test.ts test/workers/generate-stream.worker.test.ts test/workers/security-observability.worker.test.ts` | No - Wave 0 | pending |

*Status: pending / green / red / flaky*

---

## Wave 0 Requirements

- [ ] `SlideTutor-AI/test/workers/spa-routing.worker.test.ts` - Worker routing coverage for DEP-01
- [ ] `SlideTutor-AI/test/workers/generate-stream.worker.test.ts` - stream preservation coverage for DEP-02
- [ ] `SlideTutor-AI/test/workers/security-observability.worker.test.ts` - token/origin/rate-limit/logging coverage for DEP-03
- [ ] `npm install -D @cloudflare/vitest-pool-workers` - Worker-runtime test infrastructure

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Direct browser navigation to `/api/get-token`, `/api/parse`, `/api/generate`, and `/api/feedback` never falls through to the SPA shell | DEP-01 | SPA fallback behavior is easiest to confirm with a real browser navigation pass | Run the Worker locally, open each route directly in the browser, and verify each path returns API JSON/text output rather than `index.html`. |
| Cloudflare deploy configuration, secrets, and observability are complete for cutover | DEP-01, DEP-03 | Dashboard-managed bindings and logs are outside repo-only automation | Deploy to a preview Worker, confirm required secrets/bindings exist, then verify request logs and traces appear for one success path and one blocked path. |
| Feedback delivery and malicious-alert notifications leave the Worker through the Cloudflare-compatible notification path | DEP-03 | External mail provider credentials and inbox receipt are not fully mockable in the repo | Submit one feedback request and trigger one controlled malicious-intent alert in preview, then verify operator receipt or provider dashboard delivery. |

---

## Validation Sign-Off

- [x] All tasks have `<automated>` verify or Wave 0 dependencies
- [x] Sampling continuity: no 3 consecutive tasks without automated verify
- [x] Wave 0 covers all missing references
- [x] No watch-mode flags
- [x] Feedback latency < 120s
- [x] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
