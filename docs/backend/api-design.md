# API 设计

本文档描述 SlideTutor 项目的 API 设计规范和核心端点。

最后更新：2026-03-28

---


## API 设计

### 核心端点

#### 1. POST /api/generate
**用途**：AI 内容生成（解释、追问、测验）

**请求头：**
```
X-API-Token: <token>  // 必需（如果启用认证）
Content-Type: application/json
```

**请求体：**
```json
{
  "task": "explain|followup|quiz|regenerate",
  "pageNumber": 1,
  "imageBase64": "data:image/png;base64,...",
  "textContent": "幻灯片文本内容",
  "layoutBlocks": [...],  // Azure 文档智能布局
  "context": {...}        // 上下文信息
}
```

**响应：**
- 流式响应（Server-Sent Events）
- 内容类型：`text/event-stream`

#### 2. GET /api/get-token
**用途**：获取 API 认证 Token

**响应：**
```json
{
  "token": "base64(payload).base64(signature)",
  "expiresIn": 300  // 秒
}
```

#### 3. POST /api/parse
**用途**：使用 Azure Document Intelligence 解析 PDF 布局

**请求体：**
```json
{
  "imageBase64": "data:image/png;base64,..."
}
```

**响应：**
```json
{
  "blocks": [
    {
      "type": "text|table|figure",
      "content": "...",
      "boundingBox": [x1, y1, x2, y2, ...]
    }
  ]
}
```

#### 4. POST /api/feedback
**用途**：收集用户反馈

**请求体：**
```json
{
  "type": "bug|feature|other",
  "message": "用户反馈内容",
  "email": "user@example.com"  // 可选
}
```

---
