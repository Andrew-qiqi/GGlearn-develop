# Phase 1: 技术清理与基础架构

## 目标

删除不需要的模块（Clerk、Cloudflare Workers），建立MVP纯本地技术栈

## 背景

根据产品需求讨论，MVP阶段：
- ❌ 不需要用户账号系统（删除Clerk）
- ❌ 不需要云端部署（删除Cloudflare Workers）
- ✅ 纯本地应用，使用IndexedDB存储
- ✅ 简化技术栈，聚焦核心功能

## 任务清单

### 1.1 删除Clerk认证 (预计1天)

#### 背景
Clerk用于用户认证和账号管理，但MVP阶段不需要多用户功能，所有数据存本地。

#### 任务
- [ ] 删除依赖
  - `@clerk/clerk-react`
  - `@clerk/backend`
- [ ] 删除文件
  - `src/lib/auth/clerk.tsx`
  - `src/lib/auth/clerk.test.tsx`（如果存在）
- [ ] 修改入口文件
  - `src/main.tsx`：移除`ClerkAppProvider`包裹
- [ ] 更新设置组件
  - 检查`src/components/settings/`中是否有账号相关UI
  - 删除或注释掉账号相关部分
- [ ] 更新测试
  - 删除Clerk相关的测试文件
  - 更新引用Clerk的测试

#### 验证
```bash
npm run lint
npm test
npm run build
```

---

### 1.2 删除Cloudflare Workers (预计0.5天)

#### 背景
Cloudflare Workers用于边缘计算部署，但MVP是纯前端应用，不需要。

#### 任务
- [ ] 删除配置文件
  - `wrangler.toml`
  - `edgeone.json`（如果存在）
- [ ] 删除依赖
  - `@cloudflare/vite-plugin`
  - `@cloudflare/vitest-pool-workers`
  - `@cloudflare/workers-types`
  - `wrangler`
- [ ] 删除目录
  - `src/worker/`
- [ ] 删除测试配置
  - `vitest.worker.config.ts`
- [ ] 简化package.json scripts
  - 删除`deploy`、`deploy:cf`、`dev:cf`、`build:cf`
  - 保留`dev`、`build`、`preview`

#### 验证
```bash
npm install
npm run dev
```

---

### 1.3 本地存储架构 (预计1天)

#### 背景
删除Clerk后，需要本地存储方案。使用IndexedDB替代内存存储，支持大量数据和持久化。

#### 数据模型

```typescript
// 教科书
interface TextbookRecord {
  id: string;
  title: string;
  mode: 'deep' | 'exam' | 'topic';
  sourceType: 'upload' | 'search';
  sourceContent: string;
  createdAt: number;
  updatedAt: number;
}

// 章节
interface ChapterRecord {
  id: string;
  textbookId: string;
  title: string;
  level: number;
  order: number;
  parentId: string | null;
  isGenerated: boolean;
  createdAt: number;
}

// Chunk
interface ChunkRecord {
  id: string;
  chapterId: string;
  type: 'content' | 'note' | 'handwriting' | 'image' | 'ai-explanation' | 'quiz' | 'exercise';
  order: number;
  data: any; // 根据type不同，data结构不同
  createdAt: number;
  updatedAt: number;
}
```

#### 任务
- [ ] 创建`src/lib/storage/db.ts`
  - 定义IndexedDB schema
  - 封装CRUD操作
  - 错误处理和重试
- [ ] 创建`src/lib/storage/types.ts`
  - 定义存储相关类型
- [ ] 更新`src/store/textbookStore.ts`
  - 使用IndexedDB替代内存存储
  - 保持API不变（对外接口不变）
- [ ] 添加数据备份功能
  - 导出所有数据为JSON
  - 从JSON恢复数据

#### 验证
```typescript
// 测试代码
import { db } from './lib/storage/db';

// 创建教科书
const textbook = await db.textbooks.create({
  title: '测试教科书',
  mode: 'deep',
  sourceType: 'upload',
  sourceContent: '测试内容',
});

// 读取
const loaded = await db.textbooks.get(textbook.id);
console.assert(loaded.title === '测试教科书');

// 更新
await db.textbooks.update(textbook.id, { title: '更新后的标题' });

// 删除
await db.textbooks.delete(textbook.id);
```

---

### 1.4 技术选型验证 (预计0.5天)

#### 背景
后续Phase需要用到的关键技术，提前验证可行性。

#### 手写库选型

**候选方案**：
1. **Excalidraw**
   - 优点：开源、手感好、功能完整
   - 缺点：体积较大
2. **Tldraw**
   - 优点：轻量、性能好
   - 缺点：定制性较弱
3. **Fabric.js**
   - 优点：成熟、可定制
   - 缺点：需要自己实现工具栏

**验证任务**：
- [ ] 创建`tests/handwriting-poc/`目录
- [ ] 分别集成三个库的基础示例
- [ ] 测试手写流畅度
- [ ] 测试工具栏实现难度
- [ ] 测试导出功能
- [ ] 选定最终方案

#### OCR库选型

**候选方案**：
1. **Tesseract.js**
   - 优点：开源、免费
   - 缺点：数学公式识别较弱
2. **Mathpix**
   - 优点：数学公式识别强
   - 缺点：收费、需要API key

**验证任务**：
- [ ] 测试Tesseract.js识别手写文字
- [ ] 测试Tesseract.js识别数学公式
- [ ] 调研Mathpix定价和API限制
- [ ] 决定：优先Tesseract.js，数学公式识别作为可选增强

#### 搜索API选型

**候选方案**：
1. **Google Custom Search API**
   - 优点：结果质量高
   - 缺点：每天100次免费，超出收费
2. **Bing Search API**
   - 优点：有免费额度
   - 缺点：结果质量略低

**验证任务**：
- [ ] 申请Google Custom Search API key
- [ ] 申请Bing Search API key
- [ ] 测试搜索质量
- [ ] 对比定价
- [ ] 决定：优先Google，Bing作为备选

#### PDF生成库选型

**候选方案**：
1. **jsPDF**
   - 优点：纯前端、轻量
   - 缺点：排版能力有限
2. **Puppeteer**
   - 优点：排版能力强（基于Chrome）
   - 缺点：需要后端、体积大
3. **html2pdf.js**
   - 优点：基于jsPDF，支持HTML
   - 缺点：复杂布局支持有限

**验证任务**：
- [ ] 测试jsPDF生成简单PDF
- [ ] 测试html2pdf.js生成复杂布局
- [ ] 测试手写内容嵌入
- [ ] 决定：优先html2pdf.js，满足MVP需求

---

## 风险与应对

### 风险1：删除Clerk后测试失败
**应对**：
- 先运行测试，记录哪些测试依赖Clerk
- 逐个修复或删除
- 确保核心功能测试通过

### 风险2：IndexedDB兼容性问题
**应对**：
- 使用成熟的IndexedDB封装库（如Dexie.js）
- 添加降级方案（localStorage）
- 测试主流浏览器

### 风险3：技术选型验证耗时过长
**应对**：
- 每个库限时2小时验证
- 优先验证核心功能（手写流畅度、OCR准确率）
- 不追求完美，满足MVP即可

---

## 成功标准

### 代码质量
- ✅ 删除所有Clerk和Cloudflare相关代码
- ✅ 通过`npm run lint`
- ✅ 通过`npm test`
- ✅ 通过`npm run build`

### 功能完整性
- ✅ IndexedDB存储正常工作
- ✅ 教科书创建、读取、更新、删除正常
- ✅ 数据持久化（刷新页面后数据仍在）

### 技术选型
- ✅ 手写库选定
- ✅ OCR库选定
- ✅ 搜索API选定
- ✅ PDF生成库选定

---

## 时间安排

| 任务 | 预计时间 | 负责人 |
|------|---------|--------|
| 1.1 删除Clerk | 1天 | Codex |
| 1.2 删除Cloudflare | 0.5天 | Codex |
| 1.3 IndexedDB存储 | 1天 | Codex |
| 1.4 技术选型验证 | 0.5天 | Codex |
| **总计** | **3天** | |

---

## 下一步

Phase 1完成后，进入Phase 2: 资料输入与AI搜集
