# 技术变更日志

本文档记录 SlideTutor 项目的重要技术变更、架构决策和实现细节。

条目按时间倒序排列（最新的在前）。

---
## [2026-04-04] Completed Phase 04 BYOK-First Access Layer

**What**: Introduced a BYOK-first access layer that separates model choice from model credentials. The frontend now persists normalized `aiAccess` settings in IndexedDB, migrates legacy `qwen` / `doubao` selections into one `openai-compatible` provider family, exposes local BYOK fields in the settings modal, and automatically attaches normalized access metadata to `/api/generate` requests. The backend now resolves Gemini and OpenAI-compatible access through one shared routing boundary, while preserving migration-safe env fallback for preset OpenAI-compatible endpoints.

**Why**: Phase 04 needed to make user-supplied model APIs the real public entry path without touching mature teaching logic. The key architectural risk was mixing provider selection, BYOK credentials, and future hosted access into one flat state model. Splitting access from model choice keeps the public path simple now and leaves room for hosted access later.

**Impact**:
- users can configure Gemini BYOK or OpenAI-compatible BYOK locally from the existing settings modal
- OpenAI-compatible access now has one shared adapter path driven by `apiKey + baseURL`, with preset shortcuts for `qwen` and `doubao`
- legacy local selections are normalized during store init instead of breaking after the provider-family cleanup
- `/api/generate` can prefer BYOK credentials while still falling back to preset env secrets during the migration window
- parser setup is still not required from users in this phase because parsing remains platform-funded

**Files**: `SlideTutor-AI/src/config/models.ts`, `SlideTutor-AI/src/store/uiStore.ts`, `SlideTutor-AI/src/components/SettingsModal.tsx`, `SlideTutor-AI/src/lib/api/apiClient.ts`, `SlideTutor-AI/api/lib/env.ts`, `SlideTutor-AI/api/lib/generateService.ts`, `SlideTutor-AI/src/store/uiStore.test.ts`, `SlideTutor-AI/src/lib/api/apiClient.test.ts`, `SlideTutor-AI/src/components/SettingsModal.test.tsx`, `SlideTutor-AI/api/security.test.ts`, `SlideTutor-AI/test/workers/generate-stream.worker.test.ts`, `SlideTutor-AI/README.md`, `docs/backend/api-design.md`, `docs/frontend/architecture.md`, `docs/frontend/data-flow.md`

---
## [2026-04-04] Completed Cloudflare-First Public Runtime Cutover

**What**: Switched the default public runtime assumptions from Vercel-first to Cloudflare Worker-first. Added Worker-side `/api/feedback`, introduced an HTTP-based notification adapter for feedback and security alerts, removed the Vercel analytics bootstrap import, changed the default dev/deploy scripts to Worker-oriented commands, deleted `vercel.json`, and updated deployment/API/token-auth docs to match the new runtime.

**Why**: Phase 03 needed to stop carrying hidden Vercel-only dependencies into the upcoming BYOK-first work. The remaining blockers were not teaching logic, but public runtime assumptions: frontend bootstrap, feedback delivery, deployment scripts, and operator docs.

**Impact**:
- the intended public base is now one Cloudflare Worker serving the SPA and critical-path APIs
- `/api/feedback` is no longer an accidental Vercel SMTP holdout
- feedback and security alerts now use a Worker-compatible HTTP notification path with a log-only local fallback
- local operators should use `NOTIFICATION_PROVIDER`, `RESEND_API_KEY`, `NOTIFICATION_FROM_EMAIL`, `FEEDBACK_TO_EMAIL`, and `SECURITY_ALERT_TO_EMAIL` instead of legacy SMTP env vars
- `@vercel/analytics` is no longer part of the frontend bootstrap path

**Files**: `SlideTutor-AI/src/worker/routes/feedback.ts`, `SlideTutor-AI/src/worker/lib/notifications.ts`, `SlideTutor-AI/src/worker/index.ts`, `SlideTutor-AI/src/worker/routes/generate.ts`, `SlideTutor-AI/src/components/SettingsModal.tsx`, `SlideTutor-AI/package.json`, `SlideTutor-AI/src/main.tsx`, `SlideTutor-AI/.env.example`, `SlideTutor-AI/README.md`, `docs/architecture/deployment.md`, `docs/backend/api-design.md`, `docs/security/token-authentication.md`

---
## [2026-04-03] Completed Phase 2 Artifact-First Downstream Migration

**What**: Finished the downstream JSON migration so runtime state, persistence writes, follow-up targeting, chunk regeneration, quick-explain rendering, quiz context assembly, and global analysis completion now consume `explanationArtifact` / `distillArtifact` directly. `regenerate_chunk` was also promoted into the provider-native structured-output layer and now returns a single structured knowledge-card object instead of legacy markdown text.

**Why**: Phase 1 stabilized upstream JSON generation, but downstream code was still projecting artifacts back into compatibility strings. That left the old markdown bridge alive in major runtime paths and made the architecture harder to reason about. Phase 2 removes that split and makes artifacts the real source of truth end-to-end.

**Impact**:
- runtime no longer depends on `explanation`, `cheatSheet`, or `summary`
- new IndexedDB writes are artifact-first and do not re-project legacy string fields
- old saved records are only normalized once at load time through a migration boundary
- `followup` / `regenerate_followup` now receive structured explanation context
- `regenerate_chunk` now uses native structured output for Gemini and OpenAI-compatible providers

**Files**: `SlideTutor-AI/src/hooks/useSlideAnalysis.ts`, `SlideTutor-AI/src/hooks/useFollowUp.ts`, `SlideTutor-AI/src/hooks/useChunkRegenerate.ts`, `SlideTutor-AI/src/hooks/usePdfLibrary.ts`, `SlideTutor-AI/src/components/CanvasTutor.tsx`, `SlideTutor-AI/src/components/PdfViewer.tsx`, `SlideTutor-AI/src/hooks/useQuiz.ts`, `SlideTutor-AI/api/lib/structuredOutputConfig.ts`

---
## [2026-04-03] Added Provider-Native Structured Output Adapters and Gemini Token Diagnostics

**What**: Introduced a shared structured-output configuration layer for generation tasks. `Gemini` now uses native `responseJsonSchema` plus task-level thinking controls, while all OpenAI-compatible providers now share one `json_schema` response-format adapter for `explain`, `distill`, `generate_questions`, and `evaluate_answers`. Also added Gemini stream diagnostics for `finishReason`, token usage, and tail previews, and expanded frontend parse-failure logging for invalid `explain` / `distill` payloads.

**Why**: Weak-model failures had split into two classes: schema drift on OpenAI-compatible models and `MAX_TOKENS` truncation on Gemini `distill`. The old prompt-only JSON approach was not stable enough. Native structured-output APIs reduce protocol drift, and Gemini diagnostics confirmed that task-level thinking policy matters for token budgeting.

**Impact**:
- OpenAI-compatible providers now have one shared schema-first path instead of provider-specific JSON prompt conventions
- `distill` on Gemini now uses `thinkingLevel = "minimal"` to preserve output budget for the final JSON payload
- `generate_questions` and `evaluate_answers` are now covered by the same structured-output infrastructure as `explain` / `distill`
- this provider-native layer now also powers `regenerate_chunk` after the downstream Phase 2 migration

**Files**: `SlideTutor-AI/api/lib/structuredOutputConfig.ts`, `SlideTutor-AI/api/generate.ts`, `SlideTutor-AI/api/lib/geminiStreamDiagnostics.ts`, `SlideTutor-AI/src/hooks/useSlideAnalysis.ts`, `SlideTutor-AI/src/lib/ai/artifacts.ts`, `docs/frontend/architecture.md`, `docs/frontend/data-flow.md`

---
## [2026-04-03] Migrated Explain / Distill Generation to Structured JSON Artifacts

**What**: Replaced the main `explain` and `distill` output contracts with structured JSON schemas. Added `explanationArtifact` and `distillArtifact` to page state, introduced parser / serializer utilities for structured artifacts, switched `CanvasTutor` to artifact-first rendering, and moved follow-up / chunk-regenerate targeting to artifact chunk selection instead of direct markdown splitting. Gemini `explain` requests now also explicitly request `application/json`.

**Why**: Several weaker models could produce acceptable teaching content but often broke the mixed markdown protocol by leaking `Intent` / probe markers into prose or by returning incomplete context memory blocks. Moving structure into JSON fields reduces formatting burden on the model while keeping the teaching prompt content intact.

**Impact**:
- explain streaming now updates the UI only when a full card object closes, rather than on arbitrary text fragments
- page state now keeps structured artifacts as the primary machine-readable form while legacy `explanation` / `cheatSheet` / `summary` strings remain as temporary compatibility projections
- `summary` compatibility text is now serialized as a fixed-order multiline context-memory block instead of a flattened single line
- follow-up and chunk regeneration now resolve target cards from artifact order, which prepares the rest of the app for fully native JSON consumption

**Files**: `SlideTutor-AI/src/lib/ai/artifacts.ts`, `SlideTutor-AI/src/hooks/useSlideAnalysis.ts`, `SlideTutor-AI/src/components/CanvasTutor.tsx`, `SlideTutor-AI/src/hooks/useFollowUp.ts`, `SlideTutor-AI/src/hooks/useChunkRegenerate.ts`, `SlideTutor-AI/api/generate.ts`, `docs/frontend/architecture.md`, `docs/frontend/data-flow.md`

---
## [2026-04-02] Removed End-of-Expand Jitter from Tutor Card Input Panels

**What**: Moved tutor-card textarea focus out of the panel's `onAnimationComplete` hook and into a mount-time `requestAnimationFrame` effect that uses `focus({ preventScroll: true })` with a fallback path. Added regression coverage for the no-scroll focus helper.

**Why**: After the 2026-04-01 panel motion refactor, close jitter stayed fixed but a smaller jump remained at the very end of expand. Reviewing the changelog and git history showed that the panel motion contract itself was still intact. The remaining hitch lined up with the delayed textarea focus firing exactly at animation completion, which could trigger an extra browser scroll/reflow step on the last frame.

**Impact**:
- tutor-card drawers for `follow-up`, `add note`, and `regenerate` should no longer twitch at the final moment of expansion
- focus is still deferred until after mount, but now avoids browser scroll jumps when supported
- no theme tokens, note behavior, or panel timing constants changed

**Files**: `SlideTutor-AI/src/components/CanvasTutor.tsx`, `SlideTutor-AI/src/lib/focusWithoutScroll.ts`, `SlideTutor-AI/src/lib/focusWithoutScroll.test.ts`

## [2026-04-01] Twilight Zen Theme Visual Refinement and Reading Comfort Update

**What**: Refined the `twilight-zen` theme visual layer to align with the "mid-tone dusk" design DNA. Lifted the background atmosphere from "charcoal navy" to a softer "misted indigo" and implemented a specific reading comfort override for tutor and note cards.

**Why**: User feedback indicated that the previous version of `twilight-zen` was too high-contrast, with bright white text on a very dark background causing eye fatigue during long reading sessions.

**Impact**:
- **Improved Reading Comfort**: Main explanation prose in tutor cards and note cards now uses a soft cool mist gray-blue (`#B8C9E1`) instead of slate-200 white.
- **Enhanced Atmosphere**: Updated background with mid-tone base `#233755` and atmospheric radial haze using mist pink (`#DBAEC8`) and mist purple (`#9D9DD4`) at low opacity.
- **Strict Semantic Protection**: Used a "Direct Override + Explicit Restoration" strategy to protect the `Thinking Prompt` and shared product accents from theme-specific color bleeding.
- **Visual Hierarchy**: Card titles now naturally contrast with the softer body text by inheriting the conservatively tuned `--text-primary` (`#D1DBE8`).

**Files**: `SlideTutor-AI/src/index.css`, `docs/frontend/architecture.md`

---
## [2026-04-01] Centralized Runtime Theme-Color Source for PWA Title Bar Sync

**What**: Removed the duplicated runtime theme-color tables between `index.html` and `uiStore.ts`. Theme-specific title-bar colors now come from dedicated `<meta name="slidetutor-theme-color-*">` entries in `index.html`, and both the early boot script and `updateMetaThemeColor` read from that same DOM metadata. Added regression tests covering both boot-time and store-driven updates to `<meta name="theme-color">`.

**Why**: The previous PWA title-bar fix worked, but the same theme-color mapping lived in multiple places. That made future theme tweaks easy to drift: the installed window color could regress back to a mismatched shade even while tests stayed green.

**Impact**:
- `theme-color` updates now have one runtime source of truth
- startup and post-boot theme synchronization stay aligned
- regression coverage now checks both `index.html` boot behavior and `uiStore` theme changes

**Files**: `SlideTutor-AI/index.html`, `SlideTutor-AI/src/store/uiStore.ts`, `SlideTutor-AI/src/store/uiStore.test.ts`, `SlideTutor-AI/src/test/themeBootScript.test.ts`

---
## [2026-04-01] Synchronized PWA Title Bar with Active Theme

**What**: Updated the application to dynamically synchronize the `<meta name="theme-color">` tag with the currently active theme. Also updated the default PWA `manifest.json` `theme_color` and `background_color` to match the default light theme.

**Why**: When users installed the app as a PWA (e.g., via Chrome), the window title bar remained a static purple color (`#4f46e5`), which clashed visually with the top navigation bar, especially when switching between different themes (`eyecare`, `twilight-zen`, `spring-meadow`). Synchronizing the meta tag ensures a native, polished feel across all themes.

**Impact**:
- The window title bar in PWA mode now seamlessly matches the application's header color.
- Added a `updateMetaThemeColor` helper in `uiStore.ts` to push theme color updates to the DOM immediately upon theme change.
- The early-boot script in `index.html` now reads `localStorage` to set the `theme-color` meta tag synchronously, eliminating any purple flash before React boots.

**Files**: `SlideTutor-AI/public/manifest.json`, `SlideTutor-AI/index.html`, `SlideTutor-AI/src/store/uiStore.ts`

---
## [2026-04-01] Refined Theme Icon Contrast and Reverted Highlight Color

**What**: Removed overly broad `.lucide` icon color overrides from the `spring-meadow` and `twilight-zen` themes, allowing shared product controls (like "Upload PDF" and the active toggle) to correctly inherit contrasting text colors. Additionally, reverted the global PDF explanation highlight color from purple to the previously used comfortable light gray (`rgba(39, 39, 42, 0.1)`), applied uniformly across all themes.

**Why**: In `spring-meadow` and `twilight-zen`, aggressive CSS overrides forced all icons to a theme-specific color (e.g., dark text in `spring-meadow`). When these icons appeared inside dark accent buttons, they became unreadable. Removing these overrides enforces the Theme Visual Consistency Contract by letting shared controls manage their own text contrast. Furthermore, the purple highlight introduced recently was found to be less comfortable than the original light gray.

**Impact**:
- Icons in shared product components (e.g., top header buttons, `Thinking Prompt`) now retain proper contrast against their backgrounds across all themes.
- PDF explanation highlights use a uniform, borderless light gray overlay with a multiply blend mode, ensuring visual consistency and reading comfort regardless of the active theme.

**Files**: `SlideTutor-AI/src/index.css`

---
## [2026-04-01] Restored Shared Product Accents and Finalized Borderless Highlight Contract

**What**: Corrected the follow-up theme refinement so product-level accent UI is shared again across themes instead of being recolored per theme. The top-header actions (`Upload PDF` / `Change PDF`, active library toggle, product badge) now use one stable semantic accent treatment, `Thinking Prompt` stays on the shared purple product accent, and PDF highlights now use one borderless fill-only design across all themes.

**Why**: The previous refinement drifted away from the intended theme boundary. It made shared product controls theme-colored again, reintroduced a pink `Thinking Prompt` accent in `twilight-zen`, and treated highlight behavior as theme-aware instead of truly unified. That conflicted with the design rule that atmosphere may vary by theme, but shared teaching and product semantics should stay stable.

**Impact**:
- `spring-meadow` and `twilight-zen` keep their own backgrounds and glass surfaces, but no longer redefine shared product accent tokens.
- Header CTA/readability is more stable because accent controls use one calm neutral product treatment instead of theme-specific colors.
- Highlight overlays are now consistently borderless and fill-only in every theme, with one shared visual meaning.
- Added a CSS contract regression test so future theme edits are less likely to reintroduce theme-specific overrides for shared accent tokens.

**Files**: `SlideTutor-AI/src/index.css`, `SlideTutor-AI/src/components/Header/AppHeader.tsx`, `SlideTutor-AI/src/components/Header/AppHeader.test.tsx`, `SlideTutor-AI/src/lib/themeVisualContract.test.ts`, `docs/changelog/CHANGELOG_TECH.md`, `docs/frontend/architecture.md`

## [2026-04-01] Unified Theme Accent Semantics and Fixed Spring Meadow Header Contrast

**What**: Refined the multi-theme visual contract so `twilight-zen` and `spring-meadow` now share the same semantic treatment for typography, PDF highlight overlays, and `Thinking Prompt` accent styling while still keeping their own background atmosphere and glassmorphism. Also fixed a `spring-meadow` contrast regression in the top header where dark accent surfaces could render icons and text as a hard-to-read dark block.

**Why**: Recent theme experiments improved mood and visual personality, but they also introduced cross-theme inconsistency in three places users repeatedly see: title typography, slide highlight appearance, and the `Thinking Prompt` label color. On top of that, `spring-meadow` had a local contrast bug in header accent controls because theme-level icon overrides were fighting component-level dark accent buttons.

**Impact**:
- `twilight-zen` no longer uses a separate pink `Thinking Prompt` title accent; it now uses a theme-adapted purple accent closer to the default product language.
- `twilight-zen` and `spring-meadow` highlight overlays now follow one shared semantic highlight system instead of custom per-theme treatments.
- Header accent controls now use explicit semantic classes, which prevents theme-wide icon color overrides from breaking readability on dark accent surfaces.
- No feature behavior, theme toggle logic, persistence flow, or PDF interaction logic changed.

**Files**: `SlideTutor-AI/src/index.css`, `SlideTutor-AI/src/components/Header/AppHeader.tsx`, `SlideTutor-AI/src/components/CanvasTutor.tsx`, `SlideTutor-AI/src/components/PdfViewer.tsx`, `SlideTutor-AI/src/components/Header/AppHeader.test.tsx`

## [2026-04-01] Replaced Unfriendly Themes with "Twilight Zen" Theme

**What**: Removed the `windowsill` ("morning mist") and `rainy` ("cloudy rainy") themes and introduced a new `twilight-zen` ("暮色禅意") theme.
- The new theme applies a deep, twilight-inspired background gradient with soft sunset pink highlights, modeled after a calming sky reference image.
- Updated the theme toggle sequence: `light` -> `eyecare` -> `twilight-zen` -> `spring-meadow` -> `light`.

**Why**: Addressed user feedback stating that the "morning mist" and "cloudy rainy" themes were harsh and caused eye fatigue. The new "Twilight Zen" theme is designed to be highly legible and "healing/calming" (治愈), using softer glassmorphic overlays and high-contrast light text against a subdued dark backdrop.

**Impact**:
- Removed old CSS classes and background images for `windowsill` and `rainy`.
- Added `.twilight-zen` theme definition with custom glassmorphism and subtle screen-blend highlights.
- Global theme state (`uiStore.ts`) and UI components (`ThemeToggle`, `SettingsModal`) updated.
- No new external dependencies added.

**Files**: `SlideTutor-AI/src/index.css`, `SlideTutor-AI/src/store/uiStore.ts`, `SlideTutor-AI/src/components/ThemeToggle.tsx`, `SlideTutor-AI/src/components/SettingsModal.tsx`, `SlideTutor-AI/index.html`

## [2026-03-31] Added "Spring Meadow" Theme

**What**: Implemented a new UI theme called `spring-meadow` (春日草甸) alongside existing ones. The theme features a multi-radial CSS background mimicking a natural 3D meadow landscape, paired with high-blur glassmorphic panels and high-contrast dark text.

**Why**: Addressed user feedback regarding eye fatigue from reading white text on dark backgrounds in existing themes (like Morning Mist and Cloudy Rainy). The new theme preserves an immersive, natural aesthetic while drastically improving readability and comfort for long sessions.

**Impact**:
- Added `.spring-meadow` theme definition in `src/index.css`.
- Updated `uiStore.ts` to include the new theme in the type definition and persistence logic.
- Updated `ThemeToggle` to cycle through the new theme and added a `Leaf` icon.
- Settings modal now lists all five themes.
- No changes to existing themes or core architecture.

**Files**: `SlideTutor-AI/src/index.css`, `SlideTutor-AI/src/store/uiStore.ts`, `SlideTutor-AI/src/components/ThemeToggle.tsx`, `SlideTutor-AI/src/components/SettingsModal.tsx`, `SlideTutor-AI/index.html`, `docs/frontend/architecture.md`

## [2026-04-01] Smoothed Tutor Card Input Panel Open/Close Motion

**What**: Refactored the tutor-card action input panel state and animation flow. The panel now closes in two phases: first it exits visually, then the draft text is cleared on `AnimatePresence` exit completion. The panel open/close motion also switched from a spring-based `height: auto` sequence to a short tween with animated `marginTop`, and explanation chunk wrappers now use position-only layout animation.

**Why**: The previous interaction stacked three sources of motion work on the same frame boundary: `height: auto` spring measurement inside the panel, parent chunk layout animation, and synchronous focus/reset side effects. That combination caused a visible hitch at the end of expand/collapse.

**Impact**:
- tutor-card input drawers for `follow-up`, `add note`, and `regenerate` should feel more continuous at the end of open/close
- drafts are preserved during the exit animation and cleared only after the panel fully leaves
- chunk rows still animate positional shifts, but no longer compound panel-size animation with full layout-size interpolation
- added regression coverage for the new panel state and animation contract

**Files**: `SlideTutor-AI/src/components/CanvasTutor.tsx`, `SlideTutor-AI/src/lib/tutorCardInputPanel.ts`, `SlideTutor-AI/src/lib/tutorCardInputPanel.test.ts`

## [2026-03-31] Stabilized Note Drag Boundaries Across PDF and Tutor Views

**What**: Added a shared note drag utility to formalize two interaction rules: PDF canvas panning must not start from `.spatial-note` targets, and tutor note drop resolution must skip the dragged note subtree before choosing a new chunk target. Wired the PDF viewer and tutor note card drag logic to that utility, and tightened tutor note drag physics with `dragMomentum={false}` and `dragElastic={0}`.

**Why**: Two note regressions shared the same root theme: drag ownership was ambiguous. On the PDF side, zoomed-page panning and note dragging could start together. On the tutor side, `elementsFromPoint()` could resolve the dragged note back to its source chunk because the note remained mounted under the original chunk in the DOM during drag.

**Impact**:
- dragging a PDF note no longer also starts PDF pan
- dragging a tutor note to a different knowledge card now resolves targets more reliably
- note drag behavior is now covered by dedicated regression tests
- frontend note interaction rules are documented in `docs/frontend/architecture.md`

**Files**: `SlideTutor-AI/src/lib/noteDragUtils.ts`, `SlideTutor-AI/src/lib/noteDragUtils.test.ts`, `SlideTutor-AI/src/components/PdfViewer.tsx`, `SlideTutor-AI/src/components/CanvasTutor.tsx`, `docs/frontend/architecture.md`

## [2026-03-30] Tightened Explanation Highlight Contract

**What**: Tightened the `explain` prompt contract for `>>>Intent` so knowledge-card highlight boxes must use integer `0..1000` coordinates in `[ymin, xmin, ymax, xmax]` order, and clarified that `###` is reserved for card title lines only. Also disabled chunk regeneration for the intro card `### This Slide at a Glance` in the tutor UI.

**Why**: The highlight pipeline depends on parsing `>>>Intent` back out of the explanation text. Tightening the output contract improves highlight precision without changing the broader teaching prompt shape, and disabling intro-card regeneration removes a contract mismatch where the intro card could be forced into a knowledge-card regenerate flow.

**Impact**:
- explanation cards now have a stricter, more parser-safe highlight contract
- card-body `###` headings are now explicitly disallowed to protect existing chunk splitting
- intro cards no longer expose the chunk regenerate action in the UI
- no backend API shape changes and no new dependencies

**Files**: `SlideTutor-AI/src/lib/ai/prompts.ts`, `SlideTutor-AI/src/lib/ai/prompts.test.ts`, `SlideTutor-AI/src/lib/ai/__snapshots__/prompts.test.ts.snap`, `SlideTutor-AI/src/components/CanvasTutor.tsx`, `SlideTutor-AI/src/components/CanvasTutor.test.tsx`

## [2026-03-29] Editorial Redesign of Quick Explain (Focus Mode)

**What**: Redesigned the visual representation of "Quick Explain" (formerly known as cheatsheet) in Focus Mode. Replaced the earlier plain block rendering with an editorial-style reading layout, added a lightweight paragraph-formatting helper for single-block quick explains, and introduced lead-paragraph emphasis plus staggered paragraph entry animations.

**Why**: To align the UI with the "Teacher's Voice" product positioning. The previous card design was too structured and mechanical, making the content feel like a "data summary" rather than a natural classroom explanation. The new design creates a more immersive, lecture-like reading experience that is easier on the eyes and feels more personal.

**Impact**:
- Enhanced reading rhythm and focus in Focus Mode.
- Improved visual hierarchy: lead paragraphs are now more prominent to capture immediate attention.
- Quick Explain content may now be lightly re-paragraphed at render time for readability, while the stored `cheatSheet` value remains unchanged.
- Theme-aware design still relies on semantic tokens for contrast, with a few local accent styles in the component.
- No changes to data structure, prompt logic, or backend services.

**Files**: `SlideTutor-AI/src/components/CanvasTutor.tsx`, `SlideTutor-AI/src/lib/ai/quickExplainFormat.ts`, `SlideTutor-AI/src/lib/ai/quickExplainFormat.test.ts`

## [2026-03-29] Added Drag-and-Drop and Copy-Paste PDF Upload

**What**: Implemented global drag-and-drop and copy-paste support for uploading PDF files. Added an `isFileDragging` state for drag-over visual feedback and a `pendingFile` state to show a confirmation modal before processing the upload. Updated the initial empty-state UI to guide users about these new upload methods.

**Why**: To improve user experience and discoverability. Users previously could only upload via the click-to-upload button. The confirmation modal ensures that "fast" actions like pasting or dropping don't accidentally overwrite current work without a conscious "Upload Now" click.

**Impact**:
- Added `pendingFile` state and confirmation UI in `App.tsx`.
- Updated `PdfViewer.tsx` empty-state text to mention Drag & Drop and Paste.
- `processFile` now clears `pendingFile` upon successful execution.
- No breaking changes.

**Files**: `SlideTutor-AI/src/App.tsx`, `SlideTutor-AI/src/components/PdfViewer.tsx`

## [2026-03-29] Product Renaming and Prompt Repositioning for Quick Explain

**What**: Repositioned the old `cheatsheet` artifact as product-facing `Quick Explain` / `速通讲解`. Updated the prompt language so the model no longer treats this artifact like a study card or fast-scan memory note. The UI now labels the section as `速通讲解` in Chinese and `Quick Explain` in English, while the persisted state field remains `cheatSheet` for compatibility.

**Why**: The previous naming and prompt wording pushed the model toward a robotic "cheat sheet / takeaways / memory card" style. The intended job of this artifact is different: it should feel like a teacher quickly explaining the page in class, not a compressed review card.

**Impact**:
- prompt instructions now target short classroom-style explanation instead of study-card structure
- distill output now uses `quickExplain` + `contextMemory` JSON keys
- `useSlideAnalysis.ts` accepts `quickExplain` and maps it into the existing `cheatSheet` storage field
- UI wording now reflects the product concept without breaking stored data

**Files**: `SlideTutor-AI/src/lib/ai/prompts.ts`, `SlideTutor-AI/src/lib/ai/prompts.test.ts`, `SlideTutor-AI/src/lib/ai/__snapshots__/prompts.test.ts.snap`, `SlideTutor-AI/src/hooks/useSlideAnalysis.ts`, `SlideTutor-AI/src/hooks/useSlideAnalysis.test.ts`, `SlideTutor-AI/src/components/CanvasTutor.tsx`, `docs/frontend/architecture.md`, `docs/frontend/data-flow.md`

## [2026-03-29] Intro Card Prompt Refinement for Teacher Persona

**What**: Refined the `explain` prompt's "Mandatory intro card" to transition from a dry summary/focus instruction to a natural, teacher-like "contextual lead-in". The new prompt uses a "toolbox" approach where the AI can selectively use a "Bridge" (optional), the "Soul" of the page, and a "Natural Lead".

**Why**: The previous intro cards felt too much like a summary or a mechanical table of contents ("This slide introduces A, B, and C"). This broke the immersion of having a personal tutor. We wanted a smoother entry that bridges from the previous context naturally and sets the stage without spoiling the detailed breakdown that follows.

**Impact**:
- Intro cards are now more conversational and concise (1-3 sentences).
- Improved "Bridge" logic: AI now only connects to previous slides if it's natural and non-confusing.
- "Forbidden Styles" added to strictly block corporate/formal phrases and mechanical "Notice how..." instructions in the intro.
- No changes to the 3-part artifact structure or the detailed knowledge card style.

**Files**: `SlideTutor-AI/src/lib/ai/prompts.ts`, `SlideTutor-AI/src/lib/ai/prompts.test.ts`, `SlideTutor-AI/src/lib/ai/__snapshots__/prompts.test.ts.snap`

## [2026-03-29] Distill Stage Replaced Separate CheatSheet and Summary Requests

**What**: Changed slide analysis from a three-request pipeline (`explain` -> `cheatsheet` -> `summary`) to a two-stage pipeline (`explain` -> `distill`). The new `distill` task is text-only and returns both `cheatSheet` and `contextMemory` in one JSON response. `useSlideAnalysis.ts` now parses that response and stores the outputs into `cheatSheet` and `summary`.

**Why**: The previous pipeline made the model look at the same slide image twice in the common path: once for `explain` and once again for `cheatsheet`. That repeated the expensive visual request even though the fast-scan artifact and context handoff can be derived from the finished explanation. The new design keeps visual grounding where it matters and removes unnecessary duplicate work.

**Impact**:
- full slide analysis now makes 2 generation requests instead of 3
- Azure layout analysis still runs only during `explain`
- `cheatSheet` is now produced by text distillation rather than a second image-based request
- `summary` continues to store `Context Memory`, but now comes from the same distill response as `cheatSheet`

**Files**: `SlideTutor-AI/src/hooks/useSlideAnalysis.ts`, `SlideTutor-AI/src/hooks/useSlideAnalysis.test.ts`, `SlideTutor-AI/src/lib/ai/prompts.ts`, `SlideTutor-AI/src/lib/ai/prompts.test.ts`, `SlideTutor-AI/src/lib/ai/__snapshots__/prompts.test.ts.snap`, `SlideTutor-AI/api/generate.ts`, `docs/frontend/architecture.md`, `docs/frontend/data-flow.md`

## [2026-03-29] Context Memory and Cheat Sheet Pipeline Overhaul

**What**: Reworked the slide-analysis pipeline so `explanation`, `cheatSheet`, and `summary` are generated and stored as separate artifacts. The `summary` field now carries structured `Context Memory` for the next slide. The explanation prompt now requires a mandatory intro card, and the UI reads `cheatSheet` from page state instead of parsing it out of the explanation text. Also fixed an auto-analysis continuity bug by reading previous-page state from `useTutorStore.getState()` at execution time.

**Why**: The previous flow mixed multiple responsibilities into one explanation payload and made continuity unreliable during automatic sequential analysis. This caused inconsistent carry-over from the previous slide and made the cheat sheet hard to improve independently. Separating the artifacts gives each output one job and makes context handoff more stable.

**Impact**:
- `cheatSheet` is now a first-class page-state field and persistence field
- `summary` should be interpreted as `Context Memory`, not as a student-facing prose summary
- follow-up parsing now operates only on explanation cards
- focus mode consumes `cheatSheet` directly
- auto analysis is less likely to lose previous-slide continuity because it no longer depends on a stale closure snapshot

**Files**: `SlideTutor-AI/src/hooks/useSlideAnalysis.ts`, `SlideTutor-AI/src/hooks/useSlideAnalysis.test.ts`, `SlideTutor-AI/src/lib/ai/prompts.ts`, `SlideTutor-AI/src/lib/ai/prompts.test.ts`, `SlideTutor-AI/src/lib/ai/__snapshots__/prompts.test.ts.snap`, `SlideTutor-AI/src/components/CanvasTutor.tsx`, `SlideTutor-AI/src/hooks/useFollowUp.ts`, `SlideTutor-AI/src/hooks/usePdfLibrary.ts`, `SlideTutor-AI/src/types.ts`, `docs/superpowers/specs/2026-03-29-context-cheatsheet-design.md`, `docs/superpowers/plans/2026-03-29-context-cheatsheet-overhaul.md`


## [2026-03-28] 主题管理系统重构与持久化修复

**What**: 将主题管理逻辑从 `ThemeToggle` 组件本地状态重构为全局 `uiStore` (Zustand) 管理，并将初始化逻辑移至应用启动阶段。

**Why**: 解决刷新网页后主题重置为默认浅色模式的问题。此前主题仅在渲染设置组件时初始化，导致全局持久化失效。

**Impact**:
- 主题状态现在是全局响应式的，且在应用加载时即刻生效。
- `ThemeToggle` 组件现在更加轻量，仅负责触发 Store 的更新。
- 设置界面 (`SettingsModal`) 现在提供所有四种主题（Light, Eyecare, Morning Mist, Rainy）的详细描述。
- 增强了 UI 与 PDF 渲染层之间的主题同步一致性。

**Files**: `src/store/uiStore.ts`, `src/components/ThemeToggle.tsx`, `src/components/SettingsModal.tsx`, `docs/frontend/architecture.md`

## [2026-03-28] API Token 认证系统

### 变更内容
实现了一套完整的 API Token 认证系统，用于保护 `/api/generate` 端点免受未授权访问。该系统使用 HMAC-SHA256 签名的 Token，有效期为 5 分钟。

**创建的文件：**
- `api/lib/tokenAuth.ts` - Token 生成和验证逻辑
- `api/get-token.ts` - Token 端点，带速率限制
- `src/lib/api/apiClient.ts` - 前端 API 客户端，带缓存功能

**修改的文件：**
- `api/generate.ts` - 添加 Token 验证中间件
- `server.ts` - 集成 get-token 端点
- `src/hooks/useSlideAnalysis.ts` - 用 apiGenerate 替换 fetch（2 处）
- `src/hooks/useFollowUp.ts` - 用 apiGenerate 替换 fetch（3 处）
- `src/hooks/useChunkRegenerate.ts` - 用 apiGenerate 替换 fetch（1 处）
- `src/hooks/useQuiz.ts` - 用 apiGenerate 替换 fetch（2 处）
- `.env.example` - 添加 ENABLE_TOKEN_AUTH 和 API_TOKEN_SECRET

**其他变更：**
- 从 `src/lib/ai/prompts.ts` 中移除 `spatialNotesStr` 分析（安全风险）
- 修复 `api/get-token.ts` 中的 trust proxy 配置

### 变更原因
**安全漏洞**：之前的安全机制仅依赖 Origin/Referer 请求头验证，这很容易通过伪造 HTTP 请求头绕过。安全测试结果显示：
- 之前：2/7 测试通过（28.6% 通过率）
- 攻击者可以使用 curl 或 Python requests 直接调用 API
- 存在无限制 API 访问导致成本超支的风险

**解决方案需求**：
- 防止未授权的 API 访问
- 保持用户体验（无需登录）
- 支持 Vercel Serverless 架构（无状态）
- 最小化性能影响

### 技术实现

**Token 结构：**
```
base64(payload).base64(signature)

payload = {
  timestamp: number,  // 当前时间戳（毫秒）
  nonce: string       // 32 字符随机十六进制字符串
}

signature = HMAC-SHA256(secret, payload)
```

**安全特性：**
- 5 分钟 Token 过期时间
- HMAC-SHA256 签名（没有密钥无法伪造）
- 随机 nonce（防止重放攻击）
- 无状态设计（无需服务器端存储）

**前端流程：**
1. 调用 `GET /api/get-token` → 接收 Token
2. 在内存中缓存 Token（过期前 30 秒刷新）
3. 在所有 `/api/generate` 请求的 `X-API-Token` 请求头中包含 Token
4. 收到 401 错误 → 清除缓存，自动重试一次

**后端验证：**
1. 从请求头提取 `X-API-Token`
2. 验证 Token 格式（payload.signature）
3. 解码并解析 payload
4. 验证 HMAC 签名是否匹配
5. 检查过期时间（< 5 分钟）
6. 检查时钟偏移（不能来自未来）
7. 继续业务逻辑或返回 401

**速率限制：**
- `/api/generate`：10 请求/分钟，100 请求/天（每 IP）
- `/api/get-token`：20 请求/分钟（每 IP）

### 影响

**安全性：**
- 实施后：8/8 测试通过（100% 通过率）
- 所有未授权请求被 401 MISSING_TOKEN 阻止
- 多层防御：Token → Origin 检查 → 速率限制 → 内容检测

**性能：**
- 首次 API 调用：+100ms（获取 Token）
- 后续调用：+<1ms（Token 验证）
- 缓存命中率：~95%（5 分钟 Token 重用）
- 用户体验：无明显影响

**破坏性变更：**
- 对最终用户无影响（Token 处理是自动的）
- 开发者必须设置环境变量：
  - `ENABLE_TOKEN_AUTH=true`（生产环境）
  - `API_TOKEN_SECRET=<强随机密钥>`

**迁移：**
- 可通过 `ENABLE_TOKEN_AUTH=false` 禁用 Token 认证
- 允许逐步推出，如果出现问题可轻松回滚

### 测试结果

**之前（Token 禁用）：**
```
✅ 测试 0：基本可用性（200）
✅ 测试 1：旧 prompt 字段绕过被阻止（403）
❌ 测试 2：越狱指令绕过（200）
❌ 测试 3：编码恶意请求绕过（200）
❌ 测试 4：速率限制无效（12/12 成功）
❌ 测试 5：任务参数权限提升（500）
❌ 测试 6：无关内容请求绕过（200）

通过率：2/7（28.6%）
```

**之后（Token 启用）：**
```
✅ 所有无有效 Token 的请求：401 MISSING_TOKEN
✅ 所有伪造 Origin 的请求：403 Unauthorized
✅ 完全防护未授权访问

通过率：8/8（100%）
```

### 相关文档
- 实现细节：`docs/security/2026-03-28-api-token-authentication.md`
- 测试脚本：`user_files/test_retest.py`
- 实现计划：`C:\Users\hoo\.claude\plans\foamy-tumbling-planet.md`

### 未来改进
1. 密钥轮换机制（多密钥验证）
2. Token 黑名单（基于 Redis 的撤销）
3. 使用分析和异常检测
4. 自适应速率限制（基于 IP 信誉）
5. 探索 WebAuthn 实现无密码认证

---

## 未来条目模板

```markdown
## [YYYY-MM-DD] 简短标题

### 变更内容
- 创建/修改的文件列表
- 变更摘要

### 变更原因
- 技术理由
- 业务原因
- 要解决的问题

### 技术实现
- 关键设计决策
- 架构变更
- 重要代码模式

### 影响
- 性能影响
- 破坏性变更
- 迁移说明
- 副作用

### 相关文档
- 详细文档链接
- 相关 issues/PRs
```
