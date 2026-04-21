# GGlearn PDF Source Asset Internal Evidence Design

Date: 2026-04-21
Status: Draft for review

## 背景

GGlearn 当前已经确定采用 source-asset-first 架构。教材生成的核心边界不是原始来源文本，也不是通用问答式 RAG，而是 `Project -> ProjectSource -> ProjectSourceAsset`。

当前系统已经具备以下基础：

- PDF 可提取基础文本、页码和段落锚点。
- `ProjectSourceSnapshot` 已能承接 `rawText / sections / anchors / pageCount / quality`。
- `ProjectSourceAsset` 已分为 `snapshotLayer / structureLayer / retrievalLayer / planningLayer / projectionLayer`。
- 教材生成阶段已经要求每个 chunk 绑定给定的 `sourceRefIds`，用于内部证据约束。

但当前仍存在两个明显缺口：

- PDF 结构化结果主要仍以文字为主，图片、图注、公式尚未稳定进入 source asset 主链路。
- 内部来源绑定的语义已经存在，但边界还不够明确，容易被误解为未来要给用户暴露“来源回溯”。

## 核心产品原则

本方案明确采用以下产品原则：

1. 来源绑定是内部控制能力，不是用户功能。
2. 用户阅读教材时不应被来源、页码、证据面板打扰。
3. 系统内部必须保留足够强的来源约束，用于生成控制、质量校验、重生成稳定性和调试。
4. PDF 结构化改造的第一阶段只服务内部证据绑定，不改阅读端交互，不新增面向用户的来源展示。

## 目标

本轮目标不是做“可回溯来源的阅读器”，而是做“对系统内部可控、对用户前台隐形”的 PDF 结构化来源资产层。

完成后，系统内部应能稳定回答以下问题：

- 某段教材内容依赖了哪几个 PDF 内部证据单元。
- 某条公式或定义来自哪一页、哪一段、哪个 section。
- 某张图或图注是否被正确提取并绑定到对应来源位置。
- 当同一章节重生成时，系统是否还能优先复用同一批高质量证据。

## 非目标

本轮明确不做以下事项：

- 不做前台来源展示。
- 不在阅读页展示页码回溯、原文跳转、证据面板。
- 不做图片语义理解。
- 不做图文联合 embedding。
- 不让教材生成模型直接消费图片二进制。
- 不引入新的 server-first 常驻后端架构。
- 不重写现有教材生成主流程。

## 方案概览

采用一个窄的 `PDF parse adapter` 层，将外部 PDF 解析结果转换为 GGlearn 自己的标准化快照结构，再由现有 `sourceTransform` 逻辑继续生成 source asset。

数据流如下：

`PDF file`
-> `PDF parse adapter`
-> `ProjectSource`
-> `ProjectSourceSnapshot`
-> `ProjectSourceAsset`
-> `EvidencePack`
-> `Chunk generation`

这个链路中：

- parser adapter 只负责结构抽取和标准化，不负责教材业务判断。
- source asset 仍然是教材生成唯一主边界。
- retrieval 和 evidence 仍然服务教材生成，不向用户暴露。

## 为什么不直接复用 Open-NotebookLM 整套架构

Open-NotebookLM 的可借鉴价值主要在 PDF 来源处理层，而不是其整套运行时。

可以借的部分：

- MinerU 风格的 PDF 解析产物组织方式。
- 原始文件、markdown、图片资源、结构化中间产物的分层保存。
- 图像目录、content list、解析目录等来源级元数据保留方式。

不应直接搬入的部分：

- FastAPI + workflow engine 主骨架。
- 面向通用问答的上下文拼接策略。
- 以 vector store / QA 为中心的主生成范式。

原因是 GGlearn 的目标不是做通用 notebook 知识库，而是做 source-asset-first 的教材生成系统。

## 第一阶段必须保留的 PDF 信息

第一阶段进入 GGlearn 标准结构的数据至少包括：

1. 正文文本
2. section 层级或 section 序列
3. 页码和锚点
4. 图片资源引用
5. 图注或表注
6. 公式块
7. 解析质量警告

注意：这里的“保留图片资源引用”不是让生成阶段立刻看图，而是保证图片与图注不丢失，并可在内部被引用。

## 数据结构调整

### 1. `ProjectSource`

保持现有职责，继续作为原始来源容器。新增内容应尽量只保留“来源级引用”，避免把业务解释塞进该层。

建议增加或规范的字段方向：

- `parseArtifactRef`：指向解析产物目录或逻辑标识。
- `parseProvider`：记录由哪种 PDF 解析器生成。
- `parseVersion`：记录解析结构版本，便于后续迁移。

这层仍不承接教材业务语义。

### 2. `ProjectSourceSnapshot`

这是本轮最关键的扩展点。当前已有：

- `rawText`
- `sections`
- `anchors`
- `pageCount`
- `quality`

建议新增以下结构化快照字段：

- `figureAssets`
  - `id`
  - `pageNumber`
  - `anchorId`
  - `imagePath` 或逻辑资源引用
  - `captionText`
  - `label`
- `formulaSpans`
  - `id`
  - `pageNumber`
  - `anchorId`
  - `latex` 或公式文本
  - `rawText`
- `captionSpans`
  - `id`
  - `pageNumber`
  - `anchorId`
  - `text`
  - `kind`，如 `figure`、`table`

这里仍是“快照层”，只承载来源事实，不做教学判断。

### 3. `ProjectSourceAsset.structureLayer`

当前已有字段：

- `sectionTree`
- `paragraphBlocks`
- `formulaBlocks`
- `figureCaptionBlocks`
- `noiseBlocks`
- `normalizationNotes`

本轮要求：

- 不再让 `formulaBlocks` 继续为空占位。
- 不再让 `figureCaptionBlocks` 继续为空占位。
- 新增一个轻量引用结构，用于把图像资源与 caption、page、anchor 绑定。

建议新增：

- `figureRefs`
  - `id`
  - `anchorId`
  - `pageNumber`
  - `captionBlockId`
  - `assetRef`

该层的定位是“已可被教材系统消费的结构化资产”。

### 4. `retrievalLayer`

第一阶段不引入新的多模态检索，只做最小增强：

- 允许 retrieval unit 的摘要或 hint 感知公式块和图注块。
- 允许 evidence snippet 引用图注或公式文本。
- 保持主检索对象仍以文本证据为主。

换句话说，图和公式先变成“可被引用的结构化文本证据”，而不是视觉语义对象。

## `PDF parse adapter` 的职责

新增一层窄 adapter，其职责仅为：

1. 接收 PDF 解析器产物。
2. 提取正文、section、页码、图像引用、图注、公式。
3. 生成 GGlearn 标准 snapshot payload。

adapter 不负责：

- 生成 retrieval units
- 生成 concept index
- 生成 chapter seeds
- 生成教材 prompt
- 生成任何面向用户的内容

这能保证来源解析和教材业务边界清晰。

## 内部证据绑定策略

本轮证据绑定仅作为系统内部约束。

它主要服务以下场景：

- chunk 生成时只使用给定 evidence packs
- 章节重生成时复用稳定证据
- 检测 unsupported claims
- 判断某章节是否证据不足
- 绑定某个 diagram / formula / explanation 对应的来源位置

前台不展示这些绑定信息。阅读页继续保持“只呈现教材成品”的原则。

## 阅读端策略

阅读端继续保持极简：

- 不展示页码来源
- 不展示证据编号
- 不展示原文摘录
- 不展示“点击查看原文”

如果未来需要内部调试能力，应单独提供仅开发使用的 debug 视图，不进入默认学习流。

## 错误处理

PDF 结构抽取是高不确定性环节，因此需要明确定义降级方式，但不能让降级污染业务逻辑。

处理原则：

1. 优先保留结构化结果。
2. 无法结构化时，退回纯文本 snapshot。
3. 结构提取失败要留下清晰 warning，而不是静默吞掉。
4. 即使图片、图注、公式提取失败，也不能阻断基础文本进入 source asset。

建议的 warning 示例：

- `flat-section-structure`
- `missing-page-count`
- `figure-extraction-failed`
- `formula-extraction-failed`
- `caption-mapping-uncertain`

## 验证策略

本轮验证围绕“内部可控性”而不是“前台展示”。

至少应覆盖以下验证：

1. 输入含 section 标题的 PDF，snapshot 能生成稳定的 `sections` 和 `anchors`。
2. 输入含图片与图注的 PDF，snapshot 能保留 `figureAssets` 和 `captionSpans`。
3. 输入含公式的 PDF，asset 能填充 `formulaBlocks`。
4. `buildProjectSourceAsset()` 不再始终输出空的 `formulaBlocks` 与 `figureCaptionBlocks`。
5. 现有教材生成主流程不因新增字段而回归失败。
6. 阅读页默认仍不暴露来源信息。

## 分阶段实施建议

### Phase 1

- 新增 `pdf parse adapter`
- 扩展 `ProjectSourceSnapshot`
- 接通图、图注、公式进入 snapshot
- 填充 `ProjectSourceAsset.structureLayer`
- 不改前台

### Phase 2

- 让 retrieval/evidence 更稳地吸收公式和图注
- 让 chunk 生成更稳定引用公式类证据
- 加强 unsupported-claim 校验

### Phase 3

- 评估是否需要内部 debug 面板
- 评估是否需要多模态理解，但前提仍是“不打扰用户前台”

## 推荐结论

最优路径不是重建一套 NotebookLM 式底层，而是借其 PDF 来源处理经验，补强 GGlearn 自己的 source asset 结构化入口。

第一阶段的成功标准是：

- 图、图注、公式不丢
- 它们能进入 GGlearn 内部资产结构
- 教材生成仍然只暴露成品，不暴露来源
- 系统内部对证据绑定更稳，前台体验仍然干净

这条路径最符合 GGlearn 当前的产品方向和架构边界。
