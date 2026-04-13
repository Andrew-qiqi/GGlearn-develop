## Original user task

Read-only review of SlideTutor's custom OpenAI-compatible capability probe design. Focus on whether the probe layering and error semantics are sound, especially:
- whether the plain-text probe should be removed
- whether mapping the final-step failure to `MODEL_CAPABILITY_CHECK_VISION_UNSUPPORTED` is semantically accurate
- what the more reasonable probe layers and decision boundaries should be

## Final prompt sent to Gemini CLI

Read-only review of a TypeScript codebase's custom OpenAI-compatible capability probe design. Focus only on these behaviors as described: there is a 4-step probe for custom endpoints: (1) plain text non-stream chat completion, (2) plain text streaming chat completion, (3) plain text streaming + json_schema response_format, (4) image_url + text + streaming + json_schema. Step-4 failure maps to MODEL_CAPABILITY_CHECK_VISION_UNSUPPORTED. Built-in registry already marks a known model doubao-seed-1-8-251228 as supporting image_input, structured_output, and streaming, but the custom probe can still fail on step 4. Please assess three questions concisely: 1) should the plain-text probe be removed; 2) is mapping step-4 failure to VISION_UNSUPPORTED semantically inaccurate; 3) what are the more reasonable probe layers and decision boundaries for a product whose hard requirements are image input + streaming + native structured output, while text generation is assumed universal. Return a compact structured review with findings first, then recommendation.

## Gemini output (raw)

针对自定义 OpenAI 兼容端点的 4 步探测设计，评估如下：

### **核心发现 (Findings)**

1.  **纯文本探测的诊断价值**：步骤 1 和 2 虽然对于最终目标（多模态+结构化）是冗余的，但在**故障排查**中具有重要意义。它们能区分是“身份验证/网络失败”还是“高级功能特性不支持”。
2.  **步骤 4 的语义模糊性**：步骤 4 测试的是功能交集（Vision ∩ Streaming ∩ JSON Schema）。若失败，可能是由于模型不支持 Vision，也可能是模型**不支持在 Vision 模式下使用结构化输出**。将其映射为 `VISION_UNSUPPORTED` 会掩盖“组合功能受限”的具体原因。
3.  **注册表与探测的冲突**：对于 `doubao-seed-1-8-251228` 等已知模型，探测失败通常意味着 API 版本过旧、账户权限不足或特定区域的 Provider 尚未对齐官方特性，而非模型本身的能力缺陷。

### **评估建议 (Recommendation)**

**1. 关于纯文本探测：**
*   **不建议完全移除**，但建议**合并**。保留步骤 2（流式文本）作为基础协议握手（Baseline），移除步骤 1。步骤 2 失败应直接判定为 `BASE_PROTOCOL_UNSUPPORTED` 或配置错误。

**2. 关于语义映射：**
*   步骤 4 失败映射为 `VISION_UNSUPPORTED` **不够准确**。
*   更合理的映射是 `VISION_COMPATIBILITY_UNSUPPORTED` 或 `COMPOSITE_CAPABILITY_FAILED`，因为它可能仅仅是无法同时满足 Image+JSON。

**3. 优化后的探测层级与决策边界：**
*   **Layer 1: Text + Streaming** (基础连接性)
    *   *失败判定*：`PROVIDER_UNAVAILABLE` / `STREAMING_UNSUPPORTED`
*   **Layer 2: Text + Streaming + Native JSON Schema** (硬性结构化要求)
    *   *失败判定*：`STRUCTURED_OUTPUT_UNSUPPORTED`
*   **Layer 3: Image + Text + Streaming + Native JSON Schema** (全功能验证)
    *   *成功*：授予 `FULL_CAPABILITY`
    *   *失败*：判定为 `VISION_STRUCTURED_COMBINATION_UNSUPPORTED` (即：视觉能力与结构化输出无法并存)

**结论：** 既然产品硬性要求是“视觉+流式+结构化”，探测应从**协议基础**向**原子特性**再向**组合特性**递进。当前的步骤 4 过于沉重，应在确信原子特性（JSON）通过后再单独测试视觉组合，以实现精准降级逻辑。

## Concise summary

Gemini's second opinion aligns with the repository evidence on the two main points:
- the final step is testing a composite runtime contract, not pure vision support
- the plain-text non-stream step is the weakest part of the current ladder and is a candidate for removal, while a lighter protocol baseline is still useful

## Action items / next steps

- Treat step-4 failures as composite contract failures unless image-only failure is separately proven.
- Consider removing only the non-stream text probe, not the entire baseline handshake.
- Redefine probe layers around baseline streaming, structured-output, and multimodal-composite boundaries.
