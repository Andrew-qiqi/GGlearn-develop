# GGlearn Image Channel Integration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Integrate a minimal image-generation channel into the textbook diagram flow so selected `diagram` chunks can render as generated images while existing SVG/Mermaid paths continue to work.

**Architecture:** Keep the current `diagram` chunk type. Add image-specific metadata to chunk metadata, resolve the `image` role config at the app/orchestrator layer, and post-process generated diagram chunks that carry `imagePrompt` into image-backed renderables. Rendering remains a single `ChunkRenderer` branch that chooses between SVG/Mermaid and image output.

**Tech Stack:** TypeScript, React, existing AI client helpers, Node test runner

---

### Task 1: Extend chunk metadata for image-backed diagrams

**Files:**
- Modify: `GGlearn/src/types.ts`
- Test: `GGlearn/tests/textbookGeneration.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
assert.equal(chunks[0].metadata?.renderRoute, 'image');
assert.equal(chunks[0].metadata?.imageRender?.status, 'ready');
assert.ok(chunks[0].metadata?.imageRender?.dataUrl?.startsWith('data:image/png;base64,'));
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test tests/textbookGeneration.test.ts`
Expected: FAIL because `renderRoute` / `imageRender` do not exist yet.

- [ ] **Step 3: Write minimal implementation**

```ts
export interface ChunkImageRender {
  provider: 'gemini' | 'openai-compatible';
  model: string;
  mimeType: string;
  dataUrl: string;
  status: 'ready' | 'failed';
}
```

Add metadata fields:

```ts
renderRoute?: 'svg' | 'image';
imagePrompt?: string;
imageAlt?: string;
imageRender?: ChunkImageRender;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npm test tests/textbookGeneration.test.ts`
Expected: PASS for type-level expectations after implementation steps below land.

### Task 2: Add image generation helper

**Files:**
- Create: `GGlearn/src/lib/ai/imageGeneration.ts`
- Test: `GGlearn/tests/textbookGeneration.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
const render = await generateImageFromPrompt(
  {
    provider: 'openai',
    apiKey: 'image-key',
    baseUrl: 'https://example.com/v1',
    model: 'gpt-image-1',
  },
  'Generate an intuitive scene of invariant scaling.'
);

assert.equal(render.status, 'ready');
assert.equal(render.mimeType, 'image/png');
assert.ok(render.dataUrl.startsWith('data:image/png;base64,'));
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test tests/textbookGeneration.test.ts`
Expected: FAIL because the helper does not exist.

- [ ] **Step 3: Write minimal implementation**

Create a helper that supports:
- `openai-compatible` image generation through `POST {baseUrl}/images/generations`
- `gemini` image generation through the Gemini image response path

Return a normalized object:

```ts
{
  provider,
  model,
  mimeType,
  dataUrl,
  status: 'ready'
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npm test tests/textbookGeneration.test.ts`
Expected: PASS

### Task 3: Route a narrow set of diagram chunks to image generation

**Files:**
- Modify: `GGlearn/src/lib/ai/textbookGeneration.ts`
- Test: `GGlearn/tests/textbookGeneration.test.ts`

- [ ] **Step 1: Write the failing tests**

```ts
test('generateTextbookChunksWithProvider renders image-backed diagram chunks when metadata.imagePrompt is returned', async () => {
  // provider returns a diagram chunk with metadata.imagePrompt
  // image generator returns a base64 png
  // final chunk metadata.renderRoute === 'image'
});

test('generateTextbookChunksWithProvider creates image-prompt fallback for visual-aid sections', async () => {
  // visual-aid fallback should no longer require SVG-only fallback
  // image route should be selected for the fallback chunk
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test tests/textbookGeneration.test.ts`
Expected: FAIL because image prompts are ignored and visual-aid fallback still emits only `diagramCode`.

- [ ] **Step 3: Write minimal implementation**

Implement:
- `renderRoute = 'svg'` when `diagramCode` exists
- `renderRoute = 'image'` when `imagePrompt` exists
- post-process generated chunks and call image generation only for `renderRoute === 'image'`
- narrow fallback scope: `visual-aid` fallback chunks generate an `imagePrompt` instead of forced SVG

Keep existing SVG/Mermaid behavior unchanged for all other diagram chunks.

- [ ] **Step 4: Run test to verify it passes**

Run: `npm test tests/textbookGeneration.test.ts`
Expected: PASS

### Task 4: Pass image role config into generation paths

**Files:**
- Modify: `GGlearn/src/App.tsx`
- Modify: `GGlearn/src/lib/generation/orchestrator.ts`
- Test: `GGlearn/tests/generationOrchestrator.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
assert.equal(capturedImageConfig?.model, 'gpt-image-1');
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test tests/generationOrchestrator.test.ts`
Expected: FAIL because the generation path does not pass image config.

- [ ] **Step 3: Write minimal implementation**

Resolve:
- `writer` config for text generation
- `image` config for image-backed diagram rendering

Pass the image config into:
- direct chapter generation in `App.tsx`
- orchestrated generation in `generateProjectChaptersWithRun`

- [ ] **Step 4: Run test to verify it passes**

Run: `npm test tests/generationOrchestrator.test.ts`
Expected: PASS

### Task 5: Render image-backed diagram chunks

**Files:**
- Modify: `GGlearn/src/components/ChunkRenderer.tsx`

- [ ] **Step 1: Write the failing UI assumption**

```ts
// type-level or render-path assertion:
// when metadata.renderRoute === 'image' and imageRender.dataUrl exists,
// the component should render an <img>.
```

- [ ] **Step 2: Run verification to confirm the old renderer is insufficient**

Run: `npm run lint`
Expected: FAIL or missing branch until implementation is added.

- [ ] **Step 3: Write minimal implementation**

Add a new branch under `chunk.type === 'diagram'`:
- if `renderRoute === 'image'` and `imageRender.dataUrl` exists, render `<img>`
- else keep current SVG/Mermaid logic

- [ ] **Step 4: Run verification**

Run: `npm run lint`
Expected: PASS

### Task 6: Full verification

**Files:**
- No new files

- [ ] **Step 1: Run focused tests**

Run: `npm test tests/textbookGeneration.test.ts tests/generationOrchestrator.test.ts`
Expected: PASS

- [ ] **Step 2: Run typecheck**

Run: `npm run lint`
Expected: PASS

- [ ] **Step 3: Run full test suite**

Run: `npm test`
Expected: PASS

- [ ] **Step 4: Run build**

Run: `npm run build`
Expected: PASS
