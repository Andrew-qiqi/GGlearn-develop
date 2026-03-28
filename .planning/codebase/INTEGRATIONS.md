# External Integrations

**Analysis Date:** 2025-05-22

## APIs & External Services

**LLM Providers:**
- **Google Gemini** - Primary AI model for content generation and moderation.
  - SDK: `@google/genai`
  - Auth: `GEMINI_API_KEY`
- **Doubao (Bytedance AI)** - Alternative AI model provider.
  - SDK: `openai` (Compatible mode)
  - Auth: `DOUBAO_API_KEY`
- **Qwen (Alibaba AI)** - Alternative AI model provider.
  - SDK: `openai` (Compatible mode)
  - Auth: `QWEN_API_KEY`

**Document Intelligence:**
- **Azure Document Intelligence** - Advanced PDF layout analysis for ground truth slide content mapping.
  - API Version: `2023-07-31`
  - Endpoint: `AZURE_DOCUMENT_INTELLIGENCE_ENDPOINT`
  - Auth: `AZURE_DOCUMENT_INTELLIGENCE_KEY`

**Email Delivery:**
- **SMTP Service** - Used for feedback submission and security alerts.
  - Client: `nodemailer`
  - Auth: `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASS`, `SMTP_SECURE`, `SMTP_FROM`

## Data Storage

**Databases:**
- **SQLite** - (Local/Embedded)
  - Client: `better-sqlite3`
  - Purpose: Likely for local development or background data management (detected in `package.json`).

**File Storage:**
- **Client-side Memory/localStorage** - Used for persistent tutor settings (focus mode).
- **In-memory store** - (Backend) Rate-limit tracking and malicious intent alerting logic in `api/generate.ts`.

**Caching:**
- **In-memory (Backend)** - Basic request counting and IP rate-limiting in `api/generate.ts`.

## Authentication & Identity

**Auth Provider:**
- **Custom Security Layer** - `securityGuard` middleware in `api/generate.ts`.
  - Implementation: Domain check (Origin/Referer), Rate limiting (in-memory), and AI-powered moderation.

## Monitoring & Observability

**Error Tracking:**
- **None** - (Standard logging to console in `api/generate.ts`).

**Analytics:**
- **Vercel Analytics** - Used for tracking application usage.
  - Package: `@vercel/analytics`

**Logs:**
- **Console Logs** - Extensive logging in `api/generate.ts` and `server.ts` for tracking AI requests and security events.

## CI/CD & Deployment

**Hosting:**
- **Vercel** - Primary hosting platform for the frontend and serverless API.
- **Tencent Cloud EdgeOne** - Used for edge routing and API rewrites (see `edgeone.json`).

**CI Pipeline:**
- **None** - (Not explicitly detected in codebase, likely using Vercel default CI/CD).

## Environment Configuration

**Required env vars:**
- `GEMINI_API_KEY`: Required for core AI features.
- `DOUBAO_API_KEY`: For Doubao fallback.
- `QWEN_API_KEY`: For Qwen fallback.
- `AZURE_DOCUMENT_INTELLIGENCE_KEY`: Required for layout analysis.
- `SMTP_HOST`, `SMTP_USER`, `SMTP_PASS`: For feedback and security alerts.
- `APP_URL`, `SHARED_APP_URL`: Used for domain verification in security middleware.

**Secrets location:**
- `.env` file for local development.
- Vercel/EdgeOne dashboard for production.

## Webhooks & Callbacks

**Incoming:**
- `POST /api/generate`: Primary AI task execution.
- `POST /api/parse`: Slide layout analysis.
- `POST /api/feedback`: Feedback submission.

**Outgoing:**
- **AI Requests**: Outbound to Google Gemini, Doubao, or Qwen APIs.
- **Azure Analysis**: Outbound to Azure Document Intelligence endpoints.
- **Emails**: Outbound to SMTP server via `nodemailer`.

---

*Integration audit: 2025-05-22*
