# GGlearn Textbook Body Quality Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 将 GGlearn 的教材生成从“资料驱动的 chunk 卡片流”改成“后台 grounded、前台连续正文”的第一版可运行链路。

**Architecture:** 保留现有 `generateTextbookChunks(...)` 入口和 `Project.chunks` 存储，同时新增教学画像、章节蓝图、小节级正文生成和后台校验。生成结果继续映射为 `Chunk[]`，但正文 chunk 使用 metadata 标记 `sectionId`、`contentRole`、`isTextbookBody`，阅读页按正文流渲染并默认隐藏来源依据入口。

**Tech Stack:** TypeScript, React 19, Vite, node:test, tsx, existing Gemini/OpenAI-compatible client utilities.

---

## 文件结构与职责

- Modify: `GGlearn/src/types.ts`
  - 新增教材正文角色、教学画像、章节蓝图、section 输入、后台校验类型。
  - 扩展 `Chunk.metadata`，用兼容字段承载正文结构信息。
- Create: `GGlearn/src/lib/ai/textbookBody.ts`
  - 放置确定性 helper：教学画像、fallback 蓝图、section evidence 选择、正文 metadata 标准化、后台校验。
- Modify: `GGlearn/src/lib/ai/textbookGeneration.ts`
  - 保留已有公开入口，内部改成 blueprint -> section generation -> validation -> chunk normalization。
  - 新增可注入 provider 的 orchestration 函数，方便无网络单测。
- Modify: `GGlearn/src/lib/gemini.ts`
  - 继续导出 `generateTextbookChunks`，不新增前台调用面。
- Modify: `GGlearn/src/App.tsx`
  - 调用 `generateTextbookChunks(...)` 时传入 `learningBrief`，让基础水平和讲解偏好进入后台教学画像。
- Create: `GGlearn/src/lib/textbookBodyLayout.ts`
  - 提供阅读页分组 helper，把正文 chunk 按 section 聚合，互动 chunk 保持独立。
- Modify: `GGlearn/src/components/ChunkRenderer.tsx`
  - 对 `metadata.isTextbookBody` 的 chunk 采用正文呈现规则，不默认折叠，不显示 Sources 按钮，不把教材练习渲染成互动练习台。
- Modify: `GGlearn/src/views/ReaderView.tsx`
  - 按连续教材正文渲染 section；保留互动产物显示，但默认弱化来源追踪入口。
- Modify: `GGlearn/tests/textbookGeneration.test.ts`
  - 增加教学画像、蓝图、section evidence、校验和 two-stage orchestration 测试。
- Create: `GGlearn/tests/textbookBodyLayout.test.ts`
  - 验证阅读页正文分组逻辑，不需要浏览器环境。

本计划刻意不做：

- 不新增依赖。
- 不引入向量数据库。
- 不迁移历史项目数据。
- 不重做完整教材编辑器。
- 不把 citation/source refs 做成前台主功能。

---

### Task 1: 锁定共享类型和教学画像契约

**Files:**
- Modify: `GGlearn/src/types.ts`
- Create: `GGlearn/src/lib/ai/textbookBody.ts`
- Modify: `GGlearn/tests/textbookGeneration.test.ts`

- [ ] **Step 1: 写失败测试，锁定 `createTeachingProfile(...)` 的三种学习目标差异**

Add this import block to `GGlearn/tests/textbookGeneration.test.ts`:

```ts
import { createTeachingProfile } from '../src/lib/ai/textbookBody';
```

Add these tests near the existing deterministic helper tests:

```ts
test('createTeachingProfile maps mastery to a rigorous continuous textbook profile', () => {
  const profile = createTeachingProfile('mastery', {
    topic: 'Linear Algebra',
    objective: 'Understand eigenvalues deeply',
    priorKnowledgeLevel: 'basic',
    explanationPreference: 'rigorous',
    defaultProfile: 'general-adult-beginner',
  });

  assert.equal(profile.goal, 'mastery');
  assert.equal(profile.rigorLevel, 'high');
  assert.ok(profile.formalismWeight > profile.practiceWeight);
  assert.ok(profile.requiredBodyRoles.includes('definition'));
  assert.ok(profile.requiredBodyRoles.includes('derivation'));
  assert.ok(profile.requiredBodyRoles.includes('worked-example'));
  assert.equal(profile.columnDensity, 'low');
});

test('createTeachingProfile maps exam to explicit practice and mistake coverage', () => {
  const profile = createTeachingProfile('exam', {
    topic: 'Linear Algebra',
    objective: 'Prepare for an exam',
    priorKnowledgeLevel: 'rusty',
    explanationPreference: 'example-first',
    defaultProfile: 'general-adult-beginner',
  });

  assert.equal(profile.goal, 'exam');
  assert.equal(profile.rigorLevel, 'exam-focused');
  assert.ok(profile.practiceWeight > profile.formalismWeight);
  assert.ok(profile.requiredBodyRoles.includes('common-mistake'));
  assert.ok(profile.requiredBodyRoles.includes('practice'));
  assert.ok(profile.requiredBodyRoles.includes('mastery-check'));
  assert.equal(profile.columnDensity, 'high');
});

test('createTeachingProfile maps micro to a compact body sequence', () => {
  const profile = createTeachingProfile('micro', {
    topic: 'Eigenvalues',
    objective: 'Get the core idea quickly',
    priorKnowledgeLevel: 'solid',
    explanationPreference: 'intuitive',
    defaultProfile: 'general-adult-beginner',
  });

  assert.equal(profile.goal, 'micro');
  assert.equal(profile.rigorLevel, 'balanced');
  assert.ok(profile.targetBodyBlockCount <= 5);
  assert.deepEqual(profile.requiredBodyRoles, ['motivation', 'concept', 'worked-example', 'mastery-check']);
  assert.equal(profile.columnDensity, 'compact');
});
```

- [ ] **Step 2: 运行测试，确认因为新模块不存在而失败**

Run:

```bash
cd GGlearn && node --import tsx --test tests/textbookGeneration.test.ts --test-name-pattern "createTeachingProfile"
```

Expected:

```text
not ok ... Cannot find module '../src/lib/ai/textbookBody'
```

- [ ] **Step 3: 在 `GGlearn/src/types.ts` 增加正文结构类型**

Add these exports after `export type ExplanationPreference = ...`:

```ts
export type ContentRole =
  | 'motivation'
  | 'intuition'
  | 'concept'
  | 'definition'
  | 'derivation'
  | 'worked-example'
  | 'practice'
  | 'common-mistake'
  | 'summary'
  | 'mastery-check';

export interface TeachingProfile {
  goal: LearningMode;
  rigorLevel: 'balanced' | 'high' | 'exam-focused';
  intuitionWeight: number;
  formalismWeight: number;
  practiceWeight: number;
  targetBodyBlockCount: number;
  columnDensity: 'low' | 'medium' | 'high' | 'compact';
  requiredBodyRoles: ContentRole[];
}

export interface ChapterSectionBlueprint {
  id: string;
  title: string;
  teachingRole: ContentRole;
  dependsOnSections: string[];
  focusConcepts: string[];
  transitionFromPrevious: string;
  expectedBodyRoles: ContentRole[];
}

export interface ChapterBlueprint {
  chapterTitle: string;
  chapterGoal: string;
  targetReaderState: string;
  teachingProfile: TeachingProfile;
  chapterFlowNarrative: string;
  sections: ChapterSectionBlueprint[];
  endState: string;
  unsupportedGaps: string[];
}

export interface SectionGenerationInput {
  textbookTitle: string;
  chapterGoal: string;
  chapterFlowNarrative: string;
  currentSection: ChapterSectionBlueprint;
  resolvedPrerequisites: string[];
  previousSectionSummary: string;
  selectedEvidencePacks: EvidencePack[];
  teachingProfile: TeachingProfile;
}

export interface GeneratedSectionValidation {
  hasEvidenceSupport: boolean;
  unsupportedClaims: string[];
  transitionIssues: string[];
  prerequisiteIssues: string[];
  sourceLeakIssues: string[];
  bodyQualityIssues: string[];
}

export interface GeneratedChapterValidation {
  sections: Array<{
    sectionId: string;
    validation: GeneratedSectionValidation;
  }>;
  hasBlockingIssues: boolean;
}
```

Extend `Chunk.metadata` from:

```ts
  metadata?: {
    imagePrompt?: string;
    answerKey?: string;
    hint?: string;
    diagramCode?: string;
  };
```

to:

```ts
  metadata?: {
    imagePrompt?: string;
    answerKey?: string;
    hint?: string;
    diagramCode?: string;
    isTextbookBody?: boolean;
    chapterTitle?: string;
    sectionId?: string;
    sectionTitle?: string;
    contentRole?: ContentRole;
    validationWarnings?: string[];
  };
```

- [ ] **Step 4: 创建 `GGlearn/src/lib/ai/textbookBody.ts`，实现最小教学画像 helper**

Create the file with:

```ts
import type { ContentRole, LearningBrief, LearningMode, TeachingProfile } from '../../types';

function clampWeight(value: number): number {
  return Math.max(0, Math.min(1, Number(value.toFixed(2))));
}

export function createTeachingProfile(goal: LearningMode, learningBrief?: LearningBrief): TeachingProfile {
  const priorKnowledge = learningBrief?.priorKnowledgeLevel ?? 'basic';
  const explanationPreference = learningBrief?.explanationPreference ?? 'textbook';

  if (goal === 'exam') {
    const practiceBoost = explanationPreference === 'example-first' ? 0.1 : 0;
    return {
      goal,
      rigorLevel: 'exam-focused',
      intuitionWeight: clampWeight(priorKnowledge === 'none' ? 0.45 : 0.35),
      formalismWeight: clampWeight(0.35),
      practiceWeight: clampWeight(0.75 + practiceBoost),
      targetBodyBlockCount: 7,
      columnDensity: 'high',
      requiredBodyRoles: ['motivation', 'definition', 'worked-example', 'common-mistake', 'practice', 'mastery-check', 'summary'],
    };
  }

  if (goal === 'micro') {
    return {
      goal,
      rigorLevel: 'balanced',
      intuitionWeight: clampWeight(explanationPreference === 'intuitive' ? 0.75 : 0.6),
      formalismWeight: clampWeight(priorKnowledge === 'solid' ? 0.45 : 0.35),
      practiceWeight: clampWeight(0.45),
      targetBodyBlockCount: 4,
      columnDensity: 'compact',
      requiredBodyRoles: ['motivation', 'concept', 'worked-example', 'mastery-check'],
    };
  }

  const rigorous = explanationPreference === 'rigorous';
  const weakFoundation = priorKnowledge === 'none' || priorKnowledge === 'rusty';
  const requiredBodyRoles: ContentRole[] = [
    'motivation',
    'intuition',
    'definition',
    'derivation',
    'worked-example',
    'practice',
    'summary',
  ];

  return {
    goal,
    rigorLevel: rigorous ? 'high' : 'balanced',
    intuitionWeight: clampWeight(weakFoundation ? 0.7 : 0.55),
    formalismWeight: clampWeight(rigorous ? 0.8 : 0.6),
    practiceWeight: clampWeight(0.45),
    targetBodyBlockCount: 8,
    columnDensity: 'low',
    requiredBodyRoles,
  };
}
```

- [ ] **Step 5: 运行测试，确认教学画像通过**

Run:

```bash
cd GGlearn && node --import tsx --test tests/textbookGeneration.test.ts --test-name-pattern "createTeachingProfile"
```

Expected:

```text
# pass 3
# fail 0
```

- [ ] **Step 6: 提交共享类型与教学画像**

Run:

```bash
git add GGlearn/src/types.ts GGlearn/src/lib/ai/textbookBody.ts GGlearn/tests/textbookGeneration.test.ts
git commit -m "Define the textbook body generation contract" -m "Introduce the teaching profile and body metadata types before changing runtime generation so the new textbook-body semantics are executable and reviewable.

Constraint: Keep the existing Chunk storage shape compatible
Rejected: Replace chunks with a new persisted document tree immediately | migration risk is too high for the first quality pass
Confidence: high
Scope-risk: narrow
Directive: Treat generated examples, exercises, and summaries as textbook body content when metadata.isTextbookBody is true
Tested: node --import tsx --test tests/textbookGeneration.test.ts --test-name-pattern \"createTeachingProfile\"
Not-tested: Full GGlearn test suite"
```

---

### Task 2: 实现蓝图、section evidence 和后台校验 helper

**Files:**
- Modify: `GGlearn/src/lib/ai/textbookBody.ts`
- Modify: `GGlearn/tests/textbookGeneration.test.ts`

- [ ] **Step 1: 写失败测试，锁定 fallback blueprint 的教学顺序**

Add these imports to the `textbookBody` import in `GGlearn/tests/textbookGeneration.test.ts`:

```ts
import {
  createFallbackChapterBlueprint,
  createTeachingProfile,
  selectEvidencePacksForSection,
  validateGeneratedSection,
} from '../src/lib/ai/textbookBody';
```

Replace the earlier single-function import with this grouped import.

Add this test:

```ts
test('createFallbackChapterBlueprint creates a body-first teaching sequence', () => {
  const { asset } = createAsset();
  const evidencePacks = buildEvidencePacks([asset], 'Eigenvalues');
  const profile = createTeachingProfile('mastery', {
    topic: 'Linear Algebra',
    objective: 'Understand eigenvalues',
    priorKnowledgeLevel: 'basic',
    explanationPreference: 'textbook',
    defaultProfile: 'general-adult-beginner',
  });

  const blueprint = createFallbackChapterBlueprint('Linear Algebra', evidencePacks, profile, 'Eigenvalues');

  assert.equal(blueprint.chapterTitle, 'Eigenvalues');
  assert.equal(blueprint.teachingProfile.goal, 'mastery');
  assert.ok(blueprint.chapterFlowNarrative.length > 20);
  assert.ok(blueprint.sections.length >= 4);
  assert.equal(blueprint.sections[0].teachingRole, 'motivation');
  assert.ok(blueprint.sections.some((section) => section.expectedBodyRoles.includes('definition')));
  assert.ok(blueprint.sections.some((section) => section.expectedBodyRoles.includes('practice')));
});
```

- [ ] **Step 2: 写失败测试，锁定 section evidence 选择**

Add this test:

```ts
test('selectEvidencePacksForSection prioritizes focus concepts without overloading the section', () => {
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
      expectedBodyRoles: ['definition', 'concept'],
    },
    evidencePacks
  );

  assert.ok(selected.length >= 1);
  assert.ok(selected.length <= 4);
  assert.ok(
    selected.some((pack) => {
      const haystack = `${pack.retrievalUnitTitle} ${pack.summary} ${pack.excerpt} ${pack.conceptRefs.join(' ')}`.toLowerCase();
      return haystack.includes('eigenvalue') || haystack.includes('invariant');
    })
  );
});
```

- [ ] **Step 3: 写失败测试，锁定后台校验不暴露到前台但能发现问题**

Add this test:

```ts
test('validateGeneratedSection flags source leaks, missing evidence, and weak body quality', () => {
  const validation = validateGeneratedSection(
    {
      id: 'section-definition',
      title: 'What Eigenvalues Mean',
      teachingRole: 'definition',
      dependsOnSections: ['section-intuition'],
      transitionFromPrevious: 'Connect this section to the earlier intuition.',
      focusConcepts: ['Eigenvalues'],
      expectedBodyRoles: ['definition', 'concept'],
    },
    [
      {
        id: 'chunk-1',
        type: 'text',
        title: 'What Eigenvalues Mean',
        content: 'According to the source, a matrix is always diagonalizable over the reals.',
        depth: 'technical',
        metadata: {
          isTextbookBody: true,
          sectionId: 'section-definition',
          contentRole: 'definition',
        },
      },
    ],
    []
  );

  assert.equal(validation.hasEvidenceSupport, false);
  assert.ok(validation.unsupportedClaims.length >= 1);
  assert.ok(validation.transitionIssues.length >= 1);
  assert.ok(validation.sourceLeakIssues.length >= 1);
  assert.ok(validation.bodyQualityIssues.length >= 1);
});
```

- [ ] **Step 4: 运行测试，确认因为缺少导出而失败**

Run:

```bash
cd GGlearn && node --import tsx --test tests/textbookGeneration.test.ts --test-name-pattern "createFallbackChapterBlueprint|selectEvidencePacksForSection|validateGeneratedSection"
```

Expected:

```text
not ok ... does not provide an export named
```

- [ ] **Step 5: 在 `GGlearn/src/lib/ai/textbookBody.ts` 添加确定性 helper**

Append this code after `createTeachingProfile(...)`:

```ts
import type {
  ChapterBlueprint,
  ChapterSectionBlueprint,
  Chunk,
  ChunkSourceReference,
  EvidencePack,
  GeneratedSectionValidation,
  TeachingProfile,
} from '../../types';

function tokenizeForMatch(text: string): string[] {
  return text
    .toLowerCase()
    .replace(/[^\p{L}\p{N}\s]/gu, ' ')
    .split(/\s+/)
    .filter((token) => token.length >= 3);
}

function dedupePreserveOrder(values: string[]): string[] {
  const seen = new Set<string>();
  const result: string[] = [];

  values.forEach((value) => {
    const normalized = value.trim();
    if (!normalized) {
      return;
    }

    const key = normalized.toLowerCase();
    if (seen.has(key)) {
      return;
    }

    seen.add(key);
    result.push(normalized);
  });

  return result;
}

function scoreEvidencePackForSection(section: ChapterSectionBlueprint, pack: EvidencePack): number {
  let score = pack.teachingValue === 'core' ? 6 : pack.teachingValue === 'supporting' ? 4 : 2;
  score += pack.confidence === 'high' ? 3 : pack.confidence === 'medium' ? 2 : 1;

  const sectionTokens = tokenizeForMatch(`${section.title} ${section.focusConcepts.join(' ')}`);
  const haystack = `${pack.retrievalUnitTitle} ${pack.summary} ${pack.excerpt} ${pack.conceptRefs.join(' ')}`.toLowerCase();

  sectionTokens.forEach((token) => {
    if (haystack.includes(token)) {
      score += 4;
    }
  });

  return score;
}

function createSection(
  id: string,
  title: string,
  teachingRole: ChapterSectionBlueprint['teachingRole'],
  expectedBodyRoles: ChapterSectionBlueprint['expectedBodyRoles'],
  focusConcepts: string[],
  dependsOnSections: string[],
  transitionFromPrevious: string
): ChapterSectionBlueprint {
  return {
    id,
    title,
    teachingRole,
    dependsOnSections,
    focusConcepts,
    transitionFromPrevious,
    expectedBodyRoles,
  };
}

export function createFallbackChapterBlueprint(
  title: string,
  evidencePacks: EvidencePack[],
  teachingProfile: TeachingProfile,
  chapterTopic?: string
): ChapterBlueprint {
  const chapterTitle = chapterTopic?.trim() || title;
  const focusConcepts = dedupePreserveOrder(
    evidencePacks.flatMap((pack) => [pack.retrievalUnitTitle, ...pack.conceptRefs])
  ).slice(0, 5);
  const primaryConcept = focusConcepts[0] || chapterTitle;

  const sections: ChapterSectionBlueprint[] = teachingProfile.goal === 'exam'
    ? [
        createSection('section-purpose', `Why ${chapterTitle} matters for problems`, 'motivation', ['motivation', 'concept'], [primaryConcept], [], 'Open with the problem this chapter helps solve.'),
        createSection('section-core', `Core rules for ${chapterTitle}`, 'definition', ['definition', 'concept'], focusConcepts.slice(0, 3), ['section-purpose'], 'Turn the motivation into precise rules and boundaries.'),
        createSection('section-method', `How to use ${chapterTitle}`, 'worked-example', ['worked-example', 'practice'], focusConcepts.slice(0, 4), ['section-core'], 'Apply the core rules in a worked problem.'),
        createSection('section-mistakes', `Common mistakes with ${chapterTitle}`, 'common-mistake', ['common-mistake', 'mastery-check'], focusConcepts.slice(0, 4), ['section-method'], 'Separate correct use from common exam traps.'),
        createSection('section-summary', `Fast review of ${chapterTitle}`, 'summary', ['summary', 'practice'], focusConcepts.slice(0, 5), ['section-mistakes'], 'Close with recall and a short check.'),
      ]
    : teachingProfile.goal === 'micro'
      ? [
          createSection('section-purpose', `Why ${chapterTitle} matters`, 'motivation', ['motivation'], [primaryConcept], [], 'Start from the reason this small topic is worth learning.'),
          createSection('section-core', `The core idea`, 'concept', ['concept', 'worked-example'], focusConcepts.slice(0, 3), ['section-purpose'], 'Move directly from motivation into the smallest usable explanation.'),
          createSection('section-check', `Check your understanding`, 'mastery-check', ['mastery-check'], focusConcepts.slice(0, 3), ['section-core'], 'End with one compact check.'),
        ]
      : [
          createSection('section-purpose', `Why ${chapterTitle} matters`, 'motivation', ['motivation', 'intuition'], [primaryConcept], [], 'Open with the problem and intuition before formal language.'),
          createSection('section-intuition', `The intuition behind ${chapterTitle}`, 'intuition', ['intuition', 'concept'], focusConcepts.slice(0, 3), ['section-purpose'], 'Make the informal idea stable enough for a definition.'),
          createSection('section-definition', `Defining ${chapterTitle}`, 'definition', ['definition', 'concept'], focusConcepts.slice(0, 4), ['section-intuition'], 'Turn the intuition into precise textbook language.'),
          createSection('section-explanation', `How ${chapterTitle} works`, 'derivation', ['derivation', 'worked-example'], focusConcepts.slice(0, 5), ['section-definition'], 'Explain the mechanism and show it in use.'),
          createSection('section-practice', `Using ${chapterTitle}`, 'practice', ['practice', 'summary'], focusConcepts.slice(0, 5), ['section-explanation'], 'Let the learner test the idea and close the chapter.'),
        ];

  return {
    chapterTitle,
    chapterGoal: `Understand ${chapterTitle} well enough to explain the idea, use it in context, and check common misunderstandings.`,
    targetReaderState: 'The reader has the project-level background but has not yet organized this topic into a reliable learning path.',
    teachingProfile,
    chapterFlowNarrative: `The chapter starts from why ${chapterTitle} matters, builds intuition before formal language, then turns the idea into usable textbook knowledge with examples and checks.`,
    sections,
    endState: `The reader can explain ${chapterTitle}, recognize its boundaries, and apply the idea in a focused problem.`,
    unsupportedGaps: [],
  };
}

export function selectEvidencePacksForSection(
  section: ChapterSectionBlueprint,
  evidencePacks: EvidencePack[]
): EvidencePack[] {
  return [...evidencePacks]
    .map((pack) => ({ pack, score: scoreEvidencePackForSection(section, pack) }))
    .sort((left, right) => right.score - left.score)
    .slice(0, 4)
    .map((entry) => entry.pack);
}

export function validateGeneratedSection(
  section: ChapterSectionBlueprint,
  chunks: Chunk[],
  sourceRefs: ChunkSourceReference[]
): GeneratedSectionValidation {
  const combined = chunks.map((chunk) => `${chunk.title || ''}\n${chunk.content}`).join('\n\n');
  const lower = combined.toLowerCase();
  const hasEvidenceSupport = sourceRefs.length > 0 || chunks.some((chunk) => (chunk.sourceRefs?.length ?? 0) > 0);
  const sourceLeakPatterns = [/according to (the )?source/i, /the source (states|says|notes)/i, /根据(资料|来源|原文)/, /本文(指出|认为|提到)/];
  const unsupportedClaimPatterns = [/always diagonalizable over the reals/i, /always true/i, /never fails/i, /必然成立/, /一定正确/];
  const hasTransition = !section.dependsOnSections.length || lower.includes('therefore') || lower.includes('now') || lower.includes('因此') || lower.includes('接下来') || lower.includes('前面');
  const expectedRoleCovered = section.expectedBodyRoles.some((role) =>
    chunks.some((chunk) => chunk.metadata?.contentRole === role || lower.includes(role.replace('-', ' ')))
  );

  return {
    hasEvidenceSupport,
    unsupportedClaims: unsupportedClaimPatterns
      .filter((pattern) => pattern.test(combined))
      .map((pattern) => `Potential unsupported absolute claim matched ${pattern.toString()}`),
    transitionIssues: hasTransition ? [] : [`Section ${section.id} does not connect to its prerequisite sections.`],
    prerequisiteIssues: section.focusConcepts.length && combined.trim().length < 160
      ? [`Section ${section.id} is too short to establish its focus concepts.`]
      : [],
    sourceLeakIssues: sourceLeakPatterns
      .filter((pattern) => pattern.test(combined))
      .map((pattern) => `Source-facing phrase matched ${pattern.toString()}`),
    bodyQualityIssues: expectedRoleCovered && combined.length >= 220
      ? []
      : [`Section ${section.id} does not yet read like complete textbook body content.`],
  };
}
```

After appending, merge the duplicate type imports at the top of `textbookBody.ts` into one import:

```ts
import type {
  ChapterBlueprint,
  ChapterSectionBlueprint,
  Chunk,
  ChunkSourceReference,
  ContentRole,
  EvidencePack,
  GeneratedSectionValidation,
  LearningBrief,
  LearningMode,
  TeachingProfile,
} from '../../types';
```

- [ ] **Step 6: 运行 helper 测试，确认通过**

Run:

```bash
cd GGlearn && node --import tsx --test tests/textbookGeneration.test.ts --test-name-pattern "createFallbackChapterBlueprint|selectEvidencePacksForSection|validateGeneratedSection|createTeachingProfile"
```

Expected:

```text
# fail 0
```

- [ ] **Step 7: 提交蓝图、证据选择和校验 helper**

Run:

```bash
git add GGlearn/src/lib/ai/textbookBody.ts GGlearn/tests/textbookGeneration.test.ts
git commit -m "Add deterministic textbook body planning helpers" -m "Build the non-network part of the textbook-body pipeline first so section order, evidence selection, and backend quality signals are covered before provider orchestration changes.

Constraint: Source grounding remains a backend quality mechanism
Rejected: Show evidence sufficiency directly in the reader | this would make users audit sources instead of reading a textbook
Confidence: high
Scope-risk: narrow
Directive: Keep these helpers deterministic so generation behavior has stable tests around the model calls
Tested: node --import tsx --test tests/textbookGeneration.test.ts --test-name-pattern \"createFallbackChapterBlueprint|selectEvidencePacksForSection|validateGeneratedSection|createTeachingProfile\"
Not-tested: Full GGlearn test suite"
```

---

### Task 3: 增加可注入的 two-stage generation orchestration

**Files:**
- Modify: `GGlearn/src/lib/ai/textbookGeneration.ts`
- Modify: `GGlearn/tests/textbookGeneration.test.ts`

- [ ] **Step 1: 写失败测试，证明 orchestration 按蓝图逐节生成正文**

Add these imports to `GGlearn/tests/textbookGeneration.test.ts`:

```ts
import type { ChapterBlueprint, LearningBrief } from '../src/types';
import { generateTextbookChunksWithProvider } from '../src/lib/ai/textbookGeneration';
```

If `textbookGeneration` imports are already grouped, include `generateTextbookChunksWithProvider` in that group.

Add this test:

```ts
test('generateTextbookChunksWithProvider generates textbook body chunks section by section', async () => {
  const { asset } = createAsset();
  const learningBrief: LearningBrief = {
    topic: 'Linear Algebra',
    objective: 'Understand eigenvalues',
    priorKnowledgeLevel: 'basic',
    explanationPreference: 'textbook',
    defaultProfile: 'general-adult-beginner',
  };
  const sectionCalls: string[] = [];

  const chunks = await generateTextbookChunksWithProvider(
    {
      title: 'Linear Algebra',
      assets: [asset],
      goal: 'mastery',
      customFocus: '',
      language: 'en',
      attachments: [],
      chapterTopic: 'Eigenvalues',
      learningBrief,
    },
    {
      createBlueprint: async ({ fallbackBlueprint }) => fallbackBlueprint,
      generateSection: async ({ section, selectedEvidencePacks }) => {
        sectionCalls.push(section.id);
        return [
          {
            type: 'text',
            title: section.title,
            content: `This textbook section explains ${section.title} as part of a continuous chapter. It avoids source-facing language and includes enough detail for the learner to follow the idea before moving on.`,
            depth: 'conceptual',
            contentRole: section.expectedBodyRoles[0],
            sourceRefIds: selectedEvidencePacks.slice(0, 1).map((pack) => pack.id),
          },
        ];
      },
    }
  );

  assert.ok(chunks.length >= 4);
  assert.deepEqual(sectionCalls, ['section-purpose', 'section-intuition', 'section-definition', 'section-explanation', 'section-practice']);
  assert.ok(chunks.every((chunk) => chunk.metadata?.isTextbookBody === true));
  assert.ok(chunks.every((chunk) => chunk.metadata?.sectionId));
  assert.ok(chunks.every((chunk) => chunk.sourceRefs && chunk.sourceRefs.length >= 1));
});
```

- [ ] **Step 2: 运行测试，确认因为 orchestration 导出缺失而失败**

Run:

```bash
cd GGlearn && node --import tsx --test tests/textbookGeneration.test.ts --test-name-pattern "generateTextbookChunksWithProvider"
```

Expected:

```text
not ok ... does not provide an export named 'generateTextbookChunksWithProvider'
```

- [ ] **Step 3: 在 `GGlearn/src/lib/ai/textbookGeneration.ts` 扩展 imports**

Change the type import at the top from:

```ts
import type { AIConfig, Attachment, Chunk, ChunkSourceReference, EvidencePack, ProjectSourceAsset } from '../../types';
```

to:

```ts
import type {
  AIConfig,
  Attachment,
  ChapterBlueprint,
  ChapterSectionBlueprint,
  Chunk,
  ChunkSourceReference,
  ContentRole,
  EvidencePack,
  LearningBrief,
  LearningMode,
  ProjectSourceAsset,
  SectionGenerationInput,
} from '../../types';
```

Add this import below the existing source transform import:

```ts
import {
  createFallbackChapterBlueprint,
  createTeachingProfile,
  selectEvidencePacksForSection,
  validateGeneratedSection,
} from './textbookBody';
```

- [ ] **Step 4: 在 `textbookGeneration.ts` 增加 provider types 和 orchestration 输入**

Add this code after `type GeneratedChunkDraft = ...`:

```ts
type SectionChunkDraft = GeneratedChunkDraft & {
  contentRole?: ContentRole;
};

interface GenerateTextbookChunksInput {
  title: string;
  assets: ProjectSourceAsset[];
  goal: LearningMode;
  customFocus: string;
  language: 'zh' | 'en';
  attachments: Attachment[];
  chapterTopic?: string;
  learningBrief?: LearningBrief;
}

interface BlueprintProviderInput {
  title: string;
  assets: ProjectSourceAsset[];
  evidencePacks: EvidencePack[];
  fallbackBlueprint: ChapterBlueprint;
  customFocus: string;
  language: 'zh' | 'en';
  chapterTopic?: string;
}

interface SectionProviderInput extends SectionGenerationInput {
  title: string;
  assets: ProjectSourceAsset[];
  customFocus: string;
  language: 'zh' | 'en';
  attachments: Attachment[];
}

interface TextbookChunkProvider {
  createBlueprint(input: BlueprintProviderInput): Promise<ChapterBlueprint>;
  generateSection(input: SectionProviderInput): Promise<SectionChunkDraft[]>;
}
```

- [ ] **Step 5: 添加 section chunk normalization helper**

Add this code before the existing `normalizeGeneratedChunks(...)` function:

```ts
function normalizeGeneratedSectionChunks(
  drafts: SectionChunkDraft[],
  evidencePacks: EvidencePack[],
  section: ChapterSectionBlueprint,
  chapterTitle: string,
  validationWarnings: string[]
): Chunk[] {
  return drafts.map((draft, index) => {
    const metadata = {
      ...draft.metadata,
      isTextbookBody: true,
      chapterTitle,
      sectionId: section.id,
      sectionTitle: section.title,
      contentRole: draft.contentRole ?? section.expectedBodyRoles[index] ?? section.teachingRole,
      validationWarnings,
    };

    return {
      type: draft.type === 'exercise' || draft.type === 'summary' || draft.type === 'note' ? 'text' : draft.type,
      title: draft.title,
      content: draft.content,
      depth: draft.depth,
      metadata,
      id: `chunk-${Date.now()}-${section.id}-${index}`,
      sourceRefs: resolveSourceReferences(draft, evidencePacks),
    };
  });
}

function summarizeSectionChunks(chunks: Chunk[]): string {
  return chunks
    .map((chunk) => `${chunk.title || ''}: ${chunk.content}`)
    .join('\n')
    .slice(0, 700);
}

function flattenValidationWarnings(validation: ReturnType<typeof validateGeneratedSection>): string[] {
  return [
    ...validation.unsupportedClaims,
    ...validation.transitionIssues,
    ...validation.prerequisiteIssues,
    ...validation.sourceLeakIssues,
    ...validation.bodyQualityIssues,
  ];
}
```

- [ ] **Step 6: 添加 `generateTextbookChunksWithProvider(...)`**

Add this exported function before the existing `generateTextbookChunks(...)`:

```ts
export async function generateTextbookChunksWithProvider(
  input: GenerateTextbookChunksInput,
  provider: TextbookChunkProvider
): Promise<Chunk[]> {
  const evidencePacks = buildEvidencePacks(input.assets, input.chapterTopic);
  const teachingProfile = createTeachingProfile(input.goal, input.learningBrief);
  const fallbackBlueprint = createFallbackChapterBlueprint(
    input.title,
    evidencePacks,
    teachingProfile,
    input.chapterTopic
  );
  const blueprint = await provider.createBlueprint({
    title: input.title,
    assets: input.assets,
    evidencePacks,
    fallbackBlueprint,
    customFocus: input.customFocus,
    language: input.language,
    chapterTopic: input.chapterTopic,
  });

  const generatedChunks: Chunk[] = [];
  const resolvedPrerequisites: string[] = [];
  let previousSectionSummary = '';

  for (const section of blueprint.sections) {
    const selectedEvidencePacks = selectEvidencePacksForSection(section, evidencePacks);
    const drafts = await provider.generateSection({
      title: input.title,
      assets: input.assets,
      customFocus: input.customFocus,
      language: input.language,
      attachments: input.attachments,
      textbookTitle: input.title,
      chapterGoal: blueprint.chapterGoal,
      chapterFlowNarrative: blueprint.chapterFlowNarrative,
      currentSection: section,
      resolvedPrerequisites,
      previousSectionSummary,
      selectedEvidencePacks,
      teachingProfile: blueprint.teachingProfile,
    });

    const preliminaryChunks = normalizeGeneratedSectionChunks(
      drafts,
      selectedEvidencePacks,
      section,
      blueprint.chapterTitle,
      []
    );
    const sectionRefs = preliminaryChunks.flatMap((chunk) => chunk.sourceRefs ?? []);
    const validation = validateGeneratedSection(section, preliminaryChunks, sectionRefs);
    const warnings = flattenValidationWarnings(validation);
    const normalizedChunks = preliminaryChunks.map((chunk) => ({
      ...chunk,
      metadata: {
        ...chunk.metadata,
        validationWarnings: warnings,
      },
    }));

    generatedChunks.push(...normalizedChunks);
    resolvedPrerequisites.push(...section.focusConcepts);
    previousSectionSummary = summarizeSectionChunks(normalizedChunks);
  }

  return generatedChunks;
}
```

- [ ] **Step 7: 运行 orchestration 测试，确认通过**

Run:

```bash
cd GGlearn && node --import tsx --test tests/textbookGeneration.test.ts --test-name-pattern "generateTextbookChunksWithProvider"
```

Expected:

```text
# fail 0
```

- [ ] **Step 8: 提交 two-stage orchestration**

Run:

```bash
git add GGlearn/src/lib/ai/textbookGeneration.ts GGlearn/tests/textbookGeneration.test.ts
git commit -m "Orchestrate textbook generation through section body passes" -m "Add a provider-injected generation path that builds a chapter blueprint, generates each section as textbook body content, and records backend validation warnings in chunk metadata.

Constraint: Public generation callers still receive Chunk[]
Rejected: Generate the whole chapter in one model call | this is the source of weak structure and summary-like output
Confidence: high
Scope-risk: moderate
Directive: Keep generateTextbookChunksWithProvider network-free so section orchestration remains testable
Tested: node --import tsx --test tests/textbookGeneration.test.ts --test-name-pattern \"generateTextbookChunksWithProvider\"
Not-tested: Real Gemini/OpenAI provider calls"
```

---

### Task 4: 接入真实 Gemini/OpenAI provider，并让 App 传入 learningBrief

**Files:**
- Modify: `GGlearn/src/lib/ai/textbookGeneration.ts`
- Modify: `GGlearn/src/App.tsx`
- Modify: `GGlearn/tests/textbookGeneration.test.ts`

- [ ] **Step 1: 写测试，锁定公开入口可以接收可选 `learningBrief` 而不破坏旧调用**

Add this type-only compile guard test to `GGlearn/tests/textbookGeneration.test.ts`:

```ts
test('generateTextbookChunks accepts learningBrief as a backward-compatible optional argument', () => {
  type GenerateArgs = Parameters<typeof import('../src/lib/ai/textbookGeneration').generateTextbookChunks>;
  const legacyArgCount: GenerateArgs['length'] = 8;
  const supportsOptionalLearningBrief = legacyArgCount >= 8;

  assert.equal(supportsOptionalLearningBrief, true);
});
```

- [ ] **Step 2: 运行测试，确认当前类型仍能编译**

Run:

```bash
cd GGlearn && node --import tsx --test tests/textbookGeneration.test.ts --test-name-pattern "generateTextbookChunks accepts"
```

Expected:

```text
# fail 0
```

- [ ] **Step 3: 添加 prompt builders**

In `GGlearn/src/lib/ai/textbookGeneration.ts`, add this code before `generateTextbookChunks(...)`:

```ts
function buildBlueprintPrompt(
  title: string,
  evidencePacks: EvidencePack[],
  fallbackBlueprint: ChapterBlueprint,
  customFocus: string,
  language: 'zh' | 'en',
  chapterTopic?: string
): string {
  return [
    language === 'zh'
      ? `请为《${title}》生成后台章节教学蓝图，不要生成正文。`
      : `Create a backend chapter teaching blueprint for "${title}". Do not write the body yet.`,
    chapterTopic ? `Chapter topic: ${chapterTopic}` : '',
    customFocus ? `User focus: ${customFocus}` : '',
    `Teaching profile: ${JSON.stringify(fallbackBlueprint.teachingProfile)}`,
    `Fallback section order: ${JSON.stringify(fallbackBlueprint.sections)}`,
    `Evidence packs:\n${buildEvidencePrompt(evidencePacks)}`,
    language === 'zh'
      ? '返回 JSON，字段必须包含 chapterTitle, chapterGoal, targetReaderState, chapterFlowNarrative, sections, endState, unsupportedGaps。sections 必须保留 id, title, teachingRole, dependsOnSections, transitionFromPrevious, focusConcepts, expectedBodyRoles。'
      : 'Return JSON with chapterTitle, chapterGoal, targetReaderState, chapterFlowNarrative, sections, endState, unsupportedGaps. Each section must include id, title, teachingRole, dependsOnSections, transitionFromPrevious, focusConcepts, expectedBodyRoles.',
  ].filter(Boolean).join('\n\n');
}

function buildSectionPrompt(input: SectionProviderInput): string {
  const languageRule = input.language === 'zh'
    ? '请使用中文写成连续教材正文。'
    : 'Write continuous textbook body content in English.';

  return [
    languageRule,
    `Textbook title: ${input.textbookTitle}`,
    `Chapter goal: ${input.chapterGoal}`,
    `Chapter flow: ${input.chapterFlowNarrative}`,
    `Current section: ${JSON.stringify(input.currentSection)}`,
    `Resolved prerequisites: ${input.resolvedPrerequisites.join(', ') || 'none'}`,
    `Previous section summary: ${input.previousSectionSummary || 'This is the first section.'}`,
    `Teaching profile: ${JSON.stringify(input.teachingProfile)}`,
    input.customFocus ? `User focus: ${input.customFocus}` : '',
    `Selected evidence packs:\n${buildEvidencePrompt(input.selectedEvidencePacks)}`,
    input.language === 'zh'
      ? [
          '写作契约：',
          '- 输出必须是教材正文，不是资料摘要。',
          '- 不要写“根据资料”“来源指出”“本文认为”等来源泄露式表达。',
          '- 例题、练习、总结都属于正文。',
          '- 不要伪造证据未支持的具体结论。',
          '- 返回 JSON 数组，每项包含 type, title, content, depth, contentRole, sourceRefIds, metadata。',
        ].join('\n')
      : [
          'Writing contract:',
          '- Output textbook body content, not a source summary.',
          '- Do not write source-facing phrases such as "according to the source".',
          '- Examples, practice, and summaries are body content.',
          '- Do not invent specific claims outside the selected evidence.',
          '- Return a JSON array. Each item includes type, title, content, depth, contentRole, sourceRefIds, metadata.',
        ].join('\n'),
  ].filter(Boolean).join('\n\n');
}
```

- [ ] **Step 4: 添加 real provider factory**

Add this code before `generateTextbookChunks(...)`:

```ts
function normalizeBlueprintCandidate(candidate: Partial<ChapterBlueprint>, fallbackBlueprint: ChapterBlueprint): ChapterBlueprint {
  const sections = Array.isArray(candidate.sections) && candidate.sections.length > 0
    ? candidate.sections.map((section, index) => ({
        ...fallbackBlueprint.sections[Math.min(index, fallbackBlueprint.sections.length - 1)],
        ...section,
        id: section.id || fallbackBlueprint.sections[index]?.id || `section-${index}`,
        expectedBodyRoles: section.expectedBodyRoles?.length
          ? section.expectedBodyRoles
          : fallbackBlueprint.sections[index]?.expectedBodyRoles ?? ['concept'],
        focusConcepts: section.focusConcepts?.length
          ? section.focusConcepts
          : fallbackBlueprint.sections[index]?.focusConcepts ?? [],
        dependsOnSections: section.dependsOnSections ?? fallbackBlueprint.sections[index]?.dependsOnSections ?? [],
        transitionFromPrevious: section.transitionFromPrevious || fallbackBlueprint.sections[index]?.transitionFromPrevious || '',
      }))
    : fallbackBlueprint.sections;

  return {
    ...fallbackBlueprint,
    ...candidate,
    teachingProfile: fallbackBlueprint.teachingProfile,
    sections,
    unsupportedGaps: candidate.unsupportedGaps ?? fallbackBlueprint.unsupportedGaps,
  };
}

function createConfiguredTextbookProvider(config: AIConfig): TextbookChunkProvider {
  return {
    async createBlueprint(input) {
      const prompt = buildBlueprintPrompt(
        input.title,
        input.evidencePacks,
        input.fallbackBlueprint,
        input.customFocus,
        input.language,
        input.chapterTopic
      );

      if (config.provider === 'gemini') {
        const modelId = config.model?.trim() || 'gemini-3-flash-preview';
        const ai = createGeminiClient(config);
        const response = await ai.models.generateContent({
          model: modelId,
          contents: [{ role: 'user', parts: [{ text: prompt }] }],
          config: {
            responseMimeType: 'application/json',
          },
        });

        return normalizeBlueprintCandidate(JSON.parse(response.text || '{}'), input.fallbackBlueprint);
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

      return normalizeBlueprintCandidate(JSON.parse(response.choices[0].message.content || '{}'), input.fallbackBlueprint);
    },

    async generateSection(input) {
      const prompt = buildSectionPrompt(input);
      let parts: Array<{ text?: string; inlineData?: { mimeType: string; data: string } }> = [{ text: prompt }];
      if (config.provider === 'gemini' && input.attachments.length > 0) {
        parts = [
          ...input.attachments.map((attachment) => ({
            inlineData: { mimeType: attachment.mimeType, data: attachment.data },
          })),
          { text: prompt },
        ];
      }

      if (config.provider === 'gemini') {
        const modelId = config.model?.trim() || 'gemini-3-flash-preview';
        const ai = createGeminiClient(config);
        const response = await ai.models.generateContent({
          model: modelId,
          contents: [{ role: 'user', parts }],
          config: {
            responseMimeType: 'application/json',
          },
          systemInstruction: getSystemPrompt(input.teachingProfile.goal, input.language),
        });

        const json = JSON.parse(response.text || '[]');
        return Array.isArray(json) ? json : (json.chunks || json.bodyBlocks || []);
      }

      const response = await postJson<{ choices: Array<{ message: { content: string } }> }>(
        `${getOpenAIBaseUrl(config)}/chat/completions`,
        {
          model: config.model || 'gpt-4-turbo-preview',
          messages: [
            { role: 'system', content: getSystemPrompt(input.teachingProfile.goal, input.language) },
            { role: 'user', content: prompt },
          ],
          response_format: { type: 'json_object' },
        },
        getOpenAIApiKey(config)
      );

      const json = JSON.parse(response.choices[0].message.content || '{}');
      return Array.isArray(json) ? json : (json.chunks || json.bodyBlocks || []);
    },
  };
}
```

- [ ] **Step 5: 替换 `generateTextbookChunks(...)` 内部实现，保留签名兼容**

Change the function signature from:

```ts
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
```

to:

```ts
export async function generateTextbookChunks(
  title: string,
  assets: ProjectSourceAsset[],
  goal: LearningMode,
  customFocus: string,
  language: 'zh' | 'en',
  config: AIConfig,
  attachments: Attachment[] = [],
  chapterTopic?: string,
  learningBrief?: LearningBrief
): Promise<Chunk[]> {
```

Replace the body of `generateTextbookChunks(...)` with:

```ts
  return generateTextbookChunksWithProvider(
    {
      title,
      assets,
      goal,
      customFocus,
      language,
      attachments,
      chapterTopic,
      learningBrief,
    },
    createConfiguredTextbookProvider(config)
  );
```

- [ ] **Step 6: 更新 `GGlearn/src/App.tsx` 的三处调用，传入 `learningBrief`**

Find the first call around initial generation and change:

```ts
        chunks = await generateTextbookChunks(
          title,
          transformedAssets,
          goal,
          customFocus,
          settings.language,
          settings.aiConfig,
          attachments,
          topic
        );
```

to:

```ts
        chunks = await generateTextbookChunks(
          title,
          transformedAssets,
          goal,
          customFocus,
          settings.language,
          settings.aiConfig,
          attachments,
          topic,
          learningBrief
        );
```

Find the next chapter generation call and change:

```ts
      const newChunks = await generateTextbookChunks(
        activeProject.title,
        activeProject.sourceAssets,
        activeProject.mode,
        activeProject.customFocus || '',
        activeProject.language,
        settings.aiConfig,
        activeProject.sources.map(toAttachment).filter((attachment): attachment is Attachment => Boolean(attachment)),
        topic
      );
```

to:

```ts
      const newChunks = await generateTextbookChunks(
        activeProject.title,
        activeProject.sourceAssets,
        activeProject.mode,
        activeProject.customFocus || '',
        activeProject.language,
        settings.aiConfig,
        activeProject.sources.map(toAttachment).filter((attachment): attachment is Attachment => Boolean(attachment)),
        topic,
        activeProject.learningBrief
      );
```

Find the regenerate-last-chapter call and make the same final-argument change:

```ts
      const newChunks = await generateTextbookChunks(
        activeProject.title,
        activeProject.sourceAssets,
        activeProject.mode,
        activeProject.customFocus || '',
        activeProject.language,
        settings.aiConfig,
        activeProject.sources.map(toAttachment).filter((attachment): attachment is Attachment => Boolean(attachment)),
        topic,
        activeProject.learningBrief
      );
```

- [ ] **Step 7: 运行 targeted tests 和 typecheck**

Run:

```bash
cd GGlearn && node --import tsx --test tests/textbookGeneration.test.ts
```

Expected:

```text
# fail 0
```

Run:

```bash
cd GGlearn && npm run lint
```

Expected:

```text
> react-example@0.0.0 lint
> tsc --noEmit
```

Exit code must be `0`.

- [ ] **Step 8: 提交真实 provider 接入**

Run:

```bash
git add GGlearn/src/lib/ai/textbookGeneration.ts GGlearn/src/App.tsx GGlearn/tests/textbookGeneration.test.ts
git commit -m "Route live textbook generation through body sections" -m "Connect the public generation entrypoint to the two-stage provider path and pass learningBrief from the app so user goals shape the teaching profile without changing the UI contract.

Constraint: Preserve generateTextbookChunks callers with a backward-compatible optional learningBrief
Rejected: Add more user-facing generation controls | the existing goal, foundation, preference, and focus fields are sufficient for this pass
Confidence: medium
Scope-risk: moderate
Directive: Keep citation and validation data in metadata/sourceRefs unless an explicit advanced/debug surface requests it
Tested: node --import tsx --test tests/textbookGeneration.test.ts
Tested: npm run lint
Not-tested: Live Gemini/OpenAI API generation with real keys"
```

---

### Task 5: 渲染连续教材正文并默认隐藏来源依据入口

**Files:**
- Create: `GGlearn/src/lib/textbookBodyLayout.ts`
- Create: `GGlearn/tests/textbookBodyLayout.test.ts`
- Modify: `GGlearn/src/components/ChunkRenderer.tsx`
- Modify: `GGlearn/src/views/ReaderView.tsx`

- [ ] **Step 1: 写失败测试，锁定正文分组规则**

Create `GGlearn/tests/textbookBodyLayout.test.ts`:

```ts
import test from 'node:test';
import assert from 'node:assert/strict';

import type { Chunk } from '../src/types';
import { groupTextbookBodyChunks } from '../src/lib/textbookBodyLayout';

const chunks: Chunk[] = [
  {
    id: 'chunk-1',
    type: 'text',
    title: 'Why it matters',
    content: 'Motivation',
    depth: 'conceptual',
    metadata: {
      isTextbookBody: true,
      sectionId: 'section-purpose',
      sectionTitle: 'Why eigenvalues matter',
      contentRole: 'motivation',
    },
  },
  {
    id: 'chunk-2',
    type: 'text',
    title: 'Definition',
    content: 'Definition body',
    depth: 'technical',
    metadata: {
      isTextbookBody: true,
      sectionId: 'section-definition',
      sectionTitle: 'Defining eigenvalues',
      contentRole: 'definition',
    },
  },
  {
    id: 'explain-1',
    type: 'explanation',
    title: 'Question',
    content: 'Answer',
    depth: 'conceptual',
  },
  {
    id: 'chunk-3',
    type: 'text',
    title: 'Worked example',
    content: 'Example body',
    depth: 'application',
    metadata: {
      isTextbookBody: true,
      sectionId: 'section-definition',
      sectionTitle: 'Defining eigenvalues',
      contentRole: 'worked-example',
    },
  },
];

test('groupTextbookBodyChunks groups generated body chunks by section and separates interactions', () => {
  const result = groupTextbookBodyChunks(chunks);

  assert.equal(result.sections.length, 2);
  assert.equal(result.sections[0].id, 'section-purpose');
  assert.equal(result.sections[0].chunks.length, 1);
  assert.equal(result.sections[1].id, 'section-definition');
  assert.equal(result.sections[1].chunks.length, 2);
  assert.equal(result.interactionChunks.length, 1);
  assert.equal(result.interactionChunks[0].id, 'explain-1');
});
```

- [ ] **Step 2: 运行测试，确认因为模块不存在而失败**

Run:

```bash
cd GGlearn && node --import tsx --test tests/textbookBodyLayout.test.ts
```

Expected:

```text
not ok ... Cannot find module '../src/lib/textbookBodyLayout'
```

- [ ] **Step 3: 创建正文分组 helper**

Create `GGlearn/src/lib/textbookBodyLayout.ts`:

```ts
import type { Chunk } from '../types';

export interface TextbookBodySection {
  id: string;
  title: string;
  chunks: Chunk[];
}

export interface TextbookBodyLayout {
  sections: TextbookBodySection[];
  interactionChunks: Chunk[];
}

function isGeneratedBodyChunk(chunk: Chunk): boolean {
  if (chunk.type === 'explanation' || chunk.type === 'note') {
    return false;
  }

  return chunk.metadata?.isTextbookBody === true || Boolean(chunk.metadata?.sectionId);
}

export function groupTextbookBodyChunks(chunks: Chunk[]): TextbookBodyLayout {
  const sections: TextbookBodySection[] = [];
  const sectionIndex = new Map<string, TextbookBodySection>();
  const interactionChunks: Chunk[] = [];

  chunks.forEach((chunk) => {
    if (!isGeneratedBodyChunk(chunk)) {
      interactionChunks.push(chunk);
      return;
    }

    const id = chunk.metadata?.sectionId || `section-${sections.length + 1}`;
    const title = chunk.metadata?.sectionTitle || chunk.title || 'Section';
    let section = sectionIndex.get(id);

    if (!section) {
      section = { id, title, chunks: [] };
      sectionIndex.set(id, section);
      sections.push(section);
    }

    section.chunks.push(chunk);
  });

  return { sections, interactionChunks };
}
```

- [ ] **Step 4: 运行正文分组测试，确认通过**

Run:

```bash
cd GGlearn && node --import tsx --test tests/textbookBodyLayout.test.ts
```

Expected:

```text
# fail 0
```

- [ ] **Step 5: 修改 `ChunkRenderer` 的正文规则**

In `GGlearn/src/components/ChunkRenderer.tsx`, after `sanitizedDiagramSvg`, add:

```tsx
  const isTextbookBody = chunk.metadata?.isTextbookBody === true;
```

Change:

```tsx
  const isArtifact = chunk.type === 'exercise' || chunk.type === 'summary' || chunk.type === 'note';
  const [collapsed, setCollapsed] = useState(chunk.isCollapsed !== undefined ? chunk.isCollapsed : isArtifact);
```

to:

```tsx
  const isArtifact = !isTextbookBody && (chunk.type === 'exercise' || chunk.type === 'summary' || chunk.type === 'note');
  const [collapsed, setCollapsed] = useState(chunk.isCollapsed !== undefined ? chunk.isCollapsed : isArtifact);
```

Change `toggleCollapse` from:

```tsx
    if (chunk.type === 'text' || chunk.type === 'diagram') {
```

to:

```tsx
    if (isTextbookBody || chunk.type === 'text' || chunk.type === 'diagram') {
```

Change the title/action condition from:

```tsx
              {chunk.type !== 'text' && chunk.type !== 'diagram' && (
```

to:

```tsx
              {!isTextbookBody && chunk.type !== 'text' && chunk.type !== 'diagram' && (
```

Change the Sources button condition from:

```tsx
                        {chunk.sourceRefs?.length ? (
```

to:

```tsx
                        {!isTextbookBody && onViewSources && chunk.sourceRefs?.length ? (
```

Change the interactive exercise block condition from:

```tsx
                {chunk.type === 'exercise' && (
```

to:

```tsx
                {!isTextbookBody && chunk.type === 'exercise' && (
```

- [ ] **Step 6: 修改 `ReaderView`，按 section 渲染正文流**

In `GGlearn/src/views/ReaderView.tsx`, add this import:

```tsx
import { groupTextbookBodyChunks } from '../lib/textbookBodyLayout';
```

After `const learningSummary = summarizeProjectLearning(activeProject);`, add:

```tsx
  const bodyLayout = groupTextbookBodyChunks(activeTextbook.chunks);
  const hasStructuredBody = bodyLayout.sections.length > 0;
```

Replace the "On This Page" list:

```tsx
              {activeTextbook.chunks
                .filter((chunk) => chunk.title && chunk.type !== 'explanation' && chunk.type !== 'note')
                .map((chunk) => (
                  <li key={chunk.id}>
                    <button
                      onClick={() => document.getElementById(chunk.id)?.scrollIntoView({ behavior: 'smooth' })}
                      className="text-left text-sm text-[#5F6368] hover:text-[#1A1A1A] transition-colors line-clamp-2"
                    >
                      {chunk.title}
                    </button>
                  </li>
                ))}
```

with:

```tsx
              {(hasStructuredBody ? bodyLayout.sections : activeTextbook.chunks
                .filter((chunk) => chunk.title && chunk.type !== 'explanation' && chunk.type !== 'note')
                .map((chunk) => ({ id: chunk.id, title: chunk.title || 'Section', chunks: [chunk] })))
                .map((section) => (
                  <li key={section.id}>
                    <button
                      onClick={() => document.getElementById(section.id)?.scrollIntoView({ behavior: 'smooth' })}
                      className="text-left text-sm text-[#5F6368] hover:text-[#1A1A1A] transition-colors line-clamp-2"
                    >
                      {section.title}
                    </button>
                  </li>
                ))}
```

Replace the main chunk map block:

```tsx
            {activeTextbook.chunks.map((chunk, index) => {
              let isNewChapter = false;
              if (index > 0) {
                const prevChunk = activeTextbook.chunks[index - 1];
                const currentTsMatch = chunk.id.match(/-(\d+)-/);
                const prevTsMatch = prevChunk.id.match(/-(\d+)-/);
                if (currentTsMatch && prevTsMatch) {
                  const currentTs = parseInt(currentTsMatch[1], 10);
                  const prevTs = parseInt(prevTsMatch[1], 10);
                  if (Math.abs(currentTs - prevTs) > 2000) {
                    isNewChapter = true;
                  }
                }
              }

              return (
                <React.Fragment key={chunk.id}>
                  {isNewChapter && (
                    <div className="my-16 border-t border-[#E0E0DE] relative">
                      <div className="absolute left-1/2 -top-2 -translate-x-1/2 bg-white px-3 flex items-center gap-2">
                        <BookOpen size={12} className="text-[#AFB3B0]" />
                        <span className="text-[10px] uppercase tracking-[0.2em] text-[#AFB3B0] font-sans font-semibold">New Chapter</span>
                      </div>
                    </div>
                  )}
                  <Suspense
                    fallback={
                      <div className="border border-[#E0E0DE] rounded-lg p-6 bg-[#FDFFFF] text-sm text-[#5F6368]">
                        Loading section...
                      </div>
                    }
                  >
                    <ChunkRenderer
                      chunk={chunk}
                      progress={progress}
                      onComplete={onComplete}
                      onSaveHandwriting={onSaveHandwriting}
                      onExplain={onExplain}
                      onSaveNote={onSaveNote}
                      onDeleteNote={onDeleteNote}
                      onDeleteChunk={onDeleteChunk}
                      onRegenerateChunk={onRegenerateChunk}
                      onViewSources={onOpenSourceTrace}
                      isExplaining={explainingId === chunk.id}
                    />
                  </Suspense>
                </React.Fragment>
              );
            })}
```

with:

```tsx
            {hasStructuredBody ? (
              bodyLayout.sections.map((section, sectionIndex) => (
                <section key={section.id} id={section.id} className="scroll-mt-24">
                  <div className="mb-8">
                    <p className="font-sans text-xs uppercase tracking-[0.22em] text-[#5F6368]">
                      {sectionIndex + 1}
                    </p>
                    <h2 className="mt-2 font-serif text-3xl md:text-4xl tracking-tight text-[#1A1A1A]">
                      {section.title}
                    </h2>
                  </div>
                  <div className="space-y-6">
                    {section.chunks.map((chunk) => (
                      <Suspense
                        key={chunk.id}
                        fallback={
                          <div className="border border-[#E0E0DE] rounded-lg p-6 bg-[#FDFFFF] text-sm text-[#5F6368]">
                            Loading section...
                          </div>
                        }
                      >
                        <ChunkRenderer
                          chunk={chunk}
                          progress={progress}
                          onComplete={onComplete}
                          onSaveHandwriting={onSaveHandwriting}
                          onExplain={onExplain}
                          onSaveNote={onSaveNote}
                          onDeleteNote={onDeleteNote}
                          onDeleteChunk={onDeleteChunk}
                          onRegenerateChunk={onRegenerateChunk}
                          isExplaining={explainingId === chunk.id}
                        />
                      </Suspense>
                    ))}
                  </div>
                </section>
              ))
            ) : (
              activeTextbook.chunks.map((chunk) => (
                <Suspense
                  key={chunk.id}
                  fallback={
                    <div className="border border-[#E0E0DE] rounded-lg p-6 bg-[#FDFFFF] text-sm text-[#5F6368]">
                      Loading section...
                    </div>
                  }
                >
                  <ChunkRenderer
                    chunk={chunk}
                    progress={progress}
                    onComplete={onComplete}
                    onSaveHandwriting={onSaveHandwriting}
                    onExplain={onExplain}
                    onSaveNote={onSaveNote}
                    onDeleteNote={onDeleteNote}
                    onDeleteChunk={onDeleteChunk}
                    onRegenerateChunk={onRegenerateChunk}
                    onViewSources={onOpenSourceTrace}
                    isExplaining={explainingId === chunk.id}
                  />
                </Suspense>
              ))
            )}
```

- [ ] **Step 7: 弱化来源依据前台入口**

In `GGlearn/src/views/ReaderView.tsx`, change the Source Workspace button label from:

```tsx
            <BookOpen size={14} /> Source Workspace
```

to:

```tsx
            <BookOpen size={14} /> Materials
```

Remove the `View Source` navigation button block:

```tsx
          <button
            onClick={onOpenProjectSources}
            className="flex items-center gap-2 text-xs font-semibold uppercase tracking-widest text-[#5F6368] hover:text-[#1A1A1A] border px-4 py-2 rounded-sm border-[#E0E0DE] transition-colors"
          >
            <Globe size={14} /> {t.viewSource}
          </button>
```

After removing it, remove `Globe` from the lucide import if no longer used:

```tsx
import { ArrowLeft, BookOpen, Download, Menu, Plus, RefreshCw, Sparkles, X } from 'lucide-react';
```

Keep `onOpenProjectSources` in props for now if other call sites still provide it. Do not show it in the main reader navigation.

- [ ] **Step 8: 运行 layout test、完整 node tests 和 typecheck**

Run:

```bash
cd GGlearn && node --import tsx --test tests/textbookBodyLayout.test.ts
```

Expected:

```text
# fail 0
```

Run:

```bash
cd GGlearn && npm test
```

Expected:

```text
# fail 0
```

Run:

```bash
cd GGlearn && npm run lint
```

Expected exit code: `0`.

- [ ] **Step 9: 提交连续正文阅读呈现**

Run:

```bash
git add GGlearn/src/lib/textbookBodyLayout.ts GGlearn/tests/textbookBodyLayout.test.ts GGlearn/src/components/ChunkRenderer.tsx GGlearn/src/views/ReaderView.tsx
git commit -m "Render generated textbook chunks as continuous body sections" -m "Group generated body chunks into reader sections and keep source tracing out of the default reading controls so examples, practice, and summaries feel like textbook content instead of external widgets.

Constraint: Generated body content must remain compatible with existing ChunkRenderer interactions
Rejected: Keep the Sources hover action on generated body chunks | this foregrounds citation during reading
Confidence: medium
Scope-risk: moderate
Directive: Interactive notes and explanations should stay separate from metadata.isTextbookBody chunks
Tested: node --import tsx --test tests/textbookBodyLayout.test.ts
Tested: npm test
Tested: npm run lint
Not-tested: Browser visual pass"
```

---

### Task 6: Final verification and plan-aligned cleanup

**Files:**
- No planned source edits unless verification reveals a concrete failure.

- [ ] **Step 1: Run full verification**

Run:

```bash
cd GGlearn && npm test
```

Expected:

```text
# fail 0
```

Run:

```bash
cd GGlearn && npm run lint
```

Expected exit code: `0`.

Run:

```bash
cd GGlearn && npm run build
```

Expected:

```text
✓ built in
```

Exit code must be `0`.

- [ ] **Step 2: Review changed files**

Run:

```bash
git diff --stat HEAD
```

Expected: only files from this plan should appear. If unrelated pre-existing files appear, leave them unstaged and mention them in the final report.

- [ ] **Step 3: Inspect for foreground source leakage in reader/generation prompts**

Run:

```bash
rg -n "Sources|Grounding|sourceRefs|evidence|根据资料|来源指出|本文指出" GGlearn/src/views/ReaderView.tsx GGlearn/src/components/ChunkRenderer.tsx GGlearn/src/lib/ai/textbookGeneration.ts
```

Expected:

```text
```

Allowed exceptions:

- `textbookGeneration.ts` may mention `evidence` and `sourceRefIds` in backend prompts and schemas.
- `ReaderView.tsx` may retain source drawer code if it is not exposed in the default body chunk controls.
- `ChunkRenderer.tsx` may retain the legacy Sources button for non-body chunks.

- [ ] **Step 4: Commit any verification-only fixes**

If Step 1 or Step 3 requires small fixes, commit them with:

```bash
git add <fixed-files>
git commit -m "Tighten textbook body generation verification" -m "Address verification findings from the first body-quality pass without expanding the feature scope.

Constraint: Keep fixes limited to failures found by tests, typecheck, build, or source-leak inspection
Confidence: high
Scope-risk: narrow
Tested: npm test
Tested: npm run lint
Tested: npm run build"
```

If no fixes are needed, do not create an empty commit.

- [ ] **Step 5: Final report requirements**

The final report must include:

- Changed files.
- Simplifications made.
- Verification commands and outcomes.
- Remaining risks:
  - Live model output quality still needs a manual sample run with real API keys.
  - Browser visual QA is still needed for the continuous reader layout.
  - Historical generated projects may still display legacy chunk-card behavior until regenerated.

---

## Self-review

- Spec coverage:
  - 教材正文层：Task 1 metadata and Task 5 body layout.
  - 互动学习层：Task 5 keeps explanation/note chunks outside generated body sections.
  - 来源依据层后台化：Task 2 validation, Task 4 provider prompts, Task 5 hidden default Sources controls.
  - 学习目标策略化：Task 1 `createTeachingProfile`, Task 4 App passes `learningBrief`.
  - 小节级生成：Task 3 orchestration, Task 4 real providers.
  - 第一版边界：all tasks preserve `Chunk[]`, current entrypoints, and existing source assets.
- Placeholder scan:
  - No unresolved blank items or vague implementation steps.
- Type consistency:
  - `ContentRole`, `TeachingProfile`, `ChapterBlueprint`, `ChapterSectionBlueprint`, `SectionGenerationInput`, and validation types are defined in Task 1 before use.
  - `metadata.isTextbookBody`, `sectionId`, `sectionTitle`, and `contentRole` are added in Task 1 before reader work in Task 5.
  - `generateTextbookChunksWithProvider` is defined in Task 3 before public provider routing in Task 4.
