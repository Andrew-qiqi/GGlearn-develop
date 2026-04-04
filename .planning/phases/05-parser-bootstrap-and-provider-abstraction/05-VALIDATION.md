---
phase: 05
slug: parser-bootstrap-and-provider-abstraction
status: draft
nyquist_compliant: true
wave_0_complete: false
created: 2026-04-05
---

# Phase 05 Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | `vitest` |
| **Config file** | `SlideTutor-AI/vite.config.ts` |
| **Quick run command** | `npm test -- api/parserUsage.test.ts src/components/SettingsModal.test.tsx` |
| **Full suite command** | `npm test && npm run test:workers` |
| **Estimated runtime** | ~150 seconds |

---

## Sampling Rate

- **After every task commit:** Run `npm test -- api/parserUsage.test.ts src/components/SettingsModal.test.tsx`
- **After every plan wave:** Run `npm test && npm run test:workers`
- **Before `$gsd-verify-work`:** Full suite must be green
- **Max feedback latency:** 150 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 05-01-01 | 01 | 1 | PARSE-02, PARSE-04 | unit/service | `npm test -- api/parserUsage.test.ts` | No - Wave 0 | pending |
| 05-01-02 | 01 | 1 | PARSE-01, PARSE-03, PARSE-04 | unit/worker integration | `npm test -- api/parserAccess.test.ts && npm run test:workers -- test/workers/parse-route.worker.test.ts` | No - Wave 0 | pending |
| 05-02-01 | 02 | 2 | PARSE-02 | component | `npm test -- src/components/SettingsModal.test.tsx` | Yes - extend | pending |
| 05-02-02 | 02 | 2 | PARSE-01, PARSE-04 | hook/component integration | `npm test -- src/hooks/useSlideAnalysis.test.ts src/components/CanvasTutor.test.tsx` | Partial - extend/add | pending |
| 05-02-03 | 02 | 2 | PARSE-01, PARSE-02, PARSE-03, PARSE-04 | regression/integration | `npm test && npm run test:workers` | Yes | pending |

*Status: pending / green / red / flaky*

---

## Wave 0 Requirements

- [ ] `SlideTutor-AI/api/parserUsage.test.ts` - successful-only quota deduction and daily limit coverage
- [ ] `SlideTutor-AI/api/parserAccess.test.ts` - provider abstraction and degrade-not-block coverage
- [ ] `SlideTutor-AI/test/workers/parse-route.worker.test.ts` - Worker route coverage for shared parser access and settings usage endpoint
- [ ] `SlideTutor-AI/src/components/CanvasTutor.test.tsx` - downgraded-analysis warning coverage if not already present

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Settings shows the server truth as `x/10` without adding a persistent anxiety-inducing quota banner to the learning flow | PARSE-02 | UX placement and emotional weight are better judged visually than by automated assertions alone | Open Settings in the running app, verify the parser usage row appears in AI Settings only, and confirm no new always-visible quota UI appears near the main analyze flow. |
| A real downgraded explain run shows `Low accuracy` with the agreed hover text and still completes the explanation stream | PARSE-01, PARSE-04 | Hover copy and inline warning timing need a browser-level check | Exhaust or simulate exhausted parser quota in preview/local dev, analyze a slide, confirm explanation still completes, then hover the warning and verify the full detail text. |
| D1 binding and `USAGE_HASH_SECRET` are configured correctly in the Cloudflare environment | PARSE-02, PARSE-04 | Binding presence and secret configuration are deployment-managed, not repo-only behavior | Check Worker bindings and secrets in Cloudflare, run one successful parse, then confirm usage state becomes visible through the settings usage endpoint. |

---

## Validation Sign-Off

- [x] All tasks have `<automated>` verify or Wave 0 dependencies
- [x] Sampling continuity: no 3 consecutive tasks without automated verify
- [x] Wave 0 covers all missing references
- [x] No watch-mode flags
- [x] Feedback latency < 150s
- [x] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
