---
title: vfox使用指南
date: 2026-03-25 13:30:00
tags: [vfox, 版本管理]
categories: [开发工具]
---

## 什么是 vfox

vfox 是一个跨平台、可扩展的版本管理器，可以轻松管理各种开发工具和运行时环境（如 Node.js、Python、Java 等）。

## 核心特性

- ✅ **跨平台支持**：支持 Windows（非WSL）、Linux、macOS
- ✅ **插件系统**：提供简单 API，可轻松添加新工具支持
- ✅ **多 Shell 支持**：支持 PowerShell、Bash、ZSH、Fish、Clink、Nushell
- ✅ **向后兼容**：支持从现有配置文件迁移（`.tool-versions`、`.nvmrc`、`.node-version` 等）
- ✅ **统一配置**：使用单个配置文件 `.vfox.toml` 管理所有工具版本

## 安装 vfox

### Windows 系统

```powershell
# 使用 Scoop
scoop install vfox

# 使用 Winget
winget install vfox

# 手动安装
# 从 https://github.com/version-fox/vfox/releases 下载安装程序
```

### macOS 系统

```bash
# 使用 Homebrew
brew install vfox
```

### Linux 系统

**Debian/Ubuntu（APT）：**
```bash
echo "deb [trusted=yes lang=none] https://apt.fury.io/versionfox/ /" | sudo tee /etc/apt/sources.list.d/versionfox.list
sudo apt-get update
sudo apt-get install vfox
```

**CentOS/Fedora（YUM）：**
```bash
echo '[vfox]
name=VersionFox Repo
baseurl=https://yum.fury.io/versionfox/
enabled=1
gpgcheck=0' | sudo tee /etc/yum.repos.d/vfox.repo
sudo yum install vfox
```

**安装脚本（推荐）：**
```bash
# 安装到系统目录
curl -sSL https://raw.githubusercontent.com/version-fox/vfox/main/install.sh | bash

# 安装到用户目录（无需 sudo）
curl -sSL https://raw.githubusercontent.com/version-fox/vfox/main/install.sh | bash -s -- --user
```

## 配置 Shell 自动加载

安装后需要配置 Shell 才能正常使用。

### Bash

```bash
echo 'eval "$(vfox activate bash)"' >> ~/.bashrc
source ~/.bashrc
```

### Zsh

```bash
echo 'eval "$(vfox activate zsh)"' >> ~/.zshrc
source ~/.zshrc
```

### Fish

```bash
echo 'vfox activate fish | source' >> ~/.config/fish/config.fish
```

### PowerShell

```powershell
# 创建或更新配置文件
if (-not (Test-Path -Path $PROFILE)) { New-Item -Type File -Path $PROFILE -Force }
Add-Content -Path $PROFILE -Value 'Invoke-Expression "$(vfox activate pwsh)"'

# 如果遇到执行策略限制，以管理员身份运行：
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned
```

### Clink & Cmder

1. 查找脚本目录：
   ```cmd
   clink info | findstr scripts
   ```

2. 下载 [clink_vfox.lua](https://github.com/version-fox/vfox/blob/main/internal/shell/clink_vfox.lua) 到脚本目录

3. 重启 Clink 或 Cmder

### Nushell

```bash
vfox activate nushell $nu.default-config-dir | save --append $nu.config-path
```

## 基本使用

### 1. 查看可用插件

```bash
# 查看所有可用插件
vfox available

# 常用插件示例
# nodejs    - Node.js
# python    - Python
# java      - Java JDK
# golang    - Go
# dotnet    - .NET SDK
# rust      - Rust
# maven     - Maven
# gradle    - Gradle
```

### 2. 添加插件

```bash
# 添加 Node.js 插件
vfox add nodejs

# 添加 Python 插件
vfox add python

# 添加 Java 插件
vfox add java
```

### 3. 查看可用版本

```bash
# 查看所有可用版本
vfox search nodejs

# 查看特定版本的详细信息
vfox search nodejs@21.5.0
```

### 4. 安装运行时版本

```bash
# 安装指定版本
vfox install nodejs@21.5.0

# 安装最新版本
vfox install nodejs@latest

# 安装多个版本
vfox install nodejs@20.9.0
vfox install nodejs@18.0.0
```

### 5. 切换版本

vfox 支持三种作用域，优先级从高到低：

**项目级（Project）：**
```bash
# 仅当前项目生效，配置写入 .vfox.toml
vfox use -p nodejs@20.9.0
```

**会话级（Session）：**
```bash
# 当前 Shell 会话有效，关闭后自动清除
vfox use -s nodejs@18.0.0
```

**全局级（Global）：**
```bash
# 用户全局生效，所有项目可用
vfox use -g nodejs@21.5.0
```

### 作用域说明

| 作用域  | 命令              | 配置路径                   | 说明                                       |
|---------|-------------------|----------------------------|--------------------------------------------|
| Project | `vfox use -p`     | `$PWD/.vfox/sdks`          | 仅当前项目生效，适合团队协作               |
| Session | `vfox use -s`     | `~/.vfox/tmp/<pid>/sdks`   | 当前会话有效，关闭 Shell 后自动清理        |
| Global  | `vfox use -g`     | `~/.vfox/sdks`             | 全局生效，所有项目可用                     |

> 💡 **默认行为**：不指定作用域时，默认为 Session 级别

### 6. 查看当前版本

```bash
# 查看所有已安装的 SDK 及当前版本
vfox list

# 查看特定插件的版本
vfox list nodejs
```

### 7. 卸载版本

```bash
# 卸载指定版本
vfox uninstall nodejs@18.0.0

# 卸载所有版本
vfox uninstall nodejs --all
```

## 常用命令速查

| 命令                        | 说明                           |
|-----------------------------|--------------------------------|
| `vfox available`            | 查看所有可用插件               |
| `vfox add <plugin>`         | 添加插件                       |
| `vfox search <plugin>`      | 查看可用版本                   |
| `vfox install <sdk>@<ver>`  | 安装指定版本                   |
| `vfox use -p <sdk>@<ver>`   | 切换版本（项目级）             |
| `vfox use -s <sdk>@<ver>`   | 切换版本（会话级）             |
| `vfox use -g <sdk>@<ver>`   | 切换版本（全局级）             |
| `vfox list`                 | 查看已安装的 SDK               |
| `vfox uninstall <sdk>@<ver>`| 卸载指定版本                   |
| `vfox --help`               | 查看帮助                       |
| `vfox --version`            | 查看版本                       |

## 项目配置文件

### .vfox.toml

在项目根目录创建 `.vfox.toml` 文件，可以固定项目的工具版本：

```toml
[env]
NODEJS_VERSION = "20.9.0"
PYTHON_VERSION = "3.11.0"
```

使用 `vfox use -p` 命令会自动创建此文件。

### .gitignore 配置

```gitignore
# vfox 配置文件（建议提交）
# .vfox.toml

# vfox 工作目录（不要提交）
.vfox/
```

## 向后兼容

vfox 支持从现有工具的配置文件迁移：

- `.tool-versions`（asdf）
- `.nvmrc`（nvm）
- `.node-version`（nodenv）
- `.sdkmanrc`（SDKMAN）

直接在项目目录运行 `vfox use` 即可自动识别并迁移。

## 实用示例

### 场景一：新项目初始化

```bash
# 进入项目目录
cd my-project

# 添加需要的插件
vfox add nodejs
vfox add python

# 安装指定版本
vfox install nodejs@20.9.0
vfox install python@3.11.0

# 设置项目版本
vfox use -p nodejs@20.9.0
vfox use -p python@3.11.0

# 提交 .vfox.toml 到代码库
git add .vfox.toml
```

### 场景二：团队协作

```bash
# 克隆项目
git clone <repo>
cd <repo>

# 自动安装项目配置的版本
vfox install

# 版本已自动切换，无需手动操作
node --version
```

### 场景三：测试不同版本

```bash
# 全局安装稳定版本
vfox install nodejs@20.9.0
vfox use -g nodejs@20.9.0

# 在会话中测试新版本
vfox install nodejs@21.5.0
vfox use -s nodejs@21.5.0

# 测试完成，关闭终端即可恢复全局版本
```

### 场景四：多项目管理

```bash
# 项目 A 使用 Node.js 18
cd project-a
vfox use -p nodejs@18.0.0

# 项目 B 使用 Node.js 20
cd ../project-b
vfox use -p nodejs@20.9.0

# 全局默认使用 Node.js 21
cd ~
vfox use -g nodejs@21.5.0
```

## 最佳实践

1. **项目级配置优先**：使用 `vfox use -p` 设置项目版本，并提交 `.vfox.toml`
2. **忽略工作目录**：将 `.vfox/` 添加到 `.gitignore`
3. **团队统一环境**：团队成员使用相同的 `.vfox.toml` 配置
4. **定期清理**：使用 `vfox uninstall` 清理不用的旧版本
5. **版本选择**：生产环境避免使用 `@latest`，建议指定具体版本

## 常见问题

### 1. 安装后命令不生效？

确保已配置 Shell 自动加载：
```bash
# 查看是否已添加到配置文件
cat ~/.bashrc | grep vfox

# 重新加载配置
source ~/.bashrc
```

### 2. 如何更新 vfox？

```bash
# Windows (Scoop)
scoop update vfox

# Windows (Winget)
winget upgrade vfox

# macOS (Homebrew)
brew upgrade vfox

# Linux (APT)
sudo apt-get update && sudo apt-get upgrade vfox
```

### 3. 如何查看插件安装位置？

```bash
# 全局 SDK 位置
~/.vfox/sdks/

# 项目 SDK 位置
项目目录/.vfox/sdks/

# 会话临时目录
~/.vfox/tmp/<pid>/sdks/
```

### 4. 支持哪些工具？

查看所有支持的插件：
```bash
vfox available
```

常用插件包括：Node.js、Python、Java、Go、.NET、Rust、Maven、Gradle 等。

## 相关资源

- 📖 官方文档：https://vfox.dev/zh-hans/
- 🐙 GitHub 仓库：https://github.com/version-fox/vfox
- 🔌 插件列表：https://github.com/version-fox/vfox-plugins

---

> 💡 提示：使用 `vfox --help` 查看完整命令列表
