# Phase 06 Product Design: Platform API, Login, and Credits

## Metadata

- Status: Draft
- Date: 2026-04-05
- Scope: Product design only
- Source: Consolidated from discussion based on `docs/discuss/project-brief.md` and `docs/discuss/phases/06-login-hosted-access-and-credit-brief.md`

## Pain Moment

SlideTutor is moving from a pre-launch experiment into a real product. The core tutoring experience is already becoming clearer, but the hosted-access path still lacks a clean account and payment boundary.

Without that boundary, several problems appear at once:

- users who do not want to configure their own API keys still cannot smoothly use the product
- hosted model access has no clear paywall or balance model
- monetization remains vague, which makes real validation hard
- users can become suspicious if charging rules feel complicated, hidden, or unfair

The key risk is overbuilding a SaaS billing system before validating whether users will actually pay for a clean hosted path.

## Magic Moment

The learner uploads a PDF, chooses `Platform API`, logs in once, sees a small starter balance, and can immediately analyze slides without learning anything about providers, parser setup, or billing architecture.

If they later run low on balance, the product does not trap them. It gives a clear choice:

- buy more credits
- switch to `My API`

This keeps the learning flow primary while making hosted access sustainable.

## Product Razor Results

| Proposed Feature | Serves Magic Moment? | Chemistry? | Discoverable? | Missed if Cut? | 10x? | Verdict |
|---|---|---|---|---|---|---|
| Browser-local `My API` with no login | Yes | Yes | Yes | Yes | Yes | Keep |
| Hosted `Platform API` behind login | Yes | Yes | Yes | Yes | Yes | Keep |
| One-time starter credits for new users | Yes | Yes | Yes | Yes | Yes | Keep |
| Flexible recharge instead of subscription-first | Yes | Yes | Yes | Yes | Yes | Keep |
| Fixed per-action pricing | Yes | Yes | Yes | Yes | Yes | Keep |
| Complex billing dashboard | No | No | No | No | No | Cut |
| Recharge / deduction history in UI | No | No | Low value | No | No | Cut |
| Credit packs and bundle merchandising | No | No | Yes | No | No | Cut |
| Weekly free credits | Weak | Weak | Yes | No | No | Cut |
| Invite-only hosted access | No | Weak | No | No | No | Cut |

## Final Product Decisions

### 1. Two Parallel Modes

SlideTutor keeps two parallel access modes:

- `My API`
- `Platform API`

They are presented as neutral choices. The product does not recommend one over the other.

### 2. `My API` Boundary

`My API` remains the low-friction path for users who want to use their own model access.

Rules:

- no login required
- no platform credits deducted
- user-supplied model credentials stay in the local browser
- if parser capability is unavailable, the product may still run with `Low accuracy`

### 3. `Platform API` Boundary

`Platform API` is the hosted product path.

Rules:

- switching to `Platform API` immediately requires login
- hosted usage consumes credits
- users can purchase credits directly
- no invite-only gate for the initial version

The login gate happens at mode switch, not later during a hidden second checkpoint.

### 4. New User Starter Credit

Each newly registered user receives a one-time starter balance:

- `10 credits`

This is not a weekly reset and not a recurring free allowance.

### 5. Recharge Model

Recharge is intentionally simple:

- flexible recharge, not subscription-first
- no predefined credit packs
- user enters an RMB amount directly
- minimum recharge amount: `1 RMB`
- exchange rate: `1 RMB = 30 credits`

Credits do not expire.

This applies to:

- starter credits
- purchased credits

### 6. User-Facing Prices

The initial public pricing is action-based:

- `Analyze = 3 credits`
- `Follow-up = 1 credit`
- `Quiz generation = 1 credit`
- `Quiz answer analysis = 1 credit`

The design intent is that `1 RMB` covers about `10` slide analyses.

### 7. `Analyze` Is One Product Action

`Analyze` is shown to users as one single product action.

The UI should not expose internal sub-steps such as:

- parse
- explain
- distill

Internally, these may remain separate execution stages, but the hosted pricing model treats them as one user action.

### 8. Success-Only Charging

Hosted charging must follow a strict success-only rule:

- credits are checked before execution
- credits are deducted only after a successful result
- failed actions do not deduct credits

For `Analyze`, success means:

- parse succeeded
- explain succeeded
- distill succeeded

If any sub-step fails, the whole `Analyze` action is treated as failed and no credits are deducted.

### 9. No Paid Degraded Analyze

`Platform API` should not sell degraded analyze results as a normal paid result.

That means:

- if hosted `Analyze` cannot complete all required sub-steps, it is not considered a successful charged action
- `Low accuracy` is primarily a `My API` concern, especially when the user lacks parser capability

### 10. Insufficient Credit Behavior

If the user does not have enough credits for the requested hosted action:

- keep the current mode as `Platform API`
- block the action before execution
- show a clear insufficient-credit prompt

The prompt should offer exactly two meaningful exits:

- `Buy Credits`
- `Switch to My API`

The product should not silently auto-switch modes on the user's behalf.

## Recommended Experience Flow

### Flow A: Hosted First-Time Use

1. User opens settings or access controls.
2. User switches to `Platform API`.
3. If not logged in, login is required immediately.
4. After login, starter credits are visible.
5. User runs `Analyze`.
6. If the action fully succeeds, `3 credits` are deducted.

### Flow B: Insufficient Credit

1. User remains in `Platform API`.
2. User starts an action that costs more than the current balance.
3. The product blocks execution before the request is run.
4. A clear prompt appears with:
   - required credits
   - current balance
   - `Buy Credits`
   - `Switch to My API`

### Flow C: Recharge

1. User opens the `Platform API` area in settings.
2. User clicks `Buy Credits`.
3. User enters an RMB amount.
4. UI shows converted credits in real time using `1 RMB = 30 credits`.
5. User proceeds to payment.
6. Purchased credits are added to balance and do not expire.

## Settings Information Architecture

The main hosted-access UI should live inside the settings panel, not in the core study surface.

The `Platform API` area in settings should include only the essentials:

- current credit balance
- `Buy Credits` primary entry
- compact action pricing summary

The following should not be shown in the initial version:

- recharge history
- deduction history
- detailed billing dashboard
- complex financial terminology

All financial records should still be saved on the backend for audit and support purposes.

## Backend Truths Required By Product

Even though billing details stay mostly invisible in the UI, the backend must preserve durable accounting truth for:

- starter credit grant
- recharge events
- successful hosted deductions
- failed or blocked hosted actions

The product UI remains simple, but backend records must remain auditable.

## What To Cut

- weekly free credit reset
- subscription-first packaging
- bundle-based credit merchandising
- front-end billing history
- deduction history page
- auto-switching users away from `Platform API` when balance is low
- charging for degraded hosted analyze results

## Visual Direction

The billing layer should feel subordinate to the learning product, not like a trading panel.

Design guidance:

- keep credits UI inside settings
- use quiet hierarchy, not aggressive promotion
- make `Buy Credits` easy to find but not omnipresent
- avoid dashboard aesthetics, graphs, or accounting-heavy terminology
- keep insufficient-credit prompts direct and calm

The product should feel like a tutor with an access boundary, not a finance tool with a PDF feature.

## Open Items For Later Planning

These items are intentionally deferred to implementation planning rather than locked here:

- exact payment provider integration details
- exact balance schema and ledger schema
- exact copywriting for insufficient-credit prompts
- payment failure and reconciliation handling
- refund and manual adjustment operations

## Implementation Guardrails

- Do not break mature teaching logic.
- Do not store user BYOK secrets server-side by default.
- Do not mix hosted degraded results into normal paid success semantics.
- Do not turn settings into a billing center.
- Do not force login on the `My API` path.
