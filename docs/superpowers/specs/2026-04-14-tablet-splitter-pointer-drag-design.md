# Tablet Splitter Pointer Drag Design

Date: 2026-04-14

## Goal

Make the main left/right splitter between the PDF reader and tutor panel draggable on tablet devices without changing the existing desktop resizing behavior.

Success means:

- desktop mouse drag still works
- tablet touch drag works without requiring a special long-press delay
- the existing width constraints remain intact
- the change stays local to the current splitter implementation in `GGlearn-AI/src/App.tsx`

## Current Context

The splitter currently uses:

- `onMouseDown` on the divider
- global `mousemove`
- global `mouseup`

This works on desktop because the whole interaction is mouse-only. On tablets, touch input never enters that event chain, so the resize state is never activated.

## Options Considered

### Option 1: Unified Pointer Events

Replace the splitter interaction with:

- `onPointerDown` on the divider
- global `pointermove`
- global `pointerup`
- global `pointercancel`

Why this is the recommended option:

- one interaction model for mouse, touch, and pen
- minimal code churn in the existing implementation
- avoids maintaining parallel mouse and touch state machines
- maps well to the current drag-state pattern already used in `App.tsx`

### Option 2: Add Touch Events Beside Mouse Events

Keep the current mouse logic and add:

- `touchstart`
- `touchmove`
- `touchend`

Why not recommended:

- duplicates the resize logic
- increases platform-specific branching
- makes future fixes more error-prone

### Option 3: Replace With A Split-Pane Library

Swap the local divider logic for a dedicated resizable layout package.

Why not recommended:

- much larger surface area than the bug requires
- risks changing established layout behavior
- unnecessary dependency for a contained interaction fix

## Chosen Design

Use Option 1 and migrate the splitter interaction to pointer events.

### Interaction Model

On `pointerdown` over the divider:

- mark resizing active
- store that a drag session has started
- switch the body cursor to `col-resize`
- add the existing `select-none` body class

During `pointermove`:

- compute the proposed left width from `clientX / window.innerWidth`
- keep the existing `20% < width < 80%` guardrail
- update the temporary ghost divider position while dragging

On `pointerup` or `pointercancel`:

- end the resize session
- restore cursor/body state
- commit `ghostLeftWidth` into persisted `leftWidth`

### Touch-Specific Guardrail

Add `touch-action: none` to the divider so touch dragging is not intercepted by browser panning or gesture heuristics before the app receives pointer movement.

This is intentionally scoped only to the splitter handle, not the whole app shell.

## Data And State Impact

No store schema changes are needed.

The existing state remains:

- `leftWidth`
- `isResizing`
- `ghostLeftWidth`
- `isDragging.current`

Only the event source changes from mouse events to pointer events.

## Error Handling

The main failure mode here is interaction cancellation rather than thrown runtime errors.

The design therefore explicitly handles:

- `pointercancel`
- partially started drags
- cleanup of cursor/body classes even if the drag does not end with a normal `pointerup`

## Testing Strategy

Add a focused `App` interaction test that verifies a touch-style pointer drag changes the splitter width.

The regression should cover:

- `pointerdown` on the divider
- `pointermove` on `window`
- `pointerup` on `window`
- resulting width update on the left panel

Desktop behavior should continue to work because pointer events also cover mouse input.

## Implementation Scope

Files expected to change:

- `GGlearn-AI/src/App.tsx`
- `GGlearn-AI/src/App.test.tsx`

No design-system or backend changes are required.
