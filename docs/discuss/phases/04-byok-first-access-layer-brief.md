# Phase 04 Brief: BYOK-First Access Layer

## Metadata

- Status: Draft
- Phase: 04
- Related Roadmap Entry: `.planning/ROADMAP.md`
- Last Updated: 2026-04-04
- Owner: Agent-authored, user-approved
- Impacts Existing Plans: Yes
- Change Summary: Define the first public product path around user-supplied model APIs, with OpenAI-compatible BYOK as the primary direction and Gemini preserved as a separate adapter.

## Objective

在完成最小 Cloudflare 迁移方向确认后，定义并准备实现首个公开版本的 `BYOK-first` 接入层，使用户能够通过自己的模型 API 使用 GGlearn AI，而不必等待平台托管推理服务先成熟。

## Problem

项目已经明确首个公开版本要走 `BYOK-first`，但现在系统仍然主要依赖服务端环境变量注入平台自己的模型密钥，真实的“用户自带 API”能力还没有形成完整产品和架构闭环。

如果没有独立的 BYOK phase，后续很容易把模型接入、密钥存储、Provider UI、错误处理、产品定价、平台托管能力一起混成一个超大任务，结果既拖慢上线，也模糊了首个公开版本的主路径。

## In Scope

- 明确首个公开版本中 `BYOK-first` 的用户路径。
- 明确 BYOK 支持的 provider 家族优先级，尤其是 `OpenAI-compatible` 的统一接入方向。
- 明确 Gemini 作为单独 adapter 的保留策略。
- 明确用户模型密钥/endpoint 的基本配置方式与安全边界。
- 明确前端模型配置、连通性校验、错误反馈、回退策略的范围。
- 明确后端 provider routing 如何从“平台环境变量驱动”演进到“支持用户提供配置”。
- 明确 BYOK 与未来平台托管 API 的共存关系，但不在本 phase 完成后者。

## Out of Scope

- 不在本 phase 内实现完整登录系统。
- 不在本 phase 内实现平台托管模型服务的正式上线。
- 不在本 phase 内实现支付、订阅、用量计费闭环。
- 不在本 phase 内实现解析器 BYOK。
- 不在本 phase 内重新设计整套模型能力矩阵或所有 provider 的高级特性。
- 不要求一次支持所有供应商的所有差异化能力；优先保证统一主路径成立。

## Inherited Project Decisions

- 首个公开版本是 `BYOK-first`。
- 平台托管 API 是未来并行产品线，但不是当前首发主入口。
- OpenAI-compatible providers 应尽可能共用一套适配层。
- Gemini 保持单独 adapter，不强行并入 OpenAI-compatible 路径。
- 结构化 JSON 输出必须保持为长期主方向，不能因接入 BYOK 而退回 prompt-only 控制。
- 文档解析在早期可以由平台承担，不要求用户先解决 parser 配置问题。

## Phase-Specific Locked Decisions

- 本 phase 的首要目标是让用户“带着自己的模型 API 就能用起来”，而不是先做平台托管推理。
- OpenAI-compatible BYOK 是首要接入对象，应优先于为单个 provider 做大量特判。
- Gemini 继续走单独适配路线，但它的存在不能阻塞 OpenAI-compatible BYOK 的主路径推进。
- 本 phase 只定义和实现 BYOK 所必需的配置、校验、路由和错误处理，不扩展到完整商业化系统。
- 平台托管 API 在本 phase 中只作为未来兼容目标被考虑，不作为当前交付范围。

## Overrides Or Exceptions

None.

## Agent Discretion

- 可以自主提出用户配置模型的最小交互方案，只要主路径清晰、门槛低。
- 可以自主决定用户提供的信息边界，例如是否允许自定义 `baseURL`、模型名、provider 标签等。
- 可以自主提出“先支持最通用字段，再逐步开放高级参数”的分层方案。
- 可以在实现方案中保留过渡适配层，只要最终边界清楚且不增加长期混乱。

## Success Criteria

- 首个公开版本的 BYOK 主路径已被明确记录，不再停留在抽象意向层。
- 用户如何配置自己的模型 API、系统如何验证、失败时如何反馈，都有清晰边界。
- OpenAI-compatible BYOK 路径被设计成统一适配层，而不是按 provider 零散扩展。
- Gemini 的保留方式清晰，不会在后续实现时反复摇摆。
- 后续平台托管 API 仍能与 BYOK 共存，而不会被本 phase 的设计堵死。

## Constraints

- 不得破坏当前核心教学体验与结构化输出契约。
- 不得把本 phase 偷偷扩张成登录、支付、平台托管服务的一次性大开发。
- 必须考虑中国用户对 provider 可用性、配置难度、失败率的实际体验。
- 必须尽量降低首个公开版本的接入门槛，避免要求用户处理过多平台细节。
- 必须与最小 Cloudflare 迁移方向兼容，不能继续建立在旧平台假设之上。

## Canonical References

- `docs/discuss/project-brief.md`
- `docs/discuss/phases/03-minimal-cloudflare-migration-brief.md`
- `.planning/PROJECT.md`
- `.planning/ROADMAP.md`
- `.planning/REQUIREMENTS.md`
- `docs/frontend/architecture.md`
- `docs/frontend/data-flow.md`
- `GGlearn-AI/src/config/models.ts`
- `GGlearn-AI/api/generate.ts`
- `GGlearn-AI/api/lib/structuredOutputConfig.ts`

## Open Questions

- 早期 BYOK 是否完全免费、捐赠支持，还是收取少量服务费？
- 用户是否应被允许直接填写自定义 `baseURL`，还是优先提供预置 provider 选项？
- 密钥是仅保存在本地、还是未来支持登录后加密托管？
- 首个公开版本里，模型配置入口是全局设置优先，还是首次使用引导优先？

## Deferred Ideas

- 平台托管 API 的套餐、配额、风控与支付机制。
- 登录后的跨设备模型配置同步。
- 解析器 provider 的 BYOK 版本。
- 更细粒度的 provider capability matrix 与高级推理控制。

## Impact On Existing Plans

- Requires replanning because the previous `.planning` context did not treat BYOK as the primary public product path.

## Next Step

Run `gsd-plan-phase 04 --prd docs/discuss/phases/04-byok-first-access-layer-brief.md` once this brief is approved.
