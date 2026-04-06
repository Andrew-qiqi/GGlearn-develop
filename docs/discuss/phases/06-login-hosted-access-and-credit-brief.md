# Phase 06 Brief: Login, Hosted Access, and Credit Billing

## Metadata

- Status: Completed
- Phase: 06
- Related Roadmap Entry: `.planning/ROADMAP.md`
- Last Updated: 2026-04-06
- Owner: Agent-authored, user-approved
- Impacts Existing Plans: Yes
- Change Summary: Phase 06 is no longer a blank future commercialization phase. Clerk-backed hosted access, starter credits, success-only charging, and ZPAY recharge are now part of the live mainline.

## Objective

Complete the first hosted-access product path for SlideTutor without turning the app into a full billing SaaS.

The intended shape was:

- `BYOK` remains available and does not require login
- `Platform API` exists as a second, login-required path
- new users receive one-time starter credits
- hosted actions charge only on success
- recharge uses `ZPAY`

## Current State

The following are now shipped:

- explicit `My API` / `Platform API` mode switching
- Clerk-based frontend sign-in flow and Worker-side session validation
- D1-backed credit account, ledger, and recharge-order persistence
- one-time `10` starter credits
- fixed `1 RMB = 30 credits`
- success-only hosted charging
- hosted `Analyze = 3`, `Follow-up = 1`, `Quiz generation = 1`, `Quiz answer analysis = 1`
- `Analyze` charges only after `parse + explain + distill` all succeed
- signed `ZPAY` redirect checkout creation
- `ZPAY` webhook signature verification, amount validation, and idempotent recharge completion

## Locked Product Decisions

- `Platform API` requires login
- credits do not expire
- minimum recharge is `1 RMB`
- recharge stays free-form instead of bundle-based
- no user-facing billing-history or recharge-history pages
- users can choose models themselves
- parser BYOK remains out of scope for this phase

## Outcome

Phase 06 closed the hosted-access baseline without breaking the BYOK-first public product direction.

What is now true:

- hosted access and BYOK coexist cleanly
- recharge no longer depends on a mock payment adapter
- payment replay does not double-credit the user
- the user-facing settings surface remains intentionally light

## Deferred Items

- subscription billing
- bundle pricing
- billing dashboard / recharge history UI
- parser BYOK and provider selection UI
- advanced anti-fraud systems

## Next Step

Shift active planning to Phase 07: validate the real operational fit for China-based users and operators before expanding infrastructure scope further.
