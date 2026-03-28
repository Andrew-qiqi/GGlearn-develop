# Phase 01 Research: Environment & Core Stability

## 1. Security Analysis (SEC-01)
*   **CRITICAL VULNERABILITY**: `SlideTutor-AI/vite.config.ts` currently injects `GEMINI_API_KEY` into the client-side bundle via the `define` property. Although not explicitly called in `src/`, this exposes the key in the production JS bundle. **Planning must include the immediate removal of this injection.**
*   **Backend Hardening**: `api/generate.ts` implements a custom `securityGuard` for referer/origin checks and basic in-memory rate limiting. However, `server.ts` also uses `express-rate-limit`. These should be consolidated to ensure consistent protection across Vercel deployments and local dev.
*   **Environment Validation**: There is currently no "fail-fast" mechanism for missing environment variables. A validation script or module is needed to check for `GEMINI_API_KEY`, `AZURE_DOCUMENT_INTELLIGENCE_KEY`, and SMTP credentials at startup.

## 2. Regression Baseline (STAB-02)
*   **Logical Anchors**: The core stability of the app relies on `src/lib/pdf/layoutUtils.ts` (which creates the "Cognitive Map" for the AI) and `src/lib/ai/prompts.ts` (which handles the complex instruction set).
*   **Current Testing State**: `vitest` is configured, and tests exist for prompt construction. However, there are **no unit tests** for the PDF layout aggregation logic, which is the most fragile part of the "Visual Awareness" feature.
*   **Baseline Requirements**: To plan Phase 1 well, we must include the creation of:
    *   Unit tests for `layoutUtils.ts` (merging/clustering logic).
    *   Snapshot tests for `prompts.ts` to prevent "instruction drift" during interaction smoothing.

## 3. Core Logic Preservation (STAB-01)
*   **Duplication Alert**: The `aggregateBlocks` logic is currently duplicated in both `api/generate.ts` and `src/lib/pdf/layoutUtils.ts`. This increases the risk of functional drift.
*   **Maintenance Strategy**: The "concise and clean" requirement is currently met, but the `api/generate.ts` file is growing large (600+ lines). Planning should consider if security hardening logic (like moderation and alerts) should be moved to separate utilities to keep the core generation flow readable.

## Summary for Planning
To plan this phase effectively, the "Environment" part must prioritize **key isolation** (Vite config fix) and **validation**, while the "Stability" part must focus on **utility unit testing** to lock down the PDF-to-AI pipeline before Phase 2's data migration.
