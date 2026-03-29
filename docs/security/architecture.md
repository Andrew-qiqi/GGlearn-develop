# 安全架构

本文档描述 SlideTutor 项目的安全架构和多层防御策略。

最后更新：2026-03-28

---

## 安全架构

SlideTutor 采用多层防御策略保护 API 端点免受未授权访问和滥用。

### 安全层次

```
HTTP 请求
    ↓
[1] Token 验证 ✅ (主要防御)
    - HMAC 签名验证
    - 5分钟过期时间
    - 基于 Nonce 的重放保护
    ↓
[2] Origin/Referer 检查 ✅ (次要防御)
    - 域名白名单验证
    - 生产环境强制执行
    ↓
[3] 速率限制 ✅ (滥用防护)
    - 10 请求/分钟，100 请求/天 每 IP
    - Token 端点独立限制
    ↓
[4] 恶意内容检测 ✅ (内容过滤)
    - 关键词过滤
    - Gemini AI 语义分析
    ↓
[5] 输入验证 ✅ (注入防护)
    - 类型检查
    - 格式验证
    ↓
业务逻辑
```

### Token 认证系统

**Token 结构：**
```
base64(payload).base64(signature)

payload = {
  timestamp: number,  // 当前时间戳（毫秒）
  nonce: string       // 32字符随机十六进制字符串
}

signature = HMAC-SHA256(secret, payload)
```

**安全特性：**
- 5分钟 Token 过期时间
- HMAC-SHA256 签名（无密钥无法伪造）
- 随机 Nonce（防止重放攻击）
- 无状态设计（无需服务器端存储）
- 兼容 Vercel Serverless

**工作流程：**

```
前端请求流程：
1. 调用 GET /api/get-token → 接收 Token（5分钟有效期）
2. 在内存中缓存 Token（过期前30秒刷新）
3. 调用 POST /api/generate 时携带 X-API-Token 头
4. 如果收到 401 错误 → 清除缓存，自动重试一次

后端验证流程：
1. 从请求头提取 X-API-Token
2. 验证 Token 格式（payload.signature）
3. 解码并解析 payload
4. 验证 HMAC 签名
5. 检查过期时间（< 5分钟）
6. 检查时钟偏移（不能来自未来）
7. 继续业务逻辑或返回 401
```

### 速率限制

| 端点 | 限制 | 时间窗口 |
|------|------|---------|
| `/api/generate` | 10 请求 | 1 分钟 |
| `/api/generate` | 100 请求 | 24 小时 |
| `/api/get-token` | 20 请求 | 1 分钟 |

### 环境变量配置

```bash
# 启用/禁用 Token 认证
ENABLE_TOKEN_AUTH="true"  # 设置为 "false" 禁用

# HMAC 签名密钥（启用时必需）
# 生成方式：openssl rand -base64 32
API_TOKEN_SECRET="<strong-random-key>"
```

---
