# Access Modes

Last updated: 2026-04-14

## Overview

SlideTutor keeps two explicit access paths:

- `My API`
- `Platform API`

The product does not auto-switch providers for you. You choose the path that fits your setup.

## My API

Use `My API` when you want SlideTutor to send model requests with your own credentials.

Key traits:

- credentials stay in local browser storage
- parser remains platform-managed in the current product version
- you can choose `Gemini` or an `OpenAI-compatible` endpoint
- `Gemini` now supports `Google Official` and `Custom`
- `Gemini Custom` requires a `Gemini Base URL`
- Gemini model choice still stays on the curated official model dropdown
- `OpenAI-compatible` supports preset paths such as `Qwen` and `Doubao`, plus a custom base URL

China note:

- provider availability can vary by region
- China-based users should not assume `Gemini` is always available
- if one `My API` provider is unavailable, choose another `My API` provider or switch to `Platform API`

Gemini note:

- choose `Google Official` if you want the standard Gemini endpoint
- choose `Custom` only when your Gemini-compatible relay requires a custom base URL
- compatibility checks follow the active Gemini route, so changing `Gemini Base URL` will require a fresh readiness check

## Platform API

Use `Platform API` when you want SlideTutor to use your account credits instead of browser-stored model keys.

Key traits:

- requires sign-in
- uses your SlideTutor account credits
- keeps parser platform-managed
- keeps model choice explicit inside the supported platform boundary
- does not expose Gemini custom routing settings

## What This Does Not Change

- `BYOK-first` is still the main public path
- `Platform API` does not replace `My API`
- parser BYOK is still out of scope right now
- the app should not auto-switch providers when a provider is unavailable

## Related Docs

- [../operations/china-operator-checklist.md](../operations/china-operator-checklist.md)
