# Codebase Concerns

**Analysis Date:** 2025-02-13

## Tech Debt

**God Components:**
- Issue: `PdfViewer.tsx` and `CanvasTutor.tsx` are massive (1200+ lines each) and contain many sub-components, complex state logic, and interaction handlers.
- Files: `SlideTutor-AI/src/components/PdfViewer.tsx`, `SlideTutor-AI/src/components/CanvasTutor.tsx`
- Impact: Difficult to maintain, test, and refactor. High risk of side effects when making changes.
- Fix approach: Extract sub-components (e.g., `SpatialNoteItem`, `TutorCard`, `PdfPage`) and custom hooks (e.g., `usePdfInteraction`, `useTutorInteraction`) into separate files.

**Hardcoded Massive Prompts:**
- Issue: AI prompts are hardcoded in a large switch statement within a single file, mixing logic with prompt engineering.
- Files: `SlideTutor-AI/src/lib/ai/prompts.ts`
- Impact: Hard to manage versioning of prompts, difficult to support multiple languages cleanly, and makes the file bloated.
- Fix approach: Move prompts to separate template files or a dedicated prompt management system/directory.

**Redundant/Unused Dependencies:**
- Issue: `better-sqlite3` is in `package.json` but `IndexedDB` is used for client-side storage.
- Files: `SlideTutor-AI/package.json`, `SlideTutor-AI/src/lib/db.ts`
- Impact: Increased bundle size/install time (though likely not bundled for frontend, still messy).
- Fix approach: Remove unused dependencies from `package.json`.

**Duplicate Security Logic:**
- Issue: Rate limiting and security checks are implemented in both `server.ts` (using standard middleware) and `api/generate.ts` (using custom manual logic).
- Files: `SlideTutor-AI/server.ts`, `SlideTutor-AI/api/generate.ts`
- Impact: Maintenance overhead, potential for conflicting behavior, and inconsistent security posture.
- Fix approach: Consolidate security logic into a shared middleware or rely on the `server.ts` implementation.

## Security Considerations

**Prompt Injection Vulnerability:**
- Issue: User messages are directly concatenated into LLM prompts without sufficient sanitization or using structured message formats (like OpenAI's chat completion API).
- Files: `SlideTutor-AI/src/lib/ai/prompts.ts`, `SlideTutor-AI/api/generate.ts`
- Impact: Users could bypass guardrails, extract system prompts, or manipulate the AI into unintended behavior.
- Current mitigation: Basic keyword filtering and a secondary LLM moderation call in `api/generate.ts`.
- Recommendations: Use structured message arrays instead of string concatenation. Implement more robust input validation.

**In-Memory State in Serverless:**
- Issue: `api/generate.ts` uses in-memory `Map` objects (`requestCounts`, `maliciousAlerts`) for rate limiting and alert suppression.
- Files: `SlideTutor-AI/api/generate.ts`
- Impact: In a serverless environment (Vercel), these maps are reset whenever a lambda instance is recycled, making rate limiting and throttling unreliable.
- Current mitigation: None (only works while instance is "warm").
- Recommendations: Use an external store like Redis (e.g., Upstash) for rate limiting and alert state.

**Fail-Open Moderation:**
- Issue: If the moderation AI call fails, the system "fails open" and allows the request.
- Files: `SlideTutor-AI/api/generate.ts` (line 197)
- Impact: Potential for malicious requests to bypass security during intermittent API failures.
- Recommendations: Consider failing closed or logging more aggressively for failed moderation attempts.

## Performance Bottlenecks

**Double LLM Calls:**
- Issue: Every request to `/api/generate` potentially triggers two LLM calls (one for moderation, one for the actual task).
- Files: `SlideTutor-AI/api/generate.ts`
- Impact: Increased latency for the user and higher API costs.
- Improvement path: Optimize moderation (e.g., cache results, use faster/cheaper models, or combine moderation into the system prompt).

**Heavy Component Re-renders:**
- Issue: Large components connected to a flat Zustand store (`useTutorStore`) may re-render frequently.
- Files: `SlideTutor-AI/src/components/CanvasTutor.tsx`, `SlideTutor-AI/src/store/tutorStore.ts`
- Cause: Many unrelated states are stored in the same store, and large components consume them.
- Improvement path: Split the store into smaller, specialized stores (e.g., `useAnalysisStore`, `useUiStore`). Use selective selectors in `useStore`.

## Fragile Areas

**PDF Layout Aggregation:**
- Issue: The `aggregateBlocks` function uses complex spatial heuristics (overlap, gaps, coordinates) to merge layout blocks.
- Files: `SlideTutor-AI/api/generate.ts`
- Why fragile: Highly dependent on precise coordinates from OCR/Layout engines (Azure); small changes in layout detection can break semantic grouping.
- Safe modification: Add comprehensive unit tests for various layout scenarios.

**Spatial Note Interaction:**
- Issue: Drag-and-drop and positioning of notes on PDF pages involves complex coordinate transformations between screen space and PDF space.
- Files: `SlideTutor-AI/src/components/PdfViewer.tsx`
- Why fragile: Sensitive to scaling, zooming, and container resizing.

## Test Coverage Gaps

**Critical Logic Lack of Tests:**
- What's not tested: The core `aggregateBlocks` logic, most React components, the entire `api/generate.ts` flow, and Zustand store transitions.
- Files: `SlideTutor-AI/api/generate.ts`, `SlideTutor-AI/src/components/`, `SlideTutor-AI/src/store/`
- Risk: Regressions in core layout processing or state management could go unnoticed.
- Priority: High

---

*Concerns audit: 2025-02-13*
