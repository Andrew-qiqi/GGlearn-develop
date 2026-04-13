# Gemini Consultation Artifact

## Original user task
Review whether the current custom OpenAI-compatible capability probe is reasonable, especially for doubao-seed-1-8-251228 failing stage 4, and advise on probe design and diagnostics.

## Final prompt sent to Gemini CLI
```text
You are reviewing a model capability probe design for SlideTutor, an app that uses OpenAI-compatible endpoints.

Context:
- There is a built-in registry where model `doubao-seed-1-8-251228` is treated as satisfying the product hard constraints.
- When the same model is tested through a custom OpenAI-compatible BYOK path, the live capability probe fails.
- The current custom probe stages are:
  1. text completion
  2. streaming text completion
  3. streaming structured JSON output
  4. streaming structured JSON output with image input
- The current failure discussed is stage 4, surfaced as `MODEL_CAPABILITY_CHECK_VISION_UNSUPPORTED`.
- The user strongly believes this demonstrates a flaw in the probe mechanism rather than a real lack of model capability.
- The user also thinks pure text generation probe is unnecessary and should be removed to save resources.
- We are not implementing yet. We are planning carefully.

Questions:
1. Is it reasonable to remove the pure text-generation probe? Why or why not?
2. Is the current stage-4 failure semantically equivalent to “vision unsupported”? If not, how should it be described?
3. What is a more robust capability/probe strategy for unknown OpenAI-compatible endpoints?
4. Are there mature or official probe patterns for this kind of problem, or is registry + staged smoke test the normal industry practice?
5. What logs/diagnostic data would be most useful from production/Cloudflare logs to distinguish false negatives from real incompatibility?

Please answer in a structured way with:
- Verdict
- Main problems with current mechanism
- Recommended staged design
- Suggested logging fields
- Risks / trade-offs
```

## Gemini output (raw)
```text
针对 SlideTutor 的模型能力探测（Capability Probe）设计，以下是基于当前问题的深度分析与优化建议：

### 1. 结论 (Verdict)

**当前的探测机制过于冗余且存在逻辑耦合，容易产生“误报”（False Negative）。**

移除纯文本探测是合理的，但应将其转化为“基础连通性校验”的一部分。Stage 4 的失败极大概率不是因为模型不支持 Vision，而是因为 **“Streaming + Structured JSON + Vision”** 三者组合时触发了特定供应商的参数校验限制或 Prompt 格式要求。

---

### 2. 当前机制的主要问题 (Main Problems)

1.  **串行浪费 (Sequential Latency):** 4 个阶段依次进行，对于 BYOK 用户来说，完成探测可能需要 5-10 秒，极大地影响了首次配置体验。
2.  **语义模糊 (Semantic Ambiguity):** Stage 4 是一个“全能测试”。如果失败，系统无法区分是 **不支持 Vision**、**不支持 JSON Mode**、还是 **不支持在 Vision 模式下使用 JSON Mode**。
3.  **对兼容性实现的假设过强:** 许多 OpenAI-compatible 接口（如 Doubao, DeepSeek, Anthropic-via-Proxy）在 `response_format` 或 `tool_calls` 的实现上与原生 OpenAI 有细微差别，尤其是多模态输入时。
4.  **缺乏自适应能力:** 探测失败直接阻断使用（Hard Block），而不是降级使用（Graceful Degradation）。

---

### 3. 推荐的阶梯式设计 (Recommended Staged Design)

建议将探测逻辑从“线性测试”改为 **“目标导向探测 + 失败回溯诊断”**：

#### 第一阶段：目标能力测试 (Goal-Oriented Probe)
直接测试业务所需的 **最高能力组合**：`Streaming + JSON Schema + Vision`。
*   **Payload:** 一个微小的透明像素 Base64 + 简单的 JSON Schema（如 `{"type": "object", "properties": {"status": {"type": "string"}}}`）。
*   **目的:** 如果通过，直接上线，耗时仅需 1 次 RTT。

#### 第二阶段：诊断回溯 (Diagnostic Backtrack) - 仅在第一阶段失败时触发
如果第一阶段报错，通过二分法或并行测试定位痛点：
1.  **Probe A (Vision Only):** 测试图片解析，不带 JSON Schema。
2.  **Probe B (JSON Only):** 测试文本转 JSON，不带图片。
3.  **Probe C (Basic):** 极简文本流式输出。

#### 结果映射表：
*   A 成功 + B 失败 $\rightarrow$ `JSON_MODE_UNSUPPORTED`（业务可降级为纯文本 Prompt 提取）。
*   A 失败 + B 成功 $\rightarrow$ `VISION_UNSUPPORTED`（这才是真正的 Vision 不支持）。
*   A 成功 + B 成功 + Stage 1 失败 $\rightarrow$ `COMPOSITE_CAPABILITY_INCOMPATIBLE`（模型支持 Vision 和 JSON，但不能同时开启）。

---

### 4. 建议的日志/诊断字段 (Suggested Logging Fields)

为了在 Cloudflare Logs 中快速区分“假阴性”，应记录以下字段：

*   **`provider_error_code`:** 捕获 upstream 返回的具体错误码（如 Doubao 的 `400 InvalidParameter`）。
*   **`raw_response_body`:** 截断记录前 200 字节的报错信息。
*   **`request_payload_type`:** 记录使用的是 `response_format: {type: "json_schema"}` 还是 `json_object`。
*   **`processing_latency`:** 区分是网络超时还是模型处理报错。
*   **`system_fingerprint`:** 如果接口返回了该字段，用于追踪特定版本的模型行为变化。

---

### 5. 风险与权衡 (Risks / Trade-offs)

| 选项 | 风险 | 权衡 |
| :--- | :--- | :--- |
| **移除纯文本探测** | 如果 API Key 无效，用户可能在 Stage 4 看到复杂的 Vision 错误而非简单的 Auth 错误。 | **对策:** 在诊断阶段首选校验 API Key 的有效性。 |
| **合并探测阶段** | 探测逻辑变得复杂，代码维护成本增加。 | **对策:** 采用策略模式（Strategy Pattern）封装不同供应商的探测 Payload。 |
| **BYOK 兼容性** | 不同的 OpenAI 代理（如 OneAPI, NewAPI）会重写错误码。 | **对策:** 优先依赖 `HTTP Status Code` 和回溯测试的结果，而非依赖特定的报错字符串。 |

---

### 6. 行业惯例简述 (Industry Patterns)

*   **Registry (预设库):** 针对已知模型（如 `gpt-4o`, `doubao-xxx`）使用预定义的 Capability Map。这是 **最快** 的方案。
*   **Smoke Test (烟雾测试):** 针对未知 Endpoint，通常只进行一次最简化的能力探测（如 Vercel AI SDK 的做法）。
*   **Error-driven Discovery:** 并不在配置时探测，而是在 **第一次真实业务调用** 失败时，根据报错动态标记模型能力并提示用户降级。

**建议 SlideTutor 采用的方案：**
1.  **内置 Registry** 覆盖 90% 的主流模型。
2.  **BYOK 路径** 采用“一跳式”目标探测（Stage 1），失败后再自动触发“诊断回溯”。这兼顾了性能与准确性。
```

## Concise summary
Gemini was asked to evaluate the current staged probe, whether pure text probe should be removed, how to describe stage-4 failure, what mature patterns exist, and what logs would help distinguish false negatives.

## Action items / next steps
- Compare Gemini recommendations against local code review and subagent feedback.
- Extract concrete logging fields and ask the user for Cloudflare logs if needed.
