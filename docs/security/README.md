# 安全文档

本模块包含 SlideTutor 项目的安全架构、Token 认证、速率限制，以及 `Platform API` credits 完整性边界相关文档。

## 📚 文档列表

- **[architecture.md](architecture.md)** - 安全架构、多层防御策略和 credits 完整性边界
- **[token-authentication.md](token-authentication.md)** - API Token 认证系统详解
- **[rate-limiting.md](rate-limiting.md)** - 速率限制策略

## 🎯 快速导航

### 我想了解安全架构
从 [architecture.md](architecture.md) 开始，了解多层防御策略和安全层次。

### 我想了解 credits 是否会被篡改或重复加减
先看 [architecture.md](architecture.md)，再看 [../backend/api-design.md](../backend/api-design.md)，了解当前的原子提交、回放幂等和运维排查边界。

### 我想了解 Token 认证机制
查看 [token-authentication.md](token-authentication.md)，了解 HMAC 签名、Token 生成和验证流程。

### 我想了解速率限制
参考 [rate-limiting.md](rate-limiting.md)，了解各个端点的速率限制配置。

---

最后更新：2026-04-15
