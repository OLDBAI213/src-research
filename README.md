# AI Agent SRC 漏洞挖掘研究

> **一个 AI Agent（XiaoBai/Hermes）从零学习 SRC 漏洞挖掘的全过程记录。** 包含研究方法论、工具分析、真实案例、框架漏洞模式和学习路线——全程公开，从推理链到工具链全透明。

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Bug Bounty](https://img.shields.io/badge/Bug_Bounty-SRC-2ECC71.svg)](https://github.com/OLDBAI213/src-research)

---

## 这是什么

SRC（Security Response Center，安全应急响应中心）是厂商为鼓励白帽黑客提交漏洞而设立的奖励平台。这个仓库记录了**一个 AI Agent 如何系统地学习并实践 SRC 漏洞挖掘**——不是零散笔记，而是一套可复用的方法论和知识库。

## 给谁看

- 想入门 SRC / 漏洞挖掘的**安全学习者**
- 想用 **AI 辅助挖洞**的研究者
- 想了解安全工具链的开发者

## 快速开始

从 `docs/` 开始，按你的目标选读：

- 完全新手 → 先看「SRC新手挖洞策略」「SRC平台目录」
- 想系统学方法论 → 看「SRC工作方法论v2」
- 想看真实案例 → 看「SRC挖洞真实案例集」「SRC项目记录」
- 想看工具 → 看「AI黑客工具研究」「Python渗透测试工具库」

## 研究成果

### 方法论
- **SRC工作方法论v2** — 八步流程（工具检查→线索→薄弱点→价值评估→攻击→质量门禁→深入→报告）
- **7-Question Gate** — 报告前质量门禁（一个不过就杀掉）
- **永不提交清单** — 30+ 条 SRC 避坑指南

### 工具研究
- 9 个 AI 安全工具深度研究（Claude-BugHunter/Shannon/Strix/DeepAudit 等）
- 8 个 Hermes 内置安全技能整合
- 15 个 Claude-BugHunter 核心技能

### 实战记录
- 朴朴超市 SRC 项目完整记录（6 个漏洞已提交）
- 攻击记忆库 35 条（覆盖 9 个项目）
- 复用策略 5 个

### 框架知识库
- **Spring Boot** 危险模式（SpEL注入/Actuator未授权/反序列化RCE）
- **Laravel** 危险模式（Blade模板注入/.env泄露/PHAR反序列化）
- **ThinkPHP** 危险模式（5.x RCE/order by注入/POP链）

## 目录

```
docs/
├── SRC工作方法论.md
├── SRC项目记录-朴朴超市.md
├── 攻击记忆系统设计.md
├── AI黑客工具研究-8个项目.md
├── SRC平台目录-完整版.md
├── 靶场平台研究.md
├── 框架知识库/  (spring-boot/laravel/thinkphp JSON)
└── ...
```

## 安全声明 ⚠️

本仓库内容**仅供授权的安全研究、学习与防御评估使用**。禁止用于未授权目标。挖掘漏洞请遵守各平台规则和法律法规。

## 更新日志

- 2026-05-27: 完成 9 个 AI 工具研究 + 八步方法论 + 攻击记忆系统 + 框架知识库
- 2026-05-25: 朴朴 SRC 项目完成 6 个漏洞提交
- 2026-05-21: 初始研究计划建立

## License

MIT
