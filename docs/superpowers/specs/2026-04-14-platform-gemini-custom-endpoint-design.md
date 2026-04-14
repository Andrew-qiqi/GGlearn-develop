# Platform Gemini Custom Endpoint Design

**Date:** 2026-04-14

## Goal

Allow `Platform API` to route Gemini traffic through either:

- the official Gemini endpoint
- a platform-managed custom Gemini-compatible base URL

without changing the existing `My API` Gemini behavior and without splitting the shared `modelId` selection into two separate saved states.

## Problem Summary

SlideTutor already supports custom Gemini routing for `My API`, but `Platform API` still assumes one hosted Gemini path backed by the official server-side key.

This creates an operational gap:

1. platform operators may want to switch hosted Gemini traffic to a cheaper relay or intermediary
2. that routing choice is a platform operations decision, not a BYOK user decision
3. the current product boundary must prevent `My API` Gemini settings from affecting hosted Gemini routing

The immediate need is to let the platform operator configure a hosted custom Gemini source through Worker runtime variables while keeping the BYOK Gemini path unchanged.

## Agreed Design Direction

### 1. Keep `My API` and `Platform API` Gemini routing fully independent

`My API` continues to use browser-local Gemini BYOK settings:

- `apiKey`
- `endpointPreset`
- `baseURL`

`Platform API` must not read or derive behavior from those settings.

`Platform API` instead resolves Gemini routing only from Worker runtime configuration.

This is a hard boundary:

- changing BYOK Gemini official/custom mode must not affect hosted Gemini traffic
- changing hosted Gemini official/custom mode must not affect BYOK users

### 2. Keep the shared `modelId` selection unchanged

The current shared model picker remains the source of:

- `providerId`
- `modelId`

for both access modes.

This design does **not** introduce separate saved model choices for `My API` and `Platform API`.

Reason:

- the operator requirement is only about hosted route and credential isolation
- splitting model persistence would expand scope into frontend state behavior that is not needed for this change

### 3. Resolve hosted Gemini route from one environment-variable decision

`Platform API` Gemini routing should follow this rule:

- when `PLATFORM_GEMINI_BASE_URL` is empty:
  - use official Gemini
  - use `GEMINI_API_KEY`
- when `PLATFORM_GEMINI_BASE_URL` is non-empty:
  - use custom Gemini
  - require `PLATFORM_GEMINI_API_KEY`
  - require `PLATFORM_GEMINI_BASE_URL` to be a valid absolute URL

This design intentionally does **not** add a separate boolean feature flag.

Reason:

- the base URL itself already expresses whether the hosted route is official or custom
- adding a second toggle would create invalid combinations such as:
  - custom enabled with empty base URL
  - custom disabled with a filled base URL

### 4. Keep existing `GEMINI_API_KEY` for official hosted and internal Gemini usage

The existing `GEMINI_API_KEY` remains in the environment because current server-side code already uses it outside the hosted generation path.

This means:

- official hosted Gemini continues to use `GEMINI_API_KEY`
- custom hosted Gemini uses `PLATFORM_GEMINI_API_KEY`
- internal server-side Gemini helpers that already depend on `GEMINI_API_KEY` remain unchanged in this iteration

This keeps the rollout focused on hosted generation routing instead of broad Gemini credential refactoring.

## Product Boundary

### What changes

- platform operators gain a hosted Gemini custom route option through Worker env vars
- hosted Gemini requests can use a custom base URL without exposing that control in user-facing settings

### What does not change

- `My API` Gemini UI and request shape
- browser-local BYOK Gemini persistence
- shared Gemini model list
- shared `selectedModel` persistence
- existing platform auth and credit charging behavior

## Runtime Contract

### `My API`

No change.

The existing BYOK Gemini routing continues to work as follows:

- official Gemini when browser BYOK Gemini `baseURL` is absent
- custom Gemini when browser BYOK Gemini `baseURL` is present

### `Platform API`

Hosted Gemini routing becomes environment-driven:

#### Official hosted Gemini

Activated when:

- `PLATFORM_GEMINI_BASE_URL` is unset or empty

Resolved access:

```ts
{
  providerId: 'gemini',
  apiKey: env.GEMINI_API_KEY,
}
```

#### Custom hosted Gemini

Activated when:

- `PLATFORM_GEMINI_BASE_URL` is a non-empty string

Resolved access:

```ts
{
  providerId: 'gemini',
  apiKey: env.PLATFORM_GEMINI_API_KEY,
  baseURL: env.PLATFORM_GEMINI_BASE_URL,
}
```

Validation requirements:

- `PLATFORM_GEMINI_API_KEY` must exist
- `PLATFORM_GEMINI_BASE_URL` must parse as a valid absolute URL

## Environment Contract

Add these Worker runtime variables:

```env
PLATFORM_GEMINI_BASE_URL=""
PLATFORM_GEMINI_API_KEY=""
```

Semantics:

- `PLATFORM_GEMINI_BASE_URL=""`
  - hosted Gemini uses official routing
  - `PLATFORM_GEMINI_API_KEY` is ignored
- `PLATFORM_GEMINI_BASE_URL="https://example.com/gemini"`
  - hosted Gemini uses custom routing
  - `PLATFORM_GEMINI_API_KEY` is required

## Backend Implementation Shape

### Provider access resolution

The backend Gemini provider resolution should branch first on access mode:

1. `access.mode = 'byok'`
   - preserve current BYOK Gemini behavior
2. `access.mode = 'platform'`
   - ignore browser Gemini BYOK settings
   - read hosted Gemini route from env

### Gemini runtime execution

No new provider family is needed.

Existing Gemini runtime construction already supports:

- `new GoogleGenAI({ apiKey })`
- `new GoogleGenAI({ apiKey, httpOptions: { baseUrl } })`

So the implementation work should stay concentrated in provider-access resolution plus env validation.

## Error Handling

Hosted custom Gemini misconfiguration should fail as a server configuration problem, not as a user BYOK validation problem.

Required failure cases:

1. `PLATFORM_GEMINI_BASE_URL` is present but `PLATFORM_GEMINI_API_KEY` is missing
2. `PLATFORM_GEMINI_BASE_URL` is present but invalid

Desired behavior:

- fail deterministically before provider execution
- keep the error clearly attributable to hosted configuration
- do not suggest that the end user should edit `My API` settings

## Documentation Updates

The rollout should update:

- `SlideTutor-AI/.env.example`
- `docs/architecture/deployment.md`
- `docs/user_guide/access-modes.md`
- `docs/backend/platform-model-configuration.md`
- `docs/changelog/CHANGELOG_TECH.md`

Documentation must clearly state:

- hosted Gemini route selection is platform-managed
- BYOK Gemini route selection is user-managed
- the two routing systems are independent

## Validation Strategy

Validation should cover:

1. env resolution tests
   - official hosted Gemini uses `GEMINI_API_KEY` when `PLATFORM_GEMINI_BASE_URL` is empty
   - custom hosted Gemini uses `PLATFORM_GEMINI_API_KEY` and `PLATFORM_GEMINI_BASE_URL`
   - custom hosted Gemini rejects missing `PLATFORM_GEMINI_API_KEY`
   - custom hosted Gemini rejects invalid `PLATFORM_GEMINI_BASE_URL`
2. runtime wiring tests
   - official hosted Gemini constructs `GoogleGenAI` without `baseURL`
   - custom hosted Gemini constructs `GoogleGenAI` with `httpOptions.baseUrl`
3. regression coverage
   - BYOK Gemini behavior remains unchanged
   - platform OpenAI-compatible behavior remains unchanged

## Migration Notes

This change is intentionally low-risk for existing deployments:

- existing deployments that do not set `PLATFORM_GEMINI_BASE_URL` keep using official hosted Gemini
- no frontend migration is required
- no browser-stored state migration is required

Operators who want hosted custom Gemini only need to add:

- `PLATFORM_GEMINI_BASE_URL`
- `PLATFORM_GEMINI_API_KEY`

## Explicit Non-Goals

- separate saved `modelId` state for `My API` and `Platform API`
- user-facing `Platform API` Gemini official/custom controls
- broader refactor of all server-side Gemini credential consumers
- turning hosted Gemini custom routing into an OpenAI-compatible preset
