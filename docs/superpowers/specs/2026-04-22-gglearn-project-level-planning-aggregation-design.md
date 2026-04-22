# GGlearn 项目级规划聚合层设计

Date: 2026-04-22
Status: Draft for review

## 1. 文档目的

本设计用于解决 GGlearn 当前教材生成中的重复章节问题。

本次设计的目标不是继续强化 prompt 约束，也不是简单增强标题去重规则，而是修正当前章节规划机制本身。

核心判断如下：

- 当前系统把“章节身份”定义得过早。
- 当前系统把 `source-local chapter seed` 当成了后续大纲规划的主输入。
- 这会导致多个来源中关于同一概念的不同表述，在进入大纲阶段前就已经被误建模成多个章节候选。

因此，本次设计要引入的不是通用知识库系统，而是一个更窄、更明确的“项目级规划聚合层”。

## 2. 设计结论

GGlearn 应从：

`asset -> chapterSeeds -> title clustering -> outline`

改成：

`assets -> project-level topic aggregation -> outline -> chapter generation`

本次设计的关键结论如下：

1. 不应再让 `chapterSeeds` 充当正式规划边界。
2. 应在 `outlineGeneration` 之前新增项目级主题归并步骤。
3. 该步骤的输出不是最终章节，而是“项目级概念 / 主题聚合结果”。
4. `OutlineChapter` 的身份应从“标题”迁移为“conceptIds”。
5. 第一版只做运行时规划视图，不引入新的持久化真相层。

## 3. 现状问题

### 3.1 当前链路

当前 GGlearn 的大纲生成机制大致如下：

1. 每个 `ProjectSourceAsset` 在本地或 AI enrichment 阶段产出 `chapterSeeds`
2. 系统收集所有 asset 的 `chapterSeeds`
3. 基于标题 token overlap 做粗粒度聚类
4. 将聚类结果转成 `OutlineChapter`
5. 再让 AI 改写章节标题与学习目标
6. 后续正文生成继续用 `chapterTitle` 组织 evidence

这个链路的根问题不是“标题太差”，而是“章节单位来源不对”。

### 3.2 为什么会重复

假设项目里有三个来源，分别强调：

- Jacobian 的定义
- Jacobian 的多维变化率直觉
- 为什么要学习 Jacobian

当前系统会先让这三个来源各自产生自己的 `chapterSeed`，然后再尝试基于标题相似度去重。

于是：

- “什么是雅可比矩阵”
- “雅可比矩阵：多维空间的变化率”
- “为什么要学雅可比矩阵”

很可能会被保留成多个章节候选，而不是先被视为“同一个核心主题的不同教学切面”。

### 3.3 为什么单纯去重不够

当前机制里，标题不仅是展示名，还承担了过多职责：

- 章节候选的身份标识
- 去重的比较对象
- evidence 检索的 query 信号
- AI 改写的输入

这意味着只要章节标题层面的建模有偏差，后续大纲和正文都会放大这个偏差。

因此，本次设计不能只修 `clusterChapterSeeds()` 的规则，而必须改“大纲规划的输入单位”。

## 4. 目标与非目标

### 4.1 本次目标

- 让同一概念的多来源表述在进入大纲前先被合并
- 让章节规划单位从 `source-local seed` 变成 `project-level topic`
- 保持现有 `ProjectSourceAsset` 为上游主边界
- 保持 grounding 与 evidence lineage 可追踪
- 尽量少改正文生成主链路

### 4.2 非目标

- 不做 Open-NotebookLM 式 notebook knowledge base 重构
- 不将系统改成 vector-store-first 架构
- 不新增 graph database
- 不新增长期持久化的“第二真相层”
- 不在第一版重写 chapter blueprint 与 section generation 流程
- 不在第一版做跨项目统一知识池

## 5. 什么是项目级规划聚合层

“项目级规划聚合层”可以简单理解为：

在正式生成教材目录前，先站在整本书的角度，把所有来源里“其实在讲同一个主题”的内容归并成一组全局主题。

它不直接写正文，也不直接生成最终目录。

它只负责三件事：

1. 收集所有来源中的主题信号
2. 把相近主题归并成项目级概念 / 主题节点
3. 为每个主题节点保留明确的证据支持集合

这一步完成后，大纲规划器面对的就不再是几十个来源各自产生的章节建议，而是一个已经在项目级统一整理过的主题集合。

## 6. 方案概览

本方案采用 4 层结构：

### 6.1 `Source Asset Layer`

继续保留现有 `ProjectSourceSnapshot -> ProjectSourceAsset` 主链路。

这层继续负责：

- `snapshotLayer`
- `structureLayer`
- `retrievalLayer`
- `planningLayer`
- `projectionLayer`

其中：

- `retrievalUnits`
- `evidenceSnippets`
- `conceptIndex`
- `keyConcepts`

仍然是项目级规划聚合层的主要输入。

### 6.2 `Project Planning Aggregation Layer`

新增运行时 `ProjectConcept` / `TopicCluster` 聚合步骤。

这层从所有已选 source assets 中提取规划信号，并做项目级主题归并。

其输出不是最终 chapter，而是一组“chapter-worthy topic candidates”。

### 6.3 `Book Planning Layer`

大纲规划器只消费：

- `ProjectConcept[]`
- `LearningBrief`

大纲规划器的职责变成：

- 挑选哪些主题值得进书
- 决定顺序
- 决定哪些主题应合成一章
- 决定哪些主题应拆成多章

### 6.4 `Chapter Realization Layer`

正文生成阶段不再主要依赖 `chapterTitle` 去找 evidence，而应优先根据 `conceptIds` 拉取证据。

标题在这一步退回成用户可见的展示属性。

## 7. 最小数据结构

第一版建议只新增少量结构，避免引入新的持久化真相层。

```ts
type ProjectConcept = {
  id: string;
  canonicalName: string;
  aliases: string[];
  summary: string;
  evidenceRefs: ConceptEvidenceRef[];
  prerequisiteConceptIds: string[];
  difficulty: 'introductory' | 'intermediate' | 'advanced';
  coverageScore: number;
  riskFlags: string[];
};

type ConceptEvidenceRef = {
  sourceId: string;
  assetId: string;
  retrievalUnitId: string;
  snippetId?: string;
  anchorRef?: string;
  role: 'primary' | 'supporting';
};

type ProjectConceptIndex = {
  version: 1;
  concepts: ProjectConcept[];
};
```

同时建议给现有 `OutlineChapter` 增加：

```ts
type OutlineChapter = {
  // existing fields...
  conceptIds: string[];
};
```

## 8. 聚合层输入与输出

### 8.1 输入

聚合层的输入应优先来自现有资产中的稳定信号：

- `retrievalUnits`
- `conceptIndex`
- `keyConcepts`
- `evidenceSnippets`
- `sourceGuide`

`chapterSeeds` 在第一版中只能作为弱信号或兼容输入，不能再是主输入。

### 8.2 输出

每个聚合主题至少必须给出：

- `canonicalName`
- `aliases`
- `summary`
- `evidenceRefs`
- `coverageScore`
- `riskFlags`

这里最关键的是：

`evidenceRefs` 是强制项，不是可选项。

如果一个聚合主题没有明确支持它的 `retrievalUnitId / snippetId / sourceId`，它就不应进入大纲规划。

## 9. 聚合规则

第一版不建议做重型知识图谱，而应采用可解释、可回溯的轻量聚合规则。

建议优先信号如下：

1. 共享或高度重合的 `conceptRefs`
2. 共享或相邻的 `retrievalUnits`
3. 术语别名归一化
4. 多来源对同一概念的相似 summary
5. AI enrichment 中给出的概念别名或归一化提示

第一版建议保守策略：

- 宁可少合并，也不要过度误合并
- false merge 比 false split 更危险

原因是：

- 误合并会破坏课程结构
- 轻微重复仍可通过后续大纲层发现
- 但过度误合并会让正文阶段直接失真

## 10. 生成流程

新的链路建议如下：

1. 导入来源
2. 生成 `ProjectSourceSnapshot`
3. 生成 `ProjectSourceAsset`
4. 从所有选中 assets 构建 `ProjectConceptIndex`
5. 基于 `ProjectConceptIndex + LearningBrief` 生成 `EnhancedOutline`
6. 每个 chapter 绑定 `conceptIds`
7. 正文生成阶段按 `conceptIds` 选择 evidence
8. 输出最终教材 chunks

### 10.1 关键变化

变化不在正文生成的整体架构，而在“大纲规划前的输入单位”。

也就是说，本次设计优先修改的是：

- `outlineGeneration` 的输入
- `textbookGeneration` 的 evidence selection 主信号

而不是整条正文生成流水线。

## 11. 为什么这能解决重复章节

因为在新机制下：

- 同一个 Jacobian 相关主题会先被并入同一个 `ProjectConcept`
- 多个来源提供的是同一主题的不同证据，而不是多个章节候选
- 大纲规划面对的是“全项目主题集合”，不是“各来源章节建议列表”

因此：

- 章节数量不再被来源数量牵着走
- 同一概念更容易变成同一章下的不同 section
- 多来源差异被用于丰富同一章，而不是制造重复章

## 12. 与 Open-NotebookLM 的关系

本方案吸收 Open-NotebookLM 的启发，但不照搬其主架构。

### 12.1 可以借鉴的部分

- 原始来源与解析副产物的分层保存
- 项目 / notebook scoped 的来源隔离
- 检索结果必须可回跳到来源位置
- 来源映射与预览能力

### 12.2 不应照搬的部分

- raw chunk RAG 作为知识主边界
- notebook knowledge base 作为统一主心智
- 多输出功能各自直连全文 prompt
- vector-store-first 的整体架构

GGlearn 仍应坚持：

`source-asset-first`

而不是退回：

`raw-chunk-first`

## 13. 迁移策略

### 13.1 第一阶段：运行时聚合

先只增加运行时 `ProjectConceptIndex`，不落库。

这可以避免：

- 第二套持久化真相层
- 历史项目迁移压力
- 过早扩大改动范围

### 13.2 第二阶段：替换 outline 输入

把 `generateEnhancedOutline()` 的主输入从 `chapterSeeds` 切换为 `ProjectConcept[]`。

旧的 `chapterSeeds` 流程保留为 fallback，仅用于兼容。

### 13.3 第三阶段：调整 evidence selection

把正文生成中的主 evidence 组织方式从 `chapterTitle` 驱动切换为 `conceptIds` 驱动。

标题匹配只保留为 fallback。

### 13.4 第四阶段：收缩 AI enrichment 责任

`sourceAsset.ts` 中的 AI enrichment 输出重点应从：

- `chapterSeeds`

收缩为：

- `aliases`
- `normalizedConcepts`
- `coverageGaps`
- `prerequisiteHints`

也就是让 AI 帮系统做概念归一化，而不是直接替来源定章。

### 13.5 第五阶段：是否持久化再议

只有在以下条件满足后，才考虑持久化 `ProjectConceptIndex`：

- 重复章节率明显下降
- grounding 没有退化
- evidence lineage 仍然稳定
- source 删除 / 重解析 / 重生成语义已经明确

## 14. 风险与约束

### 14.1 主要风险

- 主题粒度定义不清，导致聚合过粗或过细
- 概念别名归并不稳，导致误合并
- 引入第二真相层，打乱现有边界
- 章节身份与 evidence lineage 脱钩

### 14.2 必须坚持的约束

1. `ProjectSourceAsset` 仍是上游主边界
2. 聚合层第一版只做运行时视图
3. 每个聚合主题必须保留 support set
4. planner 不得直接消费脱离 evidence 的抽象 summary
5. 第一版只改规划输入层，不扩大到整条正文链路重写

## 15. 验收标准

本方案是否成立，不看概念名词是否漂亮，而看以下指标：

1. 重复章节率是否明显下降
2. 章节间主题重叠是否降低
3. 章节 evidence coverage 是否不下降
4. sourceRefs 完整性是否不下降
5. 重生成时章节边界是否更稳定

## 16. 最终推荐

最终推荐方案如下：

- 不做 NotebookLM 式统一知识库底座
- 不继续把 `chapterSeeds` 当主规划单位
- 新增一个项目级规划聚合层
- 第一版只做运行时 `ProjectConcept / TopicCluster`
- 用它替换 `chapterSeeds` 作为大纲生成主输入
- 让 `OutlineChapter` 从“按标题定义”改成“按 conceptIds 定义”

这是一条最短、最稳、最贴近 GGlearn 当前架构的治本路径。
