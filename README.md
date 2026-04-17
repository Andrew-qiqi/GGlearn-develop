# GGlearn Workspace

This workspace contains the active GGlearn application plus the local agent and planning infrastructure used to evolve it.

## Repository Layout

- `GGlearn/`: active product repository and current MVP codebase
- `.planning/`: roadmap, requirements, and phase artifacts
- `.omx/`, `.omc/`: local agent runtime state, reports, and execution artifacts
- `.agents/`, `.codex/`, `.claude/`, `.gemini/`: local AI workflow configuration

## Current Product Focus

GGlearn is currently centered on a textbook-first study flow:

- configure AI access
- import source material
- generate a structured outline
- expand chapters into readable content
- study through notes, handwriting, AI explanation, quizzes, and export

## Cleanup Direction

The current cleanup goal is to keep the GGlearn MVP path clear, reduce leftover SlideTutor-era descriptions, and remove front-end branches that are no longer connected to the active notebook flow.
