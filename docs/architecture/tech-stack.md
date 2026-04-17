# 技术栈

本文档描述 GGlearn 项目使用的技术栈，包括前端、后端和基础设施。

最后更新：2026-03-28

---

## 技术栈

### 前端
- **框架**：React 19 + Vite
- **语言**：TypeScript
- **状态管理**：React Hooks（useState、useEffect、自定义 hooks）
- **HTTP 客户端**：Fetch API + 自定义封装（`apiClient.ts`）
- **构建工具**：Vite

### 后端
- **运行时**：Node.js
- **框架**：Express.js
- **语言**：TypeScript
- **文档分析**：Volcengine OCRPdf
- **AI 提供商**：
  - Google Gemini（主要）
  - Doubao（字节跳动）
  - Qwen（阿里巴巴）

### 基础设施
- **托管**：Vercel（Serverless Functions）
- **数据库**：better-sqlite3（本地存储）
- **邮件**：Nodemailer
- **安全**：Helmet.js、CORS、express-rate-limit

---
