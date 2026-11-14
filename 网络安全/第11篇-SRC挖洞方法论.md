---
title: 第 11 篇：SRC 挖洞方法论
layout: article
---
{% raw %}
# 第 11 篇：SRC 挖洞方法论——8 小时找到第一个高危

前面十篇是"单个漏洞怎么打"。这一篇回答一个更现实的问题：

> 给我一个完全陌生的目标，只有 8 小时，我怎么找到第一个能交的高危？

答案不是"把 AWVS 打开扫一遍"，而是**缩小范围、排优先级、按时间盒推进**。

---

## 一、先读规则，再碰目标

SRC（Security Response Center）不是"随便黑"。每次测试前确认：

- **资产范围**：哪些域名、IP、App、小程序在范围内；
- **测试类型**：是否允许扫描、爆破、上传、SSRF；
- **禁止事项**：DoS、社会工程、物理入侵、碰真实用户数据；
- **测试时间**：有些活动只允许夜间；
- **数据要求**：证明即可，不拖库、不传播、不保留；
- **重复判定**：已知问题、已公开漏洞通常不算。

**默认红线：**

```text
不拖库、不写文件、不反弹 shell、不删改数据
不碰真实用户隐私、不打扰业务、不做 DoS
只在授权资产上测试，第三方依赖先确认
```

如果读到敏感数据，立即停止，记录"能读到什么"即可，报告里打码。

---

## 二、8 小时时间盒

| 时间 | 阶段 | 产出 |
| --- | --- | --- |
| 0:00–0:30 | 读规则、建资产表 | 范围清单、排除项 |
| 0:30–1:30 | 被动收集 | 子域、证书、历史 URL |
| 1:30–2:00 | 存活 + 指纹 | 可访问资产、框架/端口 |
| 2:00–3:00 | 爬取 + 角色 | 功能地图、API、参数 |
| 3:00–5:00 | 打高概率点 | 越权、上传、SSRF、SQLi、XSS |
| 5:00–6:30 | 深挖一条线 | 把可疑点打成完整链 |
| 6:30–7:30 | 复现 + 证据 | 请求/响应、截图、视频 |
| 7:30–8:00 | 写报告 | 标题、步骤、影响、修复 |

**时间盒规则：**

- 单个功能超过 40 分钟没信号，先记到"停车场"，换下一个；
- 不要一上来就挖 RCE，先找"能稳定复现的中高危"；
- 每 60 分钟保存一次 Burp 项目，避免白干。

---

## 三、资产优先级：先打"人最多的地方"

| 优先级 | 资产 | 原因 |
| --- | --- | --- |
| P0 | 登录/注册/找回密码、支付、上传、API、SSO、管理后台 | 逻辑复杂、数据敏感、最容易出高危 |
| P1 | 搜索、导出、分享、Webhook、导入、评论 | 参数多，XSS/SQLi/SSRF 高发 |
| P2 | 用户中心、订单、消息、优惠券 | 越权和逻辑漏洞 |
| P3 | 营销页、帮助中心、静态资源 | 价值低，除非有子域接管 |
| P4 | 第三方服务、CDN、邮件网关 | 通常不在范围 |

**判断公式：**

```text
优先级 = 可达性 × 影响 × 竞争度
```

- 可达性：不用登录能不能到？要不要特殊角色？
- 影响：能读数据、改数据、拿权限、影响多少钱？
- 竞争度：首页、登录页被挖烂了，边缘子域/旧 API 反而机会大。

---

## 四、信息收集流水线

### 1. 子域

```bash
subfinder -d target.com -silent | anew subs.txt
amass enum -passive -d target.com -o amass.txt
```

证书透明日志：

```bash
curl -s 'https://crt.sh/?q=%25.target.com&output=json' \
  | jq -r '.[].name_value' | sed 's/\*\.//g' | sort -u
```

### 2. 存活与指纹

```bash
cat subs.txt | httpx -silent -mc 200,301,302,401,403 -o alive.txt
cat alive.txt | httpx -silent -title -tech-detect -status-code -o fingerprint.txt
```

### 3. 爬取 URL

```bash
cat alive.txt | katana -jc -d 2 -silent -o urls.txt
cat urls.txt | grep -E '\?[^=]+=' | sort -u > params.txt
```

历史 URL：

```bash
cat subs.txt | waybackurls | anew wayback.txt
cat subs.txt | gau --threads 5 | anew gau.txt
```

### 4. 目录/接口

```bash
ffuf -u https://target.com/FUZZ -w wordlist.txt -mc 200,301,302,401,403
```

不要用超大字典、不要高并发，避免把业务打挂。

### 5. JS 里的接口

```bash
cat alive.txt | subjs | anew js.txt
cat js.txt | grep -Eo '"/api/[^"]+"' | sort -u
```

重点看：

```text
/api/、/v1/、/v2/、/internal/、/admin/
token、secret、accessKey、appId
```

### 6. 一个可复用脚本

```bash
#!/usr/bin/env bash
set -euo pipefail

domain="${1:?usage: $0 target.com}"
out="recon/$domain"
mkdir -p "$out"

subfinder -d "$domain" -silent | anew "$out/subs.txt"
cat "$out/subs.txt" | httpx -silent -mc 200,301,302,401,403 -o "$out/alive.txt"
cat "$out/alive.txt" | httpx -silent -title -tech-detect -status-code -o "$out/finger.txt"
cat "$out/alive.txt" | katana -jc -d 2 -silent -o "$out/urls.txt"
cat "$out/urls.txt" | grep -E '\?[^=]+=' | sort -u > "$out/params.txt"

echo "done: $out"
```

需要 `subfinder`、`httpx`、`katana`、`anew`。先在小范围跑，确认不会触发防护。

---

## 五、拿到资产后，先画"功能地图"

不要急着扫漏洞，先回答：

1. 这个系统有哪些角色？匿名、普通用户、VIP、管理员、企业管理员。
2. 核心对象是什么？用户、订单、文件、发票、工单、设备。
3. 每个对象有哪些操作？增删改查、导出、分享、审批。
4. 哪些操作会改状态/花钱/给权限？
5. 哪些接口返回敏感数据？

画一张表：

| 功能 | 角色 | 接口 | 参数 | 风险点 |
| --- | --- | --- | --- | --- |
| 查看订单 | 用户 | `GET /api/order/{id}` | `id` | IDOR |
| 上传头像 | 用户 | `POST /api/avatar` | `file` | 上传绕过 |
| 邀请好友 | 用户 | `POST /api/invite` | `code` | 批量赋值/刷奖励 |
| 导出报表 | 管理员 | `GET /api/export` | `type`、`path` | 越权/穿越 |
| 图片代理 | 用户 | `GET /api/proxy` | `url` | SSRF |

这张表比"扫一遍"有用得多。

---

## 六、按功能选漏洞：别用一把锤子敲所有钉子

| 功能 | 优先测 | 典型 payload/操作 |
| --- | --- | --- |
| 登录/注册 | 用户枚举、限流、JWT、OAuth | 改 `redirect_uri`、`alg:none` |
| 找回密码 | token 预测、Host 头、复用 | 改 `Host`、改 `email` |
| 上传 | 扩展名、MIME、内容、路径 | `shell.php`、`GIF89a...` |
| API | IDOR、批量赋值、CORS | 换 ID、加 `is_admin` |
| 搜索 | XSS、SQLi、SSTI | `{{7*7}}`、`'`、`<script>` |
| URL 抓取 | SSRF | `127.0.0.1`、`169.254.169.254` |
| 导出/下载 | 越权、目录穿越 | `../../etc/passwd` |
| 支付 | 价格篡改、优惠券、回调 | `price=0.01`、并发 |
| 管理后台 | 垂直越权、未授权 | 普通用户 Cookie 访问 |
| Webhook | SSRF、签名绕过 | 自己服务器收请求 |
| 文件预览 | XXE、SSRF、LFI | `file://`、外部实体 |

### 按技术栈选

**Java：**

```text
Shiro：rememberMe=deleteMe
Fastjson：@type
Log4j：${jndi:
Spring：/actuator、/env、/heapdump
Swagger：/swagger-ui.html、/v2/api-docs
Druid：/druid/index.html
```

**PHP：**

```text
ThinkPHP：X-Powered-By、invokefunction
Laravel：APP_KEY、/.env、debug 页面
上传/包含：upload-labs、php://filter
```

**Node/前端：**

```text
原型链污染：__proto__、constructor.prototype
JWT：弱密钥、alg 混淆
SPA：source map、JS 里的 /api/
GraphQL：introspection、node(id:)
```

**企业/运维：**

```text
OA、VPN、SSO、Jenkins、GitLab、Zabbix
默认口令、未授权、已知 CVE
```

### 指纹命令

```bash
curl -sI https://target.com | grep -Ei 'server|x-powered-by|set-cookie'
whatweb -a 3 https://target.com
```

favicon 哈希、错误页、Cookie 名都能辅助识别。

---

## 七、从低危到高危：学会"升级"

很多洞单独看是低危，串起来就是高危：

| 起点 | 升级路径 | 终点 |
| --- | --- | --- |
| 信息泄露 | 拿到内网地址/账号 | 登录后台 |
| IDOR | 遍历到管理员/订单 | 数据泄露/越权操作 |
| SSRF | 打 Redis/元数据 | RCE/云凭证 |
| 上传 | 配合 LFI/解析漏洞 | RCE |
| XSS | 配合 CSRF | 管理员操作 |
| 开放重定向 | 配合 OAuth | token 窃取 |
| 子域接管 | 共享 Cookie/邮件 | 账号接管 |
| 弱口令 | 登录运维平台 | 内网渗透 |

**升级的原则：**

- 每一步都要有证据，不要"可能可以"；
- 不要为了升级而越界（比如拿 SSRF 去打内网数据库）；
- 报告里写"影响链"，但只证明到必要的一步。

---

## 八、避免重复：先查再挖

1. 看平台已公开的漏洞报告，搜目标域名、功能名。
2. Google：

```text
site:target.com inurl:api
site:target.com "swagger"
site:target.com "accessKey"
```

3. GitHub：

```text
"target.com" password
"target.com" api_key
```

4. 看目标的历史公告、补丁、更新日志。
5. 如果发现是已知问题，找**变体**：
   - 移动端 API；
   - 旧版本 `/v1/`；
   - 不同角色/租户；
   - 不同参数名；
   - 批量接口；
   - 导出/下载接口。

**重复不可怕，可怕的是交上去才发现是"已知问题"。**

---

## 九、证据与记录

每个可疑点都记录：

```markdown
## 2026-11-14 10:32 /api/order/1002

- 账号：alice（自己注册）
- 请求：GET /api/order/1002
- 响应：200，返回 bob 的订单
- 对比：1001 返回 alice 的订单
- 结论：水平越权
- 证据：burp/idor-order.png、response.json
```

要求：

- Burp 项目按目标/日期保存；
- 截图带 URL、时间、账号；
- 请求/响应原文保存；
- 复杂链录屏；
- 敏感值打码；
- 不保存真实用户数据到本地。

---

## 十、常见坑

| 坑 | 后果 | 避免 |
| --- | --- | --- |
| 测了范围外资产 | 违规、封号 | 先读 scope |
| 高并发扫描 | 业务雪崩 | 限速、小字典 |
| 把 WAF 页当漏洞 | 误报 | 看响应特征、换 payload |
| 只测前端 | 漏 API | 抓包、看 JS |
| 不换角色 | 漏越权 | 至少两个账号 |
| 不查重 | 白干 | 先搜公开报告 |
| 上传 webshell | 越界 | 只传 `echo 7*7` |
| `sqlmap --dump` | 拖库 | 只证明，不导出 |
| 报告没步骤 | 被驳回 | 附请求/响应 |

---

## 十一、一篇可执行的 8 小时计划

**上午：**

1. 读规则，列资产，排除第三方。
2. 子域 + 存活 + 指纹，标出 P0/P1。
3. 用浏览器走一遍主流程，画功能地图。
4. 抓登录、下单、上传、导出四类请求。

**下午：**

5. 用两个账号测 IDOR/越权。
6. 对上传、URL 抓取、导出各测 20 分钟。
7. 对搜索/评论测 XSS/SQLi/SSTI。
8. 选一个最有希望的点，深挖 60–90 分钟。
9. 复现、截图、写报告。
10. 清理本地敏感数据，提交。

---

## 十二、本篇自检清单

- [ ] 测试前能列出 scope、禁止项、数据要求。
- [ ] 能画出目标的功能地图（角色/对象/操作/接口）。
- [ ] 会用 subfinder + httpx + katana 跑一条流水线。
- [ ] 能按 P0–P4 给资产排优先级。
- [ ] 知道 Java/PHP/Node 各自先看什么组件。
- [ ] 能说出 5 条"低危升高危"的链。
- [ ] 会查公开报告，避免重复。
- [ ] 每个发现都有请求/响应/截图。
- [ ] 知道 8 小时怎么分配，不会在一个点上死磕。
- [ ] 能守住"只证明、不拖库、不破坏"的边界。

---


{% endraw %}

**下一篇：[《漏洞报告与沟通》](第12篇-漏洞报告与沟通.md)**：挖到洞只是一半，写清楚、说明白、让厂商能复现，才算真正交付。那篇讲报告结构、标题怎么写、影响怎么量化、和厂商怎么沟通、被驳回怎么办。
