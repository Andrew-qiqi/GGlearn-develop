# China Operational Fit Report

Last updated: 2026-04-07

This report records the current evidence from the live China-facing validation round after the Phase 07 hardening work landed.

## Evidence Rules

- `observed evidence`: something directly seen in the product, logs, deploy output, or payment result
- `inference`: a conclusion drawn from the evidence
- `must-fix now`: an issue that blocks real usage, supportability, or trustworthy charging
- `safe to defer`: an issue worth tracking, but not urgent enough to expand product scope today

Do not merge these categories together. Keep the difference explicit.

## Environment and Date

- Date: 2026-04-07
- App URL: `https://www.slidetutor-ai.com`
- Commit / deployment: live Cloudflare production deployment validated after the Phase 07 hardening round; exact deployed hash was not captured during the operator session
- Region / network conditions: China-user and China-operator validation flow
- Operator: user-performed live validation with agent-assisted investigation and documentation

## Test Conditions

- Account type: signed-in production account
- Access path tested: `Platform API` live path; `My API` assumptions re-checked through the product contract and Phase 07 error handling rather than a new mainland Gemini success case
- Payment path tested: live `ZPAY` recharge with `1 RMB`
- Parser path tested: live platform-managed Volcengine parser through hosted analyze and direct route investigation
- Notes about VPN, network, or provider restrictions: Volcengine document parsing service did not appear to become available immediately after manual enablement on the Volcengine side; parsing succeeded after waiting for service activation to settle

## My API Observations

### observed evidence

- The product still keeps `My API` and `Platform API` as explicit user choices instead of auto-switching providers.
- Phase 07 added a stable user-facing handling path for Gemini region-blocked failures so the product no longer assumes China-based Gemini availability.
- This validation round did not produce evidence that parser BYOK is required to unlock the current `My API` path.

### inference

- `CN-03` is satisfied by explicit user choice plus explicit provider-unavailable handling, not by adding more providers right now.
- The next `My API` expansion decision should be evidence-driven instead of based on intuition about which provider might be easier to obtain.

## Platform API Observations

### observed evidence

- Clerk sign-in works on the live production deployment.
- Hosted balance and credits flow remained available after the current hardening round.
- The product successfully supports the current second path of signed-in hosted usage without displacing `BYOK`.

### inference

- `Platform API` is a viable second access path for China-based users in the current launch scope.
- There is no current evidence that the product needs a user-facing billing dashboard or additional payment providers to keep this path usable.

## Parser Observations

### observed evidence

- A hosted analyze request previously failed with parser degradation and a request-level traceable error.
- Direct production checks during the investigation showed `/api/parser-usage` responding normally and `/api/parse` returning `parseMode = normal`.
- The user confirmed that parsing worked later without code or Cloudflare binding changes, after the Volcengine document parsing service had more time to become active.

### inference

- The initial parser failure was more consistent with provider-side service activation lag than with a persistent product-side parser misconfiguration.
- Current evidence does not justify moving parser BYOK or `MinerU` to the front of the roadmap.

## Recharge and Webhook Observations

### observed evidence

- Live `ZPAY` recharge succeeded with `1 RMB`.
- The hosted balance increased by `30` credits after payment.
- Login and payment were both confirmed working in production during the current validation round.

### inference

- The current `ZPAY` integration is sufficient for the present launch scope.
- There is no evidence in this round that the product should expand to more payment providers now.

## Support and Log Observations

### observed evidence

- Phase 07 added request-level observability and `requestId` parity across parser, credits, recharge, and payment routes.
- The hosted parser investigation became materially faster after these route-level diagnostics existed.
- The Clerk frontend build key vs Worker runtime secret split remains a real operator footgun, but it is now documented in the checklist and deployment docs.

### inference

- The operator support surface is now good enough to continue validating the current product without deeper infrastructure work.
- The next support improvement should be incremental ops discipline, not a major architecture expansion.

## must-fix now

No unresolved must-fix-now blockers remained after the current hardening and re-test cycle.

## safe to defer

- `parser BYOK`
- `MinerU`
- additional hosted provider presets
- additional payment providers
- deeper mainland-specific infrastructure

## Decision Gate

| Topic | Current decision | Evidence required to reopen | Status |
| --- | --- | --- | --- |
| parser BYOK | Defer | Repeated observed evidence that the platform parser is the main blocker for real users or support | Deferred |
| MinerU | Defer | Observed evidence that parser BYOK is necessary and MinerU is the best next candidate for China-based users | Deferred |
| Additional hosted presets | Defer | Observed evidence that current `My API` and `Platform API` choices still leave a major provider gap | Deferred |
| Deeper mainland infrastructure | Defer | Observed evidence that Cloudflare + Clerk + Volcengine + ZPAY cannot be operated reliably enough with the current scope | Deferred |

## Final Call

- Keep current path: Yes. Keep `BYOK-first` plus the current `Platform API`, Volcengine parser, and `ZPAY` stack.
- Expand now: No parser BYOK, no `MinerU`, no extra payment providers, and no deeper mainland-specific infrastructure in the next immediate step.
- Re-check later on: Only after repeated live evidence shows sustained parser blockage, provider acquisition friction, or operator pain that the current stack cannot absorb.
- Owner for follow-up: next milestone planning after more live usage evidence accumulates

## Related Docs

- [china-operator-checklist.md](china-operator-checklist.md)
- [../user_guide/access-modes.md](../user_guide/access-modes.md)
