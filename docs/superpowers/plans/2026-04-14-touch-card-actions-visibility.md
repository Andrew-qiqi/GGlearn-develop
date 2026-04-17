# Touch Card Actions Visibility Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make tutor-card action buttons discoverable and usable on tablet and other no-hover devices while preserving the existing desktop hover-only density.

**Architecture:** Keep the change local to `CanvasTutor.tsx`. Add a small runtime hover-capability boundary based on `matchMedia('(hover: hover) and (pointer: fine)')`, default action trays to visible on no-hover devices, and preserve the current forced-visible states for open drawers and delete confirmation. Add focused component tests that stub input capability directly instead of relying on user-agent behavior.

**Tech Stack:** React 19, TypeScript, Framer Motion, Zustand, Vitest, Testing Library, JSDOM

---

## File Map

- Modify: `GGlearn-AI/src/components/CanvasTutor.tsx`
  - Add a local hover-capability resolver with safe browser fallbacks.
  - Derive explanation-card and note-card action visibility from capability plus current open state.
  - Add stable test hooks for the action containers.
  - Prevent note action taps from interfering with note dragging.
- Modify: `GGlearn-AI/src/components/CanvasTutor.test.tsx`
  - Add `matchMedia` stubbing helpers.
  - Add regression coverage for touch/no-hover visibility and desktop hover preservation.
- Modify: `docs/frontend/architecture.md`
  - Record the new interaction boundary for tutor-card action visibility.
- Modify: `docs/changelog/CHANGELOG_TECH.md`
  - Log the touch discoverability fix and its scope.

## Task 1: Add Failing Capability-Aware Regression Tests

**Files:**
- Modify: `GGlearn-AI/src/components/CanvasTutor.test.tsx`

- [ ] **Step 1: Add a reusable `matchMedia` stub helper near the test setup**

Add a helper that can simulate both hover-capable and no-hover environments and exposes the methods used by the production effect:

```ts
function mockMatchMedia(matches: boolean) {
  const addEventListener = vi.fn();
  const removeEventListener = vi.fn();
  const addListener = vi.fn();
  const removeListener = vi.fn();

  vi.stubGlobal(
    'matchMedia',
    vi.fn().mockImplementation((query: string) => ({
      matches,
      media: query,
      onchange: null,
      addEventListener,
      removeEventListener,
      addListener,
      removeListener,
      dispatchEvent: vi.fn(),
    }))
  );

  return { addEventListener, removeEventListener, addListener, removeListener };
}
```

Reset it in `afterEach` with:

```ts
vi.unstubAllGlobals();
```

- [ ] **Step 2: Expand the test fixture so one chunk also has a tutor note**

Use the existing store setup and add one note under the visible knowledge card so note-action visibility can be asserted in the same render tree:

```ts
notes: {
  '100,100,300,500': [
    {
      id: 'note-1',
      text: 'Remember this definition.',
    },
  ],
},
```

- [ ] **Step 3: Write the failing no-hover visibility test for explanation-card actions**

Add a focused test that stubs `matchMedia` to `false`, renders `CanvasTutor`, scopes into the `Core Idea` card, and asserts that card's action tray is marked visible without hover:

```tsx
it('shows explanation card actions by default on no-hover devices', () => {
  mockMatchMedia(false);

  render(
    <CanvasTutor
      onSendMessage={vi.fn()}
      onAnalyze={vi.fn()}
      onRegenerateChunk={vi.fn()}
    />
  );

  const card = screen.getByText('Core Idea').closest('.tutor-card');
  expect(within(card!).getByTestId('tutor-card-actions')).toHaveClass(
    'opacity-100',
    'translate-y-0',
    'pointer-events-auto'
  );
});
```

- [ ] **Step 4: Write the failing no-hover visibility test for note actions**

Add a note-specific assertion against a stable test id:

```tsx
it('shows note actions by default on no-hover devices', () => {
  mockMatchMedia(false);

  render(
    <CanvasTutor
      onSendMessage={vi.fn()}
      onAnalyze={vi.fn()}
      onRegenerateChunk={vi.fn()}
      onEditNote={vi.fn()}
      onDeleteNote={vi.fn()}
    />
  );

  expect(screen.getByTestId('tutor-note-actions-note-1')).toHaveClass('opacity-100');
});
```

- [ ] **Step 5: Write the failing desktop preservation test**

Keep the desktop contract explicit:

```tsx
it('keeps explanation card actions hover-gated on hover-capable devices', () => {
  mockMatchMedia(true);

  render(
    <CanvasTutor
      onSendMessage={vi.fn()}
      onAnalyze={vi.fn()}
      onRegenerateChunk={vi.fn()}
    />
  );

  const card = screen.getByText('Core Idea').closest('.tutor-card');
  expect(within(card!).getByTestId('tutor-card-actions')).toHaveClass(
    'opacity-0',
    'translate-y-2',
    'pointer-events-none'
  );
});
```

- [ ] **Step 6: Run the targeted test file to verify RED**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-AI-main/GGlearn-AI
npx vitest --config vitest.node.config.ts run --environment jsdom --globals --exclude '.worktrees/**' src/components/CanvasTutor.test.tsx
```

Expected:

- FAIL because `CanvasTutor` does not yet expose capability-aware visibility or stable action-container selectors

- [ ] **Step 7: Commit the failing test checkpoint**

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-AI-main/GGlearn-AI
git add src/components/CanvasTutor.test.tsx
git commit -m "test(tutor): cover touch action visibility"
```

## Task 2: Add A Local Hover-Capability Boundary In `CanvasTutor`

**Files:**
- Modify: `GGlearn-AI/src/components/CanvasTutor.tsx:1-760`
- Verify with: `GGlearn-AI/src/components/CanvasTutor.test.tsx`

- [ ] **Step 1: Add a local `readSupportsHover` helper above the component exports**

Use capability detection, not user-agent detection:

```ts
function readSupportsHover() {
  if (typeof window === 'undefined' || typeof window.matchMedia !== 'function') {
    return false;
  }

  return window.matchMedia('(hover: hover) and (pointer: fine)').matches;
}
```

This intentionally falls back to `false` so touch-style visibility wins when the browser cannot report hover capability.

- [ ] **Step 2: Add `supportsHover` state plus a `matchMedia` subscription inside `CanvasTutor`**

Keep it local to the component:

```ts
const [supportsHover, setSupportsHover] = useState(readSupportsHover);

useEffect(() => {
  if (typeof window === 'undefined' || typeof window.matchMedia !== 'function') {
    return;
  }

  const mediaQuery = window.matchMedia('(hover: hover) and (pointer: fine)');
  const updateSupportsHover = () => setSupportsHover(mediaQuery.matches);

  updateSupportsHover();

  if (typeof mediaQuery.addEventListener === 'function') {
    mediaQuery.addEventListener('change', updateSupportsHover);
    return () => mediaQuery.removeEventListener('change', updateSupportsHover);
  }

  if (typeof mediaQuery.addListener === 'function') {
    mediaQuery.addListener(updateSupportsHover);
    return () => mediaQuery.removeListener(updateSupportsHover);
  }
}, []);
```

The legacy `addListener` branch matters for older Safari/WebKit behavior on iPad-class devices.

- [ ] **Step 3: Derive action-visibility helpers instead of embedding all logic inline**

Add small local booleans or helper functions:

```ts
const showActionsByDefault = !supportsHover;
const shouldForceCardActionsVisible = showActionsByDefault || Boolean(activeInput) || isConfirmingDelete;
const noteActionsClassName = showActionsByDefault
  ? 'opacity-100'
  : 'opacity-0 group-hover:opacity-100';
const cardActionsClassName = shouldForceCardActionsVisible
  ? 'opacity-100 translate-y-0 pointer-events-auto'
  : 'opacity-0 translate-y-2 pointer-events-none group-hover:opacity-100 group-hover:translate-y-0 group-hover:pointer-events-auto';
```

If a helper function is clearer, prefer one that returns class strings for:

- explanation-card action tray
- note-card action row

## Task 3: Apply The Capability Boundary To Card And Note Actions

**Files:**
- Modify: `GGlearn-AI/src/components/CanvasTutor.tsx:294-379`
- Modify: `GGlearn-AI/src/components/CanvasTutor.tsx:549-575`

- [ ] **Step 1: Update the explanation-card action tray classes**

Refactor the tray near the current hover-only block:

```tsx
<div
  data-testid="tutor-card-actions"
  className={`absolute -bottom-3 right-4 flex items-center gap-1 bg-bg-elevated border border-border-strong shadow-lg rounded-xl p-1 transition-all duration-300 z-10 ${cardActionsClassName}`}
>
```

Requirements:

- no-hover devices start visible and clickable
- desktop keeps the existing hover reveal
- `activeInput` and delete confirmation still force visibility

- [ ] **Step 2: Update the note action row to match the same capability rule**

Add a stable selector and keep the desktop hover path:

```tsx
<div
  data-testid={`tutor-note-actions-${note.id}`}
  className={cn(
    'flex items-center gap-1 transition-opacity',
    showActionsByDefault ? 'opacity-100' : 'opacity-0 group-hover:opacity-100'
  )}
>
```

- [ ] **Step 3: Prevent visible note action taps from starting an accidental drag**

Add pointer/mouse event isolation on the note action row or on each note action button:

```tsx
onPointerDown={(e) => e.stopPropagation()}
onMouseDown={(e) => e.stopPropagation()}
```

Apply this to the action controls themselves, not the whole note content. The goal is:

- tapping `Edit` still opens edit mode immediately
- tapping `Delete` still opens confirmation immediately
- dragging elsewhere on the note card still works

- [ ] **Step 4: Keep the visual footprint stable**

While adjusting classes, confirm the visible tray still sits at the current anchored position and does not expand into a multi-row layout. The fix should preserve:

- the compact pill action tray on explanation cards
- the inline icon row on notes

- [ ] **Step 5: Run the targeted test file to verify GREEN**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-AI-main/GGlearn-AI
npx vitest --config vitest.node.config.ts run --environment jsdom --globals --exclude '.worktrees/**' src/components/CanvasTutor.test.tsx
```

Expected:

- PASS
- new no-hover visibility tests pass
- existing intro-card / low-accuracy / math-rendering tests remain green

- [ ] **Step 6: Commit the implementation in the inner repo**

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-AI-main/GGlearn-AI
git add src/components/CanvasTutor.tsx src/components/CanvasTutor.test.tsx
git commit -m "fix(tutor): show card actions on touch devices"
```

## Task 4: Update Docs And Record Verification

**Files:**
- Modify: `docs/frontend/architecture.md`
- Modify: `docs/changelog/CHANGELOG_TECH.md`

- [ ] **Step 1: Update frontend architecture notes**

Add a short section describing the tutor-card interaction boundary:

- `CanvasTutor.tsx` now treats hover capability as a runtime interaction boundary
- hover-capable devices keep progressive disclosure
- no-hover devices render card actions visible by default
- note action taps are isolated from note dragging

- [ ] **Step 2: Add a changelog entry**

Record:

- what changed
- why tablets could not discover these actions before
- what behaviors are preserved on desktop
- what targeted tests now cover

- [ ] **Step 3: Re-run the targeted verification after docs are in place**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-AI-main/GGlearn-AI
npx vitest --config vitest.node.config.ts run --environment jsdom --globals --exclude '.worktrees/**' src/components/CanvasTutor.test.tsx
```

Optional sanity check if time permits:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-AI-main/GGlearn-AI
npx vitest run src/App.test.tsx
```

Note in the final write-up that full repo typechecking is still known to be blocked in this environment by pre-existing dependency/env issues unless that situation changes during implementation.

- [ ] **Step 4: Commit the docs in the outer repo**

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/GGlearn-AI-main
git add docs/frontend/architecture.md docs/changelog/CHANGELOG_TECH.md docs/superpowers/plans/2026-04-14-touch-card-actions-visibility.md
git commit -m "docs: record touch card action visibility plan"
```
