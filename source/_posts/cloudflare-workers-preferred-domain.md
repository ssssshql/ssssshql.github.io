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

在使用 Cloudflare Workers 部署服务时，默认的 `workers.dev` 域名在国内访问往往不太理想。通过为 Workers 项目配置自定义域名并配合优选 IP（域名），可以显著提升访问速度和稳定性。

本教程以自定义域 `img.cmliussss.us.kg` 为例进行演示。

---

### 步骤 1：设置自定义域 CNAME 记录至优选域名

首先，需要将你的自定义子域名通过 CNAME 指向一个已知的优选域名（如 `visa.com`、`visa.cn` 等）。

1.  登录 Cloudflare，进入域名 `cmliussss.us.kg` 的 **DNS** 设置页面。
2.  添加一条新的解析记录：
    *   **类型**：`CNAME`
    *   **名称**：`img`（即你的自定义前缀）
    *   **目标**：`visa.com`（或者其他你信任的优选域名）
    *   **代理状态**：**关闭小黄云（保持为“仅 DNS”/灰云状态）！！！**

> **重要提示**：这一步千万不要开启 Cloudflare 的代理功能，否则优选 IP 将无法生效。

---

### 步骤 2：给 Workers 项目添加路由

由于 DNS 记录没有开启代理，我们需要在 Workers 的路由设置中手动绑定域名，让 Cloudflare 知道该域名的流量应由哪个 Worker 处理。

1.  在域名管理界面，点击左侧菜单的 **Workers 路由 (Workers Routes)** -> **添加路由 (Add route)**。
2.  **路由 (Route)** 填入：`img.cmliussss.us.kg/*`
3.  **Worker**：选择你对应的 Worker 项目名称。
4.  点击 **保存**。

> **特别注意**：自定义域名的末尾**必须**加上 `/*`，例如 `img.cmliussss.us.kg/*`，否则可能导致子路径无法正常访问。

---

### 总结

通过上述配置，流量的流向如下：
1. 访问者解析 `img.cmliussss.us.kg`，得到 `visa.com` 对应的 **优选 IP**。
2. 请求到达 Cloudflare 边缘节点后，节点根据 **Workers 路由** 配置，将流量转发给对应的 Worker 程序。

这种方案既利用了优选 IP 的链路优化，又保留了 Workers 的强大功能，是目前提升 Workers 访问体验的主流方案。

---
*记录于 2026-04-24。*
