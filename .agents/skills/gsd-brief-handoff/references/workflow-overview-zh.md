# Deep Discuss + GSD 工作流

这套工作流只解决一件事：

`当你想在进入 GSD 前先做一轮更深的讨论时，如何把讨论结果变成干净的输入文档。`

核心原则是：

`先把决策写成 brief，再把 brief 交给 GSD。`

但要注意：

- 不是所有任务都要走 GSD
- 不是所有任务都要有 brief
- 进入 GSD 后，后续流程交给 GSD 自己的 `.planning/` 体系

## 1. 角色分工

### Claude

适合做：

- 深度 discuss
- 需求澄清
- 方案取舍
- 生成用户可读的 brief

### Codex

适合做：

- 阅读代码现实
- 识别技术约束
- 校验 phase 划分是否可执行
- 直接落地实现

### GSD

适合做：

- 维护 `.planning/` 文档体系
- 基于结构化输入生成 `CONTEXT / PLAN / VERIFICATION`
- 支持 `/clear` 后的继续执行

结论：

- `GSD` 不是主 deep discuss 工具
- `GSD` 是结构化执行系统
- `brief` 是进入 GSD 之前的接口文档

## 2. 文档分层

仓库内有两层文档。

### A. 上游接口文档

位置：

- `docs/discuss/project-brief.md`
- `docs/discuss/phases/*.md`

用途：

- 给人看
- 记录已拍板决策
- 作为 GSD 的输入源

这是 `pre-GSD interface`。

### B. 下游执行文档

位置：

- `.planning/PROJECT.md`
- `.planning/ROADMAP.md`
- `.planning/REQUIREMENTS.md`
- `.planning/phases/*`

用途：

- 给 GSD 和执行 agent 使用
- 承接 plan / execute / verify
- 记录执行状态与产物

这是 `derived execution artifacts`。

不要把 `.planning` 里的文档当作人类主讨论文档。
也不要把 brief 当作 GSD 执行阶段的主上下文文档。

## 3. 项目级流程

### Step 1: 先做项目级 deep discuss

你和 agent 先把以下问题聊清楚：

- 项目到底是什么
- 核心价值是什么
- 真需求是什么
- 不做什么
- 有哪些约束
- 初步 phase 怎么拆

### Step 2: agent 写 `project-brief.md`

不是你自己手写。

规则是：

- 由 agent 根据讨论结果整理
- 你负责审阅与拍板
- 不要把原始聊天直接交给 GSD

### Step 3: 再执行 `/gsd:new-project`

此时 `project-brief.md` 作为上游参考。

GSD 负责生成或更新：

- `.planning/PROJECT.md`
- `.planning/ROADMAP.md`
- `.planning/REQUIREMENTS.md`

关系如下：

`project-brief -> /gsd:new-project -> .planning 项目文档`

## 4. Phase 级流程

### Step 1: 先有 roadmap

只有当 `.planning/ROADMAP.md` 已存在并确定 phase 编号后，才创建正式的 `phase-xx-brief.md`。

因此：

- `project-brief` 在 `new-project` 之前
- `phase brief` 通常在 `new-project` 之后、该 phase 的 planning 之前

### Step 2: 为某个 phase 做 deep discuss

围绕单个 phase，讨论：

- 这一阶段到底要解决什么
- 成功标准是什么
- 哪些决策必须锁定
- 哪些细节交给 agent 自由裁量
- 有哪些明确不做的内容

### Step 3: agent 写 `phase-xx-brief.md`

文件放在：

- `docs/discuss/phases/NN-phase-slug-brief.md`

例如：

- `docs/discuss/phases/03-ergonomic-layout-visual-consistency-brief.md`

### Step 4: 把 brief 交给 GSD

优先使用：

```bash
gsd-plan-phase 3 --prd docs/discuss/phases/03-ergonomic-layout-visual-consistency-brief.md
```

这表示：

- 深度讨论已经在 brief 中完成
- GSD 此时主要负责把 brief 转成 `CONTEXT / PLAN`

如果仍有少量灰区，再使用 `gsd-discuss-phase` 做补充，而不是从头重聊。

注意：

- 一旦使用 `--prd`，这一轮 planning 会跳过 GSD 原生 discuss
- 所以只有在 brief 已经足够完整时，才直接用 `--prd`

## 5. 进入 GSD 前有 3 种方式

### 方式 A：直接交接

适合：

- 你和 agent 已经讨论得很充分
- phase brief 已足够完整

流程：

`deep discuss -> phase brief -> gsd-plan-phase --prd`

### 方式 B：混合模式

适合：

- 你已经有 phase brief
- 但还想用 GSD 的 discuss 补最后一点灰区

流程：

`deep discuss -> phase brief -> gsd-discuss-phase -> 回写 phase brief -> plan`

这里最重要的一条规则是：

`如果 gsd-discuss-phase 产生了新的锁定决策，必须同步回 phase brief。`

否则 brief 会过时，人与 GSD 的接口会漂移。

### 方式 C：纯 GSD

适合：

- 任务本身不需要额外深聊
- 或你认为 GSD 原生 discuss 已经够用

流程：

`gsd-discuss-phase -> gsd-plan-phase`

## 6. Brief 为什么必须模板化

如果 brief 只是随手笔记，那么：

- 换 agent 后会理解偏差
- GSD 输入会变脏

所以 brief 必须写成“前置接口文档”，而不是“聊天摘要”。

一个合格的 brief，应该让新 agent 一眼回答这些问题：

- 目标是什么
- 当前要解决什么问题
- 范围是什么
- 明确不做什么
- 哪些决策已锁定
- 哪些细节可以自由决定
- 成功标准是什么
- 依赖哪些文档
- 是否影响现有计划
- 下一步动作是什么

## 7. Brief 如何避免重复

要区分 `project brief` 和 `phase brief` 的所有权。

### project brief 只写

- 项目目标
- 核心价值
- 真需求
- 非目标
- 跨 phase 约束
- roadmap 级拆分方向

### phase brief 只写

- 某个 phase 当前要解决的问题
- 该 phase 的边界
- 该 phase 的成功标准
- 该 phase 的锁定决策
- 该 phase 的非范围

规则：

- phase brief 不要重复 project brief
- 项目级原则优先引用，不重复展开
- 只有本 phase 的细化、特化、例外才写出来

## 8. Brief 可以改吗

可以，但不能静默改。

每份 brief 必须带这些字段：

- `Status`
- `Last Updated`
- `Impacts Existing Plans`
- `Change Summary`

推荐状态：

- `Draft`
- `Approved`
- `In Planning`
- `In Execution`
- `Superseded`

规则：

- 说明性补充可以直接更新
- 如果影响 scope、locked decisions、success criteria、constraints，就必须视为重大变更
- 重大变更需要同步判断是否要重新 `plan`

## 9. 什么情况下不需要 brief

以下情况通常不需要 brief：

- 小修复
- 直接代码修改
- 范围很小的一次性需求
- 你明确不打算走 GSD 的任务

## 10. 你实际怎么用

最小执行方式如下：

1. 先聊
2. 让 agent 写 brief
3. 你审 brief
4. 视完整度决定：
   - 直接 `--prd`
   - 或先 `gsd-discuss-phase` 补灰区，再回写 brief
5. 进入 plan / execute / verify

不要反过来。

不要先让 GSD 接原始聊天。

不要把 `.planning` 当作人类主文档。
也不要要求每个任务都先写 brief。

## 11. 当前推荐目录

```text
AGENTS.md
docs/
  discuss/
    project-brief.md
    phases/
      03-ergonomic-layout-visual-consistency-brief.md
.planning/
  PROJECT.md
  ROADMAP.md
  REQUIREMENTS.md
  phases/
    03-ergonomic-layout-visual-consistency/
      03-CONTEXT.md
      03-01-PLAN.md
      03-02-PLAN.md
```

## 12. 一句话总结

`brief 是进入 GSD 之前的人类讨论接口，不是 GSD 执行阶段的替代品。`
