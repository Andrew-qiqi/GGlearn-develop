# GGlearn 来源解析主链路重构设计

Date: 2026-04-22
Status: Draft for review

## 1. 文档目的

本设计用于解决 GGlearn 教材生成中最上游的来源污染问题。

当前系统已经暴露出以下症状：

- 网页来源中 URL 残片、站点导航词、组件名、模板词被误当作概念信号
- 章节规划阶段会出现脏 chapter 候选
- 正文生成阶段会围绕近义标题重复讲述同一主题
- evidence 与章节身份的绑定不稳定，导致后续链路放大前面的错误

这些问题的共同根因不是单一 prompt 质量，也不是标题去重规则太弱，而是：

- GGlearn 当前没有独立、可靠的“来源解析层”
- 系统过早地从不稳定的原始文本中推导 `keyConcepts`、`conceptIndex`、`chapterSeeds`
- 规划与正文生成都建立在这个脏输入之上

因此，本设计的目标不是继续微调现有 `sourceTransform.ts`，而是正式引入 `parser-first` 的来源主链路。

## 2. 设计结论

GGlearn 应从当前近似的：

`raw source -> rawText -> heuristic extraction -> source asset -> project concept aggregation -> outline`

迁移为：

`raw source -> source parser engine -> NormalizedSourceDocument -> source asset builder -> project concept aggregation -> outline -> chapter realization`

关键结论如下：

1. `NormalizedSourceDocument` 应成为来源解析阶段的统一真相层。
2. 文档来源与网页来源应拆分为两个独立解析引擎。
3. `sourceTransform.ts` 不再承担“从脏文本猜概念”的职责。
4. `chapterSeeds` 应降级为兼容/回退数据，不再作为正式规划输入。
5. 教材规划与正文生成最终都应围绕 `conceptIds + evidence refs`，而不是围绕标题字符串。

## 3. 现状问题

### 3.1 当前来源处理方式的问题

当前 GGlearn 的上游来源处理存在以下结构性问题：

- URL 来源基本等价于“抓取正文字符串”
- PDF 来源以轻量结构重组为主，不是正式 parser-first
- `pickKeywords()`、`pickKeyConcepts()`、`buildConceptIndex()` 基于原始文本启发式推导
- `chapterSeeds` 在来源阶段过早形成“章节边界”幻觉

这意味着：

- 一旦原始文本中混入模板词、导航词、目录词、页眉页脚、广告或站点组件名
- 系统就会把这些污染直接传播到 retrieval、concept、chapter planning

### 3.2 为什么不能继续补丁式修修补补

如果继续在现有链路上做局部修补，例如：

- 增加 stopwords
- 增加更多网页噪音正则
- 强化 chapter title 去重
- 追加 prompt 约束

这些只能缓解局部症状，不能修复来源真相层缺失的问题。

原因是：

- 现有系统把“来源解析”“概念识别”“教材规划信号生成”混在同一层
- 只要最开始的概念候选脏，后面任何 AI 润色或排序都只是脏输入上的后处理

## 4. 目标与非目标

### 4.1 本次目标

- 建立 GGlearn 的统一来源真相层
- 为 PDF/Office 与 URL/Web 分别定义独立解析引擎接口
- 让 source asset 只消费规范化后的结构文档，而不是直接消费原始字符串
- 显式保留 noise / discarded 信息，避免来源污染隐身
- 为后续 evidence、concept、chapter planning 提供稳定输入

### 4.2 非目标

- 本次不直接实现完整的向量库重构
- 本次不重做整个 Reader UI
- 本次不重写所有 AI prompt
- 本次不把教材生成立即改成多阶段 agent 系统
- 本次不引入跨项目的全局知识池

## 5. 新的核心中间层：NormalizedSourceDocument

`NormalizedSourceDocument` 是来源解析层的统一输出。

无论输入来自：

- PDF
- DOCX
- PPTX
- 网页 URL
- 手动粘贴文本

都必须先转换到这个统一结构，再进入 source asset builder。

### 5.1 最小结构

```ts
type NormalizedSourceDocument = {
  id: string;
  sourceId: string;
  sourceKind: 'pdf' | 'docx' | 'pptx' | 'url' | 'text';
  parserEngine: string;
  parserVersion: string;
  title: string;
  language?: string;
  markdown: string;
  sections: NormalizedSection[];
  blocks: NormalizedBlock[];
  anchors: NormalizedAnchor[];
  assets: NormalizedEmbeddedAsset[];
  discardedBlocks: NormalizedBlock[];
  warnings: string[];
  lineage: {
    originalUrl?: string;
    originalFilePath?: string;
    generatedAt: number;
  };
};
```

### 5.2 Block 结构要求

最少需要支持：

- `heading`
- `paragraph`
- `list`
- `quote`
- `table`
- `formula`
- `figure`
- `caption`
- `code`
- `reference`
- `noise`

每个 block 至少应保留：

- `id`
- `type`
- `text`
- `sectionId`
- `anchorId`
- `order`
- `pageNumber?`
- `bbox?`
- `metadata`

### 5.3 为什么必须有 discardedBlocks

GGlearn 当前一个关键问题是：

- 上游脏内容没有被显式建模
- 系统只能看到“保留下来的文本”，看不到“本应被丢弃的噪音”

新增 `discardedBlocks` 的目的：

- 记录页眉页脚、页码、导航词、站点组件词、重复 heading、模板性残片
- 为后续调试 parser 质量提供可观察性
- 防止噪音通过隐式路径回流进 asset builder

## 6. 双引擎解析架构

### 6.1 DocumentEngine

处理对象：

- PDF
- DOCX
- PPTX
- 其他 office / ebook 类文档

职责：

- 保留阅读顺序
- 保留标题层级
- 尽可能保留表格、公式、图注、页锚点
- 输出 markdown + structured blocks

推荐方向：

- 第一优先：`MinerU`
- 可选兼容：`Docling`
- 最终保留一个 `simple` 回退实现

### 6.2 UrlEngine

处理对象：

- 新闻文章
- 博客
- 百科词条
- 在线文档
- 可能含 JavaScript 渲染的网页

职责：

- 提取正文而不是整页文本
- 去除 boilerplate、导航、页脚、站点组件与推荐内容
- 输出 markdown + structured blocks

推荐方向：

- `firecrawl`
- `jina`
- `trafilatura/readability` 类正文提取器
- 必要时浏览器渲染后再解析

### 6.3 Engine 接口建议

```ts
type ParseSourceInput = {
  sourceId: string;
  sourceKind: 'pdf' | 'docx' | 'pptx' | 'url' | 'text';
  filePath?: string;
  url?: string;
  rawText?: string;
  language?: 'zh' | 'en';
};

type SourceParserEngine = {
  id: string;
  canHandle(input: ParseSourceInput): boolean;
  parse(input: ParseSourceInput): Promise<NormalizedSourceDocument>;
};
```

关键要求：

- 业务层只能依赖接口，不应把某个 parser 的细节硬编码进教材逻辑
- `DocumentEngine` 与 `UrlEngine` 可以在配置层选择 provider
- 所有 provider 最终统一落为 `NormalizedSourceDocument`

## 7. 对现有 GGlearn 模块的职责重划

### 7.1 `sourceImport.ts`

调整为：

- 接收原始来源
- 分发给正确的 parser engine
- 产出 `NormalizedSourceDocument`
- 仅保留来源原始元数据与 parser 元数据

不再负责：

- 直接生成面向教材规划的概念信号

### 7.2 `sourceTransform.ts`

调整为：

- 从 `NormalizedSourceDocument` 派生 `ProjectSourceAsset`
- 构建 `retrievalUnits`
- 构建 `evidenceSnippets`
- 构建 `structureLayer` 与 `projectionLayer`

不再负责：

- 从原始段落直接猜 `keywords`
- 从前几段首句猜 `keyConcepts`
- 从词项匹配构造高优先级 `chapterSeeds`

### 7.3 `sourceAsset.ts`

AI enrichment 应保留，但职责必须收窄为：

- 教学导读
- 教学用途建议
- learning opportunities
- diagram opportunities

不应再允许 AI enrichment 去修改 parser 真相层，也不应直接覆盖基础概念层。

### 7.4 `projectConceptPlanning.ts`

其输入应从：

- 词项 + aliases 主导

迁移为：

- normalized blocks
- evidence refs
- section hierarchy
- retrieval units
- parser anchors

project concept 的本体应更像：

- 结构化证据簇上的主题归并

而不是：

- 文本词项归并

## 8. 哪些旧机制必须退役或降级

以下机制不应再作为正式主路径：

- `pickKeywords()`
- `pickKeyConcepts()`
- 现有 `buildConceptIndex()` 主逻辑
- `buildChapterSeeds()`
- `chapterSeeds` 作为正式规划边界
- URL 来源的纯文本去标签导入

它们最多保留为：

- fallback
- debug
- backward compatibility

但不能继续承担来源真相生成职责。

## 9. 对教材生成后续链路的直接影响

### 9.1 章节规划

章节规划将不再主要消费：

- source-local chapter seed
- raw title similarity

而主要消费：

- parser 后的结构化概念证据
- project-level concept aggregation

### 9.2 evidence 与 citation

evidence refs 应从当前偏弱的 snippet/title 匹配，升级为：

- block-level ref
- section-level ref
- page / anchor ref
- formula / figure / table ref

这样正文阶段可以真正围绕 `conceptIds + evidence refs` 组织内容。

### 9.3 正文生成

正文生成不应再把 `chapter.title` 当成强主键。

章节标题只是：

- 展示层

真正主键应是：

- `conceptIds`
- `evidenceMapping`
- `chapterPlan` 绑定的结构化证据集合

## 10. 推荐迁移顺序

为了降低风险，迁移顺序应如下：

1. 先引入 `NormalizedSourceDocument` 类型与 parser engine 接口
2. 先替换 PDF/Office 通道
3. 再替换 URL/Web 通道
4. 重写 source asset builder，让其只消费 normalized document
5. 重写 project concept aggregation
6. 最后收口到 outline / textbook generation 的 concept-first 驱动

其中第一阶段优先级最高的是：

- PDF 结构化解析质量
- URL 正文提取质量

## 11. 风险与权衡

### 11.1 会增加系统复杂度

是的，但这是必要复杂度。

当前系统的复杂度已经存在，只是被埋在错误的地方：

- 各种启发式抽取
- prompt 修补
- chapter 去重
- evidence 偏移补救

引入 parser-first 后，复杂度会前移到来源层，但整体系统会更可控。

### 11.2 会增加依赖与运行成本

是的。

但来源真相层质量是教材生成的第一性前提。  
如果来源解析质量不足，后面任何 outline 或正文优化都只是建立在错误地基上的返工。

### 11.3 为什么不做最小改动

因为本问题已经被证明不是最小修补可解的问题：

- 来源污染会持续放大到 chapter planning
- chapter planning 的错误会继续放大到正文生成
- 正文阶段再修，只会变成对错误结构的表面润色

## 12. 最终设计判断

GGlearn 应直接采用：

- `Open-NotebookLM-main` 的 parser-first 文档思路
- `open-notebook-main` 的 parser engine 抽象边界

组合成自己的来源主链路：

- 文档用高质量 parser
- URL 用独立正文引擎
- 统一汇合到 `NormalizedSourceDocument`
- 教材规划和正文生成全部建立在这一真相层之上

这不是优化项，而是教材生成质量链路中的基础设施重构。
