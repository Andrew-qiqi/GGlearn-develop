# 数据流

本文档描述 SlideTutor 项目的数据流和持久化策略。

最后更新：2026-04-03

---

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

- Azure layout analysis still runs only during `explain`
- `distill` does not send the slide image again
- `distill` consumes `fullExplanation` and returns both downstream artifacts in one request

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
6. 后端执行 Azure Document Intelligence 布局分析 (api/generate.ts)
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
