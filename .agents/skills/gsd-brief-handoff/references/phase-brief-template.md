# Phase NN Brief: <Phase Name>

## Metadata

- Status: Draft
- Phase: NN
- Related Roadmap Entry: `.planning/ROADMAP.md`
- Last Updated: YYYY-MM-DD
- Owner: Agent-authored, user-approved
- Impacts Existing Plans: Yes | No
- Change Summary:

## Objective

一句话写清这个 phase 这次到底要完成什么。

## Problem

当前存在什么问题，为什么现在要处理它。

不要重复解释整个项目是什么；项目级背景引用 `project-brief.md` 即可。

## In Scope

列出本 phase 这次要解决的内容。

## Out of Scope

列出这次明确不做的内容。

不要把其他 phase 的工作重新写进这里。

## Inherited Project Decisions

这里只列与本 phase 直接相关的项目级原则，使用引用或简短转述即可，不要大段重复 `project-brief.md`。

## Phase-Specific Locked Decisions

列出已经拍板、下游必须遵守的决策。

这里只写本 phase 新增、细化或明确化的决策。

## Overrides Or Exceptions

如果本 phase 对项目级原则有例外，写在这里；如果没有，写 `None`.

## Agent Discretion

列出允许 GSD / Codex / Claude 自主决定的部分。

## Success Criteria

列出完成时必须为真的结果。

## Constraints

列出本 phase 的特殊约束，例如：

- 不得破坏现有行为
- 必须兼容已有数据
- 风格必须延续某个原则

## Canonical References

列出开始本 phase 前必须先读的文档和文件。

至少考虑：

- `docs/discuss/project-brief.md`
- `.planning/PROJECT.md`
- `.planning/ROADMAP.md`
- 相关 `.planning/phases/*-CONTEXT.md`
- 相关设计或需求文档

## Open Questions

如仍有未决点，写在这里；如果没有，写 `None`.

## Deferred Ideas

讨论中出现但不属于本 phase 的内容，写在这里；如果没有，写 `None`.

## Impact On Existing Plans

如果已存在 plan，这里明确写：

- `No impact`
- 或 `Requires replanning because ...`

## Next Step

推荐写法：

`Run gsd-plan-phase <phase> --prd <this-file> once this brief is approved.`
