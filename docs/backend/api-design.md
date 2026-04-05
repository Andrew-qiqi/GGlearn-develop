# API Design

Last updated: 2026-04-05

## 2026-04-05 Platform API and Credits

### `POST /api/generate` platform-mode additions

Request headers:

- `Authorization: Bearer <clerk-session-token>` when `access.mode = "platform"`

Request body access modes:

```json
{
  "access": {
    "mode": "platform"
  }
}
```

or

```json
{
  "access": {
    "mode": "byok",
    "providerId": "gemini | openai-compatible",
    "apiKey": "user-supplied-key",
    "baseURL": "https://provider.example/v1",
    "endpointPreset": "qwen | doubao | custom"
  }
}
```

Hosted/platform rules:

- `Platform API` requires Clerk auth before the Worker calls generation logic
- hosted `Analyze` is treated as one billable action even though the frontend still runs `explain -> distill`
- hosted `Analyze` returns `x-slidetutor-analyze-attempt-id` from the successful `explain` preflight
- hosted `distill` must send `taskData.hostedAnalyzeAttemptId`
- hosted `Analyze` charges exactly once after successful `parse + explain + distill`
- if hosted parser access degrades, the Worker rejects before streaming with `code = "PLATFORM_ANALYZE_UNAVAILABLE"`
- hosted `followup`, `generate_questions`, and `evaluate_answers` preflight credits before execution and deduct only after successful stream completion
- hosted unsupported actions currently return `code = "UNSUPPORTED_PLATFORM_ACTION"`:
  - `regenerate_chunk`
  - `regenerate_followup`
  - `evaluate_note`

Hosted billing errors:

```json
{
  "error": "Not enough credits to continue.",
  "code": "INSUFFICIENT_CREDITS",
  "requiredCredits": 1,
  "currentBalance": 0
}
```

### `GET /api/credits/balance`

Purpose:

- lazily create a hosted credit account for the signed-in user
- grant the one-time starter credits on first lookup

Response:

```json
{
  "balance": 10,
  "starterCredits": 10,
  "currency": "credits"
}
```

### `POST /api/recharge-intent`

Purpose:

- create a recharge order from RMB input
- return provider checkout metadata

Request body:

```json
{
  "amountRmb": 1
}
```

Response:

```json
{
  "orderId": "ord_123",
  "amountRmb": 1,
  "credits": 30,
  "provider": "mock",
  "checkoutUrl": "https://slidetutor.ai/mock-pay/ord_123"
}
```

### `POST /api/payment-webhook`

Current mock adapter contract:

- header: `x-payment-webhook-secret: <PAYMENT_WEBHOOK_SECRET>`
- body:

```json
{
  "orderId": "ord_123",
  "providerOrderId": "mock_ord_123",
  "status": "paid"
}
```

Response:

```json
{
  "ok": true,
  "balance": 40,
  "alreadyCompleted": false
}
```

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

- `gemini` requires a user-provided local API key when `My API` is selected.
- `openai-compatible` requires a user-provided `apiKey + baseURL` pair through one shared adapter path.
- platform-mode requests use server-held provider secrets and do not read browser-local credentials.
- malformed or incomplete BYOK inputs are treated as request validation problems, not teaching-logic failures.

Response contract:

- content type: `text/plain; charset=utf-8`
- body: streamed plain-text chunks in the same order the frontend hooks consume today
- response headers may include:
  - `x-slidetutor-parse-mode: normal | degraded`
  - `x-slidetutor-parser-remaining: <number>`
  - `x-slidetutor-analyze-attempt-id: <attempt-id>` for hosted `task = explain`

Notes:

- Worker route applies origin checks, optional token auth, rate limiting, and request logging.
- Teaching prompts, structured artifacts, and frontend parsing contracts are intentionally unchanged by the Cloudflare migration.
- In Phase 06, `My API` and `Platform API` are now explicit frontend modes.
- In Phase 06, BYOK requests no longer fall back to server-side model secrets.
- In Phase 05, `explain` requests resolve document parsing through a shared parser-access layer instead of calling Azure directly.
- In hosted `Analyze`, degraded parser results do not stream teaching output and do not become a paid success.

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
