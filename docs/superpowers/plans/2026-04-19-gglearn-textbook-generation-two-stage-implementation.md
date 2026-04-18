# GGlearn Textbook Generation Two-Stage Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 将 GGlearn 的教材生成从单次 evidence 拼装改为“chapter blueprint -> section-by-section generation -> lightweight validation”的两阶段链路，同时保持现有 UI 调用面基本不变。

**Architecture:** 先在共享类型中引入 blueprint 和 validation 对象，再把 `textbookGeneration.ts` 拆成三个可测试层面：确定性 helper、可注入的 section orchestration、模型驱动的 provider 调用。公开的 `generateTextbookChunks(...)` 继续保留原签名，对上层 `App.tsx` 保持兼容，只替换内部实现路径。

**Tech Stack:** TypeScript, node:test, tsx, Vite, `@google/genai`, existing OpenAI-compatible HTTP client

---

## 文件结构与职责

- 修改: `GGlearn/src/types.ts`
  - 新增 `ChapterBlueprint`、`ChapterSectionBlueprint`、`SectionGenerationInput`、`GeneratedSectionValidation`、`GeneratedChapterValidation`
- 修改: `GGlearn/src/lib/ai/textbookGeneration.ts`
  - 增加 blueprint 生成、section evidence 选择、section orchestration、validation helper，并重构 `generateTextbookChunks(...)`
- 修改: `GGlearn/tests/textbookGeneration.test.ts`
  - 为新 helper、orchestration 和 validation 增加确定性测试

本计划刻意不改这些文件：

- `GGlearn/src/App.tsx`
  - 第一版保持调用方式不变，避免 UI 与生成链路同时改动
- `GGlearn/src/lib/sourceTransform.ts`
  - 这轮不重做 retrieval，只消费现有 `ProjectSourceAsset`
- `GGlearn/src/lib/gemini.ts`
  - 不额外暴露新接口，先只改已有导出面后面的实现

### Task 1: 先锁定共享类型和 failing tests

**Files:**
- Modify: `GGlearn/tests/textbookGeneration.test.ts`
- Modify: `GGlearn/src/types.ts`
- Modify: `GGlearn/src/lib/ai/textbookGeneration.ts`

- [ ] **Step 1: 在 `GGlearn/tests/textbookGeneration.test.ts` 追加两个 failing tests，先把新 helper 的行为锁住**

```ts
import {
  buildEvidencePacks,
  resolveSourceReferences,
  selectEvidencePacksForSection,
  validateGeneratedSection,
} from '../src/lib/ai/textbookGeneration';

test('selectEvidencePacksForSection prioritizes packs that match the section focus concepts', () => {
  const { asset } = createAsset();
  const evidencePacks = buildEvidencePacks([asset], 'Eigenvalues');

  const selected = selectEvidencePacksForSection(
    {
      id: 'section-definition',
      title: 'What Eigenvalues Mean',
      teachingRole: 'definition',
      dependsOnSections: ['section-intuition'],
      transitionFromPrevious: 'After motivating invariant scaling, define eigenvalues precisely.',
      focusConcepts: ['Eigenvalues', 'invariant scaling'],
    },
    evidencePacks
  );

  assert.ok(selected.length >= 1);
  assert.ok(selected.length <= 4);
  assert.ok(
    selected.some(
      (pack) =>
        pack.retrievalUnitTitle.toLowerCase().includes('eigenvalues') ||
        pack.summary.toLowerCase().includes('eigenvalues')
    )
  );
});

test('validateGeneratedSection flags unsupported claims and missing transitions', () => {
  const validation = validateGeneratedSection(
    {
      id: 'section-definition',
      title: 'What Eigenvalues Mean',
      teachingRole: 'definition',
      dependsOnSections: ['section-intuition'],
      transitionFromPrevious: 'Connect this section to the earlier intuition.',
      focusConcepts: ['Eigenvalues'],
    },
    [
      {
        id: 'chunk-1',
        type: 'text',
        title: 'What Eigenvalues Mean',
        content: 'A matrix is always diagonalizable over the reals.',
        depth: 'technical',
      },
    ],
    []
  );

  assert.equal(validation.hasEvidenceSupport, false);
  assert.ok(validation.unsupportedClaims.length >= 1);
  assert.ok(validation.transitionIssues.length >= 1);
});
```

- [ ] **Step 2: 运行单测，确认它们因为缺少导出而失败**

Run:

```bash
cd GGlearn && node --import tsx --test tests/textbookGeneration.test.ts --test-name-pattern "selectEvidencePacksForSection|validateGeneratedSection"
```

Expected:

```text
not ok ... SyntaxError or TypeError about missing export/function
```

- [ ] **Step 3: 在 `GGlearn/src/types.ts` 增加共享类型，先把后续实现的契约固定下来**

```ts
export interface ChapterSectionBlueprint {
  id: string;
  title: string;
  teachingRole: 'motivation' | 'intuition' | 'definition' | 'explanation' | 'derivation' | 'application' | 'summary';
  dependsOnSections: string[];
  transitionFromPrevious: string;
  focusConcepts: string[];
}

export interface ChapterBlueprint {
  chapterTitle: string;
  chapterGoal: string;
  targetReaderState: string;
  prerequisites: string[];
  chapterFlowNarrative: string;
  sections: ChapterSectionBlueprint[];
  endState: string;
  unsupportedGaps: string[];
}

export interface SectionGenerationInput {
  chapterGoal: string;
  chapterFlowNarrative: string;
  currentSection: ChapterSectionBlueprint;
  resolvedPrerequisites: string[];
  previousSectionSummary: string;
  selectedEvidencePacks: EvidencePack[];
}

export interface GeneratedSectionValidation {
  hasEvidenceSupport: boolean;
  unsupportedClaims: string[];
  transitionIssues: string[];
  prerequisiteIssues: string[];
}

export interface GeneratedChapterValidation {
  sections: Array<{
    sectionId: string;
    validation: GeneratedSectionValidation;
  }>;
  hasBlockingIssues: boolean;
}
```

- [ ] **Step 4: 在 `GGlearn/src/lib/ai/textbookGeneration.ts` 先补最小 helper stub，让测试从“缺少函数”推进到“断言失败”**

```ts
export function selectEvidencePacksForSection(
  _section: ChapterSectionBlueprint,
  evidencePacks: EvidencePack[]
): EvidencePack[] {
  return evidencePacks.slice(0, 4);
}

export function validateGeneratedSection(
  _section: ChapterSectionBlueprint,
  _chunks: Chunk[],
  _sourceRefs: ChunkSourceReference[]
): GeneratedSectionValidation {
  return {
    hasEvidenceSupport: true,
    unsupportedClaims: [],
    transitionIssues: [],
    prerequisiteIssues: [],
  };
}
```

- [ ] **Step 5: 再跑一次单测，确认现在是断言失败而不是接口缺失**

Run:

```bash
cd GGlearn && node --import tsx --test tests/textbookGeneration.test.ts --test-name-pattern "selectEvidencePacksForSection|validateGeneratedSection"
```

Expected:

```text
not ok ... assertion failure
```

- [ ] **Step 6: 提交这一轮契约与 failing tests**

```bash
git add GGlearn/src/types.ts GGlearn/src/lib/ai/textbookGeneration.ts GGlearn/tests/textbookGeneration.test.ts
git commit -m "Lock the two-stage generation contracts before refactoring" -m "Introduce the shared blueprint and validation types, then add failing textbook-generation tests so the refactor starts from explicit executable constraints.\n\nConstraint: Keep the public textbook-generation entrypoint stable for the current UI\nRejected: Start by rewriting provider calls first | would remove a deterministic test seam for the refactor\nConfidence: high\nScope-risk: narrow\nDirective: Keep helper contracts deterministic and testable before adding async model orchestration\nTested: node --import tsx --test tests/textbookGeneration.test.ts --test-name-pattern \"selectEvidencePacksForSection|validateGeneratedSection\"\nNot-tested: Full GGlearn test suite"
```

### Task 2: 实现确定性 helper，让 blueprint 和 validation 先可测试

**Files:**
- Modify: `GGlearn/tests/textbookGeneration.test.ts`
- Modify: `GGlearn/src/lib/ai/textbookGeneration.ts`

- [ ] **Step 1: 在 `GGlearn/tests/textbookGeneration.test.ts` 追加 fallback blueprint 测试，先要求生成出“先动机、后定义”的中骨架**

```ts
import { createFallbackChapterBlueprint } from '../src/lib/ai/textbookGeneration';

test('createFallbackChapterBlueprint builds a pedagogical section order from evidence packs', () => {
  const { asset } = createAsset();
  const evidencePacks = buildEvidencePacks([asset], 'Eigenvalues');

  const blueprint = createFallbackChapterBlueprint('Linear Algebra', evidencePacks, 'Eigenvalues');

  assert.equal(blueprint.chapterTitle, 'Eigenvalues');
  assert.ok(blueprint.sections.length >= 3);
  assert.equal(blueprint.sections[0]?.teachingRole, 'intuition');
  assert.ok(blueprint.sections.some((section) => section.teachingRole === 'definition'));
  assert.ok(blueprint.chapterFlowNarrative.length > 0);
});
```

- [ ] **Step 2: 运行该测试，确认它因缺少实现失败**

Run:

```bash
cd GGlearn && node --import tsx --test tests/textbookGeneration.test.ts --test-name-pattern "createFallbackChapterBlueprint"
```

Expected:

```text
not ok ... missing export/function
```

- [ ] **Step 3: 在 `GGlearn/src/lib/ai/textbookGeneration.ts` 实现三个确定性 helper**

```ts
export function createFallbackChapterBlueprint(
  title: string,
  evidencePacks: EvidencePack[],
  chapterTopic?: string
): ChapterBlueprint {
  const corePacks = evidencePacks.slice(0, 4);
  const focusConcepts = dedupePreserveOrder(corePacks.flatMap((pack) => pack.conceptRefs)).slice(0, 4);
  const chapterTitle = chapterTopic?.trim() || title;

  return {
    chapterTitle,
    chapterGoal: `Understand ${chapterTitle} well enough to explain the idea, its definition, and its use in context.`,
    targetReaderState: 'Knows the earlier chapter context but has not yet organized this topic into a clean learning progression.',
    prerequisites: focusConcepts.slice(0, 2),
    chapterFlowNarrative: `Start with intuition, formalize ${chapterTitle}, then connect it to problem-solving use.`,
    sections: [
      {
        id: 'section-intuition',
        title: `Why ${chapterTitle} matters`,
        teachingRole: 'intuition',
        dependsOnSections: [],
        transitionFromPrevious: 'Open the chapter by motivating the topic before introducing formal language.',
        focusConcepts: focusConcepts.slice(0, 2),
      },
      {
        id: 'section-definition',
        title: `${chapterTitle}: core definition`,
        teachingRole: 'definition',
        dependsOnSections: ['section-intuition'],
        transitionFromPrevious: 'Now turn the intuition into the formal statement readers will use later.',
        focusConcepts: focusConcepts.slice(0, 3),
      },
      {
        id: 'section-application',
        title: `Using ${chapterTitle}`,
        teachingRole: 'application',
        dependsOnSections: ['section-definition'],
        transitionFromPrevious: 'After the formal definition, show how the idea changes problem solving.',
        focusConcepts: focusConcepts.slice(0, 4),
      },
    ],
    endState: `The reader can explain ${chapterTitle} and connect it to a concrete mathematical use.`,
    unsupportedGaps: [],
  };
}

export function selectEvidencePacksForSection(
  section: ChapterSectionBlueprint,
  evidencePacks: EvidencePack[]
): EvidencePack[] {
  return [...evidencePacks]
    .map((pack) => ({
      pack,
      score: section.focusConcepts.reduce(
        (sum, concept) =>
          sum +
          (pack.summary.toLowerCase().includes(concept.toLowerCase()) ? 4 : 0) +
          (pack.retrievalUnitTitle.toLowerCase().includes(concept.toLowerCase()) ? 6 : 0) +
          (pack.conceptRefs.some((ref) => ref.toLowerCase() === concept.toLowerCase()) ? 8 : 0),
        0
      ),
    }))
    .sort((left, right) => right.score - left.score)
    .map((entry) => entry.pack)
    .slice(0, 4);
}

export function validateGeneratedSection(
  section: ChapterSectionBlueprint,
  chunks: Chunk[],
  sourceRefs: ChunkSourceReference[]
): GeneratedSectionValidation {
  const mergedContent = chunks.map((chunk) => `${chunk.title || ''} ${chunk.content}`).join('\n').toLowerCase();

  return {
    hasEvidenceSupport: sourceRefs.length > 0,
    unsupportedClaims: sourceRefs.length === 0 ? ['Section generated without any source-backed references.'] : [],
    transitionIssues: section.dependsOnSections.length > 0 && !mergedContent.includes('therefore') && !mergedContent.includes('因此')
      ? ['Section does not explicitly connect to prior material.']
      : [],
    prerequisiteIssues: section.focusConcepts.some((concept) => !mergedContent.includes(concept.toLowerCase()))
      ? ['Section does not clearly cover its declared focus concepts.']
      : [],
  };
}
```

- [ ] **Step 4: 跑 Task 1 和 Task 2 的相关测试，确认 helper 行为稳定**

Run:

```bash
cd GGlearn && node --import tsx --test tests/textbookGeneration.test.ts --test-name-pattern "selectEvidencePacksForSection|validateGeneratedSection|createFallbackChapterBlueprint"
```

Expected:

```text
ok 1 - selectEvidencePacksForSection prioritizes packs that match the section focus concepts
ok 2 - validateGeneratedSection flags unsupported claims and missing transitions
ok 3 - createFallbackChapterBlueprint builds a pedagogical section order from evidence packs
```

- [ ] **Step 5: 提交确定性 helper**

```bash
git add GGlearn/src/lib/ai/textbookGeneration.ts GGlearn/tests/textbookGeneration.test.ts
git commit -m "Make the two-stage textbook planner deterministic first" -m "Add fallback blueprint construction, section-level evidence ranking, and lightweight validation helpers so the refactor has a stable testable core before model orchestration is introduced.\n\nConstraint: This phase must stay retrieval-consumer only and not redesign source assets\nRejected: Encode a full prerequisite graph in the first pass | adds structure without proving the simpler planner first\nConfidence: high\nScope-risk: narrow\nDirective: Keep fallback planning intentionally simple; only promote new planning logic when it produces clearer chapter flow in tests and manual runs\nTested: node --import tsx --test tests/textbookGeneration.test.ts --test-name-pattern \"selectEvidencePacksForSection|validateGeneratedSection|createFallbackChapterBlueprint\"\nNot-tested: Provider-backed generation calls"
```

### Task 3: 引入可注入的 section orchestration，先把顺序和 carry-over 变成可测逻辑

**Files:**
- Modify: `GGlearn/tests/textbookGeneration.test.ts`
- Modify: `GGlearn/src/lib/ai/textbookGeneration.ts`

- [ ] **Step 1: 在 `GGlearn/tests/textbookGeneration.test.ts` 新增 orchestration test，用 fake generator 锁住顺序、上一节摘要和 evidence 选择**

```ts
import type { ChapterBlueprint, SectionGenerationInput } from '../src/types';
import { generateChunksFromBlueprint } from '../src/lib/ai/textbookGeneration';

test('generateChunksFromBlueprint walks sections in order and carries forward section summaries', async () => {
  const { asset } = createAsset();
  const evidencePacks = buildEvidencePacks([asset], 'Eigenvalues');
  const calls: SectionGenerationInput[] = [];

  const blueprint: ChapterBlueprint = {
    chapterTitle: 'Eigenvalues',
    chapterGoal: 'Understand what eigenvalues describe.',
    targetReaderState: 'Knows vectors and matrices.',
    prerequisites: ['Matrices'],
    chapterFlowNarrative: 'Move from intuition to definition to use.',
    sections: [
      {
        id: 'section-intuition',
        title: 'Why Eigenvalues Matter',
        teachingRole: 'intuition',
        dependsOnSections: [],
        transitionFromPrevious: 'Open with motivation.',
        focusConcepts: ['Eigenvalues'],
      },
      {
        id: 'section-definition',
        title: 'Defining Eigenvalues',
        teachingRole: 'definition',
        dependsOnSections: ['section-intuition'],
        transitionFromPrevious: 'Turn the motivation into a formal definition.',
        focusConcepts: ['Eigenvalues', 'eigenvectors'],
      },
    ],
    endState: 'Can explain eigenvalues in context.',
    unsupportedGaps: [],
  };

  const chunks = await generateChunksFromBlueprint(blueprint, evidencePacks, async (input) => {
    calls.push(input);
    return [
      {
        type: 'text',
        title: input.currentSection.title,
        content: `${input.previousSectionSummary || 'Start here.'} Therefore ${input.currentSection.focusConcepts.join(', ')} matter in this section.`,
        depth: 'conceptual',
        sourceRefIds: input.selectedEvidencePacks.slice(0, 1).map((pack) => pack.id),
      },
    ];
  });

  assert.equal(calls[0]?.currentSection.id, 'section-intuition');
  assert.equal(calls[0]?.previousSectionSummary, '');
  assert.equal(calls[1]?.currentSection.id, 'section-definition');
  assert.ok(calls[1]?.previousSectionSummary.includes('Why Eigenvalues Matter'));
  assert.ok(calls[1]?.selectedEvidencePacks.length <= 4);
  assert.equal(chunks.length, 2);
});
```

- [ ] **Step 2: 运行该测试，确认它因 orchestration helper 缺失而失败**

Run:

```bash
cd GGlearn && node --import tsx --test tests/textbookGeneration.test.ts --test-name-pattern "generateChunksFromBlueprint"
```

Expected:

```text
not ok ... missing export/function
```

- [ ] **Step 3: 在 `GGlearn/src/lib/ai/textbookGeneration.ts` 实现 section orchestration helper，并把 section summary carry-over 固定成小字符串**

```ts
type SectionDraftGenerator = (input: SectionGenerationInput) => Promise<GeneratedChunkDraft[]>;

function summarizeSectionDrafts(section: ChapterSectionBlueprint, drafts: GeneratedChunkDraft[]): string {
  const merged = drafts.map((draft) => `${draft.title || section.title} ${draft.content}`).join(' ');
  return merged.slice(0, 180);
}

export async function generateChunksFromBlueprint(
  blueprint: ChapterBlueprint,
  evidencePacks: EvidencePack[],
  generateSectionDrafts: SectionDraftGenerator
): Promise<Chunk[]> {
  const chunks: Chunk[] = [];
  let previousSectionSummary = '';
  const resolvedPrerequisites = [...blueprint.prerequisites];

  for (const section of blueprint.sections) {
    const selectedEvidencePacks = selectEvidencePacksForSection(section, evidencePacks);
    const drafts = await generateSectionDrafts({
      chapterGoal: blueprint.chapterGoal,
      chapterFlowNarrative: blueprint.chapterFlowNarrative,
      currentSection: section,
      resolvedPrerequisites: [...resolvedPrerequisites],
      previousSectionSummary,
      selectedEvidencePacks,
    });

    const normalized = normalizeGeneratedChunks(drafts, selectedEvidencePacks);
    chunks.push(...normalized);
    previousSectionSummary = summarizeSectionDrafts(section, drafts);
    resolvedPrerequisites.push(...section.focusConcepts);
  }

  return chunks;
}
```

- [ ] **Step 4: 运行 orchestration test，并补跑已有 evidence tests，确认没有破坏现有引用回绑**

Run:

```bash
cd GGlearn && node --import tsx --test tests/textbookGeneration.test.ts
```

Expected:

```text
ok ... buildEvidencePacks creates grounded packs from retrieval units
ok ... resolveSourceReferences uses explicit ids when the model returns valid evidence ids
ok ... resolveSourceReferences falls back to lexical matching when ids are missing
ok ... generateChunksFromBlueprint walks sections in order and carries forward section summaries
```

- [ ] **Step 5: 提交 orchestration helper**

```bash
git add GGlearn/src/lib/ai/textbookGeneration.ts GGlearn/tests/textbookGeneration.test.ts
git commit -m "Refactor textbook generation around section orchestration" -m "Introduce an injectable blueprint-driven section runner so ordering, carry-over summaries, and section-level evidence selection are testable without network calls.\n\nConstraint: Existing UI code must keep calling generateTextbookChunks with the current signature\nRejected: Test provider-backed generation directly in unit tests | would make the refactor brittle and nondeterministic\nConfidence: high\nScope-risk: moderate\nDirective: Keep carry-over summaries short and bounded; do not feed full chapter history back into each section\nTested: cd GGlearn && node --import tsx --test tests/textbookGeneration.test.ts\nNot-tested: Browser UI integration"
```

### Task 4: 接入真实 provider 调用，并把 `generateTextbookChunks(...)` 改到两阶段链路

**Files:**
- Modify: `GGlearn/src/lib/ai/textbookGeneration.ts`
- Modify: `GGlearn/tests/textbookGeneration.test.ts`

- [ ] **Step 1: 在 `GGlearn/tests/textbookGeneration.test.ts` 增加 chapter-level validation test，确保章节级问题会被收集而不是吞掉**

```ts
import { validateGeneratedChapter } from '../src/lib/ai/textbookGeneration';

test('validateGeneratedChapter reports blocking issues when a dependent section has no evidence support', () => {
  const blueprint = createFallbackChapterBlueprint('Linear Algebra', [], 'Eigenvalues');

  const validation = validateGeneratedChapter(blueprint, [
    {
      id: 'chunk-1',
      type: 'text',
      title: blueprint.sections[0]?.title,
      content: 'Introduce the topic without any transition marker.',
      depth: 'conceptual',
    },
  ]);

  assert.equal(validation.hasBlockingIssues, true);
  assert.ok(validation.sections.length >= 1);
});
```

- [ ] **Step 2: 运行该测试，确认 chapter-level validation 还未实现**

Run:

```bash
cd GGlearn && node --import tsx --test tests/textbookGeneration.test.ts --test-name-pattern "validateGeneratedChapter"
```

Expected:

```text
not ok ... missing export/function
```

- [ ] **Step 3: 在 `GGlearn/src/lib/ai/textbookGeneration.ts` 实现 provider-backed blueprint 生成、chapter validation 和新的 `generateTextbookChunks(...)` 内部流程**

```ts
export function validateGeneratedChapter(
  blueprint: ChapterBlueprint,
  chunks: Chunk[]
): GeneratedChapterValidation {
  const sections = blueprint.sections.map((section) => {
    const matchingChunks = chunks.filter((chunk) => chunk.title === section.title);
    const sourceRefs = matchingChunks.flatMap((chunk) => chunk.sourceRefs ?? []);
    return {
      sectionId: section.id,
      validation: validateGeneratedSection(section, matchingChunks, sourceRefs),
    };
  });

  return {
    sections,
    hasBlockingIssues: sections.some(
      (entry) =>
        !entry.validation.hasEvidenceSupport ||
        entry.validation.unsupportedClaims.length > 0 ||
        entry.validation.prerequisiteIssues.length > 0
    ),
  };
}

export async function generateChapterBlueprint(
  title: string,
  assets: ProjectSourceAsset[],
  goal: 'mastery' | 'exam' | 'micro',
  customFocus: string,
  language: 'zh' | 'en',
  config: AIConfig,
  chapterTopic?: string
): Promise<ChapterBlueprint> {
  const evidencePacks = buildEvidencePacks(assets, chapterTopic);

  if (!config.apiKey?.trim()) {
    return createFallbackChapterBlueprint(title, evidencePacks, chapterTopic);
  }

  const prompt = [
    `Build a chapter blueprint for ${chapterTopic?.trim() || title}.`,
    `[Goal Mode] ${goal}`,
    customFocus ? `[User Focus] ${customFocus}` : '',
    `[Output Language] ${language === 'zh' ? 'Chinese' : 'English'}`,
    `[Evidence Packs]\n${buildEvidencePrompt(evidencePacks)}`,
    `Return JSON with chapterTitle, chapterGoal, targetReaderState, prerequisites, chapterFlowNarrative, sections, endState, unsupportedGaps.`,
  ]
    .filter(Boolean)
    .join('\n\n');

  if (config.provider === 'gemini') {
    const ai = createGeminiClient(config);
    const response = await ai.models.generateContent({
      model: config.model?.trim() || 'gemini-3-flash-preview',
      contents: [{ role: 'user', parts: [{ text: prompt }] }],
      config: {
        responseMimeType: 'application/json',
        responseSchema: {
          type: Type.OBJECT,
          properties: {
            chapterTitle: { type: Type.STRING },
            chapterGoal: { type: Type.STRING },
            targetReaderState: { type: Type.STRING },
            prerequisites: { type: Type.ARRAY, items: { type: Type.STRING } },
            chapterFlowNarrative: { type: Type.STRING },
            endState: { type: Type.STRING },
            unsupportedGaps: { type: Type.ARRAY, items: { type: Type.STRING } },
            sections: {
              type: Type.ARRAY,
              items: {
                type: Type.OBJECT,
                properties: {
                  id: { type: Type.STRING },
                  title: { type: Type.STRING },
                  teachingRole: { type: Type.STRING },
                  dependsOnSections: { type: Type.ARRAY, items: { type: Type.STRING } },
                  transitionFromPrevious: { type: Type.STRING },
                  focusConcepts: { type: Type.ARRAY, items: { type: Type.STRING } },
                },
                required: ['id', 'title', 'teachingRole', 'dependsOnSections', 'transitionFromPrevious', 'focusConcepts'],
              },
            },
          },
          required: ['chapterTitle', 'chapterGoal', 'targetReaderState', 'prerequisites', 'chapterFlowNarrative', 'sections', 'endState', 'unsupportedGaps'],
        },
      },
    });

    return JSON.parse(response.text || '{}') as ChapterBlueprint;
  }

  const response = await postJson<{ choices: Array<{ message: { content: string } }> }>(
    `${getOpenAIBaseUrl(config)}/chat/completions`,
    {
      model: config.model || 'gpt-4-turbo-preview',
      messages: [{ role: 'user', content: prompt }],
      response_format: { type: 'json_object' },
    },
    getOpenAIApiKey(config)
  );

  try {
    return JSON.parse(response.choices[0]?.message.content || '{}') as ChapterBlueprint;
  } catch (error) {
    console.warn('Falling back to deterministic chapter blueprint.', error);
    return createFallbackChapterBlueprint(title, evidencePacks, chapterTopic);
  }
}

export async function generateTextbookChunks(
  title: string,
  assets: ProjectSourceAsset[],
  goal: 'mastery' | 'exam' | 'micro',
  customFocus: string,
  language: 'zh' | 'en',
  config: AIConfig,
  attachments: Attachment[] = [],
  chapterTopic?: string
): Promise<Chunk[]> {
  const evidencePacks = buildEvidencePacks(assets, chapterTopic);
  const blueprint = await generateChapterBlueprint(title, assets, goal, customFocus, language, config, chapterTopic);

  const chunks = await generateChunksFromBlueprint(blueprint, evidencePacks, async (input) => {
    const prompt = buildSectionPrompt(input, language, customFocus);

    if (config.provider === 'gemini') {
      const ai = createGeminiClient(config);
      const parts =
        attachments.length > 0
          ? [...attachments.map((attachment) => ({ inlineData: { mimeType: attachment.mimeType, data: attachment.data } })), { text: prompt }]
          : [{ text: prompt }];

      const response = await ai.models.generateContent({
        model: config.model?.trim() || 'gemini-3-flash-preview',
        contents: [{ role: 'user', parts }],
        config: {
          responseMimeType: 'application/json',
          responseSchema: {
            type: Type.ARRAY,
            items: {
              type: Type.OBJECT,
              properties: {
                type: { type: Type.STRING },
                title: { type: Type.STRING },
                content: { type: Type.STRING },
                depth: { type: Type.STRING },
                metadata: {
                  type: Type.OBJECT,
                  properties: {
                    answerKey: { type: Type.STRING },
                    hint: { type: Type.STRING },
                    diagramCode: { type: Type.STRING },
                  },
                },
                sourceRefIds: {
                  type: Type.ARRAY,
                  items: { type: Type.STRING },
                },
              },
              required: ['type', 'content', 'depth', 'sourceRefIds'],
            },
          },
          systemInstruction: getSystemPrompt(goal, language),
        },
      });

      return JSON.parse(response.text || '[]') as GeneratedChunkDraft[];
    }

    const response = await postJson<{ choices: Array<{ message: { content: string } }> }>(
      `${getOpenAIBaseUrl(config)}/chat/completions`,
      {
        model: config.model || 'gpt-4-turbo-preview',
        messages: [
          { role: 'system', content: getSystemPrompt(goal, language) },
          { role: 'user', content: prompt },
        ],
        response_format: { type: 'json_object' },
      },
      getOpenAIApiKey(config)
    );

    const json = JSON.parse(response.choices[0]?.message.content || '{}');
    return (Array.isArray(json) ? json : (json.chunks || [])) as GeneratedChunkDraft[];
  });

  const validation = validateGeneratedChapter(blueprint, chunks);
  if (validation.hasBlockingIssues) {
    console.warn('Generated chapter has blocking validation issues.', validation);
  }

  return chunks;
}
```

- [ ] **Step 4: 把 provider prompt 改成 section 级输入，保持现有 provider 分支和附件支持**

```ts
function buildSectionPrompt(input: SectionGenerationInput, language: 'zh' | 'en', customFocus: string): string {
  return [
    `[Chapter Goal] ${input.chapterGoal}`,
    `[Chapter Flow] ${input.chapterFlowNarrative}`,
    `[Output Language] ${language === 'zh' ? 'Chinese' : 'English'}`,
    `[Current Section] ${input.currentSection.title}`,
    `[Teaching Role] ${input.currentSection.teachingRole}`,
    `[Transition From Previous] ${input.currentSection.transitionFromPrevious}`,
    `[Previous Section Summary] ${input.previousSectionSummary || 'This is the first section.'}`,
    `[Resolved Prerequisites] ${input.resolvedPrerequisites.join(', ') || 'None'}`,
    `[Focus Concepts] ${input.currentSection.focusConcepts.join(', ')}`,
    customFocus ? `[User Focus] ${customFocus}` : '',
    `[Writing Contract] Write like a textbook author. Do not produce a source-summary list. Explain why this section appears now, develop the idea, and leave a bridge to the next section when needed.`,
    `[Evidence Packs]\n${buildEvidencePrompt(input.selectedEvidencePacks)}`,
  ]
    .filter(Boolean)
    .join('\n\n');
}
```

- [ ] **Step 5: 跑完整验证，确认类型检查和单测通过**

Run:

```bash
cd GGlearn && npm run lint
cd GGlearn && npm test
```

Expected:

```text
tsc --noEmit
... all tests pass
```

- [ ] **Step 6: 提交两阶段链路接入**

```bash
git add GGlearn/src/lib/ai/textbookGeneration.ts GGlearn/tests/textbookGeneration.test.ts GGlearn/src/types.ts
git commit -m "Route textbook generation through a chapter blueprint first" -m "Wire model-backed blueprint generation into the existing textbook entrypoint, then run section-by-section drafting with bounded carry-over and lightweight post-validation.\n\nConstraint: The first delivery must improve chapter flow without widening into UI or retrieval rewrites\nRejected: Return a brand-new generation payload shape to the UI immediately | expands the refactor surface without proving the new generation path first\nConfidence: medium\nScope-risk: moderate\nDirective: Keep the public generateTextbookChunks signature stable until the new pipeline is verified in reader flows\nTested: cd GGlearn && npm run lint; cd GGlearn && npm test\nNot-tested: Manual reader UX review with live provider output"
```

## 自检清单

- 本计划覆盖了 spec 中的三条主链：
  - `generateChapterBlueprint(...)`
  - blueprint-driven section generation
  - lightweight validation
- 本计划没有引入课程图谱、向量检索、citation UI 重构等非目标
- 本计划把异步模型调用后的控制逻辑拆到了可测 helper 中，避免把测试建立在真实 provider 返回值上
- 本计划保留现有 `generateTextbookChunks(...)` 调用面，降低对 `App.tsx` 的波及范围

## 执行注意事项

- 在 `GGlearn/` 目录下执行 lint 和 test，不要在仓库根目录直接运行 `npm test`
- 本计划默认只改动生成链路，不回头修改 source asset 结构
- 若在真实 provider 输出中发现 blueprint 过空，先补 fallback / parsing 约束，不要直接扩展结构复杂度
