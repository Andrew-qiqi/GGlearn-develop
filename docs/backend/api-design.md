# API Design

Last updated: 2026-04-07

## 2026-04-05 Platform API and Credits

### Operational route observability

The following routes now emit structured Worker logs with `requestId`, `path`, `status`, `durationMs`, `method`, and low-sensitivity route metadata when relevant:

- `/api/parse`
- `/api/parser-usage`
- `/api/credits/balance`
- `/api/recharge-intent`
- `/api/payment-webhook`

For these routes, JSON error responses include `requestId` so support work can correlate the user-visible failure to Worker logs. Valid ZPAY callbacks remain the one exception because the response body must stay plain-text `success`.

Operator smoke steps for these routes live in [china-operator-checklist.md](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/docs/operations/china-operator-checklist.md).

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
- if hosted parser access degrades because the daily parser quota is exhausted for the current network, the Worker rejects before streaming with `code = "PLATFORM_PARSER_LIMIT_REACHED"`
- if hosted parser access degrades because the platform parser is unavailable, the Worker rejects before streaming with `code = "PLATFORM_PARSER_UNAVAILABLE"`
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

Error note:

- JSON failures include `requestId`

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

Error note:

- JSON failures include `requestId`

Response:

```json
{
  "orderId": "ord_123",
  "amountRmb": 1,
  "credits": 30,
  "provider": "zpay",
  "checkoutUrl": "https://zpayz.cn/submit.php?..."
}
```

Notes:

- current production adapter is `zpay`
- checkout uses the page-redirect `submit.php` flow
- the Worker derives `notify_url = <APP_URL>/api/payment-webhook`
- the Worker derives `return_url = <APP_URL>`

### `GET | POST /api/payment-webhook`

Current `zpay` contract:

- accepts the `notify_url` callback from ZPAY
- verifies MD5 signature from all callback params except `sign`, `sign_type`, and empty values
- accepts only `trade_status = TRADE_SUCCESS`
- validates that the paid RMB amount matches the stored recharge order
- remains idempotent when ZPAY retries the same callback

Important callback fields:

```text
pid
trade_no
out_trade_no
money
trade_status
sign
sign_type
```

Response:

```text
success
```

Error note:

- invalid callbacks return JSON errors with `requestId`
- valid ZPAY callbacks still return plain-text `success`

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
- In Phase 05, `explain` requests resolve document parsing through a shared Volcengine-backed parser-access layer.
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

- run Volcengine OCRPdf layout extraction for a slide image
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

- the Worker route preserves the existing block shape while using Volcengine as the live platform parser provider
- unauthorized origins return a route-specific JSON `403` with `requestId`
- quota is enforced server-side through Cloudflare D1 using an anonymous identity derived from `ip_hash + date_key`
- the current daily parser limit is `10`
- JSON errors include `requestId`

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
- JSON errors include `requestId`

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
