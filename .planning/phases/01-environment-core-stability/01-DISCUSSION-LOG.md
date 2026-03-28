# Phase 01: Environment & Core Stability - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-03-26
**Phase:** 01-Environment & Core Stability
**Areas discussed:** API Security, Regression Baseline, Stability Guard

---

## API Security (SEC-01)

| Option | Description | Selected |
|--------|-------------|----------|
| Mandatory Startup Validation | Explicitly check env vars at the beginning. | ✓ |
| On-demand Detection | Check only when used. | |

**User's choice:** Mandatory Startup Validation.
**Notes:** Added requirement for Rate Limiting to prevent "abuse/scraping" of endpoints.

---

## Regression Baseline (STAB-02)

| Option | Description | Selected |
|--------|-------------|----------|
| Core Logic Unit Testing | Test PDF/AI logic directly. | ✓ |
| End-to-End (E2E) Testing | Test full user paths. | |

**User's choice:** Core Logic Unit Testing.
**Notes:** User emphasized full coverage of core features and mentioned "I will test it myself."

---

## Stability Guard (STAB-01)

| Option | Description | Selected |
|--------|-------------|----------|
| Contract Locking | Prevent API changes. | ✓ |
| Doc Specification | Written guidelines only. | |

**User's choice:** Contract Locking + Non-Destructive Polishing.
**Notes:** User demands first-principles justification for all logic changes.

---

## Claude's Discretion
- Choice of rate-limiting implementation details.

## Deferred Ideas
- Transition to IndexedDB (deferred to Phase 2).
