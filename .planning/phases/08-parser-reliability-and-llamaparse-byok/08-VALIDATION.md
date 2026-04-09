---
phase: 08
slug: parser-reliability-and-llamaparse-byok
status: ready_for_signoff
nyquist_compliant: true
wave_0_complete: true
created: 2026-04-09
updated: 2026-04-09
---

# Phase 08 Validation Strategy

> Per-phase validation contract for parser boundary cleanup, `LlamaParse` BYOK integration, and parser error normalization.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | `vitest` + `@cloudflare/vitest-pool-workers` |
| **App config** | `SlideTutor-AI/vite.config.ts` |
| **Worker config** | `SlideTutor-AI/vitest.worker.config.ts` |
| **Quick run command** | `npm test -- api/parserAccess.test.ts api/parserUsage.test.ts api/lib/parser/volcengineProvider.test.ts api/lib/parser/llamaparseProvider.test.ts api/lib/generateService.platform.test.ts src/lib/api/apiClient.test.ts` |
| **Worker run command** | `npm run test:workers -- test/workers/parse-route.worker.test.ts test/workers/generate-stream.worker.test.ts test/workers/security-observability.worker.test.ts` |
| **Frontend/settings command** | `npm test -- src/store/uiStore.test.ts src/components/SettingsModal.test.tsx src/hooks/useSlideAnalysis.test.ts src/hooks/useFollowUp.test.ts` |
| **Full phase command** | `npm test -- api/parserAccess.test.ts api/parserUsage.test.ts api/lib/parser/volcengineProvider.test.ts api/lib/parser/llamaparseProvider.test.ts api/lib/generateService.platform.test.ts src/lib/api/apiClient.test.ts src/store/uiStore.test.ts src/components/SettingsModal.test.tsx src/hooks/useSlideAnalysis.test.ts src/hooks/useFollowUp.test.ts && npm run test:workers -- test/workers/parse-route.worker.test.ts test/workers/generate-stream.worker.test.ts test/workers/security-observability.worker.test.ts && npm run lint` |

---

## Per-Task Verification Map

| Task ID | Plan | Requirement | Test Type | Automated Command | Status |
|---------|------|-------------|-----------|-------------------|--------|
| 08-01-01 | 01 | PARSE-05, PARSE-07 | backend policy + service | `npm test -- api/parserAccess.test.ts api/parserUsage.test.ts api/lib/generateService.platform.test.ts` | green |
| 08-01-02 | 01 | PARSE-05, PARSE-07 | client/hooks/settings | `npm test -- src/lib/api/apiClient.test.ts src/components/SettingsModal.test.tsx src/hooks/useSlideAnalysis.test.ts src/hooks/useFollowUp.test.ts` | green |
| 08-02-01 | 02 | PARSE-06 | settings schema + persistence | `npm test -- src/store/uiStore.test.ts src/components/SettingsModal.test.tsx` | green |
| 08-02-02 | 02 | PARSE-06 | parser provider / adapter | `npm test -- api/lib/parser/llamaparseProvider.test.ts api/lib/generateService.platform.test.ts` | green |
| 08-03-01 | 03 | PARSE-05, PARSE-07 | worker route + observability | `npm run test:workers -- test/workers/parse-route.worker.test.ts test/workers/generate-stream.worker.test.ts test/workers/security-observability.worker.test.ts` | green |
| 08-03-02 | 03 | PARSE-05, PARSE-06, PARSE-07 | static docs contract | `rg -n "Volcengine|LlamaParse|Platform API|My API|degraded|ROUTE_RATE_LIMITED|PLATFORM_PARSER|BYOK_PARSER" docs/backend/api-design.md docs/frontend/data-flow.md docs/backend/platform-model-configuration.md docs/changelog/CHANGELOG_TECH.md` | green |

---

## Wave 0 Requirements

- [x] `SlideTutor-AI/api/lib/parser/llamaparseProvider.test.ts` was created with the `LlamaParse` adapter.
- [x] Parser settings persistence tests were expanded to cover the new parser configuration fields.
- [x] Existing parser-usage tests and UI assertions were updated in the same wave that removed user-visible parser quota semantics.

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| A `Platform API` analyze flow no longer presents parser daily-limit UX during long-slide analysis | PARSE-05 | requires end-to-end UI/behavior confirmation | Run a real or staging hosted analyze flow on a longer slide deck and confirm the UI no longer frames parser capacity as a user-visible daily allowance. |
| A `My API` user can add `LlamaParse`, run analyze, and still fall back to degraded analysis when parser config is removed | PARSE-06 | crosses settings, backend, and live parser behavior | Configure parser BYOK, run analyze once, remove parser config, run analyze again, and confirm the second flow degrades instead of breaking. |
| Route-level 429, parser-unavailable, and provider-side failures are distinguishable from one another | PARSE-07 | depends on live response paths and user-facing copy | Trigger one request-rate limit, one parser-disabled/unavailable path, and one BYOK parser error; verify the user and operator-facing signals differ meaningfully. |

---

## Validation Sign-Off

- [x] Existing test infrastructure covers service, client, component, and Worker layers in scope
- [x] Parser-provider additions shipped with direct adapter tests
- [x] Docs-only deliverables have grep-verifiable contracts
- [x] Worker-route observability remained first-class validation
- [x] `nyquist_compliant: true` set in frontmatter
- [x] Phase 08 automated verification suite has been recorded in the phase summaries and validation map

**Approval:** ready for signoff
