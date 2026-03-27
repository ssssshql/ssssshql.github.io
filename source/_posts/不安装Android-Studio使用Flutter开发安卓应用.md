---
title: 不安装 Android Studio 使用 Flutter 开发安卓应用
date: 2026-03-27 01:30:00
tags: [Flutter, Android]
---

## 下载 Command line tools

https://developer.android.com/studio 页面底部找到 **Command line tools only**

## 解压并调整目录

解压后将 `cmdline-tools` 目录下的所有文件移至 `cmdline-tools/latest`

```
cmdline-tools/
  └── latest/
      ├── bin/
      ├── lib/
      └── ...
```

## 安装 SDK 工具包

```bash
cd cmdline-tools/latest/bin

sdkmanager --sdk_root=D:\android "platform-tools" "platforms;android-36" "build-tools;36.0.0"
```

`--sdk_root` 根据实际路径调整，工具包会下载到该目录。

## 接受 SDK 许可证

```bash
sdkmanager --licenses
```

按提示输入 `y` 接受所有协议。

## 配置环境变量

- `ANDROID_HOME` = `D:\android`
- `Path` 添加：
  - `%ANDROID_HOME%\platform-tools`
  - `cmdline-tools\bin` 路径

## 验证

```bash
flutter doctor
```

看到 Android toolchain 相关项无报错即配置成功。
