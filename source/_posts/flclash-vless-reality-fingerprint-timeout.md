---
title: FlClash 的 VLESS Reality 一直超时：最后发现是一行 fingerprint 惹的祸
date: 2026-11-08 01:30:00
tags: [FlClash, Mihomo, Xray, VLESS, Reality, 踩坑]
categories: 折腾
description: FlClash 连不上自建的 VLESS Reality 节点，v2ray 却能连。换 SNI、加 MLKEM、改 minClientVer、降级 Xray 全试了一圈都没用，最后在 FlClash 日志里找到真凶：订阅里多写了一行 fingerprint，新版 Mihomo 把它当成证书固定来校验了
---

昨晚折腾了一晚上，记录一下。自建的 VLESS Reality 节点突然在 FlClash 里连不上了，v2ray 客户端却好好的。排查过程走了不少弯路，最后的真凶出乎意料地简单。

## 环境

- 服务器：Oracle Cloud AMD 实例，Xray 26.10.31
- 节点：VLESS + Reality（2096，TCP，xtls-rprx-vision）、Trojan（443）、Hysteria2（UDP 443）、Shadowsocks（8388）
- 客户端：FlClash（安卓，Mihomo 内核）、v2ray 客户端
- 现象：**只有 Reality 在 FlClash 里超时**，其他三个协议都正常；同一条 Reality 节点，v2ray 客户端能连

## 排查过程：走过的弯路

### 1. 先怀疑被墙，换 SNI

第一反应是 IP 或者 SNI 被墙了。把 Reality 的伪装站从 `www.bing.com` 换成少人用的 `www.samsung.com`，重启 Xray，openssl 握手验证正常，返回 Samsung 的证书。

顺便发现一个坑：**我的订阅不是从 Xray 配置自动生成的**，而是服务器上一个小服务直接发两个静态文件（`v2ray.txt` 和 `sub.yaml`）。只改 Xray 配置不改订阅文件，客户端更新订阅拉下来的还是旧 SNI。所以两个地方都要改。

结果：还是超时。

### 2. Xray 26.9.8+ 的 MLKEM 要求

查资料发现，从 Xray 26.9.8 开始，Reality 要求客户端的 TLS 握手里必须带 `X25519MLKEM768`，否则服务端静默回落到伪装站，客户端报 `reality verification failed`。Mihomo / sing-box 内核的客户端默认不发这个。

订阅里改了两处：指纹 `firefox` 改成 `chrome`，加上 `support-x25519mlkem768: true`。

结果：还是超时。

### 3. minClientVer

Mihomo 在 Reality 握手里把客户端版本硬编码成 `1.8.2`，而 Xray 26.7.11+ 默认要求 `minClientVer` 是 `26.3.27`，Mihomo 会被静默拒绝（Mihomo #3132）。于是在服务端 Reality 配置里加了 `"minClientVer": "1.8.2"`。

结果：还是超时。

### 4. 怀疑 flow，拿掉又加回去

FlClash #2464 说 0.8.98 内置的 Mihomo 1.10.0 不认 `xtls-rprx-vision`，删掉 `flow` 就能连。于是把订阅里的 `flow` 拿掉试了一次。

结果：还是超时，只好又加回去。

### 5. 降级 Xray，又升回来

把服务端 Xray 从 26.10.31 降到 26.7.28（MLKEM 要求出来之前的最后一个版本），想着老内核的 FlClash 总该能连了。

结果：还是超时。降级白降了，又升回 26.10.31。

这时候服务器这边能试的都试完了。更奇怪的是：**服务器日志里没有任何 Reality 相关的报错**，FlClash 的尝试根本没到握手那一步。

## 真凶：FlClash 日志里的一行

最后让我去看 FlClash 的日志，里面清清楚楚写着：

```
129.153.108.11:2096 connect error: `fingerprint` is used for TLS certificate pinning.
If you need to specify the browser fingerprint, use `client-fingerprint`
```

意思是：**在新版 Mihomo 里，`fingerprint` 这个字段是用来做 TLS 证书固定的，要填证书的哈希值；浏览器指纹要用 `client-fingerprint`。**

再去看订阅文件，Phoenix-Reality 那条同时写了两行：

```yaml
fingerprint: chrome
client-fingerprint: chrome
```

`fingerprint: chrome` 在新版 Mihomo 眼里就是个非法的证书哈希，连接在客户端内部直接报错，握手都没发出去。这就解释了一切：服务端零日志、界面只显示 timeout、换 SNI 和降级 Xray 全都没用。

以前老版本的 Clash / Mihomo 里，`fingerprint` 就是浏览器指纹的意思，写 chrome、firefox 都对，订阅文件当初这么写没错。后来新版 Mihomo 把这个字段改了用途，浏览器指纹挪到 `client-fingerprint` 去了，旧写法就失效了。

## 修复

订阅文件里删掉那行 `fingerprint: chrome`，只留 `client-fingerprint: chrome`，更新订阅，重试——**通了**。

```yaml
# 改之前
fingerprint: chrome
client-fingerprint: chrome

# 改之后
client-fingerprint: chrome
```

v2ray 那边的链接不用动，`fp=chrome` 本来就是 Xray 的正确写法。

## 总结

1. **先看客户端日志，再动服务端。** 这次最大的教训：服务端日志干干净净，不代表服务端没问题，更不代表问题在网络上。FlClash 日志里那句报错直接就是答案，我绕了一大圈才去看它。
2. **`fingerprint` ≠ `client-fingerprint`。** 新版 Mihomo 里前者是证书固定（填证书哈希），后者才是浏览器指纹。订阅模板是按老写法生成的，FlClash 一升级内核就炸了。
3. **MLKEM、minClientVer 那些不是白改的。** `fingerprint` 的错发生在连接之前，卡住了后面所有步骤。把它修掉之后，Xray 26.10.31 照样会要求 MLKEM 和客户端版本，没有那两项还是连不上。等于是一层一层排过去，最后才露出最底下这一层。
4. **界面上的 timeout 不一定是网络超时。** FlClash 把连接错误在测速里显示成 timeout，很容易把人往"被墙了"的方向引，实际上看一眼日志就能省掉一晚上。
