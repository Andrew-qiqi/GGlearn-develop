# GGlearn 多AI协作开发指南

## 协作模式概述

GGlearn采用三AI协作开发模式，各司其职：

```
用户需求
    ↓
Claude Code (架构师 + 规划师)
    ↓
  制定计划 → Codex (实现者 + 调试专家)
    ↓           ↓
  审查方案 ← 实现代码
    ↓
  UI需求 → Gemini (UI/UX设计师)
    ↓           ↓
  审查设计 ← 设计实现
    ↓
  交付用户
```

---

## 角色定义

### 1. Claude Code（我）

**定位**：架构师 + 规划师 + 技术决策者

**职责**：
- ✅ 需求分析和产品规格定义
- ✅ 技术方案设计和架构决策
- ✅ 开发计划制定和任务分解
- ✅ 代码架构审查和重构建议
- ✅ 风险评估和应对策略
- ✅ 跨模块集成方案
- ✅ 技术选型和验证

**工作流**：
1. 与用户讨论需求，明确产品形态
2. 制定详细的开发计划（ROADMAP.md）
3. 为每个Phase创建详细的PLAN.md
4. 将任务分配给Codex或Gemini
5. 审查实现结果，提供反馈
6. 协调多AI协作，确保一致性

**输出物**：
- `PRODUCT_SPEC.md`：产品规格说明书
- `ROADMAP.md`：开发路线图
- `PLAN.md`：每个Phase的详细计划
- 技术方案文档
- API设计文档
- 架构审查报告

---

### 2. Codex

**定位**：实现者 + 调试专家 + 质量保证

**职责**：
- ✅ 根据PLAN.md实现具体功能
- ✅ 编写单元测试和集成测试
- ✅ Bug修复和性能优化
- ✅ 代码质量保证（lint、type check）
- ✅ 方案可行性验证和反馈
- ✅ 技术调研和竞品分析

**调用方式**：
```bash
# 功能实现
codex exec "根据 ROADMAP.md Phase X 的任务清单，实现 [具体功能]。要求：[具体要求]"

# Bug修复
codex exec "修复 [文件路径] 中的 [问题描述]。已知现象：[现象]。预期行为：[预期]"

# 技术调研
codex exec "调研 [技术/产品] 的实现方式，重点关注：[关注点]。输出调研报告"

# 测试编写
codex exec "为 [文件路径] 编写单元测试，覆盖 [场景]"
```

**工作流**：
1. 接收Claude Code分配的任务
2. 阅读PLAN.md，理解需求和技术方案
3. 实现功能代码
4. 编写测试，确保覆盖率
5. 运行lint和type check
6. 提交代码，报告完成情况
7. 如遇问题，反馈给Claude Code

**输出物**：
- 功能代码
- 单元测试和集成测试
- Bug修复报告
- 技术调研报告
- 性能优化报告

---

### 3. Gemini

**定位**：UI/UX设计师 + 前端视觉专家

**职责**：
- ✅ 前端组件视觉设计
- ✅ 用户体验优化
- ✅ 响应式布局实现
- ✅ 设计系统一致性维护
- ✅ 可访问性改进
- ✅ 动画和交互效果

**调用方式**：
```bash
# UI设计
gemini --prompt "设计 [组件名称] 的UI。要求：[设计要求]。参考：[参考文件]"

# 视觉优化
gemini --prompt "优化 [文件路径] 的视觉效果。当前问题：[问题]。期望：[期望]"

# 响应式适配
gemini --prompt "为 [组件] 实现响应式布局，适配桌面/平板/手机"

# 设计系统
gemini --prompt "基于 [设计规范] 统一 [模块] 的设计语言"
```

**工作流**：
1. 接收Claude Code的UI需求
2. 理解功能和用户场景
3. 设计UI方案（可使用设计技能）
4. 实现前端代码
5. 确保设计一致性
6. 提交代码，展示效果
7. 根据反馈迭代优化

**输出物**：
- UI组件代码
- 样式文件（CSS/Tailwind）
- 设计规范文档
- 交互效果实现
- 响应式布局

---

## 协作流程

### 标准开发流程

```
1. 需求讨论（用户 + Claude Code）
   ↓
2. 制定计划（Claude Code）
   - 输出：ROADMAP.md, PLAN.md
   ↓
3. 任务分配（Claude Code → Codex/Gemini）
   ↓
4. 并行开发
   - Codex：实现后端逻辑、数据处理、AI集成
   - Gemini：实现UI组件、视觉效果、交互
   ↓
5. 代码审查（Claude Code）
   - 检查架构一致性
   - 提出优化建议
   ↓
6. 集成测试（Codex）
   - 确保前后端协同
   ↓
7. 用户验收（用户 + Claude Code）
   ↓
8. 迭代优化（根据反馈）
```

### Phase开发流程

每个Phase的标准流程：

```
Phase N: [Phase名称]
├── 1. 讨论阶段（Claude Code）
│   └── 输出：Phase目标、技术方案、风险评估
├── 2. 计划阶段（Claude Code）
│   └── 输出：PLAN.md（详细任务清单）
├── 3. 实现阶段（Codex + Gemini）
│   ├── Codex：核心逻辑实现
│   └── Gemini：UI实现
├── 4. 审查阶段（Claude Code）
│   └── 输出：审查报告、优化建议
├── 5. 测试阶段（Codex）
│   └── 输出：测试报告
└── 6. 完成阶段（Claude Code）
    └── 输出：Phase总结、下一步计划
```

---

## 沟通规范

### 任务描述规范

**给Codex的任务**：
```
任务：[简短描述]
背景：[为什么要做这个]
输入：[需要读取的文件/数据]
输出：[期望的产出]
要求：[技术要求、约束条件]
参考：[相关文档、代码]
```

**给Gemini的任务**：
```
任务：[UI组件名称]
场景：[用户使用场景]
功能：[组件功能描述]
设计要求：[视觉风格、交互方式]
约束：[技术约束、兼容性]
参考：[设计规范、现有组件]
```

### 反馈规范

**Codex反馈给Claude Code**：
```
任务：[任务ID]
状态：[完成/遇到问题/需要澄清]
完成内容：[实现了什么]
遇到问题：[问题描述]
建议：[技术建议或方案调整]
```

**Gemini反馈给Claude Code**：
```
任务：[任务ID]
状态：[完成/需要反馈]
实现效果：[截图或描述]
设计决策：[为什么这样设计]
需要确认：[不确定的地方]
```

---

## 文件组织

### 项目结构
```
GGlearn-develop/
├── PRODUCT_SPEC.md          # 产品规格（Claude Code维护）
├── ROADMAP.md               # 开发路线图（Claude Code维护）
├── COLLABORATION.md         # 本文档（Claude Code维护）
├── .planning/               # 计划目录
│   ├── phase-1/
│   │   ├── PLAN.md         # Phase 1计划（Claude Code）
│   │   ├── REVIEW.md       # 审查报告（Claude Code）
│   │   └── SUMMARY.md      # 总结（Claude Code）
│   ├── phase-2/
│   └── ...
├── docs/                    # 技术文档
│   ├── architecture/       # 架构文档（Claude Code）
│   ├── api/                # API文档（Claude Code + Codex）
│   └── design/             # 设计文档（Gemini）
├── GGlearn/                # 主应用代码
│   ├── src/
│   │   ├── components/     # UI组件（Gemini主导）
│   │   ├── lib/            # 核心逻辑（Codex主导）
│   │   ├── store/          # 状态管理（Codex主导）
│   │   └── types/          # 类型定义（Codex主导）
│   └── tests/              # 测试（Codex）
└── reports/                # 调研报告
    ├── notebooklm-analysis.md
    └── ...
```

---

## 质量标准

### 代码质量（Codex负责）
- ✅ 通过TypeScript类型检查
- ✅ 通过ESLint检查
- ✅ 单元测试覆盖率 > 80%
- ✅ 关键路径有集成测试
- ✅ 代码注释清晰（复杂逻辑）

### UI质量（Gemini负责）
- ✅ 符合设计规范
- ✅ 响应式适配（桌面/平板/手机）
- ✅ 可访问性（ARIA标签、键盘导航）
- ✅ 性能优化（懒加载、虚拟滚动）
- ✅ 浏览器兼容性

### 架构质量（Claude Code负责）
- ✅ 模块职责清晰
- ✅ 接口设计合理
- ✅ 可扩展性好
- ✅ 技术选型合理
- ✅ 风险可控

---

## 冲突解决

### 技术分歧
1. Codex或Gemini提出不同方案
2. Claude Code评估各方案优劣
3. Claude Code做最终决策
4. 记录决策理由

### 进度冲突
1. 某个AI遇到阻塞
2. 及时反馈给Claude Code
3. Claude Code调整计划或重新分配
4. 其他AI继续并行工作

### 质量问题
1. 发现代码质量问题
2. Claude Code审查并提出改进建议
3. 责任AI修复
4. 重新审查确认

---

## 当前状态

### 已完成
- ✅ 产品形态讨论和确认
- ✅ 产品规格文档（PRODUCT_SPEC.md）
- ✅ 开发路线图（ROADMAP.md）
- ✅ 协作方式定义（本文档）

### 进行中
- ⏳ Codex调研NotebookLM和LearnAbout
- ⏳ 准备启动Phase 1: 技术清理

### 下一步
1. 等待Codex调研报告
2. 基于调研结果优化ROADMAP
3. 创建Phase 1的详细PLAN.md
4. 开始技术清理工作

---

## 附录：常用命令

### Codex命令
```bash
# 查看版本
codex --version

# 执行任务
codex exec "任务描述"

# 查看帮助
codex --help
```

### Gemini命令
```bash
# 查看版本
gemini --version

# 执行任务
gemini --prompt "任务描述"

# 查看帮助
gemini --help
```

### Git工作流
```bash
# 查看状态
git status

# 提交代码
git add .
git commit -m "feat: 功能描述"

# 推送
git push origin main
```
