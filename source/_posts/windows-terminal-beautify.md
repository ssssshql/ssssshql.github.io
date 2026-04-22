---
title: Windows11终端美化：PowerShell7 + OhMyPosh + Fastfetch
date: 2026-04-22 10:00:00
tags:
  - Windows
  - Terminal
  - PowerShell
  - OhMyPosh
categories:
  - 工具配置
---

最近把 Windows 11 的终端彻底美化了一下，告别又丑又难用的默认 CMD。这次主要用了 PowerShell 7 作为内核，Oh My Posh 做皮肤，再加上 Fastfetch 显摆一下系统信息。

整理了一下过程，方便以后重装系统参考。

### 1. 安装 PowerShell 7
虽然 Win11 自带了 PowerShell，但建议直接上最新版的 PowerShell 7，性能更好，支持也更多。

直接用 `winget` 一键搞定：
```powershell
winget install --id Microsoft.PowerShell --source winget
```

![image-20260422225549228](https://picgo.19991029.xyz/image-20260422225549228.png)

### 2. 安装 Oh My Posh
这是终端美化的灵魂。

**安装：**
```powershell
winget install JanDeDobbeleer.OhMyPosh -s winget
```

**关键点：字体！**
Oh My Posh 需要 Nerd Fonts 才能显示那些酷炫的小图标。建议安装 `Meslo LGM NF` 或者 `Cascadia Code NF`。
可以通过 Oh My Posh 命令行直接安装：

```powershell
oh-my-posh font install
```

![image-20260422225647419](https://picgo.19991029.xyz/image-20260422225647419.png)

### 3. 配置 $PROFILE
安装完还得让 PowerShell 启动时自动加载。

在终端输入：
```powershell
notepad $PROFILE
```
如果没有这个文件，选“是”创建一个。然后在里面填入：
```powershell
oh-my-posh init pwsh --config "$env:POSH_THEMES_PATH\jandedobbeleer.omp.json" | Invoke-Expression
```
*注：`jandedobbeleer.omp.json` 是默认主题，你可以去 [主题库](https://ohmyposh.dev/docs/themes) 挑个喜欢的换掉名。*

### 4. 安装 Fastfetch
Linux 上有 Neofetch（已停更），Windows 上现在流行用 Fastfetch，C 写的，速度极快。

**安装：**
```powershell
winget install fastfetch
```

想让它每次开机都显示？把 `fastfetch` 也加进上面的 `$PROFILE` 里就行：
```powershell
# 在文件最后加一行
fastfetch
```

### 5. 最终效果展示
现在打开 Windows Terminal，选择 PowerShell 7。

![2b49b5edb1bcbe30cc7a7f49554e0b92](https://picgo.19991029.xyz/2b49b5edb1bcbe30cc7a7f49554e0b92.png)

**总结：**
- **PowerShell 7**：底层。
- **Oh My Posh**：面子。
- **Fastfetch**：仪式感。
- **Nerd Fonts**：支撑。

折腾完心情都好多了，写代码也有动力了。
