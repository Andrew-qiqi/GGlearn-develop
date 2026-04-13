# Math Delimiter Normalization Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make tutor-facing math render reliably even when models emit `\(...\)` or `\[...\]` instead of dollar-delimited LaTeX.

**Architecture:** Add one shared math-markdown normalization boundary before `ReactMarkdown` rendering, then tighten generation prompts so new model output prefers `$...$` and `$$...$$`. Cover both the runtime rendering path and the upstream prompt contract with regression tests.

**Tech Stack:** React 19, TypeScript, ReactMarkdown, remark-math, rehype-katex, Vitest, Testing Library

---

### Task 1: Lock the broken behavior with tests

**Files:**
- Modify: `SlideTutor-AI/src/components/CanvasTutor.test.tsx`
- Modify: `SlideTutor-AI/src/lib/ai/prompts.test.ts`

- [ ] **Step 1: Write the failing renderer regression test**

Add a `CanvasTutor` test that renders a knowledge card whose body contains `\(x_i\)` and `\[\frac{a}{b}\]`, then assert the rendered output contains KaTeX markup instead of literal delimiter text.

- [ ] **Step 2: Run the renderer test to verify it fails**

Run: `npm test -- src/components/CanvasTutor.test.tsx`
Expected: FAIL because `remark-math` does not parse `\(...\)` / `\[...\]` yet.

- [ ] **Step 3: Write the failing prompt regression**

Add prompt assertions that `explain`, `followup`, `regenerate_followup`, and `regenerate_chunk` explicitly require `$...$` / `$$...$$` delimiters and forbid `\(...\)` / `\[...\]`.

- [ ] **Step 4: Run the prompt test to verify it fails**

Run: `npm test -- src/lib/ai/prompts.test.ts`
Expected: FAIL because the current prompt text only says "use LaTeX for math formulas".

### Task 2: Implement the shared renderer fix

**Files:**
- Create: `SlideTutor-AI/src/components/ui/MarkdownMath.tsx`
- Create: `SlideTutor-AI/src/lib/markdown/normalizeMathDelimiters.ts`
- Modify: `SlideTutor-AI/src/components/CanvasTutor.tsx`
- Modify: `SlideTutor-AI/src/components/AskYouTutor.tsx`
- Modify: `SlideTutor-AI/src/components/NoteItem.tsx`

- [ ] **Step 1: Add a pure delimiter normalizer**

Implement a small utility that converts `\(...\)` to `$...$` and `\[...\]` to display-math `$$...$$` form without changing already valid dollar-delimited math.

- [ ] **Step 2: Add one shared markdown renderer wrapper**

Create a focused component that applies delimiter normalization, `remark-math`, and `rehype-katex` in one place.

- [ ] **Step 3: Replace duplicated math-enabled ReactMarkdown usage**

Swap the tutor card, ghost question, quick explain, ask-you question/feedback, and note renderers onto the shared component so the fix is consistent across tutor-generated surfaces.

- [ ] **Step 4: Run the renderer regression**

Run: `npm test -- src/components/CanvasTutor.test.tsx`
Expected: PASS with KaTeX nodes present.

### Task 3: Tighten upstream prompt guidance

**Files:**
- Modify: `SlideTutor-AI/src/lib/ai/prompts.ts`
- Test: `SlideTutor-AI/src/lib/ai/prompts.test.ts`

- [ ] **Step 1: Add explicit delimiter rules**

Update the relevant prompt branches so math instructions say:
- inline math uses `$...$`
- display math uses `$$...$$`
- do not use `\(...\)` or `\[...\]`

- [ ] **Step 2: Run the prompt regression**

Run: `npm test -- src/lib/ai/prompts.test.ts`
Expected: PASS.

### Task 4: Verify and document

**Files:**
- Modify: `docs/changelog/CHANGELOG_TECH.md`

- [ ] **Step 1: Run focused verification**

Run: `npm test -- src/components/CanvasTutor.test.tsx src/lib/ai/prompts.test.ts`
Expected: PASS.

- [ ] **Step 2: Run a broader safety pass**

Run: `npm test -- src/components/AskYouTutor.test.tsx src/hooks/useSlideAnalysis.test.ts`
Expected: PASS or explicit note if no matching test file exists.

- [ ] **Step 3: Record the technical change**

Add a short changelog note describing the delimiter normalization boundary and the stricter prompt contract.
