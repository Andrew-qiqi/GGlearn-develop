# Gemini Custom Endpoint Design

**Date:** 2026-04-14

## Goal

Allow `My API` users to keep using the existing Gemini model catalog while choosing between the official Gemini endpoint and a custom Gemini-compatible base URL.

The immediate product need is to support relay-style Gemini access such as `Right Codes` without turning it into a branded provider path or changing the current `Platform API` boundary.

## Problem Summary

SlideTutor currently treats `Gemini` BYOK as a single path:

- user selects a Gemini model from the shared model list
- user enters a Gemini API key
- backend calls `@google/genai` with `apiKey` only

That works for the official Gemini Developer API, but it does not cover Gemini relay endpoints that still speak the Gemini protocol while requiring a custom base URL.

This creates a product gap:

1. users who can access Gemini only through a relay cannot use the current Gemini BYOK path
2. the current UI incorrectly suggests that Gemini BYOK is only one connectivity mode
3. the existing capability check would become misleading if runtime requests and probe requests do not share the same Gemini endpoint configuration

The design goal is to close that gap without:

- adding a new provider family
- turning Gemini into an OpenAI-compatible path
- expanding `Platform API`
- letting users manually type arbitrary Gemini model IDs

## Agreed Design Direction

### 1. Keep the current Gemini provider, add a Gemini endpoint mode

Within `My API`, Gemini gets two explicit endpoint modes:

- `Google Official`
- `Custom`

This is a connectivity choice inside the existing Gemini adapter, not a new provider.

### 2. Preserve the existing Gemini model catalog

Gemini users continue to choose only from the current shared Gemini model dropdown.

There is no user-editable Gemini `modelId` field in this iteration.

Reason:

- it keeps the mental model simple
- it avoids re-opening Gemini capability truth and model admission in the same task
- it aligns with the current project boundary where model choice stays explicit but curated

### 3. Keep `Platform API` unchanged

`Platform API` must not expose or consume Gemini endpoint-mode settings.

This feature is strictly a `My API` enhancement.

Reason:

- current product contract keeps BYOK credentials browser-local
- `Platform API` uses server-held secrets and must remain provider-boundary-clean
- exposing a Gemini relay concept inside `Platform API` would imply a capability the hosted path does not intend to support

### 4. Keep user-facing copy generic

User-facing copy should say only:

- `Gemini Base URL`

It should not mention `Right Codes` or any other intermediary by name.

Reason:

- the feature is productized as generic custom Gemini routing
- the app should not imply endorsement of one specific relay
- documentation and support can still reference examples externally when needed

## Product Behavior

### Model and settings behavior

When the selected model provider is `gemini` and the access mode is `byok`, the settings panel shows:

1. Gemini endpoint mode selector
   - `Google Official`
   - `Custom`
2. Gemini API key input
3. Gemini base URL input only when `Custom` is active
   - hidden or visible-but-disabled when `Google Official` is active

### Validation rules

For Gemini BYOK:

- `Google Official` requires:
  - `Gemini API Key`
- `Custom` requires:
  - `Gemini API Key`
  - `Gemini Base URL`

Missing-field messaging stays generic:

- configure your `Gemini API Key`
- configure your `Gemini Base URL`

### Shared model list behavior

The shared model list remains unchanged:

- existing Gemini models remain selectable
- there is no custom Gemini model slot
- there is no separate Gemini relay model group

## Data Model Changes

Extend the existing frontend Gemini access state from:

```ts
gemini: {
  apiKey: string;
}
```

to:

```ts
gemini: {
  apiKey: string;
  endpointPreset: 'google-official' | 'custom';
  baseURL: string;
}
```

### Defaults

Default Gemini access values:

```ts
gemini: {
  apiKey: '',
  endpointPreset: 'google-official',
  baseURL: '',
}
```

### Why this shape

This keeps Gemini-specific routing state inside the existing Gemini access object and avoids:

- polluting `SelectedModel`
- inventing a new provider ID
- coupling Gemini endpoint mode to the shared model list format

## Request And Runtime Flow

### Frontend request assembly

When building BYOK access for Gemini:

- `Google Official`
  - send `providerId = 'gemini'`
  - send `apiKey`
  - do not send `baseURL`
- `Custom`
  - send `providerId = 'gemini'`
  - send `apiKey`
  - send `baseURL`

The selected Gemini `modelId` still comes only from the shared model dropdown.

### Backend access resolution

Backend provider-access resolution should treat Gemini as:

- official Gemini BYOK when no BYOK `baseURL` is supplied
- custom Gemini BYOK when a BYOK `baseURL` is supplied
- platform Gemini when `access.mode = 'platform'`

The resolved Gemini access shape therefore needs to support:

```ts
{
  providerId: 'gemini';
  apiKey: string;
  baseURL?: string;
}
```

### Gemini execution path

Gemini runtime execution should instantiate the SDK as:

- official:
  - `new GoogleGenAI({ apiKey })`
- custom:
  - `new GoogleGenAI({ apiKey, baseURL })`

This applies to:

- normal generation
- capability probing

The critical rule is that the probe path and runtime path must use the same effective Gemini endpoint configuration.

## Capability Check Behavior

### Why this matters

The current product automatically runs a capability check for BYOK configurations and caches the result. If Gemini custom routing is added only to runtime generation but not to the probe path, the UI can falsely report that a configuration is ready when only the official endpoint was actually tested.

### Required behavior

Gemini capability checks must execute against the same endpoint mode as the active Gemini BYOK configuration:

- `Google Official` probes the official Gemini endpoint
- `Custom` probes the configured custom Gemini base URL

### Capability invalidation rules

Saved Gemini capability state must be invalidated when any of the following change:

- Gemini endpoint mode changes between `google-official` and `custom`
- Gemini custom `baseURL` changes
- Gemini API key changes
- selected Gemini model changes

Expected invalidation result:

- mark the saved capability check as `stale` when there is a previous selection snapshot
- otherwise fall back to `pending`

## Migration Strategy

Existing saved user state only includes:

```ts
gemini: {
  apiKey: string;
}
```

On load, normalization should automatically migrate old Gemini settings to:

```ts
gemini: {
  apiKey: <existing value>,
  endpointPreset: 'google-official',
  baseURL: '',
}
```

Migration constraints:

- do not clear existing Gemini API keys
- do not alter the current selected model
- do not force the user into `Custom`
- do not silently infer any relay URL

## Error And Boundary Decisions

### User-facing errors

User-facing configuration errors remain generic and local to the chosen access path:

- missing Gemini key
- missing Gemini base URL for `Custom`

The product should not introduce relay-specific branding or suggestions in these errors.

### Platform boundary

`Platform API` must ignore Gemini BYOK endpoint fields entirely.

That means:

- no hosted read of browser-local Gemini `baseURL`
- no platform-mode UI for Gemini endpoint mode
- no implication that hosted Gemini can be redirected through user-configured relays

### Model boundary

This feature does not change Gemini model admission:

- still use current curated Gemini model IDs
- do not add freeform Gemini model input
- do not widen the product boundary to arbitrary Gemini model names

## Scope

### In scope

- Gemini endpoint mode selector inside `My API`
- generic Gemini base URL input for `Custom`
- frontend state normalization and migration
- frontend BYOK request assembly for Gemini custom base URL
- backend Gemini access resolution with optional BYOK base URL
- Gemini runtime execution against custom base URL
- Gemini capability probe execution against custom base URL
- capability invalidation updates
- tests and docs for the above

### Out of scope

- changes to `Platform API`
- branded `Right Codes` UI copy
- freeform Gemini model IDs
- new provider families
- OpenAI-compatible routing changes
- parser ownership changes

## Recommended Implementation Order

### Phase A: State and normalization

Update Gemini access state, defaults, and persisted-state normalization.

Success criterion:

- old saved Gemini configs load cleanly as `google-official`

### Phase B: Settings UI

Add Gemini endpoint mode selector and conditional Gemini base URL field.

Success criterion:

- users can clearly choose official vs custom Gemini routing in `My API`

### Phase C: Request and backend routing

Pass Gemini custom base URL through the BYOK request and into backend Gemini access resolution.

Success criterion:

- runtime Gemini generation can execute against a custom base URL

### Phase D: Capability-check alignment

Make Gemini capability probes use the same endpoint configuration as runtime execution and ensure route changes invalidate cached readiness.

Success criterion:

- no false-ready state caused by probing the wrong Gemini endpoint

### Phase E: Docs and tests

Add or update tests and developer documentation for the new Gemini custom endpoint path.

Success criterion:

- behavior is covered and future changes have a stable reference

## Validation Strategy

### Frontend tests

Add settings-panel coverage for:

- Gemini shows `Google Official / Custom`
- `Custom` reveals or enables `Gemini Base URL`
- `Google Official` does not require `Gemini Base URL`
- Gemini endpoint-mode changes invalidate capability readiness

Add API client coverage for:

- Gemini official BYOK payload sends only `apiKey`
- Gemini custom BYOK payload sends `apiKey + baseURL`

### Backend tests

Add provider-resolution coverage for:

- Gemini official resolves without BYOK `baseURL`
- Gemini custom resolves with BYOK `baseURL`
- platform mode ignores Gemini BYOK `baseURL`

Add runtime and probe coverage for:

- Gemini custom passes `baseURL` into `GoogleGenAI`
- Gemini official does not
- probe and runtime stay aligned on the effective Gemini endpoint

### Manual validation

Manual validation should cover:

1. fresh user, Gemini official
2. fresh user, Gemini custom with a valid base URL
3. saved user migration from old Gemini settings
4. capability state invalidation after switching official -> custom

## Final Product Call

SlideTutor should support custom Gemini relay endpoints by extending the existing Gemini BYOK adapter, not by introducing a new provider or changing the hosted path.

The product surface stays intentionally narrow:

- one curated Gemini model list
- one endpoint-mode selector
- one generic custom Gemini base URL field

This is the smallest design that solves the real user problem while preserving the current architecture and product boundaries.
