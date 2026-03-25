---
title: 常用cmd命令
date: 2026-03-25 12:26:25
tags: [cmd]
categories: [cmd]
---

## 文件和目录操作

```cmd
# 切换目录
cd <路径>                    # 进入指定目录
cd ..                        # 返回上一级目录
cd \                         # 返回根目录

# 显示目录内容
dir                          # 显示当前目录内容
dir /w                       # 宽格式显示
dir /s                       # 包含子目录
dir *.txt                    # 显示所有txt文件

# 创建目录
mkdir <目录名>               # 创建目录
md <目录名>                  # mkdir的简写

# 删除目录
rmdir <目录名>               # 删除空目录
rmdir /s <目录名>            # 删除目录及其内容
rd <目录名>                  # rmdir的简写

# 创建文件
type nul > <文件名>          # 创建空文件
echo <内容> > <文件名>       # 创建文件并写入内容
echo <内容> >> <文件名>      # 追加内容到文件

# 删除文件
del <文件名>                 # 删除文件
del *.tmp                    # 删除所有tmp文件
del /s <文件名>              # 删除子目录中的文件

# 复制文件
copy <源文件> <目标>         # 复制文件
copy /y <源> <目标>          # 不提示确认直接覆盖
xcopy <源> <目标> /e /i      # 复制目录及子目录

# 移动/重命名
move <源> <目标>             # 移动文件或目录
ren <原文件名> <新文件名>    # 重命名文件

# 查看文件内容
type <文件名>                # 显示文件内容
more <文件名>                # 分页显示文件内容
```

## 网络相关

```cmd
# 查看网络配置
ipconfig                     # 显示IP配置
ipconfig /all                # 显示详细配置信息
ipconfig /release            # 释放IP地址
ipconfig /renew              # 重新获取IP地址

# 网络诊断
ping <主机名/IP>             # 测试网络连接
ping -t <主机>               # 持续ping直到中断
ping -n 10 <主机>            # 指定ping次数

tracert <主机>               # 追踪路由
nslookup <域名>              # 查询DNS解析

# 网络状态
netstat                      # 显示网络连接
netstat -an                  # 显示所有连接和端口
netstat -ano                 # 显示连接和进程ID

# 端口和防火墙
netstat -ano | findstr <端口>    # 查找指定端口占用
taskkill /PID <进程ID> /F        # 强制结束占用端口的进程
```

## 系统相关

```cmd
# 系统信息
systeminfo                   # 显示系统详细信息
hostname                     # 显示计算机名
whoami                       # 显示当前用户

# 环境变量
set                          # 显示所有环境变量
set <变量名>=<值>            # 设置环境变量（临时）
echo %PATH%                  # 显示PATH变量

# 关机重启
shutdown /s /t 0             # 立即关机
shutdown /r /t 0             # 立即重启
shutdown /l                  # 注销
shutdown /a                  # 取消关机计划

# 时间日期
date                         # 显示/设置日期
time                         # 显示/设置时间

# 清屏
cls                          # 清除屏幕内容
```

## 进程管理

```cmd
# 查看进程
tasklist                     # 显示所有进程
tasklist | findstr <进程名>  # 查找特定进程

# 结束进程
taskkill /IM <进程名>        # 按进程名结束
taskkill /PID <进程ID>       # 按进程ID结束
taskkill /F /IM <进程名>     # 强制结束进程

# 启动程序
start <程序名>               # 启动程序
start notepad                # 启动记事本
start http://网址            # 用默认浏览器打开网址
```

## 磁盘管理

```cmd
# 查看磁盘
wmic logicaldisk get size,freecaption    # 查看磁盘空间

# 磁盘操作
chkdsk <盘符>:               # 检查磁盘
format <盘符>:               # 格式化磁盘（谨慎使用）

# 切换磁盘
<盘符>:                      # 如 C: D: E:
```

## 文件查找和搜索

```cmd
# 搜索文件
dir /s /b <文件名>           # 搜索文件并显示完整路径

# 查找文本
find "<文本>" <文件名>       # 在文件中查找文本
findstr "<模式>" <文件名>    # 支持正则表达式查找
findstr /s /i "text" *.txt   # 在所有txt文件中查找（忽略大小写）

# where 命令
where <文件名>               # 查找文件位置
where /r <目录> <文件名>     # 在指定目录中递归查找
```

## 循环命令（for）

```cmd
# 基本循环
for %i in (item1 item2 item3) do echo %i

# 批处理文件中使用双百分号
for %%i in (item1 item2 item3) do echo %%i

# 数字循环（/L）
for /l %i in (1,1,10) do echo %i           # 输出1到10
for /l %i in (10,-1,1) do echo %i          # 倒序输出10到1
for /l %i in (0,2,10) do echo %i           # 0到10，步长2

# 遍历文件
for %i in (*.txt) do echo %i               # 遍历所有txt文件
for %i in (D:\test\*) do echo %i           # 遍历指定目录文件

# 递归遍历目录（/R）
for /r %i in (*.txt) do echo %i            # 递归查找所有txt文件
for /r "C:\test" %i in (*) do echo %i      # 指定起始目录递归

# 遍历目录（/D）
for /d %i in (*) do echo %i                # 只遍历目录，不包含文件

# 读取文件内容（/F）
for /f "tokens=*" %i in (file.txt) do echo %i

# 读取命令输出
for /f "tokens=*" %i in ('dir /b') do echo %i

# 读取多列（tokens）
for /f "tokens=1,2" %i in (file.txt) do echo 第一列:%i 第二列:%j

# 按行分割（delims）
for /f "delims=," %i in ("a,b,c") do echo %i

# 跳过前几行（skip）
for /f "skip=2" %i in (file.txt) do echo %i

# 实用示例
# 批量重命名文件
for %i in (*.txt) do ren "%i" "prefix_%i"

# 批量复制文件
for %i in (*.txt) do copy "%i" "backup\"

# 统计文件数量
set count=0
for %i in (*.txt) do set /a count+=1
echo 共有 %count% 个txt文件

# 遍历目录并执行操作
for /d %d in (*) do (
    echo 处理目录: %d
    cd %d
    dir
    cd ..
)

# 批量创建文件夹
for /l %i in (1,1,10) do mkdir "folder%i"
```

## 批处理相关

```cmd
# 注释
REM 这是注释
:: 这也是注释

# 输出
echo <内容>                  # 输出内容
echo.                        # 输出空行

# 暂停
pause                        # 暂停执行，按任意键继续

# 调用批处理
call <批处理文件>            # 调用另一个批处理文件

# 条件判断
if exist <文件名> (命令) else (命令)
if "%变量%"=="值" (命令)
if errorlevel 1 (命令)       # 如果上一条命令失败

# goto 跳转
goto :label                  # 跳转到标签
:label                       # 定义标签

# 循环配合条件
for %i in (*.txt) do (
    if exist "%i" echo 文件存在: %i
)
```

## 其他实用命令

```cmd
# 帮助
<命令> /?                    # 显示命令帮助
help                         # 显示命令列表

# 历史记录
doskey /history              # 显示命令历史

# 别名设置
doskey ls=dir                # 为dir命令设置别名ls

# 打开系统工具
calc                         # 计算器
notepad                      # 记事本
mspaint                      # 画图
control                      # 控制面板
explorer                     # 资源管理器
regedit                      # 注册表编辑器
msconfig                     # 系统配置
services.msc                 # 服务管理
```

## 常用组合示例

```cmd
# 查找并结束占用端口的进程
netstat -ano | findstr :8080
taskkill /PID <进程ID> /F

# 批量删除特定类型文件
del /s /q *.log

# 查找大文件
forfiles /S /M * /C "cmd /c if @fsize GEQ 104857600 echo @path @fsize"

# 快速创建项目目录结构
mkdir project\src project\dist project\docs
```

---
