# 数据流

本文档描述 SlideTutor 项目的数据流和持久化策略。

最后更新：2026-04-14

---

## 2026-04-14 Touch-Device Slide Extraction Hardening

The frontend slide-image extraction path now resolves a device profile before `PdfViewer.extractPageData(...)` renders a temporary analysis canvas.

### Runtime behavior

- desktop-class devices still prefer higher extraction scales for clearer slide snapshots
- touch / tablet-class devices now use a constrained extraction profile with lower retry scales and a stricter canvas pixel budget
- every failed extraction attempt now releases both the temporary canvas and the underlying `pdf.js page` resources before retrying
- `Analyze`, follow-up image reuse, and quiz generation all continue to share this same extraction entrypoint

### Why this matters

Previously, tablet-class devices could fail the first extraction attempt and then carry unreleased `pdf.js` page resources into the fallback attempt. On memory-constrained WebKit and other touch-device browsers, that made the lower-resolution retry much more likely to fail as well, surfacing the frontend error `Failed to extract slide image.` before the request ever reached backend analysis.

## 2026-04-05 Hosted Analyze and Credits Flow

Phase 06 adds a hosted credits path without changing the mature two-step analysis UI contract.

### Hosted `Analyze`

For `Platform API`, the runtime flow is now:

1. `useSlideAnalysis.ts` calls `/api/generate` with `task = explain` and `access.mode = "platform"`
2. Worker verifies Clerk auth from the bearer token
3. backend parser access runs first
4. if the platform parser is rate limited or unavailable, the request fails early with `PLATFORM_PARSER_RATE_LIMITED` or `PLATFORM_PARSER_UNAVAILABLE`
5. if parser access is healthy and credits are sufficient, the backend creates a pending analyze attempt and returns `x-slidetutor-analyze-attempt-id`
6. frontend stores that attempt id only in memory for the current analyze action
7. frontend calls `/api/generate` again with `task = distill` and `taskData.hostedAnalyzeAttemptId`
8. only after successful distill stream completion does the Worker commit the single `Analyze = 3` ledger deduction

This keeps the learner-facing `Analyze` action whole while still preserving success-only billing.

### Hosted `Follow-up` and quiz actions

For `followup`, `generate_questions`, and `evaluate_answers`:

1. frontend sends `access.mode = "platform"`
2. Worker verifies Clerk auth
3. backend preflights credits before generation starts
4. stream runs normally
5. credits are deducted only after the stream completes successfully

If credits are insufficient, the Worker returns structured JSON before any model execution.

### Hosted regenerate actions

Hosted mode now supports the same regenerate tasks as `My API`:

- `regenerate_chunk`
- `regenerate_followup`

Both tasks map to one hosted action:

- `card_regenerate = 1 credit`

The frontend no longer redirects these actions back to AI settings just because the user is in `Platform API`. The only remaining hosted guardrails here are auth and insufficient-credit handling.

### Recharge flow

1. user opens `Buy Credits` from settings or from the insufficient-credit dialog
2. frontend previews the fixed quote using `1 RMB = 30 credits`
3. frontend calls `POST /api/recharge-intent`
4. Worker creates a recharge order and returns a ZPAY `submit.php` checkout URL
5. frontend opens that checkout URL in a new tab
6. ZPAY calls `/api/payment-webhook` and the Worker replies with plain-text `success`
7. backend verifies the ZPAY signature and amount, then applies the credit ledger entry exactly once even if the webhook is replayed

## 2026-04-09 Final Parser Ownership Flow

Phase 08 finishes the parser split and removes the old quota-shaped product behavior.

### Parser ownership

- `Platform API` uses the platform-managed `Volcengine` parser.
- `Platform API` users do not configure parser providers in settings.
- `My API` may optionally configure `LlamaParse`.
- if `My API` omits parser config, `explain` still runs through the existing no-parser degraded analysis path.

### Explain flow

For `task = explain`, the runtime now behaves like this:

1. frontend calls `/api/generate`
2. if `access.mode = "platform"`, backend resolves the platform-managed Volcengine parser first
3. if Volcengine succeeds, backend injects normalized `layoutBlocks`
4. if Volcengine is rate limited or unavailable, hosted analyze fails early with `PLATFORM_PARSER_RATE_LIMITED` or `PLATFORM_PARSER_UNAVAILABLE`
5. if `access.mode = "byok"` and parser config is absent, explain continues without parser blocks and stays on the no-parser degraded path
6. if `access.mode = "byok"` and parser config is present, backend calls `LlamaParse`
7. if `LlamaParse` succeeds, backend injects normalized `layoutBlocks`
8. if `LlamaParse` fails or times out, backend returns `BYOK_PARSER_FAILED` or `BYOK_PARSER_TIMEOUT`
9. Worker still returns `x-slidetutor-parse-mode` only when a parser path was actually attempted
10. frontend stores `analysisAccuracy` on the current page state and only shows `Low accuracy` when the analysis actually degraded

### Explain bbox grounding behavior

The `explain` prompt now has two bbox-grounding modes:

- when normalized `layoutBlocks` are present, the prompt injects them as `Current Slide Parsed Regions`
- in parsed mode, `knowledgeCards[].intent` must stay traceable to those parsed regions instead of drifting freely by vision alone
- parsed-mode boxes may be:
  - exactly one parsed region
  - one minimal enclosing rectangle over spatially adjacent parsed regions
  - a small local refinement of one parsed region / parsed-region group
  - a crop inside an overly large parsed region
- parsed-mode boxes may touch at boundaries, but their areas must not overlap
- if no parser blocks are available, the existing no-parser degraded path stays unchanged and the model falls back to its own visual judgment

### Error taxonomy

- `ROUTE_RATE_LIMITED`: Worker route throttle before generation starts
- `PLATFORM_PARSER_RATE_LIMITED`: Volcengine upstream throttled
- `PLATFORM_PARSER_UNAVAILABLE`: platform parser unavailable for other reasons
- `BYOK_PARSER_FAILED`: `LlamaParse` failed on the `My API` path
- `BYOK_PARSER_TIMEOUT`: `LlamaParse` exceeded the bounded poll budget

## 2026-04-04 BYOK Access Metadata Flow

Generation requests now carry normalized access metadata from the frontend API client.

### Request assembly

`apiClient.ts` still owns token fetching and retry behavior, but it now also reads persisted `aiAccess` from `uiStore` and attaches an `access` object when local BYOK configuration is complete.

### Access rules

- `gemini` sends `{ mode: "byok", providerId: "gemini", apiKey }` when a local key exists
- `openai-compatible` sends `{ mode: "byok", providerId: "openai-compatible", apiKey, baseURL, endpointPreset }` when both key and base URL exist
- incomplete BYOK settings do not fabricate partial access payloads

### Why this matters

This keeps teaching hooks unchanged:

- `useSlideAnalysis.ts`
- `useFollowUp.ts`
- `useChunkRegenerate.ts`
- `useQuiz.ts`

They still call one shared `apiGenerate(...)` helper, while provider access policy stays centralized.

## 2026-04-03 Provider Structured-Output Routing

Structured JSON is no longer only a prompt convention. The backend now routes schema-enabled tasks through provider-native configuration.

## 2026-04-03 Phase 2 Artifact-First Data Flow

The frontend data flow is now fully artifact-first.

## 2026-04-14 Tutor Math Delimiter Compatibility Flow

Formula compatibility is now handled at the final presentation boundary, not in artifact parsing or persistence.

### Runtime rule

For tutor-facing rich text:

1. the model response is parsed into structured artifacts exactly as received
2. artifact `body` strings are stored unchanged in page state
3. right before UI markdown parsing, the shared markdown renderer normalizes `\(...\)` to `$...$`
4. right before UI markdown parsing, the same renderer normalizes `\[...\]` to `$$...$$`
5. `remark-math` and KaTeX then render the normalized string

### Why the boundary is here

This placement preserves two important invariants:

- structured-output parsing remains about schema validity, not display repair
- saved artifacts stay faithful to the original model output instead of being silently rewritten on write

### Prompt contract

The generation prompts now also explicitly require:

- `$...$` for inline math
- `$$...$$` for display math
- no `\(...\)` or `\[...\]`

That means the UI boundary is a compatibility backstop, not the primary contract.

### Live page-state writes

`useSlideAnalysis.ts` now writes:

- `explanationArtifact` during explain streaming
- `distillArtifact` after distill completion

It no longer projects runtime markdown or summary strings back into page state.

### Downstream consumers

The following consumers now read artifacts directly instead of compatibility strings:

- `CanvasTutor.tsx`
- `useFollowUp.ts`
- `useChunkRegenerate.ts`
- `PdfViewer.tsx`
- `useQuiz.ts`
- global analysis completion checks

### Persistence behavior

IndexedDB page-state updates now persist artifact-first page state only.

- new saves do not continuously write legacy `explanation` / `cheatSheet` / `summary`
- older records are normalized on load
- legacy parsing exists only inside the recovery path

### Current routing model

There are now two provider families in the generation path:

- `gemini`
- `openai-compatible`

`Gemini` uses:

- `responseMimeType`
- `responseJsonSchema`
- task-level `thinkingConfig`

OpenAI-compatible providers use:

- `response_format.type = "json_schema"`
- a shared schema adapter for every supported task

This keeps provider-specific transport details on the backend while giving the frontend one stable contract.

### Structured task set

The current structured-output task set is:

- `explain`
- `distill`
- `regenerate_chunk`
- `generate_questions`
- `evaluate_answers`

Non-structured conversational tasks still flow as plain streamed text:

- `followup`
- `regenerate_followup`

## 2026-04-03 Structured JSON Analysis Flow

This section records the current analysis flow after migrating `explain` and `distill` to structured JSON contracts.

### Explain stage

`useSlideAnalysis.ts` still streams the `explain` response, but it no longer treats the payload as display markdown. Instead:

1. the frontend accumulates a raw JSON buffer
2. it extracts only fully closed units from that buffer
3. the smallest live render unit is now a complete card object

The current live units are:

- `introCard`
- one fully closed `knowledgeCards[i]`

This means the learner no longer sees half-written protocol markers. A card appears only after its JSON object is complete enough to validate.

### Distill stage

`distill` is now parsed as one strict JSON object after the full response arrives. The stored artifact contains:

- `quickExplain.body`
- `contextMemory.established`
- `contextMemory.progress`
- `contextMemory.bridge`
- `contextMemory.avoidRepeat`

`progress` is treated as mandatory in the parser-facing contract. The other fields stay present but may be empty strings.

### Persistence flow

Per-page persistence now stores structured artifacts as the primary saved contract:

- `explanationArtifact`
- `distillArtifact`

Old compatibility strings are no longer re-derived on write. Only old records loaded from IndexedDB may still be converted forward during recovery.

## 2026-03-29 Analysis Pipeline Update

This section records the current slide-analysis flow after the context continuity fix and the Quick Explain split.

### Current analysis order

For each page, `useSlideAnalysis.ts` now runs a two-stage pipeline:

1. `explain`
2. `distill`

The second stage is text-only and produces both:

- `cheatSheet` (product-facing `Quick Explain` / `速通讲解`)
- `summary` (used as `Context Memory`)

The ordering matters:

- `explanation` remains the primary visually grounded learning artifact
- `distill` derives the `Quick Explain` artifact and next-page handoff from the completed explanation
- this avoids a second image-based generation request for the same slide

### Continuity fix for auto analysis

Auto sequential analysis previously had a stale-state bug: page `N + 1` could read an older closure value and miss the freshly generated `summary` from page `N`.

`useSlideAnalysis.ts` now reads the latest page state directly from `useTutorStore.getState()` when starting a page analysis. This means the next page can consume the newest available `Context Memory` instead of an outdated snapshot.

### Storage and rendering flow

The per-page state contract now looks like this in practice:

- `explanation` -> rendered as explain cards
- `cheatSheet` -> rendered in focus mode as `Quick Explain` / `速通讲解`
- `summary` -> stored for next-page context injection

The frontend no longer parses `>>>CheatSheet` blocks out of the explanation text. That parsing path was removed so each artifact has one clear producer and one clear consumer.

In the current UI, `Quick Explain` may pass through a lightweight presentation formatter before render. This formatter only splits a dense single paragraph into 2 to 3 readable paragraphs when needed. It does not write back to page state and does not alter the upstream `distill` response.

### Resource reuse note

The current design does not cache `layoutBlocks` for low-frequency regenerate scenarios. Instead, it optimizes the high-frequency path:

- platform layout analysis still runs only during `explain`
- `distill` does not send the slide image again
- `distill` consumes `fullExplanation` and returns both downstream artifacts in one request

## 2026-04-10 Phase 09 BYOK capability and distill hardening flow

### BYOK capability-check flow

1. settings saves or updates BYOK access/model state
2. frontend calls `POST /api/model-capability-check`
3. backend returns `usable`, `unusable`, `pending`, or `stale` plus a capability summary snapshot
4. frontend stores that result in `uiStore.modelCapabilityCheck`
5. later BYOK generation requests reuse the saved readiness state unless the selected model/access changed or the saved result was marked `stale`

### Runtime `needs recheck` behavior

Normal BYOK generation does not re-run a live provider probe on every request. Instead:

- clear capability/configuration failures can mark the saved capability status `stale`
- the next eligible BYOK request or settings save triggers a fresh capability check
- generic transient failures and `STRUCTURED_OUTPUT_TRUNCATED` do not automatically invalidate readiness

### Distill input/output hardening

`useSlideAnalysis.ts` still runs `explain -> distill`, but `distill` no longer receives the full prompt-format packaging string.

- `formatExplanationArtifactForDistill(...)` removes packaging-only lines such as `Visual Focus Box` and `Socratic Probe`
- the main explanation artifact and Focus mode rendering still keep the full teaching content
- backend `distill` now surfaces stable structured-output failures instead of silently retrying after truncation

### Follow-up implications

Follow-up logic now splits explanation cards only from the explanation payload. This keeps follow-up question context aligned with the actual teaching cards and avoids mixing in fast-scan content.

Because the chunk pipeline still uses markdown card boundaries, the explanation contract now explicitly reserves `###` for card titles only. This protects chunk parsing for highlighting, follow-up targeting, and chunk regeneration without requiring a structured JSON explanation format.


### PDF 分析流程

```
1. 用户上传 PDF
   ↓
2. 保存到 IndexedDB (src/lib/db.ts)
   ↓
3. useSlideAnalysis.ts 触发分析请求
   ↓
4. PDF 页面转换为图像/文本内容
   ↓
5. 发送请求到 /api/generate
   ↓
6. 后端执行 Volcengine OCRPdf 页面布局分析（shared parser provider）
   ↓
7. 后端根据任务和布局构建提示词
   ↓
8. 后端流式返回 AI 响应（Gemini/OpenAI）
   ↓
9. 前端更新 tutorStore.ts
   ↓
10. 结果持久化到 IndexedDB
```

### 追问对话流程

```
1. 用户输入追问问题
   ↓
2. useFollowUp.ts 收集上下文
   - 当前页面解释
   - 之前的追问历史
   - 幻灯片内容
   ↓
3. 调用 apiGenerate() 发送请求
   ↓
4. 后端生成上下文感知的回答
   ↓
5. 流式响应显示在 UI
   ↓
6. 追问历史更新并保存
```

### 测验生成流程

```
1. 用户点击"生成测验"
   ↓
2. useQuiz.ts 准备幻灯片内容
   ↓
3. 调用 /api/generate (task: 'quiz')
   ↓
4. AI 生成多选题
   ↓
5. 解析 JSON 格式的测验数据
   ↓
6. 显示测验界面
   ↓
7. 用户提交答案
   ↓
8. 本地评分和反馈
```

---
