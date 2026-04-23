# GGlearn Source Parser-First Architecture Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rebuild GGlearn's source ingestion path around a parser-first normalized document layer so PDF and URL sources stop polluting concept planning and chapter generation.

**Architecture:** Introduce a new `NormalizedSourceDocument` truth layer plus parser-engine interfaces, route PDF/Office and URL/Web through dedicated engines, and rebuild source asset construction from normalized blocks instead of raw-text heuristics. Keep a narrow compatibility fallback (`simple` engines + legacy fields) only where needed to avoid breaking persistence and existing tests during migration.

**Tech Stack:** TypeScript, Node test runner, `tsx`, existing GGlearn source import/transform modules, existing persistence layer, parser-backed source normalization, concept-first outline generation

---

## File Structure

### New files

- `GGlearn/src/lib/sourceParsers/types.ts`
  Defines `NormalizedSourceDocument`, normalized block/section/anchor types, parser input/output types.
- `GGlearn/src/lib/sourceParsers/engines.ts`
  Defines `SourceParserEngine`, engine registry, parser selection helpers, and config defaults.
- `GGlearn/src/lib/sourceParsers/simpleDocumentEngine.ts`
  Minimal parser-backed fallback for local document text and current PDF snapshot output.
- `GGlearn/src/lib/sourceParsers/simpleUrlEngine.ts`
  Minimal structured URL fallback that converts cleaned article text into normalized markdown/blocks.
- `GGlearn/tests/sourceParserEngines.test.ts`
  Covers normalized document shapes, engine selection, and discarded/noise block behavior.

### Existing files to modify

- `GGlearn/src/types.ts`
  Add normalized document types and wire them into `ProjectSourceSnapshot` / `ProjectSourceAsset`.
- `GGlearn/src/lib/sourceImport.ts`
  Replace direct snapshot creation with parser-engine-backed normalized document creation and snapshot derivation.
- `GGlearn/src/lib/sourceTransform.ts`
  Rebuild source asset generation from normalized blocks; demote heuristic keyword/chapter-seed generation to fallback only.
- `GGlearn/src/lib/ai/sourceAsset.ts`
  Prevent AI enrichment from mutating parser truth; limit to teaching overlays.
- `GGlearn/src/lib/ai/projectConceptPlanning.ts`
  Aggregate concepts from normalized blocks + evidence refs instead of raw keyword guesses.
- `GGlearn/src/lib/ai/outlineGeneration.ts`
  Make outline generation depend on concept-first source assets only; stop relying on `chapterSeeds`.
- `GGlearn/src/lib/ai/textbookGeneration.ts`
  Use concept/evidence bindings from normalized assets for chapter evidence packs.
- `GGlearn/src/lib/persistence.ts`
  Backfill normalized-document-compatible defaults for legacy projects.
- `GGlearn/tests/sourceImport.test.ts`
  Assert parser-backed snapshots and normalized document metadata.
- `GGlearn/tests/sourceTransform.test.ts`
  Assert asset generation from normalized docs.
- `GGlearn/tests/projectConceptPlanning.test.ts`
  Assert concept clustering consumes structured evidence.
- `GGlearn/tests/outlineGeneration.test.ts`
  Assert dirty seed-like signals do not leak into outline planning.
- `GGlearn/tests/textbookGeneration.test.ts`
  Assert concept-bound evidence pack selection still works after migration.
- `GGlearn/tests/persistence.test.ts`
  Assert normalized project backfills for legacy persisted data.

### Files to leave untouched in this plan

- `GGlearn/src/App.tsx`
  No UI expansion in this phase; only adapt if type changes force it later.
- `GGlearn/src/views/*`
  Reader/editor UI changes are out of scope until the new source truth layer is stable.

---

### Task 1: Introduce Normalized Source Document Types And Parser Engine Contracts

**Files:**
- Create: `GGlearn/src/lib/sourceParsers/types.ts`
- Create: `GGlearn/src/lib/sourceParsers/engines.ts`
- Modify: `GGlearn/src/types.ts`
- Test: `GGlearn/tests/sourceParserEngines.test.ts`

- [ ] **Step 1: Write the failing normalized source document tests**

Create `GGlearn/tests/sourceParserEngines.test.ts` with these tests:

```ts
import test from 'node:test';
import assert from 'node:assert/strict';

import type { ParseSourceInput, NormalizedSourceDocument, SourceParserEngine } from '../src/lib/sourceParsers/types';
import { selectSourceParserEngine } from '../src/lib/sourceParsers/engines';

const fakeDocumentEngine: SourceParserEngine = {
  id: 'fake-document',
  canHandle(input: ParseSourceInput) {
    return input.sourceKind === 'pdf';
  },
  async parse(input: ParseSourceInput): Promise<NormalizedSourceDocument> {
    return {
      id: `normalized-${input.sourceId}`,
      sourceId: input.sourceId,
      sourceKind: 'pdf',
      parserEngine: 'fake-document',
      parserVersion: '1',
      title: 'Linear Algebra',
      language: 'en',
      markdown: '# Linear Algebra\n\nA matrix maps vectors.',
      sections: [{ id: 'section-1', title: 'Linear Algebra', order: 0 }],
      blocks: [
        {
          id: 'block-1',
          type: 'paragraph',
          text: 'A matrix maps vectors.',
          sectionId: 'section-1',
          anchorId: 'anchor-1',
          order: 0,
          metadata: {},
        },
      ],
      anchors: [{ id: 'anchor-1', label: 'A matrix maps vectors.', sectionTitle: 'Linear Algebra', paragraphIndex: 0 }],
      assets: [],
      discardedBlocks: [],
      warnings: [],
      lineage: { originalFilePath: '/tmp/linear-algebra.pdf', generatedAt: 1 },
    };
  },
};

const fakeUrlEngine: SourceParserEngine = {
  id: 'fake-url',
  canHandle(input: ParseSourceInput) {
    return input.sourceKind === 'url';
  },
  async parse(input: ParseSourceInput): Promise<NormalizedSourceDocument> {
    return {
      id: `normalized-${input.sourceId}`,
      sourceId: input.sourceId,
      sourceKind: 'url',
      parserEngine: 'fake-url',
      parserVersion: '1',
      title: 'Jacobian Article',
      language: 'zh',
      markdown: '# Jacobian Article\n\n雅可比矩阵描述局部线性变化。',
      sections: [{ id: 'section-1', title: 'Jacobian Article', order: 0 }],
      blocks: [
        {
          id: 'block-1',
          type: 'paragraph',
          text: '雅可比矩阵描述局部线性变化。',
          sectionId: 'section-1',
          anchorId: 'anchor-1',
          order: 0,
          metadata: {},
        },
      ],
      anchors: [{ id: 'anchor-1', label: '雅可比矩阵描述局部线性变化。', sectionTitle: 'Jacobian Article', paragraphIndex: 0 }],
      assets: [],
      discardedBlocks: [
        {
          id: 'noise-1',
          type: 'noise',
          text: 'CurrentTime2',
          sectionId: 'section-1',
          anchorId: 'anchor-1',
          order: 99,
          metadata: { reason: 'navigation-token' },
        },
      ],
      warnings: ['navigation-heavy-page'],
      lineage: { originalUrl: 'https://example.com/jacobian', generatedAt: 1 },
    };
  },
};

test('selectSourceParserEngine chooses document engines for pdf-like sources', () => {
  const engine = selectSourceParserEngine(
    { sourceId: 'source-pdf', sourceKind: 'pdf', filePath: '/tmp/file.pdf' },
    [fakeUrlEngine, fakeDocumentEngine]
  );

  assert.equal(engine.id, 'fake-document');
});

test('selectSourceParserEngine chooses url engines for url sources', () => {
  const engine = selectSourceParserEngine(
    { sourceId: 'source-url', sourceKind: 'url', url: 'https://example.com' },
    [fakeDocumentEngine, fakeUrlEngine]
  );

  assert.equal(engine.id, 'fake-url');
});

test('normalized source documents preserve discarded blocks explicitly', async () => {
  const normalized = await fakeUrlEngine.parse({
    sourceId: 'source-url',
    sourceKind: 'url',
    url: 'https://example.com',
  });

  assert.equal(normalized.discardedBlocks.length, 1);
  assert.equal(normalized.discardedBlocks[0].type, 'noise');
  assert.equal(normalized.discardedBlocks[0].text, 'CurrentTime2');
});
```

- [ ] **Step 2: Run the parser-engine test to verify it fails**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-develop/GGlearn
node --import tsx --test tests/sourceParserEngines.test.ts
```

Expected: FAIL with module-not-found errors for `sourceParsers/types` or missing exports like `selectSourceParserEngine`.

- [ ] **Step 3: Add normalized document and parser engine type definitions**

Create `GGlearn/src/lib/sourceParsers/types.ts`:

```ts
export type NormalizedBlockType =
  | 'heading'
  | 'paragraph'
  | 'list'
  | 'quote'
  | 'table'
  | 'formula'
  | 'figure'
  | 'caption'
  | 'code'
  | 'reference'
  | 'noise';

export interface NormalizedSection {
  id: string;
  title: string;
  order: number;
  parentSectionId?: string;
}

export interface NormalizedAnchor {
  id: string;
  label: string;
  sectionTitle?: string;
  paragraphIndex?: number;
  pageNumber?: number;
  charStart?: number;
  charEnd?: number;
}

export interface NormalizedEmbeddedAsset {
  id: string;
  kind: 'image' | 'table' | 'figure';
  path?: string;
  caption?: string;
  pageNumber?: number;
}

export interface NormalizedBlock {
  id: string;
  type: NormalizedBlockType;
  text: string;
  sectionId: string;
  anchorId?: string;
  order: number;
  pageNumber?: number;
  bbox?: { x: number; y: number; width: number; height: number };
  metadata: Record<string, unknown>;
}

export interface NormalizedSourceDocument {
  id: string;
  sourceId: string;
  sourceKind: 'pdf' | 'docx' | 'pptx' | 'url' | 'text';
  parserEngine: string;
  parserVersion: string;
  title: string;
  language?: string;
  markdown: string;
  sections: NormalizedSection[];
  blocks: NormalizedBlock[];
  anchors: NormalizedAnchor[];
  assets: NormalizedEmbeddedAsset[];
  discardedBlocks: NormalizedBlock[];
  warnings: string[];
  lineage: {
    originalUrl?: string;
    originalFilePath?: string;
    generatedAt: number;
  };
}

export interface ParseSourceInput {
  sourceId: string;
  sourceKind: 'pdf' | 'docx' | 'pptx' | 'url' | 'text';
  filePath?: string;
  url?: string;
  rawText?: string;
  language?: 'zh' | 'en';
  title?: string;
}

export interface SourceParserEngine {
  id: string;
  canHandle(input: ParseSourceInput): boolean;
  parse(input: ParseSourceInput): Promise<NormalizedSourceDocument>;
}
```

Modify `GGlearn/src/types.ts` by importing these types and extending snapshot/asset types:

```ts
import type {
  NormalizedBlock,
  NormalizedEmbeddedAsset,
  NormalizedSection,
  NormalizedSourceDocument,
} from './lib/sourceParsers/types';
```

Add to `ProjectSourceSnapshot`:

```ts
  normalizedDocument?: NormalizedSourceDocument;
```

Replace `noiseBlocks: string[];` inside `ProjectSourceAsset['structureLayer']` with:

```ts
    normalizedSections?: NormalizedSection[];
    normalizedBlocks?: NormalizedBlock[];
    embeddedAssets?: NormalizedEmbeddedAsset[];
    noiseBlocks: string[];
```

- [ ] **Step 4: Add engine selection helpers**

Create `GGlearn/src/lib/sourceParsers/engines.ts`:

```ts
import type { ParseSourceInput, SourceParserEngine } from './types';

export function selectSourceParserEngine(
  input: ParseSourceInput,
  engines: SourceParserEngine[]
): SourceParserEngine {
  const matched = engines.find((engine) => engine.canHandle(input));
  if (!matched) {
    throw new Error(`No source parser engine found for sourceKind=${input.sourceKind}`);
  }
  return matched;
}

export function buildDefaultSourceParserEngines(engines: SourceParserEngine[]): SourceParserEngine[] {
  return [...engines];
}
```

- [ ] **Step 5: Run the parser-engine tests and typecheck the changed files**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-develop/GGlearn
node --import tsx --test tests/sourceParserEngines.test.ts
node --import tsx -e "import './src/types.ts'; import './src/lib/sourceParsers/types.ts'; import './src/lib/sourceParsers/engines.ts'; console.log('parser-types-ok')"
```

Expected:

```text
ok 1 - selectSourceParserEngine chooses document engines for pdf-like sources
ok 2 - selectSourceParserEngine chooses url engines for url sources
ok 3 - normalized source documents preserve discarded blocks explicitly
parser-types-ok
```

- [ ] **Step 6: Commit the type and engine contract layer**

Run:

```bash
git add GGlearn/src/lib/sourceParsers/types.ts GGlearn/src/lib/sourceParsers/engines.ts GGlearn/src/types.ts GGlearn/tests/sourceParserEngines.test.ts
git commit -m "Establish parser-first source contracts for GGlearn"
```

Expected: a single commit containing only the type and engine contract changes.

### Task 2: Add Simple Parser Engines And Snapshot Derivation From Normalized Documents

**Files:**
- Create: `GGlearn/src/lib/sourceParsers/simpleDocumentEngine.ts`
- Create: `GGlearn/src/lib/sourceParsers/simpleUrlEngine.ts`
- Modify: `GGlearn/src/lib/sourceImport.ts`
- Test: `GGlearn/tests/sourceImport.test.ts`

- [ ] **Step 1: Add failing tests for parser-backed snapshot creation**

Append these tests to `GGlearn/tests/sourceImport.test.ts`:

```ts
test('createSnapshotFromSource stores normalizedDocument for clean PDFs', () => {
  const source = createProjectSourceFromDocument('project-clean', cleanPdfDocument);
  const snapshot = createSnapshotFromSource(source);

  assert.ok(snapshot.normalizedDocument);
  assert.equal(snapshot.normalizedDocument?.sourceKind, 'pdf');
  assert.equal(snapshot.normalizedDocument?.parserEngine, 'simple-document');
  assert.ok(snapshot.normalizedDocument?.blocks.length);
});

test('createSnapshotFromSource stores discarded blocks for noisy URL-like content', () => {
  const source = createProjectSourceFromDocument('project-url', {
    ...urlDocument,
    fullText: `${urlDocument.fullText}\n\nCurrentTime2\nReadEyes2\nCollect2`,
  });
  const snapshot = createSnapshotFromSource(source);

  assert.ok(snapshot.normalizedDocument);
  assert.equal(snapshot.normalizedDocument?.sourceKind, 'url');
  assert.ok((snapshot.normalizedDocument?.discardedBlocks.length ?? 0) >= 1);
});
```

- [ ] **Step 2: Run the source import tests to verify the new assertions fail**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-develop/GGlearn
node --import tsx --test tests/sourceImport.test.ts
```

Expected: FAIL because `snapshot.normalizedDocument` is `undefined`.

- [ ] **Step 3: Implement the simple document parser engine**

Create `GGlearn/src/lib/sourceParsers/simpleDocumentEngine.ts`:

```ts
import type { NormalizedAnchor, NormalizedBlock, NormalizedSourceDocument, SourceParserEngine } from './types';

function splitMarkdownParagraphs(text: string): string[] {
  return text
    .replace(/\r/g, '')
    .split(/\n{2,}/)
    .map((part) => part.trim())
    .filter(Boolean);
}

export const simpleDocumentEngine: SourceParserEngine = {
  id: 'simple-document',
  canHandle(input) {
    return input.sourceKind === 'pdf' || input.sourceKind === 'docx' || input.sourceKind === 'pptx' || input.sourceKind === 'text';
  },
  async parse(input): Promise<NormalizedSourceDocument> {
    const title = input.title?.trim() || 'Untitled source';
    const paragraphs = splitMarkdownParagraphs(input.rawText || '');
    const anchors: NormalizedAnchor[] = paragraphs.map((paragraph, index) => ({
      id: `anchor-${input.sourceId}-${index}`,
      label: paragraph.slice(0, 48) || `${title} ${index + 1}`,
      sectionTitle: title,
      paragraphIndex: index,
    }));
    const blocks: NormalizedBlock[] = paragraphs.map((paragraph, index) => ({
      id: `block-${input.sourceId}-${index}`,
      type: 'paragraph',
      text: paragraph,
      sectionId: `section-${input.sourceId}-0`,
      anchorId: anchors[index]?.id,
      order: index,
      metadata: {},
    }));

    return {
      id: `normalized-${input.sourceId}`,
      sourceId: input.sourceId,
      sourceKind: input.sourceKind,
      parserEngine: 'simple-document',
      parserVersion: '1',
      title,
      markdown: paragraphs.join('\n\n'),
      sections: [{ id: `section-${input.sourceId}-0`, title, order: 0 }],
      blocks,
      anchors,
      assets: [],
      discardedBlocks: [],
      warnings: [],
      lineage: {
        originalFilePath: input.filePath,
        generatedAt: Date.now(),
      },
    };
  },
};
```

- [ ] **Step 4: Implement the simple URL parser engine with explicit discarded blocks**

Create `GGlearn/src/lib/sourceParsers/simpleUrlEngine.ts`:

```ts
import type { NormalizedBlock, NormalizedSourceDocument, SourceParserEngine } from './types';

const NOISE_PATTERNS = [/^https?:\/\//i, /CurrentTime\d+/i, /ReadEyes\d+/i, /Collect\d+/i];

function classifyLines(text: string): { kept: string[]; discarded: string[] } {
  const lines = text
    .replace(/\r/g, '')
    .split(/\n+/)
    .map((line) => line.trim())
    .filter(Boolean);

  const kept: string[] = [];
  const discarded: string[] = [];

  for (const line of lines) {
    if (NOISE_PATTERNS.some((pattern) => pattern.test(line))) {
      discarded.push(line);
      continue;
    }
    kept.push(line);
  }

  return { kept, discarded };
}

export const simpleUrlEngine: SourceParserEngine = {
  id: 'simple-url',
  canHandle(input) {
    return input.sourceKind === 'url';
  },
  async parse(input): Promise<NormalizedSourceDocument> {
    const title = input.title?.trim() || input.url || 'Untitled URL source';
    const { kept, discarded } = classifyLines(input.rawText || '');
    const sectionId = `section-${input.sourceId}-0`;

    return {
      id: `normalized-${input.sourceId}`,
      sourceId: input.sourceId,
      sourceKind: 'url',
      parserEngine: 'simple-url',
      parserVersion: '1',
      title,
      markdown: `# ${title}\n\n${kept.join('\n\n')}`.trim(),
      sections: [{ id: sectionId, title, order: 0 }],
      blocks: kept.map((line, index) => ({
        id: `block-${input.sourceId}-${index}`,
        type: 'paragraph',
        text: line,
        sectionId,
        anchorId: `anchor-${input.sourceId}-${index}`,
        order: index,
        metadata: {},
      })),
      anchors: kept.map((line, index) => ({
        id: `anchor-${input.sourceId}-${index}`,
        label: line.slice(0, 48),
        sectionTitle: title,
        paragraphIndex: index,
      })),
      assets: [],
      discardedBlocks: discarded.map((line, index): NormalizedBlock => ({
        id: `noise-${input.sourceId}-${index}`,
        type: 'noise',
        text: line,
        sectionId,
        anchorId: undefined,
        order: index,
        metadata: { reason: 'simple-url-noise-filter' },
      })),
      warnings: discarded.length ? ['navigation-heavy-page'] : [],
      lineage: {
        originalUrl: input.url,
        generatedAt: Date.now(),
      },
    };
  },
};
```

- [ ] **Step 5: Route snapshot creation through parser engines**

Modify `GGlearn/src/lib/sourceImport.ts` so `createSnapshotFromSource()` derives snapshots from normalized documents:

```ts
import { buildDefaultSourceParserEngines, selectSourceParserEngine } from './sourceParsers/engines';
import { simpleDocumentEngine } from './sourceParsers/simpleDocumentEngine';
import { simpleUrlEngine } from './sourceParsers/simpleUrlEngine';

const DEFAULT_SOURCE_PARSER_ENGINES = buildDefaultSourceParserEngines([
  simpleUrlEngine,
  simpleDocumentEngine,
]);
```

Inside `createSnapshotFromSource()`:

```ts
  const parserInput = {
    sourceId: source.id,
    sourceKind: source.kind === 'pasted-text' ? 'text' : source.kind === 'web' ? 'url' : source.kind,
    filePath: source.kind === 'pdf' ? source.origin : undefined,
    url: source.url,
    rawText: structuredPdfSnapshot?.rawText || normalizeMultilineText(source.fullText),
    title: source.title,
  } as const;

  const normalizedDocument = awaitOrSyncParser(parserInput);
```

Add a local helper in the same file:

```ts
function awaitOrSyncParser(input: Parameters<typeof selectSourceParserEngine>[0]) {
  const engine = selectSourceParserEngine(input, DEFAULT_SOURCE_PARSER_ENGINES);
  const parsePromise = engine.parse(input);
  throwIfPromiseRejected(parsePromise);
  return parsePromise;
}

function throwIfPromiseRejected<T>(promise: Promise<T>): T {
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
    throw new Error('Source parser engines must resolve synchronously in this phase.');
  }
  if (error) {
    throw error;
  }
  return result as T;
}
```

Then write `normalizedDocument` into the returned snapshot:

```ts
    normalizedDocument,
```

- [ ] **Step 6: Run the import tests and commit the parser-backed snapshot layer**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-develop/GGlearn
node --import tsx --test tests/sourceImport.test.ts tests/sourceParserEngines.test.ts
```

Expected: all tests PASS.

Commit:

```bash
git add GGlearn/src/lib/sourceParsers/simpleDocumentEngine.ts GGlearn/src/lib/sourceParsers/simpleUrlEngine.ts GGlearn/src/lib/sourceImport.ts GGlearn/tests/sourceImport.test.ts GGlearn/tests/sourceParserEngines.test.ts
git commit -m "Add parser-backed normalized snapshots for GGlearn sources"
```

### Task 3: Rebuild Source Asset Construction From Normalized Documents

**Files:**
- Modify: `GGlearn/src/lib/sourceTransform.ts`
- Modify: `GGlearn/src/types.ts`
- Test: `GGlearn/tests/sourceTransform.test.ts`

- [ ] **Step 1: Write failing tests that require source assets to consume normalized blocks**

Append to `GGlearn/tests/sourceTransform.test.ts`:

```ts
test('buildProjectSourceAsset carries normalized blocks into structureLayer and retrievalLayer', () => {
  const source = createProjectSourceFromDocument('project-clean', cleanPdfDocument);
  const snapshot = createSnapshotFromSource(source);
  const asset = buildProjectSourceAsset(source, snapshot);

  assert.ok(asset.structureLayer.normalizedBlocks?.length);
  assert.ok(asset.structureLayer.normalizedSections?.length);
  assert.equal(asset.structureLayer.noiseBlocks.length, snapshot.normalizedDocument?.discardedBlocks.length ?? 0);
  assert.ok(asset.retrievalLayer.retrievalUnits.every((unit) => unit.anchorRefs.length >= 1));
});

test('buildProjectSourceAsset does not promote discarded URL noise into retrieval hints', () => {
  const source = createProjectSourceFromDocument('project-url', {
    ...urlDocument,
    fullText: `${urlDocument.fullText}\n\nCurrentTime2\nReadEyes2`,
  });
  const snapshot = createSnapshotFromSource(source);
  const asset = buildProjectSourceAsset(source, snapshot);

  assert.ok(!asset.retrievalLayer.retrievalHints.includes('CurrentTime2'));
  assert.ok(!asset.projectionLayer.keyConcepts.includes('ReadEyes2'));
});
```

- [ ] **Step 2: Run the source transform tests to verify they fail**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-develop/GGlearn
node --import tsx --test tests/sourceTransform.test.ts
```

Expected: FAIL because `normalizedBlocks` fields are missing and noise still leaks into heuristic concepts.

- [ ] **Step 3: Refactor source asset building to read normalized blocks first**

Modify `GGlearn/src/lib/sourceTransform.ts`:

```ts
function buildParagraphBlocksFromNormalized(snapshot: ProjectSourceSnapshot): ParagraphBlock[] {
  const normalizedBlocks = snapshot.normalizedDocument?.blocks ?? [];
  const paragraphLikeBlocks = normalizedBlocks.filter((block) => block.type === 'paragraph' || block.type === 'quote' || block.type === 'list');

  if (!paragraphLikeBlocks.length) {
    return buildParagraphBlocks(snapshot);
  }

  return paragraphLikeBlocks.map((block, index) => ({
    id: `paragraph-${snapshot.id}-${index}`,
    sectionId: block.sectionId,
    anchorId: block.anchorId || snapshot.anchors[index]?.id || `anchor-${snapshot.id}-${index}`,
    order: index,
    text: block.text,
  }));
}
```

Replace the top of `buildProjectSourceAsset()` with:

```ts
  const normalizedDocument = snapshot.normalizedDocument;
  const paragraphs = splitIntoParagraphs(normalizedDocument?.markdown || snapshot.rawText);
  const paragraphBlocks = buildParagraphBlocksFromNormalized(snapshot);
  const discardedNoise = normalizedDocument?.discardedBlocks.map((block) => block.text) ?? [];
  const keywords = pickKeywords(normalizedDocument?.markdown || snapshot.rawText)
    .filter((keyword) => !discardedNoise.includes(keyword));
```

And in `structureLayer`:

```ts
      normalizedSections: normalizedDocument?.sections ?? [],
      normalizedBlocks: normalizedDocument?.blocks ?? [],
      embeddedAssets: normalizedDocument?.assets ?? [],
      noiseBlocks: discardedNoise,
```

- [ ] **Step 4: Demote chapterSeeds and heuristic concepts to fallback-only behavior**

Still in `buildProjectSourceAsset()`:

```ts
  const keyConcepts = dedupePreserveOrder([
    ...(normalizedDocument?.blocks
      .filter((block) => block.type === 'heading' || block.type === 'paragraph')
      .map((block) => inferTitleFromParagraph(block.text, 'Concept'))
      .filter(Boolean) ?? []),
    ...keywords,
  ])
    .filter((value) => !discardedNoise.includes(value))
    .slice(0, 8);
```

And collapse chapter seeds to compatibility:

```ts
  const chapterSeeds = buildChapterSeeds(assetId, retrievalUnits, conceptIndex).slice(0, 2);
```

Add a comment above the slice:

```ts
  // Compatibility only: chapter seeds are no longer authoritative planning inputs.
```

- [ ] **Step 5: Run transform-related tests**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-develop/GGlearn
node --import tsx --test tests/sourceTransform.test.ts tests/sourceImport.test.ts
```

Expected: PASS, especially the noise-filtering assertions.

- [ ] **Step 6: Commit the normalized asset builder changes**

Run:

```bash
git add GGlearn/src/lib/sourceTransform.ts GGlearn/src/types.ts GGlearn/tests/sourceTransform.test.ts
git commit -m "Build GGlearn source assets from normalized parser output"
```

### Task 4: Restrict AI Enrichment To Teaching Overlays And Rebuild Concept Aggregation

**Files:**
- Modify: `GGlearn/src/lib/ai/sourceAsset.ts`
- Modify: `GGlearn/src/lib/ai/projectConceptPlanning.ts`
- Test: `GGlearn/tests/projectConceptPlanning.test.ts`

- [ ] **Step 1: Write failing tests for concept aggregation on structured evidence**

Append to `GGlearn/tests/projectConceptPlanning.test.ts`:

```ts
test('buildProjectConceptIndex prefers retrieval evidence and normalized aliases over noisy projection hints', () => {
  const conceptIndex = buildProjectConceptIndex([
    createAssetWithNormalizedBlocks('asset-a', 'Jacobian Notes', ['Jacobian matrix'], [
      { id: 'unit-a', title: 'Jacobian basics', summary: 'The Jacobian matrix describes local linear change.', conceptRefs: ['Jacobian matrix'] },
    ], ['CurrentTime2']),
    createAssetWithNormalizedBlocks('asset-b', 'Jacobian Intuition', ['雅可比矩阵'], [
      { id: 'unit-b', title: '局部线性变化', summary: '雅可比矩阵近似描述多变量函数在一点附近的变化。', conceptRefs: ['雅可比矩阵'] },
    ], ['ReadEyes2']),
  ]);

  assert.equal(conceptIndex.concepts.length, 1);
  assert.ok(!conceptIndex.concepts[0].aliases.includes('CurrentTime2'));
  assert.ok(!conceptIndex.concepts[0].aliases.includes('ReadEyes2'));
});
```

- [ ] **Step 2: Run the concept planning tests to verify the new assertion fails**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-develop/GGlearn
node --import tsx --test tests/projectConceptPlanning.test.ts
```

Expected: FAIL because noisy projection hints still pollute alias collection or test helper types are missing.

- [ ] **Step 3: Prevent AI enrichment from mutating parser truth**

Modify `GGlearn/src/lib/ai/sourceAsset.ts` so enrichment no longer appends AI concepts/seeds into parser-owned truth:

```ts
      planningLayer: {
        ...asset.planningLayer,
        learningObjectives: dedupePreserveOrder([
          ...asset.planningLayer.learningObjectives,
          ...(enrichment.learningObjectives ?? []),
        ]).slice(0, 8),
        chapterSeeds: asset.planningLayer.chapterSeeds,
        exerciseSeeds: dedupePreserveOrder([
          ...asset.planningLayer.exerciseSeeds,
          ...(enrichment.exerciseSeeds ?? []),
        ]).slice(0, 6),
        diagramOpportunities: dedupePreserveOrder([
          ...asset.planningLayer.diagramOpportunities,
          ...(enrichment.diagramOpportunities ?? []),
        ]).slice(0, 6),
        coverageGaps: dedupePreserveOrder([
          ...asset.planningLayer.coverageGaps,
          ...(enrichment.coverageGaps ?? []),
        ]).slice(0, 6),
      },
      projectionLayer: {
        ...asset.projectionLayer,
        sourceGuide: enrichment.sourceGuide ?? asset.projectionLayer.sourceGuide,
        keywords: asset.projectionLayer.keywords,
        keyConcepts: asset.projectionLayer.keyConcepts,
      },
```

- [ ] **Step 4: Rebuild project concept aggregation from retrieval evidence only**

In `GGlearn/src/lib/ai/projectConceptPlanning.ts`, replace the candidate term fallback:

```ts
    const terms = unit.conceptRefs.length
      ? unit.conceptRefs
      : asset.retrievalLayer.conceptIndex.map((entry) => entry.term).filter(Boolean);
```

And filter aliases against normalized noise:

```ts
function collectNoiseTerms(asset: ProjectSourceAsset): Set<string> {
  return new Set(
    (asset.structureLayer.noiseBlocks ?? [])
      .map((value) => normalizeConceptToken(value))
      .filter(Boolean)
  );
}
```

Inside `buildProjectConceptIndex()`:

```ts
      const noiseTerms = collectNoiseTerms(asset);
      const aliases = buildAliasSet(candidate.aliases).filter((alias) => !noiseTerms.has(normalizeConceptToken(alias)));
```

- [ ] **Step 5: Run concept aggregation tests**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-develop/GGlearn
node --import tsx --test tests/projectConceptPlanning.test.ts
```

Expected: PASS with no noisy aliases in the project concept output.

- [ ] **Step 6: Commit parser-truth-safe enrichment and concept aggregation**

Run:

```bash
git add GGlearn/src/lib/ai/sourceAsset.ts GGlearn/src/lib/ai/projectConceptPlanning.ts GGlearn/tests/projectConceptPlanning.test.ts
git commit -m "Anchor GGlearn concept planning to parser-backed evidence"
```

### Task 5: Make Outline And Textbook Generation Fully Concept-First

**Files:**
- Modify: `GGlearn/src/lib/ai/outlineGeneration.ts`
- Modify: `GGlearn/src/lib/ai/textbookGeneration.ts`
- Modify: `GGlearn/src/lib/ai/textbookBody.ts`
- Test: `GGlearn/tests/outlineGeneration.test.ts`
- Test: `GGlearn/tests/textbookGeneration.test.ts`

- [ ] **Step 1: Write failing tests that reject dirty source-local seed leakage**

Append to `GGlearn/tests/outlineGeneration.test.ts`:

```ts
test('generateEnhancedOutline ignores chapter seed garbage when project concepts are clean', async () => {
  const asset = attachSingleConcept(createSnippetOnlyAsset(), {
    unitId: 'unit-jacobian',
    title: 'Jacobian basics',
    content: 'definition',
    summary: 'The Jacobian matrix describes local linear change.',
    conceptRefs: ['Jacobian matrix'],
    term: 'Jacobian matrix',
    aliases: ['Jacobian', '雅可比矩阵'],
  });

  asset.planningLayer.chapterSeeds = [
    { id: 'seed-noise-1', title: 'CurrentTime2', rationale: 'noise', retrievalUnitIds: [] },
    { id: 'seed-noise-2', title: 'https://zh', rationale: 'noise', retrievalUnitIds: [] },
  ];

  const outline = await generateEnhancedOutline(
    [asset],
    learningBrief,
    { provider: 'gemini', apiKey: '', model: 'gemini-3-flash-preview' }
  );

  assert.equal(outline.chapters.length, 1);
  assert.equal(outline.chapters[0].title.includes('CurrentTime2'), false);
});
```

Append to `GGlearn/tests/textbookGeneration.test.ts`:

```ts
test('createFallbackChapterBlueprint uses concept-bound evidence instead of chapter-title-only repetition', () => {
  const { asset } = createAsset();
  asset.retrievalLayer.retrievalUnits = [
    {
      id: 'unit-jacobian',
      assetId: asset.id,
      title: 'Jacobian matrix',
      content: 'definition',
      summary: 'The Jacobian matrix describes local linear change.',
      anchorRefs: ['anchor-jacobian'],
      conceptRefs: ['Jacobian matrix'],
      prerequisites: [],
      difficulty: 'introductory',
      teachingValue: 'core',
      citationSnippetIds: ['snippet-jacobian'],
    },
  ];

  const packs = buildEvidencePacks([asset], 'CurrentTime2', ['concept-jacobian-matrix']);
  assert.equal(packs[0].retrievalUnitId, 'unit-jacobian');
});
```

- [ ] **Step 2: Run outline/textbook tests to verify they fail**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-develop/GGlearn
node --import tsx --test tests/outlineGeneration.test.ts tests/textbookGeneration.test.ts
```

Expected: FAIL because title/seed noise still affects fallback planning.

- [ ] **Step 3: Remove chapter-seed logic from outline generation’s planning path**

Modify `GGlearn/src/lib/ai/outlineGeneration.ts`:

```ts
// Compatibility helper kept only for old tests; no longer used by generateEnhancedOutline.
export function clusterChapterSeeds(seeds: ChapterSeed[]): ChapterSeed[] {
  return seeds.filter((seed) => seed.retrievalUnitIds.length > 0);
}
```

And inside `generateEnhancedOutline()` add a guard comment and keep concept-only flow:

```ts
  // Outline planning is concept-first; source-local chapterSeeds are compatibility hints only.
  const conceptIndex = buildProjectConceptIndex(assets);
```

- [ ] **Step 4: Make textbook fallback blueprint prefer chapter concept IDs over chapter titles**

Modify `GGlearn/src/lib/ai/textbookGeneration.ts`:

```ts
function buildChapterPlanPrompt(chapterPlan?: OutlineChapter): string {
  if (!chapterPlan) {
    return '';
  }

  return [
    `[用户确认过的章节计划]`,
    chapterPlan.conceptIds.length ? `Concept IDs: ${chapterPlan.conceptIds.join(', ')}` : '',
    `Chapter title: ${chapterPlan.title}`,
    chapterPlan.learningObjectives.length ? `Learning objectives:\n${chapterPlan.learningObjectives.map((objective) => `- ${objective}`).join('\n')}` : '',
    chapterPlan.prerequisites.length ? `Prerequisites: ${chapterPlan.prerequisites.join(', ')}` : '',
    `Target chunk count: ${chapterPlan.contentPlan.targetChunkCount}`,
    `Chunk type distribution: ${JSON.stringify(chapterPlan.contentPlan.chunkTypeDistribution)}`,
    `Evidence mapping: ${JSON.stringify(chapterPlan.evidenceMapping)}`,
  ].filter(Boolean).join('\n');
}
```

Modify `GGlearn/src/lib/ai/textbookBody.ts` so fallback blueprints derive chapter identity from evidence concepts first:

```ts
  const chapterTitle = chapterTopic?.trim() || title;
  const focusConcepts = dedupePreserveOrder(
    evidencePacks.flatMap((pack) => pack.conceptRefs.length ? pack.conceptRefs : [pack.retrievalUnitTitle])
  ).slice(0, 5);
  const primaryConcept = focusConcepts[0] || chapterTitle;
```

- [ ] **Step 5: Run outline/textbook tests**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-develop/GGlearn
node --import tsx --test tests/outlineGeneration.test.ts tests/textbookGeneration.test.ts
```

Expected: PASS, including the new noise-rejection assertions.

- [ ] **Step 6: Commit the concept-first planning/generation integration**

Run:

```bash
git add GGlearn/src/lib/ai/outlineGeneration.ts GGlearn/src/lib/ai/textbookGeneration.ts GGlearn/src/lib/ai/textbookBody.ts GGlearn/tests/outlineGeneration.test.ts GGlearn/tests/textbookGeneration.test.ts
git commit -m "Make GGlearn outline and chapter generation concept-first"
```

### Task 6: Backfill Persistence And Run End-To-End Verification

**Files:**
- Modify: `GGlearn/src/lib/persistence.ts`
- Modify: `GGlearn/tests/persistence.test.ts`
- Test: `GGlearn/tests/sourceImport.test.ts`
- Test: `GGlearn/tests/sourceTransform.test.ts`
- Test: `GGlearn/tests/projectConceptPlanning.test.ts`
- Test: `GGlearn/tests/outlineGeneration.test.ts`
- Test: `GGlearn/tests/textbookGeneration.test.ts`

- [ ] **Step 1: Add failing persistence tests for normalized document backfill**

Append to `GGlearn/tests/persistence.test.ts`:

```ts
test('normalizeProject backfills normalizedDocument-compatible source snapshots', () => {
  const legacyProject = {
    id: 'project-1',
    title: 'Legacy',
    sourceSnapshots: [
      {
        id: 'snapshot-1',
        projectSourceId: 'source-1',
        rawText: 'Legacy snapshot text',
        sections: [{ id: 'section-1', title: 'Legacy', order: 0 }],
        anchors: [{ id: 'anchor-1', label: 'Legacy', sectionTitle: 'Legacy', paragraphIndex: 0 }],
        wordCount: 3,
        captureMeta: { sourceKind: 'pdf', title: 'Legacy', origin: 'legacy.pdf' },
        capturedAt: 1,
      },
    ],
  };

  const normalized = normalizeProject(legacyProject as any);

  assert.ok(normalized.sourceSnapshots[0].normalizedDocument);
  assert.equal(normalized.sourceSnapshots[0].normalizedDocument?.parserEngine, 'legacy-backfill');
});
```

- [ ] **Step 2: Run persistence tests to verify the new assertion fails**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-develop/GGlearn
node --import tsx --test tests/persistence.test.ts
```

Expected: FAIL because legacy snapshots do not yet receive `normalizedDocument`.

- [ ] **Step 3: Backfill normalized documents during project normalization**

Modify `GGlearn/src/lib/persistence.ts` where legacy snapshots/assets are normalized:

```ts
function buildLegacyNormalizedDocument(snapshot: ProjectSourceSnapshot): ProjectSourceSnapshot['normalizedDocument'] {
  return {
    id: `normalized-${snapshot.id}`,
    sourceId: snapshot.projectSourceId,
    sourceKind: snapshot.captureMeta.sourceKind === 'pasted-text' ? 'text' : snapshot.captureMeta.sourceKind === 'web' ? 'url' : snapshot.captureMeta.sourceKind,
    parserEngine: 'legacy-backfill',
    parserVersion: '1',
    title: snapshot.captureMeta.title,
    markdown: snapshot.rawText,
    sections: snapshot.sections.map((section) => ({
      id: section.id,
      title: section.title,
      order: section.order,
    })),
    blocks: snapshot.rawText
      .split(/\n{2,}/)
      .map((text, index) => text.trim())
      .filter(Boolean)
      .map((text, index) => ({
        id: `block-${snapshot.id}-${index}`,
        type: 'paragraph',
        text,
        sectionId: snapshot.sections[0]?.id || `section-${snapshot.id}-0`,
        anchorId: snapshot.anchors[index]?.id,
        order: index,
        metadata: {},
      })),
    anchors: snapshot.anchors.map((anchor) => ({ ...anchor })),
    assets: [],
    discardedBlocks: [],
    warnings: snapshot.quality?.warnings ?? [],
    lineage: {
      generatedAt: snapshot.capturedAt,
    },
  };
}
```

Use it during snapshot normalization:

```ts
      normalizedDocument: snapshot.normalizedDocument ?? buildLegacyNormalizedDocument(snapshot),
```

- [ ] **Step 4: Run the full verification set**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-develop/GGlearn
node --import tsx --test \
  tests/sourceParserEngines.test.ts \
  tests/sourceImport.test.ts \
  tests/sourceTransform.test.ts \
  tests/projectConceptPlanning.test.ts \
  tests/outlineGeneration.test.ts \
  tests/textbookGeneration.test.ts \
  tests/persistence.test.ts
```

Expected:

```text
# all listed suites pass
# no noise-token assertions fail
# no normalizedDocument backfill assertion fails
```

- [ ] **Step 5: Run a focused smoke command against the current generation path**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-develop/GGlearn
node --import tsx -e "import { createProjectSourceFromDocument, createSnapshotFromSource } from './src/lib/sourceImport.ts'; import { buildProjectSourceAsset } from './src/lib/sourceTransform.ts'; import { cleanPdfDocument } from './tests/fixtures/sourceFixtures.ts'; const source=createProjectSourceFromDocument('smoke-project', cleanPdfDocument); const snapshot=createSnapshotFromSource(source); const asset=buildProjectSourceAsset(source,snapshot); console.log(JSON.stringify({normalized:Boolean(snapshot.normalizedDocument), parser:snapshot.normalizedDocument?.parserEngine, discarded:snapshot.normalizedDocument?.discardedBlocks.length ?? 0, retrievalUnits:asset.retrievalLayer.retrievalUnits.length, noiseBlocks:asset.structureLayer.noiseBlocks.length}, null, 2));"
```

Expected output shape:

```json
{
  "normalized": true,
  "parser": "simple-document",
  "discarded": 0,
  "retrievalUnits": 1,
  "noiseBlocks": 0
}
```

- [ ] **Step 6: Commit the persistence backfill and verification pass**

Run:

```bash
git add GGlearn/src/lib/persistence.ts GGlearn/tests/persistence.test.ts
git commit -m "Backfill normalized source documents for legacy GGlearn data"
```

---

## Self-Review

### Spec coverage

- `NormalizedSourceDocument` truth layer: covered by Task 1 and Task 2.
- `DocumentEngine / UrlEngine` split: covered by Task 1 and Task 2.
- `sourceTransform.ts` stops acting as source-truth generator: covered by Task 3.
- AI enrichment narrowed to teaching overlays: covered by Task 4.
- concept-first outline + textbook generation: covered by Task 5.
- persistence/backfill for legacy projects: covered by Task 6.

No spec section is currently unassigned.

### Placeholder scan

- No `TODO` / `TBD` placeholders in tasks.
- Every code-changing step includes concrete file paths and code blocks.
- Every verification step has an exact command and expected result.

### Type consistency

- `NormalizedSourceDocument` / `SourceParserEngine` types are introduced before any downstream task references them.
- `normalizedDocument` is added to `ProjectSourceSnapshot` before source import / persistence tasks use it.
- `normalizedSections`, `normalizedBlocks`, and `embeddedAssets` are added to asset structure before source asset tasks reference them.

No unresolved name drift remains in this plan.

---

Plan complete and saved to `docs/superpowers/plans/2026-04-22-gglearn-source-parser-first-implementation-plan.md`. Two execution options:

1. Subagent-Driven (recommended) - I dispatch a fresh subagent per task, review between tasks, fast iteration
2. Inline Execution - Execute tasks in this session using executing-plans, batch execution with checkpoints

Which approach?
