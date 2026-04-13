# Gemini Custom Endpoint Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `Google Official / Custom` Gemini endpoint modes to `My API`, require `Gemini Base URL` only for `Custom`, and make runtime plus capability checks honor the active Gemini endpoint without changing `Platform API`.

**Architecture:** Keep Gemini as the existing provider family and extend only the Gemini BYOK access object with endpoint-mode state. Frontend model selection remains unchanged, while frontend request assembly, backend provider resolution, Gemini runtime execution, and Gemini capability probing all flow through one endpoint-aware configuration path. Extend the capability snapshot so endpoint-mode and custom URL changes invalidate stale readiness instead of reusing results from a previous Gemini route.

**Tech Stack:** TypeScript, React 19, Zustand, Vitest, Cloudflare Worker, `@google/genai`, existing BYOK capability-check and settings infrastructure

---

## Scope Locks

- This plan implements the approved design in `docs/superpowers/specs/2026-04-14-gemini-custom-endpoint-design.md`.
- `Platform API` remains unchanged and must not expose Gemini endpoint-mode settings.
- Gemini model choice remains the existing curated dropdown; there is no freeform Gemini `modelId`.
- User-facing copy stays generic and says `Gemini Base URL`, not `Right Codes`.
- `Custom` Gemini must not reuse compatibility results generated for `Google Official` or a different custom URL.

## File Map

### Frontend config and persisted access state

- Modify: `SlideTutor-AI/src/config/models.ts`
  - add Gemini endpoint-mode types, defaults, normalization, and capability-snapshot helpers
- Create: `SlideTutor-AI/src/config/models.test.ts`
  - lock migration defaults and Gemini endpoint helper behavior
- Modify: `SlideTutor-AI/src/store/uiStore.ts`
  - persist Gemini endpoint fields and invalidate capability state when Gemini route changes
- Modify: `SlideTutor-AI/src/store/uiStore.test.ts`
  - cover legacy migration and stale capability invalidation for Gemini route changes

### Frontend settings UI and copy

- Modify: `SlideTutor-AI/src/components/SettingsModal.tsx`
  - render Gemini endpoint-mode selector and conditional `Gemini Base URL` field
- Modify: `SlideTutor-AI/src/components/SettingsModal.test.tsx`
  - cover `Google Official / Custom` UI behavior and compatibility readiness requirements
- Modify: `SlideTutor-AI/src/lib/i18n/index.ts`
  - add labels for Gemini endpoint mode and `Gemini Base URL`

### Frontend request assembly and capability snapshots

- Modify: `SlideTutor-AI/src/lib/api/apiClient.ts`
  - send Gemini custom `baseURL` in BYOK generate and capability-check payloads
  - require `Gemini Base URL` only for `Custom`
  - store capability results against an endpoint-aware selection snapshot
- Modify: `SlideTutor-AI/src/lib/api/apiClient.test.ts`
  - lock official/custom Gemini payload shape and snapshot behavior

### Backend access resolution

- Modify: `SlideTutor-AI/api/lib/env.ts`
  - extend Gemini BYOK access payload with optional `baseURL`
  - require BYOK Gemini API key in `mode = 'byok'`
  - return optional Gemini `baseURL` to runtime callers
- Create: `SlideTutor-AI/api/lib/env.test.ts`
  - cover official/custom Gemini resolution and platform isolation

### Gemini runtime and capability probe

- Modify: `SlideTutor-AI/api/lib/generateService.ts`
  - instantiate `GoogleGenAI` with `baseURL` only for custom Gemini runtime calls
- Create: `SlideTutor-AI/api/lib/generateService.gemini.test.ts`
  - verify official/custom Gemini constructor arguments
- Modify: `SlideTutor-AI/api/lib/modelCapabilityProbe.ts`
  - run provider probes against the active Gemini endpoint when Gemini uses a custom base URL
- Modify: `SlideTutor-AI/api/lib/modelCapabilityProbe.test.ts`
  - verify Gemini custom probes use `baseURL` and do not short-circuit to registry-only ready state

### Docs

- Modify: `docs/backend/byok-capability-check.md`
  - document Gemini custom endpoint behavior in the capability-check contract
- Modify: `docs/user_guide/access-modes.md`
  - document that `My API > Gemini` supports official and custom endpoint modes
- Modify: `docs/changelog/CHANGELOG_TECH.md`
  - record the rollout

## Task 1: Add Gemini endpoint-mode types, defaults, and selection snapshots

**Files:**
- Modify: `SlideTutor-AI/src/config/models.ts`
- Create: `SlideTutor-AI/src/config/models.test.ts`
- Modify: `SlideTutor-AI/src/store/uiStore.ts`
- Test: `SlideTutor-AI/src/store/uiStore.test.ts`

- [ ] **Step 1: Write the failing config and store tests**

```ts
it('migrates legacy gemini access settings to google-official with an empty base URL', () => {
  expect(
    normalizeAiAccessSettings({
      gemini: {
        apiKey: 'legacy-gemini-key',
      },
    }),
  ).toMatchObject({
    gemini: {
      apiKey: 'legacy-gemini-key',
      endpointPreset: 'google-official',
      baseURL: '',
    },
  });
});

it('creates a distinct capability selection snapshot for custom gemini routes', () => {
  expect(
    buildModelCapabilitySelection(
      { providerId: 'gemini', modelId: 'gemini-2.5-flash' },
      {
        gemini: {
          apiKey: 'gemini-key',
          endpointPreset: 'custom',
          baseURL: 'https://right.codes/gemini',
        },
      } as AiAccessSettings,
    ),
  ).toMatchObject({
    providerId: 'gemini',
    modelId: 'gemini-2.5-flash',
    configKey: 'gemini|custom|https://right.codes/gemini|gemini-2.5-flash',
  });
});
```

Add a store regression test in `SlideTutor-AI/src/store/uiStore.test.ts`:

```ts
it('marks capability metadata stale when the saved gemini endpoint mode changes', () => {
  useUiStore.setState({
    selectedModel: { providerId: 'gemini', modelId: 'gemini-2.5-flash' },
    modelCapabilityCheck: {
      status: 'usable',
      checkedAt: '2026-04-14T12:00:00.000Z',
      lastErrorCode: null,
      capabilitySummary: {
        structuredOutput: true,
        streaming: true,
        vision: true,
        thinking: false,
      },
      selection: {
        providerId: 'gemini',
        modelId: 'gemini-2.5-flash',
        configKey: 'gemini|google-official||gemini-2.5-flash',
      },
    },
  } as any);

  useUiStore.getState().setAiAccess({
    ...useUiStore.getState().aiAccess,
    gemini: {
      apiKey: 'gemini-key',
      endpointPreset: 'custom',
      baseURL: 'https://right.codes/gemini',
    },
  });

  expect(useUiStore.getState().modelCapabilityCheck.status).toBe('stale');
});
```

- [ ] **Step 2: Run the targeted tests and verify they fail**

Run:

```bash
cd SlideTutor-AI && npm test -- src/config/models.test.ts src/store/uiStore.test.ts
```

Expected:

- FAIL because Gemini endpoint-mode types, defaults, and `configKey` snapshots do not exist yet

- [ ] **Step 3: Implement the config and store foundation**

Implementation notes:

- add `type GeminiEndpointPreset = 'google-official' | 'custom'`
- extend `AiAccessSettings.gemini` to:

```ts
gemini: {
  apiKey: string;
  endpointPreset: GeminiEndpointPreset;
  baseURL: string;
}
```

- add normalization helpers:

```ts
export function resolveGeminiBaseURL(aiAccess: AiAccessSettings) {
  return aiAccess.gemini.endpointPreset === 'custom' ? aiAccess.gemini.baseURL.trim() : '';
}
```

- extend `ModelCapabilitySelection` with `configKey: string`
- generate `configKey` from the effective runtime route, not only the catalog model id
- use the same builder in both `uiStore.ts` and `apiClient.ts`

- [ ] **Step 4: Run the tests again and verify they pass**

Run:

```bash
cd SlideTutor-AI && npm test -- src/config/models.test.ts src/store/uiStore.test.ts
```

Expected:

- PASS with Gemini legacy migration and endpoint-aware capability invalidation covered

- [ ] **Step 5: Commit**

```bash
git add SlideTutor-AI/src/config/models.ts SlideTutor-AI/src/config/models.test.ts SlideTutor-AI/src/store/uiStore.ts SlideTutor-AI/src/store/uiStore.test.ts
git commit -m "feat: add gemini endpoint mode state"
```

## Task 2: Add Gemini endpoint-mode controls to Settings

**Files:**
- Modify: `SlideTutor-AI/src/components/SettingsModal.tsx`
- Modify: `SlideTutor-AI/src/components/SettingsModal.test.tsx`
- Modify: `SlideTutor-AI/src/lib/i18n/index.ts`

- [ ] **Step 1: Write the failing settings tests**

Add tests in `SlideTutor-AI/src/components/SettingsModal.test.tsx` for:

```ts
it('shows Google Official and Custom endpoint modes for Gemini BYOK', () => {
  renderGeminiSettings();
  expect(screen.getByLabelText(/Gemini Endpoint/i)).toBeInTheDocument();
  expect(screen.getByRole('option', { name: /Google Official/i })).toBeInTheDocument();
  expect(screen.getByRole('option', { name: /Custom/i })).toBeInTheDocument();
});

it('reveals Gemini Base URL only when Gemini endpoint mode is custom', async () => {
  renderGeminiSettings();
  expect(screen.queryByLabelText(/Gemini Base URL/i)).toBeNull();

  fireEvent.change(screen.getByLabelText(/Gemini Endpoint/i), {
    target: { value: 'custom' },
  });

  expect(await screen.findByLabelText(/Gemini Base URL/i)).toBeInTheDocument();
});

it('does not start Gemini compatibility checks until Gemini Base URL is filled for custom mode', async () => {
  renderGeminiSettings();

  fireEvent.change(screen.getByLabelText(/Gemini API Key/i), {
    target: { value: 'gemini-user-key' },
  });
  fireEvent.change(screen.getByLabelText(/Gemini Endpoint/i), {
    target: { value: 'custom' },
  });

  expect(screen.queryByText(/Checking model compatibility/i)).toBeNull();
});
```

- [ ] **Step 2: Run the settings test file and verify it fails**

Run:

```bash
cd SlideTutor-AI && npm test -- src/components/SettingsModal.test.tsx
```

Expected:

- FAIL because the Gemini endpoint selector and custom base URL field do not exist yet

- [ ] **Step 3: Implement the minimal UI**

Implementation notes:

- add i18n keys for:
  - `geminiEndpoint`
  - `geminiOfficialEndpoint`
  - `geminiCustomEndpoint`
  - `geminiBaseUrl`
- render the Gemini endpoint selector only when:
  - `accessMode === 'byok'`
  - `selectedModel.providerId === 'gemini'`
- hide `Gemini Base URL` unless endpoint mode is `custom`
- update `byokConfigReady` so Gemini custom requires both:
  - `apiKey`
  - `baseURL`

- [ ] **Step 4: Run the settings tests again**

Run:

```bash
cd SlideTutor-AI && npm test -- src/components/SettingsModal.test.tsx
```

Expected:

- PASS with Gemini official/custom settings behavior covered

- [ ] **Step 5: Commit**

```bash
git add SlideTutor-AI/src/components/SettingsModal.tsx SlideTutor-AI/src/components/SettingsModal.test.tsx SlideTutor-AI/src/lib/i18n/index.ts
git commit -m "feat: add gemini custom endpoint settings"
```

## Task 3: Send endpoint-aware Gemini BYOK payloads from the client

**Files:**
- Modify: `SlideTutor-AI/src/lib/api/apiClient.ts`
- Modify: `SlideTutor-AI/src/lib/api/apiClient.test.ts`

- [ ] **Step 1: Add failing client tests**

Add tests in `SlideTutor-AI/src/lib/api/apiClient.test.ts` for:

```ts
it('sends only apiKey for google-official gemini BYOK requests', async () => {
  useUiStore.setState({
    aiAccess: {
      ...useUiStore.getState().aiAccess,
      gemini: {
        apiKey: 'gemini-user-key',
        endpointPreset: 'google-official',
        baseURL: '',
      },
    },
  } as any);

  await apiGenerate({ providerId: 'gemini', modelId: 'gemini-2.5-flash', task: 'followup' });

  expect(readGenerateBody().access).toEqual({
    mode: 'byok',
    providerId: 'gemini',
    apiKey: 'gemini-user-key',
  });
});

it('sends apiKey and baseURL for custom gemini BYOK capability checks', async () => {
  useUiStore.setState({
    aiAccess: {
      ...useUiStore.getState().aiAccess,
      gemini: {
        apiKey: 'gemini-user-key',
        endpointPreset: 'custom',
        baseURL: 'https://right.codes/gemini',
      },
    },
  } as any);

  await checkModelCapability({
    providerId: 'gemini',
    modelId: 'gemini-2.5-flash',
  });

  expect(readCapabilityBody().access).toEqual({
    mode: 'byok',
    providerId: 'gemini',
    apiKey: 'gemini-user-key',
    baseURL: 'https://right.codes/gemini',
  });
});

it('throws a Gemini Base URL configuration error when custom mode is incomplete', async () => {
  useUiStore.setState({
    aiAccess: {
      ...useUiStore.getState().aiAccess,
      gemini: {
        apiKey: 'gemini-user-key',
        endpointPreset: 'custom',
        baseURL: '',
      },
    },
  } as any);

  await expect(
    apiGenerate({ providerId: 'gemini', modelId: 'gemini-2.5-flash', task: 'followup' }),
  ).rejects.toThrow(/Gemini Base URL/i);
});
```

- [ ] **Step 2: Run the client tests and verify they fail**

Run:

```bash
cd SlideTutor-AI && npm test -- src/lib/api/apiClient.test.ts
```

Expected:

- FAIL because Gemini BYOK requests do not currently understand endpoint mode or base URL

- [ ] **Step 3: Implement endpoint-aware request assembly**

Implementation notes:

- add a small helper inside `apiClient.ts`:

```ts
function resolveEffectiveGeminiAccess() {
  const { aiAccess } = useUiStore.getState();
  return {
    apiKey: aiAccess.gemini.apiKey.trim(),
    endpointPreset: aiAccess.gemini.endpointPreset,
    baseURL: aiAccess.gemini.endpointPreset === 'custom'
      ? aiAccess.gemini.baseURL.trim()
      : '',
  };
}
```

- update `buildByokConfigurationError('gemini')` so custom mode can say `Gemini Base URL`
- send `baseURL` only when Gemini endpoint mode is `custom`
- when storing capability results, always attach the request-time `configKey` snapshot rather than reading current state after the request finishes

- [ ] **Step 4: Run the client tests again**

Run:

```bash
cd SlideTutor-AI && npm test -- src/lib/api/apiClient.test.ts
```

Expected:

- PASS with official/custom Gemini payload shape and request-time snapshot behavior covered

- [ ] **Step 5: Commit**

```bash
git add SlideTutor-AI/src/lib/api/apiClient.ts SlideTutor-AI/src/lib/api/apiClient.test.ts
git commit -m "feat: send gemini custom endpoint payloads"
```

## Task 4: Resolve Gemini custom access on the backend

**Files:**
- Modify: `SlideTutor-AI/api/lib/env.ts`
- Create: `SlideTutor-AI/api/lib/env.test.ts`

- [ ] **Step 1: Write the failing env tests**

Create `SlideTutor-AI/api/lib/env.test.ts` with:

```ts
it('resolves google-official gemini byok access without a base URL', () => {
  expect(
    resolveProviderAccess({
      providerId: 'gemini',
      access: {
        mode: 'byok',
        providerId: 'gemini',
        apiKey: 'user-gemini-key',
      },
    }),
  ).toEqual({
    providerId: 'gemini',
    apiKey: 'user-gemini-key',
  });
});

it('resolves custom gemini byok access with a base URL', () => {
  expect(
    resolveProviderAccess({
      providerId: 'gemini',
      access: {
        mode: 'byok',
        providerId: 'gemini',
        apiKey: 'user-gemini-key',
        baseURL: 'https://right.codes/gemini',
      },
    }),
  ).toEqual({
    providerId: 'gemini',
    apiKey: 'user-gemini-key',
    baseURL: 'https://right.codes/gemini',
  });
});

it('does not fall back to server secrets when gemini byok credentials are missing', () => {
  expect(() =>
    resolveProviderAccess({
      providerId: 'gemini',
      access: {
        mode: 'byok',
        providerId: 'gemini',
      },
    }, {
      GEMINI_API_KEY: 'server-secret',
    }),
  ).toThrow(/Gemini BYOK API key is required/i);
});
```

- [ ] **Step 2: Run the env tests and verify they fail**

Run:

```bash
cd SlideTutor-AI && npm test -- api/lib/env.test.ts
```

Expected:

- FAIL because Gemini BYOK access currently has no `baseURL` support and still falls back to server secrets

- [ ] **Step 3: Implement the backend resolver changes**

Implementation notes:

- extend the Gemini BYOK access payload to allow `baseURL?: string`
- extend the resolved Gemini access union to:

```ts
{
  providerId: 'gemini';
  apiKey: string;
  baseURL?: string;
}
```

- for `access.mode === 'byok'`, require a user-supplied Gemini API key
- when Gemini BYOK includes `baseURL`, return it unchanged after trimming
- keep platform Gemini resolution on server-held secrets only

- [ ] **Step 4: Run the env tests again**

Run:

```bash
cd SlideTutor-AI && npm test -- api/lib/env.test.ts
```

Expected:

- PASS with official/custom Gemini backend access resolution locked

- [ ] **Step 5: Commit**

```bash
git add SlideTutor-AI/api/lib/env.ts SlideTutor-AI/api/lib/env.test.ts
git commit -m "feat: resolve gemini custom endpoint access"
```

## Task 5: Make Gemini runtime and capability checks honor custom base URLs

**Files:**
- Modify: `SlideTutor-AI/api/lib/generateService.ts`
- Create: `SlideTutor-AI/api/lib/generateService.gemini.test.ts`
- Modify: `SlideTutor-AI/api/lib/modelCapabilityProbe.ts`
- Modify: `SlideTutor-AI/api/lib/modelCapabilityProbe.test.ts`

- [ ] **Step 1: Write the failing runtime and probe tests**

Create `SlideTutor-AI/api/lib/generateService.gemini.test.ts` with constructor-argument assertions:

```ts
it('constructs GoogleGenAI with baseURL for custom gemini runtime requests', async () => {
  resolveProviderAccessMock.mockReturnValue({
    providerId: 'gemini',
    apiKey: 'user-gemini-key',
    baseURL: 'https://right.codes/gemini',
  });

  await createGenerationStream(/* followup payload */);

  expect(googleGenAiCtorMock).toHaveBeenCalledWith({
    apiKey: 'user-gemini-key',
    baseURL: 'https://right.codes/gemini',
  });
});

it('constructs GoogleGenAI without baseURL for official gemini runtime requests', async () => {
  resolveProviderAccessMock.mockReturnValue({
    providerId: 'gemini',
    apiKey: 'user-gemini-key',
  });

  await createGenerationStream(/* followup payload */);

  expect(googleGenAiCtorMock).toHaveBeenCalledWith({
    apiKey: 'user-gemini-key',
  });
});
```

Add tests in `SlideTutor-AI/api/lib/modelCapabilityProbe.test.ts` for:

```ts
it('probes known gemini models through the custom base URL instead of returning registry-only usable', async () => {
  resolveProviderAccessMock.mockReturnValue({
    providerId: 'gemini',
    apiKey: 'user-gemini-key',
    baseURL: 'https://right.codes/gemini',
  });

  const result = await probeModelCapabilities({
    providerId: 'gemini',
    modelId: 'gemini-2.5-flash',
    access: {
      mode: 'byok',
      providerId: 'gemini',
      apiKey: 'user-gemini-key',
      baseURL: 'https://right.codes/gemini',
    },
  });

  expect(generateContentMock).toHaveBeenCalledTimes(1);
  expect(googleGenAiCtorMock).toHaveBeenCalledWith({
    apiKey: 'user-gemini-key',
    baseURL: 'https://right.codes/gemini',
  });
  expect(result.status).toBe('usable');
});
```

- [ ] **Step 2: Run the runtime and probe tests and verify they fail**

Run:

```bash
cd SlideTutor-AI && npm test -- api/lib/generateService.gemini.test.ts api/lib/modelCapabilityProbe.test.ts
```

Expected:

- FAIL because Gemini runtime constructors do not accept `baseURL` and known Gemini models short-circuit capability checks before probing the custom endpoint

- [ ] **Step 3: Implement runtime and probe alignment**

Implementation notes:

- add a small helper in `generateService.ts`:

```ts
function createGeminiClient(input: { apiKey: string; baseURL?: string }) {
  return input.baseURL
    ? new GoogleGenAI({ apiKey: input.apiKey, baseURL: input.baseURL })
    : new GoogleGenAI({ apiKey: input.apiKey });
}
```

- use that helper for:
  - Gemini generation
  - Gemini `distill`
  - any non-moderation Gemini runtime path
- leave the moderation helper on server-held official Gemini config for now; it is outside this feature boundary
- in `modelCapabilityProbe.ts`, do not short-circuit known Gemini models to immediate `usable` when the BYOK access includes a custom base URL
- instead:
  - execute one provider probe against the custom endpoint
  - keep registry-derived `capabilitySummary` on success
  - preserve auth/network failure normalization on probe failure

- [ ] **Step 4: Run the runtime and probe tests again**

Run:

```bash
cd SlideTutor-AI && npm test -- api/lib/generateService.gemini.test.ts api/lib/modelCapabilityProbe.test.ts
```

Expected:

- PASS with custom Gemini runtime and custom Gemini probe alignment covered

- [ ] **Step 5: Commit**

```bash
git add SlideTutor-AI/api/lib/generateService.ts SlideTutor-AI/api/lib/generateService.gemini.test.ts SlideTutor-AI/api/lib/modelCapabilityProbe.ts SlideTutor-AI/api/lib/modelCapabilityProbe.test.ts
git commit -m "feat: honor gemini custom endpoints at runtime"
```

## Task 6: Update docs and run the verification set

**Files:**
- Modify: `docs/backend/byok-capability-check.md`
- Modify: `docs/user_guide/access-modes.md`
- Modify: `docs/changelog/CHANGELOG_TECH.md`

- [ ] **Step 1: Update the docs**

Document:

- `My API > Gemini` now supports `Google Official / Custom`
- `Custom` requires `Gemini Base URL`
- `Platform API` remains unchanged
- Gemini custom capability checks run against the custom endpoint instead of reusing the official route

- [ ] **Step 2: Run the targeted verification suite**

Run:

```bash
cd SlideTutor-AI && npm run lint
cd SlideTutor-AI && npm test -- src/config/models.test.ts src/store/uiStore.test.ts src/components/SettingsModal.test.tsx src/lib/api/apiClient.test.ts api/lib/env.test.ts api/lib/modelCapabilityProbe.test.ts api/lib/generateService.gemini.test.ts
```

Expected:

- `npm run lint`: PASS
- targeted Vitest suite: PASS

- [ ] **Step 3: Run a regression sanity check on adjacent Gemini and hosted tests**

Run:

```bash
cd SlideTutor-AI && npm test -- api/lib/generateService.platform.test.ts test/workers/generate-stream.worker.test.ts
```

Expected:

- PASS, confirming that platform routing and Worker request forwarding did not regress

- [ ] **Step 4: Commit the docs and verification cleanups**

```bash
git add docs/backend/byok-capability-check.md docs/user_guide/access-modes.md docs/changelog/CHANGELOG_TECH.md
git commit -m "docs: record gemini custom endpoint support"
```

- [ ] **Step 5: Final review**

Before execution handoff, confirm:

- Gemini official still works with only API key
- Gemini custom requires base URL before readiness checks run
- runtime and capability checks share the same Gemini endpoint route
- `Platform API` still ignores Gemini BYOK endpoint settings
