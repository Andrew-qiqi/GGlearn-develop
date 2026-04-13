# Original user task
Challenge current custom OpenAI-compatible capability check design with a contrarian review.

# Final prompt sent to Gemini CLI
Act as a contrarian design reviewer. Context: a product has a custom OpenAI-compatible capability check. For a known model doubao-seed-1-8-251228, the built-in registry says it satisfies hard constraints, but the custom probe fails with VISION_UNSUPPORTED. The current probe does 4 sequential steps for custom OpenAI-compatible models: plain text chat, then stream=true, then stream=true + response_format=json_schema, then stream=true + response_format=json_schema + image_url data URL. On failure it maps by phase into codes like STREAMING_UNSUPPORTED, STRUCTURED_OUTPUT_UNSUPPORTED, VISION_UNSUPPORTED. UI text tells users the endpoint does not support image input required by the product. Give 5-8 sharp critique points focused on design flaws, bad assumptions, misleading error semantics, and product/debugging risk. Then give a recommended boundary definition for what this mechanism is and is not allowed to claim. Be concise and strong.

# Gemini output (raw)
System.Object[]

# Concise summary
- Current mechanism is a runtime contract smoke test, not a trustworthy capability oracle.
- VISION_UNSUPPORTED is semantically overclaimed because phase 4 bundles image + stream + schema.
- Dual truth sources (registry vs probe) create contradictory product states and debugging noise.

# Action items / next steps
- Use Gemini findings only as cross-check, not source of truth.
- Anchor final review in repository code paths and observed semantics.
