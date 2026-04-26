---
title: 删除 Shift+右键“在此处打开 PowerShell 窗口”记录
date: 2026-04-27 02:48:05
tags: [Windows, 注册表, PowerShell, 故障排查]
---

最近想把 `Shift + 鼠标右键` 里的 **“在此处打开 PowerShell 窗口”** 去掉，结果按网上常见方法删注册表项一直失败。

后面才发现不是路径找错，而是权限问题：对应注册表项的所有者是 **TrustedInstaller**，不是管理员账号，所以看起来能看到项，但实际上没权限删。

这篇记录一下完整步骤，避免下次再踩坑。

## 现象

在注册表里定位到相关项后，右键删除会报错，常见提示类似：

> 无法删除项：删除键时出错

或者会提示权限不足。

## 原因

该菜单项对应的注册表键默认由 **TrustedInstaller** 拥有。

即使你是管理员打开的注册表编辑器，只要没有接管该键的所有权并授予权限，删除操作仍然会失败。

## 解决方案

### 1）打开注册表并定位键

按 `Win + R`，输入 `regedit` 回车。

定位到（常见位置）：

```text
HKEY_CLASSES_ROOT\Directory\shell\Powershell
HKEY_CLASSES_ROOT\Directory\Background\shell\Powershell
```

> 不同系统版本可能略有差异，建议在注册表中搜索关键字 `Powershell` + `Extended`。

### 2）先接管所有者

1. 右键目标键（如 `Powershell`）→ **权限**
2. 点击 **高级**
3. 在“所有者”位置点击 **更改**
4. 输入你的管理员账号（或 `Administrators`）并确认
5. 勾选“替换子容器和对象的所有者”（如果有）

### 3）给管理员授予完全控制

1. 回到“权限”窗口
2. 选中你的管理员账号（或 `Administrators`）
3. 勾选 **完全控制** → 应用

### 4）删除对应菜单项

权限生效后，再删除对应的 `Powershell` 键（或其子项）即可。

删除后，刷新资源管理器（或重启）再按 `Shift + 右键`，菜单项应该就消失了。

## 补充说明

- 这次问题的关键点不是“找不到注册表路径”，而是 **所有权在 TrustedInstaller**。
- 很多“删不掉”本质是权限问题，先看所有者和 ACL 比反复重试更有效。
- 改注册表前建议先导出备份对应项，避免误删后难恢复。

![image-20260427024617556](https://picgo.19991029.xyz/image-20260427024617556.png)

## 参考链接

- [修改Shift右键在此处打开PowerShell窗口为在此处打开命令窗口](https://www.frogchou.com/2017/09/22/%E4%BF%AE%E6%94%B9shift%E5%8F%B3%E9%94%AE%E5%9C%A8%E6%AD%A4%E5%87%BA%E6%89%93%E5%BC%80powershell%E7%AA%97%E5%8F%A3%E4%B8%BA%E5%9C%A8%E6%AD%A4%E5%87%BA%E6%89%93%E5%BC%80/)
