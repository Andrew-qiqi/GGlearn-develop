# 速率限制

本文档描述 SlideTutor 项目的速率限制策略。

最后更新：2026-03-28

---

## 速率限制配置

为了防止 API 滥用和保护服务器资源，系统对各个端点实施了速率限制。

### 端点限制

| 端点 | 限制 | 时间窗口 | 说明 |
|------|------|---------|------|
| `/api/generate` | 10 请求 | 1 分钟 | AI 内容生成端点 |
| `/api/generate` | 100 请求 | 24 小时 | 每日总限制 |
| `/api/get-token` | 20 请求 | 1 分钟 | Token 获取端点 |

### 限制依据

- 速率限制基于客户端 IP 地址
- 使用 `express-rate-limit` 中间件实现
- 在 Vercel 部署环境中，通过 `X-Forwarded-For` 头识别真实 IP

### 超出限制的响应

当客户端超出速率限制时，服务器返回：

**状态码**：`429 Too Many Requests`

**响应头**：
```
X-RateLimit-Limit: 10
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1711234567
Retry-After: 60
```

**响应体**：
```json
{
  "error": "Too many requests, please try again later."
}
```

---

## 实现细节

### 配置代码

```typescript
import rateLimit from 'express-rate-limit';

// /api/generate 端点限制
const generateLimiter = rateLimit({
  windowMs: 60 * 1000, // 1 分钟
  max: 10, // 最多 10 个请求
  message: 'Too many requests from this IP, please try again later.',
  standardHeaders: true,
  legacyHeaders: false,
});

// /api/get-token 端点限制
const tokenLimiter = rateLimit({
  windowMs: 60 * 1000, // 1 分钟
  max: 20, // 最多 20 个请求
  message: 'Too many token requests, please try again later.',
  standardHeaders: true,
  legacyHeaders: false,
});
```

### Trust Proxy 配置

在 Vercel 等反向代理环境中，需要配置 Express 信任代理：

```typescript
app.set('trust proxy', 1);
```

这确保速率限制基于真实客户端 IP，而不是代理服务器 IP。

---

## 监控和调整

### 监控指标

- 速率限制触发频率
- 被限制的 IP 地址
- 429 错误率

### 调整建议

如果发现以下情况，可以考虑调整限制：

1. **正常用户频繁触发限制**：增加限制阈值
2. **攻击者绕过限制**：降低限制阈值或添加更严格的验证
3. **服务器负载过高**：降低限制阈值

---

## 相关文档

- [安全架构](architecture.md) - 了解完整的安全防护策略
- [Token 认证](token-authentication.md) - 了解 Token 认证机制

---

*最后更新：2026-03-28*
