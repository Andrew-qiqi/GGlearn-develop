# Spec: GGlearn Reader Chapter Visual Redesign (Editorial Style)

**Goal:** 优化 GGlearn 阅读器中 Chapter 头的视觉呈现，解决标题字号过大及由于使用卡片容器导致的层级模糊问题。

**Context:** 目前 Chapter 标题和 Intro 被包裹在巨大的白色阴影卡片中，在长篇阅读中与普通 Section 卡片形式过于雷同，且占据过多视觉空间。

---

## 1. 视觉设计原则

- **破壳而出**：移除卡片容器，让章节头部与页面背景融为一体，打破物理边界。
- **社论美感**：采用不对称排版和精细的衬线体比例，模拟高级学术期刊或精装书籍的导读页。
- **层级分明**：通过留白和特定的缩进，而非投影和边框，来界定章节的开始。

## 2. 具体调整内容

### 2.1 结构调整 (ReaderView.tsx)
- **容器变更**：
  - 移除 `bg-white`, `rounded-*`, `border`, `shadow-paper-lg`。
  - 移除背景模糊圆圈装饰 `blur-3xl`。
- **布局重构**：
  - 将 `Chapter {index}` 标签改为纯数字编号 `01.`, `02.` 等，并与标题成组布局。
  - 使用 `flex` 或 `grid` 实现标题与编号的对齐。

### 2.2 样式参数
- **Chapter Number**:
  - Font: `font-serif`
  - Style: `italic`
  - Size: `text-3xl md:text-5xl`
  - Color: `#AFB3B0` (或现有的 `text-[#AFB3B0]`)
- **Chapter Title**:
  - Font: `font-serif`
  - Size: `text-3xl md:text-5xl` (下调自 `text-7xl`)
  - Color: `#1A1A1A`
  - Leading: `leading-[1.1]`
  - Decoration: 下方 `h-1 w-16 bg-[#1A1A1A]`
- **Chapter Intro**:
  - Indent: `pl-12 md:pl-20`
  - Max Width: `max-w-2xl`
  - Font Size: `text-lg` (约 18px)
  - Color: `text-[#5F6368]`
  - Line Height: `leading-relaxed`

### 2.3 间距与分界
- **Top Margin**: `pt-24 md:pt-32` (增加新章节开始的仪式感)。
- **Intro to Section Divider**: 增加一个显著的间距 `mt-24`，并考虑添加一个轻量的视觉符号（如居中的三圆点）。

## 3. 技术实施建议

- 修改 `GGlearn/src/views/ReaderView.tsx` 中的章节渲染逻辑。
- 保持 `chapter.sections.map` 内部的 `SectionRenderer` 不变，因为这些原本就是知识卡片，需要保留阴影和边框。
- 确保在移动端下缩进和字号能够自适应缩小。

## 4. 验收标准

- Chapter 头部不再有白色投影卡片。
- Chapter 标题明显小于之前，但通过编号和留白仍具备强烈的视觉统治力。
- Intro 部分具有明显的导读性质，字号略大于正文卡片内的文字。
- 页面整体阅读节奏感提升，章节切换点的识别度增强。
