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
## Phase 08 Parser Notes

- `Platform API` keeps a platform-managed `Volcengine` parser path.
- `Platform API` does not expose parser configuration in settings.
- `My API` may optionally configure `LlamaParse`.
- If `My API` omits parser config, `explain` keeps the no-parser degraded analysis path.
- Parser errors are now separated into `ROUTE_RATE_LIMITED`, `PLATFORM_PARSER_*`, and `BYOK_PARSER_*`.

## 2026-04-11 Optional Parser Onboarding Note

For `My API`, parser setup now includes lightweight onboarding guidance directly in Settings:

- `Optional Parser` help explains that parser setup is optional and degraded no-parser analysis still exists
- when users want `LlamaParse`, the help surface links them to `LlamaCloud`
- the help copy now gives a short 3-step flow:
  1. sign in or create a `LlamaCloud` account
  2. open `API Key` in the left sidebar and create a new key
  3. copy the `llx-...` key back into Settings

This is intentionally a UI-only guidance improvement. It does **not** change parser routing, parser ownership, or BYOK parser validation rules.

## 2026-04-10 Phase 09 Model Capability Registry Notes

## 2026-04-12 BYOK OpenAI-Compatible Settings Notes

This section overrides several older assumptions about `My API` OpenAI-compatible configuration.

### Current frontend contract

- `Select Model` is now the only model-selection entry point in Settings.
- Built-in `Gemini` selections expose only the `Gemini API Key` field.
- Built-in `OpenAI-compatible` selections expose only the `OpenAI-Compatible API Key` field.
- Only `Custom OpenAI-compatible` exposes the full editable runtime tuple:
  - `API Key`
  - `Base URL`
  - `Model ID`

### Current persistence contract

OpenAI-compatible BYOK credentials are now separated by preset instead of sharing one runtime key.

Current persisted fields in [models.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/src/config/models.ts) and [uiStore.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/src/store/uiStore.ts) include:

- `qwenApiKey`
- `doubaoApiKey`
- `customApiKey`
- `customBaseURL`
- `customModelId`

The older shared `openAiCompatible.apiKey` now exists only as a legacy migration source and should not be treated as current runtime truth.

### Current custom-model identity contract

For `Custom OpenAI-compatible`, the selected catalog item stays pinned to the sentinel model id:

```ts
{
  providerId: 'openai-compatible',
  endpointPreset: 'custom',
  modelId: 'custom-openai-model'
}
```

The real runtime identity now lives in persisted BYOK fields:

- `customModelId`
- `customBaseURL`
- `customApiKey`

Do not write the real custom model id back into `selectedModel.modelId`. That older pattern caused the settings select value to stop matching any option and visually fall back to the first model.

### Current capability-check contract

`POST /api/model-capability-check` now has a more actionable custom-model path:

- built-in known models still resolve from the shared capability registry
- custom OpenAI-compatible models can now probe into a real `usable` state
- auth-style failures map to `MODEL_CAPABILITY_CHECK_AUTH_FAILED`
- unsupported-feature/protocol failures map to `MODEL_CAPABILITY_CHECK_UNSUPPORTED`
- transient failures still map to `MODEL_CAPABILITY_CHECK_FAILED`

This means `Compatibility check pending` should no longer be the steady-state success path for valid custom configuration.

### Current platform boundary

- `Platform API` still does not support `custom` OpenAI-compatible models
- the shared model picker may still display the custom item, but platform mode disables that option and the backend still rejects custom if it somehow reaches runtime

When you add or change a model now, frontend list updates are not enough.

- Current selectable built-in models live in `SlideTutor-AI/src/config/models.ts`, and backend capability truth for those models is derived from that shared config in `SlideTutor-AI/api/lib/modelCapabilities.ts`.
- A small backend-only legacy capability alias list may still exist for older saved model ids, but new selectable built-ins should not be maintained in a second manual registry.
- Save-time BYOK checks go through `POST /api/model-capability-check`.
- Persisted readiness state lives in `selectedModel + aiAccess + modelCapabilityCheck`, not in raw provider secrets.
- Runtime provider parameters are built from `structuredOutputConfig.ts` plus resolved capability truth, not from task branches alone.
- If you replace a built-in model id in `models.ts`, you must also verify that its capability metadata in the same shared config is still correct.

Current runtime rules to preserve:

- `thinking` is soft only. Gemini `thinkingConfig.thinkingLevel` must be emitted only when the resolved model capability says thinking is supported.
- `native_structured_output`, `streaming`, `image_input`, and `text_generation` remain hard product constraints.
- Gemini `explain` now uses a `6144` structured-output budget to give longer structured teaching artifacts more headroom before `MAX_TOKENS`.
- `distill` now uses a `4096` structured-output budget for Gemini and OpenAI-compatible providers.
- `distill` input may remove packaging-only lines such as `Visual Focus Box` and `Socratic Probe`, but the full explanation artifact for Focus mode quality should stay unchanged.

Current BYOK recheck policy:

- Mark saved capability state `stale` on clear capability/configuration failures such as `MODEL_CAPABILITY_UNKNOWN`, `MODEL_CAPABILITY_UNVERIFIED`, `MODEL_NOT_ELIGIBLE`, or `UNSUPPORTED_PROVIDER_SETTING`.
- Do not mark the model `stale` for `STRUCTURED_OUTPUT_TRUNCATED` alone. That is treated as a structured-output budget/runtime issue, not proof that the model lost eligibility.

Correction for the earlier "Case 1: only replace an existing model id" section:

- That older guidance is no longer accurate after Phase 09.
- Replacing a built-in selectable model id is no longer a frontend-only change.
- The safe rule now is: update the shared model definition in `SlideTutor-AI/src/config/models.ts`, verify backend capability resolution still recognizes it, then run a real request smoke test.

## 内置模型新增操作手册

适用场景：

- 你要给当前产品新增一个新的“内置可选模型”
- 这个模型会出现在设置面板里
- 它属于当前已有 provider 范围，也就是 `gemini` 或 `openai-compatible`

### 结论先说

新增内置模型的唯一主入口是：

- `SlideTutor-AI/src/config/models.ts`

当前实现下，前端可选模型列表和后端内置模型 capability truth 都从这份共享配置出发。不要再去单独维护第二份“当前内置模型 id 列表”。

### 步骤 1：确认这是不是“内置模型新增”

只有下面这种情况，才走本节：

- 只是给现有 `gemini` 或 `openai-compatible` provider 新增一个新的内置可选模型

如果不是，就不要照搬本节：

- 新增一个全新 provider
- 新增一个新的 OpenAI-compatible preset
- 只是修改 BYOK 用户自己填写的 custom model

### 步骤 2：在共享模型配置里新增模型定义

编辑：

- [models.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/src/config/models.ts)

在 `MODEL_CONFIG.providers[].models[]` 中新增模型，并补全它的共享 capability 元数据。

当前新增内置模型至少要明确这些字段：

- `id`
- `name`
- `vision`
- `endpointPreset`
  仅 `openai-compatible` 需要
- `thinking`
  如果 provider 明确不支持或我们不打算使用，就写 `false`
- `nativeStructuredOutput`
  如果不填，当前默认按 `true` 处理；只有明确不支持时才显式写 `false`
- `streaming`
  如果不填，当前默认按 `true` 处理；只有明确不支持时才显式写 `false`
- `capabilityStatus`
  当前内置可用模型通常不需要写；只有像 `custom-openai-model` 这种需要先探测、默认不能直接准入的特殊项才写 `unverified`

示例：

```ts
{
  id: 'gemini-2.5-example',
  name: 'Gemini 2.5 Example',
  vision: true,
  thinking: false,
}
```

OpenAI-compatible 示例：

```ts
{
  id: 'qwen-example',
  name: 'Qwen Example',
  vision: true,
  endpointPreset: 'qwen',
  thinking: false,
}
```

### 步骤 3：按产品硬约束检查 capability 元数据

新增前先确认，这个模型是否满足当前产品硬约束：

- `text_generation`
- `image_input`
- `native_structured_output`
- `streaming`

在当前实现里：

- `text_generation` 对内置模型默认视为 `true`
- `image_input` 由 `vision` 决定
- `native_structured_output` 默认是 `true`，只有明确不支持时才应显式写 `false`
- `streaming` 默认是 `true`，只有明确不支持时才应显式写 `false`
- `thinking` 是软能力，不影响准入，但会影响 Gemini 是否附带 `thinkingConfig`

如果一个模型已知不满足任一硬约束，就不要把它作为当前产品内置可选模型加进来。

### 步骤 4：判断是否需要保留 legacy alias

如果你只是新增一个模型，一般不需要动 legacy alias。

只有下面这种情况，才考虑修改：

- 你替换了一个旧的内置模型 id
- 并且担心已有用户或历史数据里仍保存着旧 id

这时可以在：

- [modelCapabilities.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/api/lib/modelCapabilities.ts)

里的 `LEGACY_MODEL_CAPABILITY_REGISTRY` 保留一条旧 id 映射，让旧选择不会立刻在运行时掉成 `MODEL_CAPABILITY_UNKNOWN`。

注意：

- `LEGACY_MODEL_CAPABILITY_REGISTRY` 只用于兼容旧 id
- 不要把当前仍然在 UI 可选的内置模型继续手工维护在这里
- 当前 UI 可选模型必须只在 `models.ts` 维护

### 步骤 5：检查 provider 侧前提是否真实成立

新增共享模型定义之前，先确认这些前提：

- 这个 `modelId` 在 provider 侧真实存在
- 当前 provider 路径确实支持它
- 如果是 `openai-compatible`，对应的 `endpointPreset` 没填错
- 平台模式下，如果这个模型要给 `Platform API` 用，对应服务端密钥和访问路径已经具备

不要依赖“看起来像是对的”。至少要做一次真实请求验证。

### 步骤 6：运行最小验证

新增内置模型后，至少做下面这些验证：

1. 在 `SlideTutor-AI` 目录运行：
   `npm test -- api/lib/modelCapabilities.test.ts api/lib/modelCapabilityProbe.test.ts api/lib/generateService.platform.test.ts`
2. 运行：
   `npm run lint`
3. 打开设置面板，确认新模型能正常显示
4. 选择该模型跑一次真实请求，至少验证 `analyze`
5. 如有需要，再验证 `followup` 或 `quiz`

其中第 1 步很重要，因为：

- `api/lib/modelCapabilities.test.ts` 里有回归测试，会检查当前所有共享可选模型都不会被 backend 判成 `unknown`

### 步骤 7：什么时候不应该新增为内置模型

下面这些情况，不建议直接作为内置模型加入：

- 已知不支持 native structured output
- 已知不支持 streaming
- 已知不支持图像输入
- provider 侧文档或实测能力不稳定
- 只是为了给 BYOK 用户留一个“也许能用”的实验入口

这类模型更适合：

- 不加入内置列表
- 或继续走 `custom-openai-model` / BYOK 探测路径

### 步骤 8：新增后要同步更新的认知

正常情况下，新增一个当前内置模型，不需要再手动同步第二份当前内置 capability registry。

你真正需要关心的是：

- `models.ts` 里的共享模型定义是否完整
- capability 元数据是否准确
- 是否需要保留旧 id 的 legacy alias
- 真实 provider 请求是否通过
