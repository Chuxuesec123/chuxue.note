# chuxue.note

记录我在网络安全、网络工程、操作系统和日常计算机操作中实际踩过并解决的问题。

**站点首页**：<https://chuxuesec123.github.io/chuxue.note/>

这一页就是全站目录：下面每个标题、每个文件名都能点进去。文章页是干净的单栏排版——顶部只有站点名和一个「← 目录」按钮，读完底部有「上一篇 / 下一篇」，随时能退回这里。

---

## 📚 数据结构和 Python · 考前速成

零基础、应试向。每篇都是「概念 → 可运行代码 → 逐块解释 → 常见错误 → 考题演练」的写法，代码全部实测跑过。**建议按 01 → 08 的顺序读。**

| # | 篇目（点击阅读） | 这篇讲什么 |
| :-: | --- | --- |
| 01 | [Python 速成](数据结构和python/第1篇-Python速成.md) | 变量、列表、字典、循环、函数、类——只讲后面几篇真正用得上的最小语法 |
| 02 | [复杂度与线性表](数据结构和python/第2篇-复杂度与线性表.md) | 大 O 怎么数；Python 列表当顺序表来解剖，为什么尾部快、头部慢 |
| 03 | [栈与队列](数据结构和python/第3篇-栈与队列.md) | 后进先出 vs 先进先出，括号匹配、表达式求值两道必考题 |
| 04 | [链表](数据结构和python/第4篇-链表.md) | 单链表、双链表、快慢指针、反转、环检测、合并 |
| 05 | [树与二叉树](数据结构和python/第5篇-树与二叉树.md) | 五个必背性质、三种遍历、由遍历序列还原树、BST、堆 |
| 06 | [图](数据结构和python/第6篇-图.md) | 邻接矩阵 vs 邻接表、DFS/BFS、Dijkstra、最小生成树、拓扑排序 |
| 07 | [查找](数据结构和python/第7篇-查找.md) | 顺序 / 二分 / 哈希表；判定树、ASL 计算、装填因子、冲突处理，计算题全在这 |
| 08 | [排序](数据结构和python/第8篇-排序.md) | 八大排序一张表：复杂度、稳定性、场景选型，外加"第一趟结果"怎么写 |

> **全 8 篇已完结。** 从列表下标讲到八大排序，考前直接看每篇的"小结"和"常见错误"最快。

## 🛠 工具与环境

| 篇目（点击阅读） | 这篇讲什么 |
| --- | --- |
| [🐱 猫猫的 VS Code 发光改造手记](vscode美化分享.md) | 把编辑器调成赛博霓虹风：主题、字体、透明、发光效果一步步来 |

---

## 🗂 全仓库文件

> 站内页用相对路径（GitHub 和 Pages 都能点）；配置、布局、样式这些文件 GitHub Pages 不会渲染成网页，给的是 GitHub 上的文件链接。

### 文章

| 文件 | 是什么 | 打开 |
| --- | --- | --- |
| `README.md` | 本文件 = 站点首页 = 全站目录 | 你正在看 |
| `vscode美化分享.md` | VS Code 发光美化手记 | [站内阅读](vscode美化分享.md) |
| `数据结构和python/第1篇-Python速成.md` | 01 Python 速成 | [站内阅读](数据结构和python/第1篇-Python速成.md) |
| `数据结构和python/第2篇-复杂度与线性表.md` | 02 复杂度与线性表 | [站内阅读](数据结构和python/第2篇-复杂度与线性表.md) |
| `数据结构和python/第3篇-栈与队列.md` | 03 栈与队列 | [站内阅读](数据结构和python/第3篇-栈与队列.md) |
| `数据结构和python/第4篇-链表.md` | 04 链表 | [站内阅读](数据结构和python/第4篇-链表.md) |
| `数据结构和python/第5篇-树与二叉树.md` | 05 树与二叉树 | [站内阅读](数据结构和python/第5篇-树与二叉树.md) |
| `数据结构和python/第6篇-图.md` | 06 图 | [站内阅读](数据结构和python/第6篇-图.md) |
| `数据结构和python/第7篇-查找.md` | 07 查找 | [站内阅读](数据结构和python/第7篇-查找.md) |
| `数据结构和python/第8篇-排序.md` | 08 排序 | [站内阅读](数据结构和python/第8篇-排序.md) |

### 站点配置与主题（改样式看这几个文件）

| 文件 | 作用 | 打开 |
| --- | --- | --- |
| `_config.yml` | 站点标题、描述、主题（minima）、插件，以及生成上下篇用的 `reading_order` | [GitHub](https://github.com/Chuxuesec123/chuxue.note/blob/main/_config.yml) |
| `assets/main.scss` | **全站样式**：配色、中文正文排版、标题层级、代码块、表格、页面底部导航 | [GitHub](https://github.com/Chuxuesec123/chuxue.note/blob/main/assets/main.scss) |
| `_layouts/article.html` | 文章页布局：阅读进度条 + 上一篇/下一篇 + 返回目录 | [GitHub](https://github.com/Chuxuesec123/chuxue.note/blob/main/_layouts/article.html) |
| `_layouts/home.html` | 首页（本页）布局 | [GitHub](https://github.com/Chuxuesec123/chuxue.note/blob/main/_layouts/home.html) |
| `_layouts/page.html` | 普通页面布局 | [GitHub](https://github.com/Chuxuesec123/chuxue.note/blob/main/_layouts/page.html) |
| `_includes/header.html` | 顶栏：站点名 + 「← 目录」按钮（刻意不放菜单） | [GitHub](https://github.com/Chuxuesec123/chuxue.note/blob/main/_includes/header.html) |
| `_includes/footer.html` | 页脚（一行） | [GitHub](https://github.com/Chuxuesec123/chuxue.note/blob/main/_includes/footer.html) |
| `_includes/article-nav.html` | 按 `reading_order` 生成「上一篇 / 下一篇」 | [GitHub](https://github.com/Chuxuesec123/chuxue.note/blob/main/_includes/article-nav.html) |
| `.github/workflows/jekyll-gh-pages.yml` | push 到 main 后自动构建并发布到 GitHub Pages | [GitHub](https://github.com/Chuxuesec123/chuxue.note/blob/main/.github/workflows/jekyll-gh-pages.yml) |

仓库结构：

```text
chuxue.note/
├── README.md                    ← 你在这里（GitHub Pages 首页 = 全站目录）
├── vscode美化分享.md
├── 数据结构和python/
│   ├── 第1篇-Python速成.md
│   ├── 第2篇-复杂度与线性表.md
│   ├── 第3篇-栈与队列.md
│   ├── 第4篇-链表.md
│   ├── 第5篇-树与二叉树.md
│   ├── 第6篇-图.md
│   ├── 第7篇-查找.md
│   └── 第8篇-排序.md
├── _config.yml                  ← 站点配置 / 上下篇顺序
├── _layouts/                    ← article / page / home 三个布局
├── _includes/                   ← header / footer / article-nav
├── assets/
│   └── main.scss                ← 全部样式（正文排版在这一份里）
└── .github/workflows/
    └── jekyll-gh-pages.yml      ← 自动部署
```

## 关于本站

- 用 **GitHub Pages + Jekyll** 部署，主题是 `minima`，样式在 `assets/main.scss` 里覆盖：中文全文 17px / 行高 1.85、正文行长限制在 760px、`##` 章节标题带主色竖条、宽表格可横向滚动、代码块统一浅灰底。
- **加导航**：在本文件的表格里加一行（就是上面「全仓库文件」那张表）。
- **加文章的上下篇**：再把这篇登记到 `_config.yml` 的 `reading_order` 里；没登记的页面（比如工具类笔记）只当独立文章，不会出现在上一篇/下一篇里。
- 笔记都是我自己踩坑后整理的，代码可以直接复制运行（Python 3）。
