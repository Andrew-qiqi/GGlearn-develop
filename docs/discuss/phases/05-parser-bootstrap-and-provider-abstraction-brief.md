# Phase 05 Brief: Parser Bootstrap and Provider Abstraction

## Metadata

- Status: Draft
- Phase: 05
- Related Roadmap Entry: `.planning/ROADMAP.md`
- Last Updated: 2026-04-04
- Owner: Agent-authored, user-approved
- Impacts Existing Plans: Yes
- Change Summary: Define the early parser strategy as platform-funded bootstrap plus provider abstraction, so parsing stops being an invisible Azure-default dependency.

## Objective

在不增加首发用户接入门槛的前提下，建立“早期平台承担解析成本，但架构上不再把 Azure 当作隐形永久默认”的解析器策略与实现方向。

## Problem

当前系统把文档解析深度嵌在现有生成链路里，而且历史上还依赖 Azure 的免费额度。这种做法在预研阶段可以接受，但进入真实用户阶段后，解析器已经不再是“背景服务”，而是明确的成本、稳定性和地域可用性问题。

如果这个问题不单独成 phase，后续很容易出现两种坏结果：

- 要么为了降低用户门槛，继续默认平台兜底，但成本、监控、额度和替代方案都不清晰。
- 要么为了节约成本，过早把 parser 配置甩给用户，导致首个公开版本的使用门槛明显上升。

这个 phase 的作用，就是先把早期“平台承担解析成本”的产品策略和后续“provider abstraction”的技术方向一起钉住。

## In Scope

- 明确首个公开阶段的 parser 默认策略：平台管理、平台付费、用户无感。
- 明确 parser 不再被视为隐形免费依赖，而是显式成本中心。
- 明确 parser provider abstraction 的目标边界。
- 明确当前 Azure 路径在后续架构中的位置：过渡 provider，而不是永久默认。
- 明确 parser 调用的监控、限额、失败兜底、回退策略应进入后续实现范围。
- 明确未来可替代 parser provider 的接入方向，但不要求本 phase 内完成全部接入。

## Out of Scope

- 不在本 phase 内实现 parser BYOK。
- 不在本 phase 内要求用户自己配置解析器凭据。
- 不在本 phase 内完成所有 parser provider 的正式接入。
- 不在本 phase 内设计完整计费系统或将 parser 单独商品化。
- 不在本 phase 内重写整个 `/api/generate` 与 `/api/parse` 体系。
- 不在本 phase 内解决所有中国可用 parser 的最终选型。

## Inherited Project Decisions

- 首个公开版本是 `BYOK-first`，但 BYOK 的重点在模型 API，不在 parser。
- 早期用户增长阶段，文档解析可以由平台承担，以降低使用门槛。
- Cloudflare 是目标平台方向，因此 parser 策略也必须与未来平台迁移兼容。
- 系统必须保持结构化输出与高质量教学体验，不能因 parser 调整而削弱主要学习流程。
- 后续平台托管 API 是独立并行产品线，不应和本 phase 混成同一个商业化任务。

## Phase-Specific Locked Decisions

- 首个公开阶段的 parser 默认模式是“平台管理、平台承担成本、用户无须额外配置”。
- 这只是早期 bootstrap 策略，不代表长期 parser 永久免费。
- Azure 在当前阶段可以继续作为过渡 provider，但必须从“隐形默认”转为“可替换 provider”。
- parser provider abstraction 是本 phase 的核心方向之一，不允许后续继续把 provider 差异硬编码扩散到业务逻辑里。
- parser 的监控、限额、失败兜底是必需方向，不再视为可选增强项。

## Overrides Or Exceptions

None.

## Agent Discretion

- 可以自主提出 parser abstraction 的最小边界，只要它足以支撑未来替换 provider。
- 可以自主决定第一批后续候选 provider 的研究优先级，但不必在本 phase 锁死长期唯一答案。
- 可以提出“先抽象接口、后替换默认 provider”的渐进式路径。
- 可以把成本观察、配额控制、失败兜底拆成分层能力，只要主边界清晰。

## Success Criteria

- 早期 parser 策略已被明确记录为平台 bootstrap，而不是永久隐性默认。
- 后续实现不会再把 Azure 视为不可替代的硬编码前提。
- parser provider abstraction 的必要性和边界已经明确。
- parser 成本、监控、限额、失败处理已被视为核心系统责任，而不是补充项。
- 后续 phase 可以继续低门槛服务用户，同时逐步收紧 parser 成本控制。

## Constraints

- 不得提高首个公开版本的初始使用门槛。
- 不得为了 parser 成本控制而破坏核心学习流程。
- 不得把 parser phase 膨胀成一次完整商业化或大规模平台重构。
- 必须考虑中国用户与中国运营者在 parser 可用性和成本上的现实条件。
- 必须兼容未来 Cloudflare 与 BYOK-first 路线，而不是继续建立在旧假设上。

## Canonical References

- `docs/discuss/project-brief.md`
- `docs/discuss/phases/03-minimal-cloudflare-migration-brief.md`
- `docs/discuss/phases/04-byok-first-access-layer-brief.md`
- `.planning/PROJECT.md`
- `.planning/ROADMAP.md`
- `.planning/REQUIREMENTS.md`
- `docs/architecture/deployment.md`
- `docs/backend/api-design.md`
- `SlideTutor-AI/api/generate.ts`

## Open Questions

- Azure 在过渡期是否仍应作为默认 parser provider，还是尽快替换？
- 第一批值得评估的替代 parser provider 是哪些，尤其是更适合中国场景的方案？
- 早期 parser 调用应采用什么程度的配额或限流保护？
- parser 失败时首个公开版本的最低可接受退化体验是什么？

## Deferred Ideas

- parser BYOK 产品化。
- parser 单独计费或与会员体系绑定的商业策略。
- 更深入的多 provider 智能路由。
- 中国大陆专项 parser 基础设施深度优化。

## Impact On Existing Plans

- Requires replanning because parser was previously treated as a background implementation detail rather than an explicit product and cost concern.

## Next Step

Run `gsd-plan-phase 05 --prd docs/discuss/phases/05-parser-bootstrap-and-provider-abstraction-brief.md` once this brief is approved.
