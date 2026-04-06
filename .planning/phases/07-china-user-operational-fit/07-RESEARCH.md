# Phase 07: China-User Operational Fit - Research

**Researched:** 2026-04-06
**Domain:** Validating and hardening the existing China-user / China-operator chain after live Clerk + Volcengine + ZPAY rollout
**Confidence:** MEDIUM-HIGH

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

#### Locked product posture
- `BYOK-first` remains the primary public path.
- `Platform API` remains a second path that requires login and credits, and it does not replace `BYOK`.
- Users still choose their own model/provider; the product should not auto-switch providers on their behalf.
- Platform-managed parser remains `Volcengine`.
- Recharge remains `ZPAY`.

#### Locked commercial rules to preserve
- New users receive a one-time non-expiring `10 credits` starter grant.
- `1 RMB = 30 credits`, minimum recharge `1 RMB`, and credits do not expire.
- Hosted actions deduct only after success.
- No user-facing billing-history or recharge-history pages should be introduced here.

#### Operational-fit rules for this phase
- This phase prioritizes validation, observability, documentation, and small hardening over new product surface area.
- New work must directly reduce verified China-user or China-operator friction.
- Do not add parser BYOK, `MinerU`, extra payment providers, or a billing back office in this phase.
- Do not bury users in provider recommendations or noisy pricing detail that weakens the current UX.

#### China-specific access truths
- `CN-03` is a real requirement: China-based requests must not assume Gemini API availability.
- Region/provider-blocked failures should become explicit and actionable instead of ambiguous or hidden behind fallback behavior.
- Alternative paths such as OpenAI-compatible `My API` and `Platform API` should stay available, but they must remain user-controlled choices.

#### Operator-side truths
- Operator success now depends on one chain: Cloudflare deploy + Clerk frontend/public key + Clerk Worker verification secret + Volcengine parser + D1 bindings + ZPAY callback path.
- Request-level observability must exist for credits, recharge, payment webhook, parse, and parser-usage routes, not only `/api/generate`.
- Docs must make the `APP_URL` / `notify_url` / `return_url` coupling explicit for ZPAY.

### Claude's Discretion

Copied from `CONTEXT.md`.

- The exact request-log fields and helper shape may be chosen pragmatically if they stay low-noise and do not leak secrets.
- The exact docs split between user guide, operator checklist, and operational report may be chosen pragmatically if future support work can find the right artifact quickly.
- Small UX copy additions are acceptable only if they directly explain real access friction and stay intentionally light.

### Deferred Ideas (OUT OF SCOPE)
- parser BYOK implementation
- `MinerU` adapter implementation
- additional hosted provider presets or payment providers
- billing dashboard, recharge history UI, fraud/risk systems
- mainland-specific infra rebuilds

</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| CN-01 | Re-check actual reliability bottlenecks for China-based users across model access, parser access, and streaming. | Normalize provider/region failures, keep parser path explicit, and create repeatable user-path validation artifacts. |
| CN-02 | Re-check operator-side realities for payments, support, and deployment convenience before deeper infra commitments. | Add request-level observability parity, smoke docs, and an operator checklist for the Clerk + Volcengine + ZPAY + D1 chain. |
| CN-03 | China-based requests must avoid assuming Gemini API availability. | Do not auto-fallback, but make unsupported-region failures explicit and document alternative user-controlled access paths. |

</phase_requirements>

## Summary

Phase 07 should not plan a new China-specific platform. The mainline stack is already materially in place:

- Cloudflare Worker is the canonical public runtime.
- Clerk is already wired for frontend sign-in and Worker-side bearer-token verification.
- Volcengine is already the live platform parser provider.
- ZPAY is already the live recharge provider.
- A real `1 RMB` payment has already been verified by the user and increased the balance by `30` credits.

That changes the planning question. The phase is no longer "which China stack should we choose?" It is now "what real friction remains in the current stack, and what small hardening/doc work is justified before bigger commitments?"

The research points to three concrete gaps:

1. **User-path truth gap**
   The repo still exposes Gemini as a first-class `My API` option, but the project requirements already say China-based requests must not assume Gemini availability. There is no normalized, low-noise product contract for a region-blocked provider failure yet.

2. **Operator visibility gap**
   `/api/generate` already emits request-level logs with request IDs and route metadata. Credits, recharge, payment webhook, parse, and parser-usage routes do not yet provide the same debugging quality. That makes China-specific support and payment troubleshooting harder than it needs to be.

3. **Runbook / decision-artifact gap**
   Current docs describe env vars and route contracts, but there is no single operator checklist that combines Clerk build/runtime requirements, APP_URL/ZPAY coupling, D1 bindings, parser smoke, and live recharge verification. There is also no repo-native report template that captures the evidence needed before reopening parser BYOK / `MinerU`.

**Primary recommendation:** Plan Phase 07 as three focused plans:

- normalize China-user access failures and add low-noise access guidance
- add operator-grade observability plus a China-operator smoke checklist
- create the operational-fit report and decision gate that blocks premature parser BYOK / `MinerU` work

## Standard Stack

### Core

| Library / Service | Purpose | Why Standard |
|-------------------|---------|--------------|
| Existing Cloudflare Worker runtime | Public app + APIs | Already the canonical deployment base; Phase 07 should not reopen runtime migration. |
| Clerk React + Clerk Backend SDK | Platform sign-in and bearer verification | Already shipped; Phase 07 needs clearer operational guidance, not a new auth provider. |
| Volcengine OCRPdf path | Platform-managed parser | Already live and cheaper for current product assumptions. |
| ZPAY redirect + webhook flow | Hosted recharge | Already live enough to validate; the current need is observability and ops repeatability. |
| Existing request observer + `requestId` pattern | Route diagnostics | Already proven on `/api/generate`; should be extended instead of reinvented. |

### Supporting

| Library / Tool | Purpose | When to Use |
|----------------|---------|-------------|
| `vitest` | unit/component tests | Provider-error normalization, UI copy, client error handling |
| `@cloudflare/vitest-pool-workers` | Worker route tests | credits/recharge/parser route observability and request-id coverage |
| repo docs (`docs/*`) | operational knowledge base | operator checklist, user access guide, operational-fit report |

## Architecture Patterns

### Pattern 1: Explicit user choice, explicit failure

**What:** Keep `My API` vs `Platform API` as explicit user-controlled choices, but make geography/provider failure states explicit when they happen.

**Why:** This respects the user's earlier decision that the product should not silently recommend or auto-switch providers, while still satisfying `CN-03`.

**Planning implication:** Normalize region-blocked provider failures into stable error codes/messages and add one concise help path instead of intrusive provider recommendation UX.

### Pattern 2: Observability parity across operational routes

**What:** Apply the same request-id / structured-log discipline used by `/api/generate` to the routes that now matter for real operator support: `/api/parse`, `/api/parser-usage`, `/api/credits/balance`, `/api/recharge-intent`, and `/api/payment-webhook`.

**Why:** For China-user operational fit, supportability matters as much as functionality. Hidden failures on recharge/auth/parser routes create more operator pain than one more provider option would solve.

**Planning implication:** Extend or reuse the existing observer, add low-sensitivity route metadata, and include `requestId` in JSON error responses wherever the response shape allows it.

### Pattern 3: Evidence artifact over chat memory

**What:** Capture operational-fit findings in repo-native docs: a checklist for operators and a report template for decision logging.

**Why:** If future parser BYOK or `MinerU` discussion depends on memory or chat transcripts, the repo will drift back into "intuition-based roadmap" mode.

**Planning implication:** Create docs that distinguish observed evidence, inferred conclusions, and explicitly deferred items.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| China-specific provider friction | automatic provider switching or hidden fallback | explicit error normalization plus user-controlled alternative paths | Hidden fallback would violate the current product posture and make debugging harder. |
| Operator support docs | ad hoc comments in code or one-off chat instructions | modular docs under `docs/user_guide` and `docs/operations` | Repo-native docs survive sessions and support future operators. |
| New billing/admin surface | recharge-history pages or back office | current D1 ledger + operator checklist | Scope does not justify new billing product surface. |
| New parser/provider expansion | parser BYOK or `MinerU` implementation | operational-fit report + decision gate | Evidence must come before expansion. |

## Runtime State Inventory

| Category | Items Found | Action Required |
|----------|-------------|-----------------|
| Live payment proof | User confirmed a real `1 RMB` ZPAY payment and resulting `+30 credits`. | Treat recharge as live baseline; prioritize supportability over mock-flow assumptions. |
| Auth config split | Frontend depends on `VITE_CLERK_PUBLISHABLE_KEY` or `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`; Worker verification depends on `CLERK_SECRET_KEY` or `CLERK_JWT_KEY`. | Document and smoke-test the build-time/runtime split together. |
| Parser runtime | Volcengine is the live parser path. | Keep parser platform-managed for now; do not reopen parser BYOK without evidence. |
| Route observability | `/api/generate` logs request metadata; other operational routes mostly do not. | Extend observability parity. |
| Route contracts | API docs already describe recharge/webhook/balance behavior. | Sync docs with requestId/logging additions and operator checklist. |

## Common Pitfalls

### Pitfall 1: Treating Gemini as universally available

**What goes wrong:** China-based users hit provider errors that look like generic request failures, not a region/provider availability issue.

**Why it happens:** Gemini stays a valid global provider in the product model, but the phase requirements already acknowledge that availability cannot be assumed from China.

**How to avoid:** Normalize the failure, keep the user's explicit choice intact, and document the alternative paths without auto-switching.

### Pitfall 2: Debugging recharge/auth issues without request IDs

**What goes wrong:** The operator cannot correlate a user's report to logs across `/api/credits/balance`, `/api/recharge-intent`, and `/api/payment-webhook`.

**How to avoid:** Add request-id parity and structured route logs for the hosted-access support chain.

### Pitfall 3: Forgetting Clerk build-time vs runtime config split

**What goes wrong:** A deploy can have valid Worker secrets but still boot into `My API`-only degraded mode because the public Clerk publishable key was missing from the build environment.

**How to avoid:** Keep the split explicit in docs and smoke steps.

### Pitfall 4: Reopening parser BYOK too early

**What goes wrong:** The roadmap expands into `MinerU` or parser BYOK work before the current platform-managed parser is shown to be a sustained blocker.

**How to avoid:** Require an operational-fit report section that explicitly justifies any parser BYOK escalation with observed evidence.

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | `vitest` + `@cloudflare/vitest-pool-workers` |
| App config | `SlideTutor-AI/vite.config.ts` |
| Worker config | `SlideTutor-AI/vitest.worker.config.ts` |
| Quick run command | `npm test -- api/lib/generateService.platform.test.ts src/lib/api/apiClient.test.ts src/components/SettingsModal.test.tsx src/components/settings/PlatformApiSection.test.tsx` |
| Worker run command | `npm run test:workers -- test/workers/credits-balance.worker.test.ts test/workers/recharge.worker.test.ts test/workers/security-observability.worker.test.ts` |
| Full phase command | `npm test -- api/lib/generateService.platform.test.ts src/lib/api/apiClient.test.ts src/components/SettingsModal.test.tsx src/components/settings/PlatformApiSection.test.tsx && npm run test:workers -- test/workers/credits-balance.worker.test.ts test/workers/recharge.worker.test.ts test/workers/security-observability.worker.test.ts && npm run lint` |

### Wave 0 Gaps

- [ ] `SlideTutor-AI/src/components/settings/PlatformApiSection.test.tsx` does not exist yet and should be created for low-noise access-guidance coverage.
- [ ] `SlideTutor-AI/test/workers/security-observability.worker.test.ts` should gain coverage for route logging beyond `/api/generate`.

## Sources

### Primary (HIGH confidence)

- Local repo docs and implementation anchors listed in `07-CONTEXT.md`
- Official Google AI available-regions page: `https://ai.google.dev/gemini-api/docs/available-regions`
  - Used to verify that Gemini API availability is region-scoped and to confirm that China mainland should not be assumed as supported. This is an inference from the published supported-regions list.

### Project-Verified (HIGH confidence)

- User-verified live result on 2026-04-06: successful `1 RMB` ZPAY payment increased credits by `30`.

### Secondary (MEDIUM confidence)

- Local copies of Volcengine and ZPAY docs under `tmp_files/volcengine_document_parse_intellgence/` and `tmp_files/zpay/`

## Metadata

- Standard stack confidence: HIGH
- Operational-gap diagnosis confidence: HIGH
- China-user provider-availability conclusion confidence: MEDIUM-HIGH, because it relies on the official Gemini regions page plus project/user validation rather than a broader provider survey
- Research valid until: 2026-05-06 unless the provider/payment stack changes materially
