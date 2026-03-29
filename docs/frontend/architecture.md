# 前端架构

本文档描述 SlideTutor 项目的前端架构、状态管理和 API 客户端设计。

最后更新：2026-03-29

---

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

This removes the old coupling where non-explanation content was embedded inside the explanation stream.

For presentation only, Focus Mode may lightly re-paragraph a single-block `cheatSheet` string before rendering it as `Quick Explain`. This formatting step is intentionally view-only: it does not change the stored page state, the distill output contract, or the prompt structure.

### Generation strategy update

The artifact split still exists at the page-state level, but the generation pipeline is now intentionally two-stage instead of three independent slide-generation requests:

1. `explain` generates the main teaching artifact from the slide image
2. `distill` generates both `cheatSheet` and `Context Memory` from the finished explanation text

This keeps `explanation` visually grounded while avoiding a second image-based request just to produce `Quick Explain`. In practice, `cheatSheet` is now a text-only distillation product derived from the explanation rather than a second visual interpretation pass.


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

SlideTutor 提供多主题支持，包括浅色 (Light)、护眼 (Eyecare)、晨雾 (Morning Mist) 和雨天 (Rainy) 模式。

**实现机制：**
- **全局状态**：主题状态存储在 `uiStore.ts` (Zustand) 中，使用 `Theme` 类型。
- **持久化**：通过 `src/lib/db.ts` 的 `setSetting` 和 `getSetting` 存储在 IndexedDB 中。
- **即时应用**：应用启动时在 `useUiStore.init()` 中异步读取主题，并立即应用到 `document.documentElement` 类名。
- **组件同步**：`ThemeToggle` 组件通过全局 Store 获取和更新主题，不再依赖本地状态。

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
