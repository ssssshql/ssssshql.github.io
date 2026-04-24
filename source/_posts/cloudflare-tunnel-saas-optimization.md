---
title: Cloudflare Tunnel + SaaS 接入优选 IP 实操指南
date: 2026-04-24 12:20:00
tags:
  - Cloudflare
  - Tunnel
  - SaaS
  - 优选IP
  - 19991029.xyz
categories:
  - 技术笔记
---

在通过 Cloudflare Tunnel 进行内网穿透时，默认的 Anycast 网络在国内的访问延迟往往较高。通过 Cloudflare for SaaS 的“自定义主机名”功能配合优选 IP 地址，可以有效优化访问路径。

本文记录使用中转域名 `0112306.xyz` 与服务域名 `19991029.xyz` 实现 Tunnel 加速的具体步骤。

---

### 1. 配置优选入口记录

首先，在中转域名 `0112306.xyz` 的 DNS 设置中建立一个用于优选 IP 跳转的入口。

*   **添加记录**：新建子域名 `js.0112306.xyz`。
*   **配置方式**：类型选择 `CNAME`，目标指向优选域名 `visa.cn`。
*   **代理状态**：**保持为“仅 DNS”（灰云）**，不开启 Cloudflare 代理。

这一步的作用是让该子域名直接解析到优选节点 IP。

![](https://picgo.19991029.xyz/image-20260424152509283.png)

---

### 2. 配置 Cloudflare Tunnel 映射

在 Cloudflare Zero Trust 的 Tunnel 配置中，需要为同一个内网服务添加两个不同的 Public Hostname。假设内网服务监听端口为 `38448`：

1.  **回退源映射**：
    *   **Hostname**: `origin.0112306.xyz`
    *   **Service**: `http://localhost`
2.  **服务域名映射**：
    *   **Hostname**: `nas.19991029.xyz`
    *   **Service**: `http://localhost:38448`（此处必须包含完整的服务端口）。

![](https://picgo.19991029.xyz/image-20260424152705344.png)

---

### 3. 修改服务域名解析

接下来，需要修改服务域名 `19991029.xyz` 的 DNS 记录，使其通过优选入口接入。

1.  **清理旧记录**：删除原有的 `nas`（或泛域名 `*`）直接指向 Tunnel 的 DNS 记录。
2.  **新建 CNAME**：将 `nas`（或泛域名 `*`）CNAME 指向第一步创建的 **`js.0112306.xyz`**。
3.  **代理状态**：**必须开启 Cloudflare 代理（小黄云）**。

通过这种配置，流量会先解析到优选 IP，然后由 Cloudflare 节点接管。

![](https://picgo.19991029.xyz/image-20260424152747794.png)

---

### 4. 设置 SaaS 自定义主机名与回退源

最后，利用 SaaS 的回退逻辑将流量导入隧道。

在 `0112306.xyz` 域名的 **SSL/TLS -> 自定义主机名 (Custom Hostnames)** 页面进行如下操作：

1.  **设置回退源 (Fallback Origin)**：填入 `origin.0112306.xyz`。
2.  **添加自定义主机名**：添加 `nas.19991029.xyz`（如使用泛域名，可填入 `*.19991029.xyz`）。
3.  **验证所有权**：根据提示在 `19991029.xyz` 侧添加对应的 TXT 记录，直到证书和主机名状态变为 **Active**。

![](https://picgo.19991029.xyz/image-20260424152841755.png)

---

### 5. 路由逻辑与观测总结

**流量流向分析：**
1. 访问者请求 `nas.19991029.xyz`，DNS 解析至 `js.0112306.xyz` 对应的 **优选 IP**。
2. 请求进入 Cloudflare 边缘节点，节点识别到该域名属于 SaaS 自定义主机名。
3. 触发回退机制，请求被转交给回退源 `origin.0112306.xyz`。
4. 流量最终通过 Tunnel 到达内网服务器的指定端口。

经实测，优化后 Tunnel 的平均访问延迟均能稳定在 **150ms** 左右。

![image-20260424153853562](https://picgo.19991029.xyz/image-20260424153853562.png)

---
*记录于 2026-04-24，方案验证有效。*
