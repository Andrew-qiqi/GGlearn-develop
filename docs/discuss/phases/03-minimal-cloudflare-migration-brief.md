# Phase 03 Brief: Minimal Cloudflare Migration

## Metadata

- Status: Draft
- Phase: 03
- Related Roadmap Entry: `.planning/ROADMAP.md`
- Last Updated: 2026-04-04
- Owner: Agent-authored, user-approved
- Impacts Existing Plans: Yes
- Change Summary: Define the minimum Cloudflare migration boundary so the next public product work grows on the intended deployment base instead of the current Vercel-first assumption.

## Objective

在不把整套商业化能力一起卷入的前提下，完成一轮最小可用的 Cloudflare 迁移边界定义与实施准备，使后续 `BYOK-first` 首发能力建立在正确的平台底座上。

## Problem

当前项目虽然已经明确 Cloudflare 是更适合未来商业化的方向，但现有实现和部署认知仍然带有较强的 Vercel-first 假设。如果继续在这个基础上直接开发 `BYOK`、登录、支付、平台托管 API 等能力，后续大概率会在运行时、环境变量、流式响应、鉴权、限流、邮件等环节发生二次改造。

这个 phase 的任务不是把所有基础设施一次性重写，而是先把“什么必须先迁、什么可以后迁”说清楚，并把最关键的公共运行路径迁到新的平台基座上。

## In Scope

- 明确本轮 Cloudflare 迁移的最小边界。
- 确认前端公开站点的部署路径如何迁移到 Cloudflare 方向。
- 确认核心 `/api/generate` 流式生成链路在 Cloudflare 方向上的可行方案。
- 确认环境变量、基础鉴权、限流、代理 IP、日志观察等底层能力在新平台上的最低要求。
- 识别当前 Vercel 相关假设中，哪些必须在本 phase 处理，哪些可以保留为后续过渡项。
- 产出足够清晰的实现输入，供后续 `gsd-plan-phase 03` 使用。

## Out of Scope

- 不在本 phase 内实现完整登录系统。
- 不在本 phase 内实现支付、订阅、积分或完整商业化闭环。
- 不在本 phase 内实现平台托管 API 的正式付费上线。
- 不在本 phase 内完成 BYOK 产品功能本身。
- 不在本 phase 内完成解析器 provider abstraction 的完整落地。
- 不追求一次性消灭所有 Vercel 痕迹；只处理会阻塞后续核心工作的部分。

## Inherited Project Decisions

- 首个公开版本是 `BYOK-first`。
- Cloudflare 是未来部署主方向，不再继续以 Vercel Hobby 为长期前提。
- 平台托管 API 是未来并行产品线，但不是首个公开版本的主入口。
- 文档解析在早期用户增长阶段可以由平台承担，但必须逐步从“隐形默认成本”变成“显式可控成本中心”。
- 结构化 JSON 输出和 provider-aware routing 必须继续保持，不能因迁移平台而削弱。

## Phase-Specific Locked Decisions

- 本 phase 采用“最小迁移”原则，不做整套基础设施重构。
- 本 phase 的成功标准不是“所有服务都迁完”，而是“后续 BYOK-first 能够在新平台方向上继续开发而不必大返工”。
- `/api/generate` 是本 phase 的关键后端能力；如果它在新平台方向上没有稳定方案，本 phase 不能算完成。
- 可以接受短期双平台过渡，但这只能是过渡态，不是长期运行模式。
- 目标是尽快收敛到 Cloudflare 主运行面，而不是长期同时维护 Vercel 与 Cloudflare 双栈。
- 首轮迁移应优先把“首版用户主链路 API”整体迁到 Cloudflare，而不是拆成双平台主链路。
- “首版用户主链路 API”至少包括前端公开访问路径、`/api/generate`、`/api/get-token`、`/api/parse` 以及任何首版正常学习流程必经接口。
- 非主链路接口可以后置，但不能影响首版用户的完整学习流程。
- `/api/generate` 在本 phase 中允许做小型服务端适配层重构，但重构目标仅限平台/runtime 兼容，不得借机改动成熟教学业务逻辑。
- 对于当前公开产品运行所需的配套层，首轮迁移应尽量一次完成，而不是留下大量 Cloudflare/Vercel 跨平台补洞工作。
- 这里的“配套层一起迁”包括当前主运行面实际依赖的鉴权、限流、环境变量管理、代理/IP 处理、日志观察、邮件或通知等现有支撑能力。
- 这里的“配套层一起迁”不包括登录、支付、平台托管 API 等本来就不属于本 phase 的后续系统。
- 登录、支付、平台托管 API、解析器抽象，都必须与本 phase 解耦，不能被默默打包进来。

## Overrides Or Exceptions

None.

## Agent Discretion

- 可以自主评估 Cloudflare 侧具体采用哪种最小运行拓扑，只要满足本 phase 目标。
- 可以自主判断哪些 Vercel 依赖项必须立即迁移，哪些可以短期保留。
- 可以提出一小层兼容适配，而不是直接大改所有现有实现。
- 可以把实现拆成“先验证可行性”与“再正式切换”两步，只要边界清晰。

## Success Criteria

- 已明确并记录本轮 Cloudflare 迁移的最小边界。
- 前端公开站点与核心生成 API 的迁移路径清晰，不再停留在抽象方向讨论。
- 首版用户主链路不再依赖双平台来回切换。
- 当前公开产品运行所需的主要配套层不再依赖长期双平台拆分维护。
- 已识别会影响后续 `BYOK-first`、登录、支付的关键平台差异。
- 后续 Phase 04 不需要再建立在 Vercel-first 假设上继续设计。
- 本 phase 的计划不会偷偷膨胀成一次大规模平台重构。

## Constraints

- 不得破坏当前核心学习体验。
- 不得主动修改已经成熟的教学业务逻辑、教学 prompt 意图、artifact contract 与前端消费契约。
- 不得因迁移讨论而削弱现有结构化输出契约。
- 必须考虑中国用户访问与中国运营者维护便利性。
- 必须优先考虑后续 BYOK-first 路线，而不是为了当前平台兼容性牺牲长期方向。
- 必须把范围控制住，避免和登录、支付、解析器抽象 phase 混淆。

## Canonical References

- `docs/discuss/project-brief.md`
- `.planning/PROJECT.md`
- `.planning/ROADMAP.md`
- `.planning/REQUIREMENTS.md`
- `docs/architecture/deployment.md`
- `docs/backend/api-design.md`
- `docs/security/token-authentication.md`
- `SlideTutor-AI/api/generate.ts`
- `SlideTutor-AI/src/config/models.ts`

## Open Questions

- `/api/generate` 的流式响应在 Cloudflare 方向上应采用什么最小兼容实现？
- 当前配套层里，哪些可以直接迁移复用，哪些需要做 Cloudflare 定向适配？

## Deferred Ideas

- BYOK 产品 UI 与密钥配置体验细节。
- 平台托管 API 的账户、套餐、支付和风控设计。
- 解析器多 provider 抽象与 China-friendly parser shortlist。
- 更深的中国大陆基础设施专项改造。

## Impact On Existing Plans

- Requires replanning because the old `.planning` context was centered on refinement-era work and no longer reflects the current deployment-first product direction.

## Next Step

Run `gsd-plan-phase 03 --prd docs/discuss/phases/03-minimal-cloudflare-migration-brief.md` once this brief is approved.
