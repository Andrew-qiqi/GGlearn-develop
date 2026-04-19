---
skill: discuss-round
description: Join a discussion and contribute with automatic participant registration and speaking lock
---

# Discuss Round

Join an ongoing discussion and contribute your perspective. First-time participants are automatically registered in the document header.

## Usage

```
/discuss-round <discussion-id> [--name <name>] [--role <role>] [--background <background>]
```

## Parameters

- `discussion-id` (required): The discussion ID from `/discuss-start`
- `--name` (optional): Custom name for this agent (default: auto-assigned)
- `--role` (optional): Your role in the discussion (e.g., "技术架构", "UX设计")
- `--background` (optional): Your background/expertise

## How It Works

When you invoke this skill, I will:

1. **Read discussion state**: Load current stage, round, and all previous messages
2. **Register if new**: If first time joining, add participant info to document header
3. **Acquire speaking lock**: Wait for lock to be free (with timeout protection)
4. **Generate response**: Based on all previous messages and discussion context
5. **Write message**: Append to current stage document with timestamp
6. **Release lock**: Allow next agent to speak

## Speaking Lock Mechanism

To prevent concurrent speaking conflicts:
- Only one agent can speak at a time
- Lock shows: `{holder: "Bob", timestamp: "...", status: "发言中"}`
- Other agents see: "⏳ Bob 正在发言，等待中..."
- Auto-timeout: 60 seconds (prevents deadlock)
- Leader can force-conclude even if lock is held

## Example

```
/discuss-round 2026-04-19T10-00-00-optimize-login --role "技术架构" --background "熟悉后端系统和性能优化"
```

## First-Time Join

When joining for the first time, your info is added to the document:

```markdown
### Bob
- 角色：技术架构
- 入会时间：2026-04-19 10:05
- 背景：熟悉后端系统和性能优化
```

## Speaking Format

Your message is appended to the current stage document:

```markdown
**[Bob]** 2026-04-19 10:05
从技术角度看，当前登录流程的主要瓶颈在于...
```

## Name Assignment

If you don't specify `--name`, names are auto-assigned from pool:
- Alice, Bob, Carol, David, Eve, Frank, Grace, Henry, Iris, Jack

Names are assigned in order as agents join.

## Cross-Model Support

This skill works with any AI model:
- Claude (native)
- Codex (via `/ask codex`)
- Gemini (via `/ask gemini`)

All models read/write to the same shared memory namespace and markdown documents.

## What to Contribute

As a participant, you should:
- Read all previous messages before speaking
- Provide your unique perspective (technical, UX, security, etc.)
- Reference specific points from other participants when relevant
- Be concise but thorough
- Challenge assumptions constructively
- Propose concrete solutions when possible

## Notes

- You can join at any round, not just the beginning
- Multiple rounds of discussion are expected
- Leader will conclude each stage with a summary
- You can speak multiple times in different rounds
