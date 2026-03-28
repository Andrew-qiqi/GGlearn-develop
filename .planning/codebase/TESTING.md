# Testing Patterns

**Analysis Date:** 2025-03-27

## Test Framework

**Runner:**
- Vitest ^4.1.0
- Config: `SlideTutor-AI/vite.config.ts`

**Assertion Library:**
- Included in Vitest (`expect`).
- `@testing-library/jest-dom` for DOM assertions (not yet widely used in existing tests).

**Run Commands:**
```bash
npm test              # Run all tests using Vitest
```

## Test File Organization

**Location:**
- Co-located with source files (e.g., `src/components/tutorCardPresentation.test.ts`, `src/hooks/followUpSubmission.test.ts`).

**Naming:**
- `*.test.ts`

**Structure:**
```
SlideTutor-AI/
├── src/
│   ├── components/
│   │   ├── tutorCardPresentation.ts
│   │   └── tutorCardPresentation.test.ts
│   ├── hooks/
│   │   ├── followUpSubmission.ts
│   │   └── followUpSubmission.test.ts
│   └── test/
│       └── setup.ts          # Vitest setup file
```

## Test Structure

**Suite Organization:**
```typescript
import { describe, expect, it } from 'vitest';
import { someFunction } from './someFile';

describe('someFunction', () => {
  it('does something expected', () => {
    const result = someFunction('input');
    expect(result).toBe('expected output');
  });
});
```

**Patterns:**
- `describe` blocks for function names.
- `it` blocks for specific behaviors and scenarios.
- Direct assertions on return values.

## Mocking

**Framework:** Vitest (`vi`).

**Patterns:**
- No complex mocking observed in the current test suite.
- Focus is on testing pure logic extracted from React components/hooks.

**What to Mock:**
- External API calls (not yet seen in existing tests, but required for `useFollowUp.ts`).
- Browser APIs like IndexedDB or PDF.js extraction (typically mocked in `jsdom`).

**What NOT to Mock:**
- Simple data processing functions.
- Pure utility functions.

## Fixtures and Factories

**Test Data:**
```typescript
const probeContext = {
  probeText: 'Why does this happen?',
  source: 'explanation' as const,
  sourcePage: 3,
  sourceChunkId: 'page-3-chunk-1',
};
```
- Defined as local constants within test files.

**Location:**
- Inside test files (`*.test.ts`).

## Coverage

**Requirements:** None enforced in `package.json`.

**View Coverage:**
```bash
npx vitest run --coverage
```

## Test Types

**Unit Tests:**
- Primary focus: testing logic extracted from hooks and components (e.g., `src/hooks/followUpSubmission.test.ts`, `src/lib/ai/socraticProbe.test.ts`).
- Verifies edge cases and core business logic.

**Integration Tests:**
- Not explicitly labeled, but logic tests for submission behavior (e.g., `resolveFollowUpSubmission`) cover multi-step logic.

**E2E Tests:**
- Not used (no Playwright/Cypress).

## Common Patterns

**Async Testing:**
- Standard Vitest `async/await` patterns (though current tests are mostly synchronous).

**Error Testing:**
- Expecting functions to throw or return specific error states (not yet widely seen in existing tests).

---

*Testing analysis: 2025-03-27*
