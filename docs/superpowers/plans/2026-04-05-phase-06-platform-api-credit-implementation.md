# Phase 06 Platform API and Credits Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add neutral `My API` / `Platform API` access modes with Clerk-backed login, D1-backed hosted credits, settings-only recharge UX, and success-only hosted billing for `Analyze`, `Follow-up`, and quiz actions without breaking mature teaching logic.

**Architecture:** Keep BYOK and hosted access as explicit frontend modes. Platform mode sends a Clerk session token plus `mode: "platform"` access metadata to the Worker, which verifies the user, reads and writes D1-backed balances, and only spends credits after successful hosted actions. Preserve the existing `explain -> distill` frontend flow by introducing a D1-backed pending analyze attempt: `explain` reserves eligibility and returns an attempt id, `distill` finalizes and commits the single `Analyze = 3` deduction only after `parse + explain + distill` all succeed.

**Tech Stack:** React 19, Vite, Zustand, Clerk React, Clerk Backend SDK, Cloudflare Worker, D1, Vitest, existing generation service and parser access layer

---

## Scope Locks

- This plan implements the approved product spec in [2026-04-05-phase-06-platform-api-credit-product-design.md](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/docs/superpowers/specs/2026-04-05-phase-06-platform-api-credit-product-design.md).
- `My API` stays browser-local and never uses platform credits.
- `Platform API` requires login at mode-switch time, not later at action time.
- Hosted billing covers only approved priced actions in this phase:
  - `Analyze = 3`
  - `followup = 1`
  - `generate_questions = 1`
  - `evaluate_answers = 1`
- Hosted `Analyze` must not charge for degraded parser results.
- Secondary hosted actions stay out of scope until product pricing exists:
  - `regenerate_chunk`
  - `regenerate_followup`
  - `evaluate_note`
- Recharge provider specifics remain isolated behind an adapter. This plan still builds the full recharge contract and a `mock` dev adapter. If real merchant docs and credentials are available at execution time, replace the mock adapter in the dedicated payment task.

## File Map

### Frontend access state and auth

- Modify: `SlideTutor-AI/src/config/models.ts`
  - Add explicit access-mode types and normalize away old BYOK-only assumptions.
- Modify: `SlideTutor-AI/src/store/uiStore.ts`
  - Persist `accessMode`, hold hosted balance snapshot state, and hold insufficient-credit dialog state.
- Modify: `SlideTutor-AI/src/store/uiStore.test.ts`
  - Cover new persisted access mode and hosted UI state behavior.
- Create: `SlideTutor-AI/src/lib/auth/clerk.tsx`
  - Wrap `ClerkProvider`, centralize publishable-key lookup, and expose helper hooks for platform login and session-token fetch.
- Modify: `SlideTutor-AI/src/main.tsx`
  - Mount the app under the Clerk provider.
- Modify: `SlideTutor-AI/package.json`
  - Add Clerk dependencies.

### Frontend API and hosted UI

- Modify: `SlideTutor-AI/src/lib/api/apiClient.ts`
  - Send `mode: "platform"` or BYOK payloads, attach `Authorization: Bearer ...` for hosted requests, fetch hosted balance, and create recharge intents.
- Modify: `SlideTutor-AI/src/lib/api/apiClient.test.ts`
  - Cover hosted auth headers and hosted payload shape.
- Create: `SlideTutor-AI/src/lib/platformAccess/pricing.ts`
  - Frontend constants for action pricing and RMB-to-credit conversion.
- Create: `SlideTutor-AI/src/lib/platformAccess/pricing.test.ts`
  - Cover `1 RMB = 30 credits` conversion and action price constants.
- Create: `SlideTutor-AI/src/components/settings/PlatformApiSection.tsx`
  - Render neutral mode toggle, hosted balance, price summary, and `Buy Credits`.
- Create: `SlideTutor-AI/src/components/settings/BuyCreditsDialog.tsx`
  - Render RMB input, live credit quote, and recharge submit action.
- Create: `SlideTutor-AI/src/components/CreditsRequiredDialog.tsx`
  - Render the global insufficient-credit prompt with `Buy Credits` and `Switch to My API`.
- Modify: `SlideTutor-AI/src/components/SettingsModal.tsx`
  - Delegate AI tab UI into focused subcomponents instead of expanding the current monolith.
- Modify: `SlideTutor-AI/src/components/SettingsModal.test.tsx`
  - Cover hosted balance rendering, neutral mode switch, and live recharge quote.
- Modify: `SlideTutor-AI/src/App.tsx`
  - Mount the global insufficient-credit dialog and pass through hosted UI handlers.

### Frontend action hooks

- Modify: `SlideTutor-AI/src/hooks/useSlideAnalysis.ts`
  - Capture hosted analyze attempt ids and pass them into `distill` finalize calls.
- Modify: `SlideTutor-AI/src/hooks/useFollowUp.ts`
  - Open the insufficient-credit dialog on hosted billing errors instead of leaking raw quota text.
- Modify: `SlideTutor-AI/src/hooks/useQuiz.ts`
  - Same hosted billing handling for generate/evaluate.
- Modify: `SlideTutor-AI/src/hooks/useChunkRegenerate.ts`
  - Block platform-mode regenerate and push the user back toward `My API` instead of silently using hosted capacity.

### Worker auth, credits, and payment boundary

- Create: `SlideTutor-AI/api/lib/platformAccess/types.ts`
  - Shared backend types for balances, ledger entries, analyze attempts, and recharge orders.
- Create: `SlideTutor-AI/api/lib/platformAccess/pricing.ts`
  - Canonical backend pricing table and hosted-task whitelist.
- Create: `SlideTutor-AI/api/lib/platformAccess/store.ts`
  - D1 reads and writes for balances, ledger, analyze attempts, and recharge orders.
- Create: `SlideTutor-AI/api/lib/platformAccess/service.ts`
  - Business rules for starter grant, sufficient-balance checks, analyze attempt lifecycle, successful deductions, and recharge application.
- Create: `SlideTutor-AI/api/lib/platformAccess/paymentAdapter.ts`
  - Provider-agnostic recharge adapter interface plus factory.
- Create: `SlideTutor-AI/api/lib/platformAccess/mockPaymentAdapter.ts`
  - Local and test recharge adapter that simulates checkout and webhook success.
- Create: `SlideTutor-AI/src/worker/lib/auth.ts`
  - Verify Clerk session tokens and surface `userId`.
- Create: `SlideTutor-AI/src/worker/routes/credits-balance.ts`
  - Return starter-backed hosted balance for the signed-in user.
- Create: `SlideTutor-AI/src/worker/routes/recharge-intent.ts`
  - Create a recharge order and return provider checkout data.
- Create: `SlideTutor-AI/src/worker/routes/payment-webhook.ts`
  - Verify recharge completion and apply credits exactly once.
- Modify: `SlideTutor-AI/src/worker/index.ts`
  - Register the new hosted routes and env bindings.
- Modify: `SlideTutor-AI/src/worker/routes/generate.ts`
  - Verify platform auth before hosted generate requests, pass user context into generation service, and return hosted attempt headers.
- Modify: `SlideTutor-AI/src/worker/lib/env.ts`
  - Read new payment and Clerk env flags cleanly.
- Modify: `SlideTutor-AI/api/lib/generateService.ts`
  - Enforce hosted pricing and analyze attempt semantics without changing teaching prompts.

### Database, env, and tests

- Create: `SlideTutor-AI/migrations/002_platform_access_credits.sql`
  - D1 schema for accounts, ledger, analyze attempts, and recharge orders.
- Modify: `SlideTutor-AI/.env.example`
  - Add Clerk, hosted pricing, and payment adapter env vars.
- Modify: `SlideTutor-AI/wrangler.jsonc`
  - Bind the credits D1 database and any payment-related vars needed for Worker execution.
- Create: `SlideTutor-AI/test/workers/credits-balance.worker.test.ts`
  - Cover hosted balance route and starter grant.
- Create: `SlideTutor-AI/test/workers/platform-generate.worker.test.ts`
  - Cover hosted auth, insufficient credits, hosted analyze attempt headers, and successful hosted deductions.
- Create: `SlideTutor-AI/test/workers/recharge.worker.test.ts`
  - Cover recharge intent creation and idempotent webhook credit application.
- Modify: `docs/backend/api-design.md`
  - Document hosted auth headers, credit routes, and hosted analyze behavior.
- Modify: `docs/frontend/architecture.md`
  - Document neutral access modes, hosted UI boundary, and Clerk-backed auth flow.
- Modify: `docs/frontend/data-flow.md`
  - Document hosted analyze attempt lifecycle and hosted deduction timing.
- Modify: `docs/changelog/CHANGELOG_TECH.md`
  - Record the Phase 06 rollout.

## Task 1: Persist access mode and hosted UI state

**Files:**
- Modify: `SlideTutor-AI/src/config/models.ts`
- Modify: `SlideTutor-AI/src/store/uiStore.ts`
- Test: `SlideTutor-AI/src/store/uiStore.test.ts`

- [ ] **Step 1: Write the failing store tests**

```ts
it('defaults to byok access mode and persists hosted mode changes separately from model choice', () => {
  useUiStore.getState().setAccessMode('platform');
  expect(setSetting).toHaveBeenCalledWith('slide_tutor_access_mode', 'platform');
});

it('opens a structured insufficient-credit dialog without mutating access mode', () => {
  useUiStore.getState().openInsufficientCreditsDialog({
    action: 'analyze',
    requiredCredits: 3,
    currentBalance: 1,
  });
  expect(useUiStore.getState().insufficientCreditsDialog?.requiredCredits).toBe(3);
  expect(useUiStore.getState().accessMode).toBe('byok');
});
```

- [ ] **Step 2: Run the store test file and verify it fails**

Run: `cd SlideTutor-AI && npm test -- src/store/uiStore.test.ts`

Expected: FAIL because `accessMode`, hosted balance state, and insufficient-credit dialog helpers do not exist yet.

- [ ] **Step 3: Implement the new persisted UI state**

```ts
type AccessMode = 'byok' | 'platform';

type InsufficientCreditsState = {
  action: 'analyze' | 'followup' | 'quiz.generate' | 'quiz.evaluate';
  requiredCredits: number;
  currentBalance: number;
};
```

Implementation notes:
- Persist `slide_tutor_access_mode` in IndexedDB.
- Keep hosted balance snapshot ephemeral, not persisted.
- Add `openInsufficientCreditsDialog(...)` and `closeInsufficientCreditsDialog()`.
- Do not overload `AiAccessSettings.mode`; keep mode separate from BYOK credentials.

- [ ] **Step 4: Run the store tests again**

Run: `cd SlideTutor-AI && npm test -- src/store/uiStore.test.ts`

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add SlideTutor-AI/src/config/models.ts SlideTutor-AI/src/store/uiStore.ts SlideTutor-AI/src/store/uiStore.test.ts
git commit -m "feat: persist platform access mode state"
```

## Task 2: Add Clerk frontend and backend foundations

**Files:**
- Modify: `SlideTutor-AI/package.json`
- Create: `SlideTutor-AI/src/lib/auth/clerk.tsx`
- Modify: `SlideTutor-AI/src/main.tsx`
- Modify: `SlideTutor-AI/.env.example`

- [ ] **Step 1: Write a failing auth helper test**

Create `SlideTutor-AI/src/lib/auth/clerk.test.tsx`:

```ts
it('throws a clear error when VITE_CLERK_PUBLISHABLE_KEY is missing', () => {
  expect(() => resolveClerkPublishableKey({} as ImportMetaEnv)).toThrow(/VITE_CLERK_PUBLISHABLE_KEY/);
});
```

- [ ] **Step 2: Run the auth helper test and verify it fails**

Run: `cd SlideTutor-AI && npm test -- src/lib/auth/clerk.test.tsx`

Expected: FAIL because the auth helper does not exist yet.

- [ ] **Step 3: Install and wire Clerk**

Run:

```bash
cd SlideTutor-AI
npm install @clerk/clerk-react @clerk/backend
```

Implementation notes:
- Follow the current official Clerk React quickstart and `useAuth().getToken()` / backend token verification pattern from official docs.
- Add `VITE_CLERK_PUBLISHABLE_KEY` to frontend env docs.
- Add one backend verification env:
  - preferred: `CLERK_JWT_KEY`
  - optional fallback: `CLERK_SECRET_KEY`
- Wrap `<App />` with a Clerk provider in `src/main.tsx`.

- [ ] **Step 4: Run auth helper tests**

Run: `cd SlideTutor-AI && npm test -- src/lib/auth/clerk.test.tsx`

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add SlideTutor-AI/package.json SlideTutor-AI/package-lock.json SlideTutor-AI/src/lib/auth/clerk.tsx SlideTutor-AI/src/lib/auth/clerk.test.tsx SlideTutor-AI/src/main.tsx SlideTutor-AI/.env.example
git commit -m "feat: add clerk auth foundation"
```

## Task 3: Build D1-backed hosted credits and analyze-attempt schema

**Files:**
- Create: `SlideTutor-AI/migrations/002_platform_access_credits.sql`
- Create: `SlideTutor-AI/api/lib/platformAccess/types.ts`
- Create: `SlideTutor-AI/api/lib/platformAccess/pricing.ts`
- Create: `SlideTutor-AI/api/lib/platformAccess/service.ts`
- Test: `SlideTutor-AI/api/lib/platformAccess/pricing.test.ts`

- [ ] **Step 1: Write failing pricing tests**

```ts
it('maps approved hosted actions to fixed credit costs', () => {
  expect(getHostedActionCost('analyze')).toBe(3);
  expect(getHostedActionCost('followup')).toBe(1);
});

it('rejects unsupported hosted actions', () => {
  expect(() => getHostedActionCost('regenerate_chunk')).toThrow(/unsupported/i);
});
```

- [ ] **Step 2: Run the pricing tests and verify they fail**

Run: `cd SlideTutor-AI && npm test -- api/lib/platformAccess/pricing.test.ts`

Expected: FAIL because hosted pricing helpers do not exist yet.

- [ ] **Step 3: Create the D1 schema and backend service contracts**

Use this D1 shape:

```sql
CREATE TABLE credit_accounts (
  user_id TEXT PRIMARY KEY,
  balance INTEGER NOT NULL DEFAULT 0,
  starter_granted_at TEXT,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);

CREATE TABLE credit_ledger (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL,
  entry_kind TEXT NOT NULL,
  delta INTEGER NOT NULL,
  balance_after INTEGER NOT NULL,
  idempotency_key TEXT NOT NULL UNIQUE,
  metadata_json TEXT,
  created_at TEXT NOT NULL
);

CREATE TABLE analyze_attempts (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL,
  state TEXT NOT NULL,
  cost INTEGER NOT NULL,
  request_id TEXT NOT NULL,
  model_provider TEXT NOT NULL,
  model_id TEXT NOT NULL,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  expires_at TEXT NOT NULL
);

CREATE TABLE recharge_orders (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL,
  amount_rmb_cent INTEGER NOT NULL,
  credits INTEGER NOT NULL,
  provider TEXT NOT NULL,
  status TEXT NOT NULL,
  provider_order_id TEXT,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);
```

Service rules:
- First balance lookup lazily grants `10` starter credits exactly once.
- Recharge credits never expire.
- `Analyze` uses an attempt record so `explain` and `distill` can share one final charge.
- Ledger writes must be idempotent.

- [ ] **Step 4: Run pricing tests**

Run: `cd SlideTutor-AI && npm test -- api/lib/platformAccess/pricing.test.ts`

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add SlideTutor-AI/migrations/002_platform_access_credits.sql SlideTutor-AI/api/lib/platformAccess/types.ts SlideTutor-AI/api/lib/platformAccess/pricing.ts SlideTutor-AI/api/lib/platformAccess/pricing.test.ts SlideTutor-AI/api/lib/platformAccess/service.ts
git commit -m "feat: add hosted credits schema and pricing rules"
```

## Task 4: Add Worker auth and hosted balance routes

**Files:**
- Create: `SlideTutor-AI/src/worker/lib/auth.ts`
- Create: `SlideTutor-AI/src/worker/routes/credits-balance.ts`
- Modify: `SlideTutor-AI/src/worker/index.ts`
- Modify: `SlideTutor-AI/src/worker/lib/env.ts`
- Test: `SlideTutor-AI/test/workers/credits-balance.worker.test.ts`

- [ ] **Step 1: Write failing worker tests for hosted auth and starter balance**

```ts
it('returns 401 for /api/credits/balance when the bearer token is missing', async () => {
  expect(response.status).toBe(401);
});

it('returns a starter-backed balance for a newly authenticated user', async () => {
  await expect(response.json()).resolves.toMatchObject({ balance: 10 });
});
```

- [ ] **Step 2: Run the worker tests and verify they fail**

Run: `cd SlideTutor-AI && npm run test:workers -- test/workers/credits-balance.worker.test.ts`

Expected: FAIL because the route and auth helper do not exist yet.

- [ ] **Step 3: Implement Clerk verification and hosted balance route**

Implementation notes:
- Accept `Authorization: Bearer <session token>`.
- Verify the token in Worker code and surface `userId`.
- Use `APP_URL` as an allowed audience or authorized party if supported by the Clerk verification helper you choose.
- `GET /api/credits/balance` should lazily create the account and starter ledger entry through the service layer, then return:

```json
{
  "balance": 10,
  "starterCredits": 10,
  "currency": "credits"
}
```

- [ ] **Step 4: Run the worker tests again**

Run: `cd SlideTutor-AI && npm run test:workers -- test/workers/credits-balance.worker.test.ts`

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add SlideTutor-AI/src/worker/lib/auth.ts SlideTutor-AI/src/worker/routes/credits-balance.ts SlideTutor-AI/src/worker/index.ts SlideTutor-AI/src/worker/lib/env.ts SlideTutor-AI/test/workers/credits-balance.worker.test.ts
git commit -m "feat: add hosted auth and balance route"
```

## Task 5: Teach the frontend API client about platform mode

**Files:**
- Modify: `SlideTutor-AI/src/lib/api/apiClient.ts`
- Test: `SlideTutor-AI/src/lib/api/apiClient.test.ts`
- Modify: `SlideTutor-AI/src/config/models.ts`

- [ ] **Step 1: Write failing API client tests**

```ts
it('sends access.mode = platform and a bearer token when platform mode is active', async () => {
  expect(body.access).toEqual({ mode: 'platform' });
  expect(init.headers.Authorization).toMatch(/^Bearer /);
});

it('does not fall back to server secrets when byok mode is selected but the local key is missing', async () => {
  expect(body.access).toBeUndefined();
  expect(() => assertLocalByokReady(...)).toThrow(/configure/i);
});
```

- [ ] **Step 2: Run the API client tests and verify they fail**

Run: `cd SlideTutor-AI && npm test -- src/lib/api/apiClient.test.ts`

Expected: FAIL because platform-mode payloads and Clerk bearer tokens do not exist yet.

- [ ] **Step 3: Implement hosted request payloads and helpers**

Implementation notes:
- Extend the access payload union:

```ts
type GenerateAccessPayload =
  | { mode: 'platform' }
  | { mode: 'byok'; providerId: 'gemini'; apiKey: string }
  | { mode: 'byok'; providerId: 'openai-compatible'; apiKey: string; baseURL: string; endpointPreset?: string };
```

- Add `getPlatformAuthToken()` using Clerk `getToken()`.
- Add helper calls:
  - `getHostedCreditsBalance()`
  - `createRechargeIntent(amountRmb: number)`
- When `accessMode === 'platform'`, always send `access: { mode: 'platform' }`.
- When `accessMode === 'byok'`, never silently rely on server env fallbacks.

- [ ] **Step 4: Run the API client tests**

Run: `cd SlideTutor-AI && npm test -- src/lib/api/apiClient.test.ts`

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add SlideTutor-AI/src/lib/api/apiClient.ts SlideTutor-AI/src/lib/api/apiClient.test.ts SlideTutor-AI/src/config/models.ts
git commit -m "feat: add platform api request payloads"
```

## Task 6: Add neutral mode switching and hosted settings UI

**Files:**
- Create: `SlideTutor-AI/src/lib/platformAccess/pricing.ts`
- Create: `SlideTutor-AI/src/lib/platformAccess/pricing.test.ts`
- Create: `SlideTutor-AI/src/components/settings/PlatformApiSection.tsx`
- Create: `SlideTutor-AI/src/components/settings/BuyCreditsDialog.tsx`
- Create: `SlideTutor-AI/src/components/CreditsRequiredDialog.tsx`
- Modify: `SlideTutor-AI/src/components/SettingsModal.tsx`
- Modify: `SlideTutor-AI/src/components/SettingsModal.test.tsx`
- Modify: `SlideTutor-AI/src/App.tsx`

- [ ] **Step 1: Write failing UI tests**

```tsx
it('shows neutral My API / Platform API choices without a recommended label', () => {
  expect(screen.getByText('My API')).toBeInTheDocument();
  expect(screen.getByText('Platform API')).toBeInTheDocument();
  expect(screen.queryByText(/recommended/i)).not.toBeInTheDocument();
});

it('shows live recharge conversion at 1 RMB = 30 credits', async () => {
  await user.type(screen.getByLabelText(/Amount/i), '2');
  expect(screen.getByText('60 credits')).toBeInTheDocument();
});
```

- [ ] **Step 2: Run the settings tests and verify they fail**

Run: `cd SlideTutor-AI && npm test -- src/components/SettingsModal.test.tsx src/lib/platformAccess/pricing.test.ts`

Expected: FAIL because platform UI and recharge conversion do not exist yet.

- [ ] **Step 3: Implement hosted settings UI**

Implementation notes:
- Split the AI tab so `SettingsModal.tsx` stops growing.
- Render a neutral access-mode switch.
- If an unauthenticated user clicks `Platform API`, call Clerk sign-in immediately and do not persist platform mode until auth is available.
- In platform mode show:
  - current balance
  - `Buy Credits`
  - compact action price summary
- Do not show billing history.
- Mount the global insufficient-credit dialog in `App.tsx`.

- [ ] **Step 4: Run the settings tests**

Run: `cd SlideTutor-AI && npm test -- src/components/SettingsModal.test.tsx src/lib/platformAccess/pricing.test.ts`

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add SlideTutor-AI/src/lib/platformAccess/pricing.ts SlideTutor-AI/src/lib/platformAccess/pricing.test.ts SlideTutor-AI/src/components/settings/PlatformApiSection.tsx SlideTutor-AI/src/components/settings/BuyCreditsDialog.tsx SlideTutor-AI/src/components/CreditsRequiredDialog.tsx SlideTutor-AI/src/components/SettingsModal.tsx SlideTutor-AI/src/components/SettingsModal.test.tsx SlideTutor-AI/src/App.tsx
git commit -m "feat: add platform api settings ui"
```

## Task 7: Implement hosted `Analyze` reservation and finalize semantics

**Files:**
- Modify: `SlideTutor-AI/api/lib/generateService.ts`
- Create: `SlideTutor-AI/api/lib/platformAccess/store.ts`
- Modify: `SlideTutor-AI/api/lib/platformAccess/service.ts`
- Modify: `SlideTutor-AI/src/worker/routes/generate.ts`
- Modify: `SlideTutor-AI/src/hooks/useSlideAnalysis.ts`
- Test: `SlideTutor-AI/test/workers/platform-generate.worker.test.ts`

- [ ] **Step 1: Write failing hosted analyze worker tests**

```ts
it('returns x-slidetutor-analyze-attempt-id after a successful hosted explain preflight', async () => {
  expect(response.headers.get('x-slidetutor-analyze-attempt-id')).toBeTruthy();
});

it('returns a non-charged hosted error when parser access degrades during platform explain', async () => {
  await expect(response.json()).resolves.toMatchObject({ code: 'PLATFORM_ANALYZE_UNAVAILABLE' });
});

it('commits exactly one 3-credit deduction when hosted distill finalizes a matching attempt', async () => {
  expect(balanceAfter).toBe(7);
});
```

- [ ] **Step 2: Run the hosted generate tests and verify they fail**

Run: `cd SlideTutor-AI && npm run test:workers -- test/workers/platform-generate.worker.test.ts`

Expected: FAIL because analyze attempt tracking does not exist yet.

- [ ] **Step 3: Implement hosted analyze lifecycle**

Implementation notes:
- On hosted `task = explain`:
  - verify the user is authenticated
  - require hosted balance `>= 3`
  - reject degraded parser mode with a hosted-specific error before streaming
  - create an `analyze_attempt` row
  - return `x-slidetutor-analyze-attempt-id`
- On hosted `task = distill`:
  - require `hostedAnalyzeAttemptId`
  - verify ownership and state
  - only on successful stream completion:
    - deduct `3`
    - write ledger entry
    - mark attempt `committed`
- In `useSlideAnalysis.ts`, capture the explain header and pass it into the distill request payload:

```ts
taskData: {
  outputLanguage,
  fullExplanation,
  hostedAnalyzeAttemptId,
}
```

- [ ] **Step 4: Run the hosted generate tests**

Run: `cd SlideTutor-AI && npm run test:workers -- test/workers/platform-generate.worker.test.ts`

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add SlideTutor-AI/api/lib/generateService.ts SlideTutor-AI/api/lib/platformAccess/store.ts SlideTutor-AI/api/lib/platformAccess/service.ts SlideTutor-AI/src/worker/routes/generate.ts SlideTutor-AI/src/hooks/useSlideAnalysis.ts SlideTutor-AI/test/workers/platform-generate.worker.test.ts
git commit -m "feat: add hosted analyze billing lifecycle"
```

## Task 8: Bill hosted `Follow-up` and quiz actions, and block unsupported hosted actions

**Files:**
- Modify: `SlideTutor-AI/api/lib/generateService.ts`
- Modify: `SlideTutor-AI/src/hooks/useFollowUp.ts`
- Modify: `SlideTutor-AI/src/hooks/useQuiz.ts`
- Modify: `SlideTutor-AI/src/hooks/useChunkRegenerate.ts`
- Test: `SlideTutor-AI/test/workers/platform-generate.worker.test.ts`

- [ ] **Step 1: Extend the failing worker tests**

```ts
it('deducts 1 credit after a successful hosted followup stream completes', async () => {
  expect(balanceAfter).toBe(9);
});

it('blocks regenerate_chunk in platform mode with a clear unsupported code', async () => {
  await expect(response.json()).resolves.toMatchObject({ code: 'UNSUPPORTED_PLATFORM_ACTION' });
});

it('returns INSUFFICIENT_CREDITS before hosted followup execution when balance is too low', async () => {
  await expect(response.json()).resolves.toMatchObject({ code: 'INSUFFICIENT_CREDITS' });
});
```

- [ ] **Step 2: Run the worker tests and verify they fail**

Run: `cd SlideTutor-AI && npm run test:workers -- test/workers/platform-generate.worker.test.ts`

Expected: FAIL because hosted follow-up and quiz deductions do not exist yet.

- [ ] **Step 3: Implement hosted action billing and frontend guardrails**

Implementation notes:
- Bill `1` credit on stream completion for:
  - `followup`
  - `generate_questions`
  - `evaluate_answers`
- Reject unsupported hosted actions with:

```json
{
  "error": "This action is only available in My API for now.",
  "code": "UNSUPPORTED_PLATFORM_ACTION"
}
```

- In `useFollowUp.ts`, `useQuiz.ts`, and `useChunkRegenerate.ts`, convert hosted insufficient-credit and unsupported-action codes into the global dialog or clear mode-switch guidance instead of raw `alert(...)`.

- [ ] **Step 4: Run the worker tests and any touched hook tests**

Run:

```bash
cd SlideTutor-AI
npm run test:workers -- test/workers/platform-generate.worker.test.ts
npm test -- src/hooks/useSlideAnalysis.test.ts src/lib/api/apiClient.test.ts
```

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add SlideTutor-AI/api/lib/generateService.ts SlideTutor-AI/src/hooks/useFollowUp.ts SlideTutor-AI/src/hooks/useQuiz.ts SlideTutor-AI/src/hooks/useChunkRegenerate.ts SlideTutor-AI/test/workers/platform-generate.worker.test.ts
git commit -m "feat: bill hosted followup and quiz actions"
```

## Task 9: Add recharge intent and mock payment adapter boundary

**Files:**
- Create: `SlideTutor-AI/api/lib/platformAccess/paymentAdapter.ts`
- Create: `SlideTutor-AI/api/lib/platformAccess/mockPaymentAdapter.ts`
- Create: `SlideTutor-AI/src/worker/routes/recharge-intent.ts`
- Create: `SlideTutor-AI/src/worker/routes/payment-webhook.ts`
- Test: `SlideTutor-AI/test/workers/recharge.worker.test.ts`
- Modify: `SlideTutor-AI/.env.example`
- Modify: `SlideTutor-AI/wrangler.jsonc`

- [ ] **Step 1: Write failing recharge worker tests**

```ts
it('creates a recharge order from an RMB amount and returns converted credits', async () => {
  await expect(response.json()).resolves.toMatchObject({
    amountRmb: 1,
    credits: 30,
  });
});

it('applies credits exactly once when the webhook is replayed', async () => {
  expect(balanceAfterSecondWebhook).toBe(balanceAfterFirstWebhook);
});
```

- [ ] **Step 2: Run the recharge worker tests and verify they fail**

Run: `cd SlideTutor-AI && npm run test:workers -- test/workers/recharge.worker.test.ts`

Expected: FAIL because recharge routes and adapters do not exist yet.

- [ ] **Step 3: Implement the provider-agnostic recharge contract**

Use this route contract:

```json
POST /api/recharge-intent
{
  "amountRmb": 1
}
```

Returns:

```json
{
  "orderId": "ord_123",
  "amountRmb": 1,
  "credits": 30,
  "provider": "mock",
  "checkoutUrl": "https://..."
}
```

Implementation notes:
- Keep conversion server-side as the source of truth even though the UI previews locally.
- `mock` adapter should simulate success and let Worker tests run end-to-end.
- Add env flags such as:
  - `PAYMENT_PROVIDER=mock`
  - `PAYMENT_WEBHOOK_SECRET`
- If a real provider has been selected and documented by execution time, replace `mock` in this task only. Do not leak provider details into unrelated tasks.

- [ ] **Step 4: Run the recharge worker tests**

Run: `cd SlideTutor-AI && npm run test:workers -- test/workers/recharge.worker.test.ts`

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add SlideTutor-AI/api/lib/platformAccess/paymentAdapter.ts SlideTutor-AI/api/lib/platformAccess/mockPaymentAdapter.ts SlideTutor-AI/src/worker/routes/recharge-intent.ts SlideTutor-AI/src/worker/routes/payment-webhook.ts SlideTutor-AI/test/workers/recharge.worker.test.ts SlideTutor-AI/.env.example SlideTutor-AI/wrangler.jsonc
git commit -m "feat: add hosted recharge adapter boundary"
```

## Task 10: Update docs and run full verification

**Files:**
- Modify: `docs/backend/api-design.md`
- Modify: `docs/frontend/architecture.md`
- Modify: `docs/frontend/data-flow.md`
- Modify: `docs/changelog/CHANGELOG_TECH.md`

- [ ] **Step 1: Update docs to match the shipped behavior**

Document:
- neutral access modes
- Clerk login-at-switch behavior
- hosted balance route
- recharge intent route
- hosted analyze attempt lifecycle
- unsupported hosted secondary actions

- [ ] **Step 2: Run targeted frontend and worker verification**

Run:

```bash
cd SlideTutor-AI
npm test -- src/store/uiStore.test.ts src/lib/api/apiClient.test.ts src/components/SettingsModal.test.tsx src/lib/platformAccess/pricing.test.ts
npm run test:workers -- test/workers/credits-balance.worker.test.ts test/workers/platform-generate.worker.test.ts test/workers/recharge.worker.test.ts
```

Expected: PASS

- [ ] **Step 3: Run broader regression verification**

Run:

```bash
cd SlideTutor-AI
npm test
npm run test:workers
npm run lint
```

Expected:
- all unit tests pass
- all worker tests pass
- TypeScript no-emit check passes

- [ ] **Step 4: Commit docs and verification-safe cleanup**

```bash
git add docs/backend/api-design.md docs/frontend/architecture.md docs/frontend/data-flow.md docs/changelog/CHANGELOG_TECH.md
git commit -m "docs: record platform api and credits rollout"
```

## Review Checklist

- `My API` never consumes hosted credits.
- `Platform API` requires login at mode switch.
- Hosted `Analyze` charges exactly once after `parse + explain + distill` all succeed.
- Hosted degraded parse results never become paid success.
- Hosted follow-up and quiz actions charge only on success.
- Insufficient credits never auto-switch the user out of `Platform API`.
- Recharge history and deduction history stay out of the UI.
- Backend ledger remains durable and auditable.

## Execution Notes

- Use the existing Worker runtime and D1 patterns from parser usage work as the implementation reference.
- Keep new UI logic out of `SettingsModal.tsx` as much as possible by extracting dedicated subcomponents.
- Do not reintroduce BYOK fallback to platform env secrets. Hosted access must be explicit.
- Pause before Task 9 if a real payment provider still has no docs or credentials. The rest of the plan remains executable without reopening product design.
