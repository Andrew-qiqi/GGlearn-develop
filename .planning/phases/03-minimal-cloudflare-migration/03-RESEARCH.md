# Phase 03: Minimal Cloudflare Migration - Research

**Researched:** 2026-04-04
**Domain:** Cloudflare Workers migration for a Vite SPA plus critical-path API routes
**Confidence:** MEDIUM

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

### Platform direction
- Cloudflare is the intended primary runtime direction for the next public product stage.
- The first migration is allowed to be minimal, but it must still de-risk the later BYOK-first launch.
- Short-term dual-platform transition is acceptable only as a temporary bridge, not as a steady-state operating model.

### Critical-path migration scope
- The first migration should prioritize moving the full first-release user critical path to Cloudflare.
- The critical path includes the public frontend path, `/api/generate`, `/api/get-token`, `/api/parse`, and any API required for a normal first-release PDF tutoring workflow.
- Non-critical-path endpoints may be deferred only if they do not affect the first-release learning flow.

### Runtime adaptation boundary
- `/api/generate` may receive a small, intentional server/runtime adaptation refactor during migration.
- That refactor exists only to make the runtime and streaming path compatible with Cloudflare.
- Do not use this phase to modify mature teaching business logic.
- Do not change teaching prompt intent, explanation behavior, structured artifact contracts, or frontend consumption contracts in this phase.

### Support-layer migration scope
- Support layers required by the current public runtime should be migrated with the first Cloudflare move rather than left as long-lived Vercel/Cloudflare split responsibilities.
- This includes the currently required authentication, rate limiting, environment-variable handling, proxy/IP treatment, logging/observability, and existing notification or email support that the current public runtime depends on.
- This does not include future login, billing, subscriptions, or hosted-model product systems.

### Scope protection
- Do not bundle Phase 04 BYOK-first implementation into this phase.
- Do not bundle Phase 05 parser abstraction into this phase except where parser behavior must continue to work for the critical path.
- Do not quietly expand this phase into a general platform rewrite.

### Claude's Discretion
- The exact Cloudflare runtime topology may be chosen during planning if it respects the locked boundaries above.
- The exact implementation pattern for streaming compatibility on Cloudflare may be chosen during research and planning.
- The specific internal adapter-layer refactor shape is left to the planner as long as it stays within runtime/platform concerns.

### Deferred Ideas (OUT OF SCOPE)
- Login and identity systems.
- Billing, subscriptions, or credits.
- Hosted-model user product flows.
- Full parser-provider abstraction.
- China-mainland-specific deep infrastructure changes beyond what is needed to preserve first-release viability.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| DEP-01 | Establish a minimal Cloudflare-oriented deployment path for the public app. | Single-Worker topology, Workers Static Assets SPA routing, Vite plugin, migration guide, runtime state inventory. |
| DEP-02 | Preserve core streaming generation behavior after the migration. | Workers Streams guidance, OpenAI SDK Worker support, adapter-boundary recommendation for `/api/generate`. |
| DEP-03 | Preserve request protection, environment-variable handling, and operational observability during the migration. | Workers secrets, Node compatibility/process.env behavior, Rate Limit binding semantics, Workers Logs/Traces guidance. |
</phase_requirements>

## Project Constraints (from CLAUDE.md)

- No `CLAUDE.md` file exists at the repo root, so there are no additional CLAUDE-specific constraints to propagate.

## Repo Workflow Constraints (from AGENTS.md)

- Use `gsd-brief-handoff` only for narrow pre-GSD workflow setup.
- Do not use `gsd-brief-handoff` for pure GSD discuss/planning flows like this one.

## Summary

The minimal viable Phase 03 plan is a single Cloudflare Worker deployment that serves the built Vite SPA and the first-release critical-path APIs together: `/api/generate`, `/api/get-token`, and `/api/parse`. That is the smallest move that actually removes the Vercel-first base instead of recreating it behind a split frontend/API topology. Cloudflare’s current Workers platform explicitly supports static assets, SPA fallback routing, Vite-native development, Worker-side logs/traces, secrets, and streaming responses, so the migration does not require changing the teaching pipeline itself.

The main implementation risk is not prompt logic or frontend contracts. It is the current Node/Express server shape. The repo currently couples runtime concerns inside its API handlers: Express routing, `express-rate-limit`, `nodemailer`, top-level `validateEnv()`, proxy/IP extraction, and direct `res.write()` streaming in [SlideTutor-AI/api/generate.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/api/generate.ts). A good plan should preserve `buildPrompt`, structured-output config, provider routing, and frontend artifact parsing, while replacing only the HTTP/runtime shell with Worker-native adapters.

The strongest architecture choice for this phase is: keep one deployment unit, keep one public base URL, keep provider/task contracts unchanged, and do only thin runtime refactors. The one place where a small internal cleanup is worth planning now is environment access. Today the app hard-fails at startup if `GEMINI_API_KEY` or Azure parse credentials are absent, which is workable for the current server but a bad foundation for the Phase 04 BYOK path. That should become route-scoped capability validation, not a teaching-logic rewrite.

**Primary recommendation:** Plan Phase 03 as a single Cloudflare Worker cutover with static assets + `/api/generate` + `/api/get-token` + `/api/parse`, using only thin Worker adapters and no teaching/business-logic changes.

## Standard Stack

### Core

| Library / Feature | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| `wrangler` | `4.80.0` (published 2026-04-02) | Local dev, deploy, secrets, observability config, Worker config | Official Cloudflare CLI for Workers projects. |
| `@cloudflare/vite-plugin` | `1.31.0` (published 2026-04-02) | Run the existing Vite app in `workerd`, build static assets, support SPA/full-stack Worker dev | Official Vite-native path for Cloudflare Workers. |
| Workers Static Assets | Current platform feature | Serve the built SPA and Worker code as one deployable unit | Eliminates separate static hosting vs API runtime split. |
| Workers SPA routing (`assets.not_found_handling`) | Current platform feature | Client-side route fallback to `index.html` | Official, first-class SPA behavior. |
| `nodejs_compat` compatibility flag | Current platform feature | Preserve `process.env`, `Buffer`, `crypto`, and Node-oriented SDK assumptions where supported | Minimizes code churn for the current codebase. |

### Supporting

| Library / Feature | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `@cloudflare/vitest-pool-workers` | `0.14.1` (published 2026-04-02) | Run tests inside the Workers runtime | Add Worker-runtime tests for routing, streaming, and env behavior. |
| `openai` | `6.33.0` (published 2026-03-25) | Existing OpenAI-compatible provider client for `qwen` and `doubao` | Keep for current provider path; official SDK supports Cloudflare Workers. |
| `@google/genai` | `1.48.0` (published 2026-04-01) | Existing Gemini client | Keep initially; verify in Worker runtime during implementation. |
| Workers Rate Limiting binding | Current platform feature | Replace `express-rate-limit` for app-level abuse control | Use for `/api/get-token` and `/api/generate` when code-level limits are needed. |
| Workers Logs + Traces | Current platform feature | Replace ad-hoc console-only observability | Enable for migration verification and production monitoring. |
| Cloudflare Web Analytics | Current platform feature | Replace `@vercel/analytics/react` if analytics is kept in scope | Use if frontend analytics remains desired after cutover. |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Single Worker with static assets | Cloudflare Pages + Pages Functions | Also valid, but adds an unnecessary product split for a phase whose goal is a single critical-path runtime base. |
| Plain Worker route dispatch | `hono` | Hono is reasonable if route code gets messy, but it is optional overhead for only a few endpoints. |
| Workers Rate Limiting binding | Durable Objects counter store | DOs are better for globally coordinated quotas, but are unnecessary for this phase’s simple abuse-control limits. |
| Cloudflare Web Analytics | Keep `@vercel/analytics/react` temporarily | Leaves a Vercel-specific dependency in the frontend after the platform cutover. |
| SMTP/Nodemailer in Worker | HTTP-based mail API or explicit deferral | SMTP is a poor fit for a minimal Worker migration; if feedback email is not critical, do not let it block the phase. |

**Installation:**
```bash
npm install -D wrangler @cloudflare/vite-plugin @cloudflare/workers-types @cloudflare/vitest-pool-workers
```

**Version verification:** verified with `npm view` on 2026-04-04.

## Architecture Patterns

### Recommended Project Structure

```text
SlideTutor-AI/
├── src/
│   ├── worker/
│   │   ├── index.ts              # Worker fetch entrypoint and route dispatch
│   │   ├── routes/
│   │   │   ├── generate.ts       # Worker adapter for /api/generate
│   │   │   ├── get-token.ts      # Worker adapter for /api/get-token
│   │   │   ├── parse.ts          # Worker adapter for /api/parse
│   │   │   └── feedback.ts       # Optional; migrate only if kept in scope
│   │   └── lib/
│   │       ├── env.ts            # Route-scoped secret/capability checks
│   │       ├── ip.ts             # Cloudflare-aware IP/origin helpers
│   │       ├── rate-limit.ts     # Binding adapter
│   │       └── streams.ts        # Async-iterator -> ReadableStream helpers
│   ├── ...existing frontend files
├── vite.config.ts
├── wrangler.jsonc
└── test/
    └── workers/                  # Worker-runtime tests
```

### Pattern 1: Single Worker, Single Public Base

**What:** Serve the SPA assets and `/api/*` endpoints from one Worker deployment, not a split frontend/API topology.

**When to use:** Exactly this phase. The goal is to stop growing Phase 04 on top of a Vercel-first base.

**Example:**
```jsonc
// Source: https://developers.cloudflare.com/workers/static-assets/routing/single-page-application/
{
  "name": "slidetutor-ai",
  "compatibility_date": "2026-04-03",
  "assets": {
    "directory": "./dist/",
    "not_found_handling": "single-page-application"
  }
}
```

### Pattern 2: Thin Worker Adapters Around Existing Domain Logic

**What:** Extract runtime-independent logic out of Express handlers, but keep current prompt generation, structured schemas, provider families, and frontend response contracts unchanged.

**When to use:** For `/api/generate`, `/api/get-token`, and `/api/parse`.

**Example:**
```ts
// Source basis:
// https://developers.cloudflare.com/workers/runtime-apis/streams/
// repo-specific adaptation pattern from SlideTutor-AI/api/generate.ts
export default {
  async fetch(request, env, ctx): Promise<Response> {
    const url = new URL(request.url);
    if (url.pathname === "/api/get-token") return handleGetToken(request, env);
    if (url.pathname === "/api/parse") return handleParse(request, env);
    if (url.pathname === "/api/generate") return handleGenerate(request, env, ctx);
    return new Response("Not found", { status: 404 });
  },
};
```

### Pattern 3: Web-Standard Streaming, Not Express `res.write()`

**What:** Move provider output to Worker `ReadableStream` / `Response` streaming rather than Node response mutation.

**When to use:** `/api/generate` streaming path and any future tokenized streaming path.

**Example:**
```ts
// Source: https://developers.cloudflare.com/workers/runtime-apis/streams/
export default {
  async fetch(request): Promise<Response> {
    const response = await fetch(request);
    return new Response(response.body, response);
  },
};
```

### Pattern 4: Route-Scoped Capability Validation

**What:** Validate secrets only for the route/provider that needs them. Do not keep one startup gate that requires every provider secret at deploy time.

**When to use:** Immediately for Worker migration, because the current `validateEnv()` in [env.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/api/lib/env.ts) hard-requires Gemini and Azure secrets globally.

**Example:**
```ts
// Source basis:
// https://developers.cloudflare.com/workers/configuration/secrets/
// repo-specific adaptation pattern from SlideTutor-AI/api/lib/env.ts
function requireSecret(env: Env, key: keyof Env) {
  const value = env[key];
  if (!value) throw new Error(`${String(key)} is not configured`);
  return value;
}
```

### Anti-Patterns to Avoid

- **Whole-app Express port:** Do not try to force `express()` and middleware composition directly into Workers. Move to fetch-handler routing.
- **Split critical path across Vercel and Cloudflare:** That preserves the wrong architectural base and creates future cutover debt.
- **Migration-driven prompt/business logic edits:** `buildPrompt`, structured output contracts, and frontend artifact parsing are explicitly protected.
- **Global env hard-fail at Worker startup:** This blocks future BYOK evolution and increases deployment fragility.
- **Assuming SPA fallback is harmless for API paths:** With SPA asset routing, navigation to `/api/*` can serve HTML unless routing is configured carefully.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| SPA fallback routing | Manual `/index.html` fallback logic everywhere | Workers Static Assets + `assets.not_found_handling = "single-page-application"` | Official routing behavior already exists. |
| Worker runtime tests | Homegrown fetch/runtime mocks | `@cloudflare/vitest-pool-workers` | Official Worker-runtime test harness for Vitest. |
| App-level request throttling | Ported `express-rate-limit` middleware semantics | Workers Rate Limiting binding | Fits Worker runtime and avoids Express baggage. |
| Secret injection | Checked-in `.env` or copied Vercel-style plaintext vars | Worker secrets + local `.dev.vars` / `.env` | Official secret flow; safer and deployable. |
| Operational logging | Custom log sink before basic visibility exists | Workers Logs and optional Traces | Official observability with low setup cost. |
| SMTP mail path | Force `nodemailer`/raw SMTP into the first Worker cutover | HTTP mail provider or explicit deferral | SMTP is not a clean minimal path in Workers, and port 25 is prohibited. |

**Key insight:** The phase should adopt Cloudflare’s native deploy, routing, secrets, rate-limit, and observability primitives. Rebuilding Vercel/Express behavior by hand is the main way this phase becomes larger than necessary.

## Runtime State Inventory

| Category | Items Found | Action Required |
|----------|-------------|------------------|
| Stored data | None found that is deployment-provider specific. IndexedDB page state and artifacts are client-side learning state, not Vercel-specific. Verified via [db.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/src/lib/db.ts), [usePdfLibrary.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/src/hooks/usePdfLibrary.ts), and related tests. | None. No data migration required for app state. |
| Live service config | Current deploy secrets and project settings are effectively dashboard state, not repo state. The repo contains [SlideTutor-AI/vercel.json](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/vercel.json) and a Vercel analytics import in [main.tsx](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/src/main.tsx), but no `.vercel/project.json` was found. | Cloudflare-side project creation, secret creation, custom-domain binding, observability enablement, and analytics replacement are required. This is config migration, not data migration. |
| OS-registered state | None found in repo or local project files. No systemd, launchd, PM2, Task Scheduler, or similar registrations were detected. | None verified. |
| Secrets/env vars | Current runtime depends on `GEMINI_API_KEY`, `DOUBAO_API_KEY`, `QWEN_API_KEY`, `AZURE_DOCUMENT_INTELLIGENCE_ENDPOINT`, `AZURE_DOCUMENT_INTELLIGENCE_KEY`, `APP_URL`, `SHARED_APP_URL`, `ENABLE_TOKEN_AUTH`, `API_TOKEN_SECRET`, `SMTP_*`, and `FEEDBACK_EMAIL` in [generate.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/api/generate.ts), [get-token.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/api/get-token.ts), and [.env.example](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/.env.example). | Code edit: move access to Worker secrets / env bindings. Config migration: recreate required secrets in Cloudflare. Prefer route-scoped validation over current global startup validation. |
| Build artifacts | Existing `SlideTutor-AI/dist` build output exists locally. Vercel routing assumptions live in [SlideTutor-AI/vercel.json](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/vercel.json). No `.wrangler/` state exists yet, and `wrangler` is not installed globally. | Code/config edit: add `wrangler.jsonc`, Worker entrypoint, and Cloudflare-specific build/dev commands. Local tooling setup: add `wrangler` as a dev dependency and use `npx wrangler`. |

## Common Pitfalls

### Pitfall 1: Moving only the frontend shell

**What goes wrong:** The SPA moves to Cloudflare, but `/api/generate` or `/api/get-token` stays on Vercel for “later”.

**Why it happens:** It looks smaller in the short term.

**How to avoid:** Treat frontend + `/api/generate` + `/api/get-token` + `/api/parse` as one migration unit.

**Warning signs:** New env vars, CORS rules, or proxy glue appear between platforms.

### Pitfall 2: Preserving Express instead of preserving business logic

**What goes wrong:** Too much time gets spent on emulating `express`, middleware ordering, and `res.write()`.

**Why it happens:** The current server shape makes the HTTP shell look more important than it is.

**How to avoid:** Preserve domain logic and contracts; replace only the HTTP/runtime shell.

**Warning signs:** The plan contains “Express compatibility” work items but no explicit route/service extraction boundary.

### Pitfall 3: Assuming rate limits mean the same thing on Cloudflare

**What goes wrong:** The team expects exact global request accounting.

**Why it happens:** Current `express-rate-limit` reads as if limits are precise and centralized.

**How to avoid:** Treat Worker rate limits as abuse control, not billing/accounting. Document locality and eventual consistency.

**Warning signs:** Requirements or tests expect a strict global count across all Cloudflare locations.

### Pitfall 4: Accidentally serving `index.html` for API navigation paths

**What goes wrong:** Browser navigation to `/api/...` hits SPA asset fallback instead of Worker route handling.

**Why it happens:** SPA routing is configured, but the navigation behavior is not understood.

**How to avoid:** Plan route tests for both client fetches and direct browser navigations to `/api/*`.

**Warning signs:** API smoke tests use `fetch()` only and never test a navigated URL.

### Pitfall 5: Keeping startup-wide env validation

**What goes wrong:** The whole Worker fails because a provider secret is missing, even when that provider is not in use.

**Why it happens:** Current code calls `validateEnv()` at module load.

**How to avoid:** Validate secrets per route/provider capability.

**Warning signs:** Phase 03 tasks still mention one “global env validator” file as the enforcement point.

### Pitfall 6: Letting SMTP email block the cutover

**What goes wrong:** `nodemailer`/SMTP becomes a migration blocker even though the tutoring critical path is elsewhere.

**Why it happens:** Feedback and alert email currently live inside the same file as generation logic.

**How to avoid:** Either move `/api/feedback` explicitly into this phase with an HTTP mail API, or defer it as a non-critical endpoint. Do not let it silently stay on Vercel.

**Warning signs:** The plan has unresolved SMTP compatibility work on the critical path.

## Code Examples

Verified patterns from official sources:

### Cloudflare SPA Asset Configuration

```jsonc
// Source:
// https://developers.cloudflare.com/workers/static-assets/routing/single-page-application/
{
  "name": "slidetutor-ai",
  "compatibility_date": "2026-04-03",
  "assets": {
    "directory": "./dist/",
    "not_found_handling": "single-page-application"
  }
}
```

### Enable Worker Logs

```jsonc
// Source:
// https://developers.cloudflare.com/workers/observability/logs/workers-logs/
{
  "observability": {
    "enabled": true
  }
}
```

### Enable Traces With Sampling

```jsonc
// Source:
// https://developers.cloudflare.com/workers/observability/traces/
{
  "observability": {
    "logs": {
      "enabled": true,
      "head_sampling_rate": 0.6
    },
    "traces": {
      "enabled": true,
      "head_sampling_rate": 0.05
    }
  }
}
```

### Worker Rate Limit Check

```ts
// Source:
// https://developers.cloudflare.com/workers/runtime-apis/bindings/rate-limit/
const { success } = await env.MY_RATE_LIMITER.limit({ key: customerId });
if (!success) {
  return new Response("Too Many Requests", { status: 429 });
}
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Vercel static hosting + serverless assumptions | Workers Static Assets + single Worker deploy | Cloudflare Workers static-assets path is current and documented in 2025-2026 docs | One deployment can now host SPA assets and API routes together. |
| Build locally, hope runtime matches deploy target | Cloudflare Vite plugin runs code in `workerd` during development | Vite plugin GA announced 2025-04-08 | Existing Vite apps can move to Cloudflare without a separate dev/runtime mental model. |
| Node middleware-centric throttling | Worker-native Rate Limiting binding or WAF rules | Current Workers/WAF docs | Abuse control should be platform-native, not Express-native. |
| Console-only edge visibility | Workers Logs + Traces in Wrangler config | Current Workers observability docs | DEP-03 can be satisfied without introducing a third-party logging stack. |

**Deprecated/outdated:**
- `vercel.json` as the deployment control point for this product direction: Phase 03 should replace it with `wrangler.jsonc` for the new primary runtime.
- `@vercel/analytics/react` as the only analytics path: it preserves a Vercel dependency after the cutover.

## Open Questions

1. **Should `/api/feedback` migrate in Phase 03 or be explicitly deferred as non-critical?**
   - What we know: The frontend actively uses `/api/feedback`, but the tutoring critical path does not depend on it.
   - What's unclear: Whether the team considers feedback submission part of the “current public runtime depends on it” support-layer scope.
   - Recommendation: Decide this explicitly in planning. Do not leave it as an accidental Vercel holdout.

2. **Can `@google/genai` remain unchanged inside the Worker runtime?**
   - What we know: The SDK documents both server-side and browser initialization, which strongly suggests Web/Worker compatibility, and the repo already uses it successfully in a fetch-oriented style.
   - What's unclear: There is no explicit primary-source claim found here saying “Cloudflare Workers supported” the way OpenAI’s SDK does.
   - Recommendation: Treat as MEDIUM confidence and include a small runtime spike or first implementation task to verify Gemini streaming under Workers.

3. **Are current rate-limit thresholds acceptable under Cloudflare’s locality semantics?**
   - What we know: Worker binding limits are local to a Cloudflare location and intentionally permissive/eventually consistent.
   - What's unclear: Whether the existing `10/min`, `100/day`, and `20/min` thresholds are still the desired behavior under global edge routing.
   - Recommendation: Preserve them initially for abuse control, then tune after live traffic observation.

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Node.js | Vite build, tests, Wrangler | ✓ | `v24.14.0` | — |
| npm | Package install, scripts, `npx wrangler` | ✓ | `11.9.0` | — |
| Wrangler CLI | Local Worker dev, deploy, secret commands | ✗ global | — | Add as dev dependency and use `npx wrangler` |
| Cloudflare account with Workers + zone access | Actual deploy, domain binding, secrets, observability | Unknown locally | — | Human/dashboard access required |
| Existing Vercel project settings access | Extract current build/output/env before cutover | Unknown locally | — | Human/dashboard access required |

**Missing dependencies with no fallback:**
- Cloudflare account access is required for actual deployment, secret provisioning, and custom-domain cutover.

**Missing dependencies with fallback:**
- Global `wrangler` installation is missing, but using the local package via `npx wrangler` is sufficient.

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | `vitest` `^4.1.0` in repo; current npm release `4.1.2` |
| Config file | [SlideTutor-AI/vite.config.ts](/c:/Users/hoo/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI/vite.config.ts) |
| Quick run command | `npm test -- api/security.test.ts` |
| Full suite command | `npm test` |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| DEP-01 | SPA assets and critical-path APIs resolve from one Cloudflare deployment path | worker integration | `npm test -- test/workers/spa-routing.worker.test.ts` | ❌ Wave 0 |
| DEP-02 | `/api/generate` preserves streamed output shape for explain/distill flows | worker integration | `npm test -- test/workers/generate-stream.worker.test.ts` | ❌ Wave 0 |
| DEP-03 | token auth, env handling, and rate-limit/observability wiring still behave after migration | worker integration | `npm test -- test/workers/security-observability.worker.test.ts` | ❌ Wave 0 |

### Sampling Rate

- **Per task commit:** `npm test -- api/security.test.ts`
- **Per wave merge:** `npm test`
- **Phase gate:** Full suite green plus Worker smoke checks before `/gsd:verify-work`

### Wave 0 Gaps

- [ ] `SlideTutor-AI/test/workers/spa-routing.worker.test.ts` — covers DEP-01
- [ ] `SlideTutor-AI/test/workers/generate-stream.worker.test.ts` — covers DEP-02
- [ ] `SlideTutor-AI/test/workers/security-observability.worker.test.ts` — covers DEP-03
- [ ] Framework install: `npm install -D @cloudflare/vitest-pool-workers` — Worker-runtime tests

## Sources

### Primary (HIGH confidence)

- Cloudflare Workers Vite plugin: https://developers.cloudflare.com/workers/vite-plugin/ - Vite-native Worker development and deployment model
- Cloudflare Workers Static Assets: https://developers.cloudflare.com/workers/static-assets/ - Single deploy unit for SPA assets + Worker code
- Cloudflare Workers SPA routing: https://developers.cloudflare.com/workers/static-assets/routing/single-page-application/ - `assets.directory`, `not_found_handling`, navigation behavior
- Cloudflare Workers migration guide from Vercel: https://developers.cloudflare.com/workers/static-assets/migration-guides/vercel-to-workers/ - current migration steps and custom-domain notes
- Cloudflare Workers secrets: https://developers.cloudflare.com/workers/configuration/secrets/ - Worker secrets, `.dev.vars` / `.env`, secret commands
- Cloudflare Workers Node.js compatibility: https://developers.cloudflare.com/workers/runtime-apis/nodejs/ - `nodejs_compat` behavior
- Cloudflare Workers `process.env`: https://developers.cloudflare.com/workers/runtime-apis/nodejs/process/ - `process.env` population rules with Node compatibility
- Cloudflare Workers Streams: https://developers.cloudflare.com/workers/runtime-apis/streams/ - response streaming patterns
- Cloudflare Workers Rate Limiting binding: https://developers.cloudflare.com/workers/runtime-apis/bindings/rate-limit/ - locality, accuracy, monitoring semantics
- Cloudflare Workers Logs: https://developers.cloudflare.com/workers/observability/logs/workers-logs/ - enabling logs and log limits
- Cloudflare Workers Traces: https://developers.cloudflare.com/workers/observability/traces/ - tracing config and pricing state
- Cloudflare Workers Vitest integration: https://developers.cloudflare.com/workers/testing/vitest-integration/ - official Worker-runtime testing approach
- Cloudflare Workers testing overview: https://developers.cloudflare.com/workers/testing/ - testing comparison matrix
- Cloudflare Workers TCP sockets: https://developers.cloudflare.com/workers/runtime-apis/tcp-sockets/ - SMTP-related limitations
- OpenAI Node SDK README: https://github.com/openai/openai-node - confirms Cloudflare Workers support
- Google Gen AI SDK README: https://github.com/googleapis/js-genai - browser/server initialization model and current SDK guidance

### Secondary (MEDIUM confidence)

- Cloudflare Vite plugin GA changelog: https://developers.cloudflare.com/changelog/post/2025-04-08-vite-plugin/ - timing and maturity signal for adopting the plugin
- Cloudflare Web Analytics: https://developers.cloudflare.com/web-analytics/get-started/ - frontend analytics replacement path
- Cloudflare Pages Web Analytics guide: https://developers.cloudflare.com/pages/how-to/web-analytics/ - one-click analytics on Cloudflare-hosted frontends

### Tertiary (LOW confidence)

- None. Low-confidence claims were avoided rather than promoted.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - current official Cloudflare docs plus npm registry verification
- Architecture: MEDIUM - official platform capabilities are clear, but the exact adapter shape is repo-specific and Gemini-on-Workers still needs runtime verification
- Pitfalls: HIGH - based on official platform caveats plus direct inspection of current server/frontend coupling

**Research date:** 2026-04-04
**Valid until:** 2026-05-04
