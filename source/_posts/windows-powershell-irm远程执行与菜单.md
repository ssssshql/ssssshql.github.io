---
title: Windows PowerShell irm 远程执行与菜单
date: 2026-03-26 21:00:00
tags:
  - PowerShell
  - Windows
  - 脚本
---

## irm 远程执行命令

`irm` (Invoke-RestMethod) 可以从远程服务器下载并执行脚本：

```powershell
irm 192.168.10.4:8080/1.txt | iex
```

- `irm` 下载远程文件内容
- `iex` (Invoke-Expression) 执行下载的代码

适合快速分发脚本，无需本地保存文件。

## 简单选择菜单

```powershell
# 清屏 + 显示菜单
cls
Write-Host "==========================" -ForegroundColor Cyan
Write-Host "      请选择操作"         -ForegroundColor White
Write-Host "==========================" -ForegroundColor Cyan
Write-Host " 1 - 执行功能一"
Write-Host " 2 - 执行功能二"
Write-Host "==========================" -ForegroundColor Cyan

# 获取用户输入
$choice = Read-Host "请输入数字 1 或 2"

# 判断选择
switch ($choice) {
    "1" {
        Write-Host "`n你选择了：功能一" -ForegroundColor Green
        # 在这里写你要执行的代码
        # 示例：notepad.exe
    }
    "2" {
        Write-Host "`n你选择了：功能二" -ForegroundColor Green
        # 在这里写你要执行的代码
        # 示例：mstsc.exe
    }
    default {
        Write-Host "`n输入错误！请只能输入 1 或 2" -ForegroundColor Red
    }
}

# 暂停窗口
Read-Host "`n按回车退出"
```

## 说明

- 结合 irm 可以实现远程菜单脚本分发
- 修改 switch 里的代码实现不同功能
- 适合自动化运维场景

![image-20260326104818559](https://picgo.19991029.xyz/image-20260326104818559.png)
