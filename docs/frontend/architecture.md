# 前端架构

本文档描述 SlideTutor 项目的前端架构、状态管理和 API 客户端设计。

最后更新：2026-04-03

---

## 2026-04-04 Phase 04 BYOK-First Access Layer

The frontend now separates `model selection` from `model access`.

### Runtime split

- `selectedModel` answers "which provider family / model should this task use?"
- `aiAccess` answers "how does the app reach that provider right now?"

This separation matters because the next hosted-access phase should reuse the model selection path without forcing another UI reset.

### Current BYOK shape

- `gemini` uses a local API key field
- `openai-compatible` uses one shared credential shape:
  - endpoint preset or custom base URL
  - API key
  - model id

Legacy `qwen` / `doubao` selections are normalized into the `openai-compatible` family during store initialization so older local settings do not break.

### Persistence boundary

BYOK credentials are persisted locally through the existing IndexedDB `appSettings` store.

- no account sync
- no hosted vault
- no parser credential storage

That boundary is intentional for Phase 04: parsing remains platform-funded while model access becomes user-configurable.

## 2026-04-03 Provider-Native Structured Output Layer

The analysis stack now has a provider-aware structured-output adapter instead of relying only on prompt wording.

## 2026-04-03 Phase 2 Artifact-First Runtime

Phase 2 is now complete. The app no longer treats projected markdown or summary strings as live runtime state.

### Runtime authority

Each analyzed page is now driven by two structured artifacts only:

- `explanationArtifact`
- `distillArtifact`

These are the only runtime sources for:

- tutor card rendering
- quick explain rendering
- follow-up targeting
- chunk regeneration
- cross-page context handoff
- persisted page-state writes

### Legacy bridge removal

The old compatibility fields are no longer part of the normal runtime contract:

- `explanation`
- `cheatSheet`
- `summary`

They are not continuously projected anymore, and new saves do not write them back to IndexedDB.

### Legacy recovery boundary

Backward compatibility now exists only at the persistence recovery boundary:

- older saved page states may still contain `explanation`
- older saved page states may still contain `cheatSheet` / `summary`
- `usePdfLibrary.ts` performs a one-time legacy-to-artifact normalization when those records are loaded

After recovery, the in-memory page state is normalized back into artifact-first shape and legacy fields are dropped.

### Provider split

The current architecture intentionally treats providers in two families:

- `gemini`
- `openai-compatible`

`Gemini` stays on its own adapter because it uses native `responseJsonSchema` and `thinkingConfig`.

All current and future OpenAI-compatible providers share one structured-output adapter layer. Today that includes:

- `qwen`
- `doubao`
- future user-supplied OpenAI-compatible endpoints

### Task coverage

The native structured-output layer currently covers these tasks:

- `explain`
- `distill`
- `generate_questions`
- `evaluate_answers`

This means the system now treats these tasks as schema-first generation, not "best-effort JSON by prompt convention".

### Task-level thinking policy

Thinking intensity is now a task concern, not a provider-specific accident.

The live policy is:

- `distill`: Gemini uses `thinkingLevel = "minimal"` because this is a compact restructuring task and high thinking was wasting token budget
- `explain`: structured output is enabled, but thinking is not force-reduced yet because card splitting, visual intent, and teacher-style explanation still need quality monitoring
- `generate_questions` / `evaluate_answers`: structured output is enabled, but thinking stays conservative until quality baselines are re-evaluated across providers

## 2026-04-03 Structured Explain / Distill Artifact Contract

This note records the current frontend state contract after migrating the main analysis pipeline to structured JSON outputs.

### Page-state authority

Each analyzed page now keeps two structured artifacts as the primary machine-readable state:

- `explanationArtifact`
- `distillArtifact`

The current shapes are:

- `explanationArtifact.version = "explain_v1"`
- `explanationArtifact.introCard.body`
- `explanationArtifact.knowledgeCards[] = { title, intent, body, socraticProbe }`
- `distillArtifact.version = "distill_v1"`
- `distillArtifact.quickExplain.body`
- `distillArtifact.contextMemory = { established, progress, bridge, avoidRepeat }`

### Intro / knowledge card boundary

The intro card title is no longer model-authored. The system always renders it as:

- `This Slide at a Glance`

The model only fills `introCard.body`. Knowledge cards continue to own:

- concept title
- highlight intent box
- teaching body
- optional Socratic probe string

### Frontend consumption

The tutor UI now consumes only structured artifacts:

- `CanvasTutor` renders explanation cards from `explanationArtifact`
- focus mode prefers `distillArtifact.quickExplain.body`
- follow-up target chunk selection resolves from artifact chunk order instead of splitting `###` blocks
- chunk regeneration writes back into the matching `knowledgeCards[index]` entry from a structured single-card response
- note editing and quiz context assembly also read artifact fields directly

Legacy markdown parsing remains only for old saved-state recovery and is not used by runtime consumers anymore.

## 2026-03-29 Context Continuity and Teaching Artifacts

This note documents the current frontend contract after the context / quick-explain overhaul.

### Artifact split

Each analyzed slide now stores three distinct AI artifacts in page state:

- `explanation`: the main teaching content shown as tutor cards
- `cheatSheet`: the stored field for the product-facing `Quick Explain` / `速通讲解` artifact used in focus mode and urgent review
- `summary`: now used as structured `Context Memory` for the next slide, not as a student-facing summary paragraph

This separation is intentional. `cheatSheet` (`Quick Explain`) is not a backup version of `explanation`, and `summary` is not meant for direct reading by the student. They serve different jobs in the system.

### Explain contract

The `explanation` prompt now requires a mandatory first card. That first card acts as the slide intro / navigation card:

- it tells the learner what this slide is about
- it can acknowledge continuity from the previous slide in one short sentence
- it should orient the learner before the detailed cards begin

After the intro card, the remaining cards cover the actual knowledge units on the page.

The explanation format also has two parser-facing constraints now:

- knowledge cards must emit `>>>Intent: [ymin, xmin, ymax, xmax]` using integer `0..1000` coordinates
- `###` is reserved for card title lines only, so card bodies must not introduce additional `###` headings

### Context Memory contract

The value still stored in `summary` is now semantically treated as `Context Memory`. It is a short handoff for the next page and is designed to be readable by both humans and the model.

The target structure is:

- `Established`
- `Progress`
- `Bridge`
- `Avoid Repeat`

Labels can be omitted when they are not useful. The goal is continuity, not completeness.

### UI consumption

The UI no longer extracts cheat sheet text from the explanation payload. Instead:

- tutor cards render from `explanation`
- focus mode reads `currentPageState.cheatSheet`
- follow-up chunk parsing reads only explanation card boundaries
- the intro card `### This Slide at a Glance` is not treated as a regenerate-able knowledge card in the chunk UI

This removes the old coupling where non-explanation content was embedded inside the explanation stream.

For presentation only, Focus Mode may lightly re-paragraph a single-block `cheatSheet` string before rendering it as `Quick Explain`. This formatting step is intentionally view-only: it does not change the stored page state, the distill output contract, or the prompt structure.

### Generation strategy update

The artifact split still exists at the page-state level, but the generation pipeline is now intentionally two-stage instead of three independent slide-generation requests:

1. `explain` generates the main teaching artifact from the slide image
2. `distill` generates both `cheatSheet` and `Context Memory` from the finished explanation text

This keeps `explanation` visually grounded while avoiding a second image-based request just to produce `Quick Explain`. In practice, `cheatSheet` is now a text-only distillation product derived from the explanation rather than a second visual interpretation pass.

## 2026-03-31 Note Interaction Contract

The note system now has two frontend note modes per page:

- `spatialNotes`: free-position notes created on the PDF surface
- `notes`: explanation-attached notes grouped by explanation chunk index

### PDF-side drag contract

PDF panning and note dragging share the same visual surface, so their event boundaries must stay explicit:

- dragging a `.spatial-note` must not start the PDF canvas pan interaction
- the PDF pan container should only activate from neutral page content, not from note UI, buttons, or textareas
- note drag should stay local to the note item even when the PDF is zoomed and scroll-pannable

### Tutor-side reattach contract

Explanation notes can be dragged between chunk containers, but drop-target resolution must ignore the dragged note itself. Because the dragged note still belongs to its source chunk in the DOM tree during drag, using the first `elementsFromPoint()` match can incorrectly resolve back to the source chunk.

The live contract is:

- note cards expose a stable `data-note-id`
- chunk wrappers expose `data-chunk-index`
- drop resolution scans hit-tested elements, skips the dragged note subtree, and then finds the first valid chunk target
- dragged explanation notes use tight drag settings (`dragMomentum={false}`, `dragElastic={0}`) to keep pointer tracking predictable

## 2026-04-01 Tutor Card Action Panel Motion Contract

Tutor cards expose small action drawers for `follow-up`, `add note`, and `regenerate`. These drawers live inside explanation cards, so their animation has to cooperate with both the card itself and the surrounding chunk layout.

### State contract

The live interaction now follows a two-phase close model:

- opening a drawer sets the active panel and requests textarea focus
- closing a drawer hides it immediately, but does not clear the draft synchronously
- draft text is cleared only after `AnimatePresence` reports that the exit animation is complete

This prevents the exit animation from changing both panel visibility and textarea content height in the same frame.

Textarea focus is intentionally decoupled from animation completion:

- focus is requested on the next animation frame after mount, not on the final animation callback
- the focus call uses `preventScroll` when supported, with a fallback for older browsers

This avoids a last-frame scroll/reflow hitch when the browser tries to bring the textarea caret into view.

### Motion contract

The action panel no longer relies on a spring-based `height: auto` transition. Instead, it uses a short tween animation for:

- `height`
- `opacity`
- `marginTop`

This keeps the drawer visually soft without introducing the end-of-animation hitch that came from repeated spring re-measurement.

### Layout contract

Explanation chunk wrappers now use position-only layout animation while the action panel opens or closes. That means:

- chunk rows can still shift smoothly when cards grow or shrink
- the parent chunk wrapper no longer interpolates full size layout at the same time as the nested drawer
- nested drawer animation and outer list reflow are less likely to fight each other


## 前端架构

### 状态管理

**Zustand Stores：**

1. **pdfStore.ts** - PDF 文档状态
   - 当前 PDF 文件
   - 页面元数据
   - 缩放和导航状态

2. **tutorStore.ts** - 辅导会话状态
   - 每页的 AI 解释
   - 追问历史
   - 测验数据
   - 生成状态（加载、错误）

3. **uiStore.ts** - UI 状态
   - 侧边栏可见性
   - 活动标签
   - 模态框状态

### 自定义 Hooks

**核心业务逻辑 Hooks：**

1. **useSlideAnalysis.ts**
   - 触发幻灯片分析
   - 处理流式响应
   - 更新 tutorStore
   - 持久化到 IndexedDB

2. **useFollowUp.ts**
   - 管理追问对话
   - 维护对话历史
   - 上下文感知提问

3. **useQuiz.ts**
   - 生成测验题目
   - 评估用户答案
   - 提供反馈和解释

4. **useChunkRegenerate.ts**
   - 重新生成特定内容块
   - 保留其他内容不变

### API 客户端

**apiClient.ts** - 统一的 API 调用封装

**功能：**
- Token 自动管理（获取、缓存、刷新）
- 401 错误自动重试
- 统一错误处理
- 类型安全的请求/响应

**使用示例：**
```typescript
import { apiGenerate } from '@/lib/api/apiClient';

const response = await apiGenerate({
  task: 'explain',
  pageNumber: 1,
  imageBase64: '...',
  textContent: '...'
});
```

### 主题管理 (Theme Management)

SlideTutor 提供多主题支持，包括浅色 (Light)、护眼 (Eyecare)、暮色禅意 (Twilight Zen) 和春日草甸 (Spring Meadow) 模式。

**实现机制：**
- **全局状态**：主题状态存储在 `uiStore.ts` (Zustand) 中，使用 `Theme` 类型。
- **持久化**：通过 `src/lib/db.ts` 的 `setSetting` 和 `getSetting` 存储在 IndexedDB 中。
- **即时应用**：应用启动时在 `useUiStore.init()` 中异步读取主题，并立即应用到 `document.documentElement` 类名。
- **组件同步**：`ThemeToggle` 组件通过全局 Store 获取和更新主题，不再依赖本地状态。
- **PWA 元数据同步**：通过 `updateMetaThemeColor` 和 `index.html` 中的早期注入脚本，根据当前主题实时更新 `<meta name="theme-color">`，确保 PWA 独立窗口的标题栏颜色与应用导航栏和整体氛围融为一体，且避免在 React 启动前出现颜色闪烁。
- **暮色禅意 (Twilight Zen) 优化 (2026-04-01)**：
  - **设计 DNA**：采用了中调暮色 (`#233755`, `#2C456A`) 作为基础，通过低对比度的雾化背景 (`#DBAEC8` 暖雾粉, `#9D9DD4` 雾紫) 营造“治愈感”。
  - **阅读舒适度**：为解决深色模式下亮白文字刺眼的问题，将 tutor-card 和 note-card 的长文正文颜色单独降至 cool mist gray-blue (`#B8C9E1`)。
  - **语义保护策略**：采用“显式覆盖 + 显式恢复”策略，确保 `Thinking Prompt` 等共享产品语义不被主题色“误伤”，保持全局视觉一致性。

**PDF 渲染同步：**
- `PdfViewer.tsx` 使用 `MutationObserver` 监听根元素的类名变化，根据 `eyecare` 等类名的存在与否动态调整 PDF 渲染层（如混合模式），确保 UI 与 PDF 内容风格统一。

### 持久化策略

**IndexedDB 存储：**

```typescript
// 数据库结构
{
  pdfs: {
    id: string,
    name: string,
    file: Blob,
    uploadDate: Date
  },
  pageStates: {
    pdfId: string,
    pageNumber: number,
    explanation: string,
    followUps: Array,
    quiz: Object,
    layoutBlocks: Array
  }
}
```

**同步策略：**
- 每次 AI 响应完成后自动保存
- 页面加载时从 IndexedDB 恢复状态
- 支持离线访问已分析的内容

---

## 数据流
## 2026-04-01 Theme Visual Consistency Contract

This note records the current theme-layer contract after the twilight / meadow visual cleanup pass.

### Shared semantic layer

The app now treats several recurring visual cues as cross-theme semantics rather than per-theme decoration:

- explanation highlight overlays
- `Thinking Prompt` accent box and label color
- header accent controls such as the active library toggle, product badge, and upload/change-PDF button

These elements should read as the same product system in every theme, even when the surrounding atmosphere changes.

### Theme individuality boundary

`twilight-zen` and `spring-meadow` still keep their own:

- page background gradients
- glass-panel material treatment
- theme-specific surface opacity and shadow tuning

They should not introduce separate typography systems or unrelated accent colors for shared teaching UI unless the change is required for accessibility.

### Typography contract

Global typography remains driven by the shared theme fonts (`--font-sans`, `--font-display`, `--font-serif`). Theme styles may tune contrast, opacity, or surface presentation, but should not silently fork the main font family or heading personality for one theme only.

### Highlight contract

PDF explanation highlights now rely on shared semantic tokens instead of theme-specific one-off CSS blocks. The current contract is intentionally stricter than "theme-aware highlight styling": every theme uses the same borderless, fill-only highlight treatment so learners do not need to re-interpret the meaning of a highlighted region after switching themes.

### Header contrast contract

Header accent controls now use dedicated semantic classes instead of relying only on generic `bg-text-primary text-bg-elevated` combinations. These controls are product-level accents, not theme-local decoration, so they should keep one stable shared treatment across themes. This isolates high-contrast accent controls from broad theme-level icon overrides and prevents regressions like the `spring-meadow` dark-block header bug where icons and labels became difficult to read on dark accent surfaces.
