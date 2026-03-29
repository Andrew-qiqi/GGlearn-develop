# SlideTutor 文档中心

欢迎来到 SlideTutor 项目文档中心。本目录采用模块化组织，按功能领域分类，便于查找和维护。

最后更新：2026-03-28

---

## 📚 文档模块

### 🏗️ 架构模块 (`architecture/`)

系统架构、技术栈和部署相关文档。

- **[system-overview.md](architecture/system-overview.md)** - 系统概述、设计原则和架构层次
- **[tech-stack.md](architecture/tech-stack.md)** - 技术栈详解（前端、后端、基础设施）
- **[deployment.md](architecture/deployment.md)** - 部署架构、性能优化和监控告警

### 🎨 前端模块 (`frontend/`)

前端架构、状态管理和数据流相关文档。

- **[architecture.md](frontend/architecture.md)** - 前端架构、状态管理和 API 客户端
- **[data-flow.md](frontend/data-flow.md)** - 数据流和持久化策略

### ⚙️ 后端模块 (`backend/`)

后端 API 设计和错误处理相关文档。

- **[api-design.md](backend/api-design.md)** - API 设计规范和核心端点
- **[error-handling.md](backend/error-handling.md)** - 错误处理策略和横切关注点

### 🔒 安全模块 (`security/`)

安全架构、Token 认证和速率限制相关文档。

- **[architecture.md](security/architecture.md)** - 安全架构和多层防御策略
- **[token-authentication.md](security/token-authentication.md)** - API Token 认证系统详解
- **[rate-limiting.md](security/rate-limiting.md)** - 速率限制策略

### 📝 变更日志模块 (`changelog/`)

技术变更历史记录。

- **[CHANGELOG_TECH.md](changelog/CHANGELOG_TECH.md)** - 技术变更日志

### 👥 用户指南模块 (`user_guide/`)

面向最终用户的使用指南。

- **[cli-commands.md](user_guide/cli-commands.md)** - CLI 命令使用指南

---

## 🚀 快速导航

### 我是新开发者

如果你是新加入的开发者，建议按以下顺序阅读：

1. [系统概述](architecture/system-overview.md) - 了解整体架构和设计原则
2. [技术栈](architecture/tech-stack.md) - 了解使用的技术和工具
3. [前端架构](frontend/architecture.md) - 了解前端实现
4. [API 设计](backend/api-design.md) - 了解后端接口
5. [安全架构](security_new/architecture.md) - 了解安全机制

### 我要实现新功能

1. 查看 [系统概述](architecture/system-overview.md) 了解架构层次
2. 根据功能类型查看对应模块文档（前端/后端）
3. 参考 [数据流](frontend/data-flow.md) 了解数据如何流转
4. 完成后更新 [技术变更日志](changelog/CHANGELOG_TECH.md)

### 我要修复安全问题

1. 查看 [安全架构](security/architecture.md) 了解现有安全机制
2. 查看 [Token 认证](security/token-authentication.md) 了解认证实现
3. 修复后更新相关文档和变更日志

### 我要部署项目

1. 查看 [部署架构](architecture/deployment.md) 了解部署配置
2. 查看 [技术栈](architecture/tech-stack.md) 了解依赖要求
3. 参考环境变量配置说明

---

## 📖 文档维护

### ⚠️ 重要：请先阅读维护规范

**在修改任何文档之前，请务必阅读 [DOCUMENTATION_RULES.md](DOCUMENTATION_RULES.md)**

该文件定义了：
- 严格的目录结构规则
- 各模块的职责和更新时机
- 禁止的文档组织方式
- 文档更新流程和最佳实践

### 文档更新原则

- 所有技术文档应保持与代码同步更新
- 重大功能变更需要更新相应模块文档
- 安全相关变更必须记录在 `security/` 模块
- 架构变更必须更新 `architecture/` 模块
- 所有变更都应在 `changelog/CHANGELOG_TECH.md` 中记录
- **禁止**在 docs 下创建临时文件或草稿目录

### 使用 maintain-tech-docs 技能

项目配置了 `maintain-tech-docs` 技能，可以帮助你：

- 在完成功能后更新相关文档
- 记录架构决策和技术变更
- 保持文档与代码同步

使用方法：
```bash
/maintain-tech-docs
```

该技能已适配模块化文档结构，会自动引导你更新正确的模块。

---

## 🔗 相关资源

- **项目主页**：https://www.slidetutor-ai.com
- **GitHub 仓库**：[SlideTutor-AI](https://github.com/Andrew-qiqi/SlideTutor-AI-main)
- **问题反馈**：通过应用内反馈功能或 GitHub Issues

---

## 📝 版本信息

- **文档版本**：v2.0（模块化重组）
- **最后更新**：2026-03-28
- **维护者**：SlideTutor 开发团队

---

*如有任何文档相关问题或建议，欢迎通过 GitHub Issues 反馈。*
