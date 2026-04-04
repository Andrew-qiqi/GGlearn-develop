# API Design

Last updated: 2026-04-05

## Public Base

The canonical public base is the Cloudflare Worker that serves both the SPA and the public APIs.

## Endpoints

### `POST /api/generate`

Purpose:

- explanation generation
- follow-up answers
- chunk regeneration
- distill flow
- quiz generation and evaluation

Request headers:

- `Content-Type: application/json`
- `X-API-Token: <token>` when token auth is enabled

Request body highlights:

```json
{
  "providerId": "gemini | openai-compatible",
  "modelId": "provider-model-id",
  "endpointPreset": "qwen | doubao | custom",
  "access": {
    "mode": "byok",
    "providerId": "gemini | openai-compatible",
    "apiKey": "user-supplied-key",
    "baseURL": "https://provider.example/v1",
    "endpointPreset": "qwen | doubao | custom"
  }
}
```

BYOK routing rules:

- `gemini` accepts a user-provided API key or falls back to `GEMINI_API_KEY` when present for migration compatibility.
- `openai-compatible` accepts a user-provided `apiKey + baseURL` pair through one shared adapter path.
- preset `openai-compatible` routes (`qwen`, `doubao`) may still fall back to the matching server env secret during the migration window.
- malformed BYOK inputs are treated as request validation problems, not teaching-logic failures.

Response contract:

- content type: `text/plain; charset=utf-8`
- body: streamed plain-text chunks in the same order the frontend hooks consume today
- response headers may include:
  - `x-slidetutor-parse-mode: normal | degraded`
  - `x-slidetutor-parser-remaining: <number>`

Notes:

- Worker route applies origin checks, optional token auth, rate limiting, and request logging.
- Teaching prompts, structured artifacts, and frontend parsing contracts are intentionally unchanged by the Cloudflare migration.
- In Phase 04, BYOK model access and platform-funded parsing are intentionally split concerns.
- In Phase 05, `explain` requests resolve document parsing through a shared parser-access layer instead of calling Azure directly.
- If parser quota is exhausted or the parser is unavailable, the response still streams teaching output, but `x-slidetutor-parse-mode` becomes `degraded`.

### `GET /api/get-token`

Purpose:

- mint a short-lived API token for `/api/generate`

Response:

```json
{
  "token": "base64(payload).base64(signature)",
  "expiresIn": 300
}
```

### `POST /api/parse`

Purpose:

- run Azure Document Intelligence layout extraction for a slide image
- deduct one daily platform-funded parser use only after a successful parse

Request body:

```json
{
  "base64Image": "data:image/jpeg;base64,..."
}
```

Response:

```json
{
  "blocks": [
    {
      "id": "b0",
      "type": "text",
      "text": "Example block",
      "bbox": [0, 0, 100, 100]
    }
  ],
  "used": 1,
  "remaining": 9,
  "limit": 10,
  "dateKey": "2026-04-05",
  "parseMode": "normal"
}
```

Quota reached response:

```json
{
  "error": "Daily document parsing limit reached",
  "code": "PARSER_LIMIT_REACHED",
  "used": 10,
  "remaining": 0,
  "limit": 10,
  "dateKey": "2026-04-05"
}
```

Unavailable response:

```json
{
  "error": "Document parsing is unavailable",
  "code": "PARSER_UNAVAILABLE",
  "used": 0,
  "remaining": 10,
  "limit": 10,
  "dateKey": "2026-04-05"
}
```

Notes:

- the Worker route preserves the existing block shape
- unauthorized origins return a route-specific JSON `403`
- quota is enforced server-side through Cloudflare D1 using an anonymous identity derived from `ip_hash + date_key`
- the current daily parser limit is `10`

### `GET /api/parser-usage`

Purpose:

- report the current anonymous user's daily platform-funded parser usage

Response:

```json
{
  "used": 3,
  "remaining": 7,
  "limit": 10,
  "dateKey": "2026-04-05"
}
```

Notes:

- this endpoint is designed for the settings modal, not a persistent quota banner
- if D1 or `USAGE_HASH_SECRET` is missing, the route returns an empty summary and parsing degrades elsewhere

### `POST /api/feedback`

Purpose:

- collect user feedback from the settings modal

Request body:

```json
{
  "type": "Suggestion",
  "reason": "Please add keyboard shortcuts.",
  "images": [],
  "contactAgreed": true,
  "email": "student@example.com"
}
```

Success response:

```json
{
  "success": true,
  "message": "Feedback sent successfully"
}
```

Failure response:

```json
{
  "error": "Notification provider is not configured."
}
```

Notes:

- feedback delivery now runs through the Worker notification adapter
- local development can use log-only notification mode
