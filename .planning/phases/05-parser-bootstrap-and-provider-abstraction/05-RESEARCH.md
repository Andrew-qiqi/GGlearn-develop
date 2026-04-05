# Phase 05: Parser Bootstrap and Provider Abstraction - Research

**Researched:** 2026-04-06
**Domain:** Safe provider transition from Azure to Volcengine for the platform-managed parser path
**Confidence:** MEDIUM-HIGH

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

#### Current product posture
- Early public BYOK remains fully free.
- Platform-managed `Document Parsing` remains available by default.
- Parser quota, degraded fallback, and parser usage visibility are already part of the baseline.

#### Locked provider direction
- The platform-managed parser provider for the live path must be `Volcengine`.
- Azure should no longer remain the implicit runtime default after this phase.
- Product-facing UI and copy should continue to say `Document Parsing`; users should not see provider names.

#### Contract and compatibility
- Existing frontend-facing `LayoutBlock[]` expectations must remain stable in this phase.
- `/api/parse` and integrated explain parsing should continue to return the same effective block shape to downstream consumers.
- Successful parser calls still count against usage; failed or skipped calls do not.
- Existing degraded-analysis semantics, including `Low accuracy`, must keep working.

#### Scope control
- Parser BYOK is not part of this phase.
- MinerU is a possible future BYOK-friendly parser candidate for China users, but it is explicitly deferred.
- ZPAY, hosted access, and other Phase 06 concerns are out of scope here.
- Mature teaching logic must not be reworked as part of the provider swap.

### Claude's Discretion

Copied verbatim from `CONTEXT.md` section `the agent's Discretion`.

- The exact Volcengine-to-`LayoutBlock[]` normalization layer can be designed pragmatically if the output contract stays stable.
- The exact module split for provider normalization can be chosen if later provider additions do not require re-cutting the main chain.
- If a thin compatibility adapter is helpful during the migration, it is acceptable as long as the live platform path is clearly Volcengine-backed.

### Deferred Ideas (OUT OF SCOPE)
- parser BYOK
- parser provider picker UI
- MinerU adapter
- TOS upload flow unless the implementation proves the image-base64 path is insufficient
- payment or hosted-access work
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| PARSE-01 | Early users should be able to use platform-managed document parsing with low setup friction. | Keep the current platform-managed parser path, preserve `Document Parsing` product copy, and cut over the provider underneath the existing access layer. |
| PARSE-02 | Parser usage and cost must become observable and controllable instead of relying on accidental free quota. | Preserve the current D1-backed usage truth and successful-only counting; do not redesign quota or billing in this phase. |
| PARSE-03 | Parser providers must be abstracted so Azure is no longer the only implicit path. | Finish the provider abstraction that already exists by making Volcengine the only live platform parser truth and removing remaining Azure runtime assumptions. |
| PARSE-04 | Users who do not provide their own parser access should be subject to explicit platform-managed parsing limits. | Keep the current route and integrated-explain quota behavior unchanged while replacing the provider implementation. |
</phase_requirements>

## Project Constraints (from CLAUDE.md)

- No `CLAUDE.md` file exists in the repository root, so there are no additional CLAUDE-specific constraints to enforce.

## Summary

Phase 05 is not a greenfield parser phase anymore. The current repo already has the important baseline in place: D1-backed parser usage truth, degraded fallback, settings visibility, and a shared parser access boundary. The repo also already contains partial Volcengine migration work in `SlideTutor-AI/api/lib/env.ts`, `SlideTutor-AI/api/lib/parser/volcengineProvider.ts`, `SlideTutor-AI/api/lib/parser/volcengineProvider.test.ts`, and newer parser tests. The remaining work is to finish the cutover safely, not to redesign the parser system.

The live runtime is still split. `SlideTutor-AI/api/lib/parser/accessService.ts` still wires `createAzureParserProvider()` as the configured platform provider, and `SlideTutor-AI/api/generate.ts` still imports and calls `performAzureAnalysis(...)` directly. That means the repo currently has partial Volcengine implementation alongside Azure as the actual live runtime truth. Plan 05-03 must treat that split as the main risk and remove it explicitly.

Volcengine `OCRPdf` is a good fit for current SlideTutor usage because the official response already exposes the fields the app needs: `textblocks[].text`, `textblocks[].label`, and `textblocks[].norm_box`. The current frontend sends single-page JPEG data URLs, not multi-page PDF uploads, and the official docs allow `image_base64` directly. For the current product path, `image_base64` is sufficient. Do not introduce URL upload, TOS, parser BYOK, or billing scope unless real payload size or runtime tests prove they are necessary.

**Primary recommendation:** Plan 05-03 as one safe cutover plan with three concrete work items: finish the Volcengine adapter, switch all live parser entry points to that adapter, and remove Azure parser assumptions from env/tests/docs so the runtime has one parser truth.

## Standard Stack

### Core

| Library / Service | Version | Purpose | Why Standard |
|-------------------|---------|---------|--------------|
| Volcengine Visual `OCRPdf` API | Query `Version=2021-08-23`, body `version=v3` | Platform-managed parser backend | Official parser API already returns text, labels, and normalized boxes that map cleanly to current SlideTutor needs. |
| `fetch` + `URLSearchParams` | runtime built-in | Submit signed `application/x-www-form-urlencoded` requests | Works in both Worker and Node execution paths without adding a Node-only dependency. |
| Web Crypto `crypto.subtle` HMAC-SHA256 | runtime built-in | Volcengine request signing | Matches Volcengine auth requirements and fits the Cloudflare Worker target better than assuming a Node SDK. |
| Existing `ParserProvider` + `LayoutBlock[]` + `aggregateBlocks()` | local contract | Stable provider seam and consumer contract | Preserves current quota flow, parse route shape, prompt inputs, and UI behavior. |

### Supporting

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `@volcengine/openapi` | `1.36.1` | Official Node.js OpenAPI SDK | Optional alternative only if the execution path is confirmed Node-only and Worker compatibility is intentionally out of scope. |
| `vitest` | `^4.1.0` | Unit and integration-style test runner | Use for parser adapter, env, and access-service tests. |
| `@cloudflare/vitest-pool-workers` | `^0.14.1` | Worker route testing | Use for `/api/parse` and parser unavailable behavior in Worker context. |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Raw signed `fetch` | `@volcengine/openapi` | SDK is official, but Worker compatibility was not verified during this research and the current repo already has a raw-signing path started. |
| `image_base64` | `image_url` + TOS upload | Needed only if real slide-page payloads exceed the documented `8 MB` encoded limit. Adds storage, upload, and lifecycle scope. |
| Stable `LayoutBlock[]` normalization | New provider-specific frontend contract | Richer data is possible, but it would widen scope and force downstream consumer changes for little Phase 05 value. |

**Installation:**

```bash
# Recommended path: no new package required

# Optional Node-only alternative:
npm install @volcengine/openapi@1.36.1
```

**Version verification:**

- `npm view @volcengine/openapi version time.modified description repository.url --json`
- Verified result: `1.36.1`, modified `2026-04-02T13:18:17.229Z`

## Architecture Patterns

### Recommended Project Structure

```text
SlideTutor-AI/api/lib/parser/
|-- accessService.ts        # single live provider truth + quota gating
|-- provider.ts             # parser provider contract
|-- usageStore.ts           # D1-backed successful-only usage truth
|-- volcengineProvider.ts   # live platform parser adapter
|-- azureProvider.ts        # delete or de-live after cutover
`-- azureParse.ts           # legacy Azure implementation, not live runtime

SlideTutor-AI/api/lib/env.ts            # parser secret resolution and validation
SlideTutor-AI/api/generate.ts           # legacy direct parse path to remove or neutralize
SlideTutor-AI/src/worker/routes/parse.ts # route already delegates through accessService
```

### Pattern 1: Single Live Provider Truth

**What:** `accessService.ts` is the only place that should choose the live platform parser provider. Both direct parse and integrated explain should consume that service, not bypass it.

**When to use:** Every platform-managed parser call.

**Example:**

```ts
return createParserAccessService({
  provider: createVolcengineParserProvider(),
  usageStore: createParserUsageStore(database),
  usageHashSecret,
});
```

Source: current repo seam in `SlideTutor-AI/api/lib/parser/accessService.ts` and `SlideTutor-AI/api/lib/parser/provider.ts`

### Pattern 2: Thin Volcengine Adapter That Preserves `LayoutBlock[]`

**What:** Normalize Volcengine `textblocks` into the existing four-field block contract, then pass them through the existing `aggregateBlocks()` helper.

**When to use:** In `volcengineProvider.ts` and for any future provider addition.

**Example:**

```ts
function normalizeBlock(block: VolcengineTextBlock, id: string): LayoutBlock | null {
  const bbox = normalizeNormBox(block.norm_box);
  if (!bbox) return null;

  const type =
    block.label === 'table'
      ? 'table'
      : block.label === 'image'
        ? 'figure'
        : 'text';

  return {
    id,
    type,
    text: buildFallbackText(type, block.text ?? ''),
    bbox,
  };
}
```

Source: Volcengine `OCRPdf` field docs plus local `LayoutBlock` contract in `SlideTutor-AI/src/types.ts`

### Pattern 3: Finish Partial Volcengine Work Instead of Restarting

**What:** The repo already has partial Volcengine adapter/env/test work. Use that as the base, but align it with the final research decisions.

**When to use:** Phase 05-03 implementation planning.

**Current repo reality to preserve in the plan:**

- `SlideTutor-AI/api/lib/parser/volcengineProvider.ts` already implements signed `fetch`.
- `SlideTutor-AI/api/lib/env.ts` already has `requireVolcengineDocumentParseConfig(...)`.
- `SlideTutor-AI/api/lib/parser/volcengineProvider.test.ts` already covers request signing and normalization basics.
- `SlideTutor-AI/api/parserAccess.test.ts` already contains a Volcengine-wiring expectation.

**Planning implication:** 05-03 should be a finish-and-align plan, not a from-scratch provider build.

### Anti-Patterns to Avoid

- **Dual parser truth:** Do not switch `accessService.ts` to Volcengine and leave `api/generate.ts` calling Azure directly.
- **Scope explosion:** Do not add TOS upload, parser BYOK, provider selection UI, or hosted billing work.
- **Contract churn:** Do not widen `LayoutBlock.type` or add provider-specific block fields to downstream consumers.
- **Quota regression:** Do not count failed or skipped parses against usage.
- **Provider leakage:** Do not surface `Azure` or `Volcengine` in product-facing UI copy.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Parser quota and success-only deduction | New quota store or new request accounting path | Existing `usageStore` + `accessService` | The required behavior already exists and is the baseline to preserve. |
| New frontend parser schema | Provider-specific DTOs or richer block taxonomies | Existing `LayoutBlock[]` + `aggregateBlocks()` | Downstream consumers only need `id`, `type`, `text`, and `bbox`. |
| Large-file storage flow | TOS upload pipeline and URL lifecycle management | Current `image_base64` page path | Current SlideTutor usage is single-page image parsing, and official docs support `image_base64` directly. |
| Node-only SDK dependency | Mandatory `@volcengine/openapi` integration | Raw signed `fetch` with small helper | Worker compatibility of the SDK was not verified; the current repo already started the lighter path. |
| Hidden rollback behavior | Secret Azure fallback or second provider truth | Explicit cutover and clear dead-code cleanup | Hidden fallbacks make future planning and debugging unsafe. |

**Key insight:** The safe Phase 05 plan is to finish the seam that already exists, not to invent more parser platform surface area.

## Runtime State Inventory

| Category | Items Found | Action Required |
|----------|-------------|-----------------|
| Stored data | `parser_usage_daily` in D1 stores hashed usage keys, date keys, and counts only. No provider ID or parser payload storage was found. | Code edit only. No data migration required. |
| Live service config | Platform parser credentials live in deployment-managed env/secrets, not git. Current code/docs still mix Azure and Volcengine parser expectations. | Config migration: add Volcengine access key + secret to each runtime. Remove Azure parser secrets only after cutover smoke tests pass. |
| OS-registered state | None verified from repo-visible assets. No scheduled task, pm2, launchd, or systemd references were found. | None. |
| Secrets / env vars | Current repo references `AZURE_DOCUMENT_INTELLIGENCE_ENDPOINT`, `AZURE_DOCUMENT_INTELLIGENCE_KEY`, `VOLCENGINE_ACCESS_KEY_ID`, `VOLCENGINE_SECRET_ACCESS_KEY`, and `USAGE_HASH_SECRET`. | Code edit plus deployment config migration. Remove Azure parser envs from live truth and update `.env.example`, Worker env typing, tests, and docs. |
| Build artifacts | None verified for the recommended path because it uses built-in runtime APIs. | None. If the SDK alternative is chosen, install/update package artifacts separately. |

## Common Pitfalls

### Pitfall 1: `data.detail` string-vs-array ambiguity

**What goes wrong:** The docs describe `data.detail` as an array of page results, but the official Python sample parses it with `json.loads(...)`, which implies a JSON string can also occur.

**Why it happens:** Volcengine examples and field tables are not perfectly aligned for this field.

**How to avoid:** Normalize both forms in the adapter and add tests for both.

**Warning signs:** The provider gets HTTP `200` with `code: 10000`, but normalized block output is empty because the adapter only handled one shape.

### Pitfall 2: `norm_box` coordinate order mismatch

**What goes wrong:** Boxes are converted in the wrong order and highlights drift or invert.

**Why it happens:** Volcengine exposes `x0`, `y0`, `x1`, `y1`, while SlideTutor expects `[top, left, bottom, right]` on a `0..1000` scale.

**How to avoid:** Convert to `[y0, x0, y1, x1]`, multiply by `1000`, and round once.

**Warning signs:** The parser returns plausible text, but visual highlights appear in the wrong area of the page.

### Pitfall 3: Treating captions as figures

**What goes wrong:** Figure/table captions stop behaving like textual context and can get absorbed into figure handling in ways that reduce prompt quality.

**Why it happens:** `cap` is a label adjacent to figures/tables, but it is still text content.

**How to avoid:** Map only `image` to `figure`, `table` to `table`, and keep `cap` in `text`.

**Warning signs:** Output has fewer textual blocks than expected around diagrams or tables, and prompt text loses useful caption context.

### Pitfall 4: Base64 size overshoot

**What goes wrong:** Large or high-DPI page images exceed the documented `8 MB` encoded payload limit and the provider starts failing intermittently.

**Why it happens:** The frontend sends page images as data URLs, and the payload expands further after form encoding.

**How to avoid:** Keep the current single-page image path, but verify real sample page sizes before rollout. If oversize pages appear, open a follow-up for TOS upload rather than expanding 05-03.

**Warning signs:** Volcengine returns size or decode errors only on certain pages or documents.

### Pitfall 5: Partial cutover leaves Azure as a second truth

**What goes wrong:** Some parser calls use Volcengine while others still hit Azure or still require Azure env vars.

**Why it happens:** The repo already has partial Volcengine work but still keeps Azure imports and env names in key paths.

**How to avoid:** Treat access-service wiring, legacy `api/generate.ts`, env cleanup, tests, and docs as one cutover task group.

**Warning signs:** Mixed provider IDs in tests, Azure env names still required after the migration, or docs still describe Azure as the platform parser.

## Code Examples

Verified patterns from official sources and current repo contracts:

### Normalize `norm_box` into the current bbox shape

```ts
function normalizeNormBox(normBox: { x0: number; y0: number; x1: number; y1: number }) {
  return [
    Math.round(normBox.y0 * 1000),
    Math.round(normBox.x0 * 1000),
    Math.round(normBox.y1 * 1000),
    Math.round(normBox.x1 * 1000),
  ] as [number, number, number, number];
}
```

Source: official `OCRPdf` field docs plus `SlideTutor-AI/src/types.ts`

### Normalize Volcengine labels to the stable frontend contract

```ts
function normalizeBlockType(label?: string): LayoutBlock['type'] {
  switch ((label ?? '').trim()) {
    case 'table':
      return 'table';
    case 'image':
      return 'figure';
    default:
      return 'text';
  }
}
```

Source: official `OCRPdf` label docs plus current `LayoutBlock` contract

### Make Volcengine the single configured provider

```ts
function getConfiguredService(env: EnvBag) {
  const database = env.PARSER_USAGE_DB as ParserUsageDatabase | undefined;
  const usageHashSecret = readEnvSecret(env, 'USAGE_HASH_SECRET');

  if (!database || !usageHashSecret) {
    return null;
  }

  return createParserAccessService({
    provider: createVolcengineParserProvider(),
    usageStore: createParserUsageStore(database),
    usageHashSecret,
  });
}
```

Source: current seam in `SlideTutor-AI/api/lib/parser/accessService.ts`

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Azure Document Intelligence as the implicit platform parser default | Shared provider seam with Volcengine-backed live path | Phase 05 remaining work in 2026 | Removes Azure as the only implicit parser path without changing downstream parser UX. |
| Parser migration started from zero | Finish the partial Volcengine work already present in repo | Current repo state on 2026-04-06 | Planning should focus on alignment and cleanup, not initial scaffolding. |
| Whole-PDF or object-storage-first parser assumption | Single-page `image_base64` first for current SlideTutor usage | Current frontend path plus official `OCRPdf` docs | Avoids premature TOS upload and payment/storage scope. |
| Provider-specific output shape | Stable `LayoutBlock[]` normalization | Existing frontend and prompt contract | Keeps UI, highlights, prompt text, and teaching logic stable. |

**Deprecated / outdated:**

- `SlideTutor-AI/api/lib/parser/accessService.ts` importing `createAzureParserProvider()` as the configured live provider.
- `SlideTutor-AI/api/generate.ts` importing and calling `performAzureAnalysis(...)` directly.
- `SlideTutor-AI/api/lib/env.ts` and `SlideTutor-AI/src/worker/index.ts` carrying Azure parser env assumptions as active runtime truth.
- `.env.example`, worker tests, and docs that still describe Azure as the current platform parser.

## Open Questions

1. **What exact `data.detail` shape does the live Volcengine tenant return?** What we know: official docs say array, official Python sample handles string. What is unclear: which shape the production tenant returns consistently. Recommendation: keep dual-shape handling in code and confirm with one live smoke request during implementation.

2. **Do any real SlideTutor page images exceed the documented `8 MB` encoded limit?** What we know: the frontend sends single-page JPEG data URLs and typical slide pages are likely smaller. What is unclear: the largest real page payload in the current user set. Recommendation: sample a few representative large PDFs before rollout; if they fit, keep base64-only in Phase 05.

3. **Should Azure parser code be deleted immediately or kept as inert compatibility code?** What we know: Azure must stop being live runtime truth. What is unclear: whether there is any operational rollback requirement. Recommendation: remove it from all live paths in 05-03; delete dead code if there is no explicit rollback need.

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|-------------|-----------|---------|----------|
| Node.js | Local tests, build, and scripts | Yes | `v24.14.0` | - |
| npm | Package scripts and targeted test runs | Yes | `11.9.0` | - |
| Volcengine access key / secret | Live parser smoke tests and deployment | Not verified | - | Mocked tests can proceed; live cutover cannot be validated without them. |
| Cloudflare D1 binding + `USAGE_HASH_SECRET` | Existing parser quota truth | Partially verified in repo | `PARSER_USAGE_DB` binding is present in `wrangler.jsonc`; live env not verified | Unit and worker tests cover logic, but deployment still needs real binding/secret verification. |
| TOS / object storage | Large URL-based OCR uploads | Not required for recommended scope | - | Stay on `image_base64`. |
| `@volcengine/openapi` | Optional Node-only SDK path | Not required | `1.36.1` available on npm | Use raw signed `fetch`. |

**Missing dependencies with no fallback:**

- Live Volcengine credentials for end-to-end deployment smoke tests.

**Missing dependencies with fallback:**

- TOS upload infrastructure is not needed while `image_base64` stays within documented limits.

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | `vitest` `^4.1.0` plus `@cloudflare/vitest-pool-workers` `^0.14.1` |
| Config file | `SlideTutor-AI/vite.config.ts`; `SlideTutor-AI/vitest.worker.config.ts` |
| Quick run command | `npm test -- api/lib/parser/volcengineProvider.test.ts api/parserAccess.test.ts api/security.test.ts` |
| Full suite command | `npm test && npm run test:workers && npm run lint` |

### Phase Requirements -> Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| PARSE-01 | Platform-managed `Document Parsing` still works with low setup friction after the provider swap | unit + worker | `npm test -- api/lib/parser/volcengineProvider.test.ts api/parserAccess.test.ts && npm run test:workers -- test/workers/parse-route.worker.test.ts` | Yes |
| PARSE-02 | Parser usage remains observable and successful-only | unit | `npm test -- api/parserUsage.test.ts api/parserAccess.test.ts` | Yes |
| PARSE-03 | Azure is no longer the only implicit live parser path | unit + route + static regression check | `npm test -- api/parserAccess.test.ts api/security.test.ts && npm run test:workers -- test/workers/parse-route.worker.test.ts test/workers/security-observability.worker.test.ts && rg -n "createAzureParserProvider|performAzureAnalysis|AZURE_DOCUMENT_INTELLIGENCE" api/lib/parser/accessService.ts api/generate.ts api/lib/env.ts src/worker/index.ts .env.example docs` | Partial - see Wave 0 |
| PARSE-04 | Non-BYOK users still hit explicit platform-managed parsing limits and degraded behavior | unit + worker | `npm test -- api/parserUsage.test.ts api/parserAccess.test.ts && npm run test:workers -- test/workers/parse-route.worker.test.ts test/workers/security-observability.worker.test.ts` | Yes |

### Sampling Rate

- **Per task commit:** `npm test -- api/lib/parser/volcengineProvider.test.ts api/parserAccess.test.ts api/security.test.ts`
- **Per wave merge:** `npm test && npm run test:workers`
- **Phase gate:** `npm test && npm run test:workers && npm run lint`

### Wave 0 Gaps

- [ ] `SlideTutor-AI/api/lib/parser/volcengineProvider.test.ts` should add coverage for `data.detail` arriving as a JSON string as well as an array.
- [ ] `SlideTutor-AI/api/lib/parser/volcengineProvider.test.ts` and `SlideTutor-AI/api/lib/parser/volcengineProvider.ts` should align on the research recommendation that `cap` stays `text`, not `figure`.
- [ ] Add a direct regression that `SlideTutor-AI/api/lib/parser/accessService.ts` no longer imports `createAzureParserProvider()` and that `SlideTutor-AI/api/generate.ts` no longer imports or calls `performAzureAnalysis(...)`.
- [ ] Update `SlideTutor-AI/test/workers/security-observability.worker.test.ts` so parser-unavailable coverage no longer describes missing Azure config as the live failure mode.

## Sources

### Primary (HIGH confidence)

- Official Volcengine OCRPdf docs: `https://www.volcengine.com/docs/86081/1804817?lang=zh`
  - Checked request method, endpoint, auth model, query/body parameters, response fields, labels, and documented size limits.
- Official Volcengine document-parse overview / pricing docs: `https://www.volcengine.com/docs/86081/1804813`
  - Checked product-level quota and pricing context.
- Official Volcengine SDK overview: `https://www.volcengine.com/docs/6369/156029`
  - Checked current SDK guidance and Node SDK availability.
- Official Volcengine signing references: `https://www.volcengine.com/docs/6369/67268` and `https://www.volcengine.com/docs/6369/67269`
  - Checked required headers and signature model for raw signed requests.
- Local code and planning anchors:
  - `SlideTutor-AI/api/lib/parser/accessService.ts`
  - `SlideTutor-AI/api/lib/parser/azureProvider.ts`
  - `SlideTutor-AI/api/lib/parser/volcengineProvider.ts`
  - `SlideTutor-AI/api/lib/azureParse.ts`
  - `SlideTutor-AI/api/generate.ts`
  - `SlideTutor-AI/api/lib/generateService.ts`
  - `SlideTutor-AI/api/lib/env.ts`
  - `SlideTutor-AI/src/worker/routes/parse.ts`
  - `SlideTutor-AI/src/worker/index.ts`
  - `SlideTutor-AI/src/types.ts`
  - `SlideTutor-AI/api/lib/layout.ts`
  - `SlideTutor-AI/src/lib/pdf/layoutUtils.ts`
  - `SlideTutor-AI/src/components/PdfViewer.tsx`
  - `SlideTutor-AI/api/parserAccess.test.ts`
  - `SlideTutor-AI/api/parserUsage.test.ts`
  - `SlideTutor-AI/api/security.test.ts`
  - `SlideTutor-AI/test/workers/parse-route.worker.test.ts`
  - `SlideTutor-AI/test/workers/security-observability.worker.test.ts`
  - `.planning/phases/05-parser-bootstrap-and-provider-abstraction/05-CONTEXT.md`
  - `.planning/REQUIREMENTS.md`
  - `.planning/STATE.md`
  - `docs/discuss/phases/05-parser-bootstrap-and-provider-abstraction-brief.md`

### Secondary (MEDIUM confidence)

- npm registry metadata for `@volcengine/openapi` via `npm view @volcengine/openapi version time.modified description repository.url --json`

### Tertiary (LOW confidence)

- None.

## Metadata

**Confidence breakdown:**

- Standard stack: MEDIUM-HIGH - official Volcengine docs and current repo state agree on the API surface; SDK alternative remains less certain in Worker context.
- Architecture: HIGH - based on the current code seams and the verified Azure anchors that still remain live.
- Pitfalls: HIGH - the key risks are visible directly in the repo and in the official API field docs.

**Research date:** 2026-04-06
**Valid until:** 2026-05-06
