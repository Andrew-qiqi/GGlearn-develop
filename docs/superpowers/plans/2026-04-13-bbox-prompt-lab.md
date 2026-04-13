# BBox Prompt Lab Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a standalone local experiment tool that compares bbox-grounded prompt behavior across Gemini and OpenAI-compatible models without modifying the main SlideTutor app.

**Architecture:** Add a static HTML lab in `tmp_files/` plus a tiny Node proxy in `tmp_files/` that accepts provider credentials, forwards one structured explain request, and returns raw model JSON. Keep the lab isolated from app runtime code; only reuse prompt concepts, not app wiring.

**Tech Stack:** Static HTML/CSS/JS, Node HTTP server, native `fetch`, `openai` SDK, `node:test`

**Status:** Implemented and locally validated on 2026-04-13.

**Implemented extras beyond the original minimum scope:**
- `tmp_files/bbox_prompt_lab_bridge.js`
- `tmp_files/bbox_prompt_lab_bridge.test.cjs`
- DeepSeek snapshot import so the lab can reuse parser output plus PDF-rendered page images

---

### Task 1: Define isolated files and responsibilities

**Files:**
- Create: `tmp_files/bbox_prompt_lab.html`
- Create: `tmp_files/bbox_prompt_lab_proxy.mjs`
- Create: `tmp_files/bbox_prompt_lab_proxy.test.mjs`

- [x] **Step 1: Lock file boundaries**

Decide:
- `bbox_prompt_lab.html` owns UI, local persistence, request assembly, and result overlays.
- `bbox_prompt_lab_proxy.mjs` owns CORS, provider dispatch, request validation, and raw API forwarding.
- `bbox_prompt_lab_proxy.test.mjs` owns request-building tests for Gemini and OpenAI-compatible payload generation.

- [x] **Step 2: Keep the scope minimal**

Cut:
- parser integration
- multi-page workflows
- main app imports
- explanation quality benchmarking
- production auth/storage

### Task 2: TDD the proxy request builders

**Files:**
- Test: `tmp_files/bbox_prompt_lab_proxy.test.mjs`
- Modify: `tmp_files/bbox_prompt_lab_proxy.mjs`

- [x] **Step 1: Write failing tests for request construction**

Cover:
- Gemini request body uses model + prompt + slide image + structured regions
- OpenAI-compatible request body uses the same content contract
- Shared bbox-rule patch replaces only the bbox-related prompt block

- [x] **Step 2: Run tests and confirm failure**

Run: `node --test tmp_files/bbox_prompt_lab_proxy.test.mjs`

- [x] **Step 3: Implement the minimal proxy helpers**

Add:
- prompt patch helper
- provider-specific payload builders
- one local POST endpoint for browser calls

- [x] **Step 4: Re-run tests and confirm pass**

Run: `node --test tmp_files/bbox_prompt_lab_proxy.test.mjs`

### Task 3: Build the standalone lab UI

**Files:**
- Modify: `tmp_files/bbox_prompt_lab.html`

- [x] **Step 1: Create a focused UI**

Include:
- provider and model inputs
- API key input
- slide image upload
- parsed-regions JSON textarea
- bbox-only prompt textarea
- run button
- parsed region overlay toggle
- model bbox overlay + raw JSON output

- [x] **Step 2: Persist only local experiment state**

Store in `localStorage`:
- provider
- model
- API key
- bbox prompt block
- parsed regions JSON draft

- [x] **Step 3: Keep the explanation prompt stable**

Ensure:
- only bbox-related prompt text is editable
- the rest of the request template stays fixed for comparison quality

### Task 4: Verify end to end locally

**Files:**
- Modify if needed: `tmp_files/bbox_prompt_lab.html`
- Modify if needed: `tmp_files/bbox_prompt_lab_proxy.mjs`

- [x] **Step 1: Syntax-check the proxy**

Run: `node --check tmp_files/bbox_prompt_lab_proxy.mjs`

- [x] **Step 2: Start the proxy and sanity-check browser flow**

Run: `node tmp_files/bbox_prompt_lab_proxy.mjs`

Expected:
- server starts on a documented localhost port
- browser lab can call proxy without CORS errors

- [x] **Step 3: Verify the static lab manually**

Check:
- image upload renders
- parsed regions overlay draws
- API request returns raw JSON
- first detected `intent` bbox renders

- [x] **Step 4: Record usage instructions in-file**

### Verification notes

Validated during implementation with:

- `node --test tmp_files/bbox_prompt_lab_proxy.test.mjs tmp_files/bbox_prompt_lab_bridge.test.cjs`
- `node --check tmp_files/bbox_prompt_lab_proxy.mjs`
- `node --check tmp_files/bbox_prompt_lab_bridge.js`

Manual checks covered:

- parser-region overlays draw against the imported PDF-rendered page image
- the lab can import DeepSeek snapshot data without localStorage quota failures
- Gemini / OpenAI-compatible requests return structured explain JSON and render intent boxes in the overlay viewer

Add a concise footer or help note covering:
- how to start the proxy
- accepted providers
- expected input format
