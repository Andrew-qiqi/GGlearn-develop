---
name: gsd-brief-handoff
description: Use when important project initialization or phase planning work should enter GSD only after deeper human-agent discussion is condensed into a project brief or phase brief.
---

# GSD Brief Handoff

Use this skill only for narrow pre-GSD workflow setup and execution.

This skill exists to solve one problem:

- native GSD discuss is often not deep enough for major project initialization or major phase planning
- the user wants a cleaner handoff document before entering GSD

This skill does not replace GSD's downstream `.planning/` workflow.

It should also establish the minimal repository entrypoint for this workflow when needed.

For the human-readable workflow explainer, use:

- `references/workflow-overview-zh.md`

## When To Use

Use this skill when all of these are true:

- the user wants deeper discussion before GSD
- the task is important enough to enter GSD
- the result should be captured as a brief before handoff

Typical triggers:

- project setup before `/gsd:new-project`
- major phase planning before `gsd-plan-phase`
- the user explicitly mentions brief, deep discuss, workflow stabilization, or a GSD handoff document

## When Not To Use

Do not use this skill for:

- small fixes
- direct coding tasks that will not use GSD
- pure GSD flows where native `gsd-discuss-phase` is enough

## Core Routing

Choose exactly one path:

1. Small or direct task
   Do not create a brief. Work normally.

2. Project-level pre-GSD handoff
   Ensure `AGENTS.md` exists from `references/AGENTS-template.md`, then create or update `docs/discuss/project-brief.md` using `references/project-brief-template.md`, then stop unless the user also asks to run `/gsd:new-project`.

3. Phase-level pre-GSD handoff
   Ensure `AGENTS.md` exists from `references/AGENTS-template.md`, then create or update `docs/discuss/phases/NN-phase-slug-brief.md` using `references/phase-brief-template.md`, then stop unless the user also asks to continue into GSD.

4. Pure GSD
   If no deeper pre-GSD discussion is needed, skip briefs and use native GSD flow.

## Project Brief Rules

Project brief is for project initialization only.

It should capture:

- what the project is
- core value
- true needs
- non-goals
- constraints
- initial roadmap direction
- project-level locked decisions

Do not put phase-level implementation detail into the project brief.

Default live path:

- `docs/discuss/project-brief.md`

Repository entrypoint:

- `AGENTS.md`

Canonical template:

- `references/project-brief-template.md`
- `references/AGENTS-template.md`

## Phase Brief Rules

Phase brief is for one phase only.

It should capture:

- phase objective
- current problem
- scope and non-scope
- inherited project decisions relevant to this phase
- phase-specific locked decisions
- success criteria
- constraints

Do not repeat the whole project brief.
Reference project-level decisions briefly unless this phase adds a refinement or exception.

Default live path:

- `docs/discuss/phases/NN-phase-slug-brief.md`

Canonical template:

- `references/phase-brief-template.md`

## Handoff To GSD

### Pattern A: Brief-first direct handoff

Use when the brief is complete enough.

Flow:

- deep discuss
- write or update brief
- hand off with:
  - `/gsd:new-project` for project work
  - `gsd-plan-phase <phase> --prd <brief>` for phase work

### Pattern B: Brief plus native GSD delta discuss

Use when the brief exists but some gray areas remain.

Flow:

- deep discuss
- write or update brief
- run `gsd-discuss-phase <phase>`
- sync any new locked decisions back into the phase brief
- then continue into planning

Important:

- `gsd-plan-phase --prd <brief>` skips native GSD discuss for that planning run
- if native `gsd-discuss-phase` adds new decisions after a brief already exists, update the brief so the human-readable handoff record does not drift

### Pattern C: Pure GSD

Use when no extra pre-GSD deep discussion is needed.

## Authoring Standard

Write briefs for humans first.

Requirements:

- use Chinese unless the user wants English
- separate locked decisions from agent discretion
- keep project-level and phase-level content separated
- keep the brief concise and pre-GSD focused
- include metadata fields needed for revision tracking

## Action Rule

When this skill is triggered:

1. decide whether the task should use GSD at all
2. choose project brief, phase brief, or no brief
3. if the repo does not already have a suitable `AGENTS.md`, create or update it from the canonical template
4. create or update the live brief file from the canonical template
5. summarize what was written and what route should be used next
6. do not automatically run GSD unless the user explicitly asks to continue

## References

Read only what you need:

- `references/AGENTS-template.md`
- `references/project-brief-template.md`
- `references/phase-brief-template.md`
- `references/workflow-overview-zh.md`
