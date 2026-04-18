# GGlearn 原始资料获取与 Research Strategy 设计

## 1. 文档目的

本文用于正式定义 GGlearn 中“原始资料获取”这一层的重构方向，并补齐其上游的 `Research Strategy` 设计。

本次设计解决的不是界面布局问题，而是更基础的系统问题：

- 用户的学习意图如何转成可执行的资料发现任务
- 网络搜索、直接 URL、文件导入、粘贴文本如何进入同一套清晰的资料获取模型
- 搜到、抓到、抽到正文、进入后续资料处理，这几个阶段如何严格区分
- `fast-research` 和 `deep-research` 如何在终局上走向研究代理型，同时在当前阶段保持最优路径实现

本文是产品与系统共用的结构设计文档，后续实现规划应以本文为准。

## 2. 一句话结论

GGlearn 的原始资料获取应重构为一个由 `Learning Brief` 驱动的双层系统：

`Learning Brief -> Research Strategy Layer -> Acquisition Layer -> Material Processing`

其中：

- `Research Strategy Layer` 负责围绕学习意图发现、筛选、组织 URL 候选项
- `Acquisition Layer` 负责把文件、文本、URL 候选项真正拿进系统，并生成可进入后续资料处理的正式原始材料

## 3. 核心设计原则

### 3.1 用户输入的首先是学习意图，不是搜索词

用户输入往往是宽泛甚至随意的，例如：

- 我想学雅可比矩阵
- 我想搞懂线性回归到底在做什么

因此，GGlearn 不能把用户原话直接等同于最终 search query。

GGlearn 的上游输入应先被建模为 `Learning Brief`，而不是“搜索框内容”。

### 3.2 `web-search` 不是一种资料类型

`web-search` 的本质不是“网页资料”这种新材料类型，而是 `URL 发现方式`。

因此：

- `direct-url` 是用户直接提供目标 URL
- `web-search` 是系统先发现一批 URL，再把它们交给后续获取链路

二者下游必须共用同一条 URL acquisition pipeline。

### 3.3 搜索结果默认不是原始资料

搜索引擎返回的标题、snippet、命中摘要，只能视为候选线索，不能直接升级为原始资料。

只有在真正抓到远程内容、并成功抽取出可用正文后，才能进入后续资料处理主流程。

### 3.4 原始资料获取只负责“拿进来并记清楚”

这一层的职责边界必须非常硬：

- 要做：发现入口、抓取、保留原始快照、抽取正文、记录 provenance
- 不做：可信度定论、教学价值判断、证据抽取、教材生成

### 3.5 `fast` 与 `deep` 共享同一主链路

`fast-research` 和 `deep-research` 不应建成两套系统。

它们共享同一套对象模型和主流程，差异只在研究深度、预算和组织深度。

## 4. 顶层结构

### 4.1 全局主链条

重构后的上游主链条应为：

`Learning Brief -> Research Strategy Layer -> Acquisition Layer -> Material Processing`

解释如下：

- `Learning Brief`
  负责表达用户到底想学什么、想学到什么程度，以及必要的背景约束。
- `Research Strategy Layer`
  负责围绕学习意图组织搜索和资料发现。
- `Acquisition Layer`
  负责把候选入口真正拿进系统，并生成正式原始材料。
- `Material Processing`
  负责在正式原始材料之上做结构化处理、依据组织和教材生成准备。

### 4.2 Research Strategy Layer 的职责

Research Strategy Layer 负责：

- 把学习意图转成 research 任务
- 生成基础搜索意图
- 发起 `web-search`
- 去重候选 URL
- 做轻量可获取性筛选
- 对少量候选做预抓
- 把整理好的候选结果交给 acquisition

Research Strategy Layer 不负责：

- 保存网页快照
- 正文抽取
- 可信度定论
- 教材生成

### 4.3 Acquisition Layer 的职责

Acquisition Layer 负责：

- 接收 `local-file`、`pasted-text`、`direct-url`、`web-search` 发现结果
- 执行远程内容抓取
- 保留原网页或原始远程内容快照
- 从快照中抽取正文
- 生成可进入后续资料处理的正式原始材料

Acquisition Layer 不负责：

- 搜索策略规划
- 教学价值判断
- 教材组织与讲解生成

## 5. 输入模型

### 5.1 Learning Brief

`Learning Brief` 是 GGlearn 主链条的上游输入，不只是给搜索用，也服务后续教材生成。

当前确定的最小骨架如下：

- 必填：
  - 学习主题
  - 学习目标
- 可选：
  - 用户身份 / 学习阶段
  - 已有基础
  - 讲解偏好
- 默认画像：
  - 通用成人初学者

字段规则：

- `用户身份 / 学习阶段` 为可选自由输入
- `已有基础` 为可选固定档位
- `讲解偏好` 为可选固定档位

### 5.2 Research Brief

`Research Brief` 是 `Learning Brief` 在 research layer 内部的投影。

它回答的问题是：

`为了找到合适资料，这次 research 任务到底要找什么。`

当前建议至少包含：

- 学习主题
- 学习目标
- 输出语言
- research 模式：`fast` / `deep`

关键原则：

- 用户输入首先是学习意图，而不是最终 query
- 但 Phase 1 不引入重型 query planner
- Phase 1 默认仅使用 `学习主题 + 学习目标` 生成基础搜索意图

## 6. 输入入口

GGlearn 的原始资料输入应统一建模为四类：

- `local-file`
  - 当前以 PDF 为主
- `pasted-text`
  - 用户直接粘贴正文内容
- `direct-url`
  - 用户直接提供 URL
- `web-search`
  - 系统先发现 URL，再交给 acquisition

具体规则如下：

### 6.1 local-file

`local-file` 当前主路径是 PDF 导入。

它直接进入 acquisition，不经过 research layer 的 URL 发现逻辑。

### 6.2 pasted-text

`pasted-text` 直接视为正式原始材料。

但它的 provenance 必须明确标记为：

- `user-pasted`

它不是“抓取到的外部资料”，而是“用户主动注入的原始文本材料”。

### 6.3 direct-url

`direct-url` 由用户直接给定 URL。

它不需要 research layer 先发现 URL，但从进入获取流程开始，应与 `web-search` 共用同一条 acquisition pipeline。

### 6.4 web-search

`web-search` 不是材料类型，而是 URL 发现方式。

它的产物首先是一批 `UrlLead`，而不是正式原始材料。

## 7. 对象模型

### 7.1 Research Layer 对象

Research Layer 的终局对象建议控制在四个核心对象内。

#### 7.1.1 ResearchBrief

表示一轮 research 任务的输入任务单。

#### 7.1.2 SearchIntent

表示一次具体的搜索意图，即系统最终要向搜索引擎发出的搜索任务。

Phase 1 规则：

- 只支持一个主 `SearchIntent`
- 允许在结果差时补一次另一语言搜索

#### 7.1.3 SearchRun

表示一次实际搜索执行的记录。

它应记录：

- 针对哪个 `SearchIntent`
- 使用什么搜索语言
- 返回了哪些候选结果
- 是否命中不足
- 是否触发补搜

#### 7.1.4 ResearchBatch

表示一轮 research 最终整理出的候选结果交付包。

终局下，它应包含：

- 去重后的 `UrlLead`
- 预抓状态
- 简单覆盖信息
- 候选组织结果

Phase 1 规则：

- `ResearchBatch` 可以先弱化
- 只需能稳定整理出一批 `UrlLead`

### 7.2 Acquisition Layer 对象

#### 7.2.1 UrlLead

表示一个待获取的 URL 候选项。

无论它来自：

- 用户直接输入 URL
- 搜索结果发现 URL

都应先汇合为同一种对象。

关键原则：

- `UrlLead` 不是原始资料
- `UrlLead` 只是候选入口

#### 7.2.2 FetchAttempt

表示一次实际的远程获取尝试。

之所以需要单独对象，是因为同一个 `UrlLead` 可能有多次抓取尝试。

#### 7.2.3 RawCapture

表示成功拿进系统的原网页或原始远程内容快照。

对于 URL 类输入，系统默认必须保留这一层。

#### 7.2.4 AcquiredMaterial

表示从 `RawCapture` 中抽取出来、可进入后续资料处理的正文材料。

关键原则：

- URL 抓取成功不等于材料可用
- 只有成功生成可用的 `AcquiredMaterial`，才算正式完成 acquisition

## 8. 状态机

### 8.1 Research 状态

Research Layer 建议使用以下状态：

- `idle`
- `planning`
- `searching`
- `screening`
- `pre-capturing`
- `ready-for-acquisition`
- `insufficient-results`
- `completed`

关键原则：

- `completed` 只表示研究发现阶段结束
- 不表示正式原始材料已入库

### 8.2 UrlLead 状态

`UrlLead` 建议使用以下状态：

- `discovered`
- `screened-out`
- `queued`
- `fetching`
- `captured`
- `fetch-failed`
- `abandoned`

关键原则：

- 搜索结果中的 title/snippet 再完整，也只能停留在 `UrlLead`
- 不得越级进入正式原始材料

### 8.3 AcquiredMaterial 状态

`AcquiredMaterial` 建议使用以下状态：

- `pending-extraction`
- `extracting`
- `ready`
- `partial`
- `extract-failed`

关键规则：

- `captured != ready`
- 只有 `ready` 才能自动进入资料处理主流程
- `partial` 可以保留，但默认不自动进入主流程

## 9. 搜索策略规则

### 9.1 基础搜索输入

Phase 1 中，`web-search` 的基础搜索输入固定为：

- 学习主题
- 学习目标

学习目标进入 query 的规则为：

- 直接拼入 query
- 不做额外模板映射

### 9.2 默认搜索语言

默认搜索语言跟系统输出语言走。

也就是：

- 目标输出中文教材，则默认先用中文搜索语言发 query
- 目标输出英文教材，则默认先用英文搜索语言发 query

这里的“搜索语言”定义为：

- 系统向搜索引擎发 query 时使用的语言

### 9.3 结果差时的补搜规则

当第一次搜索结果差时，系统自动补一次另一语言搜索。

“结果差” 的判断同时看两件事：

- 候选 URL 太少
- 预抓成功率太低

补搜规则如下：

- 只补一次
- 只换搜索语言
- 不重写搜索意图
- 不额外做复杂 query planner

### 9.4 搜索阶段的 AI 边界

搜索阶段的 AI 只做：

- 候选 URL 去重辅助
- 可获取性筛选
- 预抓优先级选择

搜索阶段的 AI 不做：

- 可信度定论
- 教学价值判断
- 教材规划

## 10. fast-research 与 deep-research

### 10.1 终局定义

`fast-research` 和 `deep-research` 共享同一套对象模型和主流程，不是两套不同系统。

共同主链路：

`Learning Brief -> ResearchBrief -> SearchRun -> UrlLead -> Fetch -> RawCapture -> AcquiredMaterial`

两者差异只在：

- 搜索意图数量
- 搜索轮次
- 搜索预算
- 预抓预算
- 结果组织程度

### 10.2 Phase 1 产品策略

Phase 1 产品上直接露出：

- `fast-research`
- `deep-research`

但实现上共享同一条 pipeline。

### 10.3 Phase 1 的 fast

`fast` 的目标是尽快拿到一批够用的资料入口。

特征如下：

- 搜索预算较小
- 预抓预算较小
- 更快停止

### 10.4 Phase 1 的 deep

`deep` 在 Phase 1 中不是完整研究代理，而是同链路下的更深模式。

特征如下：

- 搜索预算更大
- 预抓预算更大
- 停止阈值更高
- 有预算上限
- 但如果已经拿到足够多 `ready` 材料，可以提前停止

关键原则：

- Phase 1 的 `deep` 必须真实放大预算
- 但不引入复杂多轮自主研究逻辑

## 11. Phase 1 最小落地范围

Phase 1 的目标是跑通最小闭环，而不是一次做完终局能力。

Phase 1 必做内容：

- 支持四类输入：
  - `local-file`
  - `pasted-text`
  - `direct-url`
  - `web-search`
- `pasted-text` 直接登记为正式原始材料
- `direct-url` 与 `web-search` 共用 URL acquisition pipeline
- `web-search` 基于 `学习主题 + 学习目标`
- 默认按输出语言搜索
- 结果差时自动补一次另一语言搜索
- AI 只做可获取性筛选
- 对少量候选做预抓
- URL 获取成功后保留：
  - `RawCapture`
  - `AcquiredMaterial`
- 只有 `AcquiredMaterial.ready` 进入后续资料处理

Phase 1 明确不做：

- 多子问题拆分
- 多轮自主研究
- 重型 research package
- 复杂 query planner
- 重型可信度评估
- 自动大批量纳入正式原始材料

## 12. Phase 2 / Phase 3 演进路径

### 12.1 Phase 2

Phase 2 的目标是把 `deep-research` 从“预算增强模式”升级为“初步系统研究模式”。

建议补齐：

- 一个 `ResearchBrief` 对应多个 `SearchIntent`
- 有限多轮 `SearchRun`
- 更强的候选去重与站点多样化控制
- 轻量 `ResearchBatch`
  - 表达已有覆盖方向
  - 表达缺失方向
  - 表达已预抓成功的候选

### 12.2 Phase 3

Phase 3 再逐步走向终局研究代理型能力。

建议重点增强：

- 更稳的学习意图理解
- 更系统的 query 扩展
- 更清晰的子问题拆解
- 更成熟的候选排序
- 更完整的 `ResearchBatch`

但即便到终局，也应守住一条硬边界：

- `Research Layer` 负责发现和组织
- `Acquisition Layer` 负责拿进来和记清楚
- 研究层不直接替代用户做最终资料采纳决策

## 13. 最终设计结论

GGlearn 的原始资料获取不应继续维持“上传 PDF + 搜几个网页”的松散模式，而应重构为一个由 `Learning Brief` 驱动的双层系统：

`Learning Brief -> Research Strategy Layer -> Acquisition Layer -> Material Processing`

其中：

- 终局方向是研究代理型的 `deep-research`
- 当前阶段按最小闭环实现
- `fast` 与 `deep` 共用对象模型和主链路
- Phase 1 先以参数分层方式区分两者

这一设计既保留了后续向 NotebookLM 式 research experience 演进的空间，又避免在当前阶段过度设计或职责漂移。
