---
title: IDEA 2024 Docker 插件连接失败 - API 版本过低
date: 2026-04-14 15:17:00
tags: [IDEA, Docker, 故障排查]
---

今天用 IDEA 2024 连接 Docker 时突然报错：

> Docker integration - Status 400: client version 1.24 is too old. Minimum supported API version is 1.44, please upgrade your client to a newer version

查了一下，原因是 Docker Engine 更新到 v29.0+ 后，废弃了旧的 API 版本支持，而 IDEA 2024 的 Docker 插件还在使用旧的 API 1.24 版本。

## 解决方案

### 方案一：修改 Docker 配置（推荐）

强制 Docker 接受旧的 API 版本。

#### Windows Docker Desktop

1. 右键系统托盘的 Docker 图标，选择 **Settings**

2. 进入 **Docker Engine** 设置页

   ![image-20260414151857281](https://picgo.19991029.xyz/image-20260414151857281.png)

3. 在 JSON 配置中添加：
   ```json
   {
     "min-api-version": "1.24"
   }
   ```

   如果已有其他配置项，添加到同一个 JSON 对象中：
   ```json
   {
     "builder": {
       "gc": {
         "defaultKeepStorage": "20GB",
         "enabled": true
       }
     },
     "experimental": false,
     "min-api-version": "1.24"
   }
   ```

4. 点击 **Apply & Restart**，等待 Docker 重启完成

#### Linux

编辑 `/etc/docker/daemon.json`：

```bash
sudo vim /etc/docker/daemon.json
```

添加配置：
```json
{
  "min-api-version": "1.24"
}
```

重启 Docker 服务：
```bash
sudo systemctl restart docker
```

重新在 IDEA 中连接 Docker，应该就可以正常使用了。

### 方案二：升级 IDEA

安装最新版 IDEA（2025.2.5+），JetBrains 已经更新了 Docker 插件以支持新版 Docker API。

可以直接去官网下载，或者在 Toolbox 中更新。

### 方案三：降级 Docker Engine

如果无法修改配置且不想升级 IDEA，可以考虑降级 Docker 到 v28.x 版本，但不太建议这样做。

## 补充说明

这个问题的根本原因是 Docker v29.0+ 提高了最低 API 版本要求（从 1.24 提升到 1.44），导致使用旧版 API 的客户端工具无法连接。

除了 IDEA，其他工具如 Traefik、CI/CD 流水线中的 Docker 镜像也可能遇到类似问题，解决思路都是：
- 让服务端兼容旧版本（设置 min-api-version）
- 或者升级客户端工具

## 参考链接

- [JetBrains Docker Plugin Reviews](https://plugins.jetbrains.com/plugin/7724-docker/reviews)
