# TencentDB-Agent-Memory 深度研究

> **研究目的**：为 XBMS（小白记忆系统）设计提供参考。该项目的 4 层架构与我们的设计几乎一致，且已有 Hermes 接入分支。
>
> **分析版本**：v0.3.4+
> **研究日期**：2026-05-20
> **原始仓库**：[Tencent/TencentDB-Agent-Memory](https://github.com/Tencent/TencentDB-Agent-Memory)
> **Hermes 分支**：[LIGMinN/TencentDB-Agent-Memory-hermes](https://github.com/LIGMinN/TencentDB-Agent-Memory-hermes)（与上游几乎完全一致，仅二维码图片不同）

---

## 目录

1. [项目概述](#1-项目概述)
2. [四层架构详解（L0-L3）](#2-四层架构详解l0-l3)
3. [Mermaid 符号化记忆与 Token 压缩](#3-mermaid-符号化记忆与-token-压缩)
4. [存储模型：异构存储 + 渐进式披露](#4-存储模型异构存储--渐进式披露)
5. [Pipeline 调度引擎](#5-pipeline-调度引擎)
6. [Hermes 集成分析](#6-hermes-集成分析)
7. [与 XBMS 对比分析](#7-与-xbms-对比分析)
8. [可直接复用的设计/代码](#8-可直接复用的设计代码)
9. [关键技术决策记录](#9-关键技术决策记录)

---

## 1. 项目概述

TencentDB Agent Memory 是一套**符号化短期记忆 + 分层式长期记忆**系统，面向 Agent（OpenClaw / Hermes）提供：
- **短期**：通过 Mermaid 符号图谱压缩工具调用日志，最高节省 **61.38% Token**，同时通过率提升 **51.52%**
- **长期**：通过 L0→L1→L2→L3 语义金字塔提取个性化记忆，PersonaMem 准确率从 **48%** 提升到 **76%**

**核心哲学**：
- 拒绝暴力历史堆砌（flat context accumulation）
- 拒绝不可逆的暴力摘要（irreversible lossy summarization）
- 采用 **分层 + 符号化** 设计，确保 100% 可追溯性

---

## 2. 四层架构详解（L0-L3）

### 2.1 整体架构

```
┌──────────────────────────────────────────────────────────┐
│                      L3 Persona                          │
│             persona.md（用户画像 + 场景导航）              │
├──────────────────────────────────────────────────────────┤
│                      L2 Scene                            │
│        scene_blocks/*.md（情境叙事文档 + META 元数据）    │
├──────────────────────────────────────────────────────────┤
│                      L1 Atom                             │
│        records/*.jsonl + 向量库（结构化记忆碎片）          │
│        类型：persona / episodic / instruction              │
├──────────────────────────────────────────────────────────┤
│                      L0 Conversation                     │
│        conversations/*.jsonl（原始对话消息）               │
│        + L0 向量索引（用于全文搜索）                       │
└──────────────────────────────────────────────────────────┘
     ↑                                                       ↑
  短期（Mermaid 卸载）                                    长期（语义金字塔）
```

每个层次同时包含**短期**和**长期**两条链路，共享相同的分层抽象。

### 2.2 L0 — Conversation（原始对话层）

**定位**：原始对话的不可变日志，是所有上层记忆的**溯源起点**。

**存储格式**：
- 文件：`conversations/YYYY-MM-DD.jsonl`（每日一个文件，多 session 混写）
- 单行格式：每行一条消息的 JSON，包含 `sessionKey`、`role`、`content`、`timestamp`
- 同时写入 L0 向量索引（SQLite FTS5 + embedding），支撑对话搜索

**写入时机**（`auto-capture.ts`）：
- 每个 `agent_end` 钩子触发一次
- 增量写入：使用 `originalUserMessageCount`（位置切片）和 `afterTimestamp`（时间游标）双重防重
- 安全阀：时间戳漂移检测，Gateway 重启后通过位置切片恢复

**关键设计**：
- 注入污染处理：Framework 会在用户消息中注入 `prependContext`，L0 Recorder 用缓存的原文本替换污染后的消息
- 逐消息生成 `msg_timestamp_hex` 格式 ID
- 消息过滤：清理由 `l0l1RetentionDays` 控制（0 = 永不过期）

### 2.3 L1 — Atom（结构化记忆层）

**定位**：从 L0 对话中提取的结构化"记忆碎片"，是最细粒度的可检索记忆单元。

**存储格式**：
- 文件：`records/records.jsonl`（结构性记忆记录）
- 向量库：SQLite FTS5（BM25）+ sqlite-vec（语义向量），混合检索（RRF 融合）
- 每条记录包含：`content`、`type`（persona / episodic / instruction）、`priority`、`source_message_ids`、`metadata`、`scene_name`

**提取流程**（`l1-extractor.ts`）：

```
L0 消息 → 情境切分（Scene Segmentation） → 记忆提取（Memory Extraction） → 冲突检测（Dedup） → 写入
```

1. **读取背景消息**：从 L0 加载最多 `maxBackgroundMessages` 条历史消息 + `maxMessagesPerExtraction` 条新消息
2. **LLM 结构化提取**（一次调用，JSON 模式输出）：
   - 系统提示词（`l1-extraction.ts`）：定义三大类型 + 提取原则 + 输出 JSON schema
   - LLM 返回：`[{scene_name, message_ids, memories: [{content, type, priority, source_message_ids, metadata}]}]`
3. **冲突检测**（`l1-dedup.ts`）：批量对比已有 L1 记录，用 embedding 相似度判断是否已存在
4. **写入**：新记忆写入 records JSONL + 向量库

**三大记忆类型**：

| 类型 | 定义 | 示例 | Priority 范围 |
|------|------|------|--------------|
| `persona` | 用户的稳定属性、偏好、习惯 | "用户喜欢用 Python 写脚本" | 50-100 |
| `episodic` | 客观发生的事件、决定、结果 | "用户在上周五完成了架构评审" | 60-100 |
| `instruction` | 用户对 AI 提出的长期规则 | "用户要求 AI 以后用中文回复" | 70-100（-1 为严格死命令）|

**提取原则**："宁缺毋滥"——过滤琐碎闲聊、临时性指令、一次性操作。

### 2.4 L2 — Scene（情境场景层）

**定位**：将碎片化的 L1 记忆"编织"成连贯的叙事文档（场景块），是 Persona 生成的中间层。

**存储格式**：
- 文件：`scene_blocks/*.md`（Markdown 文件）
- LLM 可直接读写（sandboxed 到 `scene_blocks/` 目录）
- 文件包含 META 元数据段：

```
-----META-START-----
created: 2026-01-15T10:30:00Z
updated: 2026-05-20T00:15:00Z
summary: 用户的技术架构与工程实践
heat: 7
-----META-END-----

正文内容...
```

**提取流程**（`scene-extractor.ts`）：

```
L1 记忆（20条一批） → 加载已有场景索引 → Scene Count 检查 → LLM agent 操作文件
```

1. LLM 被提示为"记忆整合架构师"（Memory Consolidation Architect）
2. 有完整的**文件操作工具**（read / write / edit / soft-delete）
3. 直接操作 `scene_blocks/*.md` 文件：CREATE / INTEGRATE / REWRITE / MERGE
4. 场景数量有上限（`maxScenes`，默认 15），超出时 LLM 必须执行 MERGE
5. 删除通过写入 `[DELETED]` 标记实现软删除

**场景文件命名**：中文名，如 "技术架构与工程实践.md"、"日常生活与工作节奏.md"

**场景导航**（`scene-navigation.ts`）：
- 生成场景索引 → 追加到 `persona.md` 末尾
- 格式：场景名称 + 摘要 + 文件路径，方便 Agent 通过 `read_file` 下钻

### 2.5 L3 — Persona（用户画像层）

**定位**：用户的全景画像，包含长期偏好、行为模式、关键背景。每个用户只有一个 `persona.md`。

**存储格式**：
- 文件：`persona.md`（纯 Markdown，人机可读）
- 由 LLM agent 直接写入（tools enabled，sandboxed 到 data dir）
- 末尾自动追加场景导航（`Scene Navigation` 部分）

**生成流程**（`persona-generator.ts`）：

```
加载已有画像 → 读取场景索引 → 识别变化场景 → 构建 Prompt → LLM 写入 persona.md
```

1. **增量模式**：已有 `persona.md` 时，只分析变化场景（`changedScenes`）
2. **首次模式**：无已有画像时，全量分析所有场景
3. LLM 被提示为"心理学专家 + 笔记专家"，读取场景块文件后更新画像
4. 写入完成后，系统自动追加场景导航索引

**触发条件**（`persona-trigger.ts`，5 级优先级）：

| 优先级 | 条件 | 说明 |
|--------|------|------|
| P1 | Agent 显式请求更新 | `request_persona_update` 标记 |
| P2 | 首次冷启动 | 场景文件已存在但无 Persona |
| P2.5 | 恢复模式 | Persona 此前存在但文件丢失/为空 |
| P3 | 首个场景块提取完成 | 第一次场景提取后立即触发 |
| P4 | 达到阈值 | 默认每 50 条新记忆触发一次 |

---

## 3. Mermaid 符号化记忆与 Token 压缩

### 3.1 设计动机

在长任务中，最大的 Token 消耗来源是**繁杂的过程日志**（搜索工具返回、代码段、报错）。传统做法有两条路线，但都有缺陷：
- **全量保留**：Token 暴涨，成本失控
- **暴力摘要**：丢失可追溯性，Agent 无法纠错

### 3.2 三层卸载链路

```
工具调用日志（几十万 Token）
    │
    ├──→ ① 卸载完整原文 → refs/*.md（外部文件系统）
    │
    ├──→ ② 提取步骤摘要 → jsonl（中间层，含 result_ref）
    │
    └──→ ③ 压缩成 Mermaid 图谱 → 上下文中仅保留几百 Token
                                             │
    Agent 推理时如果需要详情 ──→ grep node_id ──→ 恢复完整原文
```

### 3.3 Mermaid 图谱的 Token 压缩原理

**核心**：Mermaid（Flowchart）语法本身具有极高的信息密度。

```
graph LR
    S1["搜索用户"<br/>001-N1] --> S2["分析结果"<br/>001-N2]
    S2 --> S3["生成报告"<br/>001-N3]
    S3 -->|"输出"| S4["完成"<br/>001-N4]
```

每条节点包含：摘要 + `node_id`，边包含：关系描述。

- **压缩比**：一个 Mermaid 节点 ≈ 50-100 Token，对应原来几千 Token 的工具日志
- **语义保留度**：Mermaid 是拓扑结构，比自然语言摘要更精确地保留了依赖关系
- **LLM 可解析性**：LLM 原生理解 Mermaid 语法

### 3.4 触发条件（`l2-mermaid.ts`）

L2（Mermaid 生成）独立于 L1 触发，有两个条件：

- **条件 A**：`offload.jsonl` 中 `node_id=null` 的条目数 ≥ `l2NullThreshold`（默认值）
- **条件 B**：距离上次 L2 触发的时间 ≥ `l2TimeoutSeconds`

**边界过滤**（`boundary`）：
- `short task` → 跳过（任务较短，不需要卸载）
- `long task` → 进入卸载流程
- 同一对话时间窗口内的连续日志被绑定到同一个 MMD 中

### 3.5 `node_id` 溯源机制

```
格式：{3位前缀}-N{序号}，如 "001-N1"
```

- Mermaid 中的每个节点都有唯一 `node_id`
- `offload.jsonl` 中每条日志记录其 `node_id`
- Agent 需要查证细节时：`grep "001-N2" refs/*.md` 即可恢复完整原文
- 实现 100% 可追溯性，无信息丢失

---

## 4. 存储模型：异构存储 + 渐进式披露

### 4.1 双层存储策略

| 层级 | 存储介质 | 内容 | 特点 |
|------|---------|------|------|
| 底层 | SQLite + 文件 | L0（JSONL）、L1（JSONL+向量）、L0 向量索引 | 海量数据、支持全文/语义检索 |
| 上层 | Markdown 文件 | L2（scene_blocks/*.md）、L3（persona.md） | 高信息密度、白盒可调、人机可读 |

**原则**：**低层保留证据，高层保留结构。**

### 4.2 100% 可追溯性链路

```
高层符号（Persona / 画布）
    ↓ 导航索引
中层索引（Scene / JSONL）
    ↓ node_id / 文件名
底层原文（L0 Conversation / refs/*.md）
```

这是一条**可逆链路**，每一层都能向下钻取到更原始的证据。

### 4.3 存储后端

两种后端可选：
1. **SQLite + sqlite-vec**（默认）：本地零配置，FTS5 BM25 + 向量检索
2. **TCVDB**（Tencent Cloud Vector Database）：云端高性能向量库

支持混合检索（hybrid）：BM25 + embedding + RRF（Reciprocal Rank Fusion）融合排序。

---

## 5. Pipeline 调度引擎

### 5.1 架构

`MemoryPipelineManager`（`pipeline-manager.ts`）管理所有异步记忆处理任务。

关键配置（`pipeline` 段）：

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `everyNConversations` | 5 | 每 N 轮对话触发一次 L1 |
| `enableWarmup` | true | 预热模式（1→2→4→...→N） |
| `l1IdleTimeoutSeconds` | 600 | 空闲超时触发 L1 |
| `l2DelayAfterL1Seconds` | 90 | L1 完成后延迟多久触发 L2 |
| `l2MinIntervalSeconds` | 900 | L2 最小间隔（15分钟） |
| `l2MaxIntervalSeconds` | 3600 | L2 最大间隔（60分钟） |
| `sessionActiveWindowHours` | 24 | Session 活跃窗口 |

### 5.2 预热模式（Warmup）

新 Session 的初始阈值设为 1，每触发一次 L1 后翻倍，逐步增长到 `everyNConversations`：

```
回合: 1→2→4→8→...→everyN（10）
```

这样在对话早期密集提取（信息最多的时候），后期自然降频。

### 5.3 调度流程

```
auto_capture（钩子）
    │
    ├── 1. L0 写入（同步，ms 级）
    ├── 2. L0 向量索引（异步，后台线程）
    └── 3. 通知 PipelineManager
            │
            ├── L1 触发条件满足？
            │   ├── yes → L1 Extraction（LLM 调用）
            │   │            │
            │   │            ├── L2 触发条件满足？
            │   │            │   ├── yes → L2 Scene Extraction（LLM agent）
            │   │            │   │            │
            │   │            │   │            └── L3 触发条件满足？
            │   │            │   │                ├── yes → L3 Persona（LLM agent）
            │   │            │   │                └── no  → 等待
            │   │            └── no → 等待
            │   └── no → 等待
            │
            └── 下一轮对话 → 回到顶部
```

### 5.4 检查点（Checkpoint）

`checkpoint.json` 跟踪每个 session 的进度：

```json
{
  "total_processed": 150,
  "memories_since_last_persona": 45,
  "last_persona_at": 1716000000000,
  "scenes_processed": 12,
  "request_persona_update": false,
  "per_session_cursors": {
    "session_123": { "maxTimestamp": 1716000000000, "messageCount": 42 }
  }
}
```

---

## 6. Hermes 集成分析

### 6.1 架构概览

```
Hermes Agent（Python）
  └─ MemoryManager
       └─ MemoryTencentdbProvider（hermes-plugin/memory/memory_tencentdb/）
            ├─ GatewaySupervisor：启动/健康检查 Gateway 子进程
            └─ MemoryTencentdbSdkClient：HTTP 客户端（POST /recall, /capture 等）
                    │
                    ▼  HTTP（127.0.0.1:8420）
            memory-tencentdb Gateway（Node.js）
               └─ TdaiCore（4 层引擎）
                    ├─ L0 Conversation store（SQLite / JSONL）
                    ├─ L1 Episodic extraction（LLM + vector dedup）
                    ├─ L2 Scene blocks（Markdown）
                    ├─ L3 Persona synthesis（persona.md）
                    └─ Storage：SQLite + sqlite-vec / TCVDB
```

### 6.2 Hermes 生命周期映射

| Hermes 回调 | Gateway 端点 | 行为 |
|-------------|-------------|------|
| `prefetch(query)` | `POST /recall` | 同步，返回 `<memory-context>` 文本注入 |
| `sync_turn(user, assistant)` | `POST /capture` | 后台线程异步执行 |
| `shutdown()` / `on_session_end` | `POST /session/end` | 刷新未完成的 Pipeline 工作 |
| `get_tool_schemas()` | — | 注册两个搜索工具 |

### 6.3 部署方式

**方式 A：Docker 一体镜像**
- `Dockerfile.hermes` 构建
- 单容器运行 Hermes + Gateway
- 一键启动，自动同步模型配置

**方式 B：Symlink 载入 Hermes**
- `hermes-plugin/memory/memory_tencentdb/` → `hermes-agent/plugins/memory/memory_tencentdb/`
- Python SDK 作为 Hermes MemoryProvider 接入
- Gateway 后台子进程自动管理

### 6.4 可靠性与生产化特性

- **断路器**：连续 5 次 Gateway 失败后暂停 60s
- **背压**：最多 4 个并发的 `sync_turn` 后台线程，超时 5s
- **看门狗**：守护线程每 10s 检查 Gateway 健康，自动复活
- **自动发现**：无配置时自动在 `~/.memory-tencentdb/` 下寻找 Gateway
- **Gateway 子进程管理**：stdout/stderr 重定向到日志文件，防止 pipe buffer 阻塞

### 6.5 与 OpenClaw 版本的区别

在**代码层面**，Hermes fork（LIGMinN/TencentDB-Agent-Memory-hermes）与原始仓库**实际上是完全相同的代码**。

唯一的区别是 `README_CN.md` 中的微信二维码图片（不同图片 ID）。

真正的架构区别在于：

| 维度 | OpenClaw 版本 | Hermes 版本 |
|------|--------------|------------|
| **宿主接口** | `OpenClawPluginApi`（进程内） | `MemoryProvider`（Python 抽象） |
| **通信方式** | 进程内直接调用 | HTTP 侧车（sidecar） |
| **LLM 调用** | OpenClaw 内置 `runEmbeddedPiAgent` | Gateway 直连 LLM API 或通过 OpenClaw proxy |
| **HostAdapter** | `OpenClawHostAdapter` | `StandaloneHostAdapter`（Gateway 内部） |
| **Python SDK** | 不需要 | `memory_tencentdb/__init__.py` 完整 SDK |
| **子进程管理** | 不需要 | `GatewaySupervisor` 完整生命周期管理 |
| **断路器/看门狗** | 不需要 | Python 侧完整实现 |
| **数据目录** | `~/.openclaw/memory-tdai/` | `~/.memory-tencentdb/memory-tdai/` |
| **模型配置** | OpenClaw 模型配置 | 环境变量 `TDAI_LLM_*` 或 Hermes 配置同步 |

核心引擎（TdaiCore + 所有 L0-L3 处理逻辑）是**完全共享的**，Hermes 只是包装了一层 Python SDK 来管理 Node.js Gateway 子进程。

---

## 7. 与 XBMS 对比分析

### 7.1 架构对比

| 维度 | TencentDB-Agent-Memory | XBMS（我们的设计） | 差异分析 |
|------|----------------------|-------------------|---------|
| **层数** | 4 层（L0-L3） | 4 层（L1-L4） | 本质一致。我们 L1=对话层，L2=记忆层，L3=场景层，L4=画像层 |
| **命名** | L0=对话, L1=原子, L2=场景, L3=画像 | L1=对话, L2=记忆, L3=场景, L4=画像 | 只是偏移量不同 |
| **短期记忆** | Mermaid 符号图谱卸载 | 未设计（计划中） | 他们的 Mermaid 方案可直接借鉴 |
| **存储** | 底层 DB + 上层 Markdown | 类似设计 | 核心设计一致 |
| **检索** | BM25 + 向量 + RRF 混合 | 计划用 Redis Stack 向量 | 方案不同但目标一致 |
| **LLM 调用** | 一次性提取（JSON 模式） | 未设计 | 他们的 Prompt 设计可直接复用 |
| **符号化** | Mermaid (graph) | 我们的 Mermaid 主要在日志可视化 | 他们的 node_id 溯源方案值得借鉴 |
| **增量更新** | 增量 Persona + 增量场景 | 设计类似 | 设计理念一致 |

### 7.2 关键差异

| 方面 | TencentDB-Agent-Memory | XBMS |
|------|----------------------|------|
| **宿主** | OpenClaw/Hermes + Node.js | Hermes (Codex CLI) + 独立网关 |
| **技术栈** | TypeScript (核心) + Python (Hermes SDK) | Python (核心，待定) |
| **存储后端** | SQLite + sqlite-vec / TCVDB | Redis Stack (向量+JSON) |
| **Gateway 端口** | 8420 | 我们设计为 8421 |
| **嵌入策略** | 按对话轮次/空闲时间触发 | 我们设计为实时 + 批处理混合 |
| **冲突检测** | Embedding 相似度 | 未设计 |
| **LLM 工具** | LLM agent 直接读写场景文件 | 未设计 |
| **场景数量限制** | 默认 15 个，超出需 MERGE | 未设计 |
| **备份机制** | `.backup/` 目录自动备份 | 未设计 |

### 7.3 XBMS 的独特优势（Tencent 没有的）

1. **Redis Stack**：更轻量的向量存储方案（无需单独部署 DB）
2. **Codex CLI 适配**：作为 Codex 工具的集成方式
3. **Mermaid 日志可视化**：我们已有 XBMermaidLogger，可将 Mermaid 用于更广泛的日志可视化
4. **Hermes Codex Skill**：我们已经有 Skilled Hermes Agent 架构
5. **Feishu 飞书集成**：跨平台记忆同步

---

## 8. 可直接复用的设计/代码

### 8.1 可直接复用的设计理念

| 设计 | 理由 | 优先级 |
|------|------|--------|
| L0→L1→L2→L3 四层架构 | 与 XBMS 完全一致，可直接沿用 | 🔴 高 |
| Mermaid 符号化 + `node_id` 溯源 | Token 压缩最佳实践 | 🔴 高 |
| 异构存储（DB + Markdown） | 底层证据 + 上层结构 | 🔴 高 |
| 三大记忆类型（persona/episodic/instruction） | 分类成熟，LLM 提取精准 | 🔴 高 |
| 增量 Persona 生成 | 避免全量重算 | 🟡 中 |
| LLM agent 直接操作场景文件 | 省去复杂的场景管理逻辑 | 🟡 中 |
| Mixed retrieval (BM25 + Vector + RRF) | 搜索质量经实践验证 | 🟡 中 |
| 看门狗 + 断路器模式 | 生产级可靠性保障 | 🟡 中 |
| 预热模式（Warmup） | 对话初期密集提取，后期自然降频 | 🟢 低 |
| 场景导航索引 | 方便 Agent 下钻 | 🟢 低 |
| 场景数量上限 + MERGE 机制 | 防止场景膨胀 | 🟢 低 |

### 8.2 可直接复用的提示词（Prompt）

TencentDB 的 LLM 提示词是**中文**的，质量非常高，可直接搬运：

1. **L1 提取提示词**（`l1-extraction.ts`）：情境切分 + 三大类型提取 + JSON 格式输出
2. **L2 场景提取提示词**（`scene-extraction.ts`）：记忆整合架构师 + 文件操作指令
3. **L3 Persona 生成提示词**（`persona-generation.ts`）：心理学专家 + 笔记专家
4. **L1 去重提示词**（`l1-dedup.ts`）：冲突检测

### 8.3 可直接复用的代码概念

| 代码模块 | 语言 | 复用方式 |
|---------|------|---------|
| `TdaiCore`（核心门面） | TS | 架构模式复制 |
| `HostAdapter`（宿主抽象） | TS | 抽象接口直接采用 |
| `RuntimeContext`（运行时上下文） | TS | 类型定义复制 |
| `pipeline-manager.ts`（调度引擎） | TS | 算法逻辑翻译 |
| `persona-generator.ts`（增量画像） | TS | 逻辑复制 |
| `scene-extractor.ts`（场景提取） | TS | 架构复制 |
| `checkpoint.ts`（检查点） | TS | 状态管理复制 |
| `backup.ts`（备份机制） | TS | 文件备份复制 |
| `sanitize.ts`（文本清洗） | TS | 过滤逻辑复制 |

### 8.4 TencentDB 的工程陷阱（XBMS 应避免）

1. **全量提取成本**：L1 提取依赖 LLM 调用，每 N 轮触发一次，云上 API 成本需注意
2. **场景文件膨胀**：LLM agent 直接操作文件，可能出现格式不一致
3. **Gateway 启动延迟**：最长 30s 等待 Gateway 启动，体验不够流畅
4. **中文硬编码**：提示词全部中文，多语言场景需全量翻译
5. **SQLite 并发限制**：多个 Gateway 实例访问同一数据目录有写冲突
6. **Python-Node 通信开销**：Hermes 版本中每个操作都走 HTTP，延迟比进程内调用高

---

## 9. 关键技术决策记录

### 9.1 为什么用 JSONL 而不是 SQL 存 L0/L1？

- **JSONL 的流式特性**：可 grep、可 tail、可流式处理
- **避免绑定特定数据库**：SQLite 只是可选的向量索引层
- **人类可读**：调试时可以 `cat conversations/*.jsonl | grep session_key=xxx`
- **易于备份/迁移**：文件级别的导出导入

### 9.2 为什么 L2/L3 用 LLM agent 操作文件？

- **LLM 直接创建/编辑 Markdown**：免去实现复杂的场景管理 CRUD
- **sandbox 限制**：LLM 只能操作 `scene_blocks/` 目录，系统文件不可见
- **灵活度高**：LLM 的叙事能力远超预定义的场景格式

### 9.3 为什么 Hermes 集成走 HTTP sidecar 而非进程内？

- **语言隔离**：核心引擎是 Node.js（TypeScript），Hermes 是 Python
- **独立生命周期**：Gateway 可以独立重启，不影响 Hermes Agent
- **共用性**：同一个 Gateway 可被多个 Hermes 实例共享

### 9.4 为什么 Mermaid 而不是其他图谱格式？

- **Token 效率**：Mermaid 语法本身比 PlantUML/Graphviz 更紧凑
- **LLM 原生理解**：LLM（尤其是 Claude/GPT-4）对 Mermaid 解析准确
- **人类可读**：Mermaid 可以直接在 GitHub/Markdown 预览

---

## 附录 A：核心文件索引

| 文件 | 功能 | 说明 |
|------|------|------|
| `src/core/tdai-core.ts` | 核心门面 | 所有操作的入口 |
| `src/core/types.ts` | 类型定义 | HostAdapter、RuntimeContext 等 |
| `src/core/conversation/l0-recorder.ts` | L0 录制 | 增量对话写入 JSONL |
| `src/core/record/l1-extractor.ts` | L1 提取 | LLM 从 L0 提取结构化记忆 |
| `src/core/record/l1-dedup.ts` | L1 去重 | Embedding 相似度去重 |
| `src/core/scene/scene-extractor.ts` | L2 场景提取 | LLM agent 操作场景文件 |
| `src/core/scene/scene-format.ts` | L2 文件格式 | META 解析/格式化 |
| `src/core/scene/scene-navigation.ts` | 场景导航 | 生成导航索引 |
| `src/core/persona/persona-generator.ts` | L3 画像生成 | 增量 Persona |
| `src/core/persona/persona-trigger.ts` | L3 触发 | 5 级触发条件 |
| `src/core/hooks/auto-recall.ts` | 召回钩子 | 注入记忆到上下文 |
| `src/core/hooks/auto-capture.ts` | 采集钩子 | L0 + 通知 Pipeline |
| `src/core/prompts/l1-extraction.ts` | L1 提示词 | 情境切分+记忆提取 |
| `src/core/prompts/l1-dedup.ts` | L1 去重提示词 | 冲突检测 |
| `src/core/prompts/scene-extraction.ts` | L2 提示词 | 场景整合架构师 |
| `src/core/prompts/persona-generation.ts` | L3 提示词 | 心理学专家 |
| `src/offload/pipelines/l2-mermaid.ts` | Mermaid 生成 | Token 压缩核心 |
| `src/utils/pipeline-manager.ts` | Pipeline 调度 | L1/L2/L3 调度引擎 |
| `src/utils/pipeline-factory.ts` | 工厂函数 | 初始化所有组件 |
| `src/config.ts` | 配置类型 | 全部配置项定义 |
| `src/gateway/server.ts` | Gateway HTTP | Hermes sidecar 服务 |
| `src/core/store/sqlite.ts` | SQLite 存储 | FTS5 + sqlite-vec |
| `src/core/store/tcvdb.ts` | TCVDB 存储 | 腾讯云向量数据库 |
| `hermes-plugin/memory/memory_tencentdb/__init__.py` | Hermes SDK | MemoryProvider 实现 |
| `hermes-plugin/memory/memory_tencentdb/client.py` | HTTP 客户端 | Gateway API 包装 |
| `hermes-plugin/memory/memory_tencentdb/supervisor.py` | 进程管理 | Gateway 生命周期 |

## 附录 B：关键配置参数速查

```yaml
memory-tencentdb:
  enabled: true
  capture:
    enabled: true
    excludeAgents: []                      # 排除的 Agent 名单
    l0l1RetentionDays: 0                   # 保留天数，0=不过期
  extraction:
    enabled: true
    enableDedup: true                      # L1 去重
    maxMemoriesPerSession: 20              # 每轮最多提取条数
    model: "provider/model"                # L1 专用模型
  pipeline:
    everyNConversations: 5                 # L1 触发频率
    enableWarmup: true                     # 预热模式
    l1IdleTimeoutSeconds: 600              # 空闲触发
    l2DelayAfterL1Seconds: 90              # L1→L2 延迟
    l2MinIntervalSeconds: 900              # L2 最小间隔
    l2MaxIntervalSeconds: 3600             # L2 最大间隔
  recall:
    enabled: true
    maxResults: 5                          # 召回结果数
    scoreThreshold: 0.3                    # 最低分数
    strategy: "hybrid"                     # keyword / embedding / hybrid
  persona:
    triggerEveryN: 50                      # 每 N 条记忆触发 L3
    maxScenes: 15                          # 场景上限
    backupCount: 3                         # Persona 备份数
    sceneBackupCount: 10                   # 场景备份数
  embedding:
    enabled: true
    provider: "openai"                     # "none" 禁用向量
    baseUrl: "https://api.openai.com/v1"
    apiKey: "${EMBEDDING_API_KEY}"
    model: "text-embedding-3-small"
    dimensions: 1536
  offload:
    enabled: false                         # 短期压缩（默认关闭）
```

---

> **研究结论**：TencentDB-Agent-Memory 是当前最接近 XBMS 设计的开源记忆系统。其四层架构、异构存储策略、增量提炼流程、以及提示词设计均可在 XBMS 中直接复用。特别是其 L1 提取提示词（中文、三大类型）、Mermaid 符号化记忆方案和增量 Persona 生成机制，是 XBMS 可以优先采纳的设计。
