---
title: n8n 解锁高级版（Enterprise 功能）
date: 2026-07-02 13:00:00
tags: [n8n, Docker, 自动化, 破解]
categories: 工具
description: n8n 2.28.3 版本手动解锁 Enterprise 功能，通过修改 license.js 实现全功能开启
---

n8n 自部署版本默认只开启了基础功能，很多高级特性（如 Git 同步、RBAC、变量等）需要 License 才能解锁。本文记录手动修改 license.js 绕过验证的方法。

## 前置条件

- Docker + Docker Compose
- n8n 版本：**2.28.3**（其他版本需验证路径）
- 验证日期：**2026-07-02**

## Docker Compose 配置

```yaml
services:
  n8n:
    image: n8nio/n8n:2.28.3
    restart: always
    volumes:
      - ./data:/home/node/.n8n
    environment:
      - N8N_USER_FOLDER=/home/node/
      - GENERIC_TIMEZONE=Asia/Shanghai
      - TZ=Asia/Shanghai
      - DB_SQLITE_VACUUM_ON_STARTUP=true
      - DB_SQLITE_POOL_SIZE=1
      - EXECUTIONS_DATA_PRUNE=true
      - EXECUTIONS_DATA_MAX_AGE=168
      - EXECUTIONS_DATA_PRUNE_MAX_COUNT=5000
      - N8N_SECURE_COOKIE=true
      - WEBHOOK_URL=https://你的域名/
      - N8N_PROXY_HOPS=1
      - N8N_TRUSTED_PROXIES=0.0.0.0/0
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
      - N8N_GIT_NODE_DISABLE_BARE_REPOS=true
      - N8N_BLOCK_ENV_ACCESS_IN_NODE=false
      - N8N_RUNNERS_ENABLED=true
      - N8N_RUNNERS_MODE=external
      - N8N_RUNNERS_BROKER_LISTEN_ADDRESS=0.0.0.0
      - N8N_RUNNERS_AUTH_TOKEN=任意一个随机字符串
      - N8N_NATIVE_PYTHON_RUNNER=true
    user: root
    entrypoint:
      - /bin/sh
      - -c
      - |
        sed -i 's/isLicensed(feature) {/isLicensed(feature) { if (feature.endsWith("apiDisabled") || feature.endsWith("showNonProdBanner")) return false; return true; \/\/ /' /usr/local/lib/node_modules/n8n/dist/license.js && \
        sed -i "s/getValue(feature) {/getValue(feature) { if (feature === 'planName') return 'Enterprise'; return -1; \/\/ /" /usr/local/lib/node_modules/n8n/dist/license.js && \
        sed -i "s/getPlanName() {/getPlanName() { return 'Enterprise'; \/\/ /" /usr/local/lib/node_modules/n8n/dist/license.js && \
        exec /docker-entrypoint.sh

  task-runners:
    image: n8nio/runners:1.111.0
    environment:
      - N8N_RUNNERS_TASK_BROKER_URI=http://n8n:5679
      - N8N_RUNNERS_AUTH_TOKEN=任意一个随机字符串
    depends_on:
      - n8n
```

## 原理说明

n8n 在 `dist/license.js` 中定义了三个关键的验证方法：

| 方法 | 作用 | 修改后行为 |
|------|------|-----------|
| `isLicensed(feature)` | 检查某功能是否授权 | 除 `apiDisabled` 和 `showNonProdBanner` 外全部返回 `true` |
| `getValue(feature)` | 获取功能对应的值 | `planName` 返回 `"Enterprise"`，其余返回 `-1` |
| `getPlanName()` | 获取当前计划名称 | 直接返回 `"Enterprise"` |

通过容器 `entrypoint` 在 n8n 启动前执行 `sed` 命令，将这三个方法的返回值改写，从而实现全功能解锁。

## 使用说明

1. 将上面的 `docker-compose.yml` 保存到本地
2. 修改 `WEBHOOK_URL` 和 `N8N_RUNNERS_AUTH_TOKEN` 为你自己的值
3. 执行 `docker compose up -d` 启动

> **注意**：必须设置 `user: root`，否则 `sed` 无法修改 `license.js` 文件。

## 验证解锁成功

启动后进入 n8n Web UI，在 **Settings → Usage and Plan** 中可以看到计划显示为 **Enterprise**，所有高级功能（Variables、External Secrets、Tags RBAC、Workflow History 等）均可正常使用。

## 参考链接

- [n8n 官方镜像](https://hub.docker.com/r/n8nio/n8n)
- [原帖 Nodeseek](https://www.nodeseek.com/post-652639-1)
