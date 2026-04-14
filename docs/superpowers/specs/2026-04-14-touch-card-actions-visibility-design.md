# Touch Card Actions Visibility Design

Date: 2026-04-14

## Goal

Make tutor-card action controls discoverable and usable on touch devices without degrading the current desktop hover-driven experience.

The affected controls include:

- explanation card actions: `follow-up`, `add note`, `regenerate`, `edit`, `delete`
- explanation note card actions: `edit`, `delete`

## Problem Summary

The current card-action UI assumes hover exists.

In `CanvasTutor.tsx`:

- the explanation card action tray starts hidden with `opacity-0`, `translate-y-2`, and `pointer-events-none`
- it only becomes visible through `group-hover:*`
- the note-card edit/delete actions use the same `group-hover` pattern

This works on desktop because pointer hover exists. On tablet devices, there is no mouse-hover equivalent, so the controls remain visually hidden and non-interactive.

## Root Cause

This is not just a generic “touch devices do not support hover” issue. There are two specific implementation choices causing the break:

1. **Visibility is CSS-hover only**
   - no explicit touch-device override exists
   - no persistent selected/open card state exists

2. **Interactivity is also hover-gated**
   - hidden action trays also keep `pointer-events-none`
   - even if a browser emulates some hover behavior, the controls are still designed around desktop hover timing rather than touch discovery

## Options Considered

### Option 1: Always show action controls on no-hover / coarse-pointer devices

Use capability detection to treat touch devices differently:

- desktop/fine-pointer devices keep the existing hover-reveal behavior
- no-hover/coarse-pointer devices render the action controls visible by default

Why this is recommended:

- smallest change set
- no new gesture/state model required
- easiest for users to discover immediately
- lowest risk of breaking follow-up / note / regenerate flows

### Option 2: Tap card to reveal actions

Introduce an explicit “selected card” state:

- first tap reveals controls
- second tap or outside-tap clears them

Why not recommended right now:

- more state coordination
- more edge cases around open drawers, focus, note dragging, and page changes
- slower to implement and test

### Option 3: Permanent visibility on all platforms

Always render action controls visible on both desktop and touch.

Why not recommended:

- changes the established desktop visual density
- loses the quieter hover-on-demand desktop behavior

## Chosen Design

Use Option 1.

### Capability boundary

Introduce a runtime `supportsHover` or equivalent capability signal based on browser input capabilities, such as:

- `matchMedia('(hover: hover) and (pointer: fine)')`

Interpretation:

- `supportsHover = true`: desktop-like hover behavior
- `supportsHover = false`: touch/no-hover behavior

This capability should be treated as an interaction boundary, not a user-agent rule.

Important considerations:

- do not use user-agent sniffing for iPad/tablet detection
- if `matchMedia` is unavailable, fall back to `supportsHover = false` so touch-style visibility wins instead of hiding controls
- if the browser reports capability changes at runtime, the UI should follow the latest media-query result rather than caching a one-time guess
- hybrid devices should follow actual input capability; a tablet with a real hover-capable pointer may keep the desktop hover behavior

## Explanation Card Contract

For explanation cards:

- keep the existing `group-hover` reveal behavior when `supportsHover` is true
- render the action tray visible and interactive by default when `supportsHover` is false

The touch-visible state must still cooperate with the existing logic that forces visibility when:

- an input drawer is open
- delete confirmation is open

That means the action tray visibility should be derived from both:

- device interaction capability
- current card interaction state

## Note Card Contract

For tutor note cards inside `CanvasTutor.tsx`:

- keep the current hover-only edit/delete reveal on desktop
- show those controls by default on no-hover / coarse-pointer devices

This should apply only to the card-action area. It should not change the drag behavior of the note card itself.

## Important Guardrails

### 1. Do not degrade desktop density

Desktop users should continue seeing the compact hover-reveal interaction they already have.

### 2. Do not block card text or drawers

When actions become always visible on touch devices:

- they must not cover important text
- they must not overlap the input drawer in a broken way
- delete confirmation and open input drawers still take precedence

### 3. Do not break note dragging

Touch-visible note actions must not interfere with:

- dragging note cards between chunks
- pointer-down stop-propagation already used to isolate note interaction

### 4. Keep the change local

The first pass should stay local to `CanvasTutor.tsx`.

No store-level “selected card” state is needed in this version.

### 5. Preserve existing interaction priority

The new touch-visible controls must not weaken current higher-priority states:

- delete-confirm UI still overrides the normal icon row
- open follow-up / note / regenerate drawers still keep the tray visibly anchored
- note-card drag gestures still begin from the note card as they do today, without accidental action activation on drag start

## Testing Strategy

Add focused component tests for `CanvasTutor` that verify:

1. on no-hover devices, explanation card actions are visible without hover
2. on no-hover devices, note-card edit/delete actions are visible without hover
3. desktop behavior still keeps hover-driven visibility semantics

Tests should avoid user-agent mocking. Prefer stubbing the interaction-capability boundary directly.

## Implementation Scope

Expected files:

- `SlideTutor-AI/src/components/CanvasTutor.tsx`
- `SlideTutor-AI/src/components/CanvasTutor.test.tsx`

Optional only if needed:

- a small local capability helper near `CanvasTutor`, but no global store change
