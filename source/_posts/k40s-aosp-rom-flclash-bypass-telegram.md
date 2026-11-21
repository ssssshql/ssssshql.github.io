---
title: Redmi K40S 刷类原生 ROM 后 Telegram 一直连接中：VPN 绕过了 tun0
date: 2026-11-21 22:30:00
tags: [Redmi K40S, 类原生 ROM, FlClash, Clash Meta, Telegram, VPN, 踩坑]
categories: 折腾
description: Redmi K40S 刷了 Android 16 类原生 ROM 后，浏览器能开 Google，Telegram 却一直连接中，Muse 登录报无网络。节点和 tun0 都正常，最后发现是代理软件默认的 bypassable 让原生 TCP 流量绕过了 tun0，直接从 wlan0 出去被墙了
---

给 K40S 刷了个 Android 16 的类原生 ROM，结果 Telegram 和 Muse 全断网了，浏览器却好好的。排查了一圈，最后发现问题不在节点，也不在网络，在 VPN 的一个默认属性上。记录一下，换机或者重刷类原生 ROM 的时候能直接照着查。

## 环境

- 设备：Redmi K40S（22021211RC），Android 16 类原生 ROM（BP4A.251205.006）
- 客户端：Clash Meta、FlClash、v2rayNG（同一网络、同一节点）
- 现象：
  - 浏览器能打开 Google
  - Telegram 一直卡在“连接中”，Muse 登录提示无网络
  - 另一台手机完全正常
  - 手机连的 Wi-Fi 显示“已连接 / 网络受限”

## 排查过程

### 1. 先确认节点和隧道没问题

第一反应是节点挂了或者 VPN 隧道没起来。直接 `adb shell` 进去用 curl 测：

```bash
curl https://www.google.com
curl https://api.telegram.org
curl https://149.154.167.50
```

三个全部返回正常（exit 0）。系统时间正常，私人 DNS 是 off。结论：**节点、网络、tun0 虚拟网卡本身都是通的**。

### 2. 排除系统限制开关

查到系统全局存在一个 `restricted_networking_mode=1`，看着很可疑。通过 adb 把它改成 0：

```bash
adb shell settings put global restricted_networking_mode 0
```

结果 TG 依然连不上。这个开关不是主因，排除。

### 3. 核心矛盾：浏览器能上，原生 TCP App 不通

这是最关键的一步。浏览器能开 Google，TG 和 Muse 却不行，说明问题不在“能不能上网”，而在“流量走了哪条路”。

在 `dumpsys connectivity` 里发现，FlClash 注册 VPN 的时候，同时附带了一个系统代理：

```
HttpProxy: [127.0.0.1] 7890
```

浏览器走的是系统 HttpProxy，所以能打开 Google；而 TG、Muse 这些用原生 TCP 的 App 不走 HttpProxy，必须走 tun0。

再去 `logcat` 里抓，看到关键报错：App 解析出了 FlClash 的地址 `198.18.0.10`，但发起 TCP 连接的时候，源地址却是 wlan0 的 `192.168.2.201`，直接超时。

也就是说：**原生 TCP 流量根本没进 tun0，直接从 wlan0 发出去了，在墙内被阻断或超时。**

实锤原因：在这批高版本 AOSP 类原生 ROM 的网络栈下，代理软件默认带有 `bypassable=true`（允许绕过）属性，导致普通 App 的原生 TCP 流量绕过了 tun0，直接从 wlan0 发出。

### 两次验证

为了确认这个判断，做了两次验证，都对上了。

**验证 1：TG 内置代理手动填 `127.0.0.1:7890`（SOCKS5）→ 秒连。**

绕过 tun0 的问题不存在了，因为流量直接走了 FlClash 的本地 SOCKS5 端口。说明节点本身没问题，问题确实在路由。

**验证 2：系统 VPN 开启「阻止未使用 VPN 的连接」强锁路由 → TG 立刻能收消息。**

把系统 VPN 的“始终开启 + 阻止未使用 VPN 的连接”打开，强制所有流量走 VPN，TG 马上就通了。进一步确认是路由绕过的问题。

## 最终解决（免 Root）

### 解决 TG / Muse 无法联网

在 FlClash 设置里找到「允许绕过 / Allow Bypass」开关，把它关掉，然后重连。App 流量就能正常进入 tun0 了。

这样就不用开系统的「阻止未使用 VPN 的连接」了，日常关掉 VPN 也不会全机断网，算是个免 Root 的完美方案。

### 解决 Wi-Fi“网络受限”

这是另一个独立的小问题。类原生 ROM 默认请求 Google 的 `connectivitycheck.gstatic.com` 来检测网络，国内被墙，wlan0 就被标为 `PARTIAL_CONNECTIVITY`，Wi-Fi 显示“网络受限”。

通过 adb 把检测地址换成国内可用的 204 接口：

```bash
adb shell settings put global captive_portal_http_url http://connect.rom.miui.com/generate_204
adb shell settings put global captive_portal_https_url https://connect.rom.miui.com/generate_204
```

开关一次 Wi-Fi，状态就恢复正常的“已连接”了。

## 总结

1. **浏览器能上 ≠ 网络没问题。** 浏览器走系统代理，原生 TCP App 走 tun0，两条路不一样。TG、Muse 这种原生 TCP 的 App 断网，先看流量到底走了哪条路。
2. **先看 `dumpsys connectivity` 和 `logcat`。** 一个看到 VPN 注册时带了什么代理，一个看到 TCP 连接的源地址到底是 tun0 还是 wlan0，两个命令就能定位到路由层面。
3. **类原生 ROM 上先查 `bypassable`。** 高版本 AOSP 的网络栈下，代理软件默认 `bypassable=true` 会让原生 TCP 绕过 tun0。FlClash 里把「允许绕过」关掉就能解决，不用 Root，也不用开系统的 VPN 强锁。
4. **Wi-Fi“网络受限”先看 captive portal 检测地址。** 类原生默认测 Google 的地址，国内被墙就显示受限，换成国内 204 接口就行。
