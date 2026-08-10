---
title: adb tcpip 5555 是干什么的？ADB 无线连接使用记录
date: 2026-08-05 13:59:23
tags: [ADB, Android, 无线调试]
categories: 工具
description: 记录 adb tcpip 5555 的实际作用、无线连接步骤、常见问题，以及它和 Android 11 无线调试的区别
---

最近需要在没有 USB 线的情况下调试安卓设备，用到了这条命令：

```bash
adb tcpip 5555
```

乍一看很像是在电脑上开放 `5555` 端口，其实不是。简单记录一下它到底做了什么，以及完整的无线连接流程。

## adb tcpip 5555 的作用

`adb tcpip 5555` 的作用是：

> **让安卓设备上的 `adbd` 守护进程重新启动，并监听 TCP 端口 `5555`。**

执行成功一般会看到：

```text
restarting in TCP mode port: 5555
```

这里需要注意几个容易混淆的点：

- 开放的是**安卓设备**上的 `5555` 端口，不是电脑端口
- ADB 服务端默认使用的 `5037` 端口与它不是一回事
- 这条命令只负责切换设备端 `adbd` 的监听方式
- 切换完成后，还要用 `adb connect 设备IP:5555` 建立连接

## 完整使用步骤

### 1. 先通过 USB 连接设备

手机打开开发者选项和 USB 调试，然后使用数据线连接电脑。

```bash
adb devices
```

手机第一次连接时会弹出 USB 调试授权，点击允许。输出中设备状态为 `device` 才算连接成功：

```text
List of devices attached
1234567890ABCDEF    device
```

![这里放 USB 调试授权截图]()

### 2. 切换到 TCP/IP 模式

```bash
adb tcpip 5555
```

如果同时连接了多台设备，需要指定序列号：

```bash
adb -s 1234567890ABCDEF tcpip 5555
```

### 3. 查看手机局域网 IP

在手机的 Wi-Fi 详情中查看 IP 地址，例如：

```text
192.168.1.100
```

电脑和手机需要连接到同一个局域网。

![这里放手机 Wi-Fi IP 地址截图]()

### 4. 建立无线连接

拔掉 USB 数据线，然后执行：

```bash
adb connect 192.168.1.100:5555
```

连接成功会看到：

```text
connected to 192.168.1.100:5555
```

最后检查设备：

```bash
adb devices
```

```text
List of devices attached
192.168.1.100:5555    device
```

现在就可以无线执行常用 ADB 命令了：

```bash
adb shell
adb install app.apk
adb logcat
adb pull /sdcard/test.txt
```

## 断开与切回 USB 模式

断开指定设备：

```bash
adb disconnect 192.168.1.100:5555
```

断开所有 TCP/IP 设备：

```bash
adb disconnect
```

重新插上数据线后，可以让 `adbd` 切回 USB 模式：

```bash
adb usb
```

输出一般为：

```text
restarting in USB mode
```

## 真正要注意的坑

### 5555 端口并不安全

传统的 `adb tcpip 5555` 方式会让设备在局域网中监听 ADB 连接。**不要在公司访客 Wi-Fi、酒店、商场等不可信网络中长时间开启。**

使用完成后建议执行：

```bash
adb disconnect
adb usb
```

如果暂时不再调试，也可以直接关闭手机的 USB 调试。

### IP 地址可能会变化

手机重新连接 Wi-Fi 后，路由器可能分配新的 IP。之前的连接失效时，重新查看手机 IP 再连接：

```bash
adb connect 新IP:5555
```

### 重启后可能失效

`adb tcpip 5555` 是否能在设备重启后继续生效，与安卓版本和厂商实现有关。失效后通常需要重新插入 USB 数据线，再执行一遍切换命令。

### 显示 offline 或连不上

可以依次尝试：

```bash
adb disconnect
adb kill-server
adb start-server
adb connect 192.168.1.100:5555
```

仍然无法连接时，再检查：

- 手机和电脑是否在同一个局域网
- 手机 IP 是否发生变化
- 路由器是否开启了 AP 隔离或客户端隔离
- Windows 防火墙是否阻止 ADB
- USB 调试授权是否被撤销

## Android 11 及以上的无线调试

Android 11 及以上系统提供了更安全的“无线调试”功能，可以通过配对码连接，不需要先插 USB 数据线。

手机进入：

```text
开发者选项 → 无线调试 → 使用配对码配对设备
```

然后在电脑执行：

```bash
adb pair 手机IP:配对端口
```

根据提示输入手机显示的配对码，配对成功后再连接：

```bash
adb connect 手机IP:连接端口
```

这里的**配对端口和连接端口可能不同**，以手机无线调试页面显示的内容为准，不要一律写成 `5555`。

两种方式简单对比：

| 方式 | 初次连接 | 安全性 | 适用场景 |
|---|---|---|---|
| `adb tcpip 5555` | 通常需要先连接 USB | 较低 | Android 10 及以下、旧设备 |
| `adb pair` 无线调试 | 配对码或二维码 | 更高 | Android 11 及以上 |

新设备优先使用系统自带的无线调试；`adb tcpip 5555` 更适合旧设备，或者临时测试时使用。

## 总结

`adb tcpip 5555` 本身不负责完成无线连接，它只是让安卓设备上的 `adbd` 开始监听 TCP `5555` 端口。完整流程是：

```text
USB 连接并授权
    ↓
adb tcpip 5555
    ↓
拔掉 USB
    ↓
adb connect 设备IP:5555
```

用完记得断开，并切回 USB 模式，避免设备长时间暴露 ADB 端口。

## 参考链接

- [Android Debug Bridge（adb）官方文档](https://developer.android.com/tools/adb)
