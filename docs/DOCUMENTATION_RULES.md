# 文档维护规范

最后更新：2026-04-07

本文件定义 `/docs` 目录的组织方式、职责边界和维护要求，目标是让文档结构长期清晰、可发现、可维护。

## 当前目录结构

```text
docs/
├── README.md
├── DOCUMENTATION_RULES.md
├── architecture/
├── frontend/
├── backend/
├── security/
├── operations/
├── changelog/
├── user_guide/
├── discuss/
└── superpowers/
```

说明：

- `operations/` 用于运维检查表、验证报告、支持链路文档。
- `discuss/` 用于项目讨论稿、brief、阶段讨论沉淀。
- `superpowers/` 是历史工作流产物归档区，默认不作为新文档落点；只有在维护已有历史资料时才更新它。

## 模块职责

### `architecture/`

记录系统结构、技术选型、部署方式和基础设施约束。

适合放：

- 系统总览
- 技术栈说明
- 部署架构

### `frontend/`

记录前端架构、状态管理、组件组织和数据流。

适合放：

- 前端架构说明
- 前端数据流说明

### `backend/`

记录 API 设计、后端契约、错误处理和服务端行为。

适合放：

- API 设计
- 错误处理约定

### `security/`

记录认证、授权、Token、限流和安全边界。

### `operations/`

记录线上运维、部署核验、支付回调核验、支持排障和证据型报告。

适合放：

- operator checklist
- operational report
- support lookup/runbook

### `changelog/`

记录重要技术变更，按时间倒序维护。

### `user_guide/`

记录面向最终用户的低噪音使用说明，不放运维细节。

### `discuss/`

记录项目 brief、阶段 brief、设计讨论前置稿。

### `superpowers/`

历史归档区。除非是在维护现有历史资料，否则不要把新文档放进这里。

## 明确禁止

不要在 `/docs` 下新增这些目录或用法：

- `temp/`
- `draft/`
- `wip/`
- `notes/`
- `scratch/`
- `ideas/`
- 以作者名、日期、周次命名的散乱目录

不要这样组织文档：

- 同一主题在多个模块重复维护
- 用户文档和运维文档混放
- 把短期讨论稿直接塞进 `architecture/`、`backend/` 等长期模块

## 新文档该放哪里

按内容性质决定，而不是按“谁写的”决定：

- 长期技术真相：放对应模块
- 用户怎么用：放 `user_guide/`
- 运维怎么验证、怎么排障：放 `operations/`
- 讨论、brief、阶段前置思考：放 `discuss/`
- 历史工作流归档：仅在必要时维护 `superpowers/`

## 更新规则

每次重要代码或架构变化后，至少检查这些问题：

1. 对应模块文档是否需要同步更新。
2. `docs/README.md` 是否仍然能正确导航到新文档。
3. `CHANGELOG_TECH.md` 是否需要补一条记录。
4. 文档中的 `Last updated` 或“最后更新”日期是否仍然正确。

## 文档质量要求

- 优先写“事实”和“边界”，不要写口头记忆。
- 用户文档尽量低噪音，不塞运维细节。
- 运维文档要可执行，最好是 checklist 或明确步骤。
- 报告类文档要区分观察事实、推断、当前结论和延后项。
- 如果一个文档已经变成历史记录，应明确说明它是归档还是当前真相。

## Docs Hub 要求

`docs/README.md` 必须始终能回答三个问题：

1. 我该从哪里了解系统？
2. 我该从哪里排查线上问题？
3. 我该从哪里看用户侧接入方式？

如果新增模块或关键文档，记得同步更新 `docs/README.md`。

## Changelog 要求

`docs/changelog/CHANGELOG_TECH.md` 用于记录重要技术变化。满足下面任一条件时应更新：

- 改了系统边界或模块职责
- 新增或替换了关键服务
- 修改了 API 契约或错误语义
- 新增了重要运维流程或验证结论

## 维护完成前检查

- [ ] 新文档放在正确模块
- [ ] 旧文档没有留下冲突表述
- [ ] `docs/README.md` 已同步导航
- [ ] 模块 README 已更新
- [ ] 需要的话已补 `CHANGELOG_TECH.md`

## 一句话原则

`/docs` 记录的是项目长期真相，不是临时聊天记录。
新内容要么进入正确模块，要么进入 `discuss/`，不要让文档结构重新变乱。
