# Technology Stack

**Analysis Date:** 2025-05-22

## Languages

**Primary:**
- TypeScript 5.8.2 - Used across frontend components (`src/`) and backend logic (`api/`, `server.ts`).

**Secondary:**
- JavaScript (ES Modules) - Used in build/config files (`vite.config.ts`, `generate-icon.ts`).

## Runtime

**Environment:**
- Node.js (Vercel and Express) - Backend operations in `api/generate.ts`.
- Browser (React 19) - Frontend interface.

**Package Manager:**
- npm (assumed from `package.json`)
- Lockfile: Missing (none detected in `SlideTutor-AI/`)

## Frameworks

**Core:**
- React 19.0.0 - Frontend UI library.
- Express 4.21.2 - Backend server framework in `api/generate.ts` and `server.ts`.
- Tailwind CSS 4.1.14 - Utility-first styling with `@tailwindcss/vite` integration.

**Testing:**
- Vitest 4.1.0 - Test runner.
- React Testing Library 16.3.2 - UI testing.

**Build/Dev:**
- Vite 6.2.0 - Build tool and dev server.
- tsx 4.21.0 - Executing TypeScript files directly (`server.ts`).

## Key Dependencies

**Critical:**
- `@xyflow/react` 12.10.1 - Node-based visualization (React Flow) for slide content mapping.
- `zustand` 5.0.11 - Client-side state management for PDF and tutor states.
- `pdfjs-dist` 5.5.207 - PDF parsing and rendering on the client.
- `lucide-react` 0.546.0 - Icon set.
- `motion` 12.23.24 - Framer Motion for UI animations.

**Infrastructure:**
- `@google/genai` 1.29.0 - SDK for Google Gemini AI models.
- `openai` 6.27.0 - SDK for OpenAI-compatible AI providers (Doubao, Qwen).
- `nodemailer` 8.0.2 - Email handling for feedback and security alerts.
- `better-sqlite3` 12.4.1 - SQLite integration (present in dependencies, but limited core usage detected).
- `express-rate-limit` 8.3.1 - Rate limiting for API security.
- `helmet` 8.1.0 - Security headers for Express.

## Configuration

**Environment:**
- `.env` file for local development (see `.env.example`).
- Key configs: `GEMINI_API_KEY`, `APP_URL`, `SMTP_HOST`, `AZURE_DOCUMENT_INTELLIGENCE_KEY`.

**Build:**
- `vite.config.ts`: Vite build configuration.
- `tsconfig.json`: TypeScript compiler options.
- `vercel.json`: Vercel deployment and routing rules.
- `edgeone.json`: Tencent Cloud EdgeOne routing.

## Platform Requirements

**Development:**
- Node.js (LTS recommended)
- TypeScript support

**Production:**
- Vercel (recommended deployment target)
- EdgeOne (Tencent Cloud) for routing and edge functions.

---

*Stack analysis: 2025-05-22*
