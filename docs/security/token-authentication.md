# Token Authentication

Last updated: 2026-04-04

## Purpose

Token authentication protects `POST /api/generate` from casual direct access while keeping the frontend flow lightweight.

The current public runtime path is Cloudflare Worker based, so token auth is now documented as a Worker-era mechanism instead of a Vercel-specific one.

## Flow

1. The frontend requests `GET /api/get-token`.
2. The Worker returns a signed token with `expiresIn: 300`.
3. The frontend includes `X-API-Token` when calling `POST /api/generate`.
4. If the Worker returns `401`, the frontend clears the cached token and retries once.

## Token Shape

```text
base64(payload).base64(signature)
```

Payload fields:

- `timestamp`
- `nonce`

Signature:

- HMAC-SHA256 over the payload using `API_TOKEN_SECRET`

## Worker Behavior

When `ENABLE_TOKEN_AUTH=true`:

- missing token returns `401` with code `MISSING_TOKEN`
- malformed or invalid signatures return `401` with a route-specific code
- expired tokens return `401` with a route-specific code

When `ENABLE_TOKEN_AUTH=false`:

- `/api/generate` skips token enforcement

## Required Secrets

- `ENABLE_TOKEN_AUTH`
- `API_TOKEN_SECRET`

These are route-scoped runtime requirements. The Worker should not fail startup just because token auth is disabled or because an unrelated provider secret is missing.

## Related Protection Layers

Token auth is not the only protection:

- origin and referer checks
- request rate limiting
- request logging and observability
- malicious-intent filtering in the generation pipeline

## Operational Notes

- keep `API_TOKEN_SECRET` private and rotate it when needed
- expect short-lived `401` retries on the frontend if a cached token expires
- monitor `401`, `403`, and `429` trends together after deployment because they describe different protection boundaries
