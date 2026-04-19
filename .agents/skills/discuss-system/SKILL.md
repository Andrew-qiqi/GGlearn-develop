---
name: discuss-system
description: Use when starting, joining, or concluding a structured multi-agent discussion that should be recorded in local project files
---

# Discuss System

Use this skill to run a lightweight, file-based discussion between agents without OMX runtime integration.

## Boundary

- Store discussion artifacts under `.omc/discussions/`.
- Do not use `.omx/` state, plans, or runtime workflows for this skill.
- Do not rely on Claude-only shared-memory MCP tools.
- Keep the discussion readable as markdown first; JSON state is only for coordination.

## Commands

- Start: `/discuss-start <topic> [--leader-name <name>]`
- Speak: `/discuss-round <discussion-id> [--name <name>] [--role <role>] [--background <background>]`
- Conclude: `/discuss-conclude <discussion-id>`

If the platform does not support slash commands, treat those strings as task intents and execute the matching flow manually.

## Storage

Each discussion lives in:

```text
.omc/discussions/{discussion-id}/
  meta.json
  messages.json
  stage-1.md
  .speaking.lock/
```

`meta.json` tracks topic, stage, round, status, leader, participants, and current lock metadata. `messages.json` stores normalized messages for agents that need structured history. Stage markdown is the human-readable source of record.

## Speaking Lock

Use a local lock directory instead of shared memory:

1. Acquire by creating `.speaking.lock/`.
2. Write `holder` and `timestamp` files inside the lock directory.
3. Re-read `meta.json` and current `stage-N.md`.
4. Append exactly one contribution.
5. Update `messages.json` and `meta.json`.
6. Remove `.speaking.lock/`.

If the lock timestamp is older than 60 seconds, treat it as stale, record that in the new message metadata, and replace the lock.

## Start Flow

1. Create a slugged discussion ID from current timestamp and topic.
2. Create the discussion directory under `.omc/discussions/`.
3. Create `meta.json`, `messages.json`, and `stage-1.md`.
4. Register the leader as the first participant.
5. Add the initial agenda and first questions to `stage-1.md`.
6. Return the discussion ID and file path.

## Round Flow

1. Read `meta.json`, `messages.json`, and the current stage file.
2. Register the speaker if this is their first contribution.
3. Acquire the speaking lock.
4. Write a concise contribution that references prior messages when useful.
5. Append the contribution to the current stage file.
6. Update JSON state and release the lock.

## Conclude Flow

1. Confirm the caller is the leader in `meta.json`.
2. Acquire the speaking lock.
3. Summarize consensus, disagreements, decisions, and action items.
4. Append the summary to the current stage file.
5. End the discussion or create the next stage based on the user's instruction.
6. Update JSON state and release the lock.

## Supporting References

- `discuss-start.md`
- `discuss-round.md`
- `discuss-conclude.md`
