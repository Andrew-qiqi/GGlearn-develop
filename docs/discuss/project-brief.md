# Project Brief

## Metadata

- Status: Draft
- Last Updated: 2026-04-04
- Owner: Agent-authored, user-approved
- Impacts Existing Plans: Yes
- Change Summary: Consolidates recent product, infrastructure, and commercialization discussion into a clean handoff brief for the next planning window, including BYOK-first public launch, minimal Cloudflare-first sequencing, and platform-funded parser bootstrap.

## Project Identity

- Project Name: SlideTutor AI
- One-line Summary: A PDF-centered AI learning assistant that explains slides, preserves teaching continuity, supports follow-up tutoring, and helps students study efficiently.
- Core Value: Turn static course slides into a guided, teacher-like learning experience with strong explanation quality, continuity, and controllable model/provider choices.

## Why This Project Exists

Slide decks and lecture PDFs are often compressed, context-light, and hard to study alone. Students do not just need OCR or summarization; they need a guided explanation flow that reconstructs reasoning, points to the right visual regions, and carries context across pages.

The product is now moving from pre-launch experimentation toward real user-facing deployment. That shifts the main problem from "can the tutoring interaction work?" to "how do we ship this sustainably for real users, especially China-based users, without making the stack too expensive or too hard to maintain?"

## True Needs

- Keep the core tutoring experience high quality, especially `explain`, `distill`, follow-up, and context continuity.
- Preserve strong output stability across both strong and weaker models through structured JSON contracts.
- Support two user modes in the future:
  - user-supplied model API keys
  - platform-hosted API service for paid users
- Support real deployment beyond the current "developer personally funds the APIs" stage.
- Reduce long-term dependency on Vercel Hobby assumptions for a future commercial product.
- Revisit document parsing as a first-class product cost center rather than an invisible background service.
- Optimize for China-based users and China-based operator convenience wherever practical.

## Non-Goals

- Do not immediately implement the entire commercialization stack in one step.
- Do not treat Cloudflare migration, BYOK, login, payments, hosted APIs, and parser-provider abstraction as one coding phase.
- Do not lock into one permanent document parser provider today.
- Do not design a pricing model that assumes infinite platform-funded inference or parsing.
- Do not rush to fix every model-specific edge case before the product-direction decisions are settled.

## Constraints

- The product is still pre-launch and currently uses the user's own API keys/services.
- Teaching prompt intent, tone, and explanation quality must remain protected; output-contract work must not dilute the educational experience.
- Most future users are expected to be in China, and the operator is also China-based.
- Platform choice must consider not only static site reachability, but also model API reliability, streaming behavior, document parsing cost, and operational simplicity.
- Current document parsing relies on Azure free quota, which is already exhausted; this cannot be treated as a durable default.
- Future architecture should prefer maintainable provider abstractions over many hard-coded one-off branches.

## Locked Decisions

- Structured JSON output is now the long-term direction for major generation tasks; prompt-only formatting control is not sufficient.
- Gemini remains a separate provider adapter; OpenAI-compatible providers should be unified as much as possible.
- Long-term architecture should support user-supplied OpenAI-compatible APIs rather than building custom logic per provider.
- Cloudflare is the leading candidate for future deployment, replacing reliance on Vercel Hobby for a commercial product path.
- Future product strategy should support two parallel modes:
  - BYOK for user-supplied model APIs
  - platform-provided APIs for paid users
- The first public user-facing version should be BYOK-first.
- In the first public version, platform-hosted APIs are a secondary track, not the primary product entry.
- Early BYOK is fully free; there is no launch-stage service fee for user-supplied model APIs.
- Preferred sequencing is to complete a minimal Cloudflare migration before building the next commercialization-critical features on top.
- A donation entry can exist, but it is secondary to a clear core usage model.
- Document parsing should be reconsidered as a configurable provider layer, not assumed to be permanently platform-funded through Azure.
- In the early user-acquisition stage, document parsing can be platform-funded by default, with later guardrails and provider abstraction work.
- Early public parsing should stay platform-funded by default, but users who do not bring their own parser access should be subject to explicit request limits.
- Current website/domain reachability is not the main China-user blocker; the user reports the domain is already managed via Cloudflare and can be accessed in China without VPN.
- Gemini availability is a real China-user risk because China-based requests can fail with `User location is not supported for the API use`.

## Agent Discretion

- The exact migration path from Vercel to Cloudflare is not yet locked beyond the minimal-first direction.
- The exact split between free features, one-time purchase, subscription, or usage-based charging is not yet locked.
- Whether BYOK should include a small service fee in early versions is still open.
- The exact launch mode for platform-hosted APIs is still open: waitlist, invite-only, or direct paid rollout.
- The long-term parser evolution path is still open: platform-funded baseline, hybrid model, or optional parser BYOK.
- The exact provider shortlist for China-friendly document parsing can be refined later.
- Model-specific capability matrices, such as which Gemini models support which thinking controls, can be decided in later implementation phases.

## Success Conditions

- The product has a realistic deployment path that does not depend on non-commercial assumptions.
- There is a clear architecture for supporting both BYOK and paid platform-hosted usage.
- Document parsing has a sustainable provider strategy and no longer depends on accidental free quota.
- China-based users can access and use the service with acceptable reliability and low setup friction.
- The system remains maintainable: provider differences are abstracted cleanly and educational quality remains protected.

## Initial Phase Direction

- Phase A: Minimal Cloudflare migration
  - Move the public deployment path off the current Vercel-first assumption.
  - Prioritize the minimum viable production topology rather than a full infrastructure rewrite.
  - Verify core frontend delivery and `/api/generate` streaming reliability on the new base.
- Phase B: BYOK-first usage architecture
  - Define how user-supplied model APIs are configured, validated, and routed.
  - Prioritize OpenAI-compatible BYOK while preserving Gemini as a separate adapter.
  - Decide whether early BYOK carries a small service fee.
- Phase C: Parser bootstrap and provider abstraction
  - Keep parser cost platform-funded in the early growth stage.
  - Add guardrails so parser cost is observable and controllable.
  - Abstract parser providers so Azure is no longer an invisible default dependency.
- Phase D: Account system and platform-hosted APIs
  - Add login and user identity infrastructure.
  - Define how paid platform-hosted model access coexists with BYOK.
  - Decide the first launch mode for hosted APIs: waitlist, invite-only, or direct paid release.
- Phase E: China-user operations
  - Re-check actual bottlenecks for users in China: model providers, parser providers, streaming, login, payments, and deployment path.
  - Only then choose whether deeper mainland-specific infrastructure work is necessary.

## Canonical References

- `AGENTS.md`
- `docs/frontend/architecture.md`
- `docs/frontend/data-flow.md`
- `docs/changelog/CHANGELOG_TECH.md`
- `SlideTutor-AI/src/config/models.ts`
- `SlideTutor-AI/api/generate.ts`
- `SlideTutor-AI/api/lib/structuredOutputConfig.ts`

## Open Questions

- How small can the first Cloudflare migration be while still de-risking later BYOK, login, and payment work?
- Should document parsing stay fully platform-managed through the first public version, or expose limited controls earlier?
- What is the minimum viable login/payment system for China-based users if platform-hosted APIs are introduced?
- How should model capability differences be represented long-term, especially for Gemini variants with inconsistent thinking-control support?

## Next Step

Use this brief as the upstream reference for the next product-direction discussion window or for `/gsd:new-project` if the project is being re-entered through a formal planning workflow.
