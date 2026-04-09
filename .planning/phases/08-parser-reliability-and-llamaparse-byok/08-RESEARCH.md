# Phase 08: Parser Reliability and LlamaParse BYOK - Research

**Researched:** 2026-04-09
**Domain:** Stabilizing the live parser path while adding a dedicated parser BYOK flow for `My API`
**Confidence:** MEDIUM-HIGH

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

#### Locked parser product posture
- `Platform API` keeps a platform-managed parser path and remains locked to `Volcengine`.
- `Platform API` users should not configure their own parser.
- `My API` now supports parser BYOK through a dedicated parser configuration path.
- The first `My API` parser BYOK provider is `LlamaParse`.
- If a `My API` user has no parser configured, the existing no-parser degraded analysis path must continue to work.

#### Locked parser quota and routing rules
- Remove user-visible parser daily quota semantics and parser trial semantics.
- Do not keep or reintroduce the old BYOK parser trial / shared-platform-parser behavior.
- Parser protection may remain as internal infrastructure safeguards only; it should not surface as product quota UX.
- Parser-related failures should be classified by source instead of being merged into one misleading generic `429` story.

#### Locked code-structure direction
- Keep the current explain pipeline shape centered on normalized parser output consumed as `LayoutBlock[]` or a compatible minimal structure.
- Add `LlamaParse` through a modular provider/adapter path rather than hardwiring provider-specific logic into the old platform parser access flow.
- Keep parser routing boundaries explicit:
  - platform parser for `Platform API`
  - parser BYOK for `My API`
  - degraded no-parser fallback when no parser is configured

#### Phase sequencing rules
- Parser stabilization and parser BYOK happen in this phase.
- Model configuration cleanup, including Gemini thinking-parameter normalization, stays in the next separate phase.

### Claude's Discretion

Copied from `CONTEXT.md`.

- The exact internal shape of parser error categories may be chosen pragmatically if the categories clearly separate platform parser unavailability, route-level limiting, and upstream provider limiting/failure.
- The exact `LlamaParse` result normalization may be chosen pragmatically if it preserves the current explain-chain contract and avoids forcing a whole-pipeline rewrite.
- The minimal frontend settings UX for parser BYOK may be chosen pragmatically if the boundary stays low-friction and explicit.

### Deferred Ideas (OUT OF SCOPE)
- additional parser BYOK providers beyond `LlamaParse`
- `LiteParse`, `Docling`, or self-hosted parser infrastructure
- replacing the platform parser provider away from `Volcengine`
- model configuration stability work, including Gemini thinking-parameter cleanup

</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| PARSE-05 | Platform-managed parsing must remain usable without user-visible daily parser quotas, parser trials, or other product-level parser allowances. | Remove D1-backed parser quota/trial UX from the product path, stop exposing parser-usage semantics as a user feature, and keep only internal protection where needed. |
| PARSE-06 | `My API` must support optional parser BYOK through a dedicated parser configuration path, starting with `LlamaParse`. | Add parser configuration to persisted BYOK settings, extend backend access resolution to pass parser credentials/config, and isolate `LlamaParse` in its own provider/adapter path. |
| PARSE-07 | Parser failures, route limits, and upstream provider limits must be surfaced accurately instead of being collapsed into misleading generic rate-limit messaging. | Normalize parser error taxonomy across backend routes, API client, and hooks; separate route-level 429 from parser/provider failures and from degraded fallback. |

</phase_requirements>

## Project Constraints (from AGENTS.md)

- The repo uses `gsd-brief-handoff` for narrow pre-GSD workflow setup when important work needs a deep human discussion before planning.
- Live pre-GSD brief files live under `docs/discuss/`, and the approved Phase 08 brief is the canonical handoff document for this phase.
- This is phase-level work, not a new-project or new-milestone initialization flow.

## Summary

Phase 08 should not be planned as "swap one parser vendor for another." The codebase already has a working platform parser path, but the current runtime mixes three different concerns in one place:

1. **User-visible parser quota policy**
   `usageStore.ts` hard-codes a `10/day` platform parser allowance, `accessService.ts` enforces it for both direct parse and integrated explain, and `SettingsModal.tsx` / `apiClient.ts` still surface parser-usage UI.

2. **Access-mode boundary leakage**
   `generateService.ts` runs the integrated parser preflight for every `explain` request before it has cleanly separated `Platform API` from `My API`, so BYOK can still accidentally depend on platform parser behavior.

3. **Error taxonomy collapse**
   There is a real Worker route limiter in `src/worker/routes/generate.ts` (`10/min` by IP), parser-specific 429/503 responses from `accessService.ts`, and hard-coded frontend copy in `useSlideAnalysis.ts` / `useFollowUp.ts` that still says `15 RPM`.

The planning implication is important: Phase 08 is first a **boundary cleanup** phase, then a **provider addition** phase.

`LlamaParse` is structurally suitable for `My API` parser BYOK because the official cloud API already exposes a dedicated parse API, job status/polling, and structured/spatial outputs. But it is **not** a drop-in replacement for the current synchronous `Volcengine` page parser:

- its cloud parse flow is job-oriented rather than single-request immediate response
- it can return markdown/text/structured outputs that are not guaranteed to match the existing `LayoutBlock[]` contract directly
- it introduces a second parser ownership model (`user-configured parser`) that must not be layered into the old platform parser quota service

**Primary recommendation:** Plan Phase 08 as three focused plans:

- one plan for removing old parser quota/product semantics and fixing error taxonomy
- one plan for adding persisted `My API` parser configuration plus the `LlamaParse` provider path
- one plan for hardening the remaining `Volcengine` platform parser path, tests, and docs around the new boundaries

## Standard Stack

### Core

| Library / Service | Purpose | Why Standard |
|-------------------|---------|--------------|
| Existing `ParserProvider` seam in `api/lib/parser/provider.ts` | Minimal backend parser contract | Already exists and should stay the normalization point for provider-specific parser logic. |
| Existing `generateService.ts` explain preflight | Central parser-to-explain bridge | This is where access-mode boundary cleanup must happen before adding parser BYOK. |
| `Volcengine` OCRPdf API | Platform-managed parser | Already live and documented locally; supports `image_base64`, structured `textblocks`, and normalized boxes. |
| `LlamaParse` Cloud API v2 | `My API` parser BYOK provider | Official API supports parse upload/job flow plus structured and spatial outputs, making it the best current parser BYOK candidate. |
| Zustand + IndexedDB-backed settings (`uiStore.ts`, `config/models.ts`, `lib/db`) | Persisted user access settings | Existing pattern for BYOK model access; parser BYOK should extend this instead of inventing a second settings store. |

### Supporting

| Library / Tool | Purpose | When to Use |
|----------------|---------|-------------|
| `vitest` | unit/component tests | parser config normalization, backend error taxonomy, API client handling, settings UI |
| `@cloudflare/vitest-pool-workers` | Worker route tests | `/api/generate`, `/api/parse`, `/api/parser-usage` behavior under the new parser semantics |
| repo docs under `docs/backend` and `docs/frontend` | architecture and contract docs | update parser ownership and request/error semantics after implementation |

### Official parser-source notes

#### `Volcengine`
- Local official snapshots show OCRPdf accepts `application/x-www-form-urlencoded`, supports `image_base64`, and returns `textblocks[].text`, `label`, and `norm_box`.
- Local pricing/rate snapshot shows free-tier usage is small and the service defaults to low QPS, with formal service support around `2 QPS`. This explains why short-burst parser traffic can become a real runtime concern even when the app itself is not parallelizing heavily.

#### `LlamaParse`
- Official API v2 guide documents a dedicated parse API with upload/job flow, not just an SDK wrapper.
- Official docs expose job status, result retrieval, and multiple output modes including structured and spatial output options.
- Official rate-limit docs show lower limits on free organizations than the general API baseline, so the adapter should be written with polling/backoff/error classification rather than a naive tight loop.

## Architecture Patterns

### Pattern 1: Access-mode-first parser routing

**What:** Select parser strategy by access mode first, provider second:

- `Platform API` -> platform-managed `Volcengine`
- `My API` with parser config -> user-managed `LlamaParse`
- `My API` without parser config -> degraded no-parser analysis

**Why:** This matches the product boundary the user locked and prevents `My API` from continuing to borrow the platform parser.

**Planning implication:** The old `accessService.ts` cannot remain the one place that decides everything for both access modes if it still assumes a shared D1 quota gate and one platform parser provider.

### Pattern 2: Keep parser normalization on the backend

**What:** Parser providers return either `LayoutBlock[]` or a minimal compatible structure before the explain chain consumes them.

**Why:** `generateService.ts` already assumes normalized blocks and turns them into prompt text. Pushing provider-specific parsing details into the frontend would create brittle cross-layer coupling.

**Planning implication:** `LlamaParse` integration should normalize results server-side. If full coordinate fidelity is not available or is unstable, define the minimum viable block shape explicitly and test it.

### Pattern 3: Separate parser error classes before touching UI copy

**What:** The system should distinguish at least these cases:

- route-level rate limit (`/api/generate` or other Worker guards)
- platform parser unavailable
- platform parser internally protected / temporarily denied
- BYOK parser unavailable or misconfigured
- upstream parser provider rate limited
- degraded no-parser fallback intentionally used

**Why:** The current user-facing `15 RPM` copy is misleading because it conflates multiple sources.

**Planning implication:** Fix backend error codes first, then update `apiClient.ts`, `useSlideAnalysis.ts`, and `useFollowUp.ts` to render source-accurate messages.

### Pattern 4: Isolate `LlamaParse` async behavior behind a provider adapter

**What:** `LlamaParse` should not force the rest of the pipeline to know about job IDs, polling, or provider-specific output retrieval.

**Why:** The rest of the explain path expects an awaited parser result.

**Planning implication:** The provider or a dedicated BYOK-parser service should encapsulate:
- request construction
- upload/start
- polling / backoff
- timeout budget
- result normalization

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Parser BYOK persistence | ad hoc `localStorage` keys or parser-only store | extend `AiAccessSettings` + `uiStore.ts` + existing DB settings pattern | Keeps parser BYOK aligned with current model BYOK persistence. |
| Parser routing policy | `if provider == X` checks scattered through frontend and backend | one explicit access-resolution boundary | Prevents platform parser leakage into BYOK paths. |
| Error UX | string-matching generic `429` copy in multiple hooks | backend codes + centralized client interpretation | Makes parser/provider failures debuggable and maintainable. |
| Platform parser protection | user-facing trial counters or daily parser balances | internal route/provider safeguards only | Matches the new product direction and avoids fake product semantics. |
| Future-proofing | large provider marketplace abstraction | smallest viable provider seam for `Volcengine` + `LlamaParse` | This phase needs maintainability, not a framework. |

## Runtime State Inventory

| Category | Items Found | Action Required |
|----------|-------------|-----------------|
| Platform parser quota truth | `DAILY_PLATFORM_PARSE_LIMIT = 10` in `api/lib/parser/usageStore.ts` and D1-backed `parser_usage_daily` persistence | Remove or demote from product behavior; decide whether any operator-only usage tracking remains needed. |
| Shared parser access layer | `accessService.ts` currently assumes one platform parser plus D1 gating for both direct parse and integrated explain | Split platform parser policy from BYOK parser policy. |
| Explain-chain parser entry | `generateService.ts` always resolves parser access before building explain input if `task === 'explain'` and no `layoutBlocks` are pre-supplied | Make access-mode-aware parser resolution the first cleanup target. |
| Frontend parser UX | `SettingsModal.tsx` fetches `/api/parser-usage`; `apiClient.ts` exposes parser-usage API; hooks still contain hard-coded quota/rate-limit copy | Remove or repurpose quota UX and align hook messages to real error codes. |
| Worker route limiter | `/api/generate` still has `10/min` IP limiting in `src/worker/routes/generate.ts` | Keep as an abuse guard, but distinguish it from parser/provider limits. |
| Existing parser provider contract | `ParserProvider.analyzePage({ base64Image, env })` is synchronous from the caller's perspective and only models page-image input | Review whether this contract needs a light extension for BYOK parser config and timeout handling. |

## Common Pitfalls

### Pitfall 1: Removing the quota constant but leaving the quota UX

**What goes wrong:** The D1 limit is deleted, but `/api/parser-usage`, `SettingsModal`, and frontend copy still imply a parser allowance product.

**How to avoid:** Plan quota removal as a cross-layer cleanup: backend policy, route surface, API client, and settings UI together.

### Pitfall 2: Adding `LlamaParse` inside the old platform parser service

**What goes wrong:** BYOK parser configuration becomes entangled with `usageStore`, platform access rules, and `Volcengine`-specific assumptions.

**How to avoid:** Introduce a separate BYOK parser configuration and resolution path before wiring the provider.

### Pitfall 3: Treating all `429` responses as one problem

**What goes wrong:** Route-level rate limits, provider rate limits, and parser policy responses collapse into one misleading message, so both users and operators chase the wrong issue.

**How to avoid:** Introduce explicit backend codes for each rate-limit/failure source and update hooks to branch on those codes.

### Pitfall 4: Underestimating `LlamaParse` latency semantics

**What goes wrong:** The adapter polls too aggressively or waits without timeout/cancellation, making explain requests feel hung.

**How to avoid:** Put polling/backoff/timeout in one adapter or service and make the error mode explicit when the BYOK parser times out or fails upstream.

### Pitfall 5: Assuming `LlamaParse` block geometry matches `Volcengine`

**What goes wrong:** The app expects `bbox`-style spatial metadata that may not arrive in the same shape.

**How to avoid:** Decide the minimum normalized block contract needed by the current explain chain and test that contract directly before expanding parser-specific richness.

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | `vitest` + `@cloudflare/vitest-pool-workers` |
| App config | `SlideTutor-AI/vite.config.ts` |
| Worker config | `SlideTutor-AI/vitest.worker.config.ts` |
| Parser unit command | `npm test -- api/parserAccess.test.ts api/parserUsage.test.ts api/lib/parser/volcengineProvider.test.ts api/lib/generateService.platform.test.ts` |
| Frontend/settings command | `npm test -- src/lib/api/apiClient.test.ts src/store/uiStore.test.ts src/components/SettingsModal.test.tsx src/hooks/useSlideAnalysis.test.ts` |
| Worker route command | `npm run test:workers -- test/workers/parse-route.worker.test.ts test/workers/security-observability.worker.test.ts` |

### Wave 0 Gaps

- [ ] No `LlamaParse` adapter tests exist yet; Phase 08 should add provider-specific tests before wiring the live path.
- [ ] No persisted parser-BYOK settings schema exists yet inside `AiAccessSettings`; config normalization tests will need to be added or expanded.
- [ ] The current parser-usage route/UI may be removed or repurposed; tests should be updated as part of the same plan instead of left to drift.

## Suggested Plan Shape

### Recommended Plan 08-01: Parser policy cleanup and error taxonomy
- Remove old parser quota/product semantics from backend and settings UX.
- Stop `My API` from implicitly reading platform parser policy.
- Normalize parser-related error codes/messages across backend, API client, and hooks.

### Recommended Plan 08-02: Persisted `My API` parser configuration + `LlamaParse` adapter
- Extend settings schema and persistence for parser BYOK.
- Pass parser configuration through the request/access path.
- Implement a `LlamaParse` provider or dedicated BYOK-parser service with polling/backoff/timeout and normalized output.

### Recommended Plan 08-03: Platform parser hardening and doc/test sync
- Keep `Volcengine` as the platform-managed parser path.
- Harden direct parse and integrated explain behavior under the new boundaries.
- Update docs/tests so parser ownership, fallback behavior, and rate-limit semantics are consistent.

## Sources

### Primary (HIGH confidence)

- `docs/discuss/phases/08-parser-reliability-and-llamaparse-byok-brief.md`
- `.planning/phases/08-parser-reliability-and-llamaparse-byok/08-CONTEXT.md`
- `SlideTutor-AI/api/lib/parser/accessService.ts`
- `SlideTutor-AI/api/lib/parser/usageStore.ts`
- `SlideTutor-AI/api/lib/generateService.ts`
- `SlideTutor-AI/src/worker/routes/generate.ts`
- `SlideTutor-AI/src/worker/routes/parse.ts`
- `SlideTutor-AI/src/worker/routes/parser-usage.ts`
- `SlideTutor-AI/src/components/SettingsModal.tsx`
- `SlideTutor-AI/src/store/uiStore.ts`
- `SlideTutor-AI/src/config/models.ts`
- `SlideTutor-AI/src/lib/api/apiClient.ts`
- `SlideTutor-AI/src/hooks/useSlideAnalysis.ts`
- `SlideTutor-AI/src/hooks/useFollowUp.ts`

### Official external sources (HIGH confidence)

- LlamaParse API v2 guide: `https://developers.llamaindex.ai/python/cloud/llamaparse/api-v2-guide/`
- LlamaCloud rate limits: `https://developers.llamaindex.ai/python/cloud/general/rate_limits/`
- LlamaCloud pricing: `https://developers.llamaindex.ai/python/cloud/general/pricing/`

### Local official snapshots (HIGH confidence for current planning context)

- `tmp_files/volcengine_document_parse_intellgence/2.md`
- `tmp_files/volcengine_document_parse_intellgence/3.md`

## Metadata

- Parser-boundary diagnosis confidence: HIGH
- `Volcengine` runtime understanding confidence: HIGH
- `LlamaParse` provider-fit confidence: MEDIUM-HIGH
- Exact `LlamaParse` -> `LayoutBlock[]` normalization confidence: MEDIUM
- Research valid until: 2026-05-09 unless parser provider contracts or product scope change materially
