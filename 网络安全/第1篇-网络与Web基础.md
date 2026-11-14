---
title: 第 1 篇：网络与 Web 基础
layout: article
---
# 第 1 篇：一个请求从浏览器到服务器，中间发生了什么

挖洞挖的是"数据流"。你输入的字符从浏览器出发，经过 DNS、TCP、TLS、HTTP，到后端路由、业务代码、数据库，再原路返回。**每个漏洞都是这条链上某一环没做检查**：SQL 注入是数据库那环，XSS 是浏览器渲染那环，SSRF 是服务端发请求那环。

这一篇把这条链拆开，全部用命令验证，不靠背概念。

---

## 一、从 URL 到 IP：DNS 和端口

浏览器输入 `https://www.example.com/login?id=1`，第一件事不是发 HTTP，而是查 DNS：

```bash
dig www.example.com +short
```

输出可能是：

```text
93.184.216.34
```

`dig` 的完整输出里重点看几个字段：

```bash
dig www.example.com
```

```text
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 12345
;; ANSWER SECTION:
www.example.com.  3600  IN  CNAME  example.com.
example.com.      3600  IN  A      93.184.216.34
```

- `CNAME`：别名。CDN、云 WAF 常把业务域名 CNAME 到厂商域名。做信息收集时，CNAME 能帮你判断目标用了哪家云。
- `A`：IPv4；`AAAA`：IPv6；`MX`：邮件；`TXT`：SPF、域名验证；`NS`：谁管这个域的 DNS。

查其他记录：

```bash
dig example.com MX +short
dig example.com TXT +short
dig example.com NS +short
```

**安全视角：** `TXT` 里经常有 `google-site-verification=...`、`v=spf1 include:...`，能反推目标用了哪些第三方服务；`NS` 指向小厂商时，可能存在域传送或接管风险（第 3 篇展开）。

### 端口：一个 IP 上开很多扇门

| 端口 | 服务 | 常见测试点 |
| --- | --- | --- |
| 21 | FTP | 匿名登录、弱口令 |
| 22 | SSH | 弱口令、密钥泄露（SRC 通常禁止爆破） |
| 80/443 | HTTP/HTTPS | Web 漏洞主战场 |
| 3306 | MySQL | 未授权、弱口令 |
| 6379 | Redis | 未授权访问、写文件 |
| 8080/8443 | Tomcat、管理后台 | 默认口令、后台漏洞 |
| 9200 | Elasticsearch | 未授权、数据泄露 |
| 27017 | MongoDB | 未授权 |

看本机监听了哪些端口：

```bash
ss -lntp
```

看目标端口（只对授权目标）：

```bash
nmap -Pn -sV -p 80,443,8080 192.0.2.10
```

`-Pn` 跳过 ping 探测，`-sV` 识别服务版本。**不要一上来 `-p-` 全端口扫**，很多 SRC 把高频扫描视为攻击行为，先看规则。

---

## 二、HTTP 报文：一行一行看懂

### 请求

```http
POST /login HTTP/1.1
Host: target.com
User-Agent: Mozilla/5.0
Content-Type: application/x-www-form-urlencoded
Cookie: PHPSESSID=abc123; lang=zh
Content-Length: 32

username=admin&password=123456
```

四部分：

1. **请求行**：方法 + 路径 + 协议版本。
2. **请求头**：`Host`、`User-Agent`、`Cookie`、`Content-Type`、`Referer`、`Authorization` 等。
3. **空行**：分隔头和体。
4. **请求体**：GET 通常没有，POST/PUT 有。

### 响应

```http
HTTP/1.1 200 OK
Server: nginx
Content-Type: text/html; charset=utf-8
Set-Cookie: PHPSESSID=abc123; Path=/; HttpOnly
Content-Length: 1024

<html>...</html>
```

### 方法和状态码

| 方法 | 语义 | 测试时注意 |
| --- | --- | --- |
| GET | 取资源，参数在 URL | 参数全在 URL，最容易测注入和越权 |
| POST | 提交数据，参数在体 | 注意 `Content-Type`：表单、JSON、multipart |
| PUT | 上传/替换资源 | 有些 API 用它写文件 |
| DELETE | 删除资源 | **不要对真实目标乱试**，容易造成破坏 |
| PATCH | 局部更新 | 常见于用户资料、订单接口 |
| OPTIONS | 预检/查支持方法 | 看 CORS 配置 |

| 状态码 | 含义 | 漏洞场景 |
| --- | --- | --- |
| 200 | 成功 | 正常返回也可能藏着越权数据 |
| 301/302 | 跳转 | 开放重定向、SSRF 跟随跳转 |
| 401 | 未认证 | 未登录访问 |
| 403 | 已认证但无权 | 越权、目录扫描命中 |
| 404 | 不存在 | 路径探测基线 |
| 405 | 方法不允许 | 换个方法可能就通了 |
| 500 | 服务端错误 | 报错信息泄露、SQL 注入线索 |

### 常用请求头逐个说

| 头 | 作用 | 安全相关 |
| --- | --- | --- |
| `Host` | 目标域名 | 虚拟主机、密码重置投毒、缓存投毒 |
| `X-Forwarded-For` | 代理链真实 IP | 有些系统直接信任它做 IP 限制，可伪造 |
| `Referer` | 来源页 | 有些接口只校验 Referer，可伪造 |
| `Origin` | 跨域请求来源 | CORS 校验、CSRF 防御 |
| `Cookie` | 会话凭证 | XSS 偷取目标、CSRF 自动携带 |
| `Authorization` | Bearer Token / Basic | 越权测试重点 |
| `Content-Type` | 体格式 | 上传绕过、JSON 解析差异 |
| `X-Requested-With` | 是否 Ajax | 有些接口靠它判断"内部请求" |

**自己改一遍：**

```bash
curl -i http://127.0.0.1:8000/login \
  -H 'Host: target.local' \
  -H 'X-Forwarded-For: 127.0.0.1' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data 'username=admin&password=123456'
```

`-i` 把响应头一起打印出来。Burp Repeater 里做的其实就是这件事，只是有界面。

---

## 三、URL 编码：payload 为什么经常要变形

URL 里 `?`、`&`、`#`、空格、中文都不能直接出现，要百分号编码：

```text
空格 → %20 或 +
单引号 → %27
双引号 → %22
< → %3C
> → %3E
/ → %2F
# → %23
```

Python 一行验证：

```bash
python3 -c "import urllib.parse; print(urllib.parse.quote(\"' OR 1=1-- -\"))"
```

```text
%27%20OR%201%3D1--%20-
```

反过来：

```bash
python3 -c "import urllib.parse; print(urllib.parse.unquote('%27%20OR%201%3D1--%20-'))"
```

### 三个容易踩的编码坑

1. **`+` 在 query 里代表空格**，在路径里就是加号。表单提交会把空格编码成 `+`，JSON 不会。
2. **双重编码**：WAF 解一次，后端再解一次，`%2527` 第一次变 `%27`，第二次变 `'`。测试时先确认后端解码几次。
3. **Unicode/全角字符**：有些过滤器只拦 ASCII 单引号，后端数据库或模板引擎却会做字符集转换。这类绕过不稳定，报告里要写清环境。

HTTP 参数污染（HPP）也在这里：`?id=1&id=2`，不同语言取法不同——PHP 取最后一个，Python/Flask 取第一个，Node/Express 可能变数组。WAF 和后端取的值不一致时，就能绕过。

---

## 四、Cookie、Session、Token：登录到底怎么维持

HTTP 本身无状态。登录成功后，服务端给你一个凭证，之后每次请求带上：

```http
Set-Cookie: PHPSESSID=8f3a...; Path=/; HttpOnly; Secure; SameSite=Lax
```

| 属性 | 作用 | 安全意义 |
| --- | --- | --- |
| `HttpOnly` | JS 读不到 | XSS 偷不到这个 Cookie |
| `Secure` | 只在 HTTPS 发送 | 防明文嗅探 |
| `SameSite=Strict/Lax/None` | 跨站是否携带 | CSRF 防御的重要一环 |
| `Domain` | 哪些域可带 | 设太宽会让子域互相污染 |
| `Path` | 哪些路径可带 | 不是安全边界，别靠它隔离 |
| `Expires/Max-Age` | 有效期 | 长期有效增加泄露风险 |

### Session 和 JWT 的区别

- **Session**：服务端存状态，Cookie 里只放一个随机 ID。服务端查表知道你是谁。优点是能随时踢人，缺点是分布式要共享存储。
- **JWT**：服务端不存状态，令牌自带用户信息和签名。格式是三段 Base64URL，用 `.` 分隔：

```text
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1aWQiOjEsInJvbGUiOiJ1c2VyIn0.signature
```

手动解一段：

```bash
python3 - <<'PY'
import base64, json
part = "eyJ1aWQiOjEsInJvbGUiOiJ1c2VyIn0"
part += "=" * (-len(part) % 4)
print(json.dumps(json.loads(base64.urlsafe_b64decode(part)), ensure_ascii=False, indent=2))
PY
```

输出：

```json
{
  "uid": 1,
  "role": "user"
}
```

**注意：** 解出来只是明文，不代表能改。改了没有正确签名，服务端会拒。JWT 的攻击面在第 10 篇讲（`alg:none`、弱密钥、`kid` 注入）。

### 实验：用 Python 保持登录

以本地 DVWA 为例（先按第 2 篇把环境起起来）。DVWA 登录表单带一个 CSRF token，需要先 GET 登录页取 token，再 POST：

```python
import re
import requests

BASE = "http://127.0.0.1:8080"
s = requests.Session()

r = s.get(f"{BASE}/login.php", timeout=10)
token = re.search(r"name='user_token' value='([^']+)'", r.text).group(1)

r = s.post(
    f"{BASE}/login.php",
    data={"username": "admin", "password": "password",
          "Login": "Login", "user_token": token},
    timeout=10,
    allow_redirects=True,
)
print("登录后状态:", r.status_code, "URL:", r.url)
print("Cookie:", s.cookies.get_dict())

# 带着会话访问需要登录的页面
r = s.get(f"{BASE}/vulnerabilities/sqli/", timeout=10)
print("SQL 页面长度:", len(r.text))
```

能打印出 `PHPSESSID` 和 SQL 页面长度，说明会话维持成功。后面写盲注、越权、并发脚本，都是在这个骨架上改。

---

## 五、同源策略：浏览器最核心的安全边界

同源 = **协议 + 域名 + 端口** 三者完全相同。

| 页面地址 | 请求地址 | 是否同源 | 原因 |
| --- | --- | --- | --- |
| `https://a.com/x` | `https://a.com/y` | 是 | 协议、域、端口相同 |
| `https://a.com` | `http://a.com` | 否 | 协议不同 |
| `https://a.com` | `https://b.a.com` | 否 | 域名不同 |
| `https://a.com` | `https://a.com:8443` | 否 | 端口不同 |

同源策略限制的是**读取响应**，不是**发送请求**。这点非常关键：

- `<img src="https://bank.com/transfer?to=attacker">` 可以发出去，但你读不到响应内容；
- `fetch("https://bank.com/api")` 默认能发（简单请求），但浏览器不给你响应；
- 这正好是 CSRF 的生存空间：**请求发出去了，副作用已经发生，读不到响应没关系**。

### CORS：服务端主动开的口子

服务端返回：

```http
Access-Control-Allow-Origin: https://a.com
Access-Control-Allow-Credentials: true
```

浏览器才允许 `a.com` 读取响应。常见错误配置：

```http
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
```

这两个同时出现是无效的（浏览器会拒），但很多代码写成"反射 Origin"，就变成任意站点可读：

```python
# 错误示范：直接回显请求的 Origin
origin = request.headers.get("Origin")
response.headers["Access-Control-Allow-Origin"] = origin
response.headers["Access-Control-Allow-Credentials"] = "true"
```

这一段在第 6 篇会用实际请求验证。

---

## 六、HTML 和 JavaScript 的最小必要知识

XSS、CSRF、点击劫持都发生在浏览器端，下面这些必须认识。

### 一个表单

```html
<form action="/login" method="post">
  <input name="username" value="">
  <input name="password" type="password">
  <button type="submit">登录</button>
</form>
```

`name` 就是提交到后端的参数名。测 SQL 注入、越权，先找 `name`。

### 会执行代码的地方

```javascript
document.getElementById("x").innerHTML = userInput;   // 插入 HTML，可能执行 onerror
document.write(userInput);                              // 直接写页面
eval(userInput);                                        // 执行 JS
location.href = userInput;                              // 跳转，javascript: 可执行
setTimeout(userInput, 100);                             // 字符串形式执行
```

这些叫 **sink（汇）**。用户可控的输入叫 **source（源）**，常见的有：

```text
location.search / location.hash / location.href
document.referrer
window.name
postMessage 的 event.data
localStorage / sessionStorage
```

DOM 型 XSS 就是"source 未经处理直接进了 sink"，服务端日志里可能完全看不到 payload。第 5 篇会逐个演示。

### fetch 发 JSON

```javascript
fetch("/api/user/1", {
  method: "POST",
  credentials: "include",
  headers: {"Content-Type": "application/json"},
  body: JSON.stringify({role: "admin"})
})
```

`credentials: "include"` 会带 Cookie。如果接口只靠 Cookie 鉴权、没有 CSRF token，就可能被跨站利用（能否读响应取决于 CORS，但写操作不需要读响应）。

---

## 七、本篇自检清单

能独立完成下面这些，再进第 2 篇：

- [ ] 用 `dig` 查出目标的 A、CNAME、MX、TXT 记录，并解释 CNAME 指向 CDN 意味着什么。
- [ ] 用 `curl -v` 完整看懂一次 HTTPS 请求，指出请求行、Host、Cookie、响应码。
- [ ] 用 `nc` 手动发一个 HTTP 请求，知道 `\r\n\r\n` 的作用。
- [ ] 把 `' OR 1=1-- -` 做 URL 编码，再解回来。
- [ ] 说清 `HttpOnly`、`Secure`、`SameSite` 各防什么。
- [ ] 解一个 JWT 的 payload，知道它只是 Base64 不是加密。
- [ ] 说清同源策略为什么挡不住 CSRF。
- [ ] 用 `requests.Session` 登录 DVWA 并访问登录后页面。

---

**下一篇：[《靶场搭建与 Burp 抓包》](第2篇-靶场搭建与Burp抓包.md)**：把 DVWA、sqli-labs、upload-labs、XSS-labs 和 Vulhub 跑起来，配好 Burp 的 HTTPS 证书，抓一次登录包并在 Repeater 里改参数。环境搭不对，后面每一篇都会卡住。
