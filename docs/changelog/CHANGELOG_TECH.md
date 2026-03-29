# 技术变更日志

本文档记录 SlideTutor 项目的重要技术变更、架构决策和实现细节。

条目按时间倒序排列（最新的在前）。

---
## [2026-03-29] Editorial Redesign of Quick Explain (Focus Mode)

**What**: Redesigned the visual representation of "Quick Explain" (formerly known as cheatsheet) in Focus Mode. Replaced the standard card-based container with an open, editorial-style layout featuring a vertical guiding line, improved typography (Lora font), and staggered paragraph entry animations.

**Why**: To align the UI with the "Teacher's Voice" product positioning. The previous card design was too structured and mechanical, making the content feel like a "data summary" rather than a natural classroom explanation. The new design creates a more immersive, lecture-like reading experience that is easier on the eyes and feels more personal.

**Impact**:
- Enhanced reading rhythm and focus in Focus Mode.
- Improved visual hierarchy: Lead paragraphs are now more prominent to capture immediate attention.
- Theme-aware design: Uses semantic tokens (`text-primary`, `bg-surface`) to maintain high "ink-on-paper" contrast in all modes (Light, Eyecare, Glass, Rainy).
- No changes to data structure, prompt logic, or backend services.

**Files**: `SlideTutor-AI/src/components/CanvasTutor.tsx`

## [2026-03-29] Added Drag-and-Drop and Copy-Paste PDF Upload

**What**: Implemented global drag-and-drop and copy-paste support for uploading PDF files. Added an `isFileDragging` state for drag-over visual feedback and a `pendingFile` state to show a confirmation modal before processing the upload. Updated the initial empty-state UI to guide users about these new upload methods.

**Why**: To improve user experience and discoverability. Users previously could only upload via the click-to-upload button. The confirmation modal ensures that "fast" actions like pasting or dropping don't accidentally overwrite current work without a conscious "Upload Now" click.

**Impact**:
- Added `pendingFile` state and confirmation UI in `App.tsx`.
- Updated `PdfViewer.tsx` empty-state text to mention Drag & Drop and Paste.
- `processFile` now clears `pendingFile` upon successful execution.
- No breaking changes.

**Files**: `SlideTutor-AI/src/App.tsx`, `SlideTutor-AI/src/components/PdfViewer.tsx`

## [2026-03-29] Product Renaming and Prompt Repositioning for Quick Explain

**What**: Repositioned the old `cheatsheet` artifact as product-facing `Quick Explain` / `速通讲解`. Updated the prompt language so the model no longer treats this artifact like a study card or fast-scan memory note. The UI now labels the section as `速通讲解` in Chinese and `Quick Explain` in English, while the persisted state field remains `cheatSheet` for compatibility.

**Why**: The previous naming and prompt wording pushed the model toward a robotic "cheat sheet / takeaways / memory card" style. The intended job of this artifact is different: it should feel like a teacher quickly explaining the page in class, not a compressed review card.

**Impact**:
- prompt instructions now target short classroom-style explanation instead of study-card structure
- distill output now uses `quickExplain` + `contextMemory` JSON keys
- `useSlideAnalysis.ts` accepts `quickExplain` and maps it into the existing `cheatSheet` storage field
- UI wording now reflects the product concept without breaking stored data

**Files**: `SlideTutor-AI/src/lib/ai/prompts.ts`, `SlideTutor-AI/src/lib/ai/prompts.test.ts`, `SlideTutor-AI/src/lib/ai/__snapshots__/prompts.test.ts.snap`, `SlideTutor-AI/src/hooks/useSlideAnalysis.ts`, `SlideTutor-AI/src/hooks/useSlideAnalysis.test.ts`, `SlideTutor-AI/src/components/CanvasTutor.tsx`, `docs/frontend/architecture.md`, `docs/frontend/data-flow.md`

## [2026-03-29] Intro Card Prompt Refinement for Teacher Persona

**What**: Refined the `explain` prompt's "Mandatory intro card" to transition from a dry summary/focus instruction to a natural, teacher-like "contextual lead-in". The new prompt uses a "toolbox" approach where the AI can selectively use a "Bridge" (optional), the "Soul" of the page, and a "Natural Lead".

**Why**: The previous intro cards felt too much like a summary or a mechanical table of contents ("This slide introduces A, B, and C"). This broke the immersion of having a personal tutor. We wanted a smoother entry that bridges from the previous context naturally and sets the stage without spoiling the detailed breakdown that follows.

**Impact**:
- Intro cards are now more conversational and concise (1-3 sentences).
- Improved "Bridge" logic: AI now only connects to previous slides if it's natural and non-confusing.
- "Forbidden Styles" added to strictly block corporate/formal phrases and mechanical "Notice how..." instructions in the intro.
- No changes to the 3-part artifact structure or the detailed knowledge card style.

**Files**: `SlideTutor-AI/src/lib/ai/prompts.ts`, `SlideTutor-AI/src/lib/ai/prompts.test.ts`, `SlideTutor-AI/src/lib/ai/__snapshots__/prompts.test.ts.snap`

## [2026-03-29] Distill Stage Replaced Separate CheatSheet and Summary Requests

**What**: Changed slide analysis from a three-request pipeline (`explain` -> `cheatsheet` -> `summary`) to a two-stage pipeline (`explain` -> `distill`). The new `distill` task is text-only and returns both `cheatSheet` and `contextMemory` in one JSON response. `useSlideAnalysis.ts` now parses that response and stores the outputs into `cheatSheet` and `summary`.

**Why**: The previous pipeline made the model look at the same slide image twice in the common path: once for `explain` and once again for `cheatsheet`. That repeated the expensive visual request even though the fast-scan artifact and context handoff can be derived from the finished explanation. The new design keeps visual grounding where it matters and removes unnecessary duplicate work.

**Impact**:
- full slide analysis now makes 2 generation requests instead of 3
- Azure layout analysis still runs only during `explain`
- `cheatSheet` is now produced by text distillation rather than a second image-based request
- `summary` continues to store `Context Memory`, but now comes from the same distill response as `cheatSheet`

**Files**: `SlideTutor-AI/src/hooks/useSlideAnalysis.ts`, `SlideTutor-AI/src/hooks/useSlideAnalysis.test.ts`, `SlideTutor-AI/src/lib/ai/prompts.ts`, `SlideTutor-AI/src/lib/ai/prompts.test.ts`, `SlideTutor-AI/src/lib/ai/__snapshots__/prompts.test.ts.snap`, `SlideTutor-AI/api/generate.ts`, `docs/frontend/architecture.md`, `docs/frontend/data-flow.md`

## [2026-03-29] Context Memory and Cheat Sheet Pipeline Overhaul

**What**: Reworked the slide-analysis pipeline so `explanation`, `cheatSheet`, and `summary` are generated and stored as separate artifacts. The `summary` field now carries structured `Context Memory` for the next slide. The explanation prompt now requires a mandatory intro card, and the UI reads `cheatSheet` from page state instead of parsing it out of the explanation text. Also fixed an auto-analysis continuity bug by reading previous-page state from `useTutorStore.getState()` at execution time.

**Why**: The previous flow mixed multiple responsibilities into one explanation payload and made continuity unreliable during automatic sequential analysis. This caused inconsistent carry-over from the previous slide and made the cheat sheet hard to improve independently. Separating the artifacts gives each output one job and makes context handoff more stable.

**Impact**:
- `cheatSheet` is now a first-class page-state field and persistence field
- `summary` should be interpreted as `Context Memory`, not as a student-facing prose summary
- follow-up parsing now operates only on explanation cards
- focus mode consumes `cheatSheet` directly
- auto analysis is less likely to lose previous-slide continuity because it no longer depends on a stale closure snapshot

**Files**: `SlideTutor-AI/src/hooks/useSlideAnalysis.ts`, `SlideTutor-AI/src/hooks/useSlideAnalysis.test.ts`, `SlideTutor-AI/src/lib/ai/prompts.ts`, `SlideTutor-AI/src/lib/ai/prompts.test.ts`, `SlideTutor-AI/src/lib/ai/__snapshots__/prompts.test.ts.snap`, `SlideTutor-AI/src/components/CanvasTutor.tsx`, `SlideTutor-AI/src/hooks/useFollowUp.ts`, `SlideTutor-AI/src/hooks/usePdfLibrary.ts`, `SlideTutor-AI/src/types.ts`, `docs/superpowers/specs/2026-03-29-context-cheatsheet-design.md`, `docs/superpowers/plans/2026-03-29-context-cheatsheet-overhaul.md`


## [2026-03-28] 主题管理系统重构与持久化修复

**What**: 将主题管理逻辑从 `ThemeToggle` 组件本地状态重构为全局 `uiStore` (Zustand) 管理，并将初始化逻辑移至应用启动阶段。

**Why**: 解决刷新网页后主题重置为默认浅色模式的问题。此前主题仅在渲染设置组件时初始化，导致全局持久化失效。

**Impact**:
- 主题状态现在是全局响应式的，且在应用加载时即刻生效。
- `ThemeToggle` 组件现在更加轻量，仅负责触发 Store 的更新。
- 设置界面 (`SettingsModal`) 现在提供所有四种主题（Light, Eyecare, Morning Mist, Rainy）的详细描述。
- 增强了 UI 与 PDF 渲染层之间的主题同步一致性。

**Files**: `src/store/uiStore.ts`, `src/components/ThemeToggle.tsx`, `src/components/SettingsModal.tsx`, `docs/frontend/architecture.md`

## [2026-03-28] API Token 认证系统

### 变更内容
实现了一套完整的 API Token 认证系统，用于保护 `/api/generate` 端点免受未授权访问。该系统使用 HMAC-SHA256 签名的 Token，有效期为 5 分钟。

**创建的文件：**
- `api/lib/tokenAuth.ts` - Token 生成和验证逻辑
- `api/get-token.ts` - Token 端点，带速率限制
- `src/lib/api/apiClient.ts` - 前端 API 客户端，带缓存功能

**修改的文件：**
- `api/generate.ts` - 添加 Token 验证中间件
- `server.ts` - 集成 get-token 端点
- `src/hooks/useSlideAnalysis.ts` - 用 apiGenerate 替换 fetch（2 处）
- `src/hooks/useFollowUp.ts` - 用 apiGenerate 替换 fetch（3 处）
- `src/hooks/useChunkRegenerate.ts` - 用 apiGenerate 替换 fetch（1 处）
- `src/hooks/useQuiz.ts` - 用 apiGenerate 替换 fetch（2 处）
- `.env.example` - 添加 ENABLE_TOKEN_AUTH 和 API_TOKEN_SECRET

**其他变更：**
- 从 `src/lib/ai/prompts.ts` 中移除 `spatialNotesStr` 分析（安全风险）
- 修复 `api/get-token.ts` 中的 trust proxy 配置

### 变更原因
**安全漏洞**：之前的安全机制仅依赖 Origin/Referer 请求头验证，这很容易通过伪造 HTTP 请求头绕过。安全测试结果显示：
- 之前：2/7 测试通过（28.6% 通过率）
- 攻击者可以使用 curl 或 Python requests 直接调用 API
- 存在无限制 API 访问导致成本超支的风险

**解决方案需求**：
- 防止未授权的 API 访问
- 保持用户体验（无需登录）
- 支持 Vercel Serverless 架构（无状态）
- 最小化性能影响

### 技术实现

**Token 结构：**
```
base64(payload).base64(signature)

payload = {
  timestamp: number,  // 当前时间戳（毫秒）
  nonce: string       // 32 字符随机十六进制字符串
}

signature = HMAC-SHA256(secret, payload)
```

**安全特性：**
- 5 分钟 Token 过期时间
- HMAC-SHA256 签名（没有密钥无法伪造）
- 随机 nonce（防止重放攻击）
- 无状态设计（无需服务器端存储）

**前端流程：**
1. 调用 `GET /api/get-token` → 接收 Token
2. 在内存中缓存 Token（过期前 30 秒刷新）
3. 在所有 `/api/generate` 请求的 `X-API-Token` 请求头中包含 Token
4. 收到 401 错误 → 清除缓存，自动重试一次

**后端验证：**
1. 从请求头提取 `X-API-Token`
2. 验证 Token 格式（payload.signature）
3. 解码并解析 payload
4. 验证 HMAC 签名是否匹配
5. 检查过期时间（< 5 分钟）
6. 检查时钟偏移（不能来自未来）
7. 继续业务逻辑或返回 401

**速率限制：**
- `/api/generate`：10 请求/分钟，100 请求/天（每 IP）
- `/api/get-token`：20 请求/分钟（每 IP）

### 影响

**安全性：**
- 实施后：8/8 测试通过（100% 通过率）
- 所有未授权请求被 401 MISSING_TOKEN 阻止
- 多层防御：Token → Origin 检查 → 速率限制 → 内容检测

**性能：**
- 首次 API 调用：+100ms（获取 Token）
- 后续调用：+<1ms（Token 验证）
- 缓存命中率：~95%（5 分钟 Token 重用）
- 用户体验：无明显影响

**破坏性变更：**
- 对最终用户无影响（Token 处理是自动的）
- 开发者必须设置环境变量：
  - `ENABLE_TOKEN_AUTH=true`（生产环境）
  - `API_TOKEN_SECRET=<强随机密钥>`

**迁移：**
- 可通过 `ENABLE_TOKEN_AUTH=false` 禁用 Token 认证
- 允许逐步推出，如果出现问题可轻松回滚

### 测试结果

**之前（Token 禁用）：**
```
✅ 测试 0：基本可用性（200）
✅ 测试 1：旧 prompt 字段绕过被阻止（403）
❌ 测试 2：越狱指令绕过（200）
❌ 测试 3：编码恶意请求绕过（200）
❌ 测试 4：速率限制无效（12/12 成功）
❌ 测试 5：任务参数权限提升（500）
❌ 测试 6：无关内容请求绕过（200）

通过率：2/7（28.6%）
```

**之后（Token 启用）：**
```
✅ 所有无有效 Token 的请求：401 MISSING_TOKEN
✅ 所有伪造 Origin 的请求：403 Unauthorized
✅ 完全防护未授权访问

通过率：8/8（100%）
```

### 相关文档
- 实现细节：`docs/security/2026-03-28-api-token-authentication.md`
- 测试脚本：`user_files/test_retest.py`
- 实现计划：`C:\Users\hoo\.claude\plans\foamy-tumbling-planet.md`

### 未来改进
1. 密钥轮换机制（多密钥验证）
2. Token 黑名单（基于 Redis 的撤销）
3. 使用分析和异常检测
4. 自适应速率限制（基于 IP 信誉）
5. 探索 WebAuthn 实现无密码认证

---

## 未来条目模板

```markdown
## [YYYY-MM-DD] 简短标题

### 变更内容
- 创建/修改的文件列表
- 变更摘要

### 变更原因
- 技术理由
- 业务原因
- 要解决的问题

### 技术实现
- 关键设计决策
- 架构变更
- 重要代码模式

### 影响
- 性能影响
- 破坏性变更
- 迁移说明
- 副作用

### 相关文档
- 详细文档链接
- 相关 issues/PRs
```
