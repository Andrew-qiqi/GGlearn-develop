# API Design

Last updated: 2026-04-15

## 2026-04-15 Atomic credits commit hardening

Hosted credit mutations now use one D1 batched transaction per completion path so the balance write, ledger insert, and final completion marker succeed together or roll back together.

This hardening applies to:

- hosted `Analyze` finalization
- hosted `followup`, regenerate, and quiz deductions
- ZPAY recharge completion

The external route contracts stay unchanged:

- valid ZPAY callbacks still return plain-text `success`
- hosted billing still remains success-only
- duplicate completion calls stay idempotent instead of replaying balance mutations

## 2026-04-05 Platform API and Credits

### Operational route observability

The following routes now emit structured Worker logs with `requestId`, `path`, `status`, `durationMs`, `method`, and low-sensitivity route metadata when relevant:

- `/api/parse`
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
    "endpointPreset": "qwen | doubao | custom",
    "parser": {
      "providerId": "llamaparse",
      "apiKey": "llx-user-key"
    }
  }
}
```

Hosted/platform rules:

- `Platform API` requires Clerk auth before the Worker calls generation logic
- hosted `Analyze` is treated as one billable action even though the frontend still runs `explain -> distill`
- hosted `Analyze` returns `x-slidetutor-analyze-attempt-id` from the successful `explain` preflight
- hosted `distill` must send `taskData.hostedAnalyzeAttemptId`
- hosted `Analyze` charges exactly once after successful `parse + explain + distill`
- hosted credit commits now happen through one D1 batched transaction so balance mutation and ledger persistence do not drift apart on retries
- if hosted parser access hits upstream Volcengine throttling, the Worker rejects before streaming with `code = "PLATFORM_PARSER_RATE_LIMITED"`
- if hosted parser access degrades because the platform parser is unavailable, the Worker rejects before streaming with `code = "PLATFORM_PARSER_UNAVAILABLE"`
- hosted `followup`, `regenerate_chunk`, `regenerate_followup`, `generate_questions`, and `evaluate_answers` preflight credits before execution and deduct only after successful stream completion
- hosted action pricing is now:
  - `analyze = 3`
  - `followup = 1`
  - `card_regenerate = 1`
  - `generate_questions = 1`
  - `evaluate_answers = 1`
- runtime tasks `regenerate_chunk` and `regenerate_followup` both map to hosted action `card_regenerate`

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
    "endpointPreset": "qwen | doubao | custom",
    "parser": {
      "providerId": "llamaparse",
      "apiKey": "llx-user-key"
    }
  }
}
```

BYOK routing rules:

- `gemini` requires a user-provided local API key when `My API` is selected.
- `openai-compatible` requires a user-provided `apiKey + baseURL` pair through one shared adapter path.
- `My API` may optionally add `parser = { providerId: "llamaparse", apiKey }`.
- if `My API` omits parser config, `explain` keeps the intentional no-parser degraded analysis path.
- if `My API` enables `LlamaParse`, parser failures use `BYOK_PARSER_FAILED` or `BYOK_PARSER_TIMEOUT`.
- platform-mode requests use server-held provider secrets and do not read browser-local credentials.
- platform-mode requests keep a platform-managed Volcengine parser and do not accept parser configuration from the browser.
- malformed or incomplete BYOK inputs are treated as request validation problems, not teaching-logic failures.

Response contract:

- content type: `text/plain; charset=utf-8`
- body: streamed plain-text chunks in the same order the frontend hooks consume today
- response headers may include:
  - `x-slidetutor-parse-mode: normal | degraded`
  - `x-slidetutor-analyze-attempt-id: <attempt-id>` for hosted `task = explain`

Notes:

- Worker route applies origin checks, optional token auth, rate limiting, and request logging.
- Worker route throttling returns `code = "ROUTE_RATE_LIMITED"`.
- Teaching prompts, structured artifacts, and frontend parsing contracts are intentionally unchanged by the Cloudflare migration.
- In Phase 06, `My API` and `Platform API` are now explicit frontend modes.
- In Phase 06, BYOK requests no longer fall back to server-side model secrets.
- `Platform API` explain requests resolve document parsing through the platform-managed Volcengine path.
- `My API` explain requests use `LlamaParse` only when parser BYOK is configured.
- `LlamaParse` results are normalized into `LayoutBlock[]` from the provider's page-level item structure before they enter prompt generation.
- when `LlamaParse` returns page-relative geometry such as `x/y/w/h` plus `page_width` / `page_height`, the backend converts that geometry into the shared `0..1000` `[ymin, xmin, ymax, xmax]` contract used by prompt grounding and frontend highlights.
- In hosted `Analyze`, degraded parser results do not stream teaching output and do not become a paid success.

### 2026-04-10 Phase 09 capability and structured-output hardening

Backend preflight now applies one backend-owned capability registry before provider execution:

- unknown models return `MODEL_CAPABILITY_UNKNOWN`
- unverified BYOK models return `MODEL_CAPABILITY_UNVERIFIED`
- models that fail the current hard product baseline return `MODEL_NOT_ELIGIBLE`

Provider config generation is now model-aware instead of task-only:

- Gemini emits `thinkingConfig.thinkingLevel` only when the resolved model capability supports thinking
- Gemini `explain` now uses a `6144` structured-output budget to reduce intermittent structured JSON truncation on longer slide explanations
- `distill` uses a `4096` structured-output budget for Gemini and OpenAI-compatible providers
- OpenAI-compatible structured tasks still use `response_format.type = "json_schema"`

`distill` hardening notes:

- Gemini `distill` is buffered server-side before the Worker returns the stream contract
- `finishReason = MAX_TOKENS` now becomes `STRUCTURED_OUTPUT_TRUNCATED`
- invalid final structured JSON becomes `STRUCTURED_OUTPUT_INVALID`
- this path intentionally does not auto-retry because the goal is to surface the real parameter/runtime failure clearly

### `POST /api/model-capability-check`

Purpose:

- run one explicit BYOK capability probe during settings save or stale first use

Response shape:

```json
{
  "status": "usable | unusable | pending | stale",
  "checkedAt": "2026-04-10T12:00:00.000Z",
  "lastErrorCode": "MODEL_NOT_ELIGIBLE",
  "capabilitySummary": {
    "structuredOutput": true,
    "streaming": true,
    "vision": true,
    "thinking": false
  }
}
```

Notes:

- this route is for explicit BYOK readiness checks, not for every normal generation request
- normal generation may still mark saved BYOK readiness `stale` after clear capability/configuration failures

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
- preserve one direct platform-parser debug/ops entrypoint for the current slide image flow

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
  "parseMode": "normal"
}
```

Rate-limited response:

```json
{
  "error": "Platform document parsing is temporarily rate limited",
  "code": "PLATFORM_PARSER_RATE_LIMITED"
}
```

Unavailable response:

```json
{
  "error": "Platform document parsing is unavailable",
  "code": "PLATFORM_PARSER_UNAVAILABLE"
}
```

Notes:

- the Worker route preserves the existing block shape while using Volcengine as the live platform parser provider
- unauthorized origins return a route-specific JSON `403` with `requestId`
- route throttling and platform parser failures are separate classes
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
