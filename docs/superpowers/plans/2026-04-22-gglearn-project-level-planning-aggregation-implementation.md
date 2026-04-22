# GGlearn Project-Level Planning Aggregation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace source-local `chapterSeeds` as the primary outline input with a runtime-only project-level planning aggregation layer that clusters project concepts before outline generation.

**Architecture:** Keep `ProjectSourceAsset` as the upstream truth, add a new runtime `ProjectConceptIndex` builder, refactor outline generation to consume concept clusters instead of flattened chapter seeds, and switch chapter evidence selection to prefer `conceptIds` over title matching. The first implementation is intentionally narrow: it changes outline inputs and chapter evidence selection without rebuilding the chapter blueprint or section generation pipelines.

**Tech Stack:** TypeScript, Node test runner, existing GGlearn source-asset pipeline, existing outline/textbook generation modules

---

### Task 1: Lock The New Planning Boundary With Failing Tests

**Files:**
- Create: `GGlearn/tests/projectConceptPlanning.test.ts`
- Modify: `GGlearn/tests/outlineGeneration.test.ts`
- Modify: `GGlearn/tests/textbookGeneration.test.ts`
- Modify: `GGlearn/tests/persistence.test.ts`

- [ ] **Step 1: Write the failing project-level concept clustering tests**

Create `GGlearn/tests/projectConceptPlanning.test.ts` with these tests:

```ts
import test from 'node:test';
import assert from 'node:assert/strict';

import type { ProjectSourceAsset } from '../src/types';
import { buildProjectConceptIndex } from '../src/lib/ai/projectConceptPlanning';

function createAsset(id: string, title: string, keyConcepts: string[], retrievalUnits: Array<{ id: string; title: string; summary: string; conceptRefs: string[] }>): ProjectSourceAsset {
  return {
    id,
    projectSourceId: `source-${id}`,
    snapshotId: `snapshot-${id}`,
    status: 'ready',
    createdAt: 1,
    updatedAt: 1,
    snapshotLayer: {
      snapshotId: `snapshot-${id}`,
      sourceKind: 'pasted-text',
      canonicalTitle: title,
      sourceMeta: {
        sourceKind: 'pasted-text',
        title,
        origin: 'test',
      },
      anchorStrategy: 'paragraph',
      parseStatus: 'parsed',
    },
    structureLayer: {
      sectionTree: [],
      paragraphBlocks: [],
      formulaBlocks: [],
      figureCaptionBlocks: [],
      noiseBlocks: [],
      normalizationNotes: [],
    },
    retrievalLayer: {
      retrievalUnits: retrievalUnits.map((unit, index) => ({
        id: unit.id,
        assetId: id,
        title: unit.title,
        content: unit.summary,
        summary: unit.summary,
        anchorRefs: [`anchor-${id}-${index}`],
        conceptRefs: unit.conceptRefs,
        prerequisites: [],
        difficulty: 'introductory',
        teachingValue: 'core',
        citationSnippetIds: [`snippet-${id}-${index}`],
      })),
      conceptIndex: keyConcepts.map((term, index) => ({
        term,
        aliases: term === 'Jacobian matrix' ? ['Jacobian', '雅可比矩阵'] : [],
        snippetIds: [`snippet-${id}-${index}`],
        importance: 10 - index,
      })),
      evidenceSnippets: retrievalUnits.map((unit, index) => ({
        id: `snippet-${id}-${index}`,
        assetId: id,
        text: unit.summary,
        anchorRef: `anchor-${id}-${index}`,
        sourceSectionTitle: unit.title,
        confidence: 'high',
        quoteKind: 'definition',
      })),
      retrievalHints: keyConcepts,
      riskFlags: [],
    },
    planningLayer: {
      chapterSeeds: [],
      learningObjectives: [],
      exerciseSeeds: [],
      diagramOpportunities: [],
      difficultySignals: [],
      coverageGaps: [],
    },
    projectionLayer: {
      sourceGuide: `${title} guide`,
      keywords: keyConcepts,
      keyConcepts,
      readingTimeEstimateMinutes: 1,
      recommendedUse: 'Use as primary source material.',
    },
  };
}

test('buildProjectConceptIndex merges overlapping Jacobian concepts across sources into one project concept', () => {
  const assets = [
    createAsset('a', 'Jacobian Definition', ['Jacobian matrix'], [
      {
        id: 'unit-a-1',
        title: '什么是雅可比矩阵',
        summary: 'The Jacobian matrix collects first-order partial derivatives.',
        conceptRefs: ['Jacobian matrix'],
      },
    ]),
    createAsset('b', 'Jacobian Intuition', ['雅可比矩阵'], [
      {
        id: 'unit-b-1',
        title: '雅可比矩阵：多维空间的变化率',
        summary: 'The Jacobian describes local rate of change in multiple dimensions.',
        conceptRefs: ['雅可比矩阵'],
      },
    ]),
  ];

  const conceptIndex = buildProjectConceptIndex(assets);

  assert.equal(conceptIndex.concepts.length, 1);
  assert.match(conceptIndex.concepts[0].canonicalName, /Jacobian|雅可比矩阵/);
  assert.equal(conceptIndex.concepts[0].evidenceRefs.length, 2);
});

test('buildProjectConceptIndex keeps unrelated topics as separate project concepts', () => {
  const assets = [
    createAsset('a', 'Jacobian Definition', ['Jacobian matrix'], [
      {
        id: 'unit-a-1',
        title: 'Jacobian basics',
        summary: 'The Jacobian matrix collects first-order partial derivatives.',
        conceptRefs: ['Jacobian matrix'],
      },
    ]),
    createAsset('b', 'Eigenvalues Notes', ['Eigenvalues'], [
      {
        id: 'unit-b-1',
        title: 'Eigenvalues basics',
        summary: 'Eigenvalues describe invariant scaling factors.',
        conceptRefs: ['Eigenvalues'],
      },
    ]),
  ];

  const conceptIndex = buildProjectConceptIndex(assets);

  assert.equal(conceptIndex.concepts.length, 2);
});
```

- [ ] **Step 2: Run the new concept planning test to verify it fails**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-develop/GGlearn
node --import tsx --test tests/projectConceptPlanning.test.ts
```

Expected: FAIL with module-not-found or missing export errors for `projectConceptPlanning`.

- [ ] **Step 3: Write failing outline tests that require project concepts instead of chapter seeds**

Modify `GGlearn/tests/outlineGeneration.test.ts` by appending these tests:

```ts
test('generateEnhancedOutline merges overlapping project concepts into one chapter and stores conceptIds', async () => {
  const assetA = createSnippetOnlyAsset();
  assetA.id = 'asset-a';
  assetA.projectSourceId = 'source-a';
  assetA.retrievalLayer.retrievalUnits = [
    {
      id: 'unit-a',
      assetId: 'asset-a',
      title: '什么是雅可比矩阵',
      content: 'definition',
      summary: 'The Jacobian matrix collects first-order partial derivatives.',
      anchorRefs: ['anchor-a'],
      conceptRefs: ['Jacobian matrix'],
      prerequisites: [],
      difficulty: 'introductory',
      teachingValue: 'core',
      citationSnippetIds: ['snippet-jacobian'],
    },
  ];
  assetA.retrievalLayer.conceptIndex = [
    { term: 'Jacobian matrix', aliases: ['Jacobian', '雅可比矩阵'], snippetIds: ['snippet-jacobian'], importance: 10 },
  ];

  const assetB = createSnippetOnlyAsset();
  assetB.id = 'asset-b';
  assetB.projectSourceId = 'source-b';
  assetB.retrievalLayer.retrievalUnits = [
    {
      id: 'unit-b',
      assetId: 'asset-b',
      title: '雅可比矩阵：多维空间的变化率',
      content: 'intuition',
      summary: 'The Jacobian describes local rate of change in multiple dimensions.',
      anchorRefs: ['anchor-b'],
      conceptRefs: ['雅可比矩阵'],
      prerequisites: [],
      difficulty: 'introductory',
      teachingValue: 'core',
      citationSnippetIds: ['snippet-jacobian'],
    },
  ];
  assetB.retrievalLayer.conceptIndex = [
    { term: '雅可比矩阵', aliases: ['Jacobian matrix', 'Jacobian'], snippetIds: ['snippet-jacobian'], importance: 10 },
  ];

  const outline = await generateEnhancedOutline(
    [assetA, assetB],
    learningBrief,
    { provider: 'gemini', apiKey: '', model: 'gemini-3-flash-preview' }
  );

  assert.equal(outline.chapters.length, 1);
  assert.ok(Array.isArray(outline.chapters[0].conceptIds));
  assert.equal(outline.chapters[0].conceptIds.length, 1);
});
```

- [ ] **Step 4: Run the outline test to verify it fails**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-develop/GGlearn
node --import tsx --test tests/outlineGeneration.test.ts
```

Expected: FAIL because `OutlineChapter` has no `conceptIds` and `generateEnhancedOutline` still emits duplicate chapters.

- [ ] **Step 5: Write failing textbook evidence-selection and persistence tests**

Modify `GGlearn/tests/textbookGeneration.test.ts` and `GGlearn/tests/persistence.test.ts` by appending:

```ts
test('buildEvidencePacks prioritizes chapter conceptIds over chapter title matching', () => {
  const { asset } = createAsset();
  asset.retrievalLayer.retrievalUnits = [
    {
      id: 'unit-jacobian',
      assetId: asset.id,
      title: 'Multi-variable rate intuition',
      content: 'definition and intuition',
      summary: 'The Jacobian matrix describes local linear change.',
      anchorRefs: ['anchor-jacobian'],
      conceptRefs: ['Jacobian matrix'],
      prerequisites: [],
      difficulty: 'introductory',
      teachingValue: 'core',
      citationSnippetIds: ['snippet-jacobian'],
    },
  ];
  asset.retrievalLayer.evidenceSnippets = [
    {
      id: 'snippet-jacobian',
      assetId: asset.id,
      text: 'The Jacobian matrix describes local linear change.',
      anchorRef: 'anchor-jacobian',
      sourceSectionTitle: 'Local change',
      confidence: 'high',
      quoteKind: 'definition',
    },
  ];

  const packs = buildEvidencePacks([asset], undefined, ['concept-jacobian']);

  assert.equal(packs.length, 1);
  assert.equal(packs[0].retrievalUnitId, 'unit-jacobian');
});
```

```ts
test('normalizeProject backfills conceptIds when legacy outline chapters are reconstructed', () => {
  const legacyProject: Partial<Project> & { id: string; title: string } = {
    id: 'legacy-2',
    title: 'Legacy',
    outline: ['Introduction'],
    chunks: [],
  };

  const normalized = normalizeProject(legacyProject);

  assert.deepEqual(normalized.enhancedOutline?.chapters[0].conceptIds, []);
});
```

- [ ] **Step 6: Run the targeted tests to verify they fail**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-develop/GGlearn
node --import tsx --test tests/textbookGeneration.test.ts tests/persistence.test.ts
```

Expected: FAIL because `buildEvidencePacks` does not accept `conceptIds` and legacy outline backfill omits the new field.

- [ ] **Step 7: Commit the red test boundary**

```bash
git add GGlearn/tests/projectConceptPlanning.test.ts GGlearn/tests/outlineGeneration.test.ts GGlearn/tests/textbookGeneration.test.ts GGlearn/tests/persistence.test.ts
git commit -m "Lock project-level planning aggregation with failing tests"
```

### Task 2: Add Runtime Project Concept Types And Aggregation Builder

**Files:**
- Modify: `GGlearn/src/types.ts`
- Create: `GGlearn/src/lib/ai/projectConceptPlanning.ts`
- Test: `GGlearn/tests/projectConceptPlanning.test.ts`

- [ ] **Step 1: Extend the core types with project-level planning objects**

Modify `GGlearn/src/types.ts` to add these definitions near `ConceptIndexEntry`, `ChapterSeed`, and `OutlineChapter`:

```ts
export interface ConceptEvidenceRef {
  sourceId: string;
  assetId: string;
  retrievalUnitId: string;
  snippetId?: string;
  anchorRef?: string;
  role: 'primary' | 'supporting';
}

export interface ProjectConcept {
  id: string;
  canonicalName: string;
  aliases: string[];
  summary: string;
  evidenceRefs: ConceptEvidenceRef[];
  prerequisiteConceptIds: string[];
  difficulty: 'introductory' | 'intermediate' | 'advanced';
  coverageScore: number;
  riskFlags: string[];
}

export interface ProjectConceptIndex {
  version: 1;
  concepts: ProjectConcept[];
}
```

Also extend `OutlineChapter`:

```ts
export interface OutlineChapter {
  id: string;
  title: string;
  order: number;
  conceptIds: string[];
  learningObjectives: string[];
  prerequisites: string[];
  estimatedDuration: number;
  // rest unchanged
}
```

- [ ] **Step 2: Implement the deterministic runtime concept aggregation module**

Create `GGlearn/src/lib/ai/projectConceptPlanning.ts` with this initial implementation:

```ts
import type {
  ConceptEvidenceRef,
  ProjectConcept,
  ProjectConceptIndex,
  ProjectSourceAsset,
  RetrievalUnit,
} from '../../types';

type ConceptCandidate = {
  key: string;
  label: string;
  aliases: string[];
  summary: string;
  retrievalUnit: RetrievalUnit;
  asset: ProjectSourceAsset;
};

function normalizeConceptToken(value: string): string {
  return value
    .toLowerCase()
    .replace(/[^\p{L}\p{N}]+/gu, ' ')
    .replace(/\s+/g, ' ')
    .trim();
}

function buildAliasSet(values: string[]): string[] {
  const seen = new Set<string>();
  const result: string[] = [];
  for (const value of values) {
    const normalized = value.trim();
    if (!normalized) continue;
    const key = normalizeConceptToken(normalized);
    if (!key || seen.has(key)) continue;
    seen.add(key);
    result.push(normalized);
  }
  return result;
}

function collectCandidates(asset: ProjectSourceAsset): ConceptCandidate[] {
  const aliasesByTerm = new Map<string, string[]>();
  for (const entry of asset.retrievalLayer.conceptIndex) {
    aliasesByTerm.set(normalizeConceptToken(entry.term), [entry.term, ...entry.aliases]);
  }

  return asset.retrievalLayer.retrievalUnits.flatMap((unit) => {
    const terms = unit.conceptRefs.length ? unit.conceptRefs : asset.projectionLayer.keyConcepts;
    return terms.map((term) => {
      const normalizedTerm = normalizeConceptToken(term);
      return {
        key: normalizedTerm,
        label: term,
        aliases: aliasesByTerm.get(normalizedTerm) ?? [term],
        summary: unit.summary,
        retrievalUnit: unit,
        asset,
      };
    });
  });
}

function createEvidenceRef(candidate: ConceptCandidate): ConceptEvidenceRef {
  return {
    sourceId: candidate.asset.projectSourceId,
    assetId: candidate.asset.id,
    retrievalUnitId: candidate.retrievalUnit.id,
    snippetId: candidate.retrievalUnit.citationSnippetIds[0],
    anchorRef: candidate.retrievalUnit.anchorRefs[0],
    role: candidate.retrievalUnit.teachingValue === 'core' ? 'primary' : 'supporting',
  };
}

export function buildProjectConceptIndex(assets: ProjectSourceAsset[]): ProjectConceptIndex {
  const concepts = new Map<string, ProjectConcept>();

  for (const asset of assets) {
    for (const candidate of collectCandidates(asset)) {
      if (!candidate.key) continue;
      const existing = concepts.get(candidate.key);
      const aliases = buildAliasSet(candidate.aliases);
      const evidenceRef = createEvidenceRef(candidate);

      if (!existing) {
        concepts.set(candidate.key, {
          id: `concept-${candidate.key.replace(/\s+/g, '-')}`,
          canonicalName: aliases[0] ?? candidate.label,
          aliases,
          summary: candidate.summary,
          evidenceRefs: [evidenceRef],
          prerequisiteConceptIds: [],
          difficulty: candidate.retrievalUnit.difficulty,
          coverageScore: evidenceRef.role === 'primary' ? 100 : 70,
          riskFlags: [],
        });
        continue;
      }

      existing.aliases = buildAliasSet([...existing.aliases, ...aliases]);
      existing.evidenceRefs = [...existing.evidenceRefs, evidenceRef];
      existing.summary = existing.summary.length >= candidate.summary.length ? existing.summary : candidate.summary;
      existing.coverageScore = Math.min(100, existing.coverageScore + 10);
    }
  }

  return {
    version: 1,
    concepts: [...concepts.values()].sort((left, right) => right.coverageScore - left.coverageScore),
  };
}
```

- [ ] **Step 3: Run the concept planning tests to verify they pass**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-develop/GGlearn
node --import tsx --test tests/projectConceptPlanning.test.ts
```

Expected: PASS with both clustering tests green.

- [ ] **Step 4: Run the typecheck to ensure the new interfaces integrate cleanly**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-develop/GGlearn
npm run lint
```

Expected: PASS with no TypeScript errors.

- [ ] **Step 5: Commit the concept aggregation substrate**

```bash
git add GGlearn/src/types.ts GGlearn/src/lib/ai/projectConceptPlanning.ts GGlearn/tests/projectConceptPlanning.test.ts
git commit -m "Add runtime project concept aggregation types and builder"
```

### Task 3: Refactor Outline Generation To Consume Project Concepts

**Files:**
- Modify: `GGlearn/src/lib/ai/outlineGeneration.ts`
- Modify: `GGlearn/tests/outlineGeneration.test.ts`
- Test: `GGlearn/tests/projectConceptPlanning.test.ts`

- [ ] **Step 1: Replace source-local seed collection with project concept planning**

Modify `GGlearn/src/lib/ai/outlineGeneration.ts` to import the new planner:

```ts
import type {
  AIConfig,
  ChapterSeed,
  EnhancedOutline,
  EvidencePack,
  LearningBrief,
  LearningMode,
  OutlineChapter,
  PriorKnowledgeLevel,
  ProjectConcept,
  ProjectSourceAsset,
} from '../../types';
import { buildProjectConceptIndex } from './projectConceptPlanning';
```

Add a chapter factory driven by `ProjectConcept`:

```ts
function createOutlineChaptersFromConcepts(
  concepts: ProjectConcept[],
  assets: ProjectSourceAsset[],
  learningMode: LearningMode
): OutlineChapter[] {
  return concepts.map((concept, index) => {
    const relevantPacks = findRelevantEvidencePacks(concept.canonicalName, assets, learningMode);
    const chapter: Partial<OutlineChapter> = {
      id: `chapter-${Date.now()}-${index}`,
      title: concept.canonicalName,
      order: index,
      conceptIds: [concept.id],
      learningObjectives: [],
      prerequisites: [],
      estimatedDuration: 30,
      evidenceMapping: {
        primaryPackIds: relevantPacks.slice(0, 5).map((p) => p.id),
        supportingPackIds: relevantPacks.slice(5, 10).map((p) => p.id),
        coverageScore: calculateCoverageScore(relevantPacks),
      },
      generationStatus: 'pending',
      generatedChunkIds: [],
    };

    chapter.contentPlan = planChunkDistribution(chapter, learningMode, assets);
    return chapter as OutlineChapter;
  });
}
```

Then change `generateEnhancedOutline()`:

```ts
export async function generateEnhancedOutline(
  assets: ProjectSourceAsset[],
  learningBrief: LearningBrief,
  config: AIConfig,
  mode?: LearningMode
): Promise<EnhancedOutline> {
  const learningMode = mode ?? (learningBrief as LearningBrief & { mode?: LearningMode }).mode ?? 'mastery';
  const conceptIndex = buildProjectConceptIndex(assets);

  if (conceptIndex.concepts.length === 0) {
    throw new Error('No project concepts found in source assets');
  }

  const fallbackChapters = createOutlineChaptersFromConcepts(conceptIndex.concepts, assets, learningMode);
  let chapters = fallbackChapters;

  try {
    const aiCandidates = await requestAiOutlineCandidates(
      conceptIndex.concepts.map((concept) => ({
        id: concept.id,
        title: concept.canonicalName,
        rationale: concept.summary,
        retrievalUnitIds: concept.evidenceRefs.map((ref) => ref.retrievalUnitId),
      })),
      learningBrief,
      config,
      learningMode
    );

    chapters = applyAiOutlineCandidates(fallbackChapters, aiCandidates);
  } catch {
    chapters = fallbackChapters;
  }

  const orderedChapters = orderChaptersByDifficulty(chapters, learningBrief.priorKnowledgeLevel || 'basic');
  return {
    chapters: orderedChapters,
    metadata: {
      totalEstimatedDuration: orderedChapters.reduce((sum, chapter) => sum + chapter.estimatedDuration, 0),
      difficultyProgression: 'stepped',
      coherenceScore: calculateCoherence(orderedChapters),
    },
  };
}
```

- [ ] **Step 2: Ensure AI-edited chapter candidates preserve conceptIds**

Keep `applyAiOutlineCandidates()` structure-preserving:

```ts
function applyAiOutlineCandidates(
  fallbackChapters: OutlineChapter[],
  candidates: AiOutlineChapterCandidate[]
): OutlineChapter[] {
  if (candidates.length === 0) {
    throw new Error('AI outline response did not include valid chapters.');
  }

  return fallbackChapters.map((chapter, index) => {
    const candidate = candidates[index];
    if (!candidate?.title?.trim()) {
      return chapter;
    }

    return {
      ...chapter,
      title: candidate.title.trim(),
      learningObjectives: Array.isArray(candidate.learningObjectives)
        ? candidate.learningObjectives.map((objective) => objective.trim()).filter(Boolean).slice(0, 3)
        : chapter.learningObjectives,
      prerequisites: Array.isArray(candidate.prerequisites)
        ? candidate.prerequisites.map((prerequisite) => prerequisite.trim()).filter(Boolean)
        : chapter.prerequisites,
      estimatedDuration:
        typeof candidate.estimatedDuration === 'number' && Number.isFinite(candidate.estimatedDuration) && candidate.estimatedDuration > 0
          ? Math.round(candidate.estimatedDuration)
          : chapter.estimatedDuration,
    };
  });
}
```

- [ ] **Step 3: Run the outline tests to verify they pass**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-develop/GGlearn
node --import tsx --test tests/outlineGeneration.test.ts
```

Expected: PASS, including the new concept-driven deduplication test.

- [ ] **Step 4: Run the focused integration between concept planning and outline generation**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-develop/GGlearn
node --import tsx --test tests/projectConceptPlanning.test.ts tests/outlineGeneration.test.ts
```

Expected: PASS with no duplicate Jacobian chapter.

- [ ] **Step 5: Commit the outline refactor**

```bash
git add GGlearn/src/lib/ai/outlineGeneration.ts GGlearn/tests/outlineGeneration.test.ts
git commit -m "Drive outline generation from project concepts"
```

### Task 4: Make Chapter Evidence Selection Prefer conceptIds

**Files:**
- Modify: `GGlearn/src/lib/ai/textbookGeneration.ts`
- Modify: `GGlearn/tests/textbookGeneration.test.ts`

- [ ] **Step 1: Extend evidence pack ranking to accept chapter concept ids**

Modify the signatures in `GGlearn/src/lib/ai/textbookGeneration.ts`:

```ts
function computePackRelevance(
  pack: EvidencePack,
  chapterTopic?: string,
  conceptIds?: string[]
): number {
  let score = pack.teachingValue === 'core' ? 6 : pack.teachingValue === 'supporting' ? 4 : 2;
  score += pack.confidence === 'high' ? 3 : pack.confidence === 'medium' ? 2 : 1;

  if (conceptIds?.length) {
    const conceptTokens = conceptIds.flatMap((conceptId) => conceptId.toLowerCase().split(/[^a-z0-9\u4e00-\u9fff]+/i));
    const conceptMatchCount = conceptTokens.filter((token) => token && pack.conceptRefs.some((ref) => ref.toLowerCase().includes(token))).length;
    score += conceptMatchCount * 5;
  }

  if (!chapterTopic?.trim()) {
    return score;
  }

  const topicTokens = tokenizeForMatch(chapterTopic);
  const haystack = `${pack.retrievalUnitTitle} ${pack.summary} ${pack.excerpt} ${pack.conceptRefs.join(' ')}`.toLowerCase();
  const matchCount = topicTokens.filter((token) => haystack.includes(token)).length;
  score += (topicTokens.length > 0 ? matchCount / topicTokens.length : 0) * 10;
  return score;
}
```

Update `buildEvidencePacks()`:

```ts
export function buildEvidencePacks(
  assets: ProjectSourceAsset[],
  chapterTopic?: string,
  conceptIds?: string[]
): EvidencePack[] {
  return assets
    .flatMap((asset) => buildEvidencePackFromAsset(asset, chapterTopic).slice(0, 6))
    .sort((left, right) => computePackRelevance(right, chapterTopic, conceptIds) - computePackRelevance(left, chapterTopic, conceptIds))
    .slice(0, chapterTopic || conceptIds?.length ? 12 : 16);
}
```

- [ ] **Step 2: Thread conceptIds through chapter generation**

Change the chapter generation entrypoint:

```ts
export async function generateTextbookChunksWithProvider(
  input: GenerateTextbookChunksInput,
  provider: TextbookChunkProvider
): Promise<Chunk[]> {
  const evidencePacks = buildEvidencePacks(
    input.assets,
    input.chapterTopic,
    input.chapterPlan?.conceptIds
  );

  // existing body unchanged
}
```

Also update any fallback helper that calls `buildEvidencePacks(input.assets, input.chapterTopic)` to pass `input.chapterPlan?.conceptIds`.

- [ ] **Step 3: Make the new tests pass**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-develop/GGlearn
node --import tsx --test tests/textbookGeneration.test.ts
```

Expected: PASS, including the new conceptIds-priority test.

- [ ] **Step 4: Run the outline + textbook boundary tests together**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-develop/GGlearn
node --import tsx --test tests/outlineGeneration.test.ts tests/textbookGeneration.test.ts
```

Expected: PASS with no type or evidence-selection regressions.

- [ ] **Step 5: Commit the evidence selection change**

```bash
git add GGlearn/src/lib/ai/textbookGeneration.ts GGlearn/tests/textbookGeneration.test.ts
git commit -m "Prefer concept-bound evidence selection for chapter generation"
```

### Task 5: Backfill Persistence And Run The Full Regression Slice

**Files:**
- Modify: `GGlearn/src/lib/persistence.ts`
- Modify: `GGlearn/tests/persistence.test.ts`
- Modify: `GGlearn/tests/generationOrchestrator.test.ts`

- [ ] **Step 1: Backfill conceptIds for legacy enhanced outlines**

Modify `backfillEnhancedOutline()` in `GGlearn/src/lib/persistence.ts`:

```ts
function backfillEnhancedOutline(project: Project): Project {
  if (project.enhancedOutline || !project.outline.length) {
    return project;
  }

  const chapters: EnhancedOutline['chapters'] = project.outline.map((title, index) => {
    const chapterChunks = project.chunks.filter(
      (c) => c.metadata?.chapterTitle === title || c.metadata?.chapterId === `chapter-${index}`
    );

    const isCompleted =
      project.currentOutlineIndex !== undefined && index < project.currentOutlineIndex;

    return {
      id: chapterChunks[0]?.metadata?.chapterId || `chapter-${index}`,
      title,
      order: index + 1,
      conceptIds: [],
      learningObjectives: [],
      prerequisites: [],
      estimatedDuration: 15,
      contentPlan: {
        targetChunkCount: 3,
        chunkTypeDistribution: { text: 1, diagram: 1, exercise: 1, summary: 0 },
      },
      evidenceMapping: {
        primaryPackIds: [],
        supportingPackIds: [],
        coverageScore: 0,
      },
      generationStatus: isCompleted ? 'completed' : 'pending',
      generatedChunkIds: chapterChunks.map((c) => c.id),
    };
  });

  return {
    ...project,
    enhancedOutline: {
      chapters,
      metadata: {
        totalEstimatedDuration: chapters.length * 15,
        difficultyProgression: 'linear',
        coherenceScore: 0,
      },
    },
  };
}
```

- [ ] **Step 2: Update test fixtures that construct OutlineChapter directly**

Modify `GGlearn/tests/generationOrchestrator.test.ts` and any other `EnhancedOutline` fixtures to include `conceptIds: []`:

```ts
const mockEnhancedOutline: EnhancedOutline = {
  chapters: [
    {
      id: 'chapter-1',
      title: 'Chapter 1',
      order: 1,
      conceptIds: [],
      learningObjectives: ['Goal 1'],
      prerequisites: [],
      estimatedDuration: 10,
      contentPlan: {
        targetChunkCount: 2,
        chunkTypeDistribution: { text: 1, diagram: 0, exercise: 1, summary: 0 },
      },
      evidenceMapping: { primaryPackIds: [], supportingPackIds: [], coverageScore: 100 },
      generationStatus: 'pending',
      generatedChunkIds: [],
    },
  ],
  metadata: {
    totalEstimatedDuration: 10,
    difficultyProgression: 'linear',
    coherenceScore: 100,
  },
};
```

- [ ] **Step 3: Run the persistence and orchestration regression tests**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-develop/GGlearn
node --import tsx --test tests/persistence.test.ts tests/generationOrchestrator.test.ts
```

Expected: PASS with conceptIds backfilled for legacy outlines and no orchestrator fixture failures.

- [ ] **Step 4: Run the full feature regression slice**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-develop/GGlearn
node --import tsx --test tests/projectConceptPlanning.test.ts tests/outlineGeneration.test.ts tests/textbookGeneration.test.ts tests/persistence.test.ts tests/generationOrchestrator.test.ts
npm run lint
```

Expected:

```text
All selected test files PASS
TypeScript reports 0 errors
```

- [ ] **Step 5: Commit the compatibility and verification pass**

```bash
git add GGlearn/src/lib/persistence.ts GGlearn/tests/persistence.test.ts GGlearn/tests/generationOrchestrator.test.ts
git commit -m "Backfill concept ids and verify planning aggregation compatibility"
```

### Task 6: Final Integration Review And Cleanup

**Files:**
- Modify: `GGlearn/src/lib/ai/sourceAsset.ts`
- Modify: `GGlearn/src/lib/ai/outlineGeneration.ts`
- Test: `GGlearn/tests/outlineGeneration.test.ts`

- [ ] **Step 1: De-emphasize chapterSeeds in source AI enrichment**

Adjust `GGlearn/src/lib/ai/sourceAsset.ts` so the enrichment prompt and merge logic make `chapterSeeds` a compatibility signal rather than the main planning product.

Change the OpenAI-side instruction text:

```ts
`Return JSON with sourceGuide, keywords, keyConcepts, learningObjectives, chapterSeeds, exerciseSeeds, diagramOpportunities, coverageGaps.`,
```

to:

```ts
`Return JSON with sourceGuide, keywords, keyConcepts, learningObjectives, chapterSeeds, exerciseSeeds, diagramOpportunities, coverageGaps. chapterSeeds are optional compatibility hints only; prioritize normalized concepts and aliases over direct chapter proposals.`,
```

And annotate the merge point:

```ts
chapterSeeds: [...asset.planningLayer.chapterSeeds, ...aiChapterSeeds]
  .filter((seed, index, all) => all.findIndex((candidate) => candidate.title.toLowerCase() === seed.title.toLowerCase()) === index)
  .slice(0, 6), // Compatibility only; outline planning now prefers project concept aggregation.
```

- [ ] **Step 2: Remove dead chapter-seed-first assumptions from outline comments**

Update any comments in `GGlearn/src/lib/ai/outlineGeneration.ts` that still describe the outline input as “collect all chapter seeds,” replacing them with wording that matches project concept aggregation:

```ts
// Step 1: Build project-level concepts from all selected source assets.
// Step 2: Generate deterministic chapters from project concepts.
// Step 3: Let AI polish the learner-facing outline copy while preserving concept binding.
```

- [ ] **Step 3: Run the final outline regression after comment and prompt cleanup**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-develop/GGlearn
node --import tsx --test tests/outlineGeneration.test.ts
```

Expected: PASS with no behavior changes from the cleanup.

- [ ] **Step 4: Commit the cleanup pass**

```bash
git add GGlearn/src/lib/ai/sourceAsset.ts GGlearn/src/lib/ai/outlineGeneration.ts
git commit -m "Align source enrichment and comments with project concept planning"
```
