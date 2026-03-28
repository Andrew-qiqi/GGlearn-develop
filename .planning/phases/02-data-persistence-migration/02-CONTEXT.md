# Phase 2: Data Persistence Migration

## Goal
Transition all application data from `LocalStorage` to `IndexedDB` to ensure robust storage and better data handling for larger datasets.

## Status
- **Phase**: 2
- **Status**: Planning
- **Depends on**: Phase 1 (Environment & Core Stability)

## Requirements
- **DATA-01: Robust Storage** —— Implement migration from LocalStorage to IndexedDB.
- **DATA-02: Data Compatibility** —— Establish data migration mechanism.

## Success Criteria
1. User data is successfully migrated from LocalStorage to IndexedDB without loss.
2. Application state (current slide, settings, theme) is preserved across sessions.
3. Subsequent updates do not break compatibility with existing IndexedDB data.

## Key Data to Migrate
1. `slide_tutor_model` (AI Settings)
2. `slide_tutor_language` (Output Language)
3. `theme` (UI Theme)
4. `slide_tutor_focus_mode` (UI Mode)
5. `pdf-page-${pdf.id}` (Last read page for each PDF)
