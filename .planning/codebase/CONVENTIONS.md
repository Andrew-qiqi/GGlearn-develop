# Coding Conventions

**Analysis Date:** 2025-03-27

## Naming Patterns

**Files:**
- React Components: PascalCase (e.g., `src/components/PdfViewer.tsx`, `src/components/CanvasTutor.tsx`)
- Hooks: camelCase starting with `use` (e.g., `src/hooks/useFollowUp.ts`, `src/hooks/useSlideAnalysis.ts`)
- Utilities/Logic: camelCase (e.g., `src/lib/db.ts`, `src/lib/ai/socraticProbe.ts`)
- Styles: `index.css` (global Tailwind styles)
- Types: `src/types.ts` (shared types)

**Functions:**
- camelCase (e.g., `handleSendMessage`, `updateFollowUp`, `extractSocraticProbe`)

**Variables:**
- camelCase (e.g., `isSettingsOpen`, `currentPdfId`, `pagesState`)
- Refs: camelCase with `Ref` suffix (e.g., `pdfViewerRef`, `currentPdfIdRef`)

**Types:**
- PascalCase for interfaces and types (e.g., `UiState`, `PageState`, `FollowUp`, `SavedPDF`)

## Code Style

**Formatting:**
- No explicit Prettier configuration found; follows standard TypeScript/React conventions.
- Single quotes used for strings.
- 2-space indentation.

**Linting:**
- Uses `tsc --noEmit` for type checking as the primary linting step.
- No ESLint configuration detected.

## Import Organization

**Order:**
1. React hooks and core libraries (`useState`, `useEffect`, `uuid`)
2. Local components (`./components/xxx`)
3. Libraries and Utilities (`./lib/xxx`)
4. Stores (`./store/xxx`)
5. Hooks (`./hooks/xxx`)

**Path Aliases:**
- `@/*` maps to `./*` as defined in `tsconfig.json` and `vite.config.ts`.

## Error Handling

**Patterns:**
- Try-catch blocks for asynchronous operations (API calls, IndexedDB).
- Explicit error throwing with descriptive messages: `throw new Error("Failed to extract slide image.");`
- Error messages displayed to users via state updates (e.g., `updateFollowUp(errorMessage)`).
- Rate limit (429) handling with specific user-friendly messages in `src/hooks/useFollowUp.ts`.

## Logging

**Framework:** `console`

**Patterns:**
- `console.log` for server-side API request logging in `server.ts`.
- `console.error` for catching and reporting errors in hooks and API routes.

## Comments

**When to Comment:**
- Used for section headers in large files (e.g., `// --- SECURITY MIDDLEWARE ---` in `api/generate.ts`).
- Brief explanations for non-obvious logic or configuration.

**JSDoc/TSDoc:**
- Not widely used; types are primarily defined via TypeScript interfaces.

## Function Design

**Size:**
- Hooks can be large (e.g., `useFollowUp.ts` is ~480 lines), often containing multiple related event handlers.
- Pure logic is extracted into smaller, testable functions (e.g., `socraticProbe.ts`).

**Parameters:**
- Mixed: some functions take multiple arguments, others take option objects (e.g., `resolveFollowUpSubmission`).

**Return Values:**
- Hooks typically return an object containing multiple state variables and handler functions.
- Pure functions return explicit types or result objects.

## Module Design

**Exports:**
- Named exports for utilities and stores.
- Default exports for main components (e.g., `App.tsx`) and the API server (`api/generate.ts`).

**Barrel Files:**
- Not heavily used; components and hooks are imported directly from their files.

---

*Convention analysis: 2025-03-27*
