# Codebase Structure

**Analysis Date:** 2025-05-15

## Directory Layout

```
SlideTutor-AI/
├── api/                # Backend serverless API routes (Express-based)
├── docs/               # Project documentation and specifications
├── public/             # Static assets (icons, manifest)
└── src/                # Frontend React application
    ├── components/     # UI components (React)
    │   ├── Header/     # App header and global analysis
    │   └── Sidebar/    # Library and PDF management
    ├── config/         # App configuration (models, settings)
    ├── hooks/          # Domain-specific logic (React Hooks)
    ├── lib/            # Utilities and external integrations
    │   ├── ai/         # Prompt engineering and AI logic
    │   ├── pdf/        # PDF layout and processing utilities
    │   └── db.ts       # IndexedDB persistence layer
    ├── store/          # Zustand global state stores
    ├── test/           # Test setup and utilities
    ├── App.tsx         # Main layout component
    ├── main.tsx        # React entry point
    └── types.ts        # Global TypeScript definitions
```

## Directory Purposes

**api/:**
- Purpose: Backend orchestration of AI services and security.
- Contains: Express app, security middleware, and route handlers.
- Key files: `generate.ts` (main AI logic), `server.ts` (entry point).

**src/components/:**
- Purpose: Reusable UI building blocks.
- Contains: React components categorized by function (Header, Sidebar, Canvas).
- Key files: `PdfViewer.tsx`, `CanvasTutor.tsx`, `AskYouTutor.tsx`.

**src/hooks/:**
- Purpose: Encapsulates business logic for the tutor application.
- Contains: Custom React hooks that bridge components and state/API.
- Key files: `useSlideAnalysis.ts`, `useFollowUp.ts`, `useQuiz.ts`.

**src/lib/:**
- Purpose: Core utilities and low-level integrations.
- Contains: AI prompt building, PDF layout logic, and database interactions.
- Key files: `db.ts`, `ai/prompts.ts`, `pdf/layoutUtils.ts`.

**src/store/:**
- Purpose: Global reactive state management.
- Contains: Zustand store definitions.
- Key files: `pdfStore.ts`, `tutorStore.ts`, `uiStore.ts`.

## Key File Locations

**Entry Points:**
- `SlideTutor-AI/src/main.tsx`: Frontend React entry point.
- `SlideTutor-AI/server.ts`: Backend Express server entry point.

**Configuration:**
- `SlideTutor-AI/src/config/models.ts`: AI model definitions and providers.
- `SlideTutor-AI/vite.config.ts`: Vite build and dev configuration.

**Core Logic:**
- `SlideTutor-AI/api/generate.ts`: Backend AI processing and Azure integration.
- `SlideTutor-AI/src/hooks/useSlideAnalysis.ts`: Frontend orchestration of slide explanation.

**Testing:**
- `SlideTutor-AI/src/test/setup.ts`: Vitest global setup.
- `SlideTutor-AI/src/lib/ai/prompts.test.ts`: AI prompt unit tests.

## Naming Conventions

**Files:**
- Components: PascalCase (`CanvasTutor.tsx`)
- Hooks: camelCase starting with `use` (`useFollowUp.ts`)
- Utilities/Stores: camelCase (`pdfStore.ts`, `db.ts`)
- Tests: `[name].test.ts` or `[name].test.tsx`

**Directories:**
- Feature-based: lowercase (`components/Header`, `lib/ai`)

## Where to Add New Code

**New AI Task/Feature:**
1. Define types in `src/types.ts`.
2. Update `src/lib/ai/prompts.ts` for prompt engineering.
3. Add backend logic in `api/generate.ts` if needed.
4. Create a custom hook in `src/hooks/` to manage the interaction.
5. Create UI components in `src/components/` and integrate into `App.tsx` or `CanvasTutor.tsx`.

**New Persistence Store:**
1. Define the object store and its methods in `src/lib/db.ts`.
2. Create a Zustand store in `src/store/` to manage the in-memory version.

**New Utility:**
- Shared helper logic should go in `src/lib/`.

## Special Directories

**api/:**
- Purpose: Serves as serverless functions on Vercel or as a standard Express app locally.
- Generated: No
- Committed: Yes

**docs/:**
- Purpose: Contains product specs, global analysis design docs, and feedback.
- Committed: Yes

---

*Structure analysis: 2025-05-15*
