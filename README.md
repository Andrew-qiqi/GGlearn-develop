# SlideTutor AI 学习助手

<div align="center">
<img width="1200" height="475" alt="SlideTutor-Banner" src="https://github.com/user-attachments/assets/0aa67016-6eaf-458a-adb2-6e31a0763ed6" />
</div>

SlideTutor AI 是一款基于 Google Gemini AI 驱动的交互式 PDF 学习助手，专为大学生及教育研究者设计。它通过苏格拉底式的引导教学，帮助用户在阅读 PDF 幻灯片时实现深度的知识获取。

## 1. 项目结构 (Project Structure)

本仓库采用多层级的工作流管理结构：

- **`SlideTutor-AI/`**: 核心应用程序源码，包括 React 前端与 Express 后端。
- **`.planning/`**: GSD (Get Shit Done) 系统的核心决策与计划文件。
- **`docs/tech_reference/`**: 技术细节参考文档（架构设计与变更日志）。
- **`.agents/`**: 包含本项目专用的智能体技能定义与工作流模板。

## 2. 核心技术特性 (Core Features)

- **交互式画布 (Interactive Canvas)**: 利用 XYFlow 将 AI 的解释直接叠加在幻灯片之上，实现视觉化的“指引式”教学。
- **苏格拉底式引导 (Socratic Probes)**: AI 不直接给出答案，而是通过逐步引导，锻炼用户的批判性思维。
- **AI 空间感知**: 集成 Azure Document Intelligence，使 AI 能够准确识别幻灯片中的文本块、表格及图像坐标。
- **安全性增强**: 具备自动恶意攻击检测（基于关键词及 AI 分析）与 SMTP 实时安全告警。

## 3. 开发者指南 (Developer Guide)

### 3.1 本地开发
1. 进入应用目录：`cd SlideTutor-AI`
2. 安装依赖：`npm install`
3. 环境配置：将 `.env.local` 复制并重命名为 `.env`，填入 `GEMINI_API_KEY` 等必要密钥。
4. 启动开发服务器：`npm run dev`

### 3.2 技术文档
详见：
- [技术架构文档 (ARCHITECTURE.md)](docs/tech_reference/ARCHITECTURE.md)
- [技术变更日志 (CHANGELOG_TECH.md)](docs/tech_reference/CHANGELOG_TECH.md)

## 4. 协作工作流 (Collaboration Workflow)

本项目深度集成了 GSD 工作流，请务必遵守：
- **`AGENTS.md`**: 通用协作规则。
- **`GEMINI.md`**: Gemini 专属测试与验证准则。
- **`maintain-tech-docs` 技能**: 任何核心逻辑变更后，务必同步更新 `docs/tech_reference`。

---
*Created by Gemini CLI Assistant on 2026-03-28*
