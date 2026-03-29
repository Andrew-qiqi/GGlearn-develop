# 部署架构

本文档描述 SlideTutor 项目的部署架构、性能优化和监控告警策略。

最后更新：2026-03-28

---

## 部署架构

### Vercel Serverless 部署

**架构特点：**
- 无状态函数（每个请求独立）
- 自动扩展
- 全球 CDN 分发
- 零配置 HTTPS

**部署配置：**

```json
// vercel.json
{
  "functions": {
    "api/**/*.ts": {
      "maxDuration": 60  // 60秒超时（支持流式响应）
    }
  },
  "rewrites": [
    {
      "source": "/api/:path*",
      "destination": "/api/:path*"
    }
  ]
}
```

**环境变量：**
```bash
# AI 服务
GEMINI_API_KEY=<key>
OPENAI_API_KEY=<key>
DOUBAO_API_KEY=<key>
QWEN_API_KEY=<key>

# Azure 服务
AZURE_DOCUMENT_INTELLIGENCE_ENDPOINT=<endpoint>
AZURE_DOCUMENT_INTELLIGENCE_KEY=<key>

# 安全配置
ENABLE_TOKEN_AUTH=true
API_TOKEN_SECRET=<secret>
ALLOWED_ORIGINS=https://www.slidetutor-ai.com

# 邮件服务
SMTP_HOST=<host>
SMTP_PORT=<port>
SMTP_USER=<user>
SMTP_PASS=<pass>
```

### 性能优化

**前端优化：**
- Vite 构建优化（代码分割、Tree Shaking）
- React.lazy() 组件懒加载
- IndexedDB 缓存减少 API 调用
- Token 缓存（5分钟重用）

**后端优化：**
- 流式响应（降低首字节时间）
- 速率限制（防止滥用）
- 无状态设计（水平扩展）

**性能指标：**
- 首次 API 调用：+100ms（Token 获取）
- 后续调用：+<1ms（Token 验证）
- Token 缓存命中率：~95%
- 用户体验：无明显影响

### 监控和告警

**关键指标：**
- Token 生成成功率（应为 ~100%）
- Token 验证成功率（应 >95%）
- 401 错误率（应 <5%，有自动重试）
- API 响应时间（不应增加 >5%）
- 速率限制触发频率

**告警阈值：**
- **高优先级**：Token 验证失败率 >10%（5分钟窗口）
- **高优先级**：INVALID_SIGNATURE 错误 >10/分钟（可能攻击）
- **中优先级**：API 响应时间增加 >50%
- **低优先级**：EXPIRED_TOKEN 错误 >100/小时（UX 问题）

---

