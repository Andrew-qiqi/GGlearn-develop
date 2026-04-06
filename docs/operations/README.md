# Operations Docs

Last updated: 2026-04-06

Use this module for operator-facing runbooks and evidence artifacts that support the live China-user chain.

## Documents

- [china-operator-checklist.md](china-operator-checklist.md)
  Use this before or during live deploy verification. It is the step-by-step smoke checklist for Cloudflare, Clerk, Volcengine, D1, and ZPAY.
- [china-operational-fit-report.md](china-operational-fit-report.md)
  Use this after running real checks. It is the evidence template for recording `My API`, `Platform API`, parser, recharge, and support findings.

## Which Document To Use

- Use the checklist when you need to confirm that the product is wired correctly right now.
- Use the report when you need to decide whether a problem is a must-fix now issue, an inference that needs more evidence, or something safe to defer.

## Decision Gate

Do not reopen `parser BYOK`, `MinerU`, extra hosted presets, or deeper mainland infrastructure based on memory alone. Record observed evidence first, then write the decision into the operational-fit report.
