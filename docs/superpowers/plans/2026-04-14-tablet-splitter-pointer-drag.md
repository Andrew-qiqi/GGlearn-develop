# Tablet Splitter Pointer Drag Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the main PDF/tutor splitter draggable on tablet touch devices while preserving the existing desktop resize behavior and width constraints.

**Architecture:** Keep the implementation local to `App.tsx` and migrate the divider from mouse-only drag handling to a unified pointer-event drag session. Add one focused regression test in `App.test.tsx` that simulates a touch pointer drag and verifies that the left panel width changes.

**Tech Stack:** React 19, TypeScript, Zustand, Vitest, Testing Library, JSDOM

---

## File Map

- Modify: `SlideTutor-AI/src/App.tsx`
  - Replace the current mouse-only splitter drag logic with pointer-event handling.
  - Keep the existing width guardrail (`20 < leftWidth < 80`) and ghost divider behavior.
  - Add a stable selector for the divider plus touch-specific interaction guardrails.
- Modify: `SlideTutor-AI/src/App.test.tsx`
  - Add a focused regression test for touch-style divider dragging.
  - Stub the browser width so width calculations are deterministic.

## Task 1: Add A Failing Touch Drag Regression Test

**Files:**
- Modify: `SlideTutor-AI/src/App.test.tsx`

- [ ] **Step 1: Add a stable selector for the future divider test target**

Planned production hook:

```tsx
<div data-testid="panel-resizer" ... />
```

Test target:

```ts
const resizer = screen.getByTestId('panel-resizer');
```

- [ ] **Step 2: Write the failing touch pointer drag test**

Add a new test near the existing `App` interaction tests:

```tsx
it('resizes panels from a touch-style pointer drag on the splitter', () => {
  Object.defineProperty(window, 'innerWidth', {
    configurable: true,
    writable: true,
    value: 1000,
  });

  const { container } = render(<App />);
  const resizer = screen.getByTestId('panel-resizer');
  const leftPanel = screen.getByTestId('pdf-panel');

  expect(leftPanel).toHaveStyle({ width: '50%' });

  fireEvent.pointerDown(resizer, {
    pointerId: 7,
    pointerType: 'touch',
    clientX: 500,
    button: 0,
  });

  fireEvent.pointerMove(window, {
    pointerId: 7,
    pointerType: 'touch',
    clientX: 650,
  });

  fireEvent.pointerUp(window, {
    pointerId: 7,
    pointerType: 'touch',
    clientX: 650,
  });

  expect(leftPanel).toHaveStyle({ width: '65%' });
});
```

- [ ] **Step 3: Run the targeted test to verify RED**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI
npx vitest run src/App.test.tsx
```

Expected:

- FAIL because the divider has no stable selector yet and/or pointer events do not trigger resizing

- [ ] **Step 4: Commit the failing test checkpoint**

```bash
git add src/App.test.tsx
git commit -m "test(app): cover touch splitter dragging"
```

## Task 2: Implement Unified Pointer-Based Splitter Dragging

**Files:**
- Modify: `SlideTutor-AI/src/App.tsx`
- Verify with: `SlideTutor-AI/src/App.test.tsx`

- [ ] **Step 1: Add refs and helpers for pointer-session tracking**

Add a dedicated pointer-session ref beside the existing drag refs:

```tsx
const activeResizePointerId = useRef<number | null>(null);
```

Add a small helper to finish the resize session cleanly:

```tsx
const finishResize = useCallback(() => {
  if (!isDragging.current) return;

  isDragging.current = false;
  activeResizePointerId.current = null;
  document.body.style.cursor = 'default';
  document.body.classList.remove('select-none');
  setIsResizing(false);
  setGhostLeftWidth(prev => {
    if (prev !== null) {
      setLeftWidth(prev);
    }
    return null;
  });
}, [setLeftWidth]);
```

- [ ] **Step 2: Replace the mouse-only global listeners with pointer listeners**

In the `useEffect` that currently registers `mousemove` and `mouseup`, switch to:

```tsx
const handlePointerMove = (e: PointerEvent) => {
  if (!isDragging.current) return;
  if (activeResizePointerId.current !== null && e.pointerId !== activeResizePointerId.current) return;

  const newWidth = (e.clientX / window.innerWidth) * 100;
  if (newWidth > 20 && newWidth < 80) {
    setGhostLeftWidth(newWidth);
  }
};

const handlePointerEnd = (e: PointerEvent) => {
  if (activeResizePointerId.current !== null && e.pointerId !== activeResizePointerId.current) return;
  finishResize();
};

window.addEventListener('pointermove', handlePointerMove);
window.addEventListener('pointerup', handlePointerEnd);
window.addEventListener('pointercancel', handlePointerEnd);
```

Cleanup should remove all three listeners.

- [ ] **Step 3: Update the divider element to start pointer drag sessions**

Replace `onMouseDown` with `onPointerDown`:

```tsx
onPointerDown={(e) => {
  if (e.button !== 0) return;

  isDragging.current = true;
  activeResizePointerId.current = e.pointerId;
  setIsResizing(true);
  document.body.style.cursor = 'col-resize';
  document.body.classList.add('select-none');
}}
```

Also add:

```tsx
data-testid="panel-resizer"
className="... touch-none ..."
```

The `touch-none` class is required so touch dragging is not intercepted by browser panning.

- [ ] **Step 4: Add a stable selector for the left panel under test**

Mark the left panel container:

```tsx
<div
  data-testid="pdf-panel"
  style={{ width: `${leftWidth}%` }}
  className="relative flex flex-col z-10"
>
```

This keeps the regression test focused on the actual resized container.

- [ ] **Step 5: Run the targeted test to verify GREEN**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI
npx vitest run src/App.test.tsx
```

Expected:

- PASS
- the new touch pointer regression passes
- existing upload interaction tests remain green

- [ ] **Step 6: Commit the implementation**

```bash
git add src/App.tsx src/App.test.tsx
git commit -m "fix(app): enable splitter dragging on touch devices"
```

## Task 3: Final Verification

**Files:**
- Verify only

- [ ] **Step 1: Re-run the focused app tests**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI
npx vitest run src/App.test.tsx
```

Expected:

- PASS

- [ ] **Step 2: Run a minimal typecheck for the touched app file**

Run:

```bash
cd /Users/qiqicute/Documents/z_cqmeng_file/local_repository/SlideTutor-AI-main/SlideTutor-AI
npx tsc --noEmit --moduleResolution bundler --module esnext --target es2022 --jsx react-jsx --skipLibCheck src/App.tsx
```

Expected:

- PASS, or if repository-wide ambient declarations block this path, record the exact blocker instead of claiming success

- [ ] **Step 3: Manual verification note**

Verify in browser on:

- desktop: divider still drags with mouse
- tablet/touch device: divider drags immediately on touch without waiting for a special long press

- [ ] **Step 4: Commit any final polish if verification required code changes**

```bash
git add src/App.tsx src/App.test.tsx
git commit -m "test(app): finalize splitter pointer drag verification"
```
