# SlideTutor 文档中心

最后更新：2026-04-07

这个目录是项目级文档入口，按模块组织，方便查找代码架构、接口约定、运维步骤和产品讨论记录。

## 模块导航

### `architecture/`

- [system-overview.md](architecture/system-overview.md)
- [tech-stack.md](architecture/tech-stack.md)
- [deployment.md](architecture/deployment.md)

### `frontend/`

- [architecture.md](frontend/architecture.md)
- [data-flow.md](frontend/data-flow.md)

### `backend/`

- [api-design.md](backend/api-design.md)
- [error-handling.md](backend/error-handling.md)

### `security/`

- [architecture.md](security/architecture.md)
- [token-authentication.md](security/token-authentication.md)
- [rate-limiting.md](security/rate-limiting.md)

### `user_guide/`

- [README.md](user_guide/README.md)
- [access-modes.md](user_guide/access-modes.md)

### `operations/`

- [README.md](operations/README.md)
- [china-operator-checklist.md](operations/china-operator-checklist.md)
- [china-operational-fit-report.md](operations/china-operational-fit-report.md)

### `changelog/`

- [CHANGELOG_TECH.md](changelog/CHANGELOG_TECH.md)

### `discuss/`

- [project-brief.md](discuss/project-brief.md)
- [phases/](discuss/phases/)

### `superpowers/`

- 历史工作流归档区，仅在维护已有资料时进入

## 快速开始

### 我想了解整体架构

1. [architecture/system-overview.md](architecture/system-overview.md)
2. [architecture/tech-stack.md](architecture/tech-stack.md)
3. [frontend/architecture.md](frontend/architecture.md)
4. [backend/api-design.md](backend/api-design.md)

### 我想部署或排查线上环境

1. [architecture/deployment.md](architecture/deployment.md)
2. [operations/china-operator-checklist.md](operations/china-operator-checklist.md)
3. [backend/api-design.md](backend/api-design.md)

### 我想了解用户侧接入方式

1. [user_guide/access-modes.md](user_guide/access-modes.md)
2. [operations/china-operational-fit-report.md](operations/china-operational-fit-report.md)

## 维护约定

- 修改代码后，同步更新对应模块文档。
- 重要技术变更记录到 [changelog/CHANGELOG_TECH.md](changelog/CHANGELOG_TECH.md)。
- 修改文档结构前，先查看 [DOCUMENTATION_RULES.md](DOCUMENTATION_RULES.md)。
- 运维检查表、线上核验结果和支持排障文档统一维护在 `operations/`。
