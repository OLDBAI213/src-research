---
date: 2026-05-19
type: learning-plan
tags:
  - 网安
  - SRC
  - 学习计划
  - 渗透测试
status: active
---

# 小白的网安SRC学习计划

> 目标：系统学习渗透测试，达到能独立在补天/漏洞盒子挖洞的水平。
> 周期：紧凑型30天集中学习 + 持续实战
> 开始日期：2026-05-19

---

## 学习原则

1. **边学边做** — 每学一个知识点，立刻在靶场/环境里验证
2. **工具先行** — 先会用工具，再理解原理（效率最高）
3. **实战导向** — 所有学习最终服务于SRC挖洞
4. **AI增强** — 用AI加速学习（问我、用AI工具辅助分析）

---

## 第一阶段：环境搭建+基础认知（第1-3天）

### 目标
搭好渗透测试环境，理解基本概念。

### 任务
- [ ] 安装VMware + Kali Linux虚拟机
- [ ] 配置Kali网络（桥接/NAT）
- [ ] 安装Burp Suite Community
- [ ] 安装nmap、sqlmap、dirsearch
- [ ] 注册补天账号 (butian.net)
- [ ] 注册漏洞盒子账号 (vulbox.com)
- [ ] 了解OWASP Top 10是什么

### 学习资源
- Kali官方文档：https://www.kali.org/docs/
- OWASP Top 10：https://owasp.org/Top10/

---

## 第二阶段：Web安全核心漏洞（第4-10天）

### 目标
掌握SRC最常见的漏洞类型。

### 每日安排

**第4天：SQL注入**
- 原理：联合注入、布尔盲注、时间盲注
- 工具：sqlmap
- 靶场：sqli-labs前10关

**第5天：SQL注入进阶**
- sqli-labs 11-30关
- sqlmap高级参数（--tamper绕过WAF）

**第6天：XSS**
- 原理：存储型、反射型、DOM型
- 靶场：DVWA XSS模块
- 工具：XSStrike

**第7天：文件上传+文件包含**
- 原理：绕过限制getshell
- 靶场：DVWA文件上传模块
- Upload-Labs靶场

**第8天：越权访问（SRC最赚钱的漏洞）**
- 水平越权：查看他人数据
- 垂直越权：普通用户变管理员
- 逻辑漏洞：密码重置、支付逻辑

**第9天：CSRF+SSRF**
- CSRF原理和利用
- SSRF原理和利用（内网探测）

**第10天：信息泄露+弱口令**
- 敏感信息泄露（配置文件、备份文件、源码）
- 弱口令爆破（Hydra、Burp Intruder）

### 靶场
- DVWA：通关所有难度
- sqli-labs：完成前30关
- Upload-Labs：通关

---

## 第三阶段：工具精通（第11-15天）

### 目标
熟练使用渗透测试工具链。

**第11天：Burp Suite**
- 抓包、改包、重放
- Intruder爆破
- Scanner扫描
- 插件安装

**第12天：nmap**
- 端口扫描（-sS -sV -sC）
- 服务识别
- 脚本扫描（--script）
- 输出格式（-oA）

**第13天：信息收集工具**
- 子域名：subfinder、OneForAll
- 端口：nmap、naabu
- 目录：dirsearch、ffuf
- 指纹识别：whatweb

**第14天：漏洞扫描工具**
- Nuclei：模板化扫描
- afrog：国产高性能扫描
- xray：长亭科技出品

**第15天：Metasploit基础**
- msfconsole基本操作
- 漏洞利用流程
- Meterpreter后渗透

---

## 第四阶段：实战准备（第16-20天）

### 目标
掌握SRC挖洞方法论。

**第16天：SRC挖洞方法**
- 信息收集→资产梳理→漏洞扫描→手工验证→提交报告
- 补天/漏洞盒子平台使用教程
- 漏洞报告撰写规范

**第17天：业务逻辑漏洞**
- 越权漏洞挖掘思路
- 支付逻辑漏洞
- 优惠券/积分漏洞
- 遍历枚举

**第18天：API接口安全**
- REST API测试方法
- JWT token安全
- API参数篡改

**第19天：JS逆向**
- 浏览器开发者工具
- JS加密逆向
- 隐藏API接口发现

**第20天：靶场综合练习**
- HackTheBox入门机2-3台
- 或Vulhub搭建漏洞环境练习

---

## 第五阶段：SRC实战（第21-30天）

### 目标
在补天/漏洞盒子挖到第一个漏洞。

**第21-25天：每天挖2-3小时**
- 目标：补天/漏洞盒子
- 策略：从信息泄露入手（最容易）
- 每天记录挖洞过程

**第26-28天：扩展目标**
- 注册1-2个企业SRC
- 研究目标资产
- 尝试业务逻辑漏洞

**第29-30天：复盘+优化**
- 总结挖到/没挖到的原因
- 整理漏洞报告模板
- 制定持续挖洞计划

---

## 六、Pentest-Swarm-AI 学习

### 目标
理解AI渗透测试框架的架构，为后续AI辅助挖洞做准备。

### 学习内容
1. 阅读项目README和架构文档
2. 理解蜂群架构（Stigmergy、信息素衰减）
3. 看代码：cmd/、internal/目录结构
4. 理解7个Agent的职责分工
5. 学习它如何调用nmap/sqlmap等工具
6. 思考如何适配国内SRC场景

### 输出
- 架构分析笔记
- 可复用的设计思路
- 适配国内SRC的方案设想

---

## 七、学习资源汇总

### 靶场
- DVWA：本地搭建
- sqli-labs：本地搭建
- Upload-Labs：本地搭建
- HackTheBox：https://hackthebox.com（需注册）
- Vulhub：https://vulhub.org（Docker漏洞环境）

### 视频教程
- B站SRC漏洞挖掘200集（2026新手入门版）
- Professor Messer Network+/Security+

### 工具文档
- nmap：https://nmap.org/book/
- Burp Suite：https://portswigger.net/burp
- sqlmap：https://sqlmap.org/
- Nuclei：https://docs.nuclei.project/

### 社区
- Freebuf：https://freebuf.com
- 先知社区：https://xz.aliyun.com
- 安全客：https://www.anquanke.com

---

## 八、验收标准

完成以下任意一项即算"学会了"：
1. 在补天/漏洞盒子提交1个有效漏洞
2. 独立完成DVWA所有难度+HackTheBox 2台入门机
3. 能独立完成一次完整的信息收集→漏洞发现→报告提交流程

---

*创建：2026-05-19*
*状态：开始执行*
