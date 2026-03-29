---
name: maintain-tech-docs
description: >
  Helps keep technical documentation in sync with code changes. When you complete features, refactor architecture, or fix complex bugs, this skill guides you through updating modular architecture docs and CHANGELOG_TECH.md so future developers understand what changed and why.
---

# Technical Documentation Maintenance

## Purpose

Technical documentation often falls behind code changes, creating knowledge gaps that slow down future development. This skill helps you maintain living documentation that captures architectural decisions, implementation details, and the reasoning behind changes.

Think of this as leaving breadcrumbs for your future self and teammates. When someone asks "why did we build it this way?" six months from now, the answer should be in the docs.

## When to Use This Skill

Consider using this skill after:

- **Feature implementation**: You added new functionality that introduces new modules, services, or architectural patterns
- **Significant refactoring**: You restructured code, changed design patterns, or reorganized the codebase
- **Complex bug fixes**: The fix required understanding or modifying core logic, not just a one-line change
- **Dependency changes**: You added, removed, or upgraded major libraries or frameworks
- **Architecture decisions**: You made choices about data flow, state management, API design, or system boundaries

**Rule of thumb**: If explaining your changes would take more than 2 minutes, they probably deserve documentation.

## What to Update

### 1. Architecture Documentation (Modular Structure)

The project uses a **modular documentation structure** under `docs/`. Update the appropriate module based on your changes:

#### `docs/architecture/` - System Architecture
Update when changes affect:
- **System structure**: New modules, services, or layers → Update `system-overview.md`
- **Technology stack**: New libraries, frameworks, or tools → Update `tech-stack.md`
- **Deployment**: Infrastructure, performance, or monitoring → Update `deployment.md`

#### `docs/frontend/` - Frontend Architecture
Update when changes affect:
- **Frontend architecture**: State management, component structure → Update `architecture.md`
- **Data flow**: How data moves through the frontend → Update `data-flow.md`

#### `docs/backend/` - Backend Architecture
Update when changes affect:
- **API design**: New endpoints, request/response formats → Update `api-design.md`
- **Error handling**: Error strategies, logging → Update `error-handling.md`

#### `docs/security/` - Security Architecture
Update when changes affect:
- **Security mechanisms**: Authentication, authorization → Update `architecture.md`
- **Token systems**: JWT, API tokens → Update `token-authentication.md`
- **Rate limiting**: Throttling, abuse prevention → Update `rate-limiting.md`

**What to include**:
- High-level overview of what changed
- Diagrams or ASCII art if it helps (optional but valuable)
- Trade-offs you considered
- Links to relevant code files or PRs

**Example entry** (in `docs/security/token-authentication.md`):
```markdown
## JWT Authentication System (Added 2026-03-15)

Implemented JWT-based authentication with refresh tokens.

**Structure**:
- `src/auth/jwt-service.ts`: Token generation and validation
- `src/auth/middleware.ts`: Express middleware for route protection
- `src/auth/refresh-handler.ts`: Refresh token rotation logic

**Why JWT**: Chose JWT over session-based auth because:
- Stateless: No server-side session storage needed
- Scalable: Works across multiple server instances
- Mobile-friendly: Easy to use in mobile apps

**Trade-offs**:
- Cannot invalidate tokens before expiry (mitigated with short expiry + refresh tokens)
- Slightly larger payload than session IDs

**Dependencies**: `jsonwebtoken` v9.0.0, `bcrypt` v5.1.0
```

### 2. Technical Changelog (`docs/changelog/CHANGELOG_TECH.md`)

This is your project's technical diary. Add entries in reverse chronological order (newest first).

**Every entry should answer**:
- **What changed**: Concrete description of the change
- **Why**: The problem you were solving or requirement you were meeting
- **Impact**: What developers need to know (breaking changes, new patterns, performance implications)

**Format template**:
```markdown
## [YYYY-MM-DD] Brief Title

**What**: [Description of changes]

**Why**: [Technical rationale or business reason]

**Impact**: [Side effects, breaking changes, or important notes]

**Files**: [Key files modified, optional]
```

**Example entries for different change types**:

#### Feature Addition
```markdown
## [2026-03-20] Real-time Notification System

**What**: Added WebSocket-based notification system for live updates. Users now receive instant notifications for comments, mentions, and status changes without polling.

**Why**: Previous polling approach caused unnecessary server load and 30-second delays. Real-time updates improve UX and reduce API calls by ~60%.

**Impact**:
- New dependency: `socket.io` v4.6.0
- New environment variable required: `WEBSOCKET_PORT`
- Frontend must connect to WebSocket endpoint on app initialization
- Existing REST notification endpoints remain for backward compatibility

**Files**: `src/notifications/websocket-server.ts`, `src/notifications/event-emitter.ts`
```

#### Refactoring
```markdown
## [2026-03-18] Database Query Layer Refactoring

**What**: Extracted raw SQL queries into a repository pattern with TypeORM query builders. Centralized all database access in `src/repositories/`.

**Why**: Direct SQL queries were scattered across service files, making it hard to track database usage and optimize queries. Repository pattern provides better testability and query reuse.

**Impact**:
- All services now depend on repository interfaces instead of direct DB access
- Easier to mock for unit tests
- Breaking change: Old `db.query()` calls removed, use repositories instead
- Migration guide: See `docs/migrations/repository-pattern.md`

**Files**: `src/repositories/*.ts`, `src/services/*.ts` (updated imports)
```

#### Bug Fix (Complex)
```markdown
## [2026-03-15] Fixed Race Condition in Payment Processing

**What**: Added distributed locking mechanism using Redis to prevent duplicate payment charges when users double-click the submit button.

**Why**: Race condition allowed multiple payment requests to process simultaneously, causing duplicate charges. Standard request deduplication wasn't sufficient due to async payment gateway callbacks.

**Impact**:
- New Redis dependency for distributed locks
- Payment processing now has 30-second lock timeout
- If Redis is unavailable, payments fail-safe (reject rather than risk duplicates)
- Added monitoring for lock timeouts in `src/monitoring/payment-locks.ts`

**Files**: `src/payments/payment-processor.ts`, `src/infrastructure/redis-lock.ts`
```

#### Dependency Change
```markdown
## [2026-03-12] Migrated from Moment.js to date-fns

**What**: Replaced all Moment.js usage with date-fns for date manipulation and formatting.

**Why**: Moment.js is deprecated and adds 67KB to bundle size. date-fns is tree-shakeable, reducing bundle by ~50KB, and has better TypeScript support.

**Impact**:
- Breaking change: Date formatting functions have new signatures
- All date utilities moved to `src/utils/date-helpers.ts`
- Update imports: `import { format } from 'date-fns'` instead of `moment()`
- See migration guide: `docs/migrations/moment-to-datefns.md`

**Files**: `src/utils/date-helpers.ts`, `package.json`
```

## Documentation Checklist

Before marking your development task as complete, verify:

- [ ] **Module docs updated** (choose the appropriate module based on your changes):
  - [ ] `docs/architecture/` - System structure, tech stack, or deployment changes
  - [ ] `docs/frontend/` - Frontend architecture or data flow changes
  - [ ] `docs/backend/` - API design or error handling changes
  - [ ] `docs/security/` - Security mechanisms, authentication, or rate limiting changes
- [ ] **Changelog entry added** in `docs/changelog/CHANGELOG_TECH.md` with What/Why/Impact
- [ ] **Date is correct** (use YYYY-MM-DD format)
- [ ] **Breaking changes highlighted** (if any)
- [ ] **New dependencies documented** (if any)
- [ ] **Migration notes provided** (if needed for other developers)

## Tips for Effective Documentation

**Be specific**: Instead of "improved performance", write "reduced API response time from 800ms to 200ms by adding database indexes on user_id and created_at columns"

**Explain trade-offs**: Document what you considered and why you chose this approach. Future developers will thank you when requirements change.

**Link to context**: Reference PR numbers, issue tickets, or design docs that provide more background.

**Write for your future self**: Imagine reading this 6 months from now with no memory of the change. Would you understand it?

**Keep it concise**: Aim for clarity, not length. A well-structured 5-line entry beats a rambling paragraph.

## When NOT to Document

You can skip documentation for:
- Trivial changes (typo fixes, formatting, minor refactoring)
- Changes that don't affect other developers (local dev environment tweaks)
- Temporary debugging code
- Changes fully explained by commit messages and PR descriptions

## Workflow Integration

This skill works best when integrated into your development workflow:

1. **During development**: Keep notes on architectural decisions as you code
2. **Before committing**: Review your changes and identify what needs documentation
3. **After PR approval**: Update docs before merging (or as part of the PR)
4. **During code review**: Reviewers can check if docs are updated

## Common Pitfalls to Avoid

**Too vague**: "Updated authentication" → Better: "Added OAuth2 support for Google and GitHub login"

**Missing 'why'**: Documenting what changed without explaining the reasoning leaves future developers guessing

**Outdated entries**: If you change your approach during development, update the docs to match the final implementation

**Over-documenting**: Not every line of code needs documentation. Focus on architectural decisions and non-obvious choices

## Final Note

Good documentation is an investment in your project's future. It reduces onboarding time, prevents repeated mistakes, and preserves institutional knowledge. When in doubt, document it—you can always refine later, but lost context is hard to recover.

Remember: Code tells you *how*, documentation tells you *why*.
