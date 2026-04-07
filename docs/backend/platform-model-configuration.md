# Platform 模型配置修改指南

最后更新：2026-04-07

本文面向开发者，说明如何修改 `Platform API` 的模型配置，包括默认模型、可选模型、提供商接入边界，以及前后端分别需要改哪些位置。

## 先理解当前架构

当前实现里，`Platform API` 不是一套独立的“后台模型配置中心”。它复用了前端的 `selectedModel` 选择结果，再由后端在 `platform` 模式下改用服务端密钥执行请求。

- 前端负责让用户选择 `providerId / modelId / endpointPreset`
- 前端把这些字段连同 `access.mode = "platform"` 一起发给 `/api/generate`
- 后端根据 `providerId + endpointPreset` 选择服务端密钥，而不是读取浏览器里的 BYOK 密钥
- 因此，修改 platform 模型配置通常不是只改后端，而是“前端可选项 + 后端密钥解析”一起看

关键代码入口：

- 前端模型配置：[SlideTutor-AI/src/config/models.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/src/config/models.ts)
- 前端设置页：[SlideTutor-AI/src/components/SettingsModal.tsx](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/src/components/SettingsModal.tsx)
- 前端请求组装：[SlideTutor-AI/src/lib/api/apiClient.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/src/lib/api/apiClient.ts)
- 前端持久化：[SlideTutor-AI/src/store/uiStore.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/src/store/uiStore.ts)
- 后端密钥与提供商解析：[SlideTutor-AI/api/lib/env.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/api/lib/env.ts)
- 后端生成主链路：[SlideTutor-AI/api/lib/generateService.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/api/lib/generateService.ts)
- 环境变量样例：[SlideTutor-AI/.env.example](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/.env.example)

## 决策树

先判断你要改的是哪一类：

1. 只想替换现有模型 ID
2. 只想修改默认平台模型
3. 想新增一个现有提供商下的可选模型
4. 想新增一个新的 OpenAI-compatible 预设提供商
5. 想新增一个全新的提供商类型

下面按这五种情况分别说明。

## 情况 1：只替换现有模型 ID

例子：

- 把 `gemini-2.5-flash` 换成 `gemini-2.5-pro`
- 把某个 `Qwen` 型号换成新的 `Qwen` 型号

你通常只需要改前端模型清单文件：

- [models.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/src/config/models.ts)

主要改动点：

- 在 `MODEL_CONFIG.providers[].models[]` 里改对应模型的 `id`
- 如果要改默认选中项，再同步修改 `DEFAULT_SELECTED_MODEL`

为什么通常不用改后端：

- 后端不会维护一份“允许的具体 modelId 白名单”
- 后端会把前端传来的 `modelId` 原样传给 Gemini 或 OpenAI-compatible SDK
- 只要当前提供商的服务端密钥仍然有效，且该 `modelId` 在供应商侧真实存在，就能工作

注意：

- 如果你把模型 ID 改错，编译不一定报错，但运行时会在供应商调用阶段失败
- 所以这种改动一定要做一次真实请求验证

## 情况 2：只修改默认平台模型

当前默认模型定义在：

- [models.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/src/config/models.ts#L44)

操作步骤：

1. 修改 `DEFAULT_SELECTED_MODEL`
2. 如果默认值是 `openai-compatible`，记得同时带上 `endpointPreset`
3. 本地启动后，用一个“全新浏览器会话”验证默认值

示例：

```ts
export const DEFAULT_SELECTED_MODEL: SelectedModel = {
  providerId: 'openai-compatible',
  modelId: 'qwen3.5-plus',
  endpointPreset: 'qwen',
};
```

非常重要：

- 这个默认值只影响“没有保存过 `slide_tutor_model` 的用户”
- 已有用户的模型选择会被本地持久化覆盖
- 持久化逻辑在 [uiStore.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/src/store/uiStore.ts)

这意味着：

- 如果你只是改 `DEFAULT_SELECTED_MODEL`，老用户不会自动切到新模型
- 如果你希望所有已有用户都迁移到新默认值，需要额外做存储迁移逻辑，而不是只改一个常量

## 情况 3：新增一个“现有提供商”下的可选模型

例子：

- 给 Gemini 新增 `gemini-2.5-flash-lite`
- 给 Qwen 预设新增 `qwen-max-latest`
- 给 Doubao 预设新增新的 seed 型号

操作步骤：

1. 在 [models.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/src/config/models.ts#L115) 的 `MODEL_CONFIG` 中加入新模型
2. 如果它属于 `openai-compatible`，写上正确的 `endpointPreset`
3. 如果希望它成为默认模型，再同步改 `DEFAULT_SELECTED_MODEL`
4. 验证前端设置页能选到它，且请求体里带对了 `providerId / modelId / endpointPreset`

Gemini 示例：

```ts
{
  id: 'gemini-2.5-flash-lite',
  name: 'Gemini 2.5 Flash Lite',
  vision: true,
}
```

Qwen 示例：

```ts
{
  id: 'qwen-max-latest',
  name: 'Qwen Max',
  vision: true,
  endpointPreset: 'qwen',
}
```

这类改动通常不需要改后端代码，但有两个前提：

- 提供商类型没变，仍然是 `gemini` 或 `openai-compatible`
- 对应服务端密钥已经存在，且该模型在供应商侧可用

## 情况 4：新增一个新的 OpenAI-compatible 预设提供商

例子：

- 现在只有 `qwen` 和 `doubao`
- 你想再加一个固定预设，比如 `deepseek`

这类改动需要前后端一起改。

### 前端要改什么

先改 [models.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/src/config/models.ts)：

1. 扩展 `OpenAiCompatibleEndpointPreset` 类型
2. 在 `OPENAI_COMPATIBLE_ENDPOINTS` 里加入新 provider 的 `label + baseURL`
3. 在 `MODEL_CONFIG` 里挂上该 provider 下的一个或多个模型

再改 [SettingsModal.tsx](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/src/components/SettingsModal.tsx#L357)：

1. 在 `Endpoint Preset` 下拉框里加入新 `<option>`
2. 保证切换预设时，`baseURL` 能自动切到你新配置的 `baseURL`

### 后端要改什么

改 [env.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/api/lib/env.ts)：

1. 扩展 `SupportedProviderId`
2. 扩展 `OpenAiCompatibleEndpointPreset`
3. 在 `OPENAI_COMPATIBLE_BASE_URLS` 加入新 provider 的服务端 `baseURL`
4. 在 `getProviderApiKeyName()` 里把新 provider 映射到新的环境变量名
5. 在 `isLegacyOpenAiProviderId()` 里允许它被识别成 OpenAI-compatible 预设
6. 检查 `resolveProviderAccess()` 是否能正确走到“平台模式用服务端密钥”的分支

再改环境变量样例：

- [SlideTutor-AI/.env.example](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/.env.example)

例如新增：

```env
DEEPSEEK_API_KEY=""
```

然后把 Cloudflare Worker 生产环境也配上对应密钥。

### 这类改动最容易漏掉的点

- 只改了前端下拉框，没改后端 `env.ts`
- 只改了 `MODEL_CONFIG`，没改 `Endpoint Preset` 下拉框
- 本地 `.env` 配了密钥，但 Cloudflare 生产环境没配
- 新 provider 实际是 OpenAI-compatible，但你没有把它接进 `resolveProviderAccess()`

## 情况 5：新增一个全新的提供商类型

例子：

- 新增 `anthropic`
- 新增 `openrouter`
- 新增“非 OpenAI-compatible 且非 Gemini”的自定义 SDK 接入

这已经不是“改配置”了，而是“加能力”。

你至少要改这些位置：

1. [models.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/src/config/models.ts)
2. [apiClient.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/src/lib/api/apiClient.ts#L117)
3. [env.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/api/lib/env.ts#L133)
4. [generateService.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/api/lib/generateService.ts#L86)
5. [generateService.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/api/lib/generateService.ts#L659)
6. [SlideTutor-AI/.env.example](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/.env.example)

你需要新增的能力通常包括：

- 新 provider 类型定义
- 前端凭据表单和请求体组装
- 服务端密钥解析
- 实际 SDK 调用实现
- 错误语义归一化
- 对应测试

如果只是想让 platform 能切一个新供应商，优先考虑“把它接成 OpenAI-compatible 预设”而不是从零加第三类 provider。这样改动面会小很多。

## 当前产品边界与限制

下面这些限制是现在代码里明确存在的，不是文档建议。

### 1. Platform 当前不支持 custom OpenAI-compatible

平台模式下，`custom` 会被后端拒绝：

- [env.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/api/lib/env.ts#L153)
- [generateService.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/api/lib/generateService.ts#L578)

原因很直接：

- `custom` 依赖用户自己填 `baseURL + apiKey`
- 这符合 `My API`
- 但不符合当前 `Platform API` 的“服务端托管密钥”边界

如果你想让 platform 支持某个新供应商，不要走 `custom`，而要把它做成“平台认可的固定 preset”。

### 2. 平台和 BYOK 共用同一个模型下拉框

当前设置页的模型下拉框是共享的：

- [SettingsModal.tsx](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/src/components/SettingsModal.tsx#L288)

这意味着：

- 如果你在 `MODEL_CONFIG` 里加入一个只适用于 `My API` 的模型
- platform 用户也可能在 UI 上看到并选到它
- 后端最后会报错，但体验上会显得“能选但不能用”

所以如果你新增的是仅 BYOK 可用的模型，最好同时补一层前端过滤，而不是只靠后端拒绝。

## 推荐操作流程

### 场景 A：只改 platform 默认模型

1. 改 [models.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/src/config/models.ts#L44) 的 `DEFAULT_SELECTED_MODEL`
2. 确认该 provider 的服务端密钥已存在
3. 用全新浏览器会话打开设置页，确认默认值变化
4. 在 `Platform API` 模式下跑一次 `Analyze` 和 `Follow-up`

### 场景 B：新增一个现有 provider 下的模型

1. 改 `MODEL_CONFIG`
2. 本地启动，看设置页下拉框是否出现新模型
3. 切到 `Platform API`，选中新模型
4. 跑一次真实请求，确认供应商侧接受该 `modelId`

### 场景 C：新增新的 OpenAI-compatible 预设 provider

1. 先改前端类型、预设映射、模型列表
2. 再改后端 `env.ts`
3. 补 `.env.example`
4. 配本地 `.env`
5. 配 Cloudflare 生产环境变量
6. 跑测试
7. 做一次本地和一次线上 smoke test

## 验证清单

代码改完后，至少做这几步：

1. 在 `SlideTutor-AI` 目录运行 `npm run lint`
2. 运行 `npm test -- src/lib/api/apiClient.test.ts src/components/SettingsModal.test.tsx api/lib/generateService.platform.test.ts`
3. 如果改到了 Worker 鉴权或 platform 路由，再运行 `npm run test:workers -- test/workers/platform-generate.worker.test.ts`
4. 本地打开设置页，确认模型下拉框、预设切换、默认值都符合预期
5. 在 `Platform API` 模式下真实跑一次 `Analyze`
6. 再跑一次 `Follow-up` 或 `Quiz`
7. 如果改了提供商，去 Cloudflare 检查对应环境变量是否已同步

## 最后一句

在这个项目里，“改 platform 模型配置”本质上分两层：

- 前端决定“允许选什么”
- 后端决定“服务端实际上用什么密钥和 baseURL 去调用”

只改其中一层，通常都会留下坑。
