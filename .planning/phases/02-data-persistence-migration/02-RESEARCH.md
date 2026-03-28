# Research: Phase 2 - Data Persistence Migration

## Current State Analysis

### LocalStorage Usage
The following keys are used in `localStorage`:
- `slide_tutor_model`: AI model configuration (JSON string).
- `slide_tutor_language`: User preferred output language.
- `slide_tutor_focus_mode`: Boolean string for UI focus mode.
- `pdf-page-${pdf.id}`: Last read page for a specific PDF.
- `theme`: UI theme ('light' or 'dark').

### IndexedDB Usage
The following object store exists in `SlideTutorDB`:
- `pdfs`: Stores `SavedPDF` objects.
  - `id`: string (KeyPath)
  - `name`: string
  - `fileData`: ArrayBuffer
  - `pagesState`: Record<number, any>
  - `lastAccessed`: number
  - `numPages`: number (optional)
  - `tags`: string[] (optional)

## Proposed IndexedDB Schema Changes

### New Object Store: `appSettings`
- `key`: string (The setting name)
- `value`: any (The setting value)

### Update to `pdfs` Store
- Add `lastReadPage`: number (to store current page per PDF)

## Migration Strategy
1.  **Detection**: On app load, check if any target `localStorage` keys exist.
2.  **Backup**: (Optional) Read all `localStorage` data into an object.
3.  **Persist**: Store the data in the new `appSettings` and `pdfs` stores in IndexedDB.
4.  **Cleanup**: Once confirmed, remove the keys from `localStorage`.
5.  **Synchronization**: Ensure that store initialization waits for migration to complete.

## Risks & Mitigations
- **IndexedDB Unavailable**: If the browser's storage is disabled, we should fallback to `localStorage` or at least not crash.
- **Data Loss**: If migration fails mid-way, we might lose data.
  - *Mitigation*: Only clear `localStorage` after confirming all data is in IndexedDB.
- **Race Conditions**: Two parts of the app trying to initialize the store simultaneously.
  - *Mitigation*: Use a single initialization/migration entry point.
