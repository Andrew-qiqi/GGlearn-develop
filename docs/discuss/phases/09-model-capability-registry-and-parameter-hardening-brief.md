# Phase 09 Brief: Model Capability Registry and Parameter Hardening

## Metadata

- Status: Draft
- Phase: 09
- Related Roadmap Entry: `.planning/ROADMAP.md` (next phase proposal)
- Last Updated: 2026-04-10
- Owner: Agent-authored, discussion-in-progress
- Impacts Existing Plans: Yes
- Change Summary: 在 parser 体系收口之后，下一阶段不再继续依赖“前端模型列表 + 后端任务分支 + provider 偶发报错”这种松散组合，而是建立一套后端主导的模型能力真相、硬约束/软约束分层、以及按模型能力生成 provider 参数的稳定执行路径。

## Objective

把 GGlearn 当前活跃 AI 任务的模型能力要求正式收口成一套后端真相，使新增模型、调整默认模型、接入 BYOK custom model 时都不会再因为参数不匹配、能力误判或文档漂移而产生脆弱故障。

## Problem

当前项目的模型配置真相分散在多个位置：

- 前端可选模型列表决定用户“能选什么”
- 后端 provider/access 解析决定请求“往哪里发”
- 结构化输出与 Gemini thinking 配置按 task 硬编码
- 真实 provider 错误直到运行时才暴露

这已经导致了明确问题：

- `gemini-2.5-flash` 在 `distill` / `regenerate_chunk` 上被错误地附加 `thinkingLevel`
- `distill` 等结构化任务会因过紧或不合适的 `maxOutputTokens` 策略触发 `MAX_TOKENS`，进而返回不完整 JSON
- 模型能力没有一份集中定义，新增或调整模型时容易漏改
- 当前活跃任务依赖的结构化输出、视觉输入、流式响应等约束没有被正式收口
- custom BYOK 模型没有清晰的“验证 -> 可用/不可用”准入路径

如果这一轮不处理，后续会继续出现：

- 新模型能在 UI 里选到，但运行时才发现参数不兼容
- 同一个 provider family 下的不同模型需要不同参数控制，却只能靠 if/else 补丁
- 文档、测试、前端和后端对“某模型到底是否可用”的判断继续漂移

## In Scope

- 盘点当前活跃 AI 任务，并将当前 capability baseline 明确收口为：
  - `explain`
  - `distill`
  - `followup`
  - `regenerate_chunk`
  - `regenerate_followup`
  - `generate_questions`
  - `evaluate_answers`
- 明确 `evaluate_note` 不进入模型能力准入基线
  - 它在代码中的残留视为待清理遗留，而不是当前产品真实需求
- 为当前产品定义一套全局模型硬约束与软约束
- 建立后端主导的模型能力注册表/能力画像结构
- 将 Gemini / OpenAI-compatible 的运行时参数构建改为基于模型能力 + 任务策略生成
- 明确当前产品对 `native structured output`、vision、streaming 的要求
- 重新梳理结构化任务上的 `maxOutputTokens` / thinking / 输出长度约束，避免关键 JSON 被截断
- 将 `distill` 截断治理纳入主线方案，但不得以牺牲 `quickExplain` / Focus mode 内容质量为代价
- 为 BYOK custom model 定义 capability probe 与准入状态流转
- 为 provider 错误建立更稳定的能力不匹配归因与错误码边界
- 补齐相关测试、开发文档和模型配置修改指北

## Out of Scope

- 重新设计 `My API` / `Platform API` 的产品路径
- 立刻新增新的 provider family
- 改写教学 prompt 目标或课堂风格
- parser、payment、auth 的进一步扩展
- 用户手动填写模型能力表
- `evaluate_note` 等死任务清理与 hosted 任务面收口
  - 这些放入紧随其后的 Phase 10

## Inherited Project Decisions

- `My API` 和 `Platform API` 仍然是明确分离的两条产品路径
- 模型选择仍然允许用户自行决定，而不是平台强制固定单一模型
- 当前产品主链路依赖结构化 artifact，而不是自由文本后处理
- 成熟主链路 `explain -> distill -> follow-up / quiz` 不应被破坏
- parser、auth、credits、payment 已在前序 phase 中分别收口，不应在本 phase 内重新打开

## Phase-Specific Locked Decisions

- 本 phase 只维护一套全局模型能力真相，不再按 `My API` / `Platform API` 各自维护模型准入标准
- 模型能力与产品策略分离：
  - “模型技术上能否完成任务”属于模型能力层
  - “某个 access mode 是否开放某任务”属于产品策略层
- 当前产品活跃任务的硬约束做并集，形成唯一全局模型准入标准
- 任何模型只要不满足这套全局硬约束中的任意一项，就直接视为“不可用模型”
  - 不采用“任务 A 可用、任务 B 再临时报错”的运行时断层模式
- 对当前依赖结构化 artifact 的任务，`native structured output` 视为正式可用模型的硬约束
- 在当前流式接口与前端消费契约不改变的前提下，streaming 视为正式可用模型的硬约束
- `thinking` 一律不再作为模型准入硬约束，只能作为软约束或优化项
- 对 custom OpenAI-compatible model，如果不满足 `native structured output`，直接视为不可用
  - 不提供所谓“实验兼容模式”
- 用户不需要也不应该手动填写模型能力表
- BYOK custom model 必须通过系统探测或系统推断进入“可用 / 不可用 / 待验证”状态，而不是直接假定可用
- provider 真实探测不应在每次正式请求时重复发生
  - 每次请求都可以做轻量本地 `preflight decision`
  - 但真实 provider probe 默认只应发生在保存配置后自动检测
- 正常使用中的正式调用成功可以视为一次隐式健康检查
  - 不需要再因为时间过期而主动要求用户重复检测
  - 如果正式调用出现明确能力不匹配、模型不可达或配置失效，再把模型状态打回“需重检”
- `maxOutputTokens` 不应继续承担主要的防滥用职责
  - 对关键结构化任务，它首先必须保证“足够容纳完整合法 JSON”
  - 防滥用应主要依赖 access control、额度、路由限流、输入约束与显式任务策略，而不是把结构化输出天花板压得过低
- `distill` 如需做输入瘦身，只能优先移除对 `quickExplain` / `contextMemory` 无直接价值的包装性信息
  - 不得为了省 token 而粗暴删减教学正文，导致 Focus mode 可读性或解释质量明显下降
- `distill` 的主要治理顺序应是：
  - 先提高安全的输出预算
  - 再做只移除包装层的输入瘦身
  - 最后提供更清晰的 structured-output 失败归因与日志，而不是自动 retry
- 本 phase 不引入自动 retry 作为 `distill` 的兜底策略
  - 先把真实错误暴露清楚，避免掩盖参数问题、预算问题或能力问题

## Overrides Or Exceptions

- 相对于此前代码中“按 task 硬编码参数”的做法，本 phase 明确例外为：任务策略不再直接决定 provider 参数，provider 参数必须经过模型能力层解析后再生成。

## Agent Discretion

- 可以自主决定模型能力注册表的具体字段命名与存储位置，只要后端是唯一技术真相
- 可以自主决定 capability probe 的最小探针形态，只要它基于真实 provider 请求而不是纯本地猜测
- 可以自主决定哪些 provider 错误签名需要被归一化为稳定能力错误
- 可以自主决定前端对“待验证 / 不可用 / 可用”的最小提示形态，只要不把能力判断责任转嫁给用户

## Success Criteria

- 当前活跃模型的能力真相不再散落在前端列表、env 解析和 task builder 之间
- `gemini-2.5-flash` 等模型不会再因为错误的 thinking 参数组合而运行时爆炸
- `distill` 不再因 `MAX_TOKENS` 频繁返回不完整 JSON，同时 `quickExplain` / Focus mode 的体验质量保持稳定
- 新增或调整模型时，开发者有一条明确的“注册能力 -> 通过测试 -> 可用”的路径
- custom BYOK model 不再默认假定可用，而是有清晰的验证与准入状态
- 结构化任务和非结构化任务的参数生成边界变得清楚
- 文档与测试能反映“为什么这个模型可用 / 不可用”，而不是只能依赖 provider 原始报错

## Constraints

- 不得破坏现有 `explain`、`distill`、`followup`、quiz 主链路
- 不得让用户承担填写能力表、理解 provider 差异细节的负担
- 不得把软约束误升级成模型准入硬门槛
- 不得把产品策略差异混入模型能力真相
- 当前 phase 应优先收口“参数合法性与能力边界”，而不是继续扩大功能范围

## Canonical References

- `docs/discuss/project-brief.md`
- `.planning/PROJECT.md`
- `.planning/ROADMAP.md`
- `docs/backend/platform-model-configuration.md`
- `docs/backend/api-design.md`
- `GGlearn-AI/src/config/models.ts`
- `GGlearn-AI/api/lib/env.ts`
- `GGlearn-AI/api/lib/structuredOutputConfig.ts`
- `GGlearn-AI/api/lib/generateService.ts`
- `GGlearn-AI/src/hooks/useSlideAnalysis.ts`
- `GGlearn-AI/src/hooks/useFollowUp.ts`
- `GGlearn-AI/src/hooks/useQuiz.ts`
- `GGlearn-AI/src/lib/ai/artifacts.ts`
- `https://ai.google.dev/gemini-api/docs/thinking`
- `https://ai.google.dev/gemini-api/docs/structured-output`
- `https://platform.openai.com/docs/guides/structured-outputs`

## Open Questions

- capability probe 的最终触发策略是否按以下默认方向锁定：
  - 以“保存配置后自动跑一次”为主
  - 正常调用成功视为持续健康证明
  - 仅在正式调用出现明确兼容性/连通性异常时打回“需重检”
- 哪些 provider 错误签名足够稳定，适合触发“需重检”而不是普通瞬时失败

## Deferred Ideas

- 从 repo 内能力表进一步发展到自动生成前端模型可见性配置
- 在未来引入 provider 级质量/稳定性评级
- 对 preview / latest / experimental 模型做额外运营提示

## Impact On Existing Plans

- Requires replanning because the current model-selection and request-generation path still assumes provider parameters can be chosen from task branches alone, which is no longer acceptable once model capability becomes a maintained product boundary.

## Next Step

Run `gsd-plan-phase 09 --prd docs/discuss/phases/09-model-capability-registry-and-parameter-hardening-brief.md` once this brief is approved.
