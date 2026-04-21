# GGlearn PDF Internal Evidence Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在不改前台阅读体验的前提下，把 PDF 中的图、图注、公式和页码锚点接入 GGlearn 的 source asset 主链路，强化内部证据绑定与重生成稳定性。

**Architecture:** 新增一个窄的 `pdfParseAdapter` 负责把解析器产物标准化为 GGlearn 自己的 snapshot 结构；`sourceTransform` 继续从 snapshot 构建 asset，但不再丢弃公式、图注和图像引用。证据增强只作用于内部 `EvidencePack`、`ChunkSourceReference` 和生成上下文，阅读页不新增任何来源展示。

**Tech Stack:** TypeScript, React 19, Node test runner, existing `sourceImport` / `sourceTransform` / `textbookGeneration` pipeline, local-first persistence.

---

## File Map

- Create: `GGlearn/src/lib/pdfParseAdapter.ts`
  - 负责把 PDF 解析器产物中的图、图注、公式转换成 snapshot-ready 结构，并绑定到已有 anchors。
- Modify: `GGlearn/src/types.ts`
  - 新增 parser artifact、snapshot artifact、asset figure ref、evidence page metadata 类型。
- Modify: `GGlearn/src/lib/sourceImport.ts`
  - 让 `createSnapshotFromSource()` 调用 `pdfParseAdapter`，把结构化 artifact 真正写进 `ProjectSourceSnapshot`。
- Modify: `GGlearn/src/lib/sourceTransform.ts`
  - 从 snapshot 中构建 `formulaBlocks`、`figureCaptionBlocks`、`figureRefs`，并把公式/图注纳入内部 evidence 层与上下文文本。
- Modify: `GGlearn/src/lib/ai/textbookGeneration.ts`
  - 让 `EvidencePack` 和 `ChunkSourceReference` 带上内部 page metadata，继续只服务内部证据约束。
- Modify: `GGlearn/src/lib/persistence.ts`
  - 为新增结构字段提供 legacy fallback 默认值，避免历史数据断裂。
- Modify: `GGlearn/tests/fixtures/sourceFixtures.ts`
  - 增加一个带图、图注、公式的 PDF fixture。
- Modify: `GGlearn/tests/sourceImport.test.ts`
  - 锁定 snapshot 层的 PDF artifact 映射行为。
- Modify: `GGlearn/tests/sourceTransform.test.ts`
  - 锁定 asset 层对公式、图注和 figure refs 的填充行为。
- Modify: `GGlearn/tests/textbookGeneration.test.ts`
  - 锁定 evidence pack 与 section context 对公式/图注的内部消费行为。
- Modify: `GGlearn/tests/persistence.test.ts`
  - 锁定新增字段对 legacy migration 的兼容。

## Task 1: Standardize Parsed PDF Artifacts Into Snapshots

**Files:**
- Create: `GGlearn/src/lib/pdfParseAdapter.ts`
- Modify: `GGlearn/src/types.ts`
- Modify: `GGlearn/src/lib/sourceImport.ts`
- Modify: `GGlearn/tests/fixtures/sourceFixtures.ts`
- Test: `GGlearn/tests/sourceImport.test.ts`

- [ ] **Step 1: Write the failing test and fixture**

```ts
// GGlearn/tests/fixtures/sourceFixtures.ts
export const annotatedPdfDocument: SourceDocument = {
  id: 'fixture-annotated-pdf',
  type: 'pdf',
  title: 'Eigenvector Geometry Notes',
  snippet: 'Vectors, eigenlines, and the core eigenvalue equation.',
  fullText: [
    'Chapter 1 Eigenvector Geometry',
    'Eigenvectors keep their direction under a linear transformation.',
    'Figure 1. Eigenvector geometry in the plane.',
    'Av = λv',
  ].join('\n'),
  selected: true,
  structureHint: {
    extraction: 'structured',
    warnings: [],
    pages: [
      {
        pageNumber: 1,
        lines: [
          { id: 'annotated-line-1', text: 'Chapter 1 Eigenvector Geometry', pageNumber: 1, lineIndex: 0, fontSize: 18, y: 780 },
          { id: 'annotated-line-2', text: 'Eigenvectors keep their direction under a linear transformation.', pageNumber: 1, lineIndex: 1, fontSize: 11, y: 748 },
          { id: 'annotated-line-3', text: 'Figure 1. Eigenvector geometry in the plane.', pageNumber: 1, lineIndex: 2, fontSize: 11, y: 720 },
          { id: 'annotated-line-4', text: 'Av = λv', pageNumber: 1, lineIndex: 3, fontSize: 12, y: 694 },
        ],
      },
    ],
    figureAssets: [
      {
        id: 'figure-1',
        pageNumber: 1,
        imagePath: 'fixtures/eigenvector-geometry.png',
        captionText: 'Figure 1. Eigenvector geometry in the plane.',
        label: 'Figure 1',
      },
    ],
    captionSpans: [
      {
        id: 'caption-1',
        pageNumber: 1,
        text: 'Figure 1. Eigenvector geometry in the plane.',
        kind: 'figure',
        label: 'Figure 1',
      },
    ],
    formulaSpans: [
      {
        id: 'formula-1',
        pageNumber: 1,
        rawText: 'Av = λv',
        latex: 'A v = \\lambda v',
      },
    ],
  },
};

// GGlearn/tests/sourceImport.test.ts
test('createSnapshotFromSource preserves parsed figures, captions, and formulas for PDFs', () => {
  const source = createProjectSourceFromDocument('project-annotated', annotatedPdfDocument);
  const snapshot = createSnapshotFromSource(source);

  assert.equal(snapshot.figureAssets.length, 1);
  assert.equal(snapshot.figureAssets[0].pageNumber, 1);
  assert.equal(snapshot.figureAssets[0].captionText, 'Figure 1. Eigenvector geometry in the plane.');
  assert.equal(snapshot.figureAssets[0].anchorId, snapshot.anchors[0]?.id);

  assert.equal(snapshot.captionSpans.length, 1);
  assert.equal(snapshot.captionSpans[0].kind, 'figure');
  assert.equal(snapshot.captionSpans[0].anchorId, snapshot.anchors[0]?.id);

  assert.equal(snapshot.formulaSpans.length, 1);
  assert.equal(snapshot.formulaSpans[0].latex, 'A v = \\lambda v');
  assert.equal(snapshot.formulaSpans[0].anchorId, snapshot.anchors[0]?.id);
});
```

- [ ] **Step 2: Run test to verify it fails**

Run:

```bash
cd GGlearn && node --import tsx --test tests/sourceImport.test.ts
```

Expected: FAIL with TypeScript property errors such as `Property 'figureAssets' does not exist` or runtime assertions showing empty artifact arrays.

- [ ] **Step 3: Write minimal implementation**

```ts
// GGlearn/src/types.ts
export interface ParsedFigureAsset {
  id: string;
  pageNumber: number;
  imagePath: string;
  captionText?: string;
  label?: string;
}

export interface ParsedCaptionSpan {
  id: string;
  pageNumber: number;
  text: string;
  kind: 'figure' | 'table';
  label?: string;
}

export interface ParsedFormulaSpan {
  id: string;
  pageNumber: number;
  rawText: string;
  latex?: string;
}

export interface SnapshotFigureAsset extends ParsedFigureAsset {
  anchorId: string;
  sectionTitle?: string;
}

export interface SnapshotCaptionSpan extends ParsedCaptionSpan {
  anchorId: string;
  sectionTitle?: string;
}

export interface SnapshotFormulaSpan extends ParsedFormulaSpan {
  anchorId: string;
  sectionTitle?: string;
}

export interface SourceStructureHint {
  extraction: 'structured' | 'fallback';
  pages: SourcePage[];
  warnings: string[];
  figureAssets?: ParsedFigureAsset[];
  captionSpans?: ParsedCaptionSpan[];
  formulaSpans?: ParsedFormulaSpan[];
}

export interface ProjectSourceSnapshot {
  id: string;
  projectSourceId: string;
  rawText: string;
  sections: SourceSection[];
  anchors: SourceAnchor[];
  figureAssets: SnapshotFigureAsset[];
  captionSpans: SnapshotCaptionSpan[];
  formulaSpans: SnapshotFormulaSpan[];
  pageCount?: number;
  wordCount: number;
  captureMeta: {
    sourceKind: ProjectSource['kind'];
    title: string;
    origin: string;
  };
  quality?: {
    extraction: 'structured' | 'fallback';
    warnings: string[];
    headingCount: number;
    anchorCount: number;
  };
  capturedAt: number;
}
```

```ts
// GGlearn/src/lib/pdfParseAdapter.ts
import type {
  ParsedCaptionSpan,
  ParsedFigureAsset,
  ParsedFormulaSpan,
  SnapshotCaptionSpan,
  SnapshotFigureAsset,
  SnapshotFormulaSpan,
  SourceAnchor,
} from '../types';

function findAnchorForPage(anchors: SourceAnchor[], pageNumber: number): SourceAnchor | undefined {
  return anchors.find((anchor) => anchor.pageNumber === pageNumber) ?? anchors[0];
}

export function mapFigureAssets(
  figureAssets: ParsedFigureAsset[] | undefined,
  anchors: SourceAnchor[]
): SnapshotFigureAsset[] {
  return (figureAssets ?? []).flatMap((figure) => {
    const anchor = findAnchorForPage(anchors, figure.pageNumber);
    if (!anchor) return [];

    return [{
      ...figure,
      anchorId: anchor.id,
      sectionTitle: anchor.sectionTitle,
    }];
  });
}

export function mapCaptionSpans(
  captionSpans: ParsedCaptionSpan[] | undefined,
  anchors: SourceAnchor[]
): SnapshotCaptionSpan[] {
  return (captionSpans ?? []).flatMap((caption) => {
    const anchor = findAnchorForPage(anchors, caption.pageNumber);
    if (!anchor) return [];

    return [{
      ...caption,
      anchorId: anchor.id,
      sectionTitle: anchor.sectionTitle,
    }];
  });
}

export function mapFormulaSpans(
  formulaSpans: ParsedFormulaSpan[] | undefined,
  anchors: SourceAnchor[]
): SnapshotFormulaSpan[] {
  return (formulaSpans ?? []).flatMap((formula) => {
    const anchor = findAnchorForPage(anchors, formula.pageNumber);
    if (!anchor) return [];

    return [{
      ...formula,
      anchorId: anchor.id,
      sectionTitle: anchor.sectionTitle,
    }];
  });
}
```

```ts
// GGlearn/src/lib/sourceImport.ts
import { mapCaptionSpans, mapFigureAssets, mapFormulaSpans } from './pdfParseAdapter';

export function createSnapshotFromSource(source: ProjectSource): ProjectSourceSnapshot {
  const structuredPdfSnapshot = buildStructuredPdfSnapshot(source);
  const prefix = source.id;
  const rawText = structuredPdfSnapshot?.rawText || normalizeMultilineText(source.fullText);
  const sections = structuredPdfSnapshot?.sections || createSourceSections(source.title, prefix);
  const anchors = structuredPdfSnapshot?.anchors || createSourceAnchors(rawText, source.title, prefix, source.url);

  return {
    id: `snapshot-${source.id}`,
    projectSourceId: source.id,
    rawText,
    sections,
    anchors,
    figureAssets: mapFigureAssets(source.structureHint?.figureAssets, anchors),
    captionSpans: mapCaptionSpans(source.structureHint?.captionSpans, anchors),
    formulaSpans: mapFormulaSpans(source.structureHint?.formulaSpans, anchors),
    pageCount: structuredPdfSnapshot?.pageCount ?? (source.kind === 'pdf' ? source.structureHint?.pages.length : undefined),
    wordCount,
    captureMeta: {
      sourceKind: source.kind,
      title: source.title,
      origin: source.origin,
    },
    quality: {
      extraction: qualityExtraction,
      warnings: qualityWarnings,
      headingCount: structuredPdfSnapshot?.headingCount ?? 0,
      anchorCount: anchors.length,
    },
    capturedAt: Date.now(),
  };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run:

```bash
cd GGlearn && node --import tsx --test tests/sourceImport.test.ts
```

Expected: PASS with all `sourceImport` tests green, including the new artifact-preservation case.

- [ ] **Step 5: Commit**

```bash
git add GGlearn/src/types.ts GGlearn/src/lib/pdfParseAdapter.ts GGlearn/src/lib/sourceImport.ts GGlearn/tests/fixtures/sourceFixtures.ts GGlearn/tests/sourceImport.test.ts
git commit -m "Preserve parsed PDF artifacts inside snapshots" -m $'Constraint: Reading UI must remain source-silent\nRejected: Store parsed figures only in parser blobs | source snapshots would stay blind to non-text evidence\nConfidence: high\nScope-risk: narrow\nDirective: Keep parser normalization isolated in pdfParseAdapter instead of leaking parser-specific logic into reader code\nTested: cd GGlearn && node --import tsx --test tests/sourceImport.test.ts\nNot-tested: manual upload flow through App.tsx'
```

### Task 2: Populate Asset Structure Blocks From Snapshot Artifacts

**Files:**
- Modify: `GGlearn/src/types.ts`
- Modify: `GGlearn/src/lib/sourceTransform.ts`
- Test: `GGlearn/tests/sourceTransform.test.ts`

- [ ] **Step 1: Write the failing asset test**

```ts
// GGlearn/tests/sourceTransform.test.ts
import { annotatedPdfDocument } from './fixtures/sourceFixtures';

test('buildProjectSourceAsset lifts formulas, captions, and figure refs into the structure layer', () => {
  const source = createProjectSourceFromDocument('project-annotated', annotatedPdfDocument);
  const snapshot = createSnapshotFromSource(source);
  const asset = buildProjectSourceAsset(source, snapshot);

  assert.equal(asset.structureLayer.formulaBlocks.length, 1);
  assert.equal(asset.structureLayer.formulaBlocks[0].content, 'Av = λv');

  assert.equal(asset.structureLayer.figureCaptionBlocks.length, 1);
  assert.match(asset.structureLayer.figureCaptionBlocks[0].content, /Eigenvector geometry/);

  assert.equal(asset.structureLayer.figureRefs.length, 1);
  assert.equal(asset.structureLayer.figureRefs[0].assetRef, 'fixtures/eigenvector-geometry.png');
  assert.equal(asset.structureLayer.figureRefs[0].captionBlockId, asset.structureLayer.figureCaptionBlocks[0].id);

  assert.ok(asset.retrievalLayer.evidenceSnippets.some((snippet) => snippet.quoteKind === 'formula'));
  assert.ok(
    asset.retrievalLayer.evidenceSnippets.some((snippet) =>
      snippet.text.includes('Figure 1. Eigenvector geometry in the plane.')
    )
  );
});
```

- [ ] **Step 2: Run test to verify it fails**

Run:

```bash
cd GGlearn && node --import tsx --test tests/sourceTransform.test.ts
```

Expected: FAIL because `figureRefs` is missing and `formulaBlocks` / `figureCaptionBlocks` are still empty arrays.

- [ ] **Step 3: Write minimal implementation**

```ts
// GGlearn/src/types.ts
export interface FigureRef {
  id: string;
  anchorId: string;
  pageNumber?: number;
  captionBlockId?: string;
  assetRef: string;
  label?: string;
}

export interface ProjectSourceAsset {
  id: string;
  projectSourceId: string;
  snapshotId: string;
  status: 'raw' | 'structured' | 'ready' | 'failed';
  createdAt: number;
  updatedAt: number;
  snapshotLayer: {
    snapshotId: string;
    sourceKind: ProjectSource['kind'];
    canonicalTitle: string;
    sourceMeta: ProjectSourceSnapshot['captureMeta'];
    anchorStrategy: 'paragraph' | 'section' | 'mixed';
    parseStatus: 'parsed' | 'fallback';
  };
  structureLayer: {
    sectionTree: SourceSection[];
    paragraphBlocks: ParagraphBlock[];
    formulaBlocks: FormulaBlock[];
    figureCaptionBlocks: FigureCaptionBlock[];
    figureRefs: FigureRef[];
    noiseBlocks: string[];
    normalizationNotes: string[];
  };
  retrievalLayer: {
    retrievalUnits: RetrievalUnit[];
    conceptIndex: ConceptIndexEntry[];
    evidenceSnippets: EvidenceSnippet[];
    retrievalHints: string[];
    riskFlags: string[];
  };
  planningLayer: {
    chapterSeeds: ChapterSeed[];
    learningObjectives: string[];
    exerciseSeeds: string[];
    diagramOpportunities: string[];
    difficultySignals: string[];
    coverageGaps: string[];
  };
  projectionLayer: {
    sourceGuide: string;
    keywords: string[];
    keyConcepts: string[];
    readingTimeEstimateMinutes: number;
    recommendedUse: string;
  };
}
```

```ts
// GGlearn/src/lib/sourceTransform.ts
function buildFormulaBlocks(snapshot: ProjectSourceSnapshot): FormulaBlock[] {
  return snapshot.formulaSpans.map((formula, index) => ({
    id: `formula-${snapshot.id}-${index}`,
    anchorId: formula.anchorId,
    content: formula.rawText,
  }));
}

function buildFigureCaptionBlocks(snapshot: ProjectSourceSnapshot): FigureCaptionBlock[] {
  return snapshot.captionSpans.map((caption, index) => ({
    id: `figure-caption-${snapshot.id}-${index}`,
    anchorId: caption.anchorId,
    content: caption.text,
  }));
}

function buildFigureRefs(
  snapshot: ProjectSourceSnapshot,
  figureCaptionBlocks: FigureCaptionBlock[]
): FigureRef[] {
  return snapshot.figureAssets.map((figure, index) => ({
    id: `figure-ref-${snapshot.id}-${index}`,
    anchorId: figure.anchorId,
    pageNumber: figure.pageNumber,
    captionBlockId: figureCaptionBlocks.find((caption) => caption.anchorId === figure.anchorId)?.id,
    assetRef: figure.imagePath,
    label: figure.label,
  }));
}

function buildEvidenceSnippets(
  assetId: string,
  paragraphBlocks: ParagraphBlock[],
  title: string,
  snapshot?: ProjectSourceSnapshot
): EvidenceSnippet[] {
  const anchorPageMap = new Map((snapshot?.anchors ?? []).map((anchor) => [anchor.id, anchor.pageNumber]));
  const paragraphSnippets = paragraphBlocks
    .filter((block) => classifyContentKind(block.text) === 'teaching')
    .slice(0, 16)
    .map((block, index) => ({
      id: `snippet-${assetId}-${index}`,
      assetId,
      text: block.text.slice(0, 800),
      anchorRef: block.anchorId,
      pageNumber: anchorPageMap.get(block.anchorId),
      sourceSectionTitle: title,
      confidence: block.text.length > 120 ? 'high' : 'medium',
      quoteKind: detectQuoteKind(block.text),
    }));

  const formulaSnippets = (snapshot?.formulaSpans ?? []).map((formula, index) => ({
    id: `formula-snippet-${assetId}-${index}`,
    assetId,
    text: formula.rawText,
    anchorRef: formula.anchorId,
    pageNumber: anchorPageMap.get(formula.anchorId),
    sourceSectionTitle: formula.sectionTitle,
    confidence: 'high' as const,
    quoteKind: 'formula' as const,
  }));

  const captionSnippets = (snapshot?.captionSpans ?? []).map((caption, index) => ({
    id: `caption-snippet-${assetId}-${index}`,
    assetId,
    text: caption.text,
    anchorRef: caption.anchorId,
    pageNumber: anchorPageMap.get(caption.anchorId),
    sourceSectionTitle: caption.sectionTitle,
    confidence: 'medium' as const,
    quoteKind: 'summary' as const,
  }));

  return dedupeEvidenceSnippets([...paragraphSnippets, ...formulaSnippets, ...captionSnippets]);
}

export function buildProjectSourceAsset(source: ProjectSource, snapshot: ProjectSourceSnapshot): ProjectSourceAsset {
  const paragraphs = splitIntoParagraphs(snapshot.rawText);
  const paragraphBlocks = buildParagraphBlocks(snapshot);
  const assetId = `asset-${source.id}`;
  const keywords = pickKeywords(snapshot.rawText);
  const keyConcepts = dedupePreserveOrder([...pickKeyConcepts(paragraphs), ...keywords]).slice(0, 8);
  const evidenceSnippets = buildEvidenceSnippets(assetId, paragraphBlocks, source.title, snapshot);
  const retrievalUnits = buildRetrievalUnits(assetId, paragraphBlocks, evidenceSnippets, keyConcepts);
  const conceptIndex = buildConceptIndex(keywords, retrievalUnits, evidenceSnippets);
  const chapterSeeds = buildChapterSeeds(assetId, retrievalUnits, conceptIndex);
  const riskFlags = buildRiskFlags(source, snapshot, paragraphBlocks);
  const formulaBlocks = buildFormulaBlocks(snapshot);
  const figureCaptionBlocks = buildFigureCaptionBlocks(snapshot);
  const figureRefs = buildFigureRefs(snapshot, figureCaptionBlocks);

  return {
    id: assetId,
    projectSourceId: source.id,
    snapshotId: snapshot.id,
    status: 'structured',
    createdAt: Date.now(),
    updatedAt: Date.now(),
    snapshotLayer: {
      snapshotId: snapshot.id,
      sourceKind: source.kind,
      canonicalTitle: source.title,
      sourceMeta: snapshot.captureMeta,
      anchorStrategy: hasPageAnchors && hasSectionAnchors ? 'mixed' : hasSectionAnchors ? 'section' : 'paragraph',
      parseStatus: snapshot.quality?.extraction === 'structured' || paragraphs.length > 0 ? 'parsed' : 'fallback',
    },
    structureLayer: {
      sectionTree: snapshot.sections,
      paragraphBlocks,
      formulaBlocks,
      figureCaptionBlocks,
      figureRefs,
      noiseBlocks: [],
      normalizationNotes: ['Deterministic phase-1 local source normalization.'],
    },
    retrievalLayer: {
      retrievalUnits,
      conceptIndex,
      evidenceSnippets,
      retrievalHints: dedupePreserveOrder([...keyConcepts, ...chapterSeeds.map((seed) => seed.title)]).slice(0, 10),
      riskFlags,
    },
    planningLayer: {
      chapterSeeds,
      learningObjectives: keyConcepts.slice(0, 5).map((concept) => `Understand ${concept} in context and explain how it connects to neighboring ideas.`),
      exerciseSeeds: retrievalUnits.slice(0, 3).map((unit) => `Practice ${unit.title} using cited source evidence.`),
      diagramOpportunities: keyConcepts.slice(0, 3).map((concept) => `Visualize the relationship around ${concept}.`),
      difficultySignals: retrievalUnits.map((unit) => unit.difficulty),
      coverageGaps: retrievalUnits.length === 0 ? ['No retrievable units created from source.'] : [],
    },
    projectionLayer: {
      sourceGuide: pickTeachingGuide(paragraphs, source.snippet),
      keywords,
      keyConcepts,
      readingTimeEstimateMinutes: Math.max(1, Math.ceil(snapshot.wordCount / 180)),
      recommendedUse: retrievalUnits.length > 6 ? 'Use as a primary source for chapter generation.' : 'Use as a supporting source.',
    },
  };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run:

```bash
cd GGlearn && node --import tsx --test tests/sourceTransform.test.ts
```

Expected: PASS with the new structure-layer assertions green and no regressions in existing source transform tests.

- [ ] **Step 5: Commit**

```bash
git add GGlearn/src/types.ts GGlearn/src/lib/sourceTransform.ts GGlearn/tests/sourceTransform.test.ts
git commit -m "Lift formulas and captions into source assets" -m $'Constraint: Source assets remain the only generation boundary\nRejected: Keep formulas and captions only in snapshots | retrieval and evidence layers would stay text-only in practice\nConfidence: high\nScope-risk: narrow\nDirective: Do not surface structure-layer artifacts in reader UI; they are for internal control only\nTested: cd GGlearn && node --import tsx --test tests/sourceTransform.test.ts\nNot-tested: manual regeneration on a live imported PDF'
```

### Task 3: Propagate Internal Evidence Metadata Through Generation And Persistence

**Files:**
- Modify: `GGlearn/src/types.ts`
- Modify: `GGlearn/src/lib/sourceTransform.ts`
- Modify: `GGlearn/src/lib/ai/textbookGeneration.ts`
- Modify: `GGlearn/src/lib/persistence.ts`
- Test: `GGlearn/tests/textbookGeneration.test.ts`
- Test: `GGlearn/tests/persistence.test.ts`

- [ ] **Step 1: Write the failing generation and persistence tests**

```ts
// GGlearn/tests/textbookGeneration.test.ts
test('buildEvidencePacks carries internal page metadata for formula and caption evidence', () => {
  const source = createProjectSourceFromDocument('project-annotated', annotatedPdfDocument);
  const snapshot = createSnapshotFromSource(source);
  const asset = buildProjectSourceAsset(source, snapshot);
  const packs = buildEvidencePacks([asset], 'Eigenvectors');

  const formulaPack = packs.find((pack) => pack.quoteKind === 'formula');
  assert.ok(formulaPack);
  assert.equal(formulaPack?.pageNumber, 1);
  assert.equal(formulaPack?.evidenceKind, 'formula');

  const captionPack = packs.find((pack) => pack.excerpt.includes('Figure 1. Eigenvector geometry in the plane.'));
  assert.ok(captionPack);
  assert.equal(captionPack?.pageNumber, 1);
  assert.equal(captionPack?.evidenceKind, 'figure-caption');
});

test('buildSectionContextFromAssets includes formulas and figure captions for internal prompting only', () => {
  const source = createProjectSourceFromDocument('project-annotated', annotatedPdfDocument);
  const snapshot = createSnapshotFromSource(source);
  const asset = buildProjectSourceAsset(source, snapshot);
  const context = buildSectionContextFromAssets([asset]);

  assert.match(context, /Formula Blocks:/);
  assert.match(context, /Av = λv/);
  assert.match(context, /Figure Captions:/);
  assert.match(context, /Figure 1. Eigenvector geometry in the plane./);
});
```

```ts
// GGlearn/tests/persistence.test.ts
assert.deepEqual(projects[0].sourceAssets[0].structureLayer.figureRefs, []);
```

- [ ] **Step 2: Run tests to verify they fail**

Run:

```bash
cd GGlearn && node --import tsx --test tests/textbookGeneration.test.ts tests/persistence.test.ts
```

Expected: FAIL because `pageNumber` / `evidenceKind` do not exist on `EvidencePack`, section context omits formula/caption lines, and legacy assets do not seed `figureRefs`.

- [ ] **Step 3: Write minimal implementation**

```ts
// GGlearn/src/types.ts
export interface ChunkSourceReference {
  sourceId: string;
  sourceTitle: string;
  assetId?: string;
  evidenceId?: string;
  retrievalUnitId?: string;
  snippetId?: string;
  anchorRef?: string;
  sourceSectionTitle?: string;
  pageNumber?: number;
  confidence?: EvidenceSnippet['confidence'];
  quoteKind?: EvidenceSnippet['quoteKind'];
  excerpt?: string;
  sourceGuide?: string;
}

export interface EvidenceSnippet {
  id: string;
  assetId: string;
  text: string;
  anchorRef: string;
  pageNumber?: number;
  sourceSectionTitle?: string;
  confidence: 'low' | 'medium' | 'high';
  quoteKind: 'definition' | 'argument' | 'example' | 'formula' | 'summary';
}

export interface EvidencePack {
  id: string;
  assetId: string;
  sourceId: string;
  sourceTitle: string;
  retrievalUnitId: string;
  retrievalUnitTitle: string;
  snippetId?: string;
  anchorRef?: string;
  sourceSectionTitle?: string;
  pageNumber?: number;
  evidenceKind: 'paragraph' | 'formula' | 'figure-caption';
  excerpt: string;
  summary: string;
  conceptRefs: string[];
  teachingValue: RetrievalUnit['teachingValue'];
  confidence: EvidenceSnippet['confidence'];
  quoteKind?: EvidenceSnippet['quoteKind'];
  sourceGuide?: string;
}
```

```ts
// GGlearn/src/lib/ai/textbookGeneration.ts
function createChunkSourceReference(pack: EvidencePack): ChunkSourceReference {
  return {
    sourceId: pack.sourceId,
    sourceTitle: pack.sourceTitle,
    assetId: pack.assetId,
    evidenceId: pack.id,
    retrievalUnitId: pack.retrievalUnitId,
    snippetId: pack.snippetId,
    anchorRef: pack.anchorRef,
    sourceSectionTitle: pack.sourceSectionTitle,
    pageNumber: pack.pageNumber,
    confidence: pack.confidence,
    quoteKind: pack.quoteKind,
    excerpt: pack.excerpt,
    sourceGuide: pack.sourceGuide,
  };
}

function buildEvidencePackFromAsset(asset: ProjectSourceAsset, chapterTopic?: string): EvidencePack[] {
  const snippetMap = new Map(asset.retrievalLayer.evidenceSnippets.map((snippet) => [snippet.id, snippet]));
  const captionAnchorIds = new Set(asset.structureLayer.figureCaptionBlocks.map((block) => block.anchorId));

  return asset.retrievalLayer.retrievalUnits.map((unit, index) => {
    const snippet = unit.citationSnippetIds
      .map((snippetId) => snippetMap.get(snippetId))
      .find((candidate): candidate is EvidenceSnippet => Boolean(candidate))
      ?? asset.retrievalLayer.evidenceSnippets[index]
      ?? asset.retrievalLayer.evidenceSnippets[0];

    const evidenceKind: EvidencePack['evidenceKind'] =
      snippet?.quoteKind === 'formula'
        ? 'formula'
        : snippet?.anchorRef && captionAnchorIds.has(snippet.anchorRef)
          ? 'figure-caption'
          : 'paragraph';

    return {
      id: `pack-${asset.id}-${unit.id}`,
      assetId: asset.id,
      sourceId: asset.projectSourceId,
      sourceTitle: asset.snapshotLayer.canonicalTitle,
      retrievalUnitId: unit.id,
      retrievalUnitTitle: unit.title,
      snippetId: snippet?.id,
      anchorRef: snippet?.anchorRef,
      sourceSectionTitle: snippet?.sourceSectionTitle,
      pageNumber: snippet?.pageNumber,
      evidenceKind,
      excerpt: snippet?.text ?? unit.content.slice(0, 240),
      summary: unit.summary,
      conceptRefs: unit.conceptRefs,
      teachingValue: unit.teachingValue,
      confidence: snippet?.confidence ?? 'medium',
      quoteKind: snippet?.quoteKind,
      sourceGuide: asset.projectionLayer.sourceGuide,
    };
  });
}
```

```ts
// GGlearn/src/lib/sourceTransform.ts
function buildAssetContext(assets: ProjectSourceAsset[], mode: 'full' | 'section'): string {
  return assets
    .map((asset) => {
      const teachingUnits = asset.retrievalLayer.retrievalUnits
        .filter((unit) => unit.teachingValue !== 'reference' || classifyContentKind(unit.content) === 'teaching')
        .slice(0, 6);
      const evidenceSnippets = asset.retrievalLayer.evidenceSnippets.slice(0, 4);
      const formulaBlocks = asset.structureLayer.formulaBlocks.slice(0, 3);
      const figureCaptionBlocks = asset.structureLayer.figureCaptionBlocks.slice(0, 3);

      const lines: string[] = [`# Source Asset: ${asset.snapshotLayer.canonicalTitle}`];

      if (asset.projectionLayer.sourceGuide) {
        lines.push(`Guide: ${asset.projectionLayer.sourceGuide}`);
      }

      if (mode === 'full') {
        if (asset.projectionLayer.keywords.length) {
          lines.push(`Keywords: ${asset.projectionLayer.keywords.join(', ')}`);
        }
        if (asset.projectionLayer.keyConcepts.length) {
          lines.push(`Key Concepts: ${asset.projectionLayer.keyConcepts.join(', ')}`);
        }
      }

      if (teachingUnits.length) {
        lines.push(`Retrieval Units:\n${teachingUnits.map((unit) => `- ${unit.title}: ${unit.summary}`).join('\n')}`);
      }
      if (formulaBlocks.length) {
        lines.push(`Formula Blocks:\n${formulaBlocks.map((formula) => `- ${formula.content}`).join('\n')}`);
      }
      if (figureCaptionBlocks.length) {
        lines.push(`Figure Captions:\n${figureCaptionBlocks.map((caption) => `- ${caption.content}`).join('\n')}`);
      }
      if (evidenceSnippets.length) {
        lines.push(`Evidence Snippets:\n${evidenceSnippets.map((snippet) => `- (${snippet.quoteKind}) ${snippet.text}`).join('\n')}`);
      }

      return lines.filter(Boolean).join('\n');
    })
    .join('\n\n');
}
```

```ts
// GGlearn/src/lib/persistence.ts
structureLayer: {
  sectionTree: [{ id: `legacy-section-${projectId}`, title, order: 0 }],
  paragraphBlocks: [],
  formulaBlocks: [],
  figureCaptionBlocks: [],
  figureRefs: [],
  noiseBlocks: [],
  normalizationNotes: ['Migrated from pre-project textbook record.'],
},
```

- [ ] **Step 4: Run tests to verify they pass**

Run:

```bash
cd GGlearn && node --import tsx --test tests/textbookGeneration.test.ts tests/persistence.test.ts
cd GGlearn && npm run lint
```

Expected:

- `textbookGeneration` tests PASS
- `persistence` tests PASS
- `npm run lint` exits with code `0`

- [ ] **Step 5: Commit**

```bash
git add GGlearn/src/types.ts GGlearn/src/lib/sourceTransform.ts GGlearn/src/lib/ai/textbookGeneration.ts GGlearn/src/lib/persistence.ts GGlearn/tests/textbookGeneration.test.ts GGlearn/tests/persistence.test.ts
git commit -m "Propagate internal PDF evidence through generation" -m $'Constraint: Internal evidence metadata must not leak into learner-facing UI\nRejected: Add a reader-side evidence panel now | violates the agreed product boundary\nConfidence: medium\nScope-risk: moderate\nDirective: Keep pageNumber and evidenceKind internal; do not wire them into ReaderView without a separate product decision\nTested: cd GGlearn && node --import tsx --test tests/textbookGeneration.test.ts tests/persistence.test.ts && npm run lint\nNot-tested: full build plus manual end-to-end generation on a real PDF'
```

## Final Verification

- [ ] Run the focused regression sweep:

```bash
cd GGlearn && node --import tsx --test tests/sourceImport.test.ts tests/sourceTransform.test.ts tests/textbookGeneration.test.ts tests/persistence.test.ts
```

Expected: PASS

- [ ] Run the broad repo checks:

```bash
cd GGlearn && npm test
cd GGlearn && npm run lint
cd GGlearn && npm run build
```

Expected:

- `npm test` PASS
- `npm run lint` exits with code `0`
- `npm run build` completes successfully

- [ ] Manually verify the product boundary after local run

```text
1. 导入一个带图、图注、公式的 PDF。
2. 确认项目生成后没有在 ReaderView 中出现来源页码、证据编号或“查看原文”入口。
3. 确认内部调试日志或断点中，snapshot 与 asset 已含 figure/caption/formula 数据。
4. 重生成同一章节时，观察 chunk 的 sourceRefs 仍然存在且无空引用。
```

## Plan Self-Review Summary

- Spec coverage: 已覆盖 `pdfParseAdapter`、snapshot artifact、asset structure、内部 evidence metadata、persistence fallback、前台不展示来源。
- Placeholder scan: 本计划未保留 `TODO` / `TBD` / “后续补充” 占位。
- Type consistency: 统一使用 `figureAssets / captionSpans / formulaSpans / figureRefs / evidenceKind / pageNumber`，避免同义字段并存。
