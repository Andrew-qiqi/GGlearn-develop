# SlideTutor AI Workspace Instructions

先遵循仓库根目录 `AGENTS.md` 中定义的通用协作与文档工作流规则；本文件只补充 Gemini 专属要求。

## 沟通语言要求
与用户的沟通（包括给用户看的文档、给用户的回复等），请使用中文，除非用户主动要求使用英文。给模型自己看的文档，请使用英文。

## Technical Documentation Maintenance
- **Mandatory Updating:** Whenever making significant architectural changes, implementing new core features, or refactoring logic, you MUST activate the `maintain-tech-docs` skill and update `docs/tech_reference/ARCHITECTURE.md` and `docs/tech_reference/CHANGELOG_TECH.md` accordingly before concluding your task.
