---
phase: 1
slug: environment-core-stability
status: draft
nyquist_compliant: true
wave_0_complete: false
created: March 26, 2026
---

# Phase 1 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | vitest |
| **Config file** | SlideTutor-AI/vite.config.ts |
| **Quick run command** | `npm run test` |
| **Full suite command** | `npm run test` |
| **Estimated runtime** | ~10 seconds |

---

## Sampling Rate

- **After every task commit:** Run `npm run test`
- **After every plan wave:** Run `npm run test`
- **Before /gsd:verify-work:** Full suite must be green
- **Max feedback latency:** 10 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 1-01-01 | 01 | 1 | SEC-01 | unit | `npm run test -- api/security.test.ts` | ❌ W0 | ⬜ pending |
| 1-01-02 | 01 | 1 | SEC-01 | unit | `npm run test -- server.test.ts` | ❌ W0 | ⬜ pending |
| 1-01-03 | 01 | 2 | STAB-02 | unit | `npm run test -- layoutUtils.test.ts` | ❌ W0 | ⬜ pending |
| 1-01-04 | 01 | 2 | STAB-02 | unit | `npm run test -- prompts.test.ts` | ✅ | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `SlideTutor-AI/api/security.test.ts` — stubs for SEC-01 (env validation)
- [ ] `SlideTutor-AI/server.test.ts` — stubs for SEC-01 (rate limiting)
- [ ] `SlideTutor-AI/src/lib/pdf/layoutUtils.test.ts` — stubs for STAB-02 (PDF layout)

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Critical vulnerability fix | SEC-01 | Verify no GEMINI_API_KEY in dist/ | Build app, grep dist/ for key. |

---

## Validation Sign-Off

- [x] All tasks have `<automated>` verify or Wave 0 dependencies
- [x] Sampling continuity: no 3 consecutive tasks without automated verify
- [x] Wave 0 covers all MISSING references
- [x] No watch-mode flags
- [x] Feedback latency < 10s
- [x] `nyquist_compliant: true` set in frontmatter

**Approval:** pending 2026-03-26
