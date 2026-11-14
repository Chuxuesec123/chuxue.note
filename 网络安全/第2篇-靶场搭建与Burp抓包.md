---
title: 第 2 篇：靶场搭建与 Burp 抓包
layout: article
---
# 第 2 篇：把靶场跑起来，把 Burp 配到能抓 HTTPS

这一篇全是动手。目标只有一个：**浏览器发出的请求，你能在 Burp 里看到、改掉、重发，并且服务器真的按你改的返回。**

环境建议分两层：虚拟机里跑 Kali（攻击机），Docker 里跑靶场（受害机）。两者用 Host-Only 或 NAT 网络互通，靶场绝对不要暴露到公网。

---

## 一、虚拟机：先做一个能回滚的快照

1. 装 VirtualBox 或 VMware Workstation Player。
2. 从 <https://www.kali.org/get-kali/> 下载 Kali 虚拟机镜像（Pre-built），导入后默认账号密码通常是 `kali/kali`。
3. 开机第一件事：`sudo apt update && sudo apt full-upgrade -y`，然后关机做快照，命名 `clean`。
4. 网络模式选 **NAT** 或 **Host-Only**。NAT 能上网，Host-Only 只能和宿主机通信，更安全。

验证网络：

```bash
ip a
ping -c 3 8.8.8.8
```

Kali 里常用的基础包：

```bash
sudo apt install -y burpsuite sqlmap ffuf dirsearch seclists jq git curl
```

`seclists` 的字典在 `/usr/share/seclists/`，后面爆破目录、子域、密码都从这里取。

---

## 二、Docker + Vulhub：一条命令起一个 CVE 靶场

Vulhub 把常见漏洞环境打包成 Docker Compose，适合复现组件漏洞（第 9 篇会大量用到）。

```bash
sudo apt install -y docker.io docker-compose-v2
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
# 退出重新登录，让 docker 组生效
docker version
```

拉 Vulhub：

```bash
git clone https://github.com/vulhub/vulhub.git
cd vulhub
ls | head
```

以 Log4j2 为例：

```bash
cd log4j/CVE-2021-44228
docker compose up -d
docker ps
```

输出里会看到端口映射，比如 `0.0.0.0:8983->8983/tcp`。浏览器访问 `http://127.0.0.1:8983` 就能看到 Solr 页面。

**用完必须关：**

```bash
docker compose down -v
```

`-v` 连数据卷一起删，下次起来是干净环境。Vulhub 的靶场大多含真实 RCE，**不要改端口映射到公网 IP**，也不要在有敏感数据的宿主机上跑。

### 检查端口有没有暴露到外网

```bash
ss -lntp | grep -E '8983|8080|6379'
```

如果看到 `0.0.0.0:8983`，说明所有网卡都能访问。只在本机练，改成 `127.0.0.1:8983:8983` 更稳妥。

---

## 三、DVWA / sqli-labs / XSS-labs / upload-labs

这四个是 Web 漏洞练习的主力，推荐用 Docker 或 phpStudy 起。

### 方式 A：Docker（推荐 Linux / macOS）

```bash
# DVWA
docker run -d --name dvwa -p 127.0.0.1:8080:80 vulnerables/web-dvwa

# sqli-labs
docker run -d --name sqli -p 127.0.0.1:8081:80 acgpiano/sqli-labs

# XSS-labs
docker run -d --name xss -p 127.0.0.1:8082:80 vulfocus/xss-labs

# upload-labs
docker run -d --name upload -p 127.0.0.1:8083:80 c0ny1/upload-labs
```

起完 `docker ps` 确认状态是 `Up`。首次访问 DVWA：

1. 打开 `http://127.0.0.1:8080`，默认 `admin / password`；
2. 点 **Create / Reset Database**；
3. 进 **DVWA Security**，把等级改成 `low`；
4. 后面每篇练完可以调 `medium` / `high` 再打一遍。

### 方式 B：Windows + phpStudy

下载 phpStudy，把 DVWA、sqli-labs 解压到 `WWW` 目录，建站点、绑域名（如 `dvwa.local`），改 `hosts`：

```text
127.0.0.1 dvwa.local
127.0.0.1 sqli.local
```

phpStudy 默认 MySQL 账号 `root/root`，DVWA 的 `config/config.inc.php` 里改成对应密码。**只在本机用，不要把 phpStudy 的 3306 映射出去。**

### 四个靶场分别练什么

| 靶场 | 练什么 | 对应篇目 |
| --- | --- | --- |
| DVWA | SQLi、XSS、CSRF、文件包含、上传、命令注入，难度可调 | 4 ～ 8 |
| sqli-labs | 65 关 SQL 注入，覆盖 union、报错、布尔、时间、堆叠 | 4 |
| XSS-labs | 20 关反射型 XSS 过滤绕过 | 5 |
| upload-labs | 21 关上传绕过 | 7 |

---

## 四、Burp Suite：代理、证书、Repeater

Burp Community 免费版够用，Intruder 有速率限制，但 Repeater 不限。

### 1. 设置代理监听

打开 Burp → **Proxy → Proxy settings**：

- 监听地址 `127.0.0.1:8080`；
- 勾选 `Enable proxy`；
- 如果浏览器和 Burp 不在同一台机器，监听地址要改成 `0.0.0.0`，同时确认防火墙只放行你的 IP。

### 2. 浏览器挂代理

Firefox 装 **FoxyProxy**，新建代理：

```text
Title: Burp
Proxy Type: HTTP
Host: 127.0.0.1
Port: 8080
```

只对测试目标启用，别全局代理——否则日常上网的流量全进 Burp，既卡又容易误抓。

### 3. 装 CA 证书，才能抓 HTTPS

浏览器访问：

```text
http://burp
```

点 **CA Certificate** 下载 `cacert.der`，然后：

- Firefox：设置 → 隐私与安全 → 证书 → 查看证书 → 证书颁发机构 → 导入，勾选"信任由此证书颁发机构标识的网站"。
- Chrome：设置 → 隐私和安全 → 安全 → 管理证书 → 受信任的根证书颁发机构 → 导入。
- 系统级（Linux）：把 `.der` 转成 `.crt` 后复制到 `/usr/local/share/ca-certificates/`，再 `sudo update-ca-certificates`。

**注意：** 只给自己的测试浏览器/虚拟机装 Burp 证书。不要在日常主力机全局信任它，Burp 能解密你所有 HTTPS 流量。

### 4. 抓第一个包

1. 浏览器打开 `http://127.0.0.1:8080/login.php`；
2. Burp → Proxy → HTTP history，找到 `POST /login.php`；
3. 右键 → **Send to Repeater**（快捷键 `Ctrl+R`）；
4. 在 Repeater 里把 `password` 改成 `wrongpass`，点 **Send**；
5. 观察响应长度、状态码、页面内容变化。

这一步看起来简单，但它是后面所有手工测试的基本动作：**改一个参数 → 发一次 → 对比响应**。

### 5. 常用面板

| 面板 | 用途 |
| --- | --- |
| Proxy → HTTP history | 看所有经过的请求，按 URL、状态码、MIME 过滤 |
| Repeater | 手工改包、重放、对比响应 |
| Intruder | 爆破、枚举；Community 版有速率限制 |
| Decoder | URL、Base64、Hex、HTML 编码转换 |
| Comparer | 对比两次响应的差异 |
| Organizer | 把有价值的请求存起来，按项目分类 |
| Logger | 记录所有工具发出的请求，方便回溯 |

### 6. 项目设置里的三个关键开关

- **Target → Scope**：把目标域名加进去，只显示范围内流量。测 SRC 时这一步能防止误点第三方链接。
- **Proxy → Options → Intercept Client Requests**：可以设置只拦截特定后缀，比如只拦 `.php`、`/api/*`。
- **Proxy → Options → Match and Replace**：批量改请求头，比如统一加 `X-Forwarded-For: 127.0.0.1`，测 IP 限制绕过很方便。

---

## 五、抓不到包？按这张表排查

| 现象 | 原因 | 解决 |
| --- | --- | --- |
| HTTP 能抓，HTTPS 提示证书错误 | 没装 Burp CA | 访问 `http://burp` 下载并信任 |
| 浏览器完全没流量 | 代理没开/端口不对 | 检查 FoxyProxy、Burp 监听、`curl -x` |
| 只有 localhost 抓不到 | 浏览器/系统绕过本地代理 | 用 `127.0.0.1` 而非 `localhost`，或关闭系统代理例外 |
| 手机 App 抓不到 | 用了证书固定（SSL Pinning） | 需 Frida/objection 绕过，仅限授权测试 |
| Burp 里全是 304 | 浏览器缓存 | DevTools 勾 Disable cache，或 Burp 里清缓存 |
| WebSocket 没显示 | 需要单独看 | Proxy → WebSockets history |
| 请求发出去但响应是 502 | 上游代理/目标不通 | 先用 `curl` 直连确认目标可达 |

命令行验证代理是否生效：

```bash
curl -x http://127.0.0.1:8080 -I http://127.0.0.1:8080
curl -x http://127.0.0.1:8080 -I https://example.com
```

第二条如果报证书错误，说明系统没信任 Burp CA；加 `-k` 可以跳过校验，但只建议在实验环境用。

---

## 六、把 sqlmap、ffuf、nuclei 装上

```bash
# sqlmap
sudo apt install -y sqlmap
sqlmap --version

# ffuf
sudo apt install -y ffuf
ffuf -V

# dirsearch
git clone https://github.com/maurosoria/dirsearch.git ~/tools/dirsearch
cd ~/tools/dirsearch && pip3 install -r requirements.txt

# ProjectDiscovery 三件套
go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest
go install -v github.com/projectdiscovery/httpx/cmd/httpx@latest
go install -v github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest
export PATH=$PATH:$(go env GOPATH)/bin
```

没有 Go 环境就用 Release 二进制：

```bash
mkdir -p ~/tools && cd ~/tools
# 去 https://github.com/projectdiscovery/nuclei/releases 复制最新 linux_amd64 链接
wget https://github.com/projectdiscovery/nuclei/releases/latest/download/nuclei_3.x.x_linux_amd64.zip
unzip nuclei_3.x.x_linux_amd64.zip
sudo mv nuclei /usr/local/bin/
```

版本号会变，以 Release 页面为准。装完先跑帮助，确认能执行：

```bash
nuclei -h | head
```

---

## 七、第一个完整练习：DVWA 登录 → 改包 → 看差异

1. 浏览器访问 `http://127.0.0.1:8080`，Burp 开拦截；
2. 输入 `admin / password` 登录，抓到 `POST /login.php`；
3. 看请求体：

```http
username=admin&password=password&Login=Login&user_token=xxxxx
```

4. 右键 Send to Repeater；
5. 把 `password` 改成 `wrongpass`，Send，记录响应长度；
6. 改回 `password`，Send，记录响应长度；
7. 对比两次响应，找到"登录成功/失败"的差异字符串。

这个差异字符串就是后面爆破、盲注判断的基线。**先建立"正常长什么样"，才知道"异常长什么样"。**

### 再练一次：改安全等级

DVWA 的 `security.php` 提交 `security=low`。在 Burp 里抓这个请求，改成 `security=medium`，看首页右上角等级有没有变。这个动作能让你熟悉"参数在 POST 体里、服务端直接信任"的逻辑漏洞雏形。

### 用 Intruder 跑一次字典（只在靶场）

1. 抓到 `POST /login.php`，Send to Intruder；
2. 清掉自动标记，只选中 `password` 的值，点 **Add §**；
3. Payloads 选 Simple list，粘贴几个密码：

```text
password
123456
admin
letmein
```

4. 点 Start attack；
5. 按 Length 排序，长度不同的那条通常就是成功。

Community 版会限速，跑几十条就够验证流程，别拿它跑大字典。真实项目里，登录口爆破前先看规则是否允许，很多 SRC 明确禁止。

---

## 八、环境安全清单

- [ ] 靶场只监听 `127.0.0.1`，或虚拟机 Host-Only 网络。
- [ ] Docker 端口映射写成 `127.0.0.1:8080:80`，不要写 `8080:80`。
- [ ] 云主机安全组不开放 8080、3306、6379。
- [ ] Kali 快照做好，打坏了直接回滚。
- [ ] Burp 证书只装在测试浏览器/虚拟机。
- [ ] 测完 `docker compose down -v`，别把含 webshell 的环境长期跑着。

---

**下一篇：[《信息收集与攻击面》](第3篇-信息收集与攻击面.md)**：环境有了，面对一个授权目标，第一步不是掏 sqlmap，而是把资产摸清楚——子域、端口、指纹、JS 接口、历史 URL、GitHub 泄露。信息收集做得好，后面手工测试才有地方下嘴。
