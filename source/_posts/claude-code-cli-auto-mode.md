---
title: Claude Code CLI 自动模式 — -c 与 --dangerously-skip-permissions 的用法
date: 2026-07-24 09:53:58
tags: [Claude Code, CLI, 工具]
categories: 工具
description: Claude Code CLI 的 --continue、--dangerously-skip-permissions 与 --permission-mode 选项组合用法，适合长时间无人值守任务的场景
---

日常工作里经常需要让 Claude Code 在终端里"自己跑完一堆事"而不需要每次都点确认。记录一下 `-c`（继续上次会话）和 `--dangerously-skip-permissions`（跳过权限检查）的常用姿势，以及几个权限模式的适用场景。

## 权限模式速览

Claude Code 提供了 6 种权限模式，加 `--permission-mode` 或对应 flag 指定：

| 模式 | 说明 | 适合场景 |
|---|---|---|
| `manual`（默认） | 只读自动放行，写操作要确认 | 日常交互 |
| `acceptEdits` | 文件编辑 + 常见 fs 命令自动放行 | 大量文件改动时 |
| `plan` | 只读，不允许编辑 | 先调研再动手 |
| `auto` | 全部放行 + 分类器安全检查 | 长时间自主任务 |
| `dontAsk` | 只放行 pre-approved 的工具 | CI / 脚本 |
| `bypassPermissions` | 全部放行，无任何检查 | **仅限隔离容器/沙箱** |

日常交互时按 `Shift+Tab` 可以在 `manual → acceptEdits → plan` 之间循环切换。

## -c / --continue：继续上次会话

退出终端后想接着之前的工作继续聊，用 `-c` 直接恢复最近一次会话：

```bash
# 交互式恢复最近会话
claude -c

# 非交互式 — 带新 prompt 继续跑
claude -c -p "刚才那个 bug 继续修"

# 指定名字恢复
claude -r "my-session" "继续写完测试"
```

配合 `-p` 可以在脚本里把多步工作串联起来：

```bash
claude -p "审查 ./src 目录下的性能问题" --output-format json > step1.json
claude -c -p "重点检查数据库查询部分，输出修复建议" --permission-mode auto
claude -c -p "生成修复汇总，写到 ISSUES.md"
```

第一步产出 JSON，后续两步接着同一个会话继续，上下文不丢。

## --dangerously-skip-permissions：跳过所有权限检查

这是一个"一刀切"的大招 — 等价于 `--permission-mode bypassPermissions`：

```bash
# 交互式
claude --dangerously-skip-permissions

# 配合 -p + -c 实现完全无人值守
claude -c -p "把 remaining todos 全部实现并提交" --dangerously-skip-permissions
```

**跑起来不会有任何确认弹窗** — 读写文件、执行命令、访问网络全都直接放行，包括 `.git`、shell rc 等受保护路径。

> **官方警告：只在没有网络访问的沙箱/容器/VM 里用这个。对宿主机用 bypassPermissions 可能会搞坏系统。**

几个限制还是有的：
- 显式配置的 `ask` 规则仍然会弹窗
- `rm -rf /` / `rm -rf ~` 这类灾难操作有硬性阻断
- Linux/macOS 上拒绝在 root/sudo 下启动
- 非交互模式（`-p`）下不弹首次警告对话

## 常用组合场景

### 场景一：长时间任务，偶尔看一眼

```bash
# 开始一个会话并给个名字
claude -n "refactor-auth" 

# 说完需求后用 Shift+Tab 切到 auto 模式，让它自己跑
# 中间不需要频繁点确认，但分类器会在后台做安全检查
```

### 场景二：继续昨天的活，全自动跑完

```bash
# 恢复最近会话，完全跳过权限检查
claude -c --dangerously-skip-permissions

# 或配合 -p 一次性执行
claude -c -p "完成上次未做完的迁移并跑测试" --dangerously-skip-permissions
```

### 场景三：脚本 / CI 里的安全用法（推荐 dontAsk）

```bash
# 只放行明确允许的工具，其他一律拒绝 — 不会卡住等确认
claude --bare -p "跑测试并修复失败用例" \
  --permission-mode dontAsk \
  --allowedTools "Bash(npm test *),Bash(npm run *),Read,Edit" \
  --output-format json \
  --max-budget-usd 3.00
```

`dontAsk` 比 `bypassPermissions` 安全得多 — 不在 allow list 里的操作直接拒绝，不会等用户确认也不会偷偷执行。

### 场景四：自动提交 + 推送

```bash
# 注意：默认 auto 模式下 git push 不会自动放行
# 可以在 ~/.claude/settings.json 里配置 ask 规则
```

```json
{
  "permissions": {
    "defaultMode": "auto",
    "ask": [
      "Bash(git push *)",
      "Bash(gh pr create *)"
    ]
  }
}
```

这样 auto 模式下日常操作自动放行，push 和 PR 创建还是要手动确认，防止搞出事故。

## 几点提醒

- **`bypassPermissions` ≠ auto 模式**。auto 模式有分类器做安全检查；bypass 是完全绕开所有检查。
- `defaultMode: "auto"` 要写在 `~/.claude/settings.json`（用户级），写在项目级 settings 里不生效，防止恶意仓库给自己提权。
- CI 场景优先用 `dontAsk` + `--allowedTools`，不要在 CI 里用 `--dangerously-skip-permissions`。
- 真要用 bypass 就用 `--dangerously-skip-permissions`，名字够吓人，看一眼就知道自己在干什么。

## 参考

- [CLI reference](https://code.claude.com/docs/en/cli-reference.md)
- [Permission modes](https://code.claude.com/docs/en/permission-modes.md)
- [Auto mode config](https://code.claude.com/docs/en/auto-mode-config.md)
- [Headless mode](https://code.claude.com/docs/en/headless.md)
