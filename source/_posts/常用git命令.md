---
title: 常用git命令
date: 2026-03-25 13:00:00
tags: [git]
categories: [git]
---

## 初始化配置

```bash
# 设置用户名和邮箱
git config --global user.name "你的名字"
git config --global user.email "你的邮箱"

# 查看配置
git config --list
git config user.name

# 设置默认编辑器
git config --global core.editor "code --wait"    # VSCode
git config --global core.editor "notepad"        # 记事本

# 设置默认分支名
git config --global init.defaultBranch main

# 设置别名
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.lg "log --oneline --graph --all"
```

## 创建仓库

```bash
# 初始化仓库
git init                                        # 在当前目录创建仓库
git init <项目名>                               # 创建新目录并初始化

# 克隆远程仓库
git clone <仓库地址>                            # 克隆仓库
git clone <仓库地址> <目录名>                   # 克隆到指定目录
git clone -b <分支名> <仓库地址>                # 克隆指定分支
git clone --depth 1 <仓库地址>                  # 浅克隆（只克隆最近一次提交）
```

## 文件操作

```bash
# 添加到暂存区
git add <文件名>                                # 添加单个文件
git add .                                       # 添加所有修改
git add -A                                      # 添加所有文件（包括删除）
git add -u                                      # 只添加已跟踪文件的修改

# 删除文件
git rm <文件名>                                 # 删除文件并添加到暂存区
git rm --cached <文件名>                        # 只从暂存区删除，保留本地文件
git rm -r <目录>                                # 递归删除目录

# 重命名文件
git mv <原文件名> <新文件名>                    # 重命名文件

# 撤销修改
git restore <文件名>                            # 撤销工作区修改（Git 2.23+）
git restore --staged <文件名>                   # 撤销暂存区（取消add）
git checkout -- <文件名>                        # 撤销工作区修改（旧语法）

# 查看状态
git status                                      # 查看状态
git status -s                                   # 简洁模式
git status -sb                                  # 简洁模式并显示分支信息
```

## 提交修改

```bash
# 提交
git commit -m "提交信息"                        # 提交暂存区的修改
git commit -am "提交信息"                       # 跳过暂存区直接提交已跟踪文件
git commit                                      # 打开编辑器写提交信息

# 修改提交
git commit --amend -m "新的提交信息"            # 修改最后一次提交信息
git commit --amend                              # 修改最后一次提交（包括文件）

# 查看提交历史
git log                                         # 详细提交历史
git log --oneline                               # 简洁显示
git log --graph                                 # 图形化显示分支
git log --graph --oneline --all                 # 图形化显示所有分支
git log -n 5                                    # 显示最近5次提交
git log --author="作者名"                       # 按作者筛选
git log --since="2024-01-01"                    # 按时间筛选
git log -p                                      # 显示每次提交的差异
git log --stat                                  # 显示文件变更统计

# 查看特定文件的历史
git log <文件名>                                # 文件的提交历史
git log -p <文件名>                             # 文件的详细变更历史
git blame <文件名>                              # 显示文件的每一行是谁修改的
```

## 分支管理

```bash
# 查看分支
git branch                                      # 查看本地分支
git branch -r                                   # 查看远程分支
git branch -a                                   # 查看所有分支
git branch -v                                   # 查看分支及最后一次提交

# 创建分支
git branch <分支名>                             # 创建新分支
git branch <分支名> <提交ID>                    # 基于指定提交创建分支

# 切换分支
git checkout <分支名>                           # 切换到已有分支
git checkout -b <分支名>                        # 创建并切换到新分支
git switch <分支名>                             # 切换分支（Git 2.23+）
git switch -c <分支名>                          # 创建并切换（Git 2.23+）

# 合并分支
git merge <分支名>                              # 合并指定分支到当前分支
git merge --no-ff <分支名>                      # 禁用快进模式合并
git merge --abort                               # 取消合并

# 删除分支
git branch -d <分支名>                          # 删除已合并的分支
git branch -D <分支名>                          # 强制删除分支

# 重命名分支
git branch -m <旧分支名> <新分支名>             # 重命名分支
git branch -m <新分支名>                        # 重命名当前分支

# 查看分支合并情况
git branch --merged                             # 已合并到当前分支的分支
git branch --no-merged                          # 未合并到当前分支的分支
```

## 远程仓库

```bash
# 查看远程仓库
git remote                                      # 显示远程仓库名
git remote -v                                   # 显示远程仓库详细信息
git remote show <远程名>                        # 显示远程仓库详情

# 添加远程仓库
git remote add origin <仓库地址>                # 添加远程仓库
git remote add <名称> <仓库地址>                # 添加多个远程仓库

# 删除远程仓库
git remote rm <远程名>                          # 删除远程仓库

# 重命名远程仓库
git remote rename <旧名称> <新名称>

# 拉取远程更新
git fetch <远程名>                              # 获取远程仓库更新
git fetch origin                                # 获取origin的更新
git fetch --all                                 # 获取所有远程仓库更新

# 拉取并合并
git pull <远程名> <分支名>                      # 拉取并合并
git pull origin main                            # 拉取origin的main分支
git pull --rebase                               # 使用rebase模式拉取

# 推送到远程
git push <远程名> <分支名>                      # 推送到远程仓库
git push origin main                            # 推送到origin的main分支
git push -u origin main                         # 推送并设置上游分支
git push --all origin                           # 推送所有分支
git push --force                                # 强制推送（慎用）
git push --force-with-lease                     # 更安全的强制推送

# 删除远程分支
git push origin --delete <分支名>               # 删除远程分支

# 跟踪远程分支
git branch --set-upstream-to=origin/<分支名> <本地分支名>
git checkout --track origin/<分支名>            # 创建跟踪分支
```

## 撤销操作

```bash
# 撤销工作区修改
git restore <文件名>                            # 撤销工作区修改
git checkout -- <文件名>                        # 撤销工作区修改（旧语法）

# 撤销暂存
git restore --staged <文件名>                   # 取消暂存
git reset HEAD <文件名>                         # 取消暂存（旧语法）

# 回退提交
git reset --soft HEAD~1                         # 回退1个提交，保留修改在暂存区
git reset --mixed HEAD~1                        # 回退1个提交，保留修改在工作区
git reset --hard HEAD~1                         # 回退1个提交，丢弃所有修改
git reset --hard <提交ID>                       # 回退到指定提交

# 查看引用日志
git reflog                                      # 查看所有操作历史
git reset --hard HEAD@{n}                       # 回退到指定操作

# 撤销已推送的提交
git revert <提交ID>                             # 创建新提交来撤销指定提交
git revert HEAD                                 # 撤销最后一次提交
git revert -n <提交ID>                          # 撤销但不自动提交
```

## 暂存工作区

```bash
# 暂存当前工作
git stash                                       # 暂存工作区
git stash save "描述信息"                       # 带描述的暂存
git stash -u                                    # 包括未跟踪的文件

# 查看暂存
git stash list                                  # 查看所有暂存

# 应用暂存
git stash apply                                 # 应用最近的暂存
git stash apply stash@{n}                       # 应用指定暂存
git stash pop                                   # 应用并删除最近的暂存

# 删除暂存
git stash drop stash@{n}                        # 删除指定暂存
git stash clear                                 # 删除所有暂存

# 查看暂存内容
git stash show                                  # 查看暂存摘要
git stash show -p                               # 查看暂存详细差异
```

## 标签管理

```bash
# 创建标签
git tag <标签名>                                # 创建轻量标签
git tag -a <标签名> -m "标签信息"               # 创建附注标签
git tag <标签名> <提交ID>                       # 给指定提交打标签

# 查看标签
git tag                                         # 查看所有标签
git tag -l "v1.*"                               # 按模式筛选标签
git show <标签名>                               # 查看标签信息

# 推送标签
git push origin <标签名>                        # 推送单个标签
git push origin --tags                          # 推送所有标签

# 删除标签
git tag -d <标签名>                             # 删除本地标签
git push origin --delete <标签名>               # 删除远程标签

# 检出标签
git checkout <标签名>                           # 检出标签（分离头指针）
git checkout -b <分支名> <标签名>               # 基于标签创建分支
```

## 查看差异

```bash
# 查看差异
git diff                                        # 工作区与暂存区的差异
git diff --staged                               # 暂存区与最新提交的差异
git diff --cached                               # 同 --staged
git diff HEAD                                   # 工作区与最新提交的差异
git diff <提交1> <提交2>                        # 两次提交之间的差异
git diff <分支1> <分支2>                        # 两个分支之间的差异
git diff <文件名>                               # 查看特定文件的差异

# 统计差异
git diff --stat                                 # 显示差异统计
git diff --shortstat                            # 简要统计
```

## 变基操作

```bash
# 变基
git rebase <分支名>                             # 将当前分支变基到指定分支
git rebase -i HEAD~n                            # 交互式变基最近n个提交

# 变基过程中
git rebase --continue                           # 解决冲突后继续变基
git rebase --skip                               # 跳过当前提交
git rebase --abort                              # 取消变基

# 常用变基场景
git rebase -i HEAD~3                            # 合并/编辑最近3次提交
git rebase -i --root                            # 从根提交开始变基
```

## 其他实用命令

```bash
# 搜索
git grep "搜索内容"                             # 在仓库中搜索
git grep -n "内容"                              # 显示行号
git grep -c "内容"                              # 显示匹配次数

# 归档
git archive --format=zip --output=project.zip HEAD    # 导出zip
git archive --format=tar --output=project.tar HEAD    # 导出tar

# 清理未跟踪文件
git clean -n                                    # 预览要删除的文件
git clean -f                                    # 删除未跟踪的文件
git clean -fd                                   # 删除未跟踪的文件和目录
git clean -fdx                                  # 包括被.gitignore忽略的文件

# 二分查找
git bisect start                                # 开始二分查找
git bisect bad                                  # 标记当前提交有问题
git bisect good <提交ID>                        # 标记正常提交
git bisect reset                                # 重置二分查找

# 查看对象
git show <提交ID>                               # 显示提交详情
git show HEAD                                   # 显示最新提交
git show HEAD~1                                 # 显示上一次提交

# 检查仓库
git fsck                                        # 检查仓库完整性
git gc                                          # 清理不必要的文件并优化仓库
git count-objects -v                            # 查看仓库对象统计
```

## 常用组合示例

```bash
# 第一次推送新仓库
git init
git add .
git commit -m "Initial commit"
git remote add origin <仓库地址>
git push -u origin main

# 同步fork的仓库
git remote add upstream <原仓库地址>
git fetch upstream
git checkout main
git merge upstream/main
git push origin main

# 修改历史提交信息
git rebase -i HEAD~3                            # 编辑最近3次提交
# 将要修改的提交前的 pick 改为 edit
git commit --amend -m "新的提交信息"
git rebase --continue

# 合并多个提交
git rebase -i HEAD~3
# 将要合并的提交前的 pick 改为 squash
# 保存后会提示编辑合并后的提交信息

# 撤销已推送的错误提交
git revert <提交ID>
git push origin main

# 创建hotfix分支修复bug
git checkout -b hotfix-123 main
# 修改代码
git commit -m "Fix issue #123"
git checkout main
git merge hotfix-123
git branch -d hotfix-123
git push origin main
```

## Git 工作流程

### Git Flow 工作流

```bash
# 主分支
main        - 生产环境代码
develop     - 开发分支
feature/*   - 功能分支
release/*   - 发布分支
hotfix/*    - 热修复分支

# 开发新功能
git checkout develop
git checkout -b feature/new-feature
# 开发...
git checkout develop
git merge feature/new-feature
git branch -d feature/new-feature

# 准备发布
git checkout -b release/v1.0 develop
# 测试、修复bug...
git checkout main
git merge release/v1.0
git tag -a v1.0 -m "Release v1.0"
git checkout develop
git merge release/v1.0
git branch -d release/v1.0

# 紧急修复
git checkout -b hotfix/v1.0.1 main
# 修复bug...
git checkout main
git merge hotfix/v1.0.1
git tag -a v1.0.1 -m "Hotfix v1.0.1"
git checkout develop
git merge hotfix/v1.0.1
git branch -d hotfix/v1.0.1
```

### GitHub Flow 工作流

```bash
# 更简单的工作流，只有一个主分支
main        - 主分支（始终可部署）

# 开发流程
git checkout main
git pull origin main
git checkout -b new-feature
# 开发、提交...
git push origin new-feature
# 在 GitHub 创建 Pull Request
# Code Review 后合并到 main
git checkout main
git pull origin main
```

---

> 💡 提示：使用 `git <命令> --help` 可以查看详细帮助文档
