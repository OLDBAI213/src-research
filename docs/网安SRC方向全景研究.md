---
date: 2026-05-19
type: research
tags:
  - 网安
  - SRC
  - 渗透测试
  - 赚钱
status: active
---

# 网安SRC方向全景研究

> 目标：搞清楚SRC挖洞赚钱的完整路径，找到我们的切入点。

## 一、国内SRC平台全景

### 第一梯队：企业自建SRC（奖励高，门槛也高）

- 腾讯TSRC (security.tencent.com) — 国内最早(2012)，单洞最高12万
- 阿里SRC (security.alibaba.com) — 阿里云/淘宝/支付宝，10万+
- 字节SRC (security.bytedance.com) — 抖音/头条/飞书，10万+
- 京东SRC (security.jd.com) — 电商+物流+金融，5万+
- 美团SRC (security.meituan.com) — 外卖+到店+酒旅，5万+
- 百度SRC (security.baidu.com) — 搜索+AI+云，5万+
- 华为PSIRT (hwpssect.huawei.com) — 终端+云+通信，CVE致谢+奖金
- 小米SRC (security.mi.com) — IoT+手机，3万+

### 第二梯队：第三方平台（门槛低，新手入门）

- **补天** (butian.net) — 奇安信旗下，公益平台，新手首选
- **漏洞盒子** (vulbox.com) — 门槛最低，啥漏洞都收，练手用
- **EduSRC** (edusrc.com) — 教育行业，必须edu域名
- **CNNVD** (cnnvd.org.cn) — 国家级，门槛极高

### 第三梯队：众测平台

- 漏洞盒子众测 — 企业付费，按项目挖
- HackerOne — 国际最大，英文报告
- Bugcrowd — 国际，公开和私有项目
- huntr.dev — 专注开源软件

### 新手路径

1. 补天/漏洞盒子练手（门槛低）
2. EduSRC（有edu邮箱的话）
3. 企业SRC（腾讯/字节/阿里）
4. HackerOne/Bugcrowd（国际化赚美元）

## 二、渗透测试学习路线（6-12个月）

### 阶段一：基础入门（1-3个月）

必学：TCP/IP、HTTP/HTTPS、DNS、Kali Linux、Python基础、OWASP Top 10

工具入门：nmap（端口扫描）、Burp Suite（抓包）、sqlmap（SQL注入）、dirsearch（目录扫描）

靶场：DVWA、sqli-labs、WebGoat

### 阶段二：漏洞攻坚（4-6个月）

核心漏洞类型（按SRC常见度排序）：
1. 越权访问（水平/垂直）— SRC最常见高危
2. 信息泄露（敏感数据暴露）
3. SQL注入（联合/盲注/堆叠）
4. XSS（存储型/反射型/DOM型）
5. 文件上传（绕过getshell）
6. 文件包含（本地/远程）
7. CSRF（跨站请求伪造）
8. 命令执行RCE（最高危）
9. 逻辑漏洞（支付/密码重置/优惠券超发）
10. 弱口令/暴力破解

进阶工具：MSF、Nuclei、afrog（国产）、xray（长亭科技）

### 阶段三：实战挖洞（7-9个月）

信息收集：OneForAll/Amass/subfinder（子域名）、鹰图/Shodan/ZoomEye（资产测绘）、天眼查/爱企查（企业信息）

SRC挖洞技巧：信息泄露入手、业务逻辑漏洞、API接口测试、JS逆向找隐藏接口

### 阶段四：进阶（10-12个月）

内网渗透、代码审计、二进制漏洞、CTF

## 三、核心工具清单

### 信息收集
- OneForAll — 子域名收集
- Amass — 子域名枚举
- Nmap — 端口扫描
- httpx — HTTP探测

### 漏洞扫描
- Nuclei — 模板化漏洞扫描（projectdiscovery）
- afrog — 高性能漏洞扫描（国产）
- xray — 安全评估（长亭科技）

### 漏洞利用
- Metasploit — 漏洞利用框架
- SQLMap — SQL注入
- Burp Suite — 抓包改包
- dirsearch — 目录扫描

### AI+安全（差异化）
- PentestGPT — LLM驱动渗透测试
- PentAGI — 自主多Agent安全测试
- openclaw-sec-skills — 网安AI Agent Skills
- MCP for Security — SQLMap/NMAP/FFUF的MCP集成

## 四、AI+安全：我们的核心优势

2026年：已有70+开源AI渗透测试工具，65+在18个月内发布。

关键趋势：
- 并行化：同时扫描整个攻击面（人类做不到）
- 零边际成本：已知攻击链执行成本趋近于零
- 不遗忘：不会忘记3小时前的发现
- 自适应：实时调整策略

我们的切入点：
- AI辅助SRC挖洞：批量扫描+人工确认
- 自动化漏洞报告生成
- 靶场自动通关
- 代码审计AI化

## 五、关键GitHub仓库

### 靶场
- WebGoat/WebGoat (9k+) — OWASP官方靶场
- Audi-1/sqli-labs (10k+) — SQL注入练习
- RandomStorm/DVWA (8k+) — 综合Web漏洞靶场
- Vulhub/vulhub (19k+) — Docker一键搭建漏洞环境

### 工具集合
- enaqx/awesome-pentest (10k+) — 渗透测试资源
- carpedm20/awesome-hacking (5k+) — Hacking教程
- taielab/awesome-hacking-lists — 工具大全
- alphaSeclab/awesome-security-collection (2.4k+) — 安全工具

### AI安全
- openclaw-sec-skills — 网安AI Agent Skills
- PentestGPT — LLM渗透测试
- nuclei-templates — Nuclei漏洞扫描模板

## 六、实战路径

### 第1个月：环境+基础
- 安装Kali Linux
- 安装Burp Suite/nmap/sqlmap
- 搭建DVWA靶场，通关所有难度
- 注册补天/漏洞盒子

### 第2-3个月：漏洞+靶场
- 完成sqli-labs全部关卡
- 学习OWASP Top 10
- 用Vulhub搭建3个+漏洞环境

### 第4-6个月：SRC实战
- 提交第一个漏洞
- 学习业务逻辑漏洞
- 研究JS逆向
- 注册1-2个企业SRC

### 第7-12个月：稳定产出
- 每月挖2-3个漏洞
- 研究AI辅助挖洞
- 考虑HackerOne国际化

## 七、法律提醒

只在授权范围内测试！不扫描非授权资产，不利用漏洞获取/修改数据，遵守《网络安全法》。

---
*创建：2026-05-19*
