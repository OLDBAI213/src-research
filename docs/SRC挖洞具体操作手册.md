# SRC挖洞具体操作手册

> 面向新手，从零开始手把手教你挖SRC漏洞
> 基于补天(butian.net)、漏洞盒子(vulbox.com)两大中国主流平台
> 资料来源：先知社区、Freebuf、CSDN、知乎、火山引擎开发者社区等实战经验

---

## 目录

1. [找目标的具体流程](#1-找目标的具体流程)
2. [信息收集实操](#2-信息收集实操)
3. [漏洞发现实操](#3-漏洞发现实操)
4. [工具在挖洞中的实际用法](#4-工具在挖洞中的实际用法)
5. [时间分配：2小时session规划](#5-时间分配2小时session规划)

---

## 1. 找目标的具体流程

### 1.1 打开补天/漏洞盒子后，第一步做什么？

**第一步：注册账号并完成实名认证**
- 补天：www.butian.net → 注册白帽子账号 → 实名认证（需要身份证）
- 漏洞盒子：www.vulbox.com → 注册 → 实名认证
- 不认证无法提现奖金，务必先完成

**第二步：仔细阅读平台规则（重点！）**
- 补天：查看「漏洞奖励计划」(butian.net/Help/plan) → 了解什么漏洞收、什么不收
- 漏洞盒子：查看漏洞收录范围 → 了解哪些厂商/资产在范围内
- **关键**：看平台「漏洞提交模版」，按格式提交能提高通过率

**第三步：选择目标类型**
- **新手优先选「公益SRC」**（补天上有上百个公益SRC目标），门槛低，不需要厂商授权
- 漏洞盒子范围更广，国内站点都收，非常适合新手练手
- 有一定经验后再尝试「专属SRC」（奖金高但竞争激烈）

### 1.2 怎么筛选目标？

**筛选流程（以补天公益SRC为例）：**

```
1. 打开补天 → 公益SRC列表
2. 看每个厂商的「收录范围」——只选范围明确的
3. 优先选那些自己熟悉的行业/系统（比如你懂教育系统的逻辑，优先选教育类）
4. 排除那些明显是「静态展示页」的厂商（只有几个页面，没交互功能）
5. 选有子域名、有登录功能的厂商
```

**具体筛选标准：**

| 筛选维度 | 好目标特征 | 差目标特征 |
|---------|-----------|-----------|
| 业务类型 | 电商、金融、社交、教育（交互多） | 纯展示型官网 |
| 技术栈 | 老系统（ASP/PHP/jsp）、常见CMS | 前后端分离、强WAF |
| 资产规模 | 多个子域名、多个业务系统 | 就1个域名，1个页面 |
| 历史漏洞 | 近期有人交过洞（说明在做） | 从来没被挖过（可能太难） |

**关键技巧**：用爱企查/天眼查搜索目标公司 → 知识产权 → 网站备案 → 获取其所有备案域名，这就是你的攻击面。

### 1.3 哪些资产类型最容易出洞？

按出洞概率从高到低排序：

```
1. OA系统（通达OA、用友NC、蓝凌OA等）—— Nday漏洞最多，一打一个准
2. CMS建站系统（Discuz、WordPress、PHPCMS、帝国CMS）—— 通杀漏洞多
3. 管理后台（admin/manager/login页面）—— 弱口令、未授权
4. 边缘子域名（test/dev/staging前缀）—— 防护弱、信息泄露多
5. API接口（/api/、/swagger/）—— 未授权访问
6. 文件上传/下载功能点 —— 任意文件读取、上传绕过
7. 小程序/APP配套服务端 —— 防护通常比Web端弱
```

**核心逻辑**：主站（www域名）防御最强，**边缘资产才是突破口**。

### 1.4 怎么判断一个目标值不值得花时间？

**「三看」法则：**

**一看资产量**：在FOFA搜索 `domain="目标.com"`，看返回了多少条结果。少于20条就算了，没有足够的攻击面。

**二看功能点**：打开目标网站，数一下有几个交互功能：
- 有登录注册？✓
- 有搜索框？✓
- 有文件上传？✓
- 有个人信息修改？✓
- 有订单/交易流程？✓
- 超过3个功能点 → 值得深入

**三看技术栈**：
- 用Wappalyzer（浏览器插件）看用了什么技术
- 如果看到：Apache + PHP + MySQL（老技术，漏洞多）→ 值得
- 如果看到：Nginx + React + 云WAF（新技术，防护强）→ 看情况

**快速判断公式**：
```
价值分 = 子域名数量 × 2 + 功能点数 × 3 - WAF强度 × 5
如果 > 10分 → 值得投入
```

---

## 2. 信息收集实操

### 2.1 用subfinder收集子域名后，怎么判断哪些是活跃的？

**实际操作步骤：**

```
# 1. 子域名收集（多种工具交叉验证）
subfinder -d example.com -o subs.txt

# 也可以用 OneForAll（更全面）
python3 oneforall.py --target example.com run

# 2. 筛选活跃子域名（关键步骤）
# 方式一：用httpx快速检测存活
cat subs.txt | httpx -o alive.txt

# 方式二：用curl写脚本检测
for domain in $(cat subs.txt); do
    if curl -s -o /dev/null -w "%{http_code}" --connect-timeout 3 $domain | grep -q "200\|301\|302\|403"; then
        echo "$domain [ALIVE]"
    fi
done
```

**判断标准：**

| 响应码 | 含义 | 行动 |
|-------|------|------|
| 200 | 正常访问 | ✅ 重点目标 |
| 301/302 | 有跳转 | ✅ 跳转后的页面可能是登录页 |
| 403 | 禁止访问 | ⚠️ 可能是有价值的内页，值得扫目录 |
| 401 | 需要认证 | ⚠️ 可能是后台，值得尝试弱口令 |
| 500 | 服务器错误 | ✅ 可能有漏洞 |
| 404/NXDOMAIN | 不存在 | ❌ 直接排除 |

**进阶技巧**：用浏览器手工打开每个活跃子域名，看页面内容：
- 看到「测试环境」「内部系统」「dev」「staging」→ 重点目标
- 看到登录框 → 马上测试弱口令+SQL注入
- 看到API文档（Swagger）→ 大肉来了

### 2.2 用nmap扫端口后，怎么从结果中找到有价值的目标？

**实际操作步骤：**

```
# 扫描常见端口（快速）
nmap -sS -Pn -n --open -T4 -p 80,443,8080,8443,8081,8089,7001,3306,6379,27017,22,21 target.com

# 全端口扫描（更彻底，但耗时）
nmap -sS -Pn -n --open -T4 -p 1-65535 target.com
```

**结果分析判定表：**

| 端口 | 常见服务 | 价值判断 |
|------|---------|---------|
| 80/443 | HTTP/HTTPS | 常规Web，必看 |
| 8080/8443 | Tomcat/Nginx备用端口 | ⭐ 经常有管理后台 |
| 8081/8089 | 开发环境/监控面板 | ⭐⭐⭐ 高危区域，防护弱 |
| 7001 | WebLogic | ⭐⭐⭐ Nday漏洞多 |
| 3306 | MySQL | ⭐ 数据库直连（如果能访问的话） |
| 6379 | Redis | ⭐⭐⭐ 未授权访问高风险 |
| 27017 | MongoDB | ⭐⭐⭐ 未授权访问高风险 |
| 22 | SSH | 弱口令可尝试 |
| 21 | FTP | 匿名登录可尝试 |
| 9090 | JMX/Monitor | ⭐ 管理控制台 |
| 5000/5001 | Docker/Flask | ⭐ 容器管理接口 |

**关键技巧**：
- 非标准端口（如33899、59222）往往藏着惊喜——开发人员随手开的后门服务
- 扫描结果中出现 `filtered` 端口，尝试用 `-sV` 做版本探测
- 重点关注「非标准端口上的Web服务」，很多开发者图省事把管理页面挂在奇怪端口上

### 2.3 用dirsearch扫目录后，怎么判断哪些目录值得深入？

**实际操作步骤：**

```
# 基本扫描
dirsearch -u https://target.com -e * -t 10

# 使用大字典（关键！默认字典不够用）
dirsearch -u https://target.com -w /path/to/big-dict.txt -e * -t 10
```

**结果分析判定表：**

| 状态码 | 发现的内容 | 价值判断 | 行动 |
|--------|-----------|---------|------|
| 200 | /admin/ | ⭐⭐⭐ | 管理后台，尝试弱口令 |
| 200 | /api/ | ⭐⭐⭐ | API接口，测试未授权 |
| 200 | /swagger-ui.html | ⭐⭐⭐⭐ | Swagger文档，API全暴露 |
| 200 | /actuator/ | ⭐⭐⭐⭐ | SpringBoot监控，信息泄露 |
| 200 | /.git/ | ⭐⭐⭐⭐⭐ | 源码泄露，大洞 |
| 200 | /backup/ | ⭐⭐⭐⭐ | 备份文件，下载分析 |
| 200 | /upload/ | ⭐⭐⭐ | 文件上传目录 |
| 200 | /phpmyadmin/ | ⭐⭐⭐ | 数据库管理面板 |
| 403 | /admin/ | ⭐⭐ | 有权限控制，但可能绕 |
| 301 | 任何路径 | ⭐ | 看跳转后的页面 |
| 401 | 任何路径 | ⭐⭐ | 需要认证，试弱密码 |

**实战经验**（来自先知社区ajie）：
- 建议整合其他师傅的路径字典到 `dirsearch/db/dicc.txt`
- 补充：SpringBoot未授权路径、Swagger路径、VMware vCenter漏洞路径
- 404和403页面不代表没东西——要一层一层fuzz
- 常见有用的路径：/druid/, /swagger/, /actuator/, /api-docs, /sitemap.xml, /robots.txt

---

## 3. 漏洞发现实操

### 3.1 信息泄露

**操作步骤：**

```
1. 打开目标网站 → 右键「查看网页源代码」
2. 搜索关键词：password、pwd、pass、api_key、secret、token、accessKey
3. 打开浏览器开发者工具(F12) → Network选项卡
4. 刷新页面，看所有请求的Response中是否包含敏感信息
5. 尝试访问常见敏感路径：
   - /robots.txt（看看爬虫禁止了什么，那里可能有料）
   - /.git/config（源码泄露）
   - /sitemap.xml（看哪些页面被收录）
   - /WEB-INF/web.xml（Java应用配置泄露）
```

**怎么判断有没有漏洞？**

| 发现内容 | 严重程度 | 说明 |
|---------|---------|------|
| 数据库连接密码（明文） | 高危 | 直接能连数据库 |
| 云服务AccessKey/SecretKey | 严重 | 可控制云资源 |
| 手机号/身份证号（批量） | 高危 | 用户隐私泄露 |
| 源代码 | 高危 | 代码审计找更多洞 |
| 内网IP/域名 | 中危 | 辅助信息 |
| API接口文档 | 中危 | 了解全部API |

**真实案例**：访问 `/api/swagger-ui.html` 看到全部API文档 → 发现未授权的订单查询接口 → 批量获取用户订单信息 → 高危。

### 3.2 越权漏洞

**操作步骤（水平越权）：**

```
1. 注册两个账号：A（攻击者）、B（受害者）
2. 用A账号登录，访问「个人信息」页面
3. Burp Suite抓包，找到请求中的用户标识参数（如id=123、uid=456）
4. 修改这个参数值为B账号的ID
5. 放包，看返回的是不是B的个人信息
6. 如果是 → 水平越权漏洞！
```

**操作步骤（垂直越权）：**

```
1. 注册一个普通用户账号、获取一个管理员账号(或猜测管理员接口)
2. 用普通用户登录，抓包获取cookie
3. 正常访问普通功能确认通畅
4. 修改请求URL为管理员功能（如 /admin/deleteUser?id=123）
5. 如果返回成功 → 垂直越权！
```

**怎么判断有没有漏洞？**

| 测试操作 | 正常结果 | 漏洞特征 |
|---------|---------|---------|
| 修改用户ID参数 | 返回「无权限」或「ID不存在」 | 返回了其他用户的数据 ✅ |
| 低权限用户访问高权限接口 | 返回403或跳转登录 | 返回200且有数据 ✅ |
| 未登录访问需登录的接口 | 跳转登录 | 直接返回数据 ✅ |

**辅助工具**：Burp Suite插件「瞎越」(XiaYue_Pro)，一键批量检测越权。
- 原理：高权限账号访问 → 记录请求 → 替换为低权限cookie → 对比响应长度
- 响应长度相同 → 可能存在越权

### 3.3 SQL注入

**操作步骤：**

```
1. 找到带参数的URL（如：https://target.com/news.php?id=1）
2. 手工测试：
   - 加单引号：id=1' → 看是否报错
   - 加 and 1=1：id=1 and 1=1 → 正常返回
   - 加 and 1=2：id=1 and 1=2 → 如果返回不同，基本确认注入
3. 用sqlmap验证（见第4章工具用法）
```

**怎么判断有没有漏洞？**

| 测试 | 结果 | 判断 |
|-----|------|------|
| 输入 `'` | 页面报数据库错误 | 很可能存在注入 ✅ |
| `and 1=1` 正常，`and 1=2` 异常 | 返回结果不同 | 确认存在注入 ✅ |
| 输入 `sleep(5)` 页面响应变慢 | 延迟了5秒 | 盲注 ✅ |
| 所有测试都正常返回 | 没有变化 | 可能不存在，或者被WAF拦截 |

**搜索注入点Google语法**：
```
site:target.com inurl:php?id=
site:target.com inurl:asp?id=
site:target.com inurl:Show.asp?
```

### 3.4 XSS（跨站脚本）

**操作步骤：**

```
1. 找到输入框/URL参数（搜索框、评论区、用户名修改、URL参数等）
2. 输入测试payload：
   <script>alert(1)</script>
   <img src=x onerror=alert(1)>
   "><script>alert(1)</script>
3. 看是否弹出alert对话框
4. 如果有输出 → 存储型XSS
5. 如果只在URL参数中触发 → 反射型XSS
```

**怎么判断有没有漏洞？**

| 测试 | 结果 | 类型 |
|-----|------|------|
| 输入 `<script>alert(1)</script>` | 弹出对话框 | 存在XSS ✅ |
| 保存后每次打开都弹 | 存储型XSS | 高危 |
| 仅当前页面显示时弹 | 反射型XSS | 中危 |
| 输入被转义成 `&lt;script&gt;` | 无漏洞 | 防护到位 |

**实战技巧**：
- 不要只试 `<script>`，现在很少能直接过了
- 尝试：`<img src=x onerror=alert(1)>`、`<svg onload=alert(1)>`、`<a href=javascript:alert(1)>`
- 在Burp里观察服务端是否对特殊字符做了编码处理

### 3.5 文件上传

**操作步骤：**

```
1. 找到有文件上传功能的位置（头像上传、附件上传、文件导入等）
2. 尝试上传一个webshell：
   - 先上传正常的 .jpg 图片 → 成功
   - 再上传 .php 文件 → 被拦截（正常）
   - 尝试绕过：
     a. 改后缀：.php5, .phtml, .php.jpg
     b. 改Content-Type：image/jpeg
     c. 使用%00截断：shell.php%00.jpg
     d. 双后缀：shell.php.jpg
3. 上传成功后，访问上传的文件路径
```

**怎么判断有没有漏洞？**

| 测试 | 结果 | 判断 |
|-----|------|------|
| 上传.php文件 | 提示不支持格式 | 有防护 |
| 改Content-Type后上传 | 成功了 → 存在漏洞 ✅ | Content-Type校验不严 |
| 改后缀.php5上传 | 成功了 → 存在漏洞 ✅ | 黑白名单不全 |
| 上传后能访问到文件 | ✅ 进一步验证能否执行 | 尝试访问/shell.php5 |

### 3.6 逻辑漏洞

**操作步骤：**

```
1. 先正常走一遍业务流程，理解完整逻辑
2. 思考「正常用户不会做的事」：
   
   【数量逻辑】
   - 下单时修改数量为负数 → 总价变负
   - 优惠券重复使用 → 叠加折扣
   - 积分/金币进行整数溢出（int类型最大值）
   
   【步骤绕过】
   - 支付环节跳过 → 直接到确认页
   - 实名认证绕过 → 修改认证状态参数
   - 订阅功能 → 取消订阅后是否还能享受服务
   
   【并发漏洞】
   - 同时用多个窗口发送同一个请求 → 多次领取同一份奖励
   - 抢购/秒杀场景 → 并发请求突破数量限制
   
3. 用Burp Suite抓包分析每个请求的参数
4. 修改关键参数看后端是否校验
```

**怎么判断有没有漏洞？**

| 测试操作 | 正常期望 | 漏洞特征 |
|---------|---------|---------|
| 修改价格为0或负数下单 | 提示价格异常 | 成功下单付款0元 ✅ |
| 重复使用同一优惠券 | 提示已使用 | 能反复抵扣 ✅ |
| 同时发送10个领取请求 | 只领取1次 | 领了10次 ✅ |
| 跳过支付步骤 | 提示先支付 | 直接生成订单 ✅ |

**挖洞核心心法**（来自资深白帽）：
> "可以直接扫描出来的洞，基本都被交完了。**登录后的漏洞重复率比登录前低很多**。——WS师傅"

> "逻辑漏洞全靠操作性，不像XSS一样直接输入alert(1)就完事。步骤最多也就三步，关键是思路要猥琐。"

---

## 4. 工具在挖洞中的实际用法

### 4.1 Nmap怎么用才能发现隐藏服务？

**基础用法（快速探测）**：
```bash
# 快速扫描常见端口（30秒完成）
nmap -sS -Pn -n --open -T4 -p 80,443,8080,8443,7001,3306,6379,27017,22,21 target.com

# 服务版本探测（发现具体服务）
nmap -sV -Pn -n --open -T4 -p 80,443,8080 target.com

# 操作系统识别
nmap -O -Pn target.com
```

**发现隐藏服务的关键技巧：**

```bash
# 技巧1：全端口扫描（发现非标准端口上的服务）
nmap -sS -Pn -n --open -T4 -p 1-65535 target.com
# 重点看非标准端口（如 33899、59222、8888等）
# 开发者经常把管理后台挂在奇怪端口上

# 技巧2：UDP扫描（很多服务只用UDP）
nmap -sU -Pn --open -T4 -p 161,53,123,500 target.com

# 技巧3：使用NSE脚本扫描常见漏洞
nmap --script=vuln -Pn target.com

# 技巧4：遍历C段（发现同一网段的其他服务器）
nmap -sS -Pn -n --open -T4 -p 80,443,8080 192.168.1.0/24

# 技巧5：探测WAF类型
nmap --script=http-waf-detect -p 80,443 target.com
```

**重点关注的服务与端口：**

| 服务 | 端口 | 隐藏可能 | 价值 |
|-----|------|---------|------|
| Tomcat管理后台 | 8080/8443 | 高 | 默认口令部署war包 |
| Jenkins | 8080/50000 | 高 | 未授权执行命令 |
| ElasticSearch | 9200/9300 | 高 | 未授权数据泄露 |
| Redis | 6379 | 中 | 未授权写SSH密钥 |
| MongoDB | 27017 | 中 | 未授权数据泄露 |
| Docker API | 2375 | 高 | 未授权控制容器 |
| ZooKeeper | 2181 | 高 | 信息泄露 |

### 4.2 Burp Suite怎么抓包改包找越权？

**完整操作流程：**

```
步骤1：设置代理
- Burp Suite → Proxy → Options → Add Proxy Listener（默认 127.0.0.1:8080）
- 浏览器安装SwitchyOmega插件 → 设置代理 127.0.0.1:8080
- 或者使用Burp内置浏览器（最简单：Proxy → Open Browser）

步骤2：安装CA证书（抓HTTPS必须）
- 浏览器访问 http://burp → 下载cacert.der
- 浏览器设置 → 隐私与安全 → 管理证书 → 导入

步骤3：抓包
- Proxy → Intercept → Interception is on（拦截开启）
- 浏览器操作 → 请求被拦截在Burp中

步骤4：改包找越权
- 拦截到一个请求后，在Raw选项卡找到关键参数
- 修改参数（如 userId=123 → userId=456）
- 点击 Forward 发送修改后的请求
- 看Response是否返回了其他用户的数据
```

**实际操作模板：**

```http
# 原始请求（普通用户查看个人信息）
GET /api/user/profile?id=1001 HTTP/1.1
Host: target.com
Cookie: session=ABC123
Authorization: Bearer token_xxx

# 修改后（尝试查看其他用户）
GET /api/user/profile?id=1002 HTTP/1.1
Host: target.com
Cookie: session=ABC123
Authorization: Bearer token_xxx
```

**进阶技巧：**

```
1. 使用Repeater快速修改和重放请求
   - 右键请求 → Send to Repeater (Ctrl+R)
   - 在Repeater中多次修改参数值测试

2. 使用Intruder批量测试ID遍历
   - 右键请求 → Send to Intruder (Ctrl+I)
   - 选中id参数 → Add §（设置payload位置）
   - Payloads → 设置数字1-10000
   - Start attack → 看哪些ID能返回非空数据

3. 越权检测自动化插件
   - 「瞎越」(XiaYue_Pro) 插件
   - 配置高权限cookie和低权限cookie
   - 自动对比响应长度，标记可疑请求
```

### 4.3 sqlmap怎么用才能判断注入点？

**完整操作流程：**

```bash
# 基础用法（GET注入）
sqlmap -u "http://target.com/news.php?id=1"

# POST注入（从Burp抓包保存的请求文件）
# 先在Burp里拦截POST请求 → 右键Save to file → 保存为 request.txt
sqlmap -r request.txt

# 带Cookie的注入（需要登录的页面）
sqlmap -u "http://target.com/user.php?id=1" --cookie="session=abc123"

# 批量检测多个URL
sqlmap -m urls.txt
```

**判断是否可注入的关键指标：**

```bash
# 执行后看这几个输出：

1. 看是否显示 "Parameter: id (GET) seems injectable"
   → 嗯，存在注入点 ✅

2. 看是否显示数据库信息
   → 如 "web application technology: PHP 7.3, Apache"
   → "back-end DBMS: MySQL 5.7"
   → 确认有注入 ✅

3. 看是否提示 "all tested parameters appear to be not injectable"
   → 所有参数都不可注入 ❌

4. 看是否提示 "WAF/IPS identified"
   → 被WAF拦截了，需要用tamper绕过
```

**绕过WAF的实战命令：**

```bash
# 使用tamper脚本绕过WAF
sqlmap -u "http://target.com/news.php?id=1" --tamper=space2comment,between,randomcase

# 降低请求速度避免触发防护
sqlmap -u "http://target.com/news.php?id=1" --delay=2 --random-agent

# 增加检测级别（level 2 检测Cookie注入，level 3 检测User-Agent注入）
sqlmap -u "http://target.com/news.php?id=1" --level=3 --risk=2

# 指定注入点位置（已知某个参数存在注入）
sqlmap -u "http://target.com/news.php?id=1*" --level=5
```

**获取数据的命令：**

```bash
# 列出所有数据库
sqlmap -u "http://target.com/news.php?id=1" --dbs

# 列出指定数据库的所有表
sqlmap -u "http://target.com/news.php?id=1" -D database_name --tables

# 列出指定表的所有列
sqlmap -u "http://target.com/news.php?id=1" -D database_name -T users --columns

# 导出数据
sqlmap -u "http://target.com/news.php?id=1" -D database_name -T users -C username,password --dump

# 尝试获取操作系统shell（需要高权限）
sqlmap -u "http://target.com/news.php?id=1" --os-shell
```

### 4.4 nuclei怎么配置才能扫出有用的结果？

**安装与基础配置：**

```bash
# 下载安装
# 从 https://github.com/projectdiscovery/nuclei/releases 下载对应版本
# 解压后放到PATH路径

# 首次运行（自动下载模板）
nuclei -u https://target.com

# 模板存储在 ~/.nuclei-templates/
```

**实战扫描命令：**

```bash
# 1. 快速扫描（日常最爱）
nuclei -u https://target.com -severity critical,high -o results.txt

# 2. 批量扫描子域名
nuclei -l subdomains.txt -o results.txt

# 3. 使用特定分类模板（减少噪音）
# 只扫OA/CMS相关漏洞
nuclei -u https://target.com -t cves/ -t exposures/ -t misconfiguration/

# 4. 排除噪音模板
nuclei -u https://target.com -et info -et unknown

# 5. 最实用的组合（推荐）
nuclei -u https://target.com \
  -severity critical,high,medium \
  -t cves/ \
  -t exposures/ \
  -t misconfiguration/ \
  -t default-logins/ \
  -exclude ~/.nuclei-templates/technologies/ \
  -o results.txt \
  -stats
```

**配置优化技巧：**

```
1. 更新模板（每周一次）
   nuclei -update-templates

2. 自定义模板目录
   nuclei -u https://target.com -t /path/to/custom-templates/

3. 去重输出（相同URL的相同结果只保留一次）
   nuclei -u https://target.com -o results.txt -duc

4. 结合Markdown输出（方便写报告）
   nuclei -u https://target.com -o results.md -me report/
```

**重点关注哪些模板类型：**

| 模板分类 | 重要性 | 说明 |
|---------|-------|------|
| cves/ | ⭐⭐⭐ | 通用CVE漏洞，新版Nday |
| exposures/ | ⭐⭐⭐ | 信息泄露、敏感文件 |
| misconfiguration/ | ⭐⭐⭐ | 配置错误（如未授权访问） |
| default-logins/ | ⭐⭐⭐ | 默认口令检测 |
| vulnerabilities/ | ⭐⭐ | 通用漏洞检测 |
| technologies/ | ⭐ | 指纹识别（辅助信息） |

**实战经验**：
- 运行nuclei时加上 `-stats` 可以看到实时进度
- 使用 `-duc` (dedup) 避免重复输出
- nuclei主要是扫已知漏洞（Nday），适合做信息收集后的快速打点
- 对于逻辑漏洞、越权等，nuclei无能为力，需要手工测

---

## 5. 时间分配：2小时session规划

### 5.1 完整2小时分配表

| 时间段 | 时长 | 做什么 | 产出目标 |
|-------|------|--------|---------|
| **0-15分钟** | 15min | **选目标 + 快速信息收集** | 确定2-3个目标 |
| **15-35分钟** | 20min | **批量扫资产**（子域名+端口+指纹） | 找到1个最有潜力的目标 |
| **35-50分钟** | 15min | **Nday扫描**（nuclei扫已知漏洞） | 快速捡漏 |
| **50-70分钟** | 20min | **手工测试**（登录框+参数+文件上传） | 找逻辑漏洞 |
| **70-90分钟** | 20min | **深入挖掘**（针对有问题的功能点） | 利用和验证 |
| **90-100分钟** | 10min | **Burp分析+抓包改包** | 找越权和隐藏接口 |
| **100-110分钟** | 10min | **整理发现，截图取证** | 准备报告材料 |
| **110-120分钟** | 10min | **漏洞提交** | 提交1-2个有效漏洞 |

### 5.2 具体到每一分钟的节奏

**0-15分钟：选目标**
```
1. 打开补天公益SRC列表或漏洞盒子目标列表 → 2min
2. 选3个自己感兴趣的厂商 → 1min
3. 快速打开厂商首页，看是不是动态网站（有交互功能）→ 2min
4. 用FOFA/爱企查查厂商资产量 → 3min
5. 确定1个最有潜力的目标 → 2min
6. 记录目标的基础信息 → 5min
```

**15-35分钟：批量扫资产**
```bash
# 同时启动以下命令（开3个终端窗口）

# 终端1：子域名收集
subfinder -d target.com | httpx -o alive.txt

# 终端2：nmap端口扫描（只扫常见端口）
nmap -sS -Pn -n --open -T4 -p 80,443,8080,8443,7001,3306,6379 target.com

# 终端3：指纹识别
# 用浏览器打开每一个活跃子域名，看是什么系统
```

**35-50分钟：Nday捡漏**
```bash
# 对全部活跃子域名跑nuclei
nuclei -l alive.txt -severity critical,high -o nday_results.txt
```

**50-70分钟：手工测试**
```
1. 打开目标网站，从头到尾点一遍所有功能 → 10min
2. 测试登录框（弱口令+SQL注入）→ 5min
3. 测试搜索框（XSS+SQL注入）→ 3min
4. 测试文件上传点 → 2min
```

**70-90分钟：深入挖掘**
```
对前面发现的有问题的地方深入利用：
- 如果有API接口 → 一个一个测未授权
- 如果有用户功能 → 注册两个账号测越权
- 如果有参数 → 一个一个改着看
```

**90-100分钟：Burp分析**
```
1. 打开Burp的HTTP History → 扫一遍所有请求记录 → 5min
2. 找到可疑的参数请求 → 发送到Repeater → 3min
3. 修改参数值测试越权 → 2min
```

**100-120分钟：收尾**
```
1. 截图保留漏洞证据 → 5min
2. 按平台模板写漏洞报告 → 10min
3. 提交漏洞 → 3min
4. 记录本次挖洞的笔记（什么没挖到、下次怎么改进）→ 2min
```

### 5.3 黄金原则

**1. 别在一个站上吊死**
- 如果15分钟还没发现任何有价值的点 → 换目标
- 公益SRC拼的是手速和广度，不是深度

**2. 时间到就收手**
- 2小时后如果还在继续 → 边际效益递减
- 建议定时器，时间到就整理提交

**3. 利用活动期**
- 补天/漏洞盒子搞活动时，目标更多、奖金更高
- 关注公众号提前获取活动信息

**4. 日常积累 > 临时抱佛脚**
- 建立自己的「挖洞笔记」（推荐用Obsidian/Notion）
- 记录每个目标的资产信息、测过的接口、发现的线索
- 下次再挖同目标时，直接从笔记续上

**5. 情绪管理**
- 挖不到是常态，别灰心
- 挖到了是惊喜，别上头继续熬夜
- 记录失败比记录成功更有价值

---

## 附录：常用资源速查

### 资产查询
| 工具 | 用途 | 地址 |
|-----|------|------|
| FOFA | 资产测绘 | fofa.info |
| 鹰图Hunter | 资产测绘 | hunter.qianxin.com |
| 360Quake | 资产测绘 | quake.360.net |
| 爱企查 | 企业信息 | aiqicha.baidu.com |
| 天眼查 | 企业信息 | tianyancha.com |
| crt.sh | 证书日志 | crt.sh |
| 站长工具 | 综合查询 | tool.chinaz.com |

### 工具下载
| 工具 | 用途 | GitHub |
|-----|------|--------|
| OneForAll | 子域名收集 | github.com/shmilylty/OneForAll |
| EHole | 指纹识别 | github.com/EdgeSecurityTeam/EHole |
| dirsearch | 目录扫描 | github.com/maurosoria/dirsearch |
| nuclei | 漏洞扫描 | github.com/projectdiscovery/nuclei |
| sqlmap | SQL注入 | github.com/sqlmapproject/sqlmap |
| 瞎越 | 越权检测 | github.com/winezer0/XiaYue_Pro |
| httpx | 存活探测 | github.com/projectdiscovery/httpx |

### 字典资源
| 名称 | 说明 | 地址 |
|-----|------|------|
| SuperWordlist | 弱口令字典 | github.com/fuzz-security/SuperWordlist |
| dict-hub | 红队字典合集 | github.com/ybdt/dict-hub |
| Assetnote Wordlists | 目录爆破字典 | wordlists.assetnote.io |

---

> **最后的话**：挖SRC最重要的不是技术，是**心态**和**细心**。
> 很多人挖一两个小时没结果就放弃了，但你只要坚持下去，记录每次的发现和不足，慢慢就会形成自己的挖洞思路。
> 正如前辈们说的：**心细则能挖天下！**
