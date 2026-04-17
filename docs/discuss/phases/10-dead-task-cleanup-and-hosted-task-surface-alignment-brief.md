# Phase 10 Brief: Dead Task Cleanup and Hosted Task Surface Alignment

## Metadata

- Status: Draft
- Phase: 10
- Related Roadmap Entry: `.planning/ROADMAP.md` (follow-up phase proposal after Phase 09)
- Last Updated: 2026-04-10
- Owner: Agent-authored, discussion-in-progress
- Impacts Existing Plans: Yes
- Change Summary: 在模型能力与参数治理之后，下一阶段继续处理当前遗留的任务面问题：彻底删除过期的 `evaluate_note`，并把 `Platform API` 与 `My API` 在 regenerate 类任务上的边界收口成一套明确、可解释、可维护的产品真相。

## Objective

清理已经过期但仍然残留在类型、错误处理、文档和测试中的任务定义，并收口当前 hosted 任务面的真实边界，避免 access mode 差异继续以“历史遗留代码”和“未解释的拦截逻辑”的形式存在。

## Problem

当前有两类遗留问题已经不适合继续拖延：

- `evaluate_note` 已经是过期功能，但仍然残留在 `generateService`、hosted action 类型和文档中
- `Platform API` 目前禁止 `regenerate_chunk / regenerate_followup`，而 `My API` 仍然允许；这个差异不是模型能力层问题，而是来自 Phase 06 的 hosted pricing scope 与产品边界决策

这带来了几个持续问题：

- 活跃任务集合不够干净，影响模型能力设计和测试矩阵
- 开发者容易把“当前产品不开放”误读成“模型技术上不支持”
- 文档、类型、pricing、前端 guardrail 与后端错误码之间存在冗余与潜在漂移

## In Scope

- 彻底移除 `evaluate_note` 的残留代码、类型、文档与测试引用
- 盘点当前仍然有效的任务集合，并形成一份明确的“活跃任务真相”
- 回溯 `Platform API` 禁用 regenerate 类任务的产品原因，并收口为明确的现行决策
- 统一前端、后端、pricing、docs、测试中对 hosted 任务面的表达
- 将 regenerate 类任务正式纳入 hosted 产品路径，使 `My API` 与 `Platform API` 在任务面上保持一致
- 为 regenerate 类 hosted 任务补齐必要的 hosted action、计费语义、前端 guardrail 与文档表达
  - 当前锁定方向是将 `regenerate_chunk` 与 `regenerate_followup` 统一映射到 hosted action `card_regenerate`
  - `card_regenerate` 固定消耗 `1 credit`

## Out of Scope

- 重做模型能力注册表
- 引入新的生成任务或新的 hosted 产品线
- parser、payment、credits、auth 的新扩展
- 改写教学 prompt 目标
- 以“顺手重构”为名扩展到无关模块

## Inherited Project Decisions

- `Platform API` 是登录 + credits 的 hosted 产品路径
- Phase 06 锁定的 hosted action pricing 只覆盖：
  - `Analyze`
  - `Follow-up`
  - `Quiz generation`
  - `Quiz answer analysis`
- Phase 06 明确把 `regenerate_chunk / regenerate_followup / evaluate_note` 视为未进入 hosted pricing scope 的 secondary actions
- 模型能力与产品策略在 Phase 09 之后应保持分离

## Phase-Specific Locked Decisions

- `evaluate_note` 作为过期功能必须彻底删除，不再保留“以后也许会恢复”的名义残留
- regenerate 类任务必须回到与 `My API` 一致的任务面，正式纳入 hosted 产品路径
- 不得再让 regenerate 的 hosted 可用性以隐式 if/else 存在
- hosted 任务面差异属于产品策略，不得再次回流到模型能力层
- regenerate 纳入 hosted 后，必须同步定义清楚其计费语义与交互语义，不能只删除拦截而不补产品定义
- 当前 regenerate hosted 语义锁定为对象导向而非笼统动词导向：
  - task 仍保持 `regenerate_chunk` / `regenerate_followup`
  - hosted action 统一为 `card_regenerate`
  - `card_regenerate = 1 credit`

## Overrides Or Exceptions

- None

## Agent Discretion

- 可以自主决定 `evaluate_note` 清理的具体顺序，只要最终不留下死引用
- 可以自主决定 hosted 任务面在文档与代码中的单一真相位置，只要不会再出现多处定义相互漂移
- 可以自主决定前端 guardrail 的最小交互形态，只要用户能理解当前边界

## Success Criteria

- `evaluate_note` 在代码、类型、文档、测试中都不再残留
- 当前活跃任务集合有清晰单一真相，不再混有过期任务
- `Platform API` 与 `My API` 在 regenerate 类任务上的关系变得明确、可解释、可维护
- hosted pricing、hosted action 类型、后端错误码、前端 guardrail、API 文档保持一致
- 开发者后续在讨论“模型能否支持某任务”时，不再需要先穿过遗留任务与旧产品边界的噪音

## Constraints

- 不得破坏当前已稳定的 `Analyze / Follow-up / Quiz` hosted 路径
- 不得把本 phase 重新扩大成模型能力治理或 payment phase
- 不得保留“死功能先留着以防万一”的模糊状态
- 如果决定开放 regenerate 类 hosted 任务，必须同步补齐产品与计费语义，而不是只去掉后端拦截

## Canonical References

- `docs/discuss/project-brief.md`
- `.planning/PROJECT.md`
- `.planning/ROADMAP.md`
- `docs/discuss/phases/06-login-hosted-access-and-credit-brief.md`
- `docs/backend/api-design.md`
- `docs/frontend/data-flow.md`
- `docs/changelog/CHANGELOG_TECH.md`
- `GGlearn-AI/api/lib/generateService.ts`
- `GGlearn-AI/api/lib/platformAccess/types.ts`
- `GGlearn-AI/src/lib/platformAccess/pricing.ts`
- `GGlearn-AI/src/hooks/useChunkRegenerate.ts`
- `GGlearn-AI/src/hooks/useFollowUp.ts`

## Open Questions

- hidden help / credits 栏位的前端收纳交互是否与本 phase 一起最小落地，还是只先完成 action/pricing/guardrail 对齐

## Deferred Ideas

- 更大范围的任务体系重命名或 prompt taxonomy 整理
- 为 hosted 与 BYOK 做更激进的 UI 差异化

## Impact On Existing Plans

- Requires replanning because the current hosted-task boundary still carries explicit Phase 06 deferrals and dead-task residue that should not survive once model capability and product task surfaces are being formalized.

## Next Step

Run `gsd-plan-phase 10 --prd docs/discuss/phases/10-dead-task-cleanup-and-hosted-task-surface-alignment-brief.md` once this brief is approved.
