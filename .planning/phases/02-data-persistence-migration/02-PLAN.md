# Plan: Phase 2 - Data Persistence Migration

## Research & Exploration
- [x] Identify all `localStorage` usage in the codebase.
- [ ] Explore the current `IndexedDB` schema in `src/lib/db.ts`.
- [ ] Design a new `settings` object store in `IndexedDB`.

## Implementation Tasks
### 1. Update `src/lib/db.ts` Schema
- [ ] Add a `settings` object store to `IndexedDB`.
- [ ] Update `SavedPDF` interface to include `lastReadPage`.
- [ ] Create functions for getting/setting settings.
- [ ] Update `updatePDFState` or add `updateLastReadPage` to persist current page number.

### 2. Implement Migration Logic
- [ ] Create a `src/lib/migrate.ts` script to check for `localStorage` and migrate to `IndexedDB`.
- [ ] Trigger migration on application startup.
- [ ] Ensure `localStorage` is only cleared after successful migration.

### 3. Refactor Stores
- [ ] Update `useUiStore` to use IndexedDB for AI model, language, and theme.
- [ ] Update `useTutorStore` to use IndexedDB for focus mode.
- [ ] Update `usePdfLibrary` and `App.tsx` to store/load last read page from IndexedDB instead of `localStorage`.

### 4. Regression Safety
- [ ] Ensure existing PDF files and their state are still accessible.
- [ ] Verify that new data is correctly persisted in IndexedDB.

## Verification Tasks
- [ ] **Data Migration**: Manually verify `localStorage` data is migrated on first load.
- [ ] **Persistence**: Change settings/theme/page, reload, and verify they are preserved.
- [ ] **PDF State**: Verify that notes and explanations for existing PDFs are still there.
- [ ] **Error Handling**: Verify that the app still works if IndexedDB is unavailable (graceful degradation or fallback).

## Testing Strategy
- [ ] Create a test suite for `migrate.ts` if possible.
- [ ] Verify `db.ts` functions with mock IndexedDB if available.
