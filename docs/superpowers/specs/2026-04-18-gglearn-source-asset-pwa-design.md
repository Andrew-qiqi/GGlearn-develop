# GGlearn 第一阶段 Source Asset 与 PWA 重建设计

## 1. 设计目标

GGlearn 第一阶段的目标，不是继续在现有原型上堆叠“更会生成教材”的页面逻辑，而是重建一个更稳定的产品骨架：

- 产品形态为本地优先、无服务端的 PWA
- 学习资产离线可用
- AI 功能在线，使用用户提供的 API key
- 支持 PDF 与 URL 导入，且以 PDF 为主路径
- 每本教材项目绑定自己的来源库
- 来源转换层的核心职责是服务后续 AI 的检索、引用与教材生成，而不是只生成给用户看的摘要卡片

本设计参考 NotebookLM 的产品思路，但不复制其实现。可确认的共性是：来源不会只作为“原文附件”存在，而会被转换为更利于后续 AI 工作的项目内知识资产。

## 2. 产品边界

### 2.1 第一阶段明确要做的事

- 创建教材项目
- 导入 PDF / URL 来源
- 为项目内来源生成静态快照
- 将来源转换为 AI-ready source assets
- 向用户展示来源工作台与来源导读投影视图
- 基于来源资产生成第一版教材
- 在教材阅读页中继续学习、记笔记、保存手写，并可回看来源依据

### 2.2 第一阶段明确不做的事

- 无全局共享资料库
- 无一个来源库生成多本教材的通用复用模型
- 无服务端代理层
- 无本地模型推理
- 无复杂跨项目协作
- 无完整的高级检索系统（如真正的向量数据库）

## 3. 核心对象模型

GGlearn 第一阶段建议固定为以下对象层级：

### 3.1 TextbookProject

教材项目，是第一阶段最重要的容器对象。

职责：

- 绑定该教材的来源库
- 管理教材生成状态
- 管理教材内容与阅读状态
- 管理学习记录

建议字段：

- `id`
- `title`
- `mode`
- `language`
- `createdAt`
- `updatedAt`
- `status`
- `sourceIds`
- `textbookId`
- `studyRecordId`

### 3.2 ProjectSource

原始来源对象，表示用户导入的一份 PDF 或一个 URL。

职责：

- 记录来源类型与原始入口
- 记录原始处理状态
- 作为 snapshot 与 asset 的上游

建议字段：

- `id`
- `projectId`
- `kind` (`pdf` | `url`)
- `title`
- `origin`
- `mimeType`
- `importedAt`
- `status`

### 3.3 ProjectSourceSnapshot

来源的冻结快照，用于保留结构化抽取结果，而不是持续依赖原始外部内容。

职责：

- 保存提取文本
- 保存位置锚点
- 保存基础元数据
- 为后续 source asset 转换提供稳定输入

建议字段：

- `id`
- `projectSourceId`
- `rawText`
- `sections`
- `anchors`
- `pageCount`
- `wordCount`
- `captureMeta`
- `capturedAt`

### 3.4 ProjectSourceAsset

项目内来源转换资产，是第一阶段真正的核心。

职责：

- 让 AI 更容易检索相关知识单元
- 让 AI 更稳地引用来源证据
- 让 AI 更容易生成教材结构与章节内容
- 支持用户看到来源导读，但导读不是资产的中心职责

### 3.5 StudyRecord

学习记录对象。

职责：

- 保存笔记
- 保存手写
- 保存完成进度
- 保存导出历史

## 4. ProjectSourceAsset 分层设计

ProjectSourceAsset 不应等同于“摘要”。建议采用 5 层结构。

### 4.1 Snapshot Layer

这层主要承接快照输入，并保留后续追溯能力。

建议内容：

- `snapshotId`
- `sourceKind`
- `canonicalTitle`
- `sourceMeta`
- `anchorStrategy`
- `parseStatus`

### 4.2 Structure Layer

这层负责把原始来源变成可理解的结构化内容。

建议内容：

- `sectionTree`
- `paragraphBlocks`
- `formulaBlocks`
- `figureCaptionBlocks`
- `noiseBlocks`
- `normalizationNotes`

设计原则：

- PDF 与 URL 最终都要尽量收敛到统一结构
- 不要求完美识别，但要明确标出脏区与低可信区

### 4.3 Retrieval Layer

这是第一阶段最关键的一层，服务后续 AI 检索与引用。

建议内容：

- `retrievalUnits`
- `conceptIndex`
- `evidenceSnippets`
- `retrievalHints`
- `riskFlags`

其中：

- `retrievalUnits` 是后续召回与拼装教材的最小主要单元
- `evidenceSnippets` 保留来源锚点，供引用和追溯
- `conceptIndex` 帮助模型围绕概念而不是围绕长原文工作
- `riskFlags` 标记 OCR 噪声、网页脏内容、重复段、低可信片段

#### RetrievalUnit 建议字段

- `id`
- `assetId`
- `title`
- `content`
- `summary`
- `anchorRefs`
- `conceptRefs`
- `prerequisites`
- `difficulty`
- `teachingValue`
- `citationSnippetIds`

#### EvidenceSnippet 建议字段

- `id`
- `assetId`
- `text`
- `anchorRef`
- `sourceSectionTitle`
- `confidence`
- `quoteKind`

### 4.4 Planning Layer

这层是来源资产服务教材生成的过渡层。

建议内容：

- `chapterSeeds`
- `learningObjectives`
- `exerciseSeeds`
- `diagramOpportunities`
- `difficultySignals`
- `coverageGaps`

这层的目标不是直接输出最终教材，而是为教材生成提供“可教学的候选组织方式”。

### 4.5 Projection Layer

这是用户能看到的来源导读投影视图。

建议内容：

- `sourceGuide`
- `keywords`
- `keyConcepts`
- `readingTimeEstimate`
- `recommendedUse`

注意：

- 这一层面向用户理解
- 但不应反过来主导整个 source asset 的设计

## 5. 来源处理流水线

第一阶段建议固定如下流水线：

### 5.1 Import

用户导入 PDF 或 URL，同时创建 `TextbookProject`。

### 5.2 Snapshot

对来源做静态化处理：

- PDF：提取文本、页码、标题线索
- URL：提取正文文本、页面标题、基础 metadata

输出 `ProjectSourceSnapshot`。

### 5.3 Normalize

对 snapshot 进行规范化处理：

- 切章节
- 切段落
- 识别公式与图示说明
- 识别明显噪声

### 5.4 Segment

把规范化内容拆为 AI 后续可用的片段：

- 生成 `retrievalUnits`
- 生成 `evidenceSnippets`
- 建立 `anchorRefs`

### 5.5 Enrich

调用在线 AI 为来源资产补充高价值结构：

- 概念索引
- 章节种子
- 学习目标
- 练习点
- 风险提示

### 5.6 Project Source Workspace

在项目内来源工作台展示：

- 来源卡片
- 来源导读
- 关键概念
- 质量状态

但底层仍以 source asset 为中心，而不是仅做展示页。

### 5.7 Generate Textbook

教材生成不再直接消费整篇原文，而是优先消费：

- `retrievalUnits`
- `evidenceSnippets`
- `chapterSeeds`
- `learningObjectives`

### 5.8 Read & Study

用户进入教材阅读页，继续：

- 阅读教材
- 写笔记
- 保存手写
- 查看来源依据

## 6. 页面结构建议

第一阶段建议稳定为 3 个主要页面区域。

### 6.1 导入页

职责：

- 创建教材项目
- 导入 PDF / URL
- 触发来源处理

### 6.2 来源工作台

职责：

- 查看项目内来源
- 查看来源导读投影视图
- 查看处理状态
- 决定是否进入教材生成

默认主流程应支持“导入后直接进入生成教材”，但来源工作台必须存在。

### 6.3 教材阅读页

职责：

- 展示教材内容
- 展示学习记录
- 支持来源追溯

## 7. 第一阶段最小可用闭环

建议第一阶段只要求跑通以下闭环：

1. 新建教材项目
2. 导入 PDF 或 URL
3. 生成项目内 snapshot
4. 转换为 AI-ready source asset
5. 在来源工作台可见来源导读与状态
6. 基于这些来源资产生成第一版教材
7. 进入阅读页并可回看来源依据

这个闭环已经足以验证 GGlearn 的核心产品假设：

- 来源转换是否真的提升教材生成质量
- 项目绑定来源库的心智是否清晰
- 离线学习资产是否足够稳定

## 8. 从现有原型迁移的顺序

当前代码已经有部分重建基础，但还没有进入“项目 + 来源资产”模型。

### 8.1 第一步：重写类型边界

优先调整 `src/types.ts`，新增：

- `TextbookProject`
- `ProjectSource`
- `ProjectSourceSnapshot`
- `ProjectSourceAsset`
- `RetrievalUnit`
- `EvidenceSnippet`
- `StudyRecord`

### 8.2 第二步：重写存储模型

调整 `src/lib/persistence.ts`，把当前“教材数组持久化”改为“项目树持久化”。

目标：

- 教材不再是唯一主对象
- 项目成为持久化根节点
- 来源、资产、学习记录都挂在项目下

### 8.3 第三步：拆分服务边界

从 `src/App.tsx` 中拆出三条服务流：

- source import flow
- source transform flow
- textbook generation flow

### 8.4 第四步：收紧 AI 模块边界

调整 `src/lib/gemini.ts` 的职责，不再只承担“直接生成教材”，而是分成更明确的两类能力：

- source transformation
- textbook generation

### 8.5 第五步：新增来源工作台视图

现有 `views` 结构可以保留，但要新增明确的 `Source Workspace` 视图位置。

### 8.6 第六步：让 Reader 依赖来源资产而不是原文拼接

教材阅读页后续应消费项目生成结果与引用关系，而不是继续把来源文本当作临时拼接材料。

## 9. 验证标准

第一阶段设计是否成功，可以用以下标准判断：

- 用户是否能创建一个项目并导入来源
- 项目内是否生成稳定的 source snapshots 与 source assets
- 教材生成是否明确依赖来源资产而不是直接依赖整篇原文
- 教材是否能回看来源依据
- 用户的笔记、手写、进度是否在离线状态下仍可使用

## 10. 决策总结

本设计的核心判断是：

GGlearn 第一阶段不应先做“更漂亮的教材页面”，而应先做“项目内、面向 AI 检索与教材生成的来源转换层”，教材只是这层资产的第一个主要消费方。

如果这一层建立成功，后续的大纲优化、章节生成、引用追溯、手写学习与导出能力都会更容易稳定下来。
