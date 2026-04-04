# API Design

Last updated: 2026-04-04

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

Notes:

- Worker route applies origin checks, optional token auth, rate limiting, and request logging.
- Teaching prompts, structured artifacts, and frontend parsing contracts are intentionally unchanged by the Cloudflare migration.
- In Phase 04, BYOK model access and platform-funded parsing are intentionally split concerns.

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
  ]
}
```

Notes:

- the Worker route preserves the existing block shape
- unauthorized origins return a route-specific JSON `403`

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
