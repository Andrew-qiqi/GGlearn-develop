---
skill: discuss-start
description: Initialize a multi-agent discussion meeting where agents can join dynamically
---

# Discuss Start

Initialize a new multi-agent discussion meeting where agents (Claude, Codex, Gemini, etc.) can join dynamically and collaborate asynchronously through shared documents.

## Usage

```
/discuss-start <topic> [--leader-name <name>]
```

## Parameters

- `topic` (required): The discussion topic
- `--leader-name` (optional): Custom name for the leader (default: auto-assigned from name pool)

## How It Works

When you invoke this skill, I will:

1. **Create discussion directory**: `.omc/discussions/{timestamp}-{topic-slug}/`
2. **Initialize shared memory**: namespace `discussion-{id}` with metadata
3. **Assign leader name**: From pool (Alice, Bob, Carol, David, Eve, Frank, Grace, Henry, Iris, Jack)
4. **Create stage document**: `stage-1.md` with participant section
5. **Set up initial state**: Ready for other agents to join

## Example

```
/discuss-start "优化登录流程"
```

This creates:
- Directory: `.omc/discussions/2026-04-19T10-00-00-optimize-login/`
- Leader: Alice (auto-assigned)
- Document: `stage-1.md` with Alice's participant info

## Output

After initialization, you'll see:
- Discussion ID
- Leader name
- Document location
- Next steps (how to invite agents and start rounds)

## Next Steps

After starting a discussion:
1. Invite other agents to join by sharing the discussion ID
2. Start the first round with `/discuss-round <discussion-id>`
3. Agents can join at any time during the discussion

## Storage

**Shared Memory Keys** (namespace: `discussion-{id}`):
- `meta`: Discussion metadata (topic, stage, round, status, participants, leader, speaking_lock)
- `messages`: Array of all messages with timestamps

**File Structure**:
```
.omc/discussions/{discussion-id}/
  ├── stage-1.md
  ├── stage-2.md (created when advancing)
  └── ...
```

## Leader Role

The leader (product director role) has special responsibilities:
- Posts initial topic and questions
- Can conclude stages with `/discuss-conclude`
- Makes final decisions on consensus vs disagreements
- Decides whether to end or proceed to next stage
- Maintains focus on user experience and first principles thinking
