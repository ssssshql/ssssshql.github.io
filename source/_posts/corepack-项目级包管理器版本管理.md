---
title: corepack - 项目级包管理器版本管理
date: 2026-03-25 20:00:00
tags:
  - Node.js
  - pnpm
  - 工具
---

## 简介

Corepack 是 Node.js 官方提供的包管理器管理工具，可以固定项目的包管理器版本，避免团队成员使用不同版本导致的问题。

## 使用步骤

### 1. 启用 corepack

```bash
corepack enable
```

### 2. 固定项目包管理器版本

在项目根目录执行：

```bash
corepack use pnpm@10.29.2
```

这会在 `package.json` 中添加 `packageManager` 字段，锁定版本。

### 3. 自动识别

配置完成后，每次进入项目目录时，corepack 会自动识别并切换到指定的 pnpm 版本，无需手动操作。