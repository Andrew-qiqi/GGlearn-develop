# Access Modes

Last updated: 2026-04-14

## Overview

GGlearn keeps two explicit access paths:

- `My API`
- `Platform API`

The product does not auto-switch providers for you. You choose the path that fits your setup.

## My API

Use `My API` when you want GGlearn to send model requests with your own credentials.

Key traits:

- credentials stay in local browser storage
- parser remains platform-managed in the current product version
- you can choose `Gemini` or an `OpenAI-compatible` endpoint
- `Gemini` now supports `Google Official` and `Custom`
- `Gemini Custom` requires a `Gemini Base URL`
- Gemini model choice still stays on the curated official model dropdown
- `OpenAI-compatible` supports preset paths such as `Qwen` and `Doubao`, plus a custom base URL
 
Routing note:

- `My API` Gemini routing (official vs custom base URL) is entirely controlled inside the browser and shares `selectedModel` metadata with Platform API, but it does not affect hosted routing.

China note:

- provider availability can vary by region
- China-based users should not assume `Gemini` is always available
- if one `My API` provider is unavailable, choose another `My API` provider or switch to `Platform API`

Gemini note:

- choose `Google Official` if you want the standard Gemini endpoint
- choose `Custom` when you need Gemini requests to use a non-default base URL
- compatibility checks follow the active Gemini route, so changing `Gemini Base URL` will require a fresh readiness check

## Platform API

Use `Platform API` when you want GGlearn to use your account credits instead of browser-stored model keys.

Key traits:

- requires sign-in
- uses your GGlearn account credits
- keeps parser platform-managed
- keeps model choice explicit inside the supported platform boundary
- does not expose Gemini custom routing settings
 
Routing note:

- `Platform API` Gemini routing is determined by Cloudflare Worker environment variables: `GEMINI_API_KEY` for official hosted access, or `PLATFORM_GEMINI_BASE_URL` + `PLATFORM_GEMINI_API_KEY` when relaying through a platform-managed Gemini router. `PLATFORM_GEMINI_BASE_URL` must be a valid absolute HTTP(S) URL. This is independent from the browser-managed `My API` Gemini settings even though the selected `modelId` is shared.
- current deployments still keep `GEMINI_API_KEY` configured even when hosted Gemini is relayed, because other server-side Gemini helpers still use the official key

## What This Does Not Change

- `BYOK-first` is still the main public path
- `Platform API` does not replace `My API`
- parser BYOK is still out of scope right now
- the app should not auto-switch providers when a provider is unavailable

## Related Docs

- [../operations/china-operator-checklist.md](../operations/china-operator-checklist.md)
