---
skill: discuss-conclude
description: Leader concludes current stage with comprehensive summary and decides next steps
---

# Discuss Conclude

Leader concludes the current discussion stage with a comprehensive summary including consensus, disagreements, decisions, and action items. Then decides whether to end the discussion or proceed to the next stage.

## Usage

```
/discuss-conclude <discussion-id>
```

## Parameters

- `discussion-id` (required): The discussion ID from `/discuss-start`

## How It Works

When you invoke this skill, I will:

1. **Verify leadership**: Confirm caller is the designated leader
2. **Acquire speaking lock**: Wait for any ongoing speeches to complete
3. **Analyze discussion**: Review all messages and identify patterns
4. **Generate summary**: Create structured summary with four sections
5. **Write to document**: Append summary to current stage document
6. **Decide next step**: Ask whether to end or proceed to next stage
7. **Execute decision**: Either conclude discussion or create new stage

## Summary Structure

The leader must provide a comprehensive summary with:

### 共识 (Consensus)
Points where all participants agree or have reached alignment.

### 分歧 (Disagreements)
Unresolved conflicts, different opinions, or areas needing further discussion.

### 决策 (Decisions)
Final decisions made by the leader based on the discussion.

### 待办 (Action Items)
Next steps, tasks to be done, or questions for the next stage.

## Example Summary

```markdown
## 总结
**[Alice]** 2026-04-19 10:30

### 共识
- 当前登录流程确实存在性能问题
- 需要引入缓存机制来提升响应速度
- 用户体验应该是首要考虑因素

### 分歧
- Redis vs 本地缓存方案尚未达成一致
- 是否需要完全重构还是渐进式优化存在不同看法

### 决策
- 先进行技术预研，评估两种缓存方案的成本和收益
- 采用渐进式优化策略，避免大规模重构风险
- 下一阶段重点讨论具体实施方案

### 待办
- Bob 负责 Redis 方案的技术调研
- Carol 设计新的登录流程原型
- 下周一前完成预研报告
```

## Stage Transitions

After summary, the leader decides:

### Option 1: End Discussion
- Sets discussion status to `concluded`
- Adds final note to document
- Discussion is complete

### Option 2: Proceed to Next Stage
- Increments stage number
- Resets round counter to 0
- Creates new `stage-N.md` document
- Adds transition note to current stage
- Leader should post new topic in next stage

## Leader Responsibilities

As the leader (product director role), you should:

1. **审核所有方案** - Review all proposals objectively
2. **用户体验思维** - Evaluate from user experience perspective
3. **第一性原理** - Distinguish real needs from fake requirements
4. **高审美追求** - Maintain high quality and design standards
5. **果断决策** - Make clear decisions when consensus isn't possible
6. **推动进展** - Keep discussion focused and moving forward

## When to Conclude

Consider concluding a stage when:
- All participants have expressed their views
- Discussion is becoming repetitive
- Key points have been thoroughly explored
- A natural breakpoint has been reached
- Ready to move from analysis to solution design
- Ready to move from design to implementation planning

## Multi-Stage Discussions

Typical stage progression:

**Stage 1**: Problem analysis and requirement clarification
**Stage 2**: Solution design and approach evaluation  
**Stage 3**: Implementation planning and task breakdown
**Stage 4**: Review and refinement

Each stage should have a clear focus and deliverable.

## Notes

- Only the designated leader can invoke this skill
- Leader can force-conclude even if speaking lock is held
- Summary should be objective and comprehensive
- Decisions should be clear and actionable
- If proceeding to next stage, leader should immediately post new topic with `/discuss-round`
