# Phase 05 Brief: Parser Bootstrap and Provider Abstraction

## Metadata

- Status: Ready for GSD planning
- Phase: 05
- Related Roadmap Entry: `.planning/ROADMAP.md`
- Last Updated: 2026-04-05
- Owner: Agent-authored, user-approved
- Impacts Existing Plans: Yes
- Change Summary: 将 Phase 05 从“零开始规划 parser guardrails”更新为“在已落地 quota / degraded / settings visibility 基础上，完成平台 parser provider 从 Azure 到 Volcengine 的切换，并清理遗留 Azure 路径”。

## Objective

完成 Phase 05 的剩余关键工作，让 SlideTutor 的平台文档解析能力真正脱离 Azure 默认假设：

- 平台自带 parser 改为 `Volcengine`
- 继续保持平台保底解析 + 服务端额度控制 + degraded fallback
- 保持当前前端依赖的 `LayoutBlock[]` 契约与主要交互不变
- 清理遗留的 Azure 直连路径，避免后续维护出现双轨真相

## Current State

以下基础已经存在，不应在本次 planning 中被重复当成“待从零实现”的内容：

- Cloudflare Worker 路线已经建立
- parser 日额度已经有 D1-backed truth
- parser 使用量已经能在设置页显示
- 超额或不可用时已经有 degraded / `Low accuracy` 降级路径
- parser access 已经有共享 service/provider 抽象雏形

本 phase 剩下的核心问题，不是“要不要做 parser abstraction”，而是“如何把已有 abstraction 真正从 Azure 默认实现收口到 Volcengine，并把旧路径清干净”。

## Problem

虽然 parser 限额、降级和设置页可见性已经落地，但当前代码仍然存在不干净的地方：

- 平台 parser 的实际实现仍然以 Azure 为主
- 旧的直接 `/api/parse` 路径还有 Azure 直连残留
- Phase 05 的规划文档还停留在 guardrail 之前的状态，和真实代码进度不一致

如果现在不收口，后面会很容易出现：

- parser 成本与产品策略已经变了，但代码里仍保留 Azure 旧真相
- 未来做 hosted access 或 payment 时，还要反过来补 parser 技术债
- 文档和 GSD 计划继续误导后续 phase 选择

## In Scope

- 将平台 parser provider 从 `Azure` 切换为 `Volcengine`
- 让共享 parser access layer 默认走 Volcengine
- 清理 `api/generate.ts` 中遗留的 Azure parser 直连路径
- 将 Volcengine 响应映射为现有前端依赖的 `LayoutBlock[]`
- 保持 parser 成功才计入使用量的规则不变
- 保持 quota、degraded fallback、`Low accuracy` 语义不变
- 更新相关 env、docs、tests 与 provider 命名

## Out of Scope

- parser BYOK
- parser provider 选择 UI
- 用户填写自己的 parser provider 凭据
- MinerU adapter 实现
- hosted access 支付与 ZPAY
- 重新设计 parser 商业化套餐
- 扩大当前 `10/day` 规则讨论到更复杂计费体系

## Inherited Project Decisions

- 首个公开版本仍是 `BYOK-first`，但 parser 不是这个 BYOK 的一部分。
- 平台可继续保底提供文档解析，只是必须有明确边界。
- parser 配额真相必须在服务端维护。
- 只有 parser 真正成功时才扣 parser 使用量。
- parser 不可用或超额时，产品继续可用，但降级为较低精度。
- 用户侧不应看到 `Azure` 字样；产品层只暴露 `Document Parsing`。
- 平台自带 parser provider 已锁定为 `Volcengine`。
- 如果未来做 parser BYOK，`MinerU` 是值得优先评估的中国友好候选，但不在本 phase 内实现。

## Phase-Specific Locked Decisions

- 平台 parser 默认 provider 是 `Volcengine`，不是 `Azure`
- `/api/parse` 和 integrated explain 的对外 block 契约继续保持稳定
- parser quota 仍按当前每日 `10` 次页面级成功解析计算
- parser 成功才计入配额，失败或降级不计入
- parser provider 切换不应改变当前设置页的 quota UX
- 本轮不做 parser BYOK，也不让用户看到 parser provider 选择
- 清理遗留 Azure 路径是本 phase 的明确目标，不是“以后再说”的技术债

## Overrides Or Exceptions

None.

## Agent Discretion

- 可以自主决定 Volcengine 响应到 `LayoutBlock[]` 的最小映射策略，只要前端主链路不需要跟着大改。
- 可以自主决定 parser provider 抽象的最小代码边界，只要后续接 MinerU 或其他 provider 时不需要重新拆主链路。
- 可以自主决定是否保留极薄的一层 Azure compatibility helper 作为过渡，只要运行主路径已经不再依赖 Azure。

## Success Criteria

- 平台 parser 主路径不再依赖 Azure
- 旧的 Azure 直连路径被移除或彻底降为非主路径
- 前端现有解释流程、quota 展示和 degraded 行为不被破坏
- parser 使用量仍然只在成功时增加
- 文档、env、测试与实现状态重新一致

## Constraints

- 不得破坏当前成熟的教学链路
- 不得因为 parser provider 切换而把首个公开版本变成高门槛产品
- 必须兼容当前 Cloudflare Worker 路线
- 必须控制变更范围，不把 payment 或 hosted access 一起卷入

## Canonical References

- `docs/discuss/project-brief.md`
- `docs/discuss/phases/06-login-hosted-access-and-credit-brief.md`
- `.planning/ROADMAP.md`
- `.planning/REQUIREMENTS.md`
- `docs/backend/api-design.md`
- `SlideTutor-AI/api/lib/parser/accessService.ts`
- `SlideTutor-AI/api/lib/parser/azureProvider.ts`
- `SlideTutor-AI/api/generate.ts`
- `tmp_files/volcengine_document_parse_intellgence/2.md`
- `tmp_files/volcengine_document_parse_intellgence/3.md`

## Open Questions

- Volcengine 是否需要按页分批或按文件上传来适配当前调用方式，应在 planning 中结合真实接口限制收口。
- 是否保留一个极小的 provider-normalization 类型层，供未来 MinerU BYOK 复用。

## Deferred Ideas

- parser BYOK
- parser provider 多实现切换 UI
- 登录后的跨设备 parser quota 归属
- MinerU adapter
- parser 商业化套餐

## Impact On Existing Plans

- Requires replanning because Phase 05 no longer starts from zero; the remaining work is now provider replacement and cleanup on top of already-landed guardrails.

## Next Step

Run:

`gsd-plan-phase 05 --prd docs/discuss/phases/05-parser-bootstrap-and-provider-abstraction-brief.md`
