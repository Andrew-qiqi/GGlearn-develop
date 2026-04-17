# Platform Gemini Custom Endpoint Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let `Platform API` route Gemini through either the official endpoint or a platform-managed custom Gemini-compatible base URL, while keeping `My API` Gemini routing unchanged.

**Architecture:** Keep Gemini as the existing provider family and make the hosted routing decision inside backend provider-access resolution only. `My API` continues to use browser-local Gemini BYOK state, while `Platform API` resolves official vs custom Gemini strictly from Worker environment variables. Existing runtime Gemini client creation already supports optional `baseURL`, so the main product change lives in env parsing, validation, and documentation.

**Tech Stack:** TypeScript, Vitest, Cloudflare Worker runtime env vars, `@google/genai`, existing hosted `Platform API` generation flow

---

## Scope Locks

- This plan implements the approved design in `docs/superpowers/specs/2026-04-14-platform-gemini-custom-endpoint-design.md`.
- `My API` Gemini behavior must remain unchanged.
- `Platform API` must not read browser-local Gemini BYOK settings.
- Shared `selectedModel` persistence remains unchanged; do not split `modelId` by access mode.
- No new frontend controls are added for hosted Gemini routing.
- `GEMINI_API_KEY` stays in place for official hosted Gemini and existing internal Gemini consumers.

## File Map

### Backend provider-access resolution

- Modify: `GGlearn-AI/api/lib/env.ts`
  - branch Gemini provider resolution by access mode
  - add platform-only custom Gemini env parsing
  - validate `PLATFORM_GEMINI_BASE_URL` as an absolute URL
  - require `PLATFORM_GEMINI_API_KEY` only when hosted custom Gemini is enabled
- Modify: `GGlearn-AI/api/lib/env.test.ts`
  - cover hosted official/custom Gemini env resolution and misconfiguration failures

### Gemini runtime regression coverage

- Modify: `GGlearn-AI/api/lib/generateService.gemini.test.ts`
  - lock that hosted Gemini still constructs `GoogleGenAI` with or without `httpOptions.baseUrl` based on resolved hosted access

### Env examples and docs

- Modify: `GGlearn-AI/.env.example`
  - add `PLATFORM_GEMINI_BASE_URL`
  - add `PLATFORM_GEMINI_API_KEY`
- Modify: `docs/architecture/deployment.md`
  - document hosted Gemini env rules
- Modify: `docs/user_guide/access-modes.md`
  - clarify that hosted Gemini routing is platform-managed and independent from `My API`
- Modify: `docs/backend/platform-model-configuration.md`
  - document the hosted Gemini route-switch case
- Modify: `docs/changelog/CHANGELOG_TECH.md`
  - record the rollout

## Task 1: Resolve Hosted Gemini Route From Worker Env

**Files:**
- Modify: `GGlearn-AI/api/lib/env.ts`
- Modify: `GGlearn-AI/api/lib/env.test.ts`

- [ ] **Step 1: Write the failing env tests**

Add tests in `GGlearn-AI/api/lib/env.test.ts` for hosted Gemini:

```ts
it('uses GEMINI_API_KEY for platform gemini when PLATFORM_GEMINI_BASE_URL is empty', () => {
  expect(
    resolveProviderAccess(
      {
        providerId: 'gemini',
        access: { mode: 'platform' },
      },
      {
        GEMINI_API_KEY: 'official-server-key',
        PLATFORM_GEMINI_BASE_URL: '',
      },
    ),
  ).toEqual({
    providerId: 'gemini',
    apiKey: 'official-server-key',
  });
});

it('uses PLATFORM_GEMINI_API_KEY and baseURL for platform gemini when PLATFORM_GEMINI_BASE_URL is set', () => {
  expect(
    resolveProviderAccess(
      {
        providerId: 'gemini',
        access: { mode: 'platform' },
      },
      {
        GEMINI_API_KEY: 'official-server-key',
        PLATFORM_GEMINI_API_KEY: 'platform-relay-key',
        PLATFORM_GEMINI_BASE_URL: 'https://right.codes/gemini',
      },
    ),
  ).toEqual({
    providerId: 'gemini',
    apiKey: 'platform-relay-key',
    baseURL: 'https://right.codes/gemini',
  });
});

it('rejects platform custom gemini when PLATFORM_GEMINI_API_KEY is missing', () => {
  expect(() =>
    resolveProviderAccess(
      {
        providerId: 'gemini',
        access: { mode: 'platform' },
      },
      {
        GEMINI_API_KEY: 'official-server-key',
        PLATFORM_GEMINI_BASE_URL: 'https://right.codes/gemini',
      },
    ),
  ).toThrow(/PLATFORM_GEMINI_API_KEY/i);
});

it('rejects platform custom gemini when PLATFORM_GEMINI_BASE_URL is not a valid absolute URL', () => {
  expect(() =>
    resolveProviderAccess(
      {
        providerId: 'gemini',
        access: { mode: 'platform' },
      },
      {
        GEMINI_API_KEY: 'official-server-key',
        PLATFORM_GEMINI_API_KEY: 'platform-relay-key',
        PLATFORM_GEMINI_BASE_URL: '/relative/path',
      },
    ),
  ).toThrow(/PLATFORM_GEMINI_BASE_URL/i);
});
```

- [ ] **Step 2: Run the targeted env tests and verify they fail**

Run:

```bash
cd GGlearn-AI && npm test -- api/lib/env.test.ts
```

Expected:

- FAIL because the current platform Gemini branch always resolves through `GEMINI_API_KEY` and does not understand `PLATFORM_GEMINI_BASE_URL` or `PLATFORM_GEMINI_API_KEY`

- [ ] **Step 3: Implement hosted Gemini env resolution**

Implementation notes for `GGlearn-AI/api/lib/env.ts`:

- keep the current BYOK Gemini branch unchanged
- inside the `input.access?.mode === 'platform' && input.providerId === 'gemini'` path:
  - read `PLATFORM_GEMINI_BASE_URL`
  - if empty, return official hosted Gemini with `requireProviderApiKey('gemini', env)`
  - if non-empty:
    - validate it with `new URL(...)`
    - require `PLATFORM_GEMINI_API_KEY`
    - return `{ providerId: 'gemini', apiKey, baseURL }`
- keep the existing OpenAI-compatible platform restrictions unchanged
- add a focused helper rather than spreading platform-Gemini env parsing across multiple branches

Recommended helper shape:

```ts
function resolvePlatformGeminiAccess(env: EnvBag) {
  const platformBaseURL = readEnvSecret(env, 'PLATFORM_GEMINI_BASE_URL');

  if (!platformBaseURL) {
    return {
      providerId: 'gemini' as const,
      apiKey: requireProviderApiKey('gemini', env),
    };
  }

  assertValidAbsoluteUrl(platformBaseURL, 'PLATFORM_GEMINI_BASE_URL');

  return {
    providerId: 'gemini' as const,
    apiKey: requireSecret(env, 'PLATFORM_GEMINI_API_KEY'),
    baseURL: platformBaseURL,
  };
}
```

- [ ] **Step 4: Run the env tests again and verify they pass**

Run:

```bash
cd GGlearn-AI && npm test -- api/lib/env.test.ts
```

Expected:

- PASS with hosted official/custom Gemini env rules covered

- [ ] **Step 5: Commit**

```bash
git add GGlearn-AI/api/lib/env.ts GGlearn-AI/api/lib/env.test.ts
git commit -m "feat: resolve platform gemini custom endpoints"
```

## Task 2: Lock Hosted Gemini Runtime Construction

**Files:**
- Modify: `GGlearn-AI/api/lib/generateService.gemini.test.ts`

- [ ] **Step 1: Add hosted Gemini constructor regression tests**

Extend `GGlearn-AI/api/lib/generateService.gemini.test.ts` with platform-mode cases:

```ts
it('constructs GoogleGenAI without baseURL for official platform gemini requests', async () => {
  resolveProviderAccessMock.mockReturnValue({
    providerId: 'gemini',
    apiKey: 'official-server-key',
  });

  await createGenerationStream(
    {
      providerId: 'gemini',
      modelId: 'gemini-3-flash-preview',
      task: 'followup',
      taskData: { message: 'Explain this.' },
      access: { mode: 'platform' },
    },
    {
      env: {},
      requestId: 'req_platform_official',
      clientIp: '127.0.0.1',
      platformUser: { userId: 'user_123' },
    },
  );

  expect(googleGenAiCtorMock).toHaveBeenCalledWith({
    apiKey: 'official-server-key',
  });
});

it('constructs GoogleGenAI with baseURL for custom platform gemini requests', async () => {
  resolveProviderAccessMock.mockReturnValue({
    providerId: 'gemini',
    apiKey: 'platform-relay-key',
    baseURL: 'https://right.codes/gemini',
  });

  await createGenerationStream(
    {
      providerId: 'gemini',
      modelId: 'gemini-3-flash-preview',
      task: 'followup',
      taskData: { message: 'Explain this.' },
      access: { mode: 'platform' },
    },
    {
      env: {},
      requestId: 'req_platform_custom',
      clientIp: '127.0.0.1',
      platformUser: { userId: 'user_123' },
    },
  );

  expect(googleGenAiCtorMock).toHaveBeenCalledWith({
    apiKey: 'platform-relay-key',
    httpOptions: {
      baseUrl: 'https://right.codes/gemini',
    },
  });
});
```

- [ ] **Step 2: Run the targeted backend tests**

Run:

```bash
cd GGlearn-AI && npm test -- api/lib/env.test.ts api/lib/generateService.gemini.test.ts
```

Expected:

- FAIL before Task 1 is implemented because hosted env resolution tests are still red

- [ ] **Step 3: Keep runtime code minimal and only adjust if a regression appears**

Implementation note:

- existing `createGeminiClient({ apiKey, baseURL })` in `GGlearn-AI/api/lib/generateService.ts` already supports both official and custom Gemini construction
- this task is primarily regression coverage for hosted mode, not a planned runtime refactor
- if the tests reveal any mode-specific regression, keep the production change minimal and local to the Gemini client construction path

- [ ] **Step 4: Re-run the targeted backend tests**

Run:

```bash
cd GGlearn-AI && npm test -- api/lib/env.test.ts api/lib/generateService.gemini.test.ts
```

Expected:

- PASS with hosted official/custom Gemini constructor behavior covered

- [ ] **Step 5: Commit**

```bash
git add GGlearn-AI/api/lib/generateService.gemini.test.ts
git commit -m "test: cover platform gemini runtime routing"
```

## Task 3: Document Hosted Gemini Routing And Env Setup

**Files:**
- Modify: `GGlearn-AI/.env.example`
- Modify: `docs/architecture/deployment.md`
- Modify: `docs/user_guide/access-modes.md`
- Modify: `docs/backend/platform-model-configuration.md`
- Modify: `docs/changelog/CHANGELOG_TECH.md`

- [ ] **Step 1: Update the env example**

Add platform Gemini vars near the existing AI provider secrets in `GGlearn-AI/.env.example`:

```env
GEMINI_API_KEY=""
PLATFORM_GEMINI_BASE_URL=""
PLATFORM_GEMINI_API_KEY=""
DOUBAO_API_KEY=""
QWEN_API_KEY=""
```

Keep comments explicit that:

- empty `PLATFORM_GEMINI_BASE_URL` means official hosted Gemini
- non-empty `PLATFORM_GEMINI_BASE_URL` means hosted custom Gemini and requires `PLATFORM_GEMINI_API_KEY`

- [ ] **Step 2: Update deployment and backend docs**

Document the hosted Gemini rule set in:

- `docs/architecture/deployment.md`
- `docs/backend/platform-model-configuration.md`

Required points:

- `Platform API` Gemini route is resolved from Worker env, not browser settings
- official hosted Gemini uses `GEMINI_API_KEY`
- custom hosted Gemini uses `PLATFORM_GEMINI_API_KEY + PLATFORM_GEMINI_BASE_URL`
- `PLATFORM_GEMINI_BASE_URL` must be a valid absolute URL

- [ ] **Step 3: Update user-facing access-mode docs**

Adjust `docs/user_guide/access-modes.md` to make this distinction explicit:

- `My API` Gemini official/custom is user-managed
- `Platform API` Gemini official/custom is platform-managed
- the two routing systems are independent even though they still share the selected `modelId`

- [ ] **Step 4: Record the rollout in the changelog**

Add a top entry in `docs/changelog/CHANGELOG_TECH.md` describing:

- what changed
- why the hosted route split was added
- impact on platform operators and BYOK isolation
- exact files changed

- [ ] **Step 5: Run final targeted verification**

Run:

```bash
cd GGlearn-AI && npm test -- api/lib/env.test.ts api/lib/generateService.gemini.test.ts
git diff -- GGlearn-AI/.env.example docs/architecture/deployment.md docs/user_guide/access-modes.md docs/backend/platform-model-configuration.md docs/changelog/CHANGELOG_TECH.md
```

Expected:

- tests PASS
- diff shows only hosted Gemini env and documentation updates

- [ ] **Step 6: Commit**

```bash
git add GGlearn-AI/.env.example docs/architecture/deployment.md docs/user_guide/access-modes.md docs/backend/platform-model-configuration.md docs/changelog/CHANGELOG_TECH.md
git commit -m "docs: record platform gemini endpoint configuration"
```

## Final Verification Checklist

- `My API` Gemini BYOK tests still pass unchanged
- `Platform API` official Gemini still resolves through `GEMINI_API_KEY`
- `Platform API` custom Gemini resolves through `PLATFORM_GEMINI_API_KEY + PLATFORM_GEMINI_BASE_URL`
- hosted custom Gemini rejects missing `PLATFORM_GEMINI_API_KEY`
- hosted custom Gemini rejects invalid `PLATFORM_GEMINI_BASE_URL`
- no frontend model-state migration is introduced
