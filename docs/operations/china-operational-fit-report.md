# China Operational Fit Report

Last updated: 2026-04-06

Use this template after running real checks against the China-user and China-operator flow.

## Evidence Rules

- `observed evidence`: something directly seen in the product, logs, deploy output, or payment result
- `inference`: a conclusion drawn from the evidence
- `must-fix now`: an issue that blocks real usage, supportability, or trustworthy charging
- `safe to defer`: an issue worth tracking, but not urgent enough to expand product scope today

Do not merge these categories together. Keep the difference explicit.

## Report Template

### Environment and Date

- Date:
- App URL:
- Commit / deployment:
- Region / network conditions:
- Operator:

### Test Conditions

- Account type:
- Access path tested:
- Payment path tested:
- Parser path tested:
- Notes about VPN, network, or provider restrictions:

### My API Observations

#### observed evidence

- 

#### inference

- 

### Platform API Observations

#### observed evidence

- 

#### inference

- 

### Parser Observations

#### observed evidence

- 

#### inference

- 

### Recharge and Webhook Observations

#### observed evidence

- 

#### inference

- 

### Support and Log Observations

#### observed evidence

- 

#### inference

- 

## must-fix now

Rank items from highest operator or user impact to lowest.

1. 
2. 
3. 

## safe to defer

- 
- 
- 

## Decision Gate

| Topic | Current decision | Evidence required to reopen | Status |
| --- | --- | --- | --- |
| parser BYOK | Defer | Repeated observed evidence that the platform parser is the main blocker for real users or support | Deferred |
| MinerU | Defer | Observed evidence that parser BYOK is necessary and MinerU is the best next candidate for China-based users | Deferred |
| Additional hosted presets | Defer | Observed evidence that current `My API` and `Platform API` choices still leave a major provider gap | Deferred |
| Deeper mainland infrastructure | Defer | Observed evidence that Cloudflare + Clerk + Volcengine + ZPAY cannot be operated reliably enough with the current scope | Deferred |

## Final Call

- Keep current path:
- Expand now:
- Re-check later on:
- Owner for follow-up:

## Related Docs

- [china-operator-checklist.md](china-operator-checklist.md)
- [../user_guide/access-modes.md](../user_guide/access-modes.md)
