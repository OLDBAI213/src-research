---
title: LLM-Wiki 深度研究
type: 项目解剖
source: nashsu/llm_wiki
created: 2026-05-20
tags: [llm-wiki, knowledge-graph, louvain, rag, human-in-the-loop, obsidian]
---

# LLM-Wiki 深度研究：基于知识图谱的自演化知识库系统

> 项目地址：https://github.com/nashsu/llm_wiki
> 作者：nashsu
> 基础概念：Andrej Karpathy 的 [LLM Wiki Pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
> 技术栈：Tauri v2 + React 19 + TypeScript + Rust（后端）

---

## 一、项目概览

LLM Wiki 是一个跨平台桌面应用，将你的文档自动转化为有组织、互相关联的知识库。核心理念是：**LLM 增量式构建并维护一个持久的 wiki，而不是每次查询都重新检索和生成。**

### 与传统 RAG 的本质区别

| 维度 | 传统 RAG | LLM Wiki |
|------|----------|----------|
| 知识形态 | 扁平化的文档块检索 | 结构化的互连 Wiki 页面 |
| 查询方式 | 每次从原始文档重新检索 | 从已编译的知识库查询 |
| 知识积累 | 无状态——每次查询独立 | 有状态——知识持续累积 |
| 理解深度 | 局部匹配加 LLM 合成 | 全局关联的持久化理解 |
| 维护成本 | 低（无需维护知识结构） | 中（LLM 自动维护） |
| 引文质量 | 引用原始片段 | 引用已整理的知识页面 |
| 跨文档关联 | 隐式（语义相似度） | 显式（Wiki 链接 + 知识图谱） |

**Karpathy 原话：** "Wiki 是一种持久的、复利增长的产物。交叉引用已经就位，矛盾已经被标记，综合已经反映了你读过的所有内容。"

---

## 二、增量式 Wiki 构建流程

### 2.1 两阶段 Chain-of-Thought 注入

nashsu 的实现将 Karpathy 的单步注入拆分为两步顺序 LLM 调用，大幅提升了注入质量：

#### 阶段1：分析（Step 1——LLM 调用 #1）

- **温度**: 0.1 | **max_tokens**: 4096
- 系统提示要求：不输出推理链或内省记录

分析内容包括：
- 关键实体（人物、组织、产品）
- 关键概念（理论、方法、技术）
- 主要论点和发现
- 与现有 Wiki 的关联
- 矛盾与张力（contradictions & tensions）
- 对 Wiki 结构的建议

#### 阶段2：生成（Step 2——LLM 调用 #2）

- **温度**: 0.1 | **max_tokens**: 8192
- 严格输出格式：必须以 `---FILE:` 开头

生成的内容：
1. 源文档摘要页 -> `wiki/sources/<basename>.md`
2. 实体页 -> `wiki/entities/`
3. 概念页 -> `wiki/concepts/`
4. 更新的索引 -> `wiki/index.md`
5. 日志条目 -> `wiki/log.md`
6. 更新的概览 -> `wiki/overview.md`

### 2.2 合并策略

- `log.md`：**追加**内容
- `index.md` / `overview.md`：**整体覆盖**
- 其他页面：**合并**——frontmatter 数组字段（sources, tags, related）联合合并，正文通过 LLM 调用合并

### 2.3 安全防护

- **路径穿越防护**：`isSafeIngestPath()` 拒绝绝对路径、`..` 段、Windows 驱动器号等
- **并发锁**：同一项目同时只能有一个注入操作
- **语言校验**：逐文件检查正文语言是否匹配配置语言
- **备份机制**：覆盖前保存原始内容到 `.llm-wiki/page-history/`

### 2.4 缓存系统

- 源内容哈希对比——命中缓存则跳过 LLM 管线
- 图片标注有 SHA-256 键值缓存（`.llm-wiki/image-caption-cache.json`）

---

## 三、知识图谱与四信号关联模型

这是 nashsu 实现相对于 Karpathy 原始概念的最重要扩展之一。

### 3.1 图谱数据结构

```typescript
interface GraphNode {
  id: string
  label: string
  type: string        // entity | concept | source | synthesis | query
  path: string
  linkCount: number   // 入链 + 出链
  community: number   // Louvain 社区 ID
}

interface GraphEdge {
  source: string
  target: string
  weight: number      // 四信号综合关联分数
}
```

### 3.2 四信号关联模型

| 信号 | 权重 | 含义 |
|------|------|------|
| Direct Link（直接链接） | ×3.0 | 页面间通过 `[[wikilinks]]` 互相关联 |
| Source Overlap（源重叠） | ×4.0 | 共享相同原始文档（通过 frontmatter `sources[]` 判定） |
| Common Neighbor / Adamic-Adar | ×1.5 | 共享共同邻居（按邻居度数加权——`1/log(degree)`） |
| Type Affinity（类型亲和） | ×1.0 | 同类型页面间的加分（如实体↔实体、概念↔概念） |

#### 类型亲和矩阵

```
entity   → concept: 1.2, entity: 0.8, source: 1.0, synthesis: 1.0, query: 0.8
concept  → entity: 1.2, concept: 0.8, source: 1.0, synthesis: 1.2, query: 1.0
source   → entity: 1.0, concept: 1.0, source: 0.5, query: 0.8, synthesis: 1.0
query    → concept: 1.0, entity: 0.8, synthesis: 1.0, source: 0.8, query: 0.5
synthesis→ concept: 1.2, entity: 1.0, source: 1.0, query: 1.0, synthesis: 0.8
```

#### 综合分值公式

```
totalScore = (forwardLinks + backwardLinks) × 3.0
           + sharedSourceCount × 4.0
           + sum(1/log(degree_of_shared_neighbor)) × 1.5
           + typeAffinityValue × 1.0
```

**设计洞察**：Source Overlap 权重最高（×4.0），因为共享源是最强的语义信号——两个页面如果引用同一篇论文，它们几乎不可能无关。Adamic-Adar 按度数加权是图论经典算法，能有效避免"中心页"的过度干扰。

### 3.3 图谱构建流程

1. **文件发现**：扫描 `{projectPath}/wiki/` 下的 `.md` 文件
2. **元数据提取**：title、type（frontmatter）、wikilinks（正则 `\[\[([^\]|]+?)(?:\|[^\]]+?)?\]\]`）
3. **过滤**：排除 `type: "query"` 的节点（中间产物）
4. **边去重**：确保每条无向边只出现一次
5. **关联加权**：通过四信号模型计算权重
6. **社区检测**：Louvain 算法在加权图上运行

### 3.4 检索时的图谱扩展

查询管线（Phase 2）用图谱做扩展：
- 以 Top 搜索结果作为种子节点
- 用四信号关联模型找到相关页面
- **2 跳遍历**（2-hop traversal）带衰减

---

## 四、Louvain 社区检测

### 4.1 算法细节

使用 `graphology-communities-louvain` 库，运行在无向加权图上：

```typescript
// 创建无向图
const graph = new UndirectedGraph()

// 从批处理添加节点和边...

// 运行 Louvain 社区检测
const { assignments, communities } = detectCommunities()
```

### 4.2 社区信息

```typescript
interface CommunityInfo {
  id: number
  nodeCount: number
  cohesion: number       // 社区内边密度
  topNodes: string[]     // 按 linkCount 排序的前 5 个节点
}
```

### 4.3 聚类质量指标

- **内聚度（cohesion）** = `实际社区内边数 / 可能边数`
  - `可能边数 = n × (n-1) / 2`（完全图）
  - 高内聚度 = 紧密的知识簇
- **按节点数降序排列**社区
- **重编号**社区 ID 为连续序列（0, 1, 2...）

### 4.4 图谱洞察

构建图谱后系统提供：
- **Surprising Connections**：来自不同社区但共享源的页面——跨学科交叉点
- **Knowledge Gaps**：无人指向的孤立页面——知识盲区
- **交互式探索**：点击导航、悬停查看详情

---

## 五、人类在环审核系统（Review System）

### 5.1 审核项生成

在注入管线的 Step 4，LLM 解析 `---REVIEW: type | Title---` 标记，生成以下类型的审核项：

| 类型 | 含义 |
|------|------|
| `contradiction` | 与现有 Wiki 内容矛盾 |
| `duplicate` | 实体/概念可能已存在 |
| `missing-page` | 重要概念缺少专属页面 |
| `suggestion` | 进一步研究的建议 |

### 5.2 自动清理（Sweep Reviews）

`src/lib/sweep-reviews.ts` 实现在注入队列空时自动审核：

**第一阶段：规则匹配（快速）**
- **Missing Page**：从审核项提取候选页面名 → 检查是否已存在于 Wiki 索引中 → 自动解决
- **Duplicate**：检查相关页面是否已被删除 → 自动解决

**第二阶段：LLM 语义判断（昂贵）**
- **批处理**：每次 40 项，最多 5 批
- **LLM 提示**中包含当前 Wiki 页面列表 + 待审核项
- **保守原则**："只有确信问题已解决时才标记为已解决。对于矛盾、确认或需人工判断的项，默认保持待审核。"
- **响应格式**：`{"resolved": ["id1", "id2"]}`

### 5.3 安全防护

- **项目切换保护**：每次异步操作后检查当前项目是否匹配
- **中止信号**：通过 abort signal 传递到所有异步操作
- **三处检查点**：开始前、构建索引后、每轮循环中

### 5.4 人类参与的位置

系统设计者明确倾向于"在关键决策点引入人类判断"，而不是让 LLM 全自动处理：
- 矛盾项需要人类判断
- 确认项（confirm）留给人类
- LLM 只自动解决明显已过时/已解决项

---

## 六、完整查询管线（4 阶段）

### 第 1 阶段：分词搜索
- 英文：单词拆分 + 停用词移除
- 中文：CJK 二元分词（bigram tokenization）
- 标题匹配加分（+10）
- 同时搜索 `wiki/` 和 `raw/sources/`

### 第 1.5 阶段：向量语义搜索（可选）
- 支持任何 OpenAI 兼容的 `/v1/embeddings` 端点
- 存储在 LanceDB（Rust 后端）中，支持快速 ANN 检索
- 余弦相似度
- 召回率从 58.2% 提升到 71.4%

### 第 2 阶段：图谱扩展
- Top 搜索结果作为种子节点
- 四信号关联模型 + 2 跳遍历 + 衰减

### 第 3 阶段：预算控制
- 可配置上下文窗口：4K → 1M tokens
- 比例分配：60% Wiki 页面、20% 聊天历史、5% 索引、15% 系统

### 第 4 阶段：上下文组装
- 带编号的页面 + 完整内容
- 系统提示包含：purpose.md、语言规则、引用格式、index.md
- LLM 按编号引用页面：`[1]`、`[2]` 等

---

## 七、目的文件（Purpose.md）——Wiki 的灵魂

nashsu 引入的另一个重要创新：`purpose.md` 定义 Wiki 存在的 **原因**——目标、关键问题、研究范围。

Karpathy 原始模式有 Schema 但没有正式的目的定义。Purpose.md 让 LLM 在分析和生成时有一个"北极星"，显著提升了注入质量。

---

## 八、与 Obsidian 知识库（E:\小白知识库）的整合分析

### 8.1 直接适配可能性

LLM Wiki 的核心数据模型是 **Markdown 文件 + YAML frontmatter + Wikilinks**，这与 Obsidian 完全兼容。一个 Wiki 仓库本质上就是一个 Obsidian Vault。

**可以直接复用的组件：**

| 组件 | 复用方式 |
|------|----------|
| 四信号关联模型 | 可直接移植，计算 Obsidian 笔记间的关联 |
| Louvain 社区检测 | 可对现有 1000+ 笔记进行聚类分析 |
| 两阶段注入 | 可作为 Obsidian 自动整理新笔记的管线 |
| 审核系统 | 标记矛盾/重复，建议合并 |

### 8.2 需要适配的地方

| 差异 | LLM Wiki | 小白知识库 | 适配方案 |
|------|----------|------------|----------|
| 页面类型 | entity/concept/source/synthesis | 自由分类 | 自动推断类型或扩展类型系统 |
| 目录结构 | `wiki/entities/`, `wiki/concepts/` 等 | `研究/`, `项目解剖/` 等 | 保留原有目录，通过 frontmatter 标记类型 |
| 源管理 | 严格的 sources[] 追踪 | 部分笔记有来源记录 | 对旧笔记回填 sources[] |
| 注入过程 | LLM 自动读写 | 需要人类确认 | 启用审核队列 |

### 8.3 推荐实施路径

**阶段 1——知识图谱分析（纯可视化）**
- 从现有 vault 构建初始知识图谱
- 运行 Louvain 社区检测，发现知识簇
- 标记孤立页面（Knowledge Gaps）
- 发现跨簇意外关联（Surprising Connections）
- 产出：图谱可视化 + 报告

**阶段 2——增量注入管线**
- 选择 1-2 个研究主题（如"AI Agent 记忆系统"）
- 配置 purpose.md
- 对新添加的笔记启动 LLM Wiki 注入
- 自动生成摘要页、实体页、概念页
- 启动审核队列，人工确认

**阶段 3——全量整合**
- 对历史笔记做批量注入
- 建立统一的索引和概览
- 启用 Deep Research 自动补全知识盲区
- 将审核系统作为质量保障的持续流程

### 8.4 技术整合方案

**方案 A：独立运行 + 周期同步**
- LLM Wiki 作为独立应用运行
- 设置工作目录指向 Obsidian vault
- LLM Wiki 的注入结果直接写入 Obsidian 笔记
- Obsidian Graph View 天然兼容 wikilinks

**方案 B：作为 Obsidian 插件**
- 将 nashsu/llm_wiki 的核心逻辑封装为 Obsidian 插件
- 直接复用 Obsidian 的编辑器、图谱视图、标签系统
- 用 Obsidian API 替换 Tauri 的文件操作

**方案 C：脚本桥接**
- 用 Python/Node.js 脚本调用 LLM API
- 读取 Obsidian vault 中的笔记
- 执行分析、生成、合并
- 将结果写回 vault

### 8.5 预期效果

- **自动发现**笔记间的隐性关联（≠ Obsidian 的显式链接）
- **自动聚类**研究主题（Louvain 社区）
- **自动检测**知识盲区（孤岛笔记）
- **自动生成**综合页和索引页
- **人类在环**审核确保质量

---

## 九、局限性

1. **LLM 成本**：每次注入两次 LLM 调用，token 消耗较大
2. **冷启动**：空 Wiki 没有上下文，首次注入质量较低
3. **中英文混排**：CJK 分词器对混合文档的支持有限
4. **大规模扩展**：README 提到适合 ~100 个源、数百页规模，更大规模可能需调整
5. **依赖 LLM 能力**：分析准确性依赖于 LLM 的指令跟随能力
6. **知识锁定**：生成的 Wiki 可能强化 LLM 的偏见或错误
7. **合并冲突**：多源同时注入时可能出现语义冲突

---

## 十、总结

nashsu/llm_wiki 是对 Karpathy LLM Wiki Pattern 的最完整实现，在原始概念基础上增加了大量实质性扩展：

- ✅ 两阶段 Chain-of-Thought 注入管线
- ✅ 四信号知识图谱关联模型
- ✅ Louvain 社区检测和知识聚类
- ✅ 人类在环审核系统
- ✅ 向量搜索 + 图谱检索混合管线
- ✅ 多格式文档支持
- ✅ purpose.md 驱动注入质量
- ✅ 深色模式、KaTeX 数学渲染、浏览器剪藏插件

**与小白知识库的整合是完全可行的**，建议从阶段 1（知识图谱分析）开始，在不修改任何笔记的前提下体验核心价值，再逐步推进到自动注入。
