# China Operator Checklist

Last updated: 2026-04-14

Use this checklist when validating the live China-facing chain on Cloudflare.

## Scope

This checklist covers:

- Clerk frontend bootstrap and Worker-side session verification
- Volcengine parser availability
- D1 bindings for credits and parser usage
- ZPAY recharge intent and payment webhook handling
- requestId-based support lookup for the operational routes

## Required Configuration

### Build-time variables

These must exist in the frontend build environment, not only the Worker runtime:

- `VITE_CLERK_PUBLISHABLE_KEY` or `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`

### Worker runtime secrets and bindings

- `APP_URL`
- `SHARED_APP_URL` when a secondary public origin is used
- `CLERK_SECRET_KEY` or `CLERK_JWT_KEY`
- `VOLCENGINE_ACCESS_KEY_ID`
- `VOLCENGINE_SECRET_ACCESS_KEY`
- `CREDITS_DB`
- `PARSER_USAGE_DB`
- `USAGE_HASH_SECRET`
- `PAYMENT_PROVIDER=zpay`
- `ZPAY_PID`
- `ZPAY_PKEY`
- `ZPAY_PAYMENT_TYPE`
- `NOTIFICATION_PROVIDER=resend` for production feedback delivery
- `RESEND_API_KEY`
- `NOTIFICATION_FROM_EMAIL`
- `FEEDBACK_TO_EMAIL`
- `SECURITY_ALERT_TO_EMAIL`

### Coupling to verify

- `APP_URL` is the canonical public origin
- ZPAY `notify_url` is derived as `<APP_URL>/api/payment-webhook`
- ZPAY `return_url` is derived as `<APP_URL>`
- if `APP_URL` is wrong, both the payment return path and the webhook callback path drift together
- feedback delivery also depends on the live public domain family because the sender domain behind `NOTIFICATION_FROM_EMAIL` must already be verified for Resend
- the current live public origin recorded in ops evidence is `https://www.slidetutor-ai.com`; do not assume older local examples using `slidetutor.ai` are authoritative

## Route Observability Contract

The operational routes now emit structured Worker logs with:

- `requestId`
- `path`
- `status`
- `durationMs`
- `method`
- `providerId`, `task`, or `code` when relevant

JSON error responses for these routes include `requestId`:

- `/api/parse`
- `/api/parser-usage`
- `/api/credits/balance`
- `/api/recharge-intent`
- `/api/payment-webhook`

Valid ZPAY callbacks are the one exception: they must still return plain-text `success`.

## Smoke Checklist

### 1. Confirm the deployed app can boot Clerk

1. Open the deployed app at `APP_URL`.
2. Open the settings surface and confirm `Platform API` is visible.
3. If the console shows `Clerk publishable key is missing`, the build-time public key was not present when the frontend bundle was built.

Expected result:

- the page loads normally
- `Platform API` is available instead of silently disappearing because of a broken build

### 2. Confirm Worker-side Clerk verification

1. Sign in through the deployed app.
2. Switch to `Platform API`.
3. Trigger the hosted balance fetch from the settings panel.

Expected result:

- `/api/credits/balance` returns `200`
- the UI shows the current credits balance

If it fails:

- `401` usually means the bearer token is missing or invalid
- `500` usually means `CLERK_SECRET_KEY` or `CLERK_JWT_KEY` is missing in the Worker runtime
- capture the JSON `requestId` and search Worker logs with that value

### 3. Confirm starter or existing balance

1. For a new account, verify the first balance lookup returns starter credits.
2. For an existing account, verify the balance matches the latest known recharge state.

Expected result:

- new user: `balance = 10`, `starterCredits = 10`
- existing user: current balance is preserved

### 4. Confirm Volcengine parser availability

1. Call `/api/parser-usage` from the app settings or a direct request.
2. Trigger one platform parser request through `/api/parse` or a hosted analyze flow that needs parsing.

Expected result:

- `/api/parser-usage` returns a usage summary
- `/api/parse` returns either parsed blocks or a JSON error with `requestId`

If it fails:

- `PARSER_UNAVAILABLE` usually means Volcengine secrets or parser runtime wiring are missing
- `PARSER_LIMIT_REACHED` means the anonymous daily quota is exhausted
- use the returned `requestId` to inspect the matching Worker log entry

### 5. Confirm D1 bindings

1. Verify parser usage changes after one successful parser call.
2. Verify credits balance exists and updates after recharge.

Expected result:

- `PARSER_USAGE_DB` persists parser usage changes
- `CREDITS_DB` persists starter grants, recharge orders, and balance updates

### 6. Confirm recharge intent creation

1. While signed in, create a `1 RMB` recharge through the UI or `POST /api/recharge-intent`.
2. Confirm the response returns a ZPAY checkout URL.

Expected result:

- response status `200`
- `provider = "zpay"`
- `checkoutUrl` points to `https://zpayz.cn/submit.php?...`

If it fails:

- JSON errors include `requestId`
- search logs by `requestId` and `path = /api/recharge-intent`

### 7. Confirm a valid ZPAY callback

1. Complete a real recharge payment.
2. Verify ZPAY reaches `/api/payment-webhook`.
3. Verify the callback response body is plain-text `success`.
4. Verify the balance increases by `amountRmb * 30`.

Expected result:

- webhook response body is exactly `success`
- credits increase once
- Worker logs contain a `/api/payment-webhook` entry with `status = 200`

### 8. Confirm replay-safe callback behavior

1. Replay the same valid callback payload once.
2. Confirm the webhook still returns plain-text `success`.
3. Confirm the balance does not increase a second time.

Expected result:

- second callback is accepted as idempotent
- the order stays completed
- no duplicate credit grant is applied

### 9. Confirm requestId-based support lookup

1. Trigger one intentional JSON error on `/api/parse`, `/api/credits/balance`, or `/api/recharge-intent`.
2. Copy the `requestId` from the response body.
3. Search Worker logs for that `requestId`.

Expected result:

- exactly one matching request log is easy to find
- the log includes `path`, `status`, `durationMs`, and route metadata

### 10. Confirm feedback email delivery

1. Open the deployed app at `APP_URL`.
2. Submit one test feedback entry through the settings modal with `contactAgreed = true` and a reachable email address.
3. Confirm the frontend receives the existing success contract from `/api/feedback`.
4. Confirm an email reaches `FEEDBACK_TO_EMAIL`.
5. Open the delivered email and confirm `reply-to` points at the reporting user email.

Expected result:

- `/api/feedback` returns success without falling back to log-only local mode
- the operator inbox receives the feedback email
- replying to the email targets the reporting user instead of the sender mailbox

## Fast Diagnosis Map

- `Platform API` UI missing:
  check the build-time Clerk publishable key
- `Platform API` sign-in works but balance fails:
  check `CLERK_SECRET_KEY` or `CLERK_JWT_KEY`
- parser usage route works but parsing fails:
  check `VOLCENGINE_ACCESS_KEY_ID` and `VOLCENGINE_SECRET_ACCESS_KEY`
- recharge intent works but payment does not settle:
  check `APP_URL`, `ZPAY_PID`, `ZPAY_PKEY`, and public reachability of `/api/payment-webhook`
- feedback route succeeds but no email arrives:
  check `NOTIFICATION_PROVIDER`, `RESEND_API_KEY`, `NOTIFICATION_FROM_EMAIL`, verified sender-domain status in Resend, and `FEEDBACK_TO_EMAIL`
- balance changed incorrectly after callback replay:
  inspect `CREDITS_DB` order state and webhook request logs

## Related Docs

- [../architecture/deployment.md](../architecture/deployment.md)
- [../backend/api-design.md](../backend/api-design.md)
- [china-operational-fit-report.md](china-operational-fit-report.md)
