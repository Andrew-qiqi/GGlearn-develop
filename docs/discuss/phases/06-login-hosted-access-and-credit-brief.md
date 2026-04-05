# Phase 06 Brief: Login, Hosted Access, and Credit Billing

## Metadata

- Status: Draft
- Phase: 06
- Related Roadmap Entry: `.planning/ROADMAP.md`
- Last Updated: 2026-04-05
- Owner: Agent-authored, user-approved
- Impacts Existing Plans: Yes
- Change Summary: 同步最新已确认的 hosted product 决策，并补充当前已经落地的 Clerk 与 credits 基础，明确 Phase 06 剩余工作将以 ZPAY 接入和 hosted hardening 为主，而不是重新讨论产品模型。

## Objective

在不把项目做成复杂 SaaS 的前提下，完成 SlideTutor 第一版 hosted access 产品边界：

- `BYOK` 继续可用且不强制登录
- `Platform API` 作为登录后的第二路径存在
- 新用户获得一次性 starter credits
- hosted action 按 credits 计费，且成功才扣费
- payment provider 使用 `ZPAY`

## Current State

以下基础已经存在，不应在未来 planning 中被误判为“尚未开始”：

- `My API` / `Platform API` 模式切换 UI 已存在
- Clerk 前端 provider 与基础登录跳转已接通
- Worker 侧已有 Clerk session 校验基础
- D1-backed credits account / ledger / recharge order 基础已存在
- starter credits、recharge quote、hosted balance 查询、本地 happy path 已经验证过
- hosted action 的 success-only charging skeleton 已经存在

因此，Phase 06 剩余工作更接近“产品化收尾与支付落地”，而不是再从头设计 auth 或 credits 模型。

## Problem

虽然 hosted access 基础已经开了头，但还没有达到可继续推进的完整状态：

- payment adapter 仍是 mock
- `ZPAY` 尚未接入
- Phase 06 的文档没有明确标出哪些已经落地、哪些才是剩余工作
- 如果此时跳过 parser phase 直接猛推 payment，很容易让 parser/provider 技术债继续扩散

## In Scope

- 基于现有 Clerk 基础继续收口 hosted access
- 用 `ZPAY` 替换当前 mock payment adapter
- 完成 hosted recharge、payment webhook、防重和余额入账链路
- 收口 hosted credits 与 action preflight / finalize 的边界
- 保持设置页为主要充值入口
- 保持用户侧 UI 简洁，不增加账单历史页和充值历史页

## Out of Scope

- 订阅制
- 套餐包
- billing dashboard
- 用户侧扣费记录与充值记录页
- parser BYOK
- parser provider 选择 UI
- 大规模风控与复杂财务系统

## Inherited Project Decisions

- Cloudflare 是当前运行底座
- BYOK-first 仍是首个公开版本的主入口
- `Platform API` 是第二路径，而不是替代 BYOK
- 用户自己的模型密钥仍只保存在本地浏览器
- parser 与 payment 不应混成一个 phase 来做
- 中国用户与中国运营者友好的支付路径优先

## Phase-Specific Locked Decisions

- `Platform API` 需要登录
- 新用户一次性获得 `10 credits`
- 不做每周赠送 credits
- credits 不过期
- 最低充值 `1 RMB`
- 固定兑换率为 `1 RMB = 30 credits`
- 充值采用自由输入金额，而不是套餐包
- 用户侧不展示扣费记录和充值记录
- hosted action 定价锁定为：
  - `Analyze = 3 credits`
  - `Follow-up = 1 credit`
  - `Quiz 生成 = 1 credit`
  - `Quiz 答案分析 = 1 credit`
- `Analyze` 是一个整体动作，只有 `parse + explain + distill` 全部成功才扣 `3 credits`
- 所有 hosted action 都遵循“成功才扣费”
- payment provider 锁定为 `ZPAY`
- 用户可以自己选择模型，不应被平台固定死

## Overrides Or Exceptions

None.

## Agent Discretion

- 可以自主设计订单、防重、webhook 和入账表字段，只要未来审计能力足够。
- 可以自主决定 hosted access 剩余技术任务的切分方式，只要不重新打开已经锁定的产品决策。
- 可以自主决定设置页中 recharge 与 balance 的具体文案和布局，只要不增加不必要的 billing 干扰。

## Success Criteria

- `Platform API` 的登录、credits、recharge、payment webhook 能闭环
- `ZPAY` 替代 mock adapter 成为真实支付路径
- hosted action 的成功扣费规则继续成立
- 用户侧体验仍保持简单，不被 billing 页面打扰
- BYOK 路径不被 login/payment 逻辑污染

## Constraints

- 不得破坏当前成熟教学逻辑
- 不得让 BYOK 路径变成必须登录
- 不得把本 phase 扩展成完整 SaaS back office
- 不得引入与当前 Cloudflare 路线明显冲突的支付/存储设计

## Canonical References

- `docs/discuss/project-brief.md`
- `docs/discuss/phases/05-parser-bootstrap-and-provider-abstraction-brief.md`
- `.planning/ROADMAP.md`
- `docs/backend/api-design.md`
- `docs/superpowers/specs/2026-04-05-phase-06-platform-api-credit-product-design.md`
- `docs/superpowers/plans/2026-04-05-phase-06-platform-api-credit-implementation.md`

## Open Questions

- ZPAY 的最小 webhook 与订单状态模型如何设计，才能满足当前 credits 入账需求且不引入过度复杂度？
- 在 Phase 06 正式回到 GSD 时，哪些已落地基础应直接视为既有上下文，而不是重复规划？

## Deferred Ideas

- 订阅制
- 进阶 billing 后台
- parser BYOK
- 跨设备同步更多账户数据
- 更复杂的 anti-abuse / fraud 系统

## Impact On Existing Plans

- Requires replanning because Phase 06 is no longer blank; Clerk and credits foundations already exist and should become the baseline for future planning.

## Next Step

在 Phase 05 完成后，再使用本 brief 回到 GSD：

`gsd-plan-phase 06 --prd docs/discuss/phases/06-login-hosted-access-and-credit-brief.md`
