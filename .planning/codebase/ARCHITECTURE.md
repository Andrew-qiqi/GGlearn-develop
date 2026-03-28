# Architecture

**Analysis Date:** 2025-05-15

## Pattern Overview

**Overall:** Client-Server AI Orchestrator

**Key Characteristics:**
- **Frontend SPA:** A React 19 application built with Vite, managing user interactions and local state.
- **AI-Centric Backend:** An Express server that acts as a proxy/orchestrator for multiple AI services (Gemini, OpenAI, Azure Document Intelligence).
- **Offline-First Persistence:** Heavy use of IndexedDB via `src/lib/db.ts` to store PDF files and their generated AI explanations/notes, ensuring data persists across sessions without a centralized database.
- **Streaming Responses:** AI-generated content is streamed from the backend to the frontend for a responsive "real-time" experience.

## Layers

**UI Layer (React):**
- Purpose: Renders the PDF viewer, tutor canvas, and sidebar.
- Location: `SlideTutor-AI/src/components`
- Contains: Functional React components using Tailwind CSS and Framer Motion.
- Depends on: `Store Layer`, `Logic Layer (Hooks)`
- Used by: Entry point `SlideTutor-AI/src/main.tsx`

**State Layer (Zustand):**
- Purpose: Manages global reactive state (UI, PDF metadata, Tutor session state).
- Location: `SlideTutor-AI/src/store`
- Contains: Zustand stores like `pdfStore.ts`, `tutorStore.ts`, `uiStore.ts`.
- Depends on: `types.ts`
- Used by: `UI Layer`, `Logic Layer`

**Logic Layer (Custom Hooks):**
- Purpose: Encapsulates domain-specific business logic and side effects.
- Location: `SlideTutor-AI/src/hooks`
- Contains: Hooks for PDF analysis (`useSlideAnalysis.ts`), follow-up management (`useFollowUp.ts`), and quiz generation (`useQuiz.ts`).
- Depends on: `State Layer`, `API Client`, `Persistence Layer`
- Used by: `UI Layer` (primarily `App.tsx`)

**Persistence Layer (IndexedDB):**
- Purpose: Handles local storage of PDF binary data and associated AI state.
- Location: `SlideTutor-AI/src/lib/db.ts`
- Contains: CRUD operations using the browser's IndexedDB API.
- Depends on: None
- Used by: `Logic Layer`

**API Layer (Express):**
- Purpose: Provides secure, rate-limited access to AI services and handles complex layout analysis.
- Location: `SlideTutor-AI/api/` and `SlideTutor-AI/server.ts`
- Contains: Route handlers for `/api/generate`, `/api/parse`, and `/api/feedback`.
- Depends on: `AI SDKs`, `Azure SDK`
- Used by: `Logic Layer` via `fetch`

## Data Flow

**PDF Analysis Flow:**

1. User uploads a PDF; it's saved to IndexedDB (`src/lib/db.ts`).
2. `useSlideAnalysis.ts` triggers an analysis request.
3. PDF page is converted to image/text content.
4. Request is sent to `/api/generate`.
5. Backend performs Azure Document Intelligence layout analysis (`api/generate.ts`).
6. Backend builds a prompt based on task and layout.
7. Backend streams the AI response (Gemini/OpenAI) back to the frontend.
8. Frontend updates `tutorStore.ts` and persists results to IndexedDB.

**State Management:**
- **Transient State:** Handled by Zustand stores for immediate UI reactivity.
- **Persistent State:** Synchronized with IndexedDB periodically to survive page reloads.

## Key Abstractions

**PageState:**
- Purpose: Represents the complete state of an analyzed PDF page (explanations, follow-ups, notes, layout blocks).
- Examples: `src/types.ts`
- Pattern: Domain Model

**LayoutBlock:**
- Purpose: Represents a semantic unit detected in a PDF (text, table, figure) with bounding box info.
- Examples: `src/types.ts`, `api/generate.ts`
- Pattern: Structural Metadata

## Entry Points

**Frontend Entry:**
- Location: `SlideTutor-AI/src/main.tsx`
- Triggers: Browser page load.
- Responsibilities: Initializes React, mounts `App.tsx`, sets up Vercel Analytics.

**Backend Entry:**
- Location: `SlideTutor-AI/server.ts`
- Triggers: Server startup (`tsx server.ts`).
- Responsibilities: Configures Express, security middleware, rate limiting, and mounts API routes + Vite dev server.

## Error Handling

**Strategy:** Fail-soft with user-facing alerts and logging.

**Patterns:**
- **Security Guard:** `api/generate.ts` blocks unauthorized domains and handles rate limits.
- **Moderation Check:** `checkMaliciousIntent` in backend filters out jailbreak attempts.
- **Abort Controller:** `useTutorStore.ts` manages `AbortController` to cancel ongoing AI generation.

## Cross-Cutting Concerns

**Logging:** Backend API logging in `server.ts` tracks IP, method, and duration.
**Validation:** Backend checks input types and formats in `api/generate.ts`.
**Authentication:** Implicitly handled by domain-level security guards and API keys stored in environment variables.

---

*Architecture analysis: 2025-05-15*
