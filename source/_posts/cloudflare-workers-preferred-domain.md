---
title: Cloudflare Workers 优选域名实操指南
date: 2026-04-24 16:00:00
tags:
  - Cloudflare
  - Workers
  - 优选IP
categories:
  - 技术笔记
---

在使用 Cloudflare Workers 部署服务时，默认的 `workers.dev` 域名在国内访问往往不太理想。通过为 Workers 项目配置自定义域名并配合优选 IP，可以显著提升访问速度。

本文是 [《Cloudflare Tunnel + SaaS 接入优选 IP 实操指南》](https://blog.19991029.xyz/2026/04/24/cloudflare-tunnel-saas-optimization/) 的进阶篇，利用已经配置好的优选基础设施来加速 Workers。

---

### 前置条件

在继续之前，确保你已经按照前文完成了中转域名 `0112306.xyz` 的优选入口配置：
*   已创建 `js.0112306.xyz` 并 CNAME 指向优选域名 `visa.cn`（灰云）。
*   服务域名 `19991029.xyz` 的相关记录（如泛域名 `*`）已 CNAME 指向 `js.0112306.xyz`（开启小黄云）。

这意味着，任何指向 `19991029.xyz` 的请求实际上都已经通过优选 IP 接入了 Cloudflare 网络。

---

### 核心步骤：给 Workers 项目添加路由

由于我们已经在上一篇文章中完成了 DNS 侧的优选接入，现在只需要告诉 Cloudflare：当某个特定的子域名请求到达边缘节点时，由哪个 Worker 来处理。

这里以 `bit.19991029.xyz` 为例：

1.  进入域名 `19991029.xyz` 的管理界面。
2.  点击左侧菜单的 **Workers 路由 (Workers Routes)** -> **添加路由 (Add route)**。

![在 Cloudflare 后台找到 Workers 路由添加按钮](https://picgo.19991029.xyz/image-20260424165658014.png)

3.  **路由 (Route)** 填入：`bit.19991029.xyz/*`
4.  **Worker**：选择你想要触发的 Worker 项目。
5.  点击 **保存**。

![填写 Workers 路由详情并保存](https://picgo.19991029.xyz/image-20260424165722493.png)

> **特别注意**：路由末尾**必须**加上 `/*`（即 `bit.19991029.xyz/*`），这样才能确保该域名下的所有子路径都能正确触发 Worker。

---
### 为什么这样做？

在这种配置下，你**不需要**在 Workers 的“自定义域”中添加域名，只需要使用“Workers 路由”。

**流量流向示意图：**

```text
[ 用户访问 ] bit.19991029.xyz
      |
      | (DNS 解析: CNAME -> js.0112306.xyz -> visa.cn)
      v
[ 优选 IP 节点 ] (Cloudflare Edge)
      |
      | (识别域名并匹配 Workers 路由)
      v
[ bit.19991029.xyz/* ]
      |
      | (触发执行)
      v
[ Worker 程序 ]
```

这种方案直接复用了 SaaS 优选的链路，无需为每个 Worker 单独配置复杂的 DNS 记录，非常高效。
---
*记录于 2026-04-24，基于 SaaS 优选架构优化。*
