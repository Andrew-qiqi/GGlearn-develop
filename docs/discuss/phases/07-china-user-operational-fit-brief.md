# Phase 07 Brief: China-User Operational Fit

## Metadata

- Status: Ready for GSD planning
- Phase: 07
- Related Roadmap Entry: `.planning/ROADMAP.md`
- Last Updated: 2026-04-06
- Owner: Agent-authored, user-approved
- Impacts Existing Plans: Yes
- Change Summary: 在 Cloudflare、Clerk、Volcengine parser、ZPAY 充值链路都已落地后，Phase 07 不再讨论“要不要做中国用户适配”，而是验证当前真实链路在中国用户与中国运营者场景下是否真的顺手、稳定、可维护。

## Objective

验证 SlideTutor 当前真实产品链路，是否已经足够适配中国用户与中国运营者：

- 用户是否能顺畅使用 `My API`
- 用户是否能顺畅使用 `Platform API`
- 运营者是否能稳定维护 parser、login、payment、部署与支持链路

## Problem

前面的阶段已经把核心技术链路搭出来了，但“能跑通”不等于“适合真实使用”。

当前最容易被误判的点是：

- 中国用户是否真的容易拿到可用模型 API
- `Platform API` 是否真的比让用户自己折腾更省心
- Volcengine parser、Clerk、ZPAY、Cloudflare 这条组合在真实运营里是否有隐藏摩擦
- 我们是否应该立刻做 parser BYOK / MinerU，还是先用真实反馈判断

如果不先做这一轮 operational fit 验证，就很容易过早投入更多基础设施或更复杂的 provider 扩展。

## In Scope

- 验证中国用户使用当前 `BYOK-first` 路径的主要摩擦点
- 验证中国用户使用当前 `Platform API` 路径的主要摩擦点
- 验证中国运营者维护以下链路的真实成本与复杂度：
  - Cloudflare 部署
  - Clerk 登录
  - Volcengine parser
  - ZPAY 充值
- 明确哪些问题是“必须尽快修”，哪些问题应该继续延后
- 形成对下一阶段的明确决策输入：
  - 是否需要 parser BYOK
  - 是否优先评估 `MinerU`
  - 是否需要更多模型 provider 预设
  - 是否需要补更强的运营监控或支持文档

## Out of Scope

- 直接实现 parser BYOK
- 直接接入 `MinerU`
- 新增更多 payment provider
- 大规模改造部署架构
- 新做 billing dashboard、运营后台、风控系统
- 把本 phase 变成新的“基础设施大重构”

## Inherited Project Decisions

- `BYOK-first` 仍然是首个公开版本的主路径
- `Platform API` 是第二路径，不替代 `BYOK`
- 平台 parser 默认使用 `Volcengine`
- 充值使用 `ZPAY`
- credits 规则已锁定：
  - 新用户一次性 `10 credits`
  - `1 RMB = 30 credits`
  - 最低充值 `1 RMB`
  - credits 不过期
- parser BYOK 与 `MinerU` 讨论此前已明确延后到当前平台链路稳定之后

## Phase-Specific Locked Decisions

- 本 phase 以“验证与收口”优先，不默认新增大功能
- 任何新增实现都必须直接服务于真实摩擦点，而不是为了“以后可能有用”
- 如果当前真实链路已经足够可用，则继续保持 parser 平台托管，不急着开启 parser BYOK
- 只有当中国用户真实使用中，parser 获取成本或接入难度持续成为主要阻塞，才重新提高 `MinerU` / parser BYOK 的优先级

## Overrides Or Exceptions

None.

## Agent Discretion

- 可以自主设计这一 phase 的验证维度、验证顺序、记录格式与结论模板
- 可以自主决定哪些问题应归类为产品问题、运营问题、配置问题或架构问题
- 可以在不扩 scope 的前提下，补必要的小型 hardening、文档或可观测性改进

## Success Criteria

- 我们对中国用户的真实阻塞点有了明确排序，而不是继续凭感觉判断
- 我们知道当前 `Platform API` 路径是否真的降低了中国用户门槛
- 我们知道当前 parser 托管路径是否已经足够，还是必须尽快推进 parser BYOK
- 我们对下一阶段的投入顺序有明确结论，而不是继续并行发散

## Constraints

- 不得破坏当前已经跑通的充值、登录、parser、hosted access 主链路
- 不得把“验证阶段”膨胀成实现多个新系统的 phase
- 必须优先保护现有产品节奏与代码可维护性

## Canonical References

- `docs/discuss/project-brief.md`
- `.planning/ROADMAP.md`
- `docs/discuss/phases/06-login-hosted-access-and-credit-brief.md`
- `docs/backend/api-design.md`
- `docs/architecture/deployment.md`
- `docs/changelog/CHANGELOG_TECH.md`

## Open Questions

- 当前中国用户在模型 API 获取上，最大阻塞点究竟是获取难度、支付难度、还是配置复杂度？
- 当前平台托管 parser 是否已经足够解决“低门槛使用”问题？
- `MinerU` 是否真的是下一步最值得做的 parser BYOK 候选，还是只是讨论中的直觉最优？

## Deferred Ideas

- parser BYOK 具体实现
- `MinerU` adapter 实现
- 更多 hosted provider 扩展
- 更复杂的 billing / support / fraud 系统

## Impact On Existing Plans

- Requires replanning because the next major decision is no longer “how to ship hosted access”, but “what real bottlenecks remain after hosted access has shipped”.

## Next Step

Run:

`gsd-plan-phase 07 --prd docs/discuss/phases/07-china-user-operational-fit-brief.md`
