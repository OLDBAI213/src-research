# PentAGI 深度研究

> 研究日期：2026-05-19
> 来源：github.com/vxcontrol/pentagi (12,500+ Stars, MIT License)
> 最新版本：v1.2.0 (2026-02-25)
> 前置关联：[Pentest-Swarm-AI架构分析](./Pentest-Swarm-AI架构分析.md)、[SRC挖洞实战指南](../SRC挖洞实战指南.md)

---

## 一、项目总览

PentAGI（Penetration testing Artificial General Intelligence）是 VXControl 开发的**全自主多Agent渗透测试系统**，2025年初发布，2026年初爆发（12,500+ Stars, 1,600+ Forks）。

**官方定位**：""Fully autonomous AI Agent that performs complicated penetration testing tasks using terminal, browser, editor, and external search system.""

**核心差异**：不是扫描器（如Nessus/Nuclei），而是一个**能推理、能规划、能自主决策的AI红队**。

---

## 二、架构深度分析

### 2.1 层次结构

PentAGI 采用 **Flows → Tasks → Subtasks → Actions** 四层层级，不同于传统的单Agent ReAct循环。

```
用户目标
    │
    ▼
┌─────────────────────────────┐
│     Orchestrator Agent      │  规划全局策略
│  (管理/协调所有子Agent)      │
└──────────┬──────────────────┘
           │
    ┌──────┼──────────┬──────────────┐
    ▼      ▼          ▼              ▼
 Researcher  Developer  Executor    Adviser
 (搜索)     (编码)    (执行)      (监督)
    │      │          │              │
    ▼      ▼          ▼              ▼
 外部搜索  工具调用   容器执行    反思/纠错
```

### 2.2 13+ 专业Agent角色

| Agent角色 | 职责 | 核心能力 |
|-----------|------|---------|
| **Manager / Orchestrator** | 规划任务、分派工作、协调全局 | 任务分解、资源调度 |
| **Searcher** | 信息收集、OSINT、DNS查询 | 调用Tavily/Perplexity/Google等7个搜索源 |
| **Developer / Coder** | 编写漏洞利用脚本、生成代码 | Python/Go/Bash代码生成 |
| **Pentester** | 执行扫描和漏洞利用 | 调用Nmap/Metasploit/sqlmap等20+工具 |
| **Installer** | 管理工具依赖 | 在Docker沙箱内安装工具 |
| **Adviser** | 策略指导和专业建议 | 使用更强模型做推理规划 |
| **Reflector** | 反思分析、识别失败模式 | 检测死循环、建议方向调整 |
| **Enricher** | 数据补全和上下文丰富 | 搜索补充缺失信息 |
| **Generator/Refiner** | 生成/改进漏洞报告 | 输出Markdown/JSON格式报告 |

### 2.3 三层记忆系统

| 记忆层 | 存储技术 | 用途 |
|--------|---------|------|
| **长程记忆** | PostgreSQL + pgvector | 跨会话的知识积累 |
| **工作记忆** | 当前会话状态 | 活跃目标和系统状态 |
| **情节记忆** | 行为-结果记录 | 成功模式识别 |

### 2.4 知识图谱（可选）

基于 **Neo4j + Graphiti** 的时序知识图谱，建立CVE、漏洞利用、配置之间的语义关联。
- 默认关闭（需要OpenAI API密钥做实体抽取）
- 启用后增强多目标、多会话间的关联分析能力

### 2.5 上下文管理（链式摘要）

PentAGI 的链式摘要系统是其核心技术之一：

- **全局摘要器**：最近50KB保留原文，更早对话压缩为QA对
- **助手摘要器**：更大保留空间（75KB），保持推理连续性
- 最大支持 ~200,000 tokens（取决于LLM提供商限制）

**对我们的启示**：SRC挖洞涉及长时间多目标侦察，链式摘要机制直接解决""扫着扫着忘了前面发现""的问题。

### 2.6 监督机制

| 机制 | 状态 | 效果 |
|------|------|------|
| **执行监控**（Execution Monitor） | Beta | 对小模型(<32B)效果显著，2x质量提升，但2-3x耗时/Token消耗 |
| **智能任务规划** | Beta | 小模型(<32B)建议开启，配合Adviser增强效果 |
| **工具调用限制** | 始终启用 | `MAX_GENERAL_AGENT_TOOL_CALLS` 和 `MAX_LIMITED_AGENT_TOOL_CALLS` |
| **Reflector集成** | 始终启用 | 通过 `done` / `ask` 关键词实现自我反思 |

---

## 三、LLM提供商支持

### 3.1 支持的提供商

通过 LiteLLM 实现12+ 提供商支持：

| 类型 | 提供商 | 说明 |
|------|--------|------|
| **商业API** | OpenAI, Anthropic, Google Gemini, AWS Bedrock, DeepSeek | 开箱即用 |
| **聚合平台** | OpenRouter, DeepInfra | 多模型选择 |
| **自托管** | Ollama, vLLM | 完全本地运行 |
| **自定义** | 任何兼容OpenAI API的端点 | 通过YAML配置 |

### 3.2 本地部署（核心关注点）

PentAGI **完全支持离线运行**，官方有单独文档讲解此配置。

**推荐方案：vLLM + Qwen3.5-27B-FP8**

| 配置 | VRAM | 上下文 | 吞吐量 | 成本估算 |
|------|------|--------|--------|---------|
| 4× RTX 5090 (128GB) | 128 GB | 262k (原生) | ~13K tok/s (输入) / ~650 tok/s (生成) | ~$50,000 硬件 |
| 1× H100 (80GB) | 80 GB | 262k | 略低于4×5090 | ~$30,000/GPU |
| 2× RTX 5090 (64GB) | 64 GB | ≤131k | 可用 | ~$25,000 硬件 |
| Ollama (Qwen3-8B/14B) | 8-16 GB | 32k-65k | 有限 | 已有硬件即可 |

**关键发现**：Qwen3.5是阿里云出品，**对中文场景天然支持**，且27B参数足够完成复杂渗透测试推理。

### 3.3 Agent-模型分配策略

PentAGI 一个重要设计：**不同Agent使用不同模型配置**：

- **Adviser（顾问）**：启用思考模式（thinking mode），使用最强模型做策略规划
- **Pentester/Developer**：使用低温度(0.6)精确模式做编码和执行
- **Searcher/Enricher**：使用非思考模式(0.7温度)做快速搜索
- **Reflector**：非思考模式做快速反思分析

这意味着我们不需要所有Agent都用同一个强模型，**可以混合使用**——顾问用Claude/GPT-4o，执行Agent用Qwen本地模型。

---

## 四、安全工具集成

### 4.1 核心工具（20+）

| 工具 | 用途 | 执行方式 |
|------|------|---------|
| Nmap | 端口/服务扫描 | Docker沙箱内执行 |
| Metasploit | 漏洞利用框架 | Docker沙箱内执行 |
| sqlmap | SQL注入检测/利用 | Docker沙箱内执行 |
| Nikto | Web漏洞扫描 | Docker沙箱内执行 |
| Gobuster/Dirbuster | 目录/文件发现 | Docker沙箱内执行 |
| Hydra | 暴力破解 | Docker沙箱内执行 |

### 4.2 智能链式调用

PentAGI 的关键创新：**工具链自动编排**。

```
Nmap扫描发现443端口开放Web服务
    → 自动触发Nikto扫描
    → 发现路径后调用Gobuster做目录枚举
    → 发现注入点后调用sqlmap
    → 尝试Metasploit利用模块
```

这和 Pentest-Swarm-AI 的黑板模式不同——PentAGI 是**Orchestrator驱动的主动编排**，Pentest-Swarm-AI是**黑板触发式被动响应**。

### 4.3 外部搜索（7个引擎）

Tavily, Traversaal, Perplexity, DuckDuckGo, Google Custom Search, **Sploitus**(漏洞利用搜索), SearXNG + 独立网页抓取器

**对我们SRC场景的意义**：PentAGI能自动搜索Sploitus查找最新公开漏洞利用，这对挖0day/1day非常有价值。

### 4.4 沙箱安全

| 安全措施 | 详情 |
|---------|------|
| 运行用户 | nobody (UID 65534)，永不root |
| 文件系统 | 只读根文件系统 |
| 能力限制 | `cap_drop: ALL`，必要时只加`NET_RAW` |
| seccomp | 自定义配置文件限制系统调用 |
| 资源限制 | 默认1 CPU, 512MB RAM |
| 网络隔离 | 只允许访问授权目标 |

**人类审批门**：shell执行、漏洞利用、提权、资源删除需要300秒内人工确认（超时自动拒绝）。

---

## 五、部署模型

### 5.1 系统要求

| 资源 | 最低 | 推荐 |
|------|------|------|
| vCPU | 2 | 8+ |
| 内存 | 4 GB | 16+ GB |
| 磁盘 | 20 GB | 100+ GB（含工具镜像） |
| 环境 | Docker + Docker Compose | Docker + WSL2 (Windows) |

### 5.2 部署方式

**方式一：交互式安装器（推荐）**
```bash
curl -fsSL https://raw.githubusercontent.com/vxcontrol/pentagi/main/scripts/install.sh | sudo bash
sudo ./pentagi-installer
```

**方式二：手动Docker Compose**
```bash
docker compose up -d
```
Web界面：`https://localhost:8443`

**方式三：生产分布式部署**
- 双节点架构：控制节点 + Worker节点
- Worker节点独立服务器隔离不受信任的代码执行
- 见：[Worker Node Setup](https://github.com/vxcontrol/pentagi/blob/main/examples/guides/worker_node.md)

### 5.3 完整技术栈

| 组件 | 技术 | 角色 |
|------|------|------|
| 后端 | Go + GraphQL/REST | 业务逻辑、Agent编排 |
| 前端 | React + TypeScript | 监控/控制界面 |
| 数据库 | PostgreSQL + pgvector | 持久化存储、向量搜索 |
| 知识图谱 | Neo4j + Graphiti | 语义关系 |
| 监控 | Grafana + VictoriaMetrics + Jaeger + Loki | 仪表盘、链路追踪 |
| LLM分析 | Langfuse + ClickHouse | 模型交互分析 |
| 缓存 | Redis | 限流、会话状态 |
| 对象存储 | MinIO | S3兼容存储 |
| 执行 | Docker (沙箱) | 攻击操作隔离 |

### 5.4 对SRC场景的部署建议

```
┌─────────────────────────────────────┐
│  控制节点 (低配置, 4vCPU/8GB)       │
│  - Go后端 + React前端               │
│  - PostgreSQL + Neo4j               │
│  - Grafana监控栈                    │
└────────────────┬────────────────────┘
                 │
┌────────────────▼────────────────────┐
│  Worker节点 (GPU, 可选)             │
│  - 执行Agent容器                    │
│  - 安全工具镜像                    │
│  - 可选：本地LLM (vLLM+Qwen)       │
└─────────────────────────────────────┘
```

**关键结论**：如果只使用商业API（如Claude/GPT-4o），Worker节点不需要GPU。如果需要完全本地/离线，需要4×RTX 5090或类似配置。

---

## 六、成本分析

### 6.1 API使用成本

PentAGI 没有官方成本数据，但可以根据 Escape 的基准测试推算：

| 测试场景 | 耗时 | Token估算 | 模型 | 成本估算 |
|---------|------|-----------|------|---------|
| Duck Store (20漏洞) | 4小时 | ~500K-1M tokens | DeepSeek v3.2 | ~$3-10 |
| 单目标SRC轻扫 | 1-2小时 | ~200K-500K tokens | GPT-4o-mini | ~$1-3 |
| 单目标深度渗透 | 4-8小时 | ~1M-3M tokens | Claude Sonnet 4.6 | ~$10-30 |
| 多目标SRC批量 | 8-24小时 | ~5M-10M tokens | 混合策略 | ~$20-80 |

**关键成本控制手段**：
1. **使用执行监控**：质量×2但成本×2-3，建议仅在深度测试时开启
2. **混合模型策略**：顾问用强模型 + 执行用弱模型
3. **Prompt缓存**：Claude支持缓存，Recon/分类阶段节省40-60%成本
4. **链式摘要**：自动压缩历史上下文，防止Token浪费
5. **工具调用限制**：防止Agent在死循环中浪费Token

### 6.2 本地部署成本

| 方案 | 硬件成本 | 电费/月 | 推理速度 | 适合场景 |
|------|---------|---------|---------|---------|
| 4×RTX 5090 (Qwen3.5-27B) | ~¥350,000 | ~¥2,500 | 快 | 重度使用、离线 |
| 1×RTX 5090 (Qwen3-14B) | ~¥25,000 | ~¥600 | 中等 | 个人主力 |
| RTX 4090 (Qwen3-8B) | ~¥15,000 | ~¥400 | 有限 | 入门体验 |
| 仅API调用（无GPU） | ¥0 | ¥0 | 依赖API | 轻量使用、试水 |

### 6.3 与传统渗透测试对比

| 方式 | 单次Web应用测试 | 年成本(12次) |
|------|---------------|-------------|
| 传统人工渗透测试 | ¥30,000-150,000 | ¥360,000-1,800,000 |
| PentAGI (Claude API) | ¥70-200 | ¥840-2,400 |
| PentAGI (本地Qwen) | ¥5-20（电费） | ¥60-240 |
| 对比节省 | **99.8%+** | **99.8%+** |

---

## 七、实际效果评估

### 7.1 Escape基准测试（最重要数据）

2026年4月，Escape团队对5款AI渗透测试工具进行了对比测试：

**测试环境**：Duck Store（FastAPI + React，20个漏洞，含业务逻辑场景）
**测试条件**：灰盒（URL + OpenAPI规格 + 默认凭据），单轮运行

**结果**：

| 工具 | 模型 | 发现 | 得分 | 时长 | 误报率 |
|------|------|------|------|------|--------|
| Escape (商业) | 专有 | 15/20 | **75%** | 4h | **6.25%** |
| Claude Code | Claude Opus 4.6 | 14/20 | 70% | **10min** | 6.67% |
| **PentAGI** | **DeepSeek v3.2** | **9/20** | **45%** | **4h** | **10%** |
| Shannon | DeepSeek v3.2 | 6/20 | 30% | 6h | 25% |
| Strix | DeepSeek v3.2 | 1/20 | 5% | 2h | 0% |

**核心发现**：
- Shannon、Strix、PentAGI都用的DeepSeek v3.2，但得分分别是6/20 vs 9/20 vs 1/20——**编排层比模型本身更重要**
- PentAGI在注入（SQLi/SSRF等）和业务逻辑方面表现最好（Open Source组）
- PentAGI完全没找到XSS——这是一个已知短板
- 误报率10%，在可接受范围内但需要人工校验

**对我们的意义**：PentAGI是目前开源AI渗透测试工具中**综合效果最好**的，45%检出率已接近商业化工具Escape的75%。

### 7.2 实际使用反馈

来自 GitHub Discussion #217（内部安全团队评估）：

**优点**：
- 1-5小时内给出初步测试结果，速度远超人工
- 适用于批量、持续的自动化初步评估
- 攻击链覆盖率在弱模型上也有不错表现

**不足**：
- 误报率10-30%（取决于配置）
- 业务逻辑漏洞理解有限
- 漏洞验证不够深入（理论风险 > 实际利用验证）
- 严重性评估有时偏颇

**维护者回应**：
> ""公测版评估是合理的。PentAGI不是实验原型，是团队持续开发的产品。目标：增大自主性，在不同模型上获得稳定结果。""

### 7.3 社区真实评价

- **Rafay Baloch（知名安全研究员）**：""PentAGI最好作为渗透测试者、DevSecOps团队和漏洞赏金猎人的**力量倍增器**，不是替代品。""
- **RedSecLabs**：""自动化处理已知剧本很好，但将发现转化为有意义的攻击链的创造性、上下文性、对抗性思维仍然需要人类。""
- **AppSec Santa研究**：""实验室到现实的差距巨大——GPT-4能利用87%的已知CVE，但Agent只解决了13%的真实CVE。""

---

## 八、与Shannon和Pentest-Swarm-AI的架构对比

### 8.1 三维对比矩阵

| 维度 | PentAGI | Pentest-Swarm-AI | Shannon |
|------|---------|-----------------|---------|
| **架构模式** | 多Agent + 规划器 | 蜂群（黑板+信息素） | 白盒+浏览器自动化 |
| **编排方式** | Orchestrator主导 | 涌现式自组织 | 顺序流水线 |
| **Stars** | 12,500+ | ~500+ | 39,000+ |
| **语言** | Go + React | Go + Next.js | Python (Anthropic SDK) |
| **工具数量** | 20+ | 8 (计划扩展) | 浏览器DOM + 源码分析 |
| **本地模型** | ✅ vLLM/Ollama | ✅ Ollama/LM Studio | ❌ 仅Claude API |
| **白盒分析** | ❌ 不侧重 | ❌ 不侧重 | ✅ 核心能力 |
| **知识图谱** | ✅ Neo4j+Graphiti | ❌ 仅有pgvector | ❌ 无 |
| **三层记忆** | ✅ | ✅ (黑板模式) | ❌ 仅有会话 |
| **外部搜索** | ✅ 7个引擎 | ❌ 无 | ❌ 无 |
| **误报率** | ~10% | 未知 | ~25% (基准测试) |
| **检出率** | 45% (基准测试) | 无公开基准 | 30% (基准测试) |
| **授权** | MIT | AGPL-3.0 | AGPL-3.0 |
| **部署复杂度** | 高（多容器） | 中（单二进制） | 中（Python环境） |

### 8.2 架构风格差异的本质

**PentAGI = 集中式智能**
- 一个Orchestrator Agent做全局规划
- Specialist Agent按分配执行子任务
- 类似企业组织架构：CEO→部门经理→员工
- **优势**：全局最优、策略一致
- **劣势**：单点瓶颈、Orchestrator的上下文窗口是约束

**Pentest-Swarm-AI = 分布式智能**
- 无中心调度器，Agent通过黑板自组织
- 信息素机制模拟自然界蜂群行为
- 类似自由市场：个体独立决策、整体涌现
- **优势**：弹性高、无单点故障、天然并行
- **劣势**：全局规划弱、难以保证覆盖完整性

**Shannon = 专家系统**
- 聚焦白盒分析：读源码找漏洞
- 浏览器自动化验证：动态确认
- 更像一个有AI辅助的资深白盒审计师
- **优势**：精确度高、零误报承诺
- **劣势**：需要源码、覆盖面窄（不擅长黑盒/网络层）

### 8.3 对我们SRC场景的适配度评估

| 需求 | PentAGI | Pentest-Swarm-AI | Shannon |
|------|---------|-----------------|---------|
| **中文目标适配** | ✅ (Qwen中文模型) | ⚠️ (需适配) | ⚠️ (英文为主) |
| **业务逻辑漏洞** | ⚠️ (有基础能力) | ❌ (不侧重) | ✅ (源码分析可发现) |
| **黑盒扫描** | ✅ (20+工具) | ✅ (8个工具) | ❌ (需要源码) |
| **SRC报告生成** | ✅ (Generator/Refiner) | ✅ (Report Agent) | ❌ (需自定义) |
| **批量自动化** | ✅ (CI/CD API) | ✅ (GitHub Action) | ❌ (交互式为主) |
| **成本控制** | ⚠️ (依赖配置) | ✅ (Go单二进制) | ⚠️ (Claude API贵) |
| **离线运行** | ✅ (vLLM+Qwen) | ✅ (Ollama) | ❌ (必须Claude) |
| **学习门槛** | ❌ 高 | ⚠️ 中 | ⚠️ 中 |

### 8.4 综合结论

**选择PentAGI作为核心框架的理由**：

1. **完整度最高**：13+ Agent、三层记忆、知识图谱、链式摘要——是三个项目中架构最完整的
2. **中文友好**：Qwen3.5是阿里云模型，中文理解天然优势
3. **基准测试领先**：45%检出率在开源组最高
4. **MIT许可证**：最宽松，商用改无忌讳
5. **生态丰富**：Langfuse分析、Grafana监控、CI/CD集成

**需要补充的短板**：
1. XSS检测能力弱——需结合专门的XSS扫描工具
2. 业务逻辑漏洞不足——需Shannon的源码分析方式补充
3. 部署复杂——需要Docker和一定运维能力

---

## 九、与现有研究体系的整合方案

### 9.1 现有成果的复用

| 现有文件 | 与PentAGI的关联 |
|---------|----------------|
| [SRC挖洞实战指南](../SRC挖洞实战指南.md) | PentAGI可作为自动化执行层，指南中的方法论作为Agent提示词 |
| [SRC挖洞具体操作手册](../SRC挖洞具体操作手册.md) | 工具使用流程可映射为PentAGI的Agent工作流 |
| [SRC挖洞真实案例集](../SRC挖洞真实案例集.md) | 案例库可作为PentAGI知识图谱的种子数据 |
| [SRC法律风险与合规指南](../SRC法律风险与合规指南.md) | PentAGI的范围限制（Scope）配置需符合合规要求 |
| [小白成为白客的能力评估](../小白成为白客的能力评估.md) | PentAGI为能力不足时的""外挂""方案 |
| [Pentest-Swarm-AI架构分析](../项目解剖/Pentest-Swarm-AI架构分析.md) | 蜂群黑板模式可借鉴到PentAGI的Agent通信中 |

### 9.2 差异化构建计划

基于 PentAGI 框架，构建 **XiaoBai-SRC-Hunter** 系统：

```
阶段一：基础设施（2周）
├── 部署PentAGI（Docker Compose）
├── 配置本地Qwen3.5（通过API或Ollama）
├── 导入现有工具链（sqlmap/nmap/nuclei配置）
└── 连接国内SRC情报源（补天/漏洞盒子情报）

阶段二：SRC适配（3周）
├── 编写Agent提示词（中文SRC场景优化）
├── 配置Scope验证（只扫授权目标）
├── 定制报告模板（中文+SRC格式要求）
└── 集成XSS专项检测（补充PentAGI短板）

阶段三：知识积累（持续）
├── 历史案例导入知识图谱
├── 成功/失败模式记录
├── 业务逻辑漏洞场景库
└── 成本-收益分析看板（Langfuse）

阶段四：自动化工作流（2周）
├── 批处理脚本（多目标夜间自动扫描）
├── 结果自动分流（高危→人工确认）
├── CI/CD集成（配合目标更新）
└── 报告自动生成与提交
```

### 9.3 关键风险与缓解

| 风险 | 缓解措施 |
|------|---------|
| PentAGI误报率10-30% | 搭建人工确认流程、编写自动去重脚本 |
| API成本不可控 | 设置Agent Token预算、优先本地模型 |
| 法律风险 | Scope严格限制、只扫授权目标、记录完整审计日志 |
| XSS检测死角 | 集成专门的XSS扫描器（如XSStrike） |
| 业务逻辑盲区 | 人工提供业务流程图作为Agent参考输入 |

---

## 十、最终决策

### 推荐方案：PentAGI 为主框架 + Shannon 源码分析能力借鉴

**理由**：
1. PentAGI 是目前开源AI渗透测试工具中**综合能力最强的**——架构完整、检出率最高、生态最好
2. Qwen3.5-27B 的**中文能力天然适合国内SRC场景**
3. MIT许可证允许**自由修改和商用**
4. Escape基准测试证明其**编排层优于其他同类工具**

**不选Pentest-Swarm-AI的原因**：
- 蜂群架构新颖但成熟度不足（Alpha状态）
- 工具链只有8个，远少于PentAGI的20+
- 无公开基准测试数据
- AGPL-3.0许可证对商业使用限制更严

**不选Shannon的原因**：
- 必须使用Claude API，不能本地/离线运行
- 需要源码输入，不适合黑盒SRC场景
- 检出率仅30%，低于PentAGI的45%
- 误报率25%，高于PentAGI的10%

### 下一步行动

1. ✅ 已完成：[PentAGI深度研究]（本文）
2. ✅ 已完成：[Pentest-Swarm-AI架构分析](./Pentest-Swarm-AI架构分析.md)
3. 🔲 待开始：部署PentAGI本地环境（Docker + 配置LLM Provider）
4. 🔲 待开始：编写SRC场景Agent提示词
5. 🔲 待开始：搭建知识图谱（导入现有研究案例）
6. 🔲 待开始：配置Langfuse监控+成本分析

---

> *本研究的核心结论：PentAGI是我们SRC自动化猎人系统最合适的基座框架。但AI只做第一轮筛选，人工确认和业务逻辑分析仍然不可替代。正确的定位是：**让PentAGI做80%的体力活，人类专注20%的创造性漏洞发现**。*

> 关联阅读：[Pentest-Swarm-AI工具调用详解](./Pentest-Swarm-AI工具调用详解.md)、[Pentest-Swarm-AI代码深度分析](./Pentest-Swarm-AI代码深度分析.md)
