# Repository Workflow Entry

This repository uses the `gsd-brief-handoff` skill for narrow pre-GSD workflow setup.

Use that skill when:

- important work should enter GSD only after deeper human-agent discussion
- a `project brief` is needed before `/gsd:new-project`
- a `phase brief` is needed before `gsd-plan-phase`

Do not use that skill for every task.

- small fixes can be handled directly
- pure GSD flows can use native GSD discuss and planning

Live pre-GSD brief files:

- `docs/discuss/project-brief.md`
- `docs/discuss/phases/*.md`

Canonical rules and templates live in:

- `.agents/skills/gsd-brief-handoff/`
