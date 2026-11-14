---
title: 猫猫的 VS Code 发光改造手记
layout: article
---
# 🐱 猫猫的 VS Code 发光改造手记：把代码编辑器变成赛博霓虹小窝
嗨呀，是我，一只喜欢在键盘上踩来踩去的猫猫。

今天要带你把我家铲屎官的 VS Code，从"灰扑扑的写字板"改造成**会发光、有背景、点一下还带弹跳**的赛博小窝。

四步，一步步来，命令和截图都在下面。

不过先说清楚：这套美化会让插件去改 VS Code 的核心文件，好看是真的好看，风险也是真的。**文末那段「翻车预警」，建议动手前先看一眼。**

## 🎨 第一步：让代码自己发光
**插件：SynthWave '84（作者 Robb Owen）**

这一步是整套美化的灵魂。

SynthWave '84 默认只给你配色，**霓虹光晕要手动开**。

**操作**

1. 打开扩展面板（`Ctrl+Shift+X`），搜索 `SynthWave '84`，认准作者 **Robb Owen**，安装。
2. 装完先把配色主题切成 **SynthWave '84**（命令面板搜 `Color Theme`）——发光只在这个主题下生效。
3. 按 `Ctrl+Shift+P` 打开命令面板，输入 `Enable Neon Dreams`，回车执行。
4. 它会提示你重启。重启之后，代码里的关键字就会泛起粉紫色的光。

**猫猫提示**：Windows 用户如果开不起来，试试右键 VS Code 图标选"以管理员身份运行"再执行一次。每次 VS Code 更新后，发光会失效，重新执行一遍 `Enable Neon Dreams` 就好。

想调光晕亮度？打开 `settings.json`，加一行：

```json
"synthwave84.brightness": 0.45
```

数字从 0 到 1，0 是全透明，默认 0.45。**改完要再执行一次 **`Enable Neon Dreams`** 才会生效。** 别调太高，猫猫眼睛会瞎。

![](https://cdn.nlark.com/yuque/0/2026/png/55623416/1794584644587-c3afa843-58f5-4f64-a1cf-b7a75a2a6a05.png)

## 🖼️ 第二步：给整个窗口铺上背景
**插件：VS Code - Background Cover**

光有发光代码还不够，背景空着多寂寞。

**操作**

1. 扩展商店搜索 `VS Code - Background Cover`，安装。
2. 安装完成后，**完全关闭 VS Code 再重新打开**（不是关窗口，是彻底退出）。
3. 点左侧活动栏的 `Background Cover` 图标打开配置面板（也可以在命令面板执行 `backgroundCover - start`），选你的图片或视频，调一下透明度和模糊度。
4. 之后想换图，点底部状态栏的小按钮就行。

**猫猫提示**：这个插件也是改核心文件的，VS Code 可能弹"安装损坏"的提示，插件已经自动帮你屏蔽了，不用管它。如果之后 VS Code 打不开了，去安装目录下 `resources/app/out/vs/workbench/` 里，把 `workbench.desktop.main.js.bak` 改回原名就能复活。

看看本喵的背景图：

![](https://cdn.nlark.com/yuque/0/2026/png/55623416/1794584695940-7bbcbb59-eb91-4d51-98a1-81f4edbd9ac5.png)

## 🐾 第三步：让文件图标也发光
**插件：Seti Folder（图标主题）+ Custom UI Style（样式注入）**

代码亮了，背景有了，文件树还黑乎乎的，不像话。

**操作**

1. 扩展商店安装 `Seti Folder`（图标主题）和 `Custom UI Style`（CSS 注入工具）。
2. `Ctrl+Shift+P` 输入 `File Icon Theme`，选 **Seti Folder**。
3. 打开 `settings.json`，加入这段：

```json
"custom-ui-style.stylesheet": {
    ".monaco-icon-label.file-icon::before": "filter: drop-shadow(0 0 2.5px currentColor);"
}
```

4. `Ctrl+Shift+P` 执行 `Custom UI Style: Reload`，或者彻底重启 VS Code。

原理很简单：Seti Folder 给每种文件类型分配了颜色，`currentColor` 拿到这个颜色，`drop-shadow` 就照着这个颜色晕开一圈光。想更亮就把 `2.5px` 改大，猫猫建议别超过 `4px`。

![](https://cdn.nlark.com/yuque/0/2026/png/55623416/1794584751410-6f51b5bd-c857-440a-a65b-e581e4d4a3f1.png)

## ✨ 第四步：让界面动起来
**插件：VSCode Animations**

最后给交互加点小动作，不然太安静了。

**操作**

1. 扩展商店搜索 `VSCode Animations`，安装。
2. 安装过程中它会提示你装一个"注入器"，选 **Custom UI Style**（上一步已经装了，直接确认就行）。想换注入器，去命令面板执行 `Animations: Change Install Method`——**两个注入器同时装会打架。**
3. 按提示重载窗口。
4. `Ctrl+Shift+P` 输入 `Animations: Enable Animations`，执行。

之后命令面板弹出会有缩放/滑入效果，侧边栏调整大小也会平滑过渡。

想关掉就执行 `Animations: Disable Animations`：动画会立刻停掉，不用重载窗口；不过它只是把开关关上，不会卸载已经注入的脚本。

**猫猫提示**：如果你觉得光标动画怪怪的，去设置里搜 `Editor: Cursor Blinking`，改成 `smooth`，会顺眼很多。

![](https://cdn.nlark.com/yuque/0/2026/gif/55623416/1794585419987-04be4e3b-cd88-43a4-aeea-96978ca810dd.gif)

## 🐱 猫猫的翻车预警
这套方案里，**SynthWave '84 的霓虹、Background Cover、Custom UI Style、VSCode Animations（靠 Custom UI Style 注入）**，都会碰到 VS Code 的核心文件。猫猫的建议是：

1. **一次只开一个效果，确认稳定了再加下一个**，别一口气全上。
2. 如果 VS Code 弹"安装损坏"：Background Cover 会自动屏蔽；SynthWave 只是弹个提示，按它说的忽略掉就行，不用慌。
3. **VS Code 每次更新后**，发光、背景、图标光晕、动画都可能失效，重新执行对应的启用命令，或者重新设置一遍。
4. 真打不开的时候，别急着乱改名：SynthWave 用 `Synthwave '84: Disable Neon Dreams` 关掉；Background Cover 才走 `workbench.desktop.main.js.bak` 那条路；Custom UI Style 有 `Custom UI Style: Rollback`。**别拿旧版本的备份，去覆盖刚更新完的文件。**

改完重启一下 VS Code，就是上面那样了。真翻车了，按第 4 条那三个命令往回退，安装目录里的 `.bak` 文件别删。
