# AI黑客工具研究（GitHub API实时数据）

**日期**: 2026-05-27
**数据源**: GitHub REST API（实时）
**目的**: 研究哪些对我们的SRC作业有用

---

## 项目排名（按★）

### 🥇 Shannon — 43,747★ | Forks 5,034
- **地址**: https://github.com/KeygraphHQ/shannon
- **语言**: TypeScript | 许可: AGPL v3
- **创建**: 2025-09-27 | **最后推送**: 2026-05-26（昨天！）
- **Open Issues**: 15（很少，维护良好）
- **最新Release**: v1.3.0（2026-05-26，昨天发的）
- **标签**: penetration-testing, security-audit, security-automation
- **简介**: 白盒AI渗透测试，分析源码→识别攻击向量→执行真实exploit
- **亮点**: 在OWASP Juice Shop发现20+漏洞（含认证绕过+数据泄露），零误报
- **对SRC**: ⭐⭐⭐ 需要源码，我们SRC是黑盒。但浏览器自动化exploit执行和PoC验证机制值得学

### 🥈 Strix — 25,624★ | Forks 2,862
- **地址**: https://github.com/usestrix/strix
- **语言**: Python | 许可: Apache 2.0
- **创建**: 2025-08-05 | **最后推送**: 2026-05-21
- **Open Issues**: 92
- **最新Release**: v0.8.3（2026-03-23，2个月没更新）
- **标签**: agents, cybersecurity, llm, penetration-testing
- **简介**: AI自动化漏洞发现+PoC验证，Docker沙箱隔离执行
- **亮点**: `strix --target <url>` 直接对URL做自动化测试，有SaaS平台（app.strix.ai）
- **对SRC**: ⭐⭐⭐⭐ 黑盒+动态测试+PoC验证，最接近我们需求。更新频率下降

### 🥉 PentAGI — 17,245★ | Forks 2,363
- **地址**: https://github.com/vxcontrol/pentagi
- **语言**: Go | 许可: MIT
- **创建**: 2025-01-06 | **最后推送**: 2026-05-23
- **Open Issues**: 38
- **最新Release**: v2.0.0（2026-04-11）
- **标签**: ai-agents, multi-agent-system, golang, graphql
- **简介**: 全自动渗透测试AGI系统，多Agent协作
- **亮点**: 支持9种LLM（含DeepSeek/GLM/Qwen），知识图谱集成，Langfuse可观测
- **⚠️ Issues**: 有secret泄露报告（#316），DeepSeek旧模型名兼容问题
- **对SRC**: ⭐⭐⭐ 架构参考价值高，但部署重

### 🏅 DeepAudit — 6,191★ | Forks 766
- **地址**: https://github.com/lintsinghua/DeepAudit
- **语言**: Python | 许可: AGPL v3
- **创建**: 2025-09-19 | **最后推送**: 2026-04-01（2个月没更新）
- **Open Issues**: 84（较多）
- **最新Release**: v3.0.4（2026-01-24，4个月没更新）
- **标签**: ai, code-audit, code-quality, devsecops
- **简介**: 清华团队，国内首个开源AI代码漏洞挖掘多智能体系统
- **亮点**: 21种漏洞类型，6阶段流水线，攻击记忆系统，支持Ollama私有部署
- **⚠️ Issues**: 多个未解决bug，Xiaomi API兼容问题，Azure AI Foundry需求
- **对SRC**: ⭐⭐⭐⭐⭐ 最适合我们：中文+代码审计+攻击记忆+开源。但更新停滞

### 🏅 CyberStrikeAI — 3,949★ | Forks 669
- **地址**: https://github.com/Ed1s0nZ/CyberStrikeAI
- **语言**: Go | 许可: Apache 2.0
- **创建**: 2025-11-08 | **最后推送**: 2026-05-26（昨天）
- **Open Issues**: 20
- **最新Release**: v1.6.24（2026-05-26，昨天！更新极快）
- **标签**: ai, ai-hacking, ai-penetration-testing, ctf-tools, mcp
- **简介**: AI原生安全测试平台，100+安全工具集成
- **⚠️ 警告**: 已被攻击者用于FortiGate攻击（600+设备，55国），The Hacker News 2026年3月报道
- **⚠️ Issues**: C2加密绕过漏洞（#124）
- **对SRC**: ⭐⭐ 了解架构可以，不建议直接用（恶意利用历史+安全风险）

### 🏅 AI-Infra-Guard — 3,781★ | Forks 372
- **地址**: https://github.com/tencent/AI-Infra-Guard
- **语言**: Python | 许可: Apache 2.0
- **创建**: 2024-12-25 | **最后推送**: 2026-05-26（昨天）
- **Open Issues**: 13
- **最新Release**: v4.1.9（2026-05-21）
- **标签**: agent-security, ai-red-teaming, llm-jailbreak
- **简介**: 腾讯玄武实验室，AI基础设施红队平台（Agent/Skills/MCP/LLM扫描）
- **对SRC**: ⭐⭐ 定位不同（AI系统安全），未来做AI Agent安全审计可参考

### 🏅 Pentest-Swarm-AI — 1,280★ | Forks 259
- **地址**: https://github.com/Armur-Ai/Pentest-Swarm-AI
- **语言**: Go | 许可: AGPL v3
- **创建**: 2024-03-26 | **最后推送**: 2026-05-25
- **Open Issues**: 8
- **最新Release**: v0.1.0（2026-05-07）
- **标签**: bug-bounty, penetration-testing-framework
- **简介**: 蜂群多智能体渗透测试，ReAct推理，bug bounty专项
- **对SRC**: ⭐⭐⭐ Bug bounty专项思路值得学，Swarm协作模式有启发

### 🏅 Claude-BugHunter — 1,076★ | Forks 157 ⭐新加入
- **地址**: https://github.com/elementalsouls/Claude-BugHunter
- **语言**: Python | 许可: MIT
- **创建**: 2026-05-05 | **最后推送**: 2026-05-26（昨天）
- **Open Issues**: 0
- **简介**: Claude Code skill bundle，51个技能+15个命令+681个披露报告模式
- **核心能力**:
  - 5阶段非线性猎捕工作流 + 批判性思维框架
  - 24种漏洞类型 × 每种检测模式、payload、绕过表、链模板
  - 企业攻击链：M365/Entra ID、Okta、VMware vCenter、SSL VPN设备
  - 报告模板：H1/Bugcrowd/Intigriti/免疫efi + 红队交付格式
  - 7-Question Gate质量门禁
  - Burp MCP集成
- **经过验证**: DVWA、OWASP Juice Shop、Hacker101、testphp.vulnweb.com + 授权红队实战
- **对SRC**: ⭐⭐⭐⭐⭐ **最直接有用**！Claude Code原生skill，装上就能用。681个报告模式覆盖Bugcrowd/HackerOne，正好是我们需要的

### 🏅 PHP_AUDIT_SKILLS — 47★ | Forks 2
- **地址**: https://github.com/yunmengya/PHP_AUDIT_SKILLS
- **语言**: PHP | 许可: 未声明
- **创建**: 2026-02-27 | **最后推送**: 2026-05-08
- **Open Issues**: 0
- **最新Release**: v2.0.0（2026-04-09）
- **简介**: PHP代码审计AI技能库，Claude Code Agent Teams，21种漏洞类型
- **对SRC**: ⭐⭐⭐⭐ PHP专项利器，国内SRC很多PHP目标。但依赖Claude Code

---

## 维护活跃度排名（最近推送时间）

| 项目 | 最后推送 | 最新Release | 更新频率判断 |
|---|---|---|---|
| Shannon | 2026-05-26 | v1.3.0 5月26日 | 🔥 极活跃 |
| CyberStrikeAI | 2026-05-26 | v1.6.24 5月26日 | 🔥 极活跃（但有风险） |
| AI-Infra-Guard | 2026-05-26 | v4.1.9 5月21日 | 🔥 活跃 |
| Claude-BugHunter | 2026-05-26 | 无Release | 🔥 活跃（项目才3周） |
| Pentest-Swarm-AI | 2026-05-25 | v0.1.0 5月7日 | 🟡 活跃 |
| PentAGI | 2026-05-23 | v2.0.0 4月11日 | 🟡 一般 |
| Strix | 2026-05-21 | v0.8.3 3月23日 | 🟡 放缓 |
| PHP_AUDIT_SKILLS | 2026-05-08 | v2.0.0 4月9日 | 🟠 低频 |
| DeepAudit | 2026-04-01 | v3.0.4 1月24日 | 🔴 停滞 |

---

## 对我们SRC作业的建议

### 直接可用（装上就能跑）
1. **Claude-BugHunter** — Claude Code原生skill，51个技能+681个报告模式，Bug bounty专项，黑盒也能用
2. **PHP_AUDIT_SKILLS** — PHP目标专项审计，Claude Code Agent Teams

### 借机制（框架/架构参考）
1. **攻击记忆系统**（DeepAudit）→ 我们缺这个，漏洞经验不能从零来
2. **6阶段流水线**（DeepAudit/PHP_AUDIT_SKILLS）→ 整合到SRC方法论
3. **PoC自动验证**（Strix/Shannon）→ 漏洞发现后自动生成报告
4. **Bug bounty流程**（Pentest-Swarm-AI/Claude-BugHunter）→ 可借鉴
5. **浏览器自动化exploit**（Shannon）→ 黑盒测试执行

### 不建议直接用
1. **CyberStrikeAI** — 恶意利用历史，安全风险
2. **AI-Infra-Guard** — 定位不同（AI系统安全）
3. **PentAGI** — 太重，部署依赖多

### 我们的差距
1. **报告自动化**: Claude-BugHunter有681个报告模式+VRT映射，我们还在手动写
2. **攻击记忆**: DeepAudit的记忆系统是我们缺失的
3. **对抗循环**: 8轮攻击+自动回滚的思路可以简化后用
