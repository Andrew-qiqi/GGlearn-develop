# Phase 05: Parser Bootstrap and Provider Abstraction - Research

**Researched:** 2026-04-05
**Domain:** Cloudflare-first parser quota control, provider abstraction, and graceful fallback UX
**Confidence:** MEDIUM

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

### Product posture
- Early public BYOK remains fully free.
- The platform continues to provide `Document Parsing` by default during the early public stage.
- Users should not be forced to configure their own parser provider in this phase.

### Parser limit policy
- Platform-funded parsing is limited to `10` successful page-level parsing operations per natural day.
- Quota deduction is based on a real successful platform parser call, not on button clicks or failed attempts.
- When the parser limit is exhausted, AI analysis must continue, but it should run without document parsing and therefore with lower precision.

### User experience
- Quota visibility should live in the Settings entry, not in the main analysis flow.
- The settings display should show an exact numeric format such as `7/10`.
- Only analyses that are actually downgraded due to parser unavailability should show a lightweight warning.
- The warning text should be `Low accuracy`.
- The hover detail should read: `Document parsing is unavailable for this analysis, so precision may be lower.`
- Product-facing copy should say `Document Parsing` and should not expose `Azure`.

### Service truth and storage
- Quota truth must be enforced server-side, not in local storage.
- The minimal implementation should use Cloudflare `D1` as the source of truth for anonymous parser quota tracking.
- The anonymous identity key for this phase should be `ip_hash + date_key`.
- IP hashing must use a dedicated `USAGE_HASH_SECRET`, not `API_TOKEN_SECRET` or another existing secret.
- Because current user volume is very small, the first version should avoid complex anti-abuse, multi-device reconciliation, or large-scale fallback design.

### Architecture boundary
- This phase must introduce a clean parser abstraction boundary so Azure is no longer the only implicit implementation path.
- Azure may remain the internal first provider implementation for now, but it must sit behind that abstraction.
- Mature teaching business logic must not be changed in this phase.

### Deferred Ideas (OUT OF SCOPE)
- Parser BYOK.
- Multiple parser providers with a formal provider picker UI.
- Login-based quota sync across devices.
- Paid parser packs, subscriptions, or commercialization rules.
- Complex anti-abuse logic beyond the very small current-user scenario.

</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| PARSE-01 | Early users should be able to use platform-managed document parsing with low setup friction. | Keep current no-setup parser path, but wrap it behind a single parser service. |
| PARSE-02 | Parser usage and cost must become observable and controllable instead of relying on accidental free quota. | D1-backed daily quota service plus a settings usage endpoint and downgrade metadata. |
| PARSE-03 | Parser providers must be abstracted so Azure is no longer the only implicit path. | Introduce a small parser provider interface and a single access service used by both `/api/parse` and integrated explain parsing. |
| PARSE-04 | Users who do not provide their own parser access should be subject to explicit platform-managed parsing limits. | Enforce `10/day` only on successful platform parse calls and expose remaining usage in Settings. |
</phase_requirements>

## Repo Workflow Constraints (from AGENTS.md)

- Use `gsd-brief-handoff` only for pre-GSD brief preparation, not for this planning run.
- Work should now continue through normal GSD planning/execution artifacts.

## Summary

Phase 05 should be planned as one thin server-side parser access layer plus one thin frontend visibility layer. The server side needs three pieces only: a minimal parser provider interface, a D1-backed daily usage service keyed by `ip_hash + date_key`, and a downgrade-aware integration path that can either perform parsing or intentionally skip it without failing the analysis. The frontend side needs two pieces only: a Settings usage row and a per-analysis downgraded-accuracy flag that can show `Low accuracy` with the agreed hover detail.

The current code already has two natural integration points: [SlideTutor-AI/src/worker/routes/parse.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/src/worker/routes/parse.ts) for explicit parse requests, and [SlideTutor-AI/api/lib/generateService.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/api/lib/generateService.ts) for the integrated explain flow that currently calls `performAzureAnalysis(...)` opportunistically before prompt construction. That means the minimal abstraction boundary should sit below those two callers, not inside the teaching prompt code.

The best downgrade signaling mechanism is not to alter the streamed explain body. Because the explain response is already a streaming text response consumed by [useSlideAnalysis.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/src/hooks/useSlideAnalysis.ts), the lightest safe signal is an HTTP response header such as `x-slidetutor-parse-mode: normal|degraded`. The hook can read that header before streaming begins, persist a small per-page accuracy flag in `tutorStore`, and the result UI can show the orange `Low accuracy` warning only when the server explicitly downgraded that analysis.

For quota truth, D1 is the correct minimal fit. It is server-side, Cloudflare-native, and easy to bind into the existing Worker environment. For the current very small user volume, the simplest acceptable flow is:

1. derive an anonymous usage identity from client IP plus `USAGE_HASH_SECRET`
2. read current daily usage from D1
3. if usage is still below `10`, allow the parser call
4. only after a successful parser result, increment the daily usage row
5. if usage is exhausted, skip parser work and return downgraded analysis metadata instead of blocking

This does not fully eliminate race conditions under concurrent requests, but it matches the user-approved low-complexity assumption for the current sub-50-user stage. If stronger correctness is needed later, the same service boundary can move to stricter transactional logic or a login-based identity model without re-cutting the feature shape.

**Primary recommendation:** Plan Phase 05 as two execution plans: one backend plan for parser abstraction, D1 quota truth, and downgraded analysis signaling; one frontend/integration plan for Settings usage display, per-analysis `Low accuracy` UX, and docs/tests.

## Standard Stack

### Core

| Library / Feature | Purpose | Why Standard |
|---------|---------|--------------|
| Cloudflare `D1` binding | Daily parser usage truth store | Native SQLite-backed storage for Workers and enough for the current tiny-user quota model. |
| Existing Worker runtime | Server-side quota enforcement and parser access | Current deployment base is already Cloudflare-first. |
| Existing Azure parser code | Initial provider implementation | Keeps current parsing behavior while moving behind an abstraction boundary. |
| Existing Zustand stores | Settings and analysis-status UI state | Minimal incremental fit for displaying quota and downgraded-analysis metadata. |

### Supporting

| Library / Feature | Purpose | When to Use |
|---------|---------|-------------|
| `crypto.subtle` / Worker crypto helpers | Derive stable anonymous usage hashes from IP + secret | Use in the quota identity helper instead of exposing raw IPs in D1. |
| Worker response headers | Pass downgraded-analysis metadata to the streaming frontend | Use for `/api/generate`, because it avoids changing the streamed artifact contract. |
| A small usage endpoint like `/api/parser-usage` | Populate the Settings `7/10` display from server truth | Use only for the Settings panel, not for inline anxiety-inducing UI. |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| D1 | KV | Easier key/value writes, but weaker fit for future quota queries and less appropriate as quota truth. |
| D1 | R2 | Wrong storage shape for small mutable usage counters. |
| Response header for downgrade state | Embed status into streamed text body | Risks polluting the existing artifact parsing contract and teaching flow. |
| Dedicated parser service boundary | Scattered inline quota checks around callers | Faster to hack, but makes later provider replacement much messier. |
| Strict anti-race quota accounting now | Durable Object or transactional lock service | More correct, but too heavy for the user-approved current scale. |

## Architecture Patterns

### Pattern 1: Parser Access Service Above Provider Implementations

**What:** Create one small service such as `performDocumentParsing(...)` or `resolveParserAccess(...)` that both `/api/parse` and integrated explain parsing call.

**When to use:** Immediately. This is the minimal way to satisfy `PARSE-03` without changing prompt logic.

**Why it fits this repo:** The current direct dependency on [azureParse.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/api/lib/azureParse.ts) appears in more than one runtime path already.

### Pattern 2: Quota Service Separate From Provider Service

**What:** Keep the daily usage truth in a dedicated module, for example:

- `api/lib/parser/usageIdentity.ts`
- `api/lib/parser/usageStore.ts`
- `api/lib/parser/accessService.ts`

**When to use:** Immediately. Quota truth and parser provider selection are separate concerns and should stay that way.

**Minimal schema recommendation:**

```sql
CREATE TABLE parser_usage_daily (
  usage_key TEXT PRIMARY KEY,
  date_key TEXT NOT NULL,
  usage_count INTEGER NOT NULL DEFAULT 0,
  updated_at TEXT NOT NULL
);
```

Where `usage_key` is a hash derived from `ip_hash + ":" + date_key`.

### Pattern 3: Degrade-Not-Block Integration

**What:** The parser access layer returns a structured result that distinguishes:

- parser succeeded
- parser unavailable because daily quota is exhausted
- parser failed unexpectedly

**When to use:** In `generateService` and `/api/parse`.

**Recommended behavior:**
- On quota exhaustion: continue explain flow with no `layoutBlocks`, mark response as degraded.
- On parser failure: fail open for integrated explain, but do not decrement quota.
- On explicit `/api/parse` route over quota: return a structured non-200 business response or a clear 429-like parse-specific response only for that endpoint, not for explain.

### Pattern 4: Settings-Only Usage Visibility

**What:** Fetch usage truth from the server only when the Settings panel is opened or the AI tab is visible.

**When to use:** For the `7/10` display.

**Why it fits:** The user explicitly wants low-anxiety visibility, not a quota meter shadowing every learning action.

### Pattern 5: Per-Analysis Accuracy Flag in Page State

**What:** Add a small field to page state such as:

```ts
analysisAccuracy?: 'normal' | 'low';
analysisAccuracyReason?: 'parser-unavailable';
```

**When to use:** Set it in the slide-analysis hook from the response header, not from client inference.

**Why it fits:** It keeps warning display honest and tied to the actual server execution path.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Client-side quota truth | LocalStorage or IndexedDB counters | D1-backed server truth | Client storage is trivial to bypass and drifts from server behavior. |
| Overengineered identity | Device fingerprinting, multi-factor anonymous identity | `ip_hash + date_key` | User volume is tiny and login is a later phase. |
| Parser-provider product UI | A provider picker or parser credential UI | Keep parser platform-managed for now | Explicitly out of scope for this phase. |
| Inline quota stress UI | Persistent banners around Analyze | Settings-only display plus downgraded-analysis warning | Matches the product direction and reading experience goals. |
| Stream-body protocol changes | Custom prelude markers in explain stream | Response headers | Safer with the existing streaming and artifact parsing path. |

## Current Code Inventory

| Area | What Exists | Planning Implication |
|------|-------------|----------------------|
| Parser provider | [SlideTutor-AI/api/lib/azureParse.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/api/lib/azureParse.ts) contains Azure-specific submission and polling logic | Good first provider implementation; move behind an interface, do not rewrite the extraction math now. |
| Integrated explain parsing | [SlideTutor-AI/api/lib/generateService.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/api/lib/generateService.ts) performs parser work before prompt construction and already fails open on parser exceptions | Natural place to add quota-aware degrade behavior while protecting teaching logic. |
| Direct parse route | [SlideTutor-AI/src/worker/routes/parse.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/src/worker/routes/parse.ts) calls Azure directly | Must be switched to the same parser access layer so quota truth is shared. |
| Settings UI | [SlideTutor-AI/src/components/SettingsModal.tsx](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/src/components/SettingsModal.tsx) already hosts AI settings and BYOK copy | Best place to show `Document Parsing 7/10`. |
| Analysis state | [SlideTutor-AI/src/store/tutorStore.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/src/store/tutorStore.ts) tracks per-page explanation state but not downgraded-accuracy metadata yet | Small store extension is sufficient for the warning UX. |
| Worker config | [SlideTutor-AI/wrangler.jsonc](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/wrangler.jsonc) already exists | Add D1 binding and new `USAGE_HASH_SECRET` env support here. |

## Common Pitfalls

### Pitfall 1: Deducting quota on intent instead of success

**What goes wrong:** A click, retry, or failed parser call still burns quota.

**How to avoid:** Increment only after a successful provider parse result is returned.

### Pitfall 2: Showing downgrade warnings based on client guesswork

**What goes wrong:** The UI shows `Low accuracy` for reasons unrelated to parser unavailability, or misses real downgraded analyses.

**How to avoid:** Make the server declare downgrade state explicitly via a header or structured metadata.

### Pitfall 3: Leaving `/api/parse` and integrated explain on different quota logic

**What goes wrong:** Settings numbers drift from actual parsing behavior, or one route bypasses the limit.

**How to avoid:** Both call paths must go through the same usage and provider abstraction layer.

### Pitfall 4: Letting Phase 05 absorb parser commercialization

**What goes wrong:** The phase expands into parser BYOK, credits, or plan tiers.

**How to avoid:** Treat this phase as platform bootstrap only. Anything involving user-provided parser credentials stays deferred.

### Pitfall 5: Tight-coupling D1 details into UI code

**What goes wrong:** The frontend starts reasoning about anonymous identity or quota calculations.

**How to avoid:** The frontend should only consume a usage summary API and downgrade metadata.

### Pitfall 6: Rewriting teaching logic while integrating degrade behavior

**What goes wrong:** The explain/distill contracts or prompt inputs change in ways unrelated to parser access.

**How to avoid:** Keep the change boundary around parser preparation, response headers, and UI display only.

## Recommended Plan Shape

### Plan 05-01: Backend parser access layer and D1 quota truth

Should cover:
- D1 binding and migration
- anonymous usage identity helper
- daily usage read/increment service
- parser provider interface with Azure implementation
- shared parser access service used by `/api/parse` and integrated explain parsing
- downgrade metadata from `/api/generate`
- server tests for quota deduction rules and degrade-not-block behavior

### Plan 05-02: Frontend settings visibility and downgraded-analysis UX

Should cover:
- parser usage endpoint consumption in Settings
- exact `7/10` display inside AI Settings
- per-page downgraded-analysis state
- orange `Low accuracy` warning with hover text
- docs and tests for the new product behavior

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | Existing `vitest` setup plus Worker tests already used in Phases 03/04 |
| Quick run command | `npm test -- api/security.test.ts` or targeted route/store tests |
| Full suite command | `npm test` and `npm run test:workers` |

### Phase Requirements -> Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| PARSE-01 | A user can still analyze slides without configuring parser access | worker integration | `npm run test:workers -- test/workers/generate-stream.worker.test.ts` | Partial coverage exists; extend |
| PARSE-02 | Successful parser calls increment usage in D1-backed logic and settings can read server truth | service + route | `npm test -- api/parserUsage.test.ts src/components/SettingsModal.test.tsx` | No |
| PARSE-03 | Azure sits behind a parser abstraction used by both parse entry points | unit + worker integration | `npm test -- api/parserAccess.test.ts test/workers/parse-route.worker.test.ts` | No |
| PARSE-04 | Over-quota explain requests degrade instead of blocking, and show `Low accuracy` only when degraded | hook + UI + worker integration | `npm test -- src/hooks/useSlideAnalysis.test.ts src/components/CanvasTutor.test.tsx` | Extend/add |

### Wave 0 Gaps

- [ ] `SlideTutor-AI/api/parserUsage.test.ts` for successful-only deduction and daily limit behavior
- [ ] `SlideTutor-AI/api/parserAccess.test.ts` for provider abstraction and degrade semantics
- [ ] `SlideTutor-AI/test/workers/parse-route.worker.test.ts` for shared route behavior and usage endpoint behavior
- [ ] `SlideTutor-AI/src/components/SettingsModal.test.tsx` extension for `Document Parsing 7/10`
- [ ] `SlideTutor-AI/src/hooks/useSlideAnalysis.test.ts` extension for downgraded-analysis header handling

## Sources

### Primary (HIGH confidence)

- Cloudflare D1 Worker API: https://developers.cloudflare.com/d1/worker-api/ - Worker binding query model for D1
- Cloudflare D1 Local development docs: https://developers.cloudflare.com/d1/best-practices/local-development/ - binding and local development behavior
- Cloudflare Workers headers / request model: https://developers.cloudflare.com/workers/runtime-apis/request/ - request metadata and Worker request semantics

### Repo-primary (HIGH confidence)

- `SlideTutor-AI/api/lib/azureParse.ts`
- `SlideTutor-AI/api/lib/generateService.ts`
- `SlideTutor-AI/src/worker/routes/parse.ts`
- `SlideTutor-AI/src/components/SettingsModal.tsx`
- `SlideTutor-AI/src/store/tutorStore.ts`
- `SlideTutor-AI/src/hooks/useSlideAnalysis.ts`
- `SlideTutor-AI/wrangler.jsonc`

## Metadata

**Confidence breakdown:**
- Integration boundary: HIGH - current code already exposes the correct cut points
- D1 quota model: MEDIUM - correct for this scale, but not hardened for concurrency
- UI downgrade path: HIGH - response headers plus page-state metadata fits the current architecture cleanly

**Research date:** 2026-04-05
**Valid until:** 2026-05-05
