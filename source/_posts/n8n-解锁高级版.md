---
title: n8n 2.28.x 解锁高级版（Enterprise 功能）
date: 2026-07-02 13:00:00
tags: [n8n, Docker, 自动化]
categories: 工具
description: n8n 2.28.3/2.28.4 版本通过挂载修改后的 license.js 解锁 Enterprise 功能，避免 user:root 导致的数据丢失问题
---

n8n 自部署版本默认只开启了基础功能，很多高级特性（如 Variables、External Secrets、RBAC、Workflow History 等）需要 License 才能解锁。本文记录通过挂载修改后的 `license.js` 绕过验证的方法。

## 前置条件

- Docker + Docker Compose
- n8n 版本：**2.28.3 / 2.28.4**
- 验证日期：**2026-07-02**

## 方案选择

网上常见的做法是在 `entrypoint` 中用 `sed` 修改 `license.js`，但这需要容器以 `root` 用户运行。实际测试发现加了 `user: root` 后重启会丢失所有数据（文件权限问题导致）。

更稳妥的做法：**提前准备好修改好的 `license.js`，以只读卷挂载进去**。

## 制作修改后的 license.js

先启动一个临时容器，把原始的 `license.js` 复制出来修改：

```bash
# 创建临时容器
docker run -d --name temp-n8n hotwa/n8n-chinese:2.28.4

# 复制 license.js 到当前目录
docker cp temp-n8n:/usr/local/lib/node_modules/n8n/dist/license.js ./license.js

# 停止并删除临时容器
docker rm -f temp-n8n
```

修改 `license.js` 中的三个关键方法：

找到 `isLicensed(feature) {` 这一行，将其改为：

```javascript
isLicensed(feature) { if (feature.endsWith("apiDisabled") || feature.endsWith("showNonProdBanner")) return false; return true; // }
```

找到 `getValue(feature) {` 这一行，将其改为：

```javascript
getValue(feature) { if (feature === 'planName') return 'Enterprise'; return -1; // }
```

找到 `getPlanName() {` 这一行，将其改为：

```javascript
getPlanName() { return 'Enterprise'; // }
```

也可以用 `sed` 一键修改：

```bash
sed -i 's/isLicensed(feature) {/isLicensed(feature) { if (feature.endsWith("apiDisabled") || feature.endsWith("showNonProdBanner")) return false; return true; \/\/ /' license.js
sed -i "s/getValue(feature) {/getValue(feature) { if (feature === 'planName') return 'Enterprise'; return -1; \/\/ /" license.js
sed -i "s/getPlanName() {/getPlanName() { return 'Enterprise'; \/\/ /" license.js
```

## Docker Compose 配置（仅主应用）

```yaml
services:
  n8n:
    container_name: n8n
    image: hotwa/n8n-chinese:2.28.4
    restart: always
    ports:
      - 5678:5678
    volumes:
      - ./data:/home/node/.n8n
      - ./license.js:/usr/local/lib/node_modules/n8n/dist/license.js:ro
    environment:
      - GENERIC_TIMEZONE=Asia/Shanghai
      - N8N_DEFAULT_LOCALE=zh-CN
      - N8N_OTEL_ENABLED=false
      - N8N_SECURE_COOKIE=false
      - NODE_FUNCTION_ALLOW_EXTERNAL=*
    networks:
      - 1panel-network
```

> **关键点**：
> - `license.js` 以 **`:ro`（只读）** 方式挂载，不需要 `user: root`
> - 重启容器**不会**丢失数据
> - 只部署了主应用，没有额外的 task-runners 等服务
> - `hotwa/n8n-chinese` 是 n8n 的中文汉化版镜像，与官方版本同步更新推送

## 原理说明

n8n 在 `dist/license.js` 中定义了三个关键的验证方法：

| 方法 | 作用 | 修改后行为 |
|------|------|-----------|
| `isLicensed(feature)` | 检查某功能是否授权 | 除 `apiDisabled` 和 `showNonProdBanner` 外全部返回 `true` |
| `getValue(feature)` | 获取功能对应的值 | `planName` 返回 `Enterprise`，其余返回 `-1` |
| `getPlanName()` | 获取当前计划名称 | 直接返回 `Enterprise` |

## 验证解锁成功

启动后进入 n8n Web UI，在 **Settings → Usage and Plan** 中可以看到计划显示为 **Enterprise**，所有高级功能均可正常使用。

## 参考链接

- [hotwa/n8n-chinese Docker 镜像](https://hub.docker.com/r/hotwa/n8n-chinese)
- [n8n 官方镜像](https://hub.docker.com/r/n8nio/n8n)
- [原帖 Nodeseek](https://www.nodeseek.com/post-652639-1)
