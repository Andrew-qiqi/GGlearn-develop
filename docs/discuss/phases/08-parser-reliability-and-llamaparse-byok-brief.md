# Phase 08 Brief: Parser Reliability and LlamaParse BYOK

## Metadata

- Status: Ready for GSD planning
- Phase: 08
- Related Roadmap Entry: `.planning/ROADMAP.md` (next phase proposal)
- Last Updated: 2026-04-09
- Owner: Agent-authored, user-approved
- Impacts Existing Plans: Yes
- Change Summary: 基于 Phase 07 之后的深度讨论，下一阶段不再延续“平台 parser 每日配额 + BYOK 借用平台 parser”的旧思路，而是同时完成两件事：一是把现有 Volcengine 平台 parser 收敛为真正可用、可解释、低摩擦的稳定路径；二是立即为 `My API` 增加 `LlamaParse` 作为独立 parser BYOK 路径。

## Objective

完成 parser 体系的这一次真正收口：

- 平台侧继续使用 `Volcengine`，但去掉当前错误的产品配额与误导性的 429 体验
- `My API` 不再借平台 parser，而是正式支持用户配置自己的 `LlamaParse`
- 保持当前 explain 主链路、`LayoutBlock[]` 契约、降级分析能力和整体代码结构清晰可维护

## Problem

当前 parser 体系已经不是“还能不能用”的问题，而是“结构边界和真实产品行为已经不一致”：

- 平台 parser 仍然带着 `10/day/network` 这样的产品配额，导致长文档自动分析天然失败
- `BYOK` 现在实际上还会先尝试平台 parser，和之前已经讨论过的产品边界冲突
- 前端把多种不同来源的 429 混成同一种 `15 RPM` 文案，严重误导排查与用户理解
- `Platform API` 既然已经是 credits 模式，再叠 parser 产品配额，在产品逻辑上不成立
- `My API` 侧已经明确需要 parser BYOK，但当前还没有独立 parser 配置轴

如果这一轮不收口，后续会继续出现：

- 用户不知道自己到底是撞了平台配额、全局限流，还是上游服务限流
- `BYOK` 和 `Platform API` 在 parser 成本边界上继续相互污染
- 后续接入 `LlamaParse` 时，不得不在旧的 parser access 逻辑上继续打补丁

## In Scope

- 移除平台 parser 的每日产品配额与对应的用户侧“剩余额度”语义
- 移除 `BYOK` 借用平台 parser 的隐式体验
- 重新梳理 parser 错误语义，拆开：
  - 平台 parser 不可用
  - Worker / route 级限流
  - 上游 provider 级限流或失败
- 去掉前端误导性的 `15 RPM` 通用 429 提示
- 保留“无 parser 的降级分析”作为 `My API` 未配置 parser 时的默认路径
- 为 `My API` 增加独立 parser 配置能力，首个 provider 锁定为 `LlamaParse`
- 在后端增加 `LlamaParse` parser provider / adapter
- 将 `LlamaParse` 输出按现有主链路需要归一化到 `LayoutBlock[]` 或可兼容的最小结构
- 保持平台 parser 仍然默认使用 `Volcengine`
- 补齐相关文档、测试、配置和迁移说明

## Out of Scope

- 替换平台 parser provider，不把 `Volcengine` 换成 `LlamaParse`
- 引入第二个 parser BYOK provider
- 实现 `LiteParse`、`Docling` 或其他自部署 parser
- 将 parser 做成 marketplace / provider 选择中心
- 在本 phase 内处理 model 配置稳态问题（包括 Gemini thinking 配置）
- 重做整个 Worker 路由架构或引入重型分布式队列系统

## Inherited Project Decisions

- `My API` 和 `Platform API` 已经是明确分离的两条产品路径
- `Platform API` 使用登录 + credits，而不是订阅制
- 平台 parser 当前默认 provider 是 `Volcengine`
- 解析失败时，产品必须继续可用；能降级时优先降级，而不是彻底阻断用户
- 现有 explain 主链路依赖的是结构化的 parser 结果被收敛到 `LayoutBlock[]`，而不是某个 provider 的原始响应格式
- 代码结构目标仍然是简单、鲁棒、统一、可维护，不为未来扩展过度设计

## Phase-Specific Locked Decisions

- 平台 parser 的 `10/day/network` 产品配额删除，不再保留任何 parser 每日产品限额
- 之前讨论中的 `BYOK` parser 试用路径正式取消；`BYOK` 不再隐式消耗平台 parser
- `My API` 现在就接入 `LlamaParse`，不再“以后再说”
- `My API` 如果没有配置 parser，则继续沿用当前“无 parser 的降级分析”路径
- 平台 parser 仍然锁定为 `Volcengine`，本 phase 的目标是修稳而不是替换
- 不再向用户暴露 parser provider 的平台内部细节，`Platform API` 不允许用户自配 parser
- parser 相关保护仅保留为内部基础设施保护，不再表现成用户可见的产品配额

## Overrides Or Exceptions

- 相对于 `project-brief.md` 和旧的 Phase 05 讨论记录中“parser BYOK 继续 deferred / MinerU 仅作后续候选”的表述，本 phase 明确例外为：parser BYOK 不再延后，且首个正式接入对象改为 `LlamaParse`。

## Agent Discretion

- 可以自主决定平台 parser 内部保护采用“极薄的节流 / 短退避 / 错误分类”中的具体最小实现，只要不重新引入用户可见的 parser 配额
- 可以自主决定 `LlamaParse` 结果归一化到 `LayoutBlock[]` 的最小映射策略，只要 explain 主链路不被迫整体重写
- 可以自主决定 `My API` parser 配置在前端的最小 UI 形态，只要保持低摩擦和边界清晰
- 可以自主决定是否保留 `/api/parser-usage` 这类历史运营接口，只要最终产品语义不再依赖 parser 配额

## Success Criteria

- `Platform API` 下的长文档自动分析不再因为 parser 每日产品配额而天然失败
- `BYOK` 请求不再消耗平台 parser，也不再读取平台 parser 配额语义
- 用户看到的 429 / parser 错误信息能真实反映问题来源，而不是继续显示误导性的 `15 RPM`
- `My API` 用户可以配置 `LlamaParse` 并在 explain 主链路中实际使用
- 未配置 `LlamaParse` 的 `My API` 用户仍然可以正常走“无 parser 的降级分析”
- explain 主链路仍然保持现有 `LayoutBlock[] -> prompt cognitive map -> knowledgeCards.intent` 的总体结构
- 代码边界比现在更清楚，而不是把新 parser 继续塞进旧的 platform parser access 逻辑里

## Constraints

- 不得破坏当前成熟的 explain / distill / follow-up / quiz 主链路
- 不得重新引入用户感知的 parser 每日配额、网络配额或试用配额
- 必须兼容当前 Cloudflare Worker 路线
- 不得把 parser phase 和 model 配置 phase 混做在一起
- 任何为 `LlamaParse` 增加的结构都应当保持可插拔，避免把 parser BYOK 逻辑焊死在单一 provider 细节上

## Canonical References

- `docs/discuss/project-brief.md`
- `.planning/PROJECT.md`
- `.planning/ROADMAP.md`
- `docs/discuss/phases/05-parser-bootstrap-and-provider-abstraction-brief.md`
- `docs/discuss/phases/07-china-user-operational-fit-brief.md`
- `docs/backend/api-design.md`
- `GGlearn-AI/api/lib/generateService.ts`
- `GGlearn-AI/api/lib/parser/accessService.ts`
- `GGlearn-AI/api/lib/parser/usageStore.ts`
- `GGlearn-AI/api/lib/parser/volcengineProvider.ts`
- `GGlearn-AI/src/hooks/useSlideAnalysis.ts`
- `GGlearn-AI/src/worker/routes/generate.ts`
- `tmp_files/volcengine_document_parse_intellgence/2.md`
- `https://developers.llamaindex.ai/python/cloud/llamaparse/api-v2-guide/`
- `https://developers.llamaindex.ai/python/cloud/general/rate_limits/`
- `https://developers.llamaindex.ai/python/cloud/general/pricing/`

## Open Questions

- `LlamaParse` 的结构化结果中，`items` / `metadata` 是否能稳定提供足够直接的数值坐标字段，以无损映射到当前 `LayoutBlock[]`
- 现有 `/api/generate` 的全局限流是否需要按 `Platform API` / `My API` 分开处理，还是只需放宽并保留为最薄 abuse guard
- `/api/parser-usage` 与设置页中的 parser 配额可视化是否应彻底删除，还是转为 operator-only 语义

## Deferred Ideas

- `LiteParse` 自建 parser 服务
- `Docling` / 自部署 parser 路线
- 为 `Platform API` 增加第二个 parser provider
- model 配置与 Gemini thinking 稳态修复

## Impact On Existing Plans

- Requires replanning because the old Phase 05 assumptions are no longer valid for the live product: parser quota is no longer a desired product behavior, parser BYOK is no longer deferred, and the next parser phase now combines Volcengine hardening with immediate `LlamaParse` BYOK integration.

## Next Step

Recommended route after this brief is approved:

1. Add the next roadmap phase for parser stabilization and BYOK parser support.
2. Run `gsd-plan-phase 08 --prd docs/discuss/phases/08-parser-reliability-and-llamaparse-byok-brief.md`.
