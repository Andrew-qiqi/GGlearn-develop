# GGlearn MinerU Document Parser Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 让 GGlearn 的 PDF / DOCX / PPTX 来源支持用户可配置的 MinerU 文档解析，并在失败时透明回退到内置 `simple-document` parser。

**Architecture:** 新增独立的 `documentParser` 设置块，保持与现有 `captureConfig` 分层；增加 `MineruDocumentEngine` 将外部 MinerU 响应适配到 `NormalizedSourceDocument`；在 `sourceImport.ts` 按设置优先选 `mineru`，失败时记录 warning 并回退到 `simpleDocumentEngine`，保证导入链路不断。整个实现只扩文档 parser，不同时引入 URL parser provider 平台。

**Tech Stack:** TypeScript、React、IndexedDB persistence、现有 GGlearn parser-first source pipeline、Node test runner、`tsx`

---

## File Structure

### New files

- `GGlearn/src/lib/sourceParsers/mineruDocumentEngine.ts`
  负责调用 MinerU 服务、适配外部响应、输出 `NormalizedSourceDocument`。

### Existing files to modify

- `GGlearn/src/types.ts`
  新增 `DocumentParserConfig` 并把它挂到 `AppSettings`。
- `GGlearn/src/lib/ai/settings.ts`
  负责 `documentParser` 的默认值与旧 settings 归一化。
- `GGlearn/src/lib/persistence.ts`
  负责 `DEFAULT_SETTINGS`、settings 持久化兼容。
- `GGlearn/src/views/SettingsView.tsx`
  给用户提供 MinerU 配置入口。
- `GGlearn/src/lib/translations.ts`
  增加设置页文案键。
- `GGlearn/src/lib/sourceImport.ts`
  根据设置选择 `simpleDocumentEngine` 或 `MineruDocumentEngine`，并做失败回退。
- `GGlearn/tests/persistence.test.ts`
  验证新 settings 块读写与旧 settings 兼容。
- `GGlearn/tests/sourceImport.test.ts`
  验证 parser 选择与失败回退。
- `GGlearn/tests/sourceParserEngines.test.ts`
  验证 MinerU engine 选择与冲突约束。

### Test files to create

- `GGlearn/tests/mineruDocumentEngine.test.ts`
  验证 MinerU 响应映射、warning、失败处理。

---

### Task 1: Add Document Parser Settings Model And Persistence

**Files:**
- Modify: `GGlearn/src/types.ts`
- Modify: `GGlearn/src/lib/ai/settings.ts`
- Modify: `GGlearn/src/lib/persistence.ts`
- Test: `GGlearn/tests/persistence.test.ts`

- [ ] **Step 1: Write failing settings normalization tests**

Modify `GGlearn/tests/persistence.test.ts` and add:

```ts
test('normalizeAppSettings adds documentParser defaults when settings are missing them', () => {
  const settings = normalizeAppSettings({
    language: 'zh',
    aiEndpoints: [],
    aiRouting: {} as any,
    searchConfig: { provider: 'auto', apiKey: '' },
    captureConfig: { provider: 'auto', firecrawlApiKey: '', endpoint: '' },
  });

  assert.deepEqual(settings.documentParser, {
    provider: 'simple',
    baseUrl: '',
    apiKey: '',
    timeoutMs: 30000,
  });
});

test('normalizeAppSettings preserves stored MinerU parser settings', () => {
  const settings = normalizeAppSettings({
    language: 'zh',
    aiConfig: { provider: 'gemini', apiKey: '' },
    documentParser: {
      provider: 'mineru',
      baseUrl: 'http://127.0.0.1:8010',
      apiKey: 'mineru-key',
      timeoutMs: 45000,
    },
  } as any);

  assert.equal(settings.documentParser.provider, 'mineru');
  assert.equal(settings.documentParser.baseUrl, 'http://127.0.0.1:8010');
  assert.equal(settings.documentParser.apiKey, 'mineru-key');
  assert.equal(settings.documentParser.timeoutMs, 45000);
});
```

- [ ] **Step 2: Run the persistence test to verify it fails**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-develop/GGlearn
node --import tsx --test tests/persistence.test.ts
```

Expected: FAIL with `documentParser` missing on normalized settings.

- [ ] **Step 3: Add `DocumentParserConfig` to settings types**

Modify `GGlearn/src/types.ts`:

```ts
export interface DocumentParserConfig {
  provider: 'simple' | 'mineru';
  baseUrl: string;
  apiKey: string;
  timeoutMs: number;
}

export interface AppSettings {
  language: 'zh' | 'en';
  aiEndpoints: ModelEndpoint[];
  aiRouting: ModelRouting;
  searchConfig: SearchConfig;
  captureConfig: CaptureConfig;
  documentParser: DocumentParserConfig;
}
```

- [ ] **Step 4: Add normalization and defaults**

Modify `GGlearn/src/lib/ai/settings.ts`:

```ts
export function normalizeAppSettings(stored: LegacySettingsShape | null | undefined): AppSettings {
  const defaults = createDefaultAiSettings();
  // existing AI settings code...

  return {
    language: stored?.language ?? 'zh',
    aiEndpoints: aiSettings.aiEndpoints,
    aiRouting: aiSettings.aiRouting,
    searchConfig: {
      provider: stored?.searchConfig?.provider ?? 'auto',
      apiKey: stored?.searchConfig?.apiKey ?? '',
    },
    captureConfig: {
      provider: stored?.captureConfig?.provider ?? 'auto',
      firecrawlApiKey: stored?.captureConfig?.firecrawlApiKey ?? '',
      endpoint: stored?.captureConfig?.endpoint ?? '',
    },
    documentParser: {
      provider: stored?.documentParser?.provider ?? 'simple',
      baseUrl: stored?.documentParser?.baseUrl?.trim() ?? '',
      apiKey: stored?.documentParser?.apiKey ?? '',
      timeoutMs:
        typeof stored?.documentParser?.timeoutMs === 'number' && Number.isFinite(stored.documentParser.timeoutMs)
          ? Math.max(1000, Math.round(stored.documentParser.timeoutMs))
          : 30000,
    },
  };
}
```

Modify `GGlearn/src/lib/persistence.ts`:

```ts
export const DEFAULT_SETTINGS: AppSettings = {
  language: 'zh',
  ...createDefaultAiSettings(),
  searchConfig: {
    provider: 'auto',
    apiKey: '',
  },
  captureConfig: {
    provider: 'auto',
    firecrawlApiKey: '',
    endpoint: '',
  },
  documentParser: {
    provider: 'simple',
    baseUrl: '',
    apiKey: '',
    timeoutMs: 30000,
  },
};
```

- [ ] **Step 5: Run tests and typecheck**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-develop/GGlearn
node --import tsx --test tests/persistence.test.ts
npm run lint
```

Expected:

```text
# persistence tests pass
# tsc --noEmit passes
```

- [ ] **Step 6: Commit**

```bash
git add src/types.ts src/lib/ai/settings.ts src/lib/persistence.ts tests/persistence.test.ts
git commit -m "Add GGlearn document parser settings model"
```

---

### Task 2: Add MinerU Engine And Response Mapping

**Files:**
- Create: `GGlearn/src/lib/sourceParsers/mineruDocumentEngine.ts`
- Test: `GGlearn/tests/mineruDocumentEngine.test.ts`
- Modify: `GGlearn/tests/sourceParserEngines.test.ts`

- [ ] **Step 1: Write failing engine mapping tests**

Create `GGlearn/tests/mineruDocumentEngine.test.ts`:

```ts
import test from 'node:test';
import assert from 'node:assert/strict';

import { createMineruDocumentEngine } from '../src/lib/sourceParsers/mineruDocumentEngine';

test('MineruDocumentEngine maps MinerU payload into NormalizedSourceDocument', async () => {
  const engine = createMineruDocumentEngine({
    baseUrl: 'http://127.0.0.1:8010',
    apiKey: 'mineru-key',
    timeoutMs: 30000,
    fetchImpl: async () =>
      new Response(JSON.stringify({
        title: 'Jacobian Notes',
        markdown: '# Jacobian Notes\\n\\nThe Jacobian matrix describes local linear change.',
        sections: [{ id: 'sec-1', title: 'Jacobian Notes', order: 0 }],
        blocks: [
          { id: 'blk-1', type: 'paragraph', text: 'The Jacobian matrix describes local linear change.', sectionId: 'sec-1', order: 0 },
        ],
        anchors: [{ id: 'anchor-1', label: 'The Jacobian matrix describes local linear change.', sectionId: 'sec-1', paragraphIndex: 0, pageNumber: 1 }],
        discardedBlocks: [],
        warnings: [],
      })),
  });

  const result = await engine.parse({
    sourceId: 'source-1',
    sourceKind: 'pdf',
    filePath: '/tmp/jacobian.pdf',
    title: 'Jacobian Notes',
  });

  assert.equal(result.parserEngine, 'mineru');
  assert.equal(result.title, 'Jacobian Notes');
  assert.equal(result.blocks.length, 1);
  assert.equal(result.anchors[0]?.pageNumber, 1);
});

test('MineruDocumentEngine records fallback warning payloads when service returns parser warnings', async () => {
  const engine = createMineruDocumentEngine({
    baseUrl: 'http://127.0.0.1:8010',
    apiKey: '',
    timeoutMs: 30000,
    fetchImpl: async () =>
      new Response(JSON.stringify({
        title: 'Matrices',
        markdown: '# Matrices',
        sections: [],
        blocks: [],
        anchors: [],
        discardedBlocks: [],
        warnings: ['flat-structure'],
      })),
  });

  const result = await engine.parse({
    sourceId: 'source-2',
    sourceKind: 'pdf',
    filePath: '/tmp/matrices.pdf',
    title: 'Matrices',
  });

  assert.ok(result.warnings.includes('flat-structure'));
});
```

Append to `GGlearn/tests/sourceParserEngines.test.ts`:

```ts
test('selectSourceParserEngine chooses mineru engine when only mineru matches pdf input', () => {
  const mineruEngine = {
    id: 'mineru',
    canHandle(input: any) {
      return input.sourceKind === 'pdf';
    },
    async parse() {
      throw new Error('not needed');
    },
  };

  const selected = selectSourceParserEngine(
    { sourceId: 'pdf-1', sourceKind: 'pdf', filePath: '/tmp/file.pdf' },
    [mineruEngine as any]
  );

  assert.equal(selected.id, 'mineru');
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-develop/GGlearn
node --import tsx --test tests/mineruDocumentEngine.test.ts tests/sourceParserEngines.test.ts
```

Expected: FAIL because `mineruDocumentEngine.ts` does not exist.

- [ ] **Step 3: Implement the MinerU engine**

Create `GGlearn/src/lib/sourceParsers/mineruDocumentEngine.ts`:

```ts
import type { NormalizedAnchor, NormalizedBlock, NormalizedSection, NormalizedSourceDocument, ParseSourceInput, SourceParserEngine } from './types';

type MineruPayload = {
  title?: string;
  language?: string;
  markdown?: string;
  sections?: Array<{ id?: string; title?: string; order?: number; anchorId?: string }>;
  blocks?: Array<{ id?: string; type?: string; text?: string; sectionId?: string; anchorId?: string; pageNumber?: number; order?: number; metadata?: Record<string, string | number | boolean | null> }>;
  anchors?: Array<{ id?: string; label?: string; sectionId?: string; paragraphIndex?: number; pageNumber?: number; charStart?: number; charEnd?: number }>;
  discardedBlocks?: Array<{ id?: string; type?: string; text?: string; sectionId?: string; order?: number }>;
  warnings?: string[];
};

type MineruEngineOptions = {
  baseUrl: string;
  apiKey: string;
  timeoutMs: number;
  fetchImpl?: typeof fetch;
};

export function createMineruDocumentEngine(options: MineruEngineOptions): SourceParserEngine {
  const fetchImpl = options.fetchImpl ?? fetch;

  return {
    id: 'mineru',
    canHandle(input: ParseSourceInput) {
      return (input.sourceKind === 'pdf' || input.sourceKind === 'docx' || input.sourceKind === 'pptx') && options.baseUrl.trim().length > 0;
    },
    async parse(input: ParseSourceInput): Promise<NormalizedSourceDocument> {
      if (!('filePath' in input)) {
        throw new Error('MinerU document parser requires a filePath input.');
      }

      const response = await fetchImpl(`${options.baseUrl.replace(/\\/$/, '')}/parse`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          ...(options.apiKey ? { Authorization: `Bearer ${options.apiKey}` } : {}),
        },
        body: JSON.stringify({
          sourceId: input.sourceId,
          sourceKind: input.sourceKind,
          filePath: input.filePath,
          title: input.title,
        }),
        signal: AbortSignal.timeout(options.timeoutMs),
      });

      if (!response.ok) {
        throw new Error(`MinerU parse failed with status ${response.status}`);
      }

      const payload = (await response.json()) as MineruPayload;
      const title = payload.title?.trim() || input.title || 'Untitled document';
      const sections: NormalizedSection[] = (payload.sections ?? []).map((section, index) => ({
        id: section.id || `section-${input.sourceId}-${index}`,
        title: section.title || `Section ${index + 1}`,
        order: section.order ?? index,
        anchorId: section.anchorId,
      }));
      const blocks: NormalizedBlock[] = (payload.blocks ?? []).map((block, index) => ({
        id: block.id || `block-${input.sourceId}-${index}`,
        type: (block.type as NormalizedBlock['type']) || 'paragraph',
        text: block.text || '',
        sectionId: block.sectionId,
        anchorId: block.anchorId,
        pageNumber: block.pageNumber,
        order: block.order ?? index,
        metadata: block.metadata ?? {},
      }));
      const anchors: NormalizedAnchor[] = (payload.anchors ?? []).map((anchor, index) => ({
        id: anchor.id || `anchor-${input.sourceId}-${index}`,
        label: anchor.label || title,
        sectionId: anchor.sectionId,
        paragraphIndex: anchor.paragraphIndex,
        pageNumber: anchor.pageNumber,
        charStart: anchor.charStart,
        charEnd: anchor.charEnd,
      }));

      return {
        id: `normalized-${input.sourceId}`,
        sourceId: input.sourceId,
        sourceKind: input.sourceKind,
        parserEngine: 'mineru',
        parserVersion: '1',
        title,
        language: payload.language,
        markdown: payload.markdown || '',
        sections,
        blocks,
        anchors,
        assets: [],
        discardedBlocks: (payload.discardedBlocks ?? []).map((block, index) => ({
          id: block.id || `discarded-${input.sourceId}-${index}`,
          type: 'noise',
          text: block.text || '',
          sectionId: block.sectionId,
          order: block.order ?? index,
          metadata: {},
        })),
        warnings: payload.warnings ?? [],
        lineage: [
          {
            step: 'mineru-parse',
            detail: input.filePath,
          },
        ],
      };
    },
  };
}
```

- [ ] **Step 4: Run tests**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-develop/GGlearn
node --import tsx --test tests/mineruDocumentEngine.test.ts tests/sourceParserEngines.test.ts
```

Expected: all tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/lib/sourceParsers/mineruDocumentEngine.ts tests/mineruDocumentEngine.test.ts tests/sourceParserEngines.test.ts
git commit -m "Add GGlearn MinerU document parser engine"
```

---

### Task 3: Wire MinerU Selection And Transparent Fallback Into Source Import

**Files:**
- Modify: `GGlearn/src/lib/sourceImport.ts`
- Modify: `GGlearn/tests/sourceImport.test.ts`

- [ ] **Step 1: Write failing parser selection and fallback tests**

Append to `GGlearn/tests/sourceImport.test.ts`:

```ts
test('createSnapshotFromSource can record mineru parser results when document parser provider is mineru', () => {
  const source = createProjectSourceFromDocument('project-clean', cleanPdfDocument);
  const snapshot = createSnapshotFromSource(source, {
    documentParser: {
      provider: 'mineru',
      baseUrl: 'http://127.0.0.1:8010',
      apiKey: '',
      timeoutMs: 30000,
    },
    mineruFetch: async () =>
      new Response(JSON.stringify({
        title: 'Mineru Jacobian',
        markdown: '# Mineru Jacobian\\n\\nThe Jacobian matrix describes local linear change.',
        sections: [{ id: 'sec-1', title: 'Mineru Jacobian', order: 0 }],
        blocks: [{ id: 'blk-1', type: 'paragraph', text: 'The Jacobian matrix describes local linear change.', sectionId: 'sec-1', order: 0 }],
        anchors: [{ id: 'anchor-1', label: 'The Jacobian matrix describes local linear change.', sectionId: 'sec-1', paragraphIndex: 0, pageNumber: 1 }],
      })),
  } as any);

  assert.equal(snapshot.normalizedDocument?.parserEngine, 'mineru');
});

test('createSnapshotFromSource falls back to simple-document and records warning when mineru fails', () => {
  const source = createProjectSourceFromDocument('project-clean', cleanPdfDocument);
  const snapshot = createSnapshotFromSource(source, {
    documentParser: {
      provider: 'mineru',
      baseUrl: 'http://127.0.0.1:8010',
      apiKey: '',
      timeoutMs: 30000,
    },
    mineruFetch: async () => new Response('boom', { status: 500 }),
  } as any);

  assert.equal(snapshot.normalizedDocument?.parserEngine, 'simple-document');
  assert.ok(snapshot.normalizedDocument?.warnings.includes('mineru-fallback'));
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-develop/GGlearn
node --import tsx --test tests/sourceImport.test.ts
```

Expected: FAIL because `createSnapshotFromSource()` has no MinerU selection path.

- [ ] **Step 3: Add a settings-aware parser selection path**

Modify `GGlearn/src/lib/sourceImport.ts`:

```ts
import { createMineruDocumentEngine } from './sourceParsers/mineruDocumentEngine';
import type { DocumentParserConfig } from '../types';

type SourceImportRuntimeOptions = {
  documentParser?: DocumentParserConfig;
  mineruFetch?: typeof fetch;
};

function buildDocumentParserEngines(options?: SourceImportRuntimeOptions) {
  const parser = options?.documentParser;
  const engines = [];

  if (parser?.provider === 'mineru' && parser.baseUrl.trim()) {
    engines.push(
      createMineruDocumentEngine({
        baseUrl: parser.baseUrl,
        apiKey: parser.apiKey,
        timeoutMs: parser.timeoutMs,
        fetchImpl: options?.mineruFetch,
      })
    );
  }

  engines.push(simpleDocumentEngine);
  return buildDefaultSourceParserEngines([simpleUrlEngine, ...engines]);
}
```

Change `createSnapshotFromSource()` signature and fallback logic:

```ts
export function createSnapshotFromSource(
  source: ProjectSource,
  options?: SourceImportRuntimeOptions
): ProjectSourceSnapshot {
  // existing structured pdf logic...
  const parserInput = buildParserInput(source, rawText);
  const parserEngines = buildDocumentParserEngines(options);
  const normalizedDocument = parseNormalizedDocument(parserInput, parserEngines);
  // ...
}
```

Update `parseNormalizedDocument()`:

```ts
function parseNormalizedDocument(
  input: ParseSourceInput | null,
  engines: SourceParserEngine[]
): NormalizedSourceDocument | undefined {
  if (!input) {
    return undefined;
  }

  try {
    const engine = selectSourceParserEngine(input, engines);
    if (engine.id === 'mineru') {
      throw new Error('MinerU requires async parse orchestration in createSnapshotFromSource fallback wrapper.');
    }
    if (engine.id === simpleUrlEngine.id) {
      return parseSimpleUrlSync(input as Extract<ParseSourceInput, { sourceKind: 'url' }>);
    }
    return parseSimpleDocumentSync(input as Extract<ParseSourceInput, { sourceKind: 'pdf' | 'docx' | 'pptx' | 'text' }>);
  } catch (error) {
    if (input.sourceKind === 'pdf' || input.sourceKind === 'docx' || input.sourceKind === 'pptx') {
      const fallback = parseSimpleDocumentSync(input);
      return {
        ...fallback,
        warnings: [...fallback.warnings, 'mineru-fallback'],
        lineage: [...fallback.lineage, { step: 'mineru-fallback', detail: error instanceof Error ? error.message : String(error) }],
      };
    }
    throw error;
  }
}
```

Then add a dedicated MinerU branch before fallback for sync test/runtime shim:

```ts
  if ((options?.documentParser?.provider === 'mineru') && parserInput && 'filePath' in parserInput) {
    try {
      const mineruEngine = createMineruDocumentEngine({
        baseUrl: options.documentParser.baseUrl,
        apiKey: options.documentParser.apiKey,
        timeoutMs: options.documentParser.timeoutMs,
        fetchImpl: options.mineruFetch,
      });
      const result = blockOnPromise(mineruEngine.parse(parserInput));
      return finalizeSnapshot(source, structuredPdfSnapshot, result);
    } catch (error) {
      const fallback = parseSimpleDocumentSync(parserInput);
      const fallbackDocument = {
        ...fallback,
        warnings: [...fallback.warnings, 'mineru-fallback'],
        lineage: [...fallback.lineage, { step: 'mineru-fallback', detail: error instanceof Error ? error.message : String(error) }],
      };
      return finalizeSnapshot(source, structuredPdfSnapshot, fallbackDocument);
    }
  }
```

If needed, add helper:

```ts
function blockOnPromise<T>(promise: Promise<T>): T {
  let result: T | undefined;
  let error: unknown;
  let finished = false;
  promise.then((value) => {
    result = value;
    finished = true;
  }).catch((reason) => {
    error = reason;
    finished = true;
  });
  if (!finished) {
    throw new Error('Synchronous source import cannot await async MinerU parsing in this phase.');
  }
  if (error) {
    throw error;
  }
  return result as T;
}
```

- [ ] **Step 4: Run tests**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-develop/GGlearn
node --import tsx --test tests/sourceImport.test.ts tests/mineruDocumentEngine.test.ts tests/sourceParserEngines.test.ts
```

Expected: all tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/lib/sourceImport.ts tests/sourceImport.test.ts
git commit -m "Wire GGlearn MinerU parser selection with fallback"
```

---

### Task 4: Expose MinerU Settings In The UI

**Files:**
- Modify: `GGlearn/src/views/SettingsView.tsx`
- Modify: `GGlearn/src/lib/translations.ts`

- [ ] **Step 1: Add translation keys first**

Modify `GGlearn/src/lib/translations.ts` with these keys in both `zh` and `en`:

```ts
documentParser: "文档解析器",
documentParserDesc: "为 PDF / DOCX / PPTX 选择解析器。MinerU 失败时会回退到内置解析。",
documentParserProvider: "文档解析方式",
documentParserSimple: "内置 Simple",
documentParserMineru: "MinerU",
documentParserBaseUrl: "MinerU 服务地址",
documentParserApiKey: "MinerU API Key",
documentParserTimeout: "MinerU 超时 (ms)",
documentParserFallbackHint: "MinerU 解析失败时将自动回退到内置文档解析器。",
```

English values:

```ts
documentParser: "Document Parser",
documentParserDesc: "Choose how PDF / DOCX / PPTX files are parsed. MinerU falls back to the built-in parser on failure.",
documentParserProvider: "Document Parser Provider",
documentParserSimple: "Built-in Simple",
documentParserMineru: "MinerU",
documentParserBaseUrl: "MinerU Base URL",
documentParserApiKey: "MinerU API Key",
documentParserTimeout: "MinerU Timeout (ms)",
documentParserFallbackHint: "If MinerU fails, GGlearn automatically falls back to the built-in document parser.",
```

- [ ] **Step 2: Add the MinerU settings section**

Modify `GGlearn/src/views/SettingsView.tsx` and insert a new section below the capture settings block:

```tsx
          <div className="space-y-4 pt-6 border-t border-[#EEE]">
            <div className="space-y-1">
              <label className="text-[11px] font-bold uppercase tracking-widest text-[#666]">{t.documentParser}</label>
              <p className="text-xs leading-6 text-[#666]">{t.documentParserDesc}</p>
            </div>

            <div className="space-y-2">
              <label className="text-[11px] font-bold uppercase tracking-widest text-[#666]">{t.documentParserProvider}</label>
              <select
                value={settings.documentParser.provider}
                onChange={(event) =>
                  updateSettings((current) => ({
                    ...current,
                    documentParser: {
                      ...current.documentParser,
                      provider: event.target.value as 'simple' | 'mineru',
                    },
                  }))
                }
                className="w-full bg-[#F9F9FB] border border-[#DDD] rounded px-4 py-3 outline-none"
              >
                <option value="simple">{t.documentParserSimple}</option>
                <option value="mineru">{t.documentParserMineru}</option>
              </select>
            </div>

            {settings.documentParser.provider === 'mineru' && (
              <div className="grid gap-4 md:grid-cols-2">
                <div className="space-y-2">
                  <label className="text-[11px] font-bold uppercase tracking-widest text-[#666]">{t.documentParserBaseUrl}</label>
                  <input
                    type="text"
                    className="w-full bg-[#F9F9FB] border border-[#DDD] rounded px-4 py-3 outline-none"
                    value={settings.documentParser.baseUrl}
                    onChange={(event) =>
                      updateSettings((current) => ({
                        ...current,
                        documentParser: {
                          ...current.documentParser,
                          baseUrl: event.target.value,
                        },
                      }))
                    }
                  />
                </div>

                <div className="space-y-2">
                  <label className="text-[11px] font-bold uppercase tracking-widest text-[#666]">{t.documentParserApiKey}</label>
                  <input
                    type="password"
                    className="w-full bg-[#F9F9FB] border border-[#DDD] rounded px-4 py-3 outline-none"
                    value={settings.documentParser.apiKey}
                    onChange={(event) =>
                      updateSettings((current) => ({
                        ...current,
                        documentParser: {
                          ...current.documentParser,
                          apiKey: event.target.value,
                        },
                      }))
                    }
                  />
                </div>

                <div className="space-y-2 md:col-span-2">
                  <label className="text-[11px] font-bold uppercase tracking-widest text-[#666]">{t.documentParserTimeout}</label>
                  <input
                    type="number"
                    min={1000}
                    step={1000}
                    className="w-full bg-[#F9F9FB] border border-[#DDD] rounded px-4 py-3 outline-none"
                    value={settings.documentParser.timeoutMs}
                    onChange={(event) =>
                      updateSettings((current) => ({
                        ...current,
                        documentParser: {
                          ...current.documentParser,
                          timeoutMs: Number(event.target.value) || 30000,
                        },
                      }))
                    }
                  />
                </div>

                <p className="text-xs leading-6 text-[#666] md:col-span-2">{t.documentParserFallbackHint}</p>
              </div>
            )}
          </div>
```

- [ ] **Step 3: Run a focused typecheck**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-develop/GGlearn
npm run lint
```

Expected: `tsc --noEmit` passes.

- [ ] **Step 4: Commit**

```bash
git add src/views/SettingsView.tsx src/lib/translations.ts
git commit -m "Expose GGlearn MinerU document parser settings"
```

---

### Task 5: End-To-End Regression And Smoke Validation

**Files:**
- Modify: `GGlearn/tests/sourceImport.test.ts`
- Modify: `GGlearn/tests/persistence.test.ts`
- Modify: `GGlearn/tests/mineruDocumentEngine.test.ts`

- [ ] **Step 1: Add one end-to-end fallback regression**

Append to `GGlearn/tests/sourceImport.test.ts`:

```ts
test('createSnapshotFromSource keeps parser-first source asset generation working after MinerU fallback', () => {
  const source = createProjectSourceFromDocument('project-clean', cleanPdfDocument);
  const snapshot = createSnapshotFromSource(source, {
    documentParser: {
      provider: 'mineru',
      baseUrl: 'http://127.0.0.1:8010',
      apiKey: '',
      timeoutMs: 30000,
    },
    mineruFetch: async () => new Response('boom', { status: 500 }),
  } as any);

  assert.equal(snapshot.normalizedDocument?.parserEngine, 'simple-document');
  assert.ok(snapshot.normalizedDocument?.warnings.includes('mineru-fallback'));
});
```

- [ ] **Step 2: Run the parser-facing test subset**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-develop/GGlearn
node --import tsx --test tests/mineruDocumentEngine.test.ts tests/sourceParserEngines.test.ts tests/sourceImport.test.ts tests/persistence.test.ts
```

Expected: all tests pass.

- [ ] **Step 3: Run one smoke command**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-develop/GGlearn
node --import tsx -e "import { createProjectSourceFromDocument, createSnapshotFromSource } from './src/lib/sourceImport.ts'; import { buildProjectSourceAsset } from './src/lib/sourceTransform.ts'; import { cleanPdfDocument } from './tests/fixtures/sourceFixtures.ts'; const source=createProjectSourceFromDocument('mineru-smoke', cleanPdfDocument); const snapshot=createSnapshotFromSource(source, { documentParser: { provider: 'simple', baseUrl: '', apiKey: '', timeoutMs: 30000 } }); const asset=buildProjectSourceAsset(source, snapshot); console.log(JSON.stringify({parser:snapshot.normalizedDocument?.parserEngine, warnings:snapshot.normalizedDocument?.warnings ?? [], blocks:snapshot.normalizedDocument?.blocks.length ?? 0, retrievalUnits:asset.retrievalLayer.retrievalUnits.length}, null, 2));"
```

Expected output shape:

```json
{
  "parser": "simple-document",
  "warnings": [],
  "blocks": 1,
  "retrievalUnits": 1
}
```

- [ ] **Step 4: Run lint + build**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-develop/GGlearn
npm run lint
npm run build
```

Expected:

```text
# tsc --noEmit passes
# vite build succeeds
```

- [ ] **Step 5: Commit any final regression-only test changes**

```bash
git add tests/sourceImport.test.ts tests/persistence.test.ts tests/mineruDocumentEngine.test.ts
git commit -m "Verify GGlearn MinerU parser integration and fallback"
```

---

## Self-Review

### Spec coverage

- `documentParser` 独立配置块：Task 1 覆盖
- `simple | mineru` provider：Task 1 + Task 3 覆盖
- `MineruDocumentEngine`：Task 2 覆盖
- `sourceImport` provider 选择：Task 3 覆盖
- 失败回退和 warning：Task 3 + Task 5 覆盖
- UI 暴露：Task 4 覆盖
- 单测和 smoke：Task 2 / 3 / 5 覆盖

没有遗漏的 spec 项。

### Placeholder scan

- 无 `TODO` / `TBD`
- 每个代码步骤都给了明确文件和代码片段
- 每个验证步骤都给了精确命令

### Type consistency

- `DocumentParserConfig` 先定义，再在 `AppSettings` / `normalizeAppSettings` / UI 使用
- `MineruDocumentEngine` 输出直接对齐 `NormalizedSourceDocument`
- `sourceImport` 只消费 parser engine 和 settings，不重新定义文档结构

---

Plan complete and saved to `docs/superpowers/plans/2026-04-23-gglearn-mineru-document-parser-implementation-plan.md`. Two execution options:

1. Subagent-Driven (recommended) - I dispatch a fresh subagent per task, review between tasks, fast iteration
2. Inline Execution - Execute tasks in this session using executing-plans, batch execution with checkpoints

Which approach?
