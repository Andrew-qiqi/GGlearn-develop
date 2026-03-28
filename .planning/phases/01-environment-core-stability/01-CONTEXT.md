# Phase 01: Environment & Core Stability - Context

**Gathered:** March 26, 2026
**Status:** Ready for planning

<domain>
## Phase Boundary

Secure the application environment and ensure a baseline for stable iteration by hardening API key handling, adding rate limiting, and establishing a core regression test suite.
</domain>

<decisions>
## Implementation Decisions

### API Security (SEC-01)
- **D-01:** **Mandatory Startup Validation** — Validate existence and basic format of required environment variables (e.g., `GEMINI_API_KEY`) at app startup/function initialization.
- **D-04:** **Endpoint Rate Limiting** — Implement IP-based rate limiting for sensitive API routes (e.g., `/api/generate`) to prevent credit drain and abuse.

### Regression Safety (STAB-02)
- **D-02:** **Full Core Coverage** — Establish unit tests for critical business logic, specifically PDF parsing/navigation and AI prompt construction/generation.
- **D-03:** **Contract Locking** — Explicitly lock API interface contracts to ensure existing functional logic remains untouched during future UI/UX polish.

### Stability Guard (STAB-01)
- **D-05:** **Non-Destructive Polishing** — All interaction improvements must be additive or "smoothing," never altering the proven core operational flow.
- **D-06:** **First-Principles Logic** — Every modification must be justified via first-principles thinking and logical consistency, requiring user understanding and approval of the underlying "why."

### Claude's Discretion
- Implementation details for the validation layer and specific rate-limit thresholds (within reasonable bounds).

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Project Core
- `.planning/PROJECT.md` — Vision and Core Values.
- `.planning/REQUIREMENTS.md` — V1 requirements (SEC, STAB).
- `SlideTutor-AI/package.json` — Dependency and script reference (vitest, dotenv).

### Existing Logic
- `SlideTutor-AI/api/generate.ts` — Main API endpoint logic to be secured and tested.

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `vitest`: Already configured for unit testing.
- `dotenv`: Used for environment variable management.

### Integration Points
- `process.env`: Central point for environment variable injection (Vercel standard).
- `api/generate.ts`: The primary target for security hardening and rate limiting.

</code_context>

<specifics>
## Specific Ideas
- "I will test them myself" — Decisions must prioritize developer-facing clarity for collaborative testing.
- "Explain problems and logic clearly" — High emphasis on technical transparency and rationale.

</specifics>

<deferred>
## Deferred Ideas
- IndexedDB migration (Phase 2)
- UI ergonomics refinement (Phase 3)

</deferred>

---

*Phase: 01-environment-core-stability*
*Context gathered: March 26, 2026*
