# GGlearn 教材生成两阶段优化设计

## 1. 文档目的

本设计用于优化 GGlearn 的教材生成 part，目标不是继续增强一次性 prompt，而是修正当前生成机制，使输出更像认真编写的教材，而不是来源资料的拼接整理。

本次优化直接针对以下问题：

- 单章缺少主线，读下来像并列片段
- 前置概念铺垫不足，顺序容易错位
- 文风偏资料摘要，不像教学讲解
- 正文展开偏空，缺少解释、过渡和推导

## 2. 设计结论

GGlearn 的教材生成应从“单次基于 evidence packs 直接生成整章 chunks”，改成“教学骨架先行的两阶段生成”。

新的生成链路固定为：

1. `chapter blueprint generation`
2. `section-by-section textbook generation`
3. `lightweight validation`

核心原则如下：

- 教学结构先于正文写作
- 教学职责先于证据分配
- evidence 服务教学写作，不反过来支配章节结构
- citation 继续保留，但属于验证与追溯层，不主导用户可见的教材形态

## 3. 现状问题

当前链路大致为：

- 基于 `ProjectSourceAsset` 构建 `evidencePacks`
- 将整章上下文和 evidence packs 一次性发给模型
- 让模型直接输出一组教材 chunks

这种方式的主要问题不是 prompt 不够长，而是控制粒度不对：

- 模型没有章节级主线约束
- 模型没有显式前置依赖约束
- 模型没有节级教学职责约束
- 模型在整章范围内同时处理结构、讲解和证据，容易退化成资料拼装

因此，若继续维持单阶段整章生成，即使 prompt 强化，也难以稳定解决教材感问题。

## 4. 目标与非目标

### 4.1 本次目标

- 让单章具备清晰主线与递进关系
- 让内容先铺垫基础，再进入更难概念
- 让正文更像教材讲解，而不是来源摘要
- 让每一节承担明确教学职责
- 在不破坏 grounding 的前提下提升教材感

### 4.2 非目标

- 不在第一版引入课程图谱系统
- 不在第一版重做 retrieval 架构
- 不在第一版引入向量数据库或新依赖
- 不在第一版重做 citation UI
- 不在第一版把每一节机械模板化成固定栏目

## 5. 两阶段生成总览

### 5.1 阶段一：生成 Chapter Blueprint

第一阶段不生成正文，只生成本章教学蓝图。

目标是先回答这些问题：

- 这一章到底要带读者学会什么
- 读者开始时默认知道什么、不知道什么
- 应先讲哪些概念，再讲哪些概念
- 每一节为什么存在
- 各节之间如何衔接

输出对象建议命名为 `ChapterBlueprint`。

### 5.2 阶段二：按 Blueprint 逐节生成正文

第二阶段不再一次性生成整章，而是按 section 顺序逐节生成。

每次生成只关注：

- 当前节的教学职责
- 当前节承接什么
- 当前节解决什么
- 当前节为下一节铺什么
- 当前节可以依赖哪些 evidence

这样可以显式控制顺序、过渡和展开深度。

### 5.3 阶段三：轻量校验

正文生成后，增加一个轻量校验步骤，用于检查：

- 每节是否有足够 evidence support
- 是否存在明显超出处据范围的强断言
- 是否存在节间跳跃或未解决的前置依赖

这一阶段不负责重写教材，只负责识别问题并返回校验信号。

## 6. Chapter Blueprint 设计

### 6.1 对象定义

建议新增如下对象：

```ts
type ChapterBlueprint = {
  chapterTitle: string;
  chapterGoal: string;
  targetReaderState: string;
  prerequisites: string[];
  chapterFlowNarrative: string;
  sections: ChapterSectionBlueprint[];
  endState: string;
  unsupportedGaps: string[];
};

type ChapterSectionBlueprint = {
  id: string;
  title: string;
  teachingRole:
    | 'motivation'
    | 'intuition'
    | 'definition'
    | 'explanation'
    | 'derivation'
    | 'application'
    | 'summary';
  dependsOnSections: string[];
  transitionFromPrevious: string;
  focusConcepts: string[];
};
```

### 6.2 字段职责

- `chapterTitle`
  当前章节的教学标题，不要求直接复述来源标题。
- `chapterGoal`
  明确本章学完后读者应掌握的结果。
- `targetReaderState`
  说明进入本章时默认读者已具备的理解基础。
- `prerequisites`
  本章展开前必须先铺好的概念或能力。
- `chapterFlowNarrative`
  用一段简短叙述说明整章主线如何推进。
- `sections`
  章节内部顺序化教学单元。
- `endState`
  本章结束时读者应达到的理解状态。
- `unsupportedGaps`
  当前来源不足以严谨展开但可能被误讲的内容。

### 6.3 Blueprint 设计原则

- 先定义教学顺序，再生成正文
- 先定义节职责，再选择 evidence
- blueprint 要服务“讲解”，不是服务“覆盖所有来源”
- blueprint 输出粒度保持中等，不在第一版过度细化到每个例题位或练习位

## 7. Section 正文生成设计

### 7.1 生成模式

正文生成阶段按 `ChapterBlueprint.sections` 顺序运行。

每一节单独生成，避免整章同时处理导致结构控制失效。

### 7.2 每节输入

建议每节生成时固定输入以下结构：

- `chapterGoal`
- `chapterFlowNarrative`
- `currentSection`
- `resolvedPrerequisites`
- `previousSectionSummary`
- `selectedEvidencePacks`
- `writingContract`

其中：

- `resolvedPrerequisites` 表示到当前节为止，已经在前文明确铺过的概念
- `previousSectionSummary` 用于确保节间过渡来自真实已生成内容，而不是 blueprint 的假设
- `writingContract` 用于明确禁止资料摘要式表达

### 7.3 写作约束

建议固定如下写作约束：

- 必须写成教材讲解，不得写成来源摘要
- 必须解释“为什么现在讲这一节”
- 必须围绕当前节职责展开，不要发散覆盖整章
- 可以概括来源，但不得伪造超出处据的具体结论
- 过渡句必须服务教学推进，而不是形式化连接词堆叠

### 7.4 每节输出结构

第一版不建议允许完全自由 chunk 输出，而建议约束为“节内教学结构”。

可选结构片段包括：

- `intro`
- `concept`
- `explanation`
- `example`
- `exercise`
- `summary`
- `bridge`

但并非每节都必须包含全部片段。应由 `teachingRole` 决定：

- `intuition` 更适合 `intro + explanation + example`
- `definition` 更适合 `intro + concept + explanation`
- `derivation` 更适合 `intro + explanation + bridge`
- `summary` 更适合 `summary + bridge`

第一版的重点不是模板复杂度，而是让每节承担明确教学功能。

## 8. Evidence 与 Citation 的角色分工

### 8.1 Blueprint 阶段

在 blueprint 阶段，evidence 主要用于判断：

- 哪些概念适合作为主线
- 哪些概念应后置
- 哪些主题证据充分
- 哪些推导存在资料缺口

此阶段不要求输出“每节绑定哪些 sourceRefIds”，避免教学骨架被 snippet 分配逻辑绑架。

### 8.2 Section 生成阶段

在 section 生成阶段，再为当前节挑选小而准的 evidence support。

建议规则：

- 每节选择 2-4 个相关 evidence packs
- 优先支持本节核心概念和展开解释
- 不追求单节覆盖所有来源
- 不为 citation 完整性牺牲节内连贯性

### 8.3 Citation 定位

citation 继续保留，但定位调整为：

- 证明当前节的内容具备来源支撑
- 为回看与追溯提供依据
- 作为 post-check 的输入之一

不应把“模型返回了 sourceRefIds”误当成“这一节已经写得像教材”。

## 9. 接口与职责调整建议

### 9.1 建议新增接口

```ts
async function generateChapterBlueprint(
  title: string,
  assets: ProjectSourceAsset[],
  goal: 'mastery' | 'exam' | 'micro',
  customFocus: string,
  language: 'zh' | 'en',
  config: AIConfig,
  chapterTopic?: string
): Promise<ChapterBlueprint>
```

### 9.2 建议调整接口

现有 `generateTextbookChunks(...)` 不再直接从 evidence packs 生成整章，而应改为：

- 调用 `generateChapterBlueprint(...)`
- 按 blueprint 逐节生成 section chunks
- 合并为 chapter-level chunk list

### 9.3 建议新增校验接口

```ts
type GeneratedSectionValidation = {
  hasEvidenceSupport: boolean;
  unsupportedClaims: string[];
  transitionIssues: string[];
  prerequisiteIssues: string[];
};

async function validateGeneratedSection(...): Promise<GeneratedSectionValidation>
```

第一版也可以只做 `validateGeneratedChapter(...)`，以降低实现复杂度。

## 10. 测试范围

第一版至少需要补这些测试：

- blueprint 生成结果包含章节主线、前置概念、顺序化 sections
- section 生成按 blueprint 顺序执行，不允许乱序
- 每节只消费小规模 selected evidence，而不是整章 evidence 全量灌入
- 若前一节摘要存在，当前节输出中应体现节间承接
- 校验步骤能够识别 unsupported claims 或 prerequisite gaps

测试重点不是模型文风绝对正确，而是链路约束已经从“整章自由生成”变成“骨架驱动的节级生成”。

## 11. 实施顺序

建议按以下顺序落地：

1. 新增 `ChapterBlueprint` 类型与 blueprint 生成接口
2. 重构 `generateTextbookChunks(...)` 为 blueprint-driven section generation
3. 增加 section 或 chapter 级轻量校验
4. 基于实际效果再决定是否增强 prerequisite graph、coverage budgeting 等机制

## 12. 风险与取舍

### 12.1 已接受的取舍

- 调用次数会上升，但换来更强的结构控制
- 第一版不追求节内最丰富的教学样式，而先追求主线和递进正确
- 第一版不做复杂课程规划器，避免过度设计

### 12.2 主要风险

- 若 blueprint 太空，第二阶段仍会写出空洞内容
- 若 section evidence 选择过宽，仍可能退化成资料拼接
- 若节间摘要过长，token 成本会快速上升

因此，第一版的控制重点应放在：

- blueprint 字段是否足够具体
- section 输入是否足够小且 focused
- validation 是否能指出明显结构问题

## 13. 最小落地范围

第一版建议只做以下三件事：

1. 新增 `generateChapterBlueprint(...)`
2. 将 `generateTextbookChunks(...)` 改为按 blueprint 逐节生成
3. 新增轻量 validation 步骤

其他增强项，如课程图谱、coverage optimizer、复杂 citation 展示，都应延后。
