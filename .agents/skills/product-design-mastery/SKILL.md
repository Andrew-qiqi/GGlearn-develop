---
name: product-design-mastery
description: "Senior product designer with deep expertise in world-class product aesthetics, razor-sharp product positioning, and vibe coding. Use BEFORE any feature design, UI/UX decision, or product direction discussion. Applies design philosophy from Apple, Google, Dieter Rams, and top-tier digital products to create products with exceptional taste, not feature bloat. Triggers: product planning, feature design, UI architecture, product positioning, aesthetic direction, deciding what to build vs. what to cut."
---

# Product Design Mastery

Transform product thinking from "what features to add" to "what experience to create." This skill enforces a design philosophy where every pixel, every interaction, and every feature decision serves a singular product vision.

## Core Design Philosophy

### The Subtraction Principle

Great products are defined by what they **remove**, not what they add.

- Apple didn't add features to make iPhone great — they removed the keyboard, the stylus, and the manual
- Google Search didn't add features — they removed everything except a text box
- Claude Code didn't add a GUI — they removed it entirely and bet on terminal-native flow

**Rule: Before adding any feature, ask "What can we remove instead?"**

### The Chemistry Test

A feature earns its place ONLY if it creates a "chemical reaction" — an emotional response that makes the user think "这就是我要的" (this is exactly what I needed).

Evaluate every feature through this lens:

| Signal | Chemistry ✅ | No Chemistry ❌ |
|--------|-------------|----------------|
| User reaction | "Wow, how did it know?" | "Oh, that's... nice I guess" |
| Return rate | User comes back daily | User tries once, forgets |
| Word of mouth | "You HAVE to try this" | "It's okay, it does X" |
| Replaceability | "Nothing else does this" | "I can use Y instead" |

### The 10x Bar

Do not build something that is marginally better. If the experience is not **10x better** than the current alternative, it's not worth building. A 2x improvement gets ignored. A 10x improvement changes behavior.

## Product Decision Framework

### Step 1: Identify the Pain Moment

Do NOT start with "what feature should we build." Start with:

> "What is the **specific moment** when the user feels pain/frustration/confusion?"

Describe this moment in first person, present tense:
> "I open the slide, I stare at it, I have no idea what this diagram means, and the professor's recording is 2 hours long and I don't know which part explains this."

### Step 2: Design the Magic Moment

The magic moment is the **instant** when pain transforms into delight:

> "I click one button, and in 15 seconds, the AI points at the exact diagram and explains it to me like a patient tutor."

Constraints for the magic moment:
- **≤ 1 click** to trigger
- **≤ 15 seconds** to first value
- **Zero learning curve** — no onboarding, no tutorial, no "how do I use this?"

### Step 3: Apply the Razor

For every proposed feature, run it through the Product Razor:

```
┌─────────────────────────────────────────────┐
│              THE PRODUCT RAZOR              │
│                                             │
│  1. Does it serve the magic moment?         │
│     NO → Cut it.                            │
│                                             │
│  2. Does it pass the Chemistry Test?        │
│     NO → Cut it.                            │
│                                             │
│  3. Can the user discover it without        │
│     being told?                             │
│     NO → Redesign or cut it.               │
│                                             │
│  4. If we ship without it, would users      │
│     notice AND complain?                    │
│     NO → Cut it.                            │
│                                             │
│  5. Is it 10x better than the alternative?  │
│     NO → Don't build a 2x version.         │
└─────────────────────────────────────────────┘
```

### Step 4: Design the Experience Flow

Design as a **continuous flow**, not as a set of features. The user should never think "which feature should I use?" The product should guide them through a natural sequence:

```
Trigger → Value → Confirmation → Next Step
(1 click)  (15s)   (I got it!)    (automatic)
```

**Anti-pattern:** Tabs, menus, mode switches, settings panels. These are **escape hatches** that signal the core flow failed.

## Aesthetic Principles — The Visual Language

### 1. Restrained Expressiveness
- Use **one** accent color, not a rainbow
- Typography carries emotion: weight, spacing, and scale matter more than color
- White space is a feature, not wasted space
- Motion should be purposeful — if an animation doesn't communicate state change, remove it

### 2. Information Hierarchy as UI
- The most important action on screen should be **unmistakable** without labels
- Secondary actions should be discoverable but invisible until needed
- Use progressive disclosure: show less by default, reveal on engagement
- If you need a tooltip to explain a button, the button has failed

### 3. The "Screenshot Test"
Take a screenshot of the product. Show it to someone for 3 seconds. Ask:
- What does this product do?
- What should I click first?
- Does it look expensive or cheap?

If they can't answer all three correctly, the visual design has failed.

### 4. Emotional Texture
- Micro-interactions should feel **physical** — like pressing a real button, not clicking a flat rectangle
- State transitions should feel **natural** — fade, slide, spring. Not instant-swap
- Empty states are opportunities for personality, not just "No data yet"
- Error states should be **helpful**, never blaming ("我们没找到结果" not "搜索失败")

## Anti-Patterns — What to Never Do

### Feature Graveyard
Adding features that "might be useful someday." If you can't name 3 real users who asked for it, don't build it.

### Settings Sprawl
Every setting is an admission that you couldn't make a decision. Default to the best choice. Only offer settings when genuinely divergent user needs exist.

### Dashboard Disease
Showing all data at once because "users might want to see it." Show only what triggers the next action.

### Copycat Features
Building something because a competitor has it. If your product's differentiator is X, double down on X. Don't dilute it with Y because others have Y.

### Premature Platforming
Building extensibility, plugin systems, or APIs before you have 1000 users who love the core experience. Platforms emerge from beloved products, not the other way around.

## Output Format

When applying this skill to a product decision, structure the output as:

```markdown
## Pain Moment
[First-person description of the user's pain]

## Magic Moment  
[How the product transforms pain → delight, with ≤1 click, ≤15s latency]

## Product Razor Results
| Proposed Feature | Serves Magic Moment? | Chemistry? | Discoverable? | Missed if Cut? | 10x? | Verdict |
|---|---|---|---|---|---|---|

## Recommended Experience Flow
[Trigger → Value → Confirmation → Next Step]

## What to Cut
[Features/elements to remove, with reasoning]

## Visual Direction
[Specific aesthetic guidance for this decision]
```

## Reference Sources

This skill draws philosophy from:
- **Dieter Rams** — 10 Principles of Good Design ("Less, but better")
- **Apple HIG** — Progressive disclosure, spatial consistency, clarity over decoration
- **Google Material You** — Personalized expressiveness within systematic constraints
- **Anthropic Claude** — Dense capability behind radical simplicity of interface
- **Linear** — Opinionated defaults, keyboard-first, no clutter
- **Arc Browser** — Reimagining established patterns rather than iterating on them
- **Notion** — Composable primitives over specialized tools
