# Deployment Architecture

Last updated: 2026-04-04

## Current Target

SlideTutor AI is now intended to ship from one Cloudflare Worker base URL.

The Worker is responsible for:

- serving the built SPA assets
- handling `/api/get-token`
- handling `/api/parse`
- handling `/api/generate`
- handling `/api/feedback`
- handling `/api/credits/balance`
- handling `/api/recharge-intent`
- handling `/api/payment-webhook`

This removes the old Vercel-first split between static hosting and serverless API routes.

## Runtime Topology

```text
Browser
  -> Cloudflare Worker
       -> Static asset response for app routes
       -> /api/get-token
       -> /api/parse
       -> /api/generate
       -> /api/feedback
```

API paths are dispatched before SPA fallback, so direct navigation to `/api/*` must return API responses or JSON errors instead of `index.html`.

## Local Development

Primary path:

```bash
npm run dev
```

This uses the Cloudflare/Vite worker-oriented setup.

Legacy Node shell:

```bash
npm run dev:node
```

The Node shell still exists only as a compatibility path while migration work is finishing. It is no longer the default runtime assumption.

## Build And Deploy

Build:

```bash
npm run build
```

Deploy:

```bash
npm run deploy
```

Worker configuration lives in [wrangler.jsonc](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/wrangler.jsonc).

## Required Secrets

Core runtime secrets:

- `APP_URL`
- `SHARED_APP_URL` if a secondary public origin is used
- `VITE_CLERK_PUBLISHABLE_KEY` or `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` for the SPA-side Clerk bootstrap
- `CLERK_SECRET_KEY` or `CLERK_JWT_KEY` for Worker-side platform-session verification
- `GEMINI_API_KEY`
- `DOUBAO_API_KEY` when that provider is enabled
- `QWEN_API_KEY` when that provider is enabled
- `VOLCENGINE_ACCESS_KEY_ID`
- `VOLCENGINE_SECRET_ACCESS_KEY`
- `ENABLE_TOKEN_AUTH`
- `API_TOKEN_SECRET`
- `PAYMENT_PROVIDER`
- `ZPAY_PID` when `PAYMENT_PROVIDER=zpay`
- `ZPAY_PKEY` when `PAYMENT_PROVIDER=zpay`
- `ZPAY_PAYMENT_TYPE` when `PAYMENT_PROVIDER=zpay`

Notification secrets:

- `NOTIFICATION_PROVIDER`
- `RESEND_API_KEY` when `NOTIFICATION_PROVIDER=resend`
- `NOTIFICATION_FROM_EMAIL`
- `FEEDBACK_TO_EMAIL`
- `SECURITY_ALERT_TO_EMAIL`

Local development can use `NOTIFICATION_PROVIDER=log` to avoid external delivery.

Clerk note:

- the Clerk publishable key is a frontend build variable, not just a Worker runtime secret
- if the publishable key is missing, the app now degrades to `My API` mode instead of crashing at boot

ZPAY note:

- production recharge now uses the ZPAY page-redirect `submit.php` checkout flow
- `APP_URL` must be the canonical public origin because the Worker derives both `return_url` and `notify_url` from it
- `/api/payment-webhook` must remain publicly reachable and must answer successful callbacks with plain-text `success`

## Observability

Worker logs and traces are enabled in Wrangler config.

Important signals to watch after deployment:

- `/api/generate` request volume and status distribution
- token-auth `401` frequency
- origin-check `403` frequency
- rate-limit `429` frequency
- feedback delivery failures

## Cutover Checklist

- confirm Worker secrets are present
- confirm `/api/get-token` returns JSON
- confirm `/api/parse` returns JSON or a route-specific JSON error
- confirm `/api/generate` streams plain text
- confirm `/api/feedback` returns the existing success contract on successful delivery
- confirm `/api/recharge-intent` returns a ZPAY checkout URL when `PAYMENT_PROVIDER=zpay`
- confirm `/api/payment-webhook` is reachable from the public internet and returns plain-text `success` for valid ZPAY callbacks
- confirm direct browser navigation to `/api/*` does not return the SPA shell
