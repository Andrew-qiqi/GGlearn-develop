# GGlearn Reader Chapter Visual Redesign Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 重构 GGlearn 阅读器中的 Chapter 头部，采用“社论风格”排版，移除卡片容器，缩小标题字号，并优化视觉层级。

**Architecture:** 
- 移除 `ReaderView.tsx` 中 Chapter 头部外层的卡片容器样式。
- 采用不对称布局：左侧装饰性数字编号 + 衬线体标题。
- 为 Chapter Intro 增加明显的左缩进，与后文的知识卡片形成视觉区分。
- 在 Intro 结束后增加视觉分割点（三圆点）。

**Tech Stack:** React, Tailwind CSS, Lucide React (optional for decorative icons).

---

## File Map

- Modify: `GGlearn/src/views/ReaderView.tsx`
  - 重写章节头部渲染逻辑，实现社论风格。

---

## Task 1: Deconstruct Chapter Container & Implement Basic Editorial Layout

**Files:**
- Modify: `GGlearn/src/views/ReaderView.tsx`

- [ ] **Step 1: Locate and modify the Chapter wrapper style**

移除 `bg-white`, `rounded-[2rem]`, `border`, `shadow-paper-lg` 等类。增加顶部和底部的间距。

```tsx
// GGlearn/src/views/ReaderView.tsx 约 248 行处
// 修改前:
// className="scroll-mt-24 pt-16 md:pt-24 pb-20 md:pb-32 px-8 md:px-24 bg-white rounded-[2rem] md:rounded-[4rem] border border-[#E0E0DE]/50 shadow-paper-lg relative overflow-hidden group/chapter"

// 修改后:
className="scroll-mt-24 pt-24 md:pt-32 pb-16 md:pb-24 px-8 md:px-12 relative overflow-hidden group/chapter"
```

- [ ] **Step 2: Implement the Chapter Number and Title Layout**

重构标题部分，使用 `01.` 格式的数字，并调整字号。

```tsx
// GGlearn/src/views/ReaderView.tsx 约 253 行处
<div className="flex flex-col md:flex-row items-baseline gap-4 md:gap-8 mb-12">
  <span className="font-serif italic text-4xl md:text-6xl text-[#AFB3B0] leading-none opacity-80 select-none">
    {chapter.chapterIndex < 10 ? `0${chapter.chapterIndex}.` : `${chapter.chapterIndex}.`}
  </span>
  <div className="flex-1">
    <h1 className="font-serif text-3xl sm:text-4xl md:text-5xl tracking-tight text-[#1A1A1A] leading-[1.1] mb-6">
      {chapter.title}
    </h1>
    <div className="h-1 w-16 bg-[#1A1A1A] rounded-full" />
  </div>
</div>
```

- [ ] **Step 3: Refactor Chapter Intro with Indentation**

增加左缩进，并调整文字样式。

```tsx
// GGlearn/src/views/ReaderView.tsx 约 259 行处
{chapter.intro?.content ? (
  <div className="pl-0 md:pl-20 lg:pl-28 max-w-2xl text-[#5F6368] text-lg leading-relaxed opacity-95">
    <RichMarkdown content={chapter.intro.content} lowProfileTitles={true} />
  </div>
) : null}
```

- [ ] **Step 4: Verify visually (Manually)**

由于是 UI 变更，请运行项目并观察：
1. Chapter 头部是否不再有卡片背景和投影。
2. 标题是否变得更克制。
3. 编号是否正确显示（如 01., 02.）。

- [ ] **Step 5: Commit changes**

```bash
git add GGlearn/src/views/ReaderView.tsx
git commit -m "style: redesign chapter header with editorial layout and smaller titles"
```

---

## Task 2: Add Visual Divider & Spacing Polish

**Files:**
- Modify: `GGlearn/src/views/ReaderView.tsx`

- [ ] **Step 1: Add the Three-Dot Divider after Intro**

在 Chapter Intro 之后，Section 开始之前增加视觉分割点。

```tsx
// GGlearn/src/views/ReaderView.tsx 在章节头部 div 结尾处
{/* Visual Divider to Sections */}
<div className="mt-20 mb-20 md:mt-24 md:mb-24 flex justify-center gap-4">
  <div className="w-1.5 h-1.5 rounded-full bg-[#E0E0DE]" />
  <div className="w-1.5 h-1.5 rounded-full bg-[#E0E0DE]" />
  <div className="w-1.5 h-1.5 rounded-full bg-[#E0E0DE]" />
</div>
```

- [ ] **Step 2: Adjust spacing between Chapters**

确保不同章节之间有足够的呼吸空间。检查 `space-y-16 md:space-y-24` 是否足够。

- [ ] **Step 3: Run build and lint to ensure no regressions**

Run:
```bash
cd GGlearn && npm run lint && npm run build
```

Expected: PASS

- [ ] **Step 4: Commit changes**

```bash
git add GGlearn/src/views/ReaderView.tsx
git commit -m "style: add visual divider and spacing polish to chapter headers"
```
