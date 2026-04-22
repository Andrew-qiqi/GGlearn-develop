# GGlearn MinerU 文档解析接入设计

Date: 2026-04-23
Status: Draft for review

## 1. 文档目的

本设计用于在 GGlearn 当前已经落地的 `parser-first` 来源主链路上，正式接入 `MinerU` 作为文档解析器。

当前 GGlearn 已具备：

- `NormalizedSourceDocument` 作为来源真相层
- `simple-document` / `simple-url` 两个回退 parser
- `sourceImport -> normalizedDocument -> sourceAsset -> projectConcept -> outline -> textbook generation` 的基本主链路

但当前系统仍存在一个关键缺口：

- PDF / DOCX / PPTX 仍未真正使用高质量结构化解析器
- 用户无法在设置页配置 MinerU 服务
- 文档来源仍主要依赖 simple parser 近似映射，无法充分保留标题层级、结构信息和更强的结构锚点

因此，本设计的目标不是继续讨论 parser-first 是否正确，而是明确：

- MinerU 以什么配置方式进入 GGlearn
- MinerU 引擎如何接到现有 parser-first 主链路
- 第一版需要支持哪些能力，暂时不支持哪些能力

## 2. 设计结论

GGlearn 应在当前文档 parser 通道中引入：

- `documentParser` 设置块
- `MineruDocumentEngine`
- `simpleDocumentEngine` 回退策略

并采用以下运行方式：

1. 用户在设置页选择文档解析器为 `simple` 或 `mineru`
2. 当用户选择 `mineru` 时，PDF / DOCX / PPTX 文档优先走 `MineruDocumentEngine`
3. MinerU 解析失败时，系统自动回退到 `simpleDocumentEngine`
4. 回退必须显式记录 warning，不能静默吞掉
5. MinerU 的外部响应格式必须先进入适配层，再映射为 GGlearn 内部的 `NormalizedSourceDocument`

## 3. 本次范围

### 3.1 本次要做的

- 支持用户在设置页配置 MinerU 文档解析
- 文档 parser 支持 `simple | mineru`
- 支持 `baseUrl`
- 支持可选 `apiKey`
- 支持 `timeoutMs`
- 新增 `MineruDocumentEngine`
- 在 `sourceImport.ts` 中根据设置选择文档解析器
- MinerU 失败时自动回退到 `simpleDocumentEngine`
- 把 MinerU 输出映射到 `NormalizedSourceDocument`
- 增加单测、回退测试和一条集成 smoke 验证

### 3.2 本次不做的

- 不同时实现 URL parser provider 平台
- 不接入 `Docling`
- 不增加 parser provider marketplace
- 不做图像/表格资产的深度教学利用
- 不做 bbox 驱动的细粒度 citation UI
- 不在第一版里增加独立的 “测试 MinerU 连接” 页面

## 4. 用户配置模型

MinerU 不应混入现有 `captureConfig`。

原因：

- `captureConfig` 面向的是 URL 获取阶段
- MinerU 面向的是文档解析阶段
- 两者职责不同，混在一起会继续污染设置模型

因此建议新增独立配置块：

```ts
type DocumentParserConfig = {
  provider: 'simple' | 'mineru';
  baseUrl: string;
  apiKey: string;
  timeoutMs: number;
};
```

并将其并入 `AppSettings`：

```ts
type AppSettings = {
  // existing settings...
  documentParser: DocumentParserConfig;
};
```

### 4.1 用户使用方式

设置页新增“文档解析器”区域：

- `Simple (Built-in)`
- `MinerU`

当用户选择 `MinerU` 时显示：

- `Base URL`
- `API Key`，可留空
- `Timeout (ms)`

### 4.2 支持的部署方式

第一版同时支持两种部署方式：

1. 本地 / 自托管 MinerU
   典型形式：
   - 只填写 `baseUrl`
   - 不要求必须有 `apiKey`

2. 远程托管 MinerU 网关
   典型形式：
   - 填写 `baseUrl`
   - 填写 `apiKey`

## 5. 引擎接线方式

### 5.1 新增引擎

新增文件：

- `GGlearn/src/lib/sourceParsers/mineruDocumentEngine.ts`

该文件只负责：

- 构造请求
- 调用 MinerU 服务
- 解析 MinerU 返回
- 将其转换为 `NormalizedSourceDocument`

它不应负责：

- settings 持久化
- UI 呈现
- source asset 构建
- concept aggregation

### 5.2 parser 选择顺序

建议在 `sourceImport.ts` 中增加一层文档 parser 选择函数。

逻辑如下：

- 当 `documentParser.provider === 'simple'`
  - 直接使用 `simpleDocumentEngine`

- 当 `documentParser.provider === 'mineru'`
  - 优先使用 `MineruDocumentEngine`
  - 失败时记录 warning
  - 自动回退到 `simpleDocumentEngine`

### 5.3 回退原则

第一版必须支持自动回退。

原因：

- 用户可能填错 `baseUrl`
- 用户可能没有配置远程服务权限
- 远程服务可能短时不可用
- 如果不回退，导入文档会直接失败，用户体验过硬

但回退不能静默发生。

因此系统应：

- 继续导入
- 继续生成 `NormalizedSourceDocument`
- 在 `warnings` 中写入 `mineru-fallback` 或更具体错误标识
- 在 source workspace 或后续调试界面可见

## 6. MinerU API 适配层

MinerU 的外部返回格式不应直接泄漏到 GGlearn 全局逻辑。

应采用两层适配：

1. `raw MinerU response -> normalized mineru payload`
2. `normalized mineru payload -> NormalizedSourceDocument`

这样做的目的：

- 把外部字段名变化限制在单点
- 降低 MinerU 版本变动对系统其余部分的冲击
- 提高调试能力：可以区分是 MinerU 抽坏了，还是 GGlearn 映射坏了

## 7. 第一版所需最小返回信息

MinerU 第一版接入不要求覆盖全部高级能力，但最少需要：

### 7.1 文档级信息

- `title`
- `language`，可选
- `warnings`，可选

### 7.2 结构正文

- `markdown`
- `sections`
- `blocks`

### 7.3 定位信息

- `anchors`
- 尽量包含 `pageNumber`
- 如果有 `bbox` 更好，但第一版不是硬要求

### 7.4 噪音信息

- 如果 MinerU 能直接给出 discarded / ignored blocks，则直接接入
- 如果不能，GGlearn 仍应从 MinerU 结构输出中尽量分离：
  - 页眉
  - 页脚
  - 页码
  - 重复 heading

## 8. 映射原则

MinerU 映射到 `NormalizedSourceDocument` 时，应坚持以下原则：

### 8.1 `markdown` 是可读主文本

- 用于阅读、调试、fallback
- 不是唯一真相来源

### 8.2 `blocks` 是规划与 retrieval 主输入

- 后续 source asset builder、concept aggregation、evidence mapping 主要消费 `blocks`

### 8.3 `anchors` 是 citation 与 section 绑定主输入

- page / paragraph / section 的来源定位应尽量通过 `anchors` 保存

### 8.4 `discardedBlocks` 是污染调试入口

- 是评估 parser 质量的重要证据
- 不能省略

## 9. 第一版实现顺序

推荐顺序如下：

1. 增加 settings 数据模型与持久化
2. SettingsView 增加文档解析器 UI
3. 新增 `MineruDocumentEngine`
4. `sourceImport.ts` 接入 provider 选择
5. 增加失败回退和 warning 记录
6. 增加单测、回退测试、集成 smoke

这样做的好处：

- 先把配置模型钉住
- 再做 engine 本体
- 最后接 UI 和链路
- 更利于隔离问题来源

## 10. 验证要求

第一版至少应包含以下验证：

### 10.1 单元测试

- `normalizeAppSettings` 能正确读写 `documentParser`
- `MineruDocumentEngine` 请求构造正确
- MinerU 成功返回时能正确映射为 `NormalizedSourceDocument`
- MinerU 失败时会回退到 `simpleDocumentEngine`
- 回退 warning 会被写入结果

### 10.2 集成 smoke

至少需要一条 mock MinerU 返回的 smoke：

- 传入一个 PDF source
- 最终能形成 `snapshot.normalizedDocument`
- `parserEngine === 'mineru'` 或在失败场景下有明确回退标记
- 后续 `buildProjectSourceAsset()` 能正常构建 retrieval units

## 11. 风险与权衡

### 11.1 为什么不是一次做通用 parser 平台

因为当前用户最痛的不是“平台不够通用”，而是：

- PDF 还没有高质量 parser

因此第一版优先解决文档解析质量。

### 11.2 为什么仍然保留 `simple`

因为：

- 服务可用性不可假定
- 用户本地环境差异大
- 第一版必须保证功能不中断

### 11.3 为什么要求显式回退 warning

因为如果系统静默回退，用户会误以为自己已经在用 MinerU。  
这会直接影响质量判断，也不利于后续调试。

## 12. 最终设计判断

GGlearn 第一版 MinerU 接入应采用：

- `documentParser` 独立配置块
- `simple | mineru` 两个文档 provider
- `MineruDocumentEngine` + `simpleDocumentEngine` 回退
- 适配层映射到 `NormalizedSourceDocument`
- 失败回退但状态透明

这是当前最合适的最小落地方式：

- 比简单硬编码更清晰
- 比一次做完整 parser provider 平台更收敛
- 能直接把 MinerU 接到教材生成主链上
