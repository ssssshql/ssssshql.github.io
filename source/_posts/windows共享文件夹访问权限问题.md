---
title: Windows 访问共享文件夹提示"组织的安全策略阻止未经身份验证的来宾访问"
date: 2026-04-13 10:02:00
tags: [Windows, 故障排查]
---

在公司内网访问共享文件夹时，突然弹出一个提示：

> 你不能访问此共享文件夹，因为你组织的安全策略阻止未经身份验证的来宾访问。

这个问题在 Windows 10/11 中比较常见，主要是系统的安全策略限制了不安全的来宾登录。下面记录几种解决方案。

## 方法一：通过组策略编辑器（推荐）

适用于 Windows 10/11 专业版、企业版、教育版。

### 步骤

1. 按 `Win + R`，输入 `gpedit.msc`，打开组策略编辑器

2. 依次展开：
   ```
   计算机配置 → 管理模板 → 网络 → Lanman工作站
   ```

3. 在右侧找到 **"启用不安全的来宾登录"**，双击打开

   ![image-20260413102242667](https://picgo.19991029.xyz/image-20260413102242667.png)

4. 选择 **"已启用"**，点击确定

5. 以管理员身份运行命令提示符或 PowerShell，执行：
   ```cmd
   gpupdate /force
   ```
   等待提示"计算机策略更新成功"

重新访问共享文件夹，应该就可以了。

## 方法二：修改注册表

如果方法一无效，或者你的系统是 Windows 家庭版，可以尝试修改注册表。

### 步骤

1. 按 `Win + R`，输入 `regedit`，打开注册表编辑器

2. 定位到：
   ```
   HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\LanmanWorkstation\Parameters
   ```

3. 在右侧找到 `AllowInsecureGuestAuth`，将其值改为 `1`
   
   如果不存在，右键新建 → DWORD (32位) 值，命名为 `AllowInsecureGuestAuth`，值设为 `1`

4. 建议同时检查：
   ```
   HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Windows\LanmanWorkstation
   ```
   同样将 `AllowInsecureGuestAuth` 设为 `1`

关闭注册表编辑器，重新访问共享文件夹。

## Windows 家庭版用户

家庭版默认没有组策略编辑器，需要先启用。可以创建一个 `.bat` 文件，内容如下：

```batch
@echo off
pushd "%~dp0"
dir /b %SystemRoot%\servicing\Packages\Microsoft-Windows-GroupPolicy-ClientExtensions-Package~3*.mum >List.txt
dir /b %SystemRoot%\servicing\Packages\Microsoft-Windows-GroupPolicy-ClientTools-Package~3*.mum >>List.txt
for /f %%i in ('findstr /i . List.txt 2^>nul') do dism /online /norestart /add-package:"%SystemRoot%\servicing\Packages\%%i"
pause
```

右键以管理员身份运行，等待完成后重启电脑，就可以使用 `gpedit.msc` 了。

## 安全提示

启用"不安全的来宾登录"会降低系统安全性，建议只在可信的内部网络环境中使用。如果共享文件夹支持身份验证，建议配置正确的账号密码进行访问。

## 参考链接

- [Windows 10/11解决"无法访问共享文件夹—组织安全策略阻止未经身份验证的来宾访问"](https://juejin.cn/post/7579629985062420506)
- [访问共享文件夹提示解决方案](https://blog.csdn.net/qq_36953067/article/details/136443544)