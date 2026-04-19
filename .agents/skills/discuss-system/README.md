# Multi-Agent Discussion System

A collaborative discussion framework where multiple agents (Claude, Codex, Gemini, etc.) can join dynamically and participate in structured, asynchronous conversations recorded in local project files.

## Overview

This system enables AI agents to hold structured discussions similar to human meetings, with:
- **Dynamic participation**: Agents can join at any time
- **Speaking lock mechanism**: Prevents concurrent speaking conflicts
- **Multi-stage progression**: Discussions can evolve through multiple stages
- **Cross-model support**: Works with any agent that can read and write the local discussion files
- **Persistent documentation**: All discussions saved as readable markdown

## Skills

### `/discuss-start` - Initialize Discussion
Create a new discussion meeting with a leader.

**Usage:**
```bash
/discuss-start "topic description" [--leader-name Name]
```

**Example:**
```bash
/discuss-start "优化登录流程"
```

### `/discuss-round` - Participate in Discussion
Join and contribute to an ongoing discussion.

**Usage:**
```bash
/discuss-round <discussion-id> [--name Name] [--role "role"] [--background "background"]
```

**Example:**
```bash
/discuss-round 2026-04-19T10-00-00-optimize-login --role "技术架构" --background "熟悉后端系统"
```

### `/discuss-conclude` - Conclude Stage
Leader summarizes and decides next steps (end or proceed to next stage).

**Usage:**
```bash
/discuss-conclude <discussion-id>
```

## Quick Start

1. **Start a discussion:**
   ```bash
   /discuss-start "Should we redesign the dashboard?"
   ```

2. **Invite agents to join:**
   Share the discussion ID with other agents (Claude, Codex, Gemini)

3. **Agents participate:**
   ```bash
   /discuss-round 2026-04-19T10-00-00-redesign-dashboard --role "UX Designer"
   ```

4. **Leader concludes:**
   ```bash
   /discuss-conclude 2026-04-19T10-00-00-redesign-dashboard
   ```

## Architecture

### Storage Structure
```
.omc/discussions/
  └── {timestamp}-{topic-slug}/
      ├── meta.json
      ├── messages.json
      ├── stage-1.md
      ├── stage-2.md
      └── ...
```

### Local State
- `meta.json`: Discussion metadata (stage, round, participants, leader, speaking_lock)
- `messages.json`: Array of all messages with timestamps
- `stage-N.md`: Human-readable discussion record

### Speaking Lock
Prevents concurrent speaking:
1. Agent checks whether `.speaking.lock/` exists
2. Acquires lock by creating `.speaking.lock/`
3. Writes `holder` and `timestamp` inside the lock directory
4. Generates and writes message
5. Releases lock by removing `.speaking.lock/`
6. Auto-timeout: 60 seconds

### Participant Names
Auto-assigned from pool: **Alice, Bob, Carol, David, Eve, Frank, Grace, Henry, Iris, Jack**

## Document Format

```markdown
# 讨论：优化登录流程

## 参与者

### Alice (Leader)
- 角色：产品总监
- 入会时间：2026-04-19 10:00
- 背景：负责整体产品方向，关注用户体验

### Bob
- 角色：技术架构
- 入会时间：2026-04-19 10:05
- 背景：熟悉后端系统和性能优化

### Carol
- 角色：UX设计
- 入会时间：2026-04-19 10:12
- 背景：专注交互设计和用户研究

---

## Stage 1: 需求分析

### Round 1
**[Alice]** 2026-04-19 10:00
最近收到很多用户反馈登录体验差，主要问题是响应慢...

**[Bob]** 2026-04-19 10:05
从技术角度看，当前瓶颈在于数据库查询...

**[Carol]** 2026-04-19 10:12
用户研究显示，除了速度，登录流程的步骤也偏多...

### Round 2
**[Bob]** 2026-04-19 10:20
我建议引入Redis缓存来优化查询性能...

**[Alice]** 2026-04-19 10:25
@Bob 成本如何？需要多少资源？

---

## 总结
**[Alice]** 2026-04-19 10:40

### 共识
- 当前登录流程确实存在性能问题
- 需要同时优化技术性能和用户体验

### 分歧
- Redis vs 本地缓存方案待定
- 是否需要完全重构流程

### 决策
- 先做技术预研，评估两种方案
- 采用渐进式优化，避免大规模重构

### 待办
- Bob 负责缓存方案调研
- Carol 设计简化流程原型
- 下周一前完成预研

---

## 进入下一阶段
**[Alice]** 2026-04-19 10:45
讨论进入 Stage 2
```

## Discussion Flow

### Typical Stage Progression

**Stage 1: Problem Analysis**
- Clarify the problem
- Understand user needs
- Identify constraints

**Stage 2: Solution Design**
- Propose approaches
- Evaluate tradeoffs
- Select direction

**Stage 3: Implementation Planning**
- Break down tasks
- Assign responsibilities
- Define timeline

**Stage 4: Review & Refinement**
- Review decisions
- Address concerns
- Finalize plan

## Leader Role

The leader (product director) has special responsibilities:

1. **审核所有方案** - Review all proposals objectively
2. **用户体验思维** - Think from UX perspective
3. **第一性原理** - Distinguish real vs fake requirements
4. **高审美追求** - Maintain high quality standards
5. **果断决策** - Make clear decisions
6. **推动进展** - Keep discussion focused

## Cross-Model Support

This system works across different AI models:

- **Claude** (native in Claude Code)
- **Codex** (via `/ask codex`)
- **Gemini** (via `/ask gemini`)

All models read/write the same local `.omc/discussions/` files, enabling multi-model collaboration without OMX runtime integration.

## Best Practices

### For Participants
- Read all previous messages before speaking
- Provide unique perspective (technical, UX, security, etc.)
- Reference specific points from others
- Be concise but thorough
- Challenge assumptions constructively
- Propose concrete solutions

### For Leaders
- Set clear agenda for each stage
- Encourage diverse viewpoints
- Synthesize discussions objectively
- Make decisions when consensus isn't possible
- Keep discussions focused and moving forward
- Document decisions clearly

### When to Conclude a Stage
- All participants have expressed views
- Discussion becoming repetitive
- Key points thoroughly explored
- Natural breakpoint reached
- Ready to move to next phase

## Technical Details

### Local State Operations
- Read `meta.json` before contributing.
- Read `messages.json` when structured history is needed.
- Append the human-facing contribution to the current `stage-N.md`.
- Update `messages.json` and `meta.json` after writing.

### Lock Acquisition Logic
- Acquire by creating `.speaking.lock/`; directory creation is the lock operation.
- Write `holder` and `timestamp` files inside the lock directory.
- If the timestamp is older than 60 seconds, treat the lock as stale and replace it.
- Release by removing `.speaking.lock/` after state and markdown writes are complete.

## Troubleshooting

### Lock Timeout
If an agent holds the lock for >60 seconds, it's automatically released.

### Concurrent Writes
The lock directory prevents normal concurrent writes. If manual edits still conflict, preserve all contributions and resolve the markdown order explicitly.

### Missing Participants
Participants are registered on first speak. If someone doesn't show up in the list, they haven't spoken yet.

### Stage Transitions
Only the leader can create new stages via `/discuss-conclude`.

## Examples

See individual skill files for detailed examples:
- `discuss-start.md`
- `discuss-round.md`
- `discuss-conclude.md`

## Future Enhancements

Potential improvements:
- Vote mechanism for decisions
- @mention notifications
- Discussion templates
- Export to other formats
- Integration with task systems
- Real-time collaboration UI
