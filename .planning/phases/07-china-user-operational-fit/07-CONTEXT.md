# Phase 07: China-User Operational Fit - Context

**Gathered:** 2026-04-06
**Status:** Ready for planning
**Source:** PRD Express Path (`docs/discuss/phases/07-china-user-operational-fit-brief.md`)

<domain>
## Phase Boundary

Phase 07 is not a fresh architecture exploration. The Cloudflare-first runtime, BYOK-first access path, Volcengine parser cutover, Clerk auth boundary, and ZPAY recharge flow already exist in the repo and have reached initial live validation.

This phase should validate and harden the real product/operations chain for China-based users and operators by:

- making China-user access assumptions explicit without turning the UI into a recommendation engine
- normalizing user-facing failure modes where the current stack has geography/provider-specific friction
- giving the operator request-level visibility across parser, auth, credits, and recharge routes
- codifying the evidence needed before revisiting parser BYOK, `MinerU`, or deeper mainland-specific infrastructure

This phase must not become a new provider expansion phase, a billing/admin phase, or a mainland-infra rewrite.

</domain>

<decisions>
## Implementation Decisions

### Locked product posture
- `BYOK-first` remains the primary public path.
- `Platform API` remains a second path that requires login and credits, and it does not replace `BYOK`.
- Users still choose their own model/provider; the product should not auto-switch providers on their behalf.
- Platform-managed parser remains `Volcengine`.
- Recharge remains `ZPAY`.

### Locked commercial rules to preserve
- New users receive a one-time non-expiring `10 credits` starter grant.
- `1 RMB = 30 credits`, minimum recharge `1 RMB`, and credits do not expire.
- Hosted actions deduct only after success.
- No user-facing billing-history or recharge-history pages should be introduced here.

### Operational-fit rules for this phase
- This phase prioritizes validation, observability, documentation, and small hardening over new product surface area.
- New work must directly reduce verified China-user or China-operator friction.
- Do not add parser BYOK, `MinerU`, extra payment providers, or a billing back office in this phase.
- Do not bury users in provider recommendations or noisy pricing detail that weakens the current UX.

### China-specific access truths
- `CN-03` is a real requirement: China-based requests must not assume Gemini API availability.
- Region/provider-blocked failures should become explicit and actionable instead of ambiguous or hidden behind fallback behavior.
- Alternative paths such as OpenAI-compatible `My API` and `Platform API` should stay available, but they must remain user-controlled choices.

### Operator-side truths
- Operator success now depends on one chain: Cloudflare deploy + Clerk frontend/public key + Clerk Worker verification secret + Volcengine parser + D1 bindings + ZPAY callback path.
- Request-level observability must exist for credits, recharge, payment webhook, parse, and parser-usage routes, not only `/api/generate`.
- Docs must make the `APP_URL` / `notify_url` / `return_url` coupling explicit for ZPAY.

### the agent's Discretion
- The exact request-log fields and helper shape may be chosen pragmatically if they stay low-noise and do not leak secrets.
- The exact docs split between user guide, operator checklist, and operational report may be chosen pragmatically if future support work can find the right artifact quickly.
- Small UX copy additions are acceptable only if they directly explain real access friction and stay intentionally light.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Product scope and locked decisions
- `docs/discuss/project-brief.md` - Project-level direction, locked commercial rules, and China-focused product posture.
- `docs/discuss/phases/07-china-user-operational-fit-brief.md` - Phase-specific scope, constraints, and deferred decisions.
- `docs/discuss/phases/06-login-hosted-access-and-credit-brief.md` - Confirms what is already shipped and should now be treated as baseline.

### Planning truth
- `.planning/ROADMAP.md` - Phase 07 goal, requirements, and plan list.
- `.planning/REQUIREMENTS.md` - `CN-01`, `CN-02`, `CN-03`.
- `.planning/STATE.md` - Current state and active concerns.

### User-path and runtime docs
- `docs/backend/api-design.md` - Current route contracts for hosted access, balance, recharge, parser, and webhook flows.
- `docs/architecture/deployment.md` - Runtime topology, env requirements, and deployment footguns.
- `docs/frontend/architecture.md` - Current frontend access-mode boundary and Clerk bootstrap behavior.
- `docs/frontend/data-flow.md` - Existing request flows for hosted access, parser, and recharge.

### Existing implementation anchors
- `SlideTutor-AI/src/lib/auth/clerk.tsx` - Frontend Clerk boundary and degraded bootstrap behavior.
- `SlideTutor-AI/src/components/settings/PlatformApiSection.tsx` - Current `My API` / `Platform API` UX boundary.
- `SlideTutor-AI/src/components/SettingsModal.tsx` - Current AI settings surface.
- `SlideTutor-AI/src/lib/api/apiClient.ts` - Shared client-side access and hosted route calls.
- `SlideTutor-AI/api/lib/generateService.ts` - Provider execution path and hosted action lifecycle.
- `SlideTutor-AI/src/worker/lib/auth.ts` - Worker-side Clerk token verification.
- `SlideTutor-AI/src/worker/lib/observability.ts` - Current request-log helper.
- `SlideTutor-AI/src/worker/routes/generate.ts` - Existing high-signal route logging model.
- `SlideTutor-AI/src/worker/routes/parse.ts` - Current parse route with no log parity yet.
- `SlideTutor-AI/src/worker/routes/parser-usage.ts` - Current parser usage route with no log parity yet.
- `SlideTutor-AI/src/worker/routes/credits-balance.ts` - Current hosted balance route.
- `SlideTutor-AI/src/worker/routes/recharge-intent.ts` - Current recharge intent route.
- `SlideTutor-AI/src/worker/routes/payment-webhook.ts` - Current ZPAY callback route.
- `SlideTutor-AI/api/lib/platformAccess/service.ts` - Hosted credits and recharge application logic.
- `SlideTutor-AI/api/lib/platformAccess/zpayAdapter.ts` - ZPAY checkout and webhook contract.

### Supporting evidence
- `tmp_files/volcengine_document_parse_intellgence/` - Local snapshot of Volcengine parser docs and pricing references.
- `tmp_files/zpay/` - Local snapshot of ZPAY merchant docs.

</canonical_refs>

<specifics>
## Specific Ideas

- A real `1 RMB` ZPAY recharge has already been validated by the user and credited `30`, so payment is no longer theoretical; observability and supportability are now more important than new payment surface area.
- `/api/generate` already emits request-level logs with `requestId`, `path`, `status`, `durationMs`, `providerId`, and `task`, but credits/recharge/parser routes do not yet match that standard.
- The current UI already keeps `My API` and `Platform API` explicit. The next improvement should stay small: make region/provider limitations explicit without turning the settings surface into a recommendation wall.
- Any new parser BYOK or `MinerU` work should be blocked on evidence captured by a repo-native operational-fit report.

</specifics>

<deferred>
## Deferred Ideas

- parser BYOK implementation
- `MinerU` adapter implementation
- additional hosted provider presets or payment providers
- billing dashboard, recharge history UI, fraud/risk systems
- mainland-specific infra rebuilds

</deferred>

---

*Phase: 07-china-user-operational-fit*
*Context gathered: 2026-04-06 via PRD Express Path*
