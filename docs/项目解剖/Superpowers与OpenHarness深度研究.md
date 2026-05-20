---
date: 2026-05-19
type: research
tags:
  - superpowers
  - OpenHarness
  - Hermes
  - 架构分析
---

# Superpowers & OpenHarness 深度研究

> 对我们Hermes/小白的实际价值分析

---

## 一、obra/superpowers（196k★）

### 核心理念

"技能是强制工作流，不是建议。" Agent在执行任何任务前**必须**检查相关skill，包括回答问题之前。

### 关键设计

#### 1. 强制Skill检查（using-superpowers）

这是最重要的设计。规则：
- 每次对话开始，先检查是否有匹配的skill
- **包括回答问题之前**——不是"做完事再检查"，是"开始做之前就检查"
- 如果有skill，**必须按skill执行**，不能跳过
- 如果想跳过，说明你在"合理化"——这是red flag，必须停下

对比我们现在：skill是"可选的"，我经常忘记用。superpowers的思路是"不检查skill就不准开口"。

#### 2. Red Flag检测

superpowers定义了一套"red flag"——当Agent开始有这些想法时，说明在合理化，必须停下：
- "这个任务太简单了，不需要skill"
- "我已经知道怎么做了"
- "先做了再说"
- "用户催了，先跳过检查"

这跟老白对我的要求一致："你又在生成最能接受的答案，而不是真实的判断"

#### 3. Verification Before Completion

声称"做完了"之前，必须：
- 运行验证命令
- 确认输出符合预期
- 有证据才能说"完成"

对比我们：经常说"搞定了"但没验证。

#### 4. 7步标准工作流

brainstorming → git worktree → writing plans → subagent development → TDD → code review → finish branch

每一步都是强制的，不是"建议"。

### 对我们的价值

| 设计 | 我们现在 | 学了之后 |
|------|---------|---------|
| 强制Skill检查 | skill可选，经常忘 | 每次任务前自动检查 |
| Red Flag | 没有 | 自动检测"合理化"行为 |
| Verification | 经常说"搞定了"没验证 | 必须验证后才能说完成 |

---

## 二、HKUDS/OpenHarness

### 核心理念

"模型是Agent，代码是harness。" harness提供手（工具）、眼睛（搜索）、记忆、安全边界。

### 架构（14个子系统）

```
engine/        → Agent循环：查询→流式→工具调用→循环
tools/         → 43个工具
skills/        → 按需加载的skill
plugins/       → 扩展：命令、钩子、Agent、MCP
permissions/   → 安全：多级权限、路径规则、命令拒绝
hooks/         → 生命周期：PreToolUse/PostToolUse
commands/      → 54个命令
mcp/           → MCP客户端
memory/        → 跨会话持久记忆
tasks/         → 后台任务管理
coordinator/   → 多Agent协调
prompts/       → 上下文组装
config/        → 多层配置
ui/            → React TUI
```

### 关键设计

#### 1. PreToolUse/PostToolUse Hooks

工具调用前后的生命周期钩子。比如：
- 调用shell前：检查命令是否在黑名单里
- 调用文件写入前：检查路径是否在允许范围内
- 调用后：记录日志、更新状态

对比我们：Hermes有`transform_terminal_output` hook，但没有PreToolUse级别的钩子。

#### 2. Permissions系统

多级权限模式：
- 路径级规则（哪些目录可读写）
- 命令拒绝列表（哪些命令禁止执行）
- 交互式审批对话框

对我们SRC场景的价值：**预检目标是否在授权范围**——这正是老白说的"dry run"。

#### 3. Tasks后台任务管理

- `local_agent_task`：后台Agent任务
- `local_shell_task`：后台Shell任务
- `manager.py`：任务生命周期管理
- `stop_task.py`：停止任务

对比我们：Hermes有cronjob和background terminal，但没有统一的任务管理器。

#### 4. Coordinator多Agent协调

- TeamRecord：团队记录
- TeamRegistry：团队注册表
- AgentDefinition：Agent定义

对我们SRC场景的价值：如果以后要多Agent协作挖洞（一个扫信息收集、一个扫漏洞、一个写报告），这个设计可以参考。

### 对我们的价值

| 设计 | 我们现在 | 学了之后 |
|------|---------|---------|
| PreToolUse hooks | 没有 | 工具调用前自动检查scope/权限 |
| Permissions | 没有 | SRC目标白名单、命令黑名单 |
| Dry Run | 没有 | 执行前预检，告诉你ready/warning/blocked |
| Tasks管理 | cronjob分散 | 统一的任务生命周期管理 |

---

## 三、我们该学什么

### 立刻能用的（改skill/config即可）

1. **强制Skill检查** — 学superpowers的`using-superpowers`，在xiaobai skill里加一条规则：每次任务前先检查skill列表
2. **Verification规则** — 学superpowers的`verification-before-completion`，声称完成前必须验证
3. **Red Flag检测** — 学superpowers的合理化检测，当我开始"先做了再说"时自动停下

### 需要开发的（写插件/skill）

4. **PreToolUse Hook** — 学OpenHarness，在调用渗透工具前自动检查scope
5. **Dry Run模式** — 学OpenHarness，SRC挖洞前预检目标授权状态
6. **Permissions白名单** — 学OpenHarness，定义允许扫描的目标范围

### 长期参考的（架构设计）

7. **Tasks管理** — 学OpenHarness的统一任务管理器
8. **Coordinator** — 学OpenHarness的多Agent协调，为以后多Agent挖洞做准备

---

*创建：2026-05-19*
