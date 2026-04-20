# GGlearn Settings Endpoint Routing Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the single `aiConfig` settings shape with endpoint-plus-routing settings for planner/writer/image roles, preserve old settings through migration, and update the settings UI without expanding scope into textbook generation behavior.

**Architecture:** The settings layer becomes `aiEndpoints + aiRouting`. Internal tasks still map to planner/writer/image roles, with `planning + diagram` sharing the planner role. Persistence performs a lossless migration from legacy `aiConfig`, and the settings screen exposes role binding plus endpoint management.

**Tech Stack:** TypeScript, React, IndexedDB persistence, Node test runner

---

### Task 1: Define settings model boundaries

**Files:**
- Modify: `GGlearn/src/types.ts`
- Test: `GGlearn/tests/generationOrchestrator.test.ts`

- [ ] **Step 1: Write the failing type/test updates**

```ts
const mockSettings: AppSettings = {
  language: 'en',
  aiEndpoints: [
    {
      id: 'planner',
      label: 'Planner',
      provider: 'gemini',
      kind: 'text',
      apiKey: 'test-key',
      model: 'gemini-3-flash-preview',
      baseUrl: 'https://generativelanguage.googleapis.com',
    },
  ],
  aiRouting: {
    planner: 'planner',
    writer: 'planner',
    image: 'planner',
  },
  searchConfig: { provider: 'auto', apiKey: '' },
  captureConfig: { provider: 'auto', firecrawlApiKey: '', endpoint: '' },
};
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test tests/generationOrchestrator.test.ts`
Expected: FAIL with `AppSettings` / `aiConfig` shape mismatch.

- [ ] **Step 3: Write minimal implementation**

```ts
export type ModelProvider = 'gemini' | 'openai-compatible';
export type ModelKind = 'text' | 'image' | 'multimodal';

export interface ModelEndpoint {
  id: string;
  label: string;
  provider: ModelProvider;
  kind: ModelKind;
  apiKey: string;
  model: string;
  baseUrl: string;
}

export interface ModelRouting {
  planner: string;
  writer: string;
  image: string;
}

export interface AppSettings {
  language: 'zh' | 'en';
  aiEndpoints: ModelEndpoint[];
  aiRouting: ModelRouting;
  searchConfig: SearchConfig;
  captureConfig: CaptureConfig;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npm test tests/generationOrchestrator.test.ts`
Expected: PASS

### Task 2: Migrate and load settings safely

**Files:**
- Modify: `GGlearn/src/lib/persistence.ts`
- Test: `GGlearn/tests/persistence.test.ts`

- [ ] **Step 1: Write the failing persistence tests**

```ts
test('loadSettings migrates legacy aiConfig into endpoint routing', async () => {
  window.localStorage.setItem('gglearn_settings', JSON.stringify({
    language: 'zh',
    aiConfig: {
      provider: 'gemini',
      apiKey: 'legacy-key',
      baseUrl: 'https://proxy.example.com',
      model: 'gemini-3-flash-preview',
    },
  }));

  const settings = await loadSettings();

  assert.equal(settings.aiEndpoints.length >= 2, true);
  assert.equal(settings.aiRouting.planner.length > 0, true);
  assert.equal(settings.aiRouting.writer.length > 0, true);
  assert.equal(settings.aiRouting.image.length > 0, true);
});

test('loadSettings preserves new endpoint routing shape', async () => {
  await saveSettings({
    ...DEFAULT_SETTINGS,
    aiEndpoints: [
      {
        id: 'writer-endpoint',
        label: 'Writer',
        provider: 'openai-compatible',
        kind: 'text',
        apiKey: 'writer-key',
        model: 'gpt-4.1-mini',
        baseUrl: 'https://example.com/v1',
      },
    ],
    aiRouting: {
      planner: 'writer-endpoint',
      writer: 'writer-endpoint',
      image: 'writer-endpoint',
    },
  });

  const settings = await loadSettings();
  assert.equal(settings.aiEndpoints[0].id, 'writer-endpoint');
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test tests/persistence.test.ts`
Expected: FAIL because legacy migration and new settings shape are not implemented.

- [ ] **Step 3: Write minimal implementation**

```ts
function createDefaultEndpoint(id: string, label: string, provider: ModelProvider, kind: ModelKind): ModelEndpoint {
  return {
    id,
    label,
    provider,
    kind,
    apiKey: '',
    model: provider === 'gemini' ? 'gemini-3-flash-preview' : '',
    baseUrl: provider === 'gemini'
      ? 'https://generativelanguage.googleapis.com'
      : 'https://api.openai.com/v1',
  };
}

function migrateLegacyAiConfig(aiConfig?: AIConfig): Pick<AppSettings, 'aiEndpoints' | 'aiRouting'> {
  const textEndpoint = {
    ...createDefaultEndpoint('legacy-text', 'Legacy Text Model', aiConfig?.provider === 'gemini' ? 'gemini' : 'openai-compatible', 'text'),
    apiKey: aiConfig?.apiKey ?? '',
    model: aiConfig?.model || 'gemini-3-flash-preview',
    baseUrl: aiConfig?.baseUrl || 'https://generativelanguage.googleapis.com',
  };

  const imageEndpoint = {
    ...textEndpoint,
    id: 'legacy-image',
    label: 'Legacy Image Model',
    kind: 'image',
  };

  return {
    aiEndpoints: [textEndpoint, imageEndpoint],
    aiRouting: {
      planner: textEndpoint.id,
      writer: textEndpoint.id,
      image: imageEndpoint.id,
    },
  };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npm test tests/persistence.test.ts`
Expected: PASS

### Task 3: Route app code through planner/writer roles

**Files:**
- Modify: `GGlearn/src/App.tsx`
- Modify: `GGlearn/src/lib/generation/orchestrator.ts`
- Modify: `GGlearn/src/lib/ai/research.ts`
- Test: `GGlearn/tests/generationOrchestrator.test.ts`
- Test: `GGlearn/tests/research.test.ts`

- [ ] **Step 1: Write the failing tests**

```ts
test('orchestrator uses planner endpoint config', async () => {
  const plannerEndpoint = {
    id: 'planner',
    label: 'Planner',
    provider: 'gemini',
    kind: 'text',
    apiKey: 'planner-key',
    model: 'gemini-3-flash-preview',
    baseUrl: 'https://generativelanguage.googleapis.com',
  };

  const settings = {
    ...mockSettings,
    aiEndpoints: [plannerEndpoint],
    aiRouting: { planner: 'planner', writer: 'planner', image: 'planner' },
  };

  // Assert orchestrator resolves planner config before generation.
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test tests/generationOrchestrator.test.ts tests/research.test.ts`
Expected: FAIL because app code still reads `settings.aiConfig`.

- [ ] **Step 3: Write minimal implementation**

```ts
const plannerConfig = resolveRoleConfig(settings, 'planner');
const writerConfig = resolveRoleConfig(settings, 'writer');
```

Use planner config for:
- outline generation
- orchestration planning
- diagram/spec generation
- Gemini web search fallback

Use writer config for:
- chunk/body writing
- single-chunk regeneration

- [ ] **Step 4: Run test to verify it passes**

Run: `npm test tests/generationOrchestrator.test.ts tests/research.test.ts`
Expected: PASS

### Task 4: Update translations and settings screen

**Files:**
- Modify: `GGlearn/src/lib/translations.ts`
- Modify: `GGlearn/src/views/SettingsView.tsx`

- [ ] **Step 1: Write the failing UI assumptions**

```ts
// If no UI test exists, encode the target props/state contract in a small helper:
const plannerEndpointId = settings.aiRouting.planner;
const endpointOptions = settings.aiEndpoints.map((endpoint) => endpoint.label);
assert.ok(endpointOptions.includes('默认文本模型'));
```

- [ ] **Step 2: Run test or typecheck to verify it fails**

Run: `npm run lint`
Expected: FAIL because `SettingsView` still expects `settings.aiConfig`.

- [ ] **Step 3: Write minimal implementation**

```tsx
<section>
  <label>{t.plannerModel}</label>
  <select value={settings.aiRouting.planner}>...</select>
</section>
<section>
  <label>{t.modelEndpoints}</label>
  {settings.aiEndpoints.map((endpoint) => (
    <div key={endpoint.id}>
      <input value={endpoint.label} />
      <select value={endpoint.provider}>...</select>
      <input value={endpoint.model} />
      <input value={endpoint.baseUrl} />
      <input value={endpoint.apiKey} />
    </div>
  ))}
</section>
```

Add translation keys for planner/writer/image roles and endpoint labels.

- [ ] **Step 4: Run verification**

Run: `npm run lint`
Expected: PASS

### Task 5: Full verification

**Files:**
- No new files

- [ ] **Step 1: Run focused tests**

Run: `npm test tests/persistence.test.ts tests/generationOrchestrator.test.ts tests/research.test.ts`
Expected: PASS

- [ ] **Step 2: Run full typecheck**

Run: `npm run lint`
Expected: PASS

- [ ] **Step 3: Run full test suite if focused verification is green**

Run: `npm test`
Expected: PASS
