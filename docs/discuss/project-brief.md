# Project Brief

## Metadata

- Status: Active
- Last Updated: 2026-04-05
- Owner: Agent-authored, user-approved
- Impacts Existing Plans: Yes
- Change Summary: 同步已完成的 Cloudflare-first 与 BYOK-first 阶段、已落地的 Clerk + credits 基础、以及最新锁定的 parser 与 hosted access 决策；明确下一次回到 GSD 时应先完成 Phase 05 的剩余 parser provider 工作，而不是直接把 parser、payment、hosted access 混做一团。

## Project Identity

- Project Name: GGlearn AI
- One-line Summary: 一个以 PDF 课件为中心的 AI 学习助手，帮助学生解释 slide、保持上下文连续、做 follow-up 学习与 quiz 自测。
- Core Value: 把静态课件变成更像老师带学的体验，同时给用户明确的接入边界：既支持 `My API`，也支持未来的 `Platform API`。

## Why This Project Exists

课件和 lecture PDF 通常信息压缩严重、上下文断裂、缺少讲解线索。学生需要的不是普通 OCR 或摘要，而是能围绕页面内容、视觉区域和跨页上下文持续解释的学习助手。

现在项目已经从“内部试验是否可用”进入“如何真实发布并可持续运营”的阶段。问题不再只是解释效果够不够好，而是：

- 如何在不破坏教学体验的前提下上线真实产品
- 如何为中国用户和中国运营者选择更实际的 provider 与支付路径
- 如何把 parser、hosted access、credits 这些成本边界做成可维护的产品能力

## True Needs

- 保持 `explain`、`distill`、follow-up、quiz 等核心学习体验的质量与连续性。
- 用结构化输出维持不同模型下的结果稳定性。
- 让首个公开版本可以低门槛使用，优先支持用户自带模型。
- 给不想折腾 API key 的用户保留后续 `Platform API` 路线。
- 把文档解析成本从“背景免费假设”变成显式可控的系统能力。
- 尽量选择更适合中国用户与中国运营者的模型、解析与支付方案。

## Non-Goals

- 不在一个 phase 里同时做完 parser、hosted access、payment、billing back office。
- 不把首个公开版本做成必须登录、必须充值、必须自配 parser 的高门槛产品。
- 不为了后续扩展而过度设计当前小用户量阶段的系统。
- 不因为 provider 切换或计费改造而破坏成熟的教学主链路。

## Constraints

- 项目仍处于早期公开前阶段，代码和产品方向都在收口中。
- 教学 prompt 意图、语气、解释结构和前端消费契约必须优先保护。
- 目标用户和运营者都偏中国场景，provider 选择必须考虑可获取性、成本和稳定性。
- Cloudflare 已是当前运行底座，后续 auth、quota、credits、payment 都应尽量贴合这条路线。
- 文档解析不再能依赖 Azure 的免费额度假设。

## Locked Decisions

- Cloudflare-first 迁移已完成，不再以 Vercel-first 为后续前提。
- 首个公开版本仍然是 `BYOK-first`。
- `BYOK` 继续免费，不强制登录。
- 用户自己的模型密钥默认只保存在浏览器本地，不上传云端。
- 用户可以在 `My API` 和 `Platform API` 之间切换。
- `Platform API` 是第二条产品路径，需要登录。
- 认证方案采用成熟外部方案，当前锁定为 `Clerk`。
- hosted access 相关产品/业务数据以 Cloudflare 原生存储为主，当前基线是 `D1`。
- 新用户获得一次性 `10 credits`，不做每周赠送。
- credits 不过期。
- 充值方式不是套餐包，而是自由充值；`1 RMB` 起充。
- 固定兑换率为 `1 RMB = 30 credits`。
- 不做用户侧的扣费记录页和充值记录页；后端保留必要账务数据即可。
- hosted action 计费权重锁定为：
  - `Analyze = 3 credits`
  - `Follow-up = 1 credit`
  - `Quiz 生成 = 1 credit`
  - `Quiz 答案分析 = 1 credit`
- `Analyze` 被视为一个整体动作，包含 parse、explain、distill；只有整体成功才扣费。
- 所有 hosted action 都遵循“成功才扣费”。
- 平台自带文档解析服务的目标 provider 锁定为 `Volcengine`，核心原因是成本更低且更适合当前需求。
- 当前 GGlearn 的 parser 只需要稳定提供页面级文本块、块类型和规范化坐标，不需要追求 Azure 全量能力对等。
- parser BYOK 不是当前回到 GSD 的目标；如果未来真的要做中国用户友好的 parser BYOK，`MinerU` 是值得优先评估的候选，但暂不进入本轮实现范围。
- 支付方向锁定为 `ZPAY`，但 payment 接入属于后续 Phase 06 收尾工作，不应混入当前 parser phase。
- 下一次回到 GSD 时，应优先完成 Phase 05 的剩余工作：平台 parser provider 从 Azure 收口到 Volcengine，并清理遗留 Azure 路径。

## Agent Discretion

- Volcengine 返回结果如何最小映射到现有 `LayoutBlock[]` 契约，可在不破坏前端的前提下由 agent 自主设计。
- 未来 parser BYOK 是否只支持 MinerU，或再增加其他 provider，暂未锁定。
- hosted access 的首发开放方式仍可后续决定：直接开放、invite-only、或其他更克制的节奏。
- ZPAY 的具体 webhook、防重和订单字段方案可以在 Phase 06 规划时细化。

## Success Conditions

- 首个公开版本继续以 BYOK 为主，不被 hosted access 反客为主。
- 平台 parser 不再依赖 Azure 的隐形默认真相。
- parser、auth、credits、payment 的边界清晰，后续每个 phase 可以独立推进。
- 中国用户侧的模型、解析和支付路径比之前更现实、更低摩擦。
- 系统仍然保持可维护，provider 差异被收敛在明确边界内。

## Current Phase Direction

- Phase 05 remaining work:
  - 完成平台 parser provider 从 Azure 到 Volcengine 的切换。
  - 保持现有 `LayoutBlock[]` 契约、quota 规则、degraded fallback 和前端使用方式不变。
  - 清理旧的直接 Azure 路径，避免后面出现“双 parser 真相”。
- Phase 06 later work:
  - 基于已经落地的 Clerk + credits 基础继续完成 hosted access。
  - 用 ZPAY 替换当前 mock payment adapter。
  - 收口 hosted action、payment webhook 和生产配置。

## Canonical References

- `AGENTS.md`
- `docs/discuss/phases/05-parser-bootstrap-and-provider-abstraction-brief.md`
- `docs/discuss/phases/06-login-hosted-access-and-credit-brief.md`
- `.planning/ROADMAP.md`
- `.planning/STATE.md`
- `docs/backend/api-design.md`
- `docs/changelog/CHANGELOG_TECH.md`

## Open Questions

- Volcengine 的返回数据应采用怎样的最小映射，才能既保留现有前端契约又给未来 parser BYOK 留出空间？
- Phase 06 在完成 ZPAY 后，hosted access 首发是直接开放还是更克制地灰度开放？

## Next Step

使用本 brief 作为总入口，但下一次具体回到 GSD 时，优先执行：

`gsd-plan-phase 05 --prd docs/discuss/phases/05-parser-bootstrap-and-provider-abstraction-brief.md`
