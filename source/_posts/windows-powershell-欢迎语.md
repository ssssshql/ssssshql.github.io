---
title: Windows PowerShell 添加欢迎语
date: 2026-03-26 20:00:00
tags:
  - PowerShell
  - Windows
  - 终端美化
---

## 效果

每次打开 PowerShell，显示一个炫酷的欢迎界面，包含用户信息、IP、MAC 地址和随机编程名言。

 ![image-20260326103019180](https://picgo.19991029.xyz/image-20260326103019180.png)

## 位置

```
C:\Users\autotoll\Documents\WindowsPowerShell\Microsoft.PowerShell_profile.ps1
```

## 代码

```powershell
function Show-WelcomeBanner {
    $net = Get-CimInstance Win32_NetworkAdapterConfiguration | Where-Object { $_.IPEnabled -eq $true } | Select-Object -First 1
    $ip = $net.IPAddress[0]
    $mac = $net.MACAddress
    $hostname = $env:COMPUTERNAME
    $username = $env:USERNAME
    $date = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    
    $quotes = @(
        "Talk is cheap. Show me the code. February Linus Torvalds"
        "Stay hungry, stay foolish. February Steve Jobs"
        "Done is better than perfect. February Facebook"
        "First solve the problem. Then write the code. February John Johnson"
        "Simplicity is the soul of efficiency. February Austin Freeman"
        "Make it work, make it right, make it fast. February Kent Beck"
        "Code is like humor. When you have to explain it, it is bad. February Cory House"
    )
    $quote = $quotes | Get-Random

    Write-Host ''
    Write-Host '    ______           __           ______        ___    ___' -ForegroundColor Cyan
    Write-Host '   /\  _  \         /\ \__       /\__  _\      /\_ \  /\_ \' -ForegroundColor Cyan
    Write-Host '   \ \ \L\ \  __  __\ \ ,_\   ___\/_/\ \/   ___\//\ \ \//\ \' -ForegroundColor Cyan
    Write-Host '    \ \  __ \/\ \/\ \\ \ \/  / __`\ \ \ \  / __`\\ \ \  \ \ \' -ForegroundColor Cyan
    Write-Host '     \ \ \/\ \ \ \_\ \\ \ \_/\ \L\ \ \ \ \/\ \L\ \\_\ \_ \_\ \_' -ForegroundColor Cyan
    Write-Host '      \ \_\ \_\ \____/ \ \__\ \____/  \ \_\ \____//\____\/\____\' -ForegroundColor Cyan
    Write-Host '       \/_/\/_/\/___/   \/__/\/___/    \/_/\/___/ \/____/\/____/' -ForegroundColor Cyan
    Write-Host ''
    Write-Host '  +---------------------------------------------------------------+' -ForegroundColor DarkGray
    $line1 = "  |  User: $username@$hostname"
    Write-Host $line1.PadRight(62) -NoNewline -ForegroundColor Yellow
    Write-Host '|' -ForegroundColor DarkGray
    $line2 = "  |  IP:   $ip"
    Write-Host $line2.PadRight(62) -NoNewline -ForegroundColor Green
    Write-Host '|' -ForegroundColor DarkGray
    $line3 = "  |  MAC:  $mac"
    Write-Host $line3.PadRight(62) -NoNewline -ForegroundColor Green
    Write-Host '|' -ForegroundColor DarkGray
    $line4 = "  |  Time: $date"
    Write-Host $line4.PadRight(62) -NoNewline -ForegroundColor White
    Write-Host '|' -ForegroundColor DarkGray
    Write-Host '  +---------------------------------------------------------------+' -ForegroundColor DarkGray
    Write-Host ''
    Write-Host "  $quote" -ForegroundColor Magenta
    Write-Host ''
}

Show-WelcomeBanner

# OPENSPEC:START - OpenSpec completion (managed blocks, do not edit manually)
. "C:\Users\autotoll\Documents\PowerShell\OpenSpecCompletion.ps1"
# OPENSPEC:END
```

## 说明

- PowerShell 启动时会自动执行 profile 文件
- ASCII art 需要用等宽字体才能正常显示
- 每次打开终端会随机显示一句编程名言
