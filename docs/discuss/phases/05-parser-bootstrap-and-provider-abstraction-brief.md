# Phase 05 Brief: Parser Bootstrap and Provider Abstraction

## Metadata

- Status: Draft
- Phase: 05
- Related Roadmap Entry: `.planning/ROADMAP.md`
- Last Updated: 2026-04-05
- Owner: Agent-authored, user-approved
- Impacts Existing Plans: Yes
- Change Summary: 将文档解析从“隐形 Azure 默认依赖”提升为显式产品能力与成本边界，采用平台保底解析 + 服务端额度控制 + provider abstraction 的最小可行方案。

## Objective

在不抬高首个公开版本使用门槛的前提下，建立一套最小可行的文档解析策略：

- 用户默认仍可直接使用平台提供的 `Document Parsing`
- 平台对解析成本有显式、可控的每日限额
- 超额后产品继续可用，但自动降级为无文档解析模式
- 架构层不再把 Azure 写死成长期默认真相，而是为后续替换或扩展 parser provider 留出清晰边界

## Problem

当前系统虽然已经完成 Cloudflare-first 和 BYOK-first 两个关键阶段，但文档解析仍然存在几个未被正式收口的问题：

- 平台目前继续承担解析成本，但没有明确的额度边界
- 解析调用仍然更像“背景实现细节”，而不是一个显式产品能力
- Azure 仍然是实际默认实现，但这不应该继续以“系统隐形真相”的形式存在
- 对中国用户而言，模型 provider 可用性已经证明是现实问题，后续不能继续假设所有上游服务天然可用

如果这一阶段不单独收口，后续很容易出现两种坏结果：

- 要么继续无限制地平台兜底解析，导致成本与运营边界失控
- 要么为了控成本过早把 parser 配置直接甩给用户，抬高首个公开版本门槛

## In Scope

- 明确早期公开版本的文档解析产品策略：平台保底提供，而不是要求用户先自配 parser
- 明确平台解析的每日免费额度与重置规则
- 明确“什么时候扣额度”的判定方式
- 明确超额后的降级策略与用户提示方式
- 明确解析额度必须由服务端维护，而不是本地存储做真实限制
- 明确当前 parser provider abstraction 的最小边界
- 明确 Azure 在当前阶段的角色：内部默认实现，而非产品层暴露对象
- 明确设置页中解析额度状态的展示方案

## Out of Scope

- 不在本 phase 内实现 parser BYOK
- 不在本 phase 内要求用户填写自己的 parser provider 凭据
- 不在本 phase 内做 parser 多 provider 正式接入
- 不在本 phase 内设计完整计费系统或 parser 商业化方案
- 不在本 phase 内引入登录后额度同步
- 不在本 phase 内为小用户量场景提前做复杂并发兜底、风控或设备指纹方案

## Inherited Project Decisions

- 首个公开版本仍然是 `BYOK-first`，但这个 BYOK 的核心是模型 API，而不是 parser
- 平台在早期用户获取阶段可以继续承担文档解析成本
- Cloudflare 已是当前运行底座，因此解析额度和解析服务边界也必须与 Cloudflare 兼容
- 用户侧不应该感知 Azure；产品层只暴露 `Document Parsing` 这项能力
- 中国用户不应再默认依赖 Gemini 可用性，因为已经出现 `User location is not supported for the API use`

## Phase-Specific Locked Decisions

- 平台默认继续提供 `Document Parsing`
- 当前公开阶段的默认免费解析额度为：每天 `10` 次页面级解析
- 额度按自然日重置
- 是否扣额度，不看用户点击了几次“分析”，只看是否真实发生并成功完成了一次平台 parser 调用
- 超额后不阻止 AI 继续分析，而是自动降级为无文档解析模式
- 降级后产品仍可继续使用，只是精度下降
- 解析额度的真相必须放在服务端维护
- 当前阶段按最小可行方案实现，优先采用 `D1`
- 当前阶段不为很小的用户量提前设计复杂兜底机制
- 第一版匿名解析额度维度采用 `ip_hash + date_key`
- IP 哈希使用独立的 `USAGE_HASH_SECRET`，不复用 `API_TOKEN_SECRET` 或其他现有 secret
- 用户平时不应被额度持续打扰，因此额度状态主入口放在设置页
- 额度状态只放在设置入口内的 AI 设置区域，不额外在主分析流程中做常驻展示
- 设置页中显示精确数字，格式为 `7/10`
- 只有当本次分析真的因超额而降级时，才在结果区顶部显示一条轻量橙色提示 `Low accuracy`
- `Low accuracy` 文案默认只显示短提示；鼠标悬停后再显示详细说明：`Document parsing is unavailable for this analysis, so precision may be lower.`
- 产品侧不暴露 `Azure` 字样，用户只看到 `Document Parsing`

## Overrides Or Exceptions

None.

## Agent Discretion

- 可以自主决定第一版额度记录表的最小字段，只要足以支撑每日额度、自然日重置、服务端计数即可
- 可以自主决定设置页里的解析额度展示样式，只要不制造过度焦虑
- 可以自主决定降级提示的具体英文文案，只要语气是轻提醒而非报错
- 可以自主决定 parser abstraction 的最小代码边界，只要后续替换 provider 时不需要再拆业务主链路

## Success Criteria

- 平台解析不再是“无限隐形兜底”，而是有明确每日边界
- 用户在超额后仍可继续分析，不会被硬阻断
- 产品能让用户真实感知“有文档解析”和“无文档解析”的体验差异
- 设置页中能清楚看到当日解析额度，例如 `7/10`
- 降级提示克制、轻量，不会持续制造焦虑
- Azure 不再作为长期硬编码真相散落在业务流程里
- 后续登录系统完成后，解析额度体系可以自然升级，而不需要推翻本阶段基础设计

## Constraints

- 不得破坏当前成熟的教学业务逻辑
- 不得因为 parser 限额而把首个公开版本做成高门槛产品
- 不得把小用户量阶段的问题过度设计成大规模系统
- 必须考虑中国用户侧与中国运营侧的现实条件
- 必须与当前 Cloudflare Worker 路线兼容

## Canonical References

- `docs/discuss/project-brief.md`
- `docs/discuss/phases/03-minimal-cloudflare-migration-brief.md`
- `docs/discuss/phases/04-byok-first-access-layer-brief.md`
- `.planning/ROADMAP.md`
- `.planning/REQUIREMENTS.md`
- `docs/backend/api-design.md`
- `docs/frontend/architecture.md`
- `docs/frontend/data-flow.md`
- `SlideTutor-AI/api/lib/azureParse.ts`
- `SlideTutor-AI/src/components/SettingsModal.tsx`
- `SlideTutor-AI/wrangler.jsonc`

## Open Questions

None.

## Deferred Ideas

- parser BYOK
- parser provider 多实现切换 UI
- 登录后的跨设备额度同步
- parser 商业化与更高额度套餐
- 为大规模用户量准备的复杂配额/风控体系

## Impact On Existing Plans

- Requires replanning because parser can no longer be treated as a free invisible dependency.

## Next Step

Run `gsd-plan-phase 05 --prd docs/discuss/phases/05-parser-bootstrap-and-provider-abstraction-brief.md` once this brief is approved.
