# 文档维护规范

**最后更新：2026-03-29**

本文档定义了 `/docs` 目录的组织规则和维护标准，确保文档结构清晰、一致，避免混乱。

---

## 📁 目录结构（严格执行）

```
docs/
├── README.md                    # 文档中心入口，提供导航
├── DOCUMENTATION_RULES.md       # 本文件，定义维护规范
│
├── architecture/                # 系统架构模块
│   ├── README.md               # 模块索引
│   ├── system-overview.md      # 系统概述、设计原则
│   ├── tech-stack.md           # 技术栈详解
│   └── deployment.md           # 部署架构、性能优化
│
├── frontend/                    # 前端架构模块
│   ├── README.md               # 模块索引
│   ├── architecture.md         # 前端架构、状态管理
│   └── data-flow.md            # 数据流和持久化
│
├── backend/                     # 后端架构模块
│   ├── README.md               # 模块索引
│   ├── api-design.md           # API 设计规范
│   └── error-handling.md       # 错误处理策略
│
├── security/                    # 安全架构模块
│   ├── README.md               # 模块索引
│   ├── architecture.md         # 安全架构总览
│   ├── token-authentication.md # Token 认证系统
│   └── rate-limiting.md        # 速率限制策略
│
├── changelog/                   # 技术变更日志模块
│   ├── README.md               # 模块索引
│   └── CHANGELOG_TECH.md       # 技术变更历史（核心文件）
│
└── user_guide/                  # 用户指南模块（可选）
    └── README.md               # 模块索引
```

---

## 🚫 严格禁止

### 1. 禁止的目录和文件
- ❌ **临时文件**：不允许 `temp/`, `draft/`, `wip/`, `superpowers/` 等临时目录
- ❌ **个人笔记**：不允许 `notes/`, `scratch/`, `ideas/` 等个人笔记目录
- ❌ **规划文件**：不允许 `plans/`, `specs/`, `proposals/` 等规划文档（应放在 `.claude/plans/` 或项目根目录）
- ❌ **备份文件**：不允许 `.bak`, `.old`, `.backup` 等备份文件
- ❌ **非 Markdown 文件**：除非有特殊需求（如图片），否则只允许 `.md` 文件

### 2. 禁止的文档组织方式
- ❌ **按日期组织**：不允许 `2026-03/`, `week-12/` 等按时间组织的目录
- ❌ **按作者组织**：不允许 `john/`, `team-a/` 等按作者组织的目录
- ❌ **混合语言**：同一模块内的文档应使用统一语言（中文或英文）
- ❌ **重复内容**：不允许在多个地方维护相同内容的文档

---

## ✅ 模块职责定义

### `architecture/` - 系统架构
**职责**：记录系统整体设计、技术选型、部署架构
**更新时机**：
- 添加新的系统模块或服务
- 更换技术栈或框架
- 修改部署架构或基础设施
- 做出重大架构决策

**文件说明**：
- `system-overview.md`：系统概述、核心能力、设计原则、架构层次
- `tech-stack.md`：前端、后端、基础设施的技术选型和理由
- `deployment.md`：Vercel 部署、环境变量、性能优化、监控告警

---

### `frontend/` - 前端架构
**职责**：记录前端实现细节、状态管理、数据流
**更新时机**：
- 修改状态管理方案（Zustand store 结构）
- 改变组件架构或设计模式
- 调整数据流或持久化策略
- 添加新的 API 客户端逻辑

**文件说明**：
- `architecture.md`：前端架构、状态管理、API 客户端、主题系统
- `data-flow.md`：数据流向、持久化策略、缓存机制

---

### `backend/` - 后端架构
**职责**：记录后端 API 设计、错误处理、服务端逻辑
**更新时机**：
- 添加新的 API 端点
- 修改请求/响应格式
- 改变错误处理策略
- 调整服务端中间件

**文件说明**：
- `api-design.md`：API 设计规范、核心端点、请求/响应格式
- `error-handling.md`：错误处理策略、日志记录、横切关注点

---

### `security/` - 安全架构
**职责**：记录安全机制、认证授权、防护策略
**更新时机**：
- 实现新的安全机制
- 修改认证或授权逻辑
- 调整速率限制策略
- 发现并修复安全漏洞

**文件说明**：
- `architecture.md`：安全架构总览、多层防御策略
- `token-authentication.md`：Token 认证系统详解、实现细节
- `rate-limiting.md`：速率限制策略、IP 黑名单

---

### `changelog/` - 技术变更日志
**职责**：记录所有重要技术变更的历史
**更新时机**：**每次完成功能、重构、或复杂 bug 修复后都必须更新**

**文件说明**：
- `CHANGELOG_TECH.md`：技术变更历史，按时间倒序排列

**条目格式**（严格执行）：
```markdown
## [YYYY-MM-DD] 简短标题

**What**: 具体变更内容

**Why**: 技术理由或业务原因

**Impact**:
- 性能影响
- 破坏性变更
- 迁移说明
- 新增依赖

**Files**: 关键文件列表
```

---

### `user_guide/` - 用户指南（可选）
**职责**：面向最终用户的使用指南
**更新时机**：
- 添加新的用户功能
- 修改用户界面或交互流程
- 提供 CLI 或工具使用说明

**注意**：此模块是可选的，如果项目没有面向用户的文档需求，可以删除此目录。

---

## 📝 文档更新流程

### 1. 完成代码变更后
```bash
# 1. 确定影响的模块
# 2. 更新对应模块的文档
# 3. 在 changelog/CHANGELOG_TECH.md 中添加条目
# 4. 与代码一起提交
```

### 2. 使用 maintain-tech-docs skill
```bash
# 在完成功能后调用
/maintain-tech-docs
```

该 skill 会引导你：
- 识别需要更新的模块
- 提供文档模板
- 确保 changelog 条目完整

### 3. 文档审查清单
- [ ] 更新了正确的模块文档
- [ ] 添加了 changelog 条目（包含 What/Why/Impact）
- [ ] 日期格式正确（YYYY-MM-DD）
- [ ] 标注了破坏性变更（如有）
- [ ] 记录了新增依赖（如有）
- [ ] 语言统一（中文或英文）
- [ ] 没有临时文件或草稿

---

## 🔧 维护命令

### 检查文档结构
```bash
# 列出所有文档文件
find docs -type f -name "*.md" | sort

# 检查是否有非 Markdown 文件
find docs -type f ! -name "*.md"

# 检查是否有临时目录
find docs -type d -name "temp" -o -name "draft" -o -name "wip"
```

### 清理临时文件
```bash
# 删除临时目录
rm -rf docs/temp docs/draft docs/wip docs/superpowers

# 删除备份文件
find docs -name "*.bak" -o -name "*.old" -delete
```

---

## 🎯 最佳实践

### 1. 模块化原则
- 每个模块只负责一个领域
- 模块之间通过链接引用，不重复内容
- 每个模块必须有 README.md 作为索引

### 2. 及时更新原则
- 代码变更和文档更新应在同一个 commit
- 不要积累文档债务
- 使用 maintain-tech-docs skill 辅助更新

### 3. 清晰简洁原则
- 使用清晰的标题和结构
- 提供代码示例和实际用例
- 避免冗长的描述，突出关键信息

### 4. 版本控制原则
- 所有文档都应纳入 Git 版本控制
- 重大变更应在 changelog 中记录
- 使用有意义的 commit message

---

## ⚠️ 违规处理

如果发现以下情况，应立即清理：

1. **临时文件或目录**：立即删除
2. **重复内容**：合并到正确的模块
3. **过时文档**：更新或删除
4. **混乱的组织**：重新组织到正确的模块

---

## 📞 问题反馈

如果对文档结构有疑问或建议，请：
1. 在团队会议中讨论
2. 提交 GitHub Issue
3. 更新本规范文档

---

**记住**：良好的文档是项目长期维护的基石。严格执行这些规范，保持文档清晰、一致、有用。
