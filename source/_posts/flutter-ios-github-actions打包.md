---
title: Flutter 通过 GitHub Actions 打包 iOS 版本
date: 2026-04-01 17:22:00
tags: [Flutter, iOS, GitHub Actions]名称Bundle ID平台操作liftDispatchSdkcom.smc.liftDispatchSdkUNIVERSAL 
---

使用 GitHub Actions 自动化打包 iOS 应用，无需 Mac 电脑即可完成整个流程。

## 一、通过 appuploader 获取开发者证书

### 准备工作

- 拥有 Apple 开发者账号
- 下载 [appuploader](https://www.appuploader.net/) 工具

### 生成 Bundle ID

打开 appuploader，创建 Bundle ID，需要与 Flutter 项目中的 `bundle identifier` 保持一致。

![image-20260401175159600](https://picgo.19991029.xyz/image-20260401175159600.png)

### 生成证书

在 appuploader 中创建开发证书或发布证书：

![image-20260401175212136](https://picgo.19991029.xyz/image-20260401175212136.png)

### 获取设备 UDID

创建描述文件前，需要添加测试设备的 UDID。可通过蒲公英分发平台的 UDID 工具获取：

https://www.pgyer.com/tools/udid

使用 iOS 设备访问该链接，按照提示安装描述文件后即可获取设备的 UDID。

![image-20260401175444352](https://picgo.19991029.xyz/image-20260401175444352.png)


### 创建描述文件

在 appuploader 中根据证书类型创建对应的描述文件（Provisioning Profile），添加测试设备。

![image-20260401175221588](https://picgo.19991029.xyz/image-20260401175221588.png)

## 二、配置 GitHub Actions

### 上传代码到 GitHub

将 Flutter 项目推送到 GitHub 仓库。

### 创建 Workflow 文件

在项目根目录创建 `.github/workflows/ios-build.yml`：

```yaml
name: Build iOS

on:
  push:
    branches: [ main, master ]
  workflow_dispatch:

env:
  FLUTTER_VERSION: '3.24.0'

jobs:
  build-ios:
    runs-on: macos-latest
    
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ env.FLUTTER_VERSION }}
          channel: 'stable'

      - name: Install Dependencies
        run: flutter pub get

      - name: Install CocoaPods Dependencies
        working-directory: ios
        run: pod install

      - name: Install Apple Certificate & Provisioning Profile
        env:
          IOS_CERTIFICATE: ${{ secrets.IOS_CERTIFICATE }}
          IOS_CERTIFICATE_PASSWORD: ${{ secrets.IOS_CERTIFICATE_PASSWORD }}
          IOS_PROVISIONING_PROFILE: ${{ secrets.IOS_PROVISIONING_PROFILE }}
          KEYCHAIN_PASSWORD: ${{ secrets.KEYCHAIN_PASSWORD }}
        run: |
          # Create temporary keychain
          KEYCHAIN_PATH=$RUNNER_TEMP/app-signing.keychain-db
          security create-keychain -p "$KEYCHAIN_PASSWORD" $KEYCHAIN_PATH
          security set-keychain-settings -lut 21600 $KEYCHAIN_PATH
          security unlock-keychain -p "$KEYCHAIN_PASSWORD" $KEYCHAIN_PATH

          # Import certificate
          CERT_PATH=$RUNNER_TEMP/certificate.p12
          echo -n "$IOS_CERTIFICATE" | base64 --decode -o $CERT_PATH
          security import $CERT_PATH -P "$IOS_CERTIFICATE_PASSWORD" -A -t cert -f pkcs12 -k $KEYCHAIN_PATH
          security list-keychain -d user -s $KEYCHAIN_PATH

          # Install provisioning profile with UUID as filename
          PROFILE_DIR=~/Library/MobileDevice/Provisioning\ Profiles
          mkdir -p "$PROFILE_DIR"
          
          TEMP_PROFILE=$RUNNER_TEMP/profile.mobileprovision
          echo -n "$IOS_PROVISIONING_PROFILE" | base64 --decode > "$TEMP_PROFILE"
          
          # Extract UUID and install with correct filename
          PROFILE_UUID=$(security cms -D -i "$TEMP_PROFILE" | plutil -extract UUID raw -)
          mv "$TEMP_PROFILE" "$PROFILE_DIR/$PROFILE_UUID.mobileprovision"
          
          echo "Installed profile: $PROFILE_UUID.mobileprovision"

      - name: Build iOS
        run: flutter build ios --release --no-codesign

      - name: Sign and Package IPA
        run: |
          # Get signing identity
          SIGNING_IDENTITY=$(security find-identity -v -p codesigning $RUNNER_TEMP/app-signing.keychain-db | grep "Apple Development" | head -1 | awk -F'"' '{print $2}')
          echo "Signing identity: $SIGNING_IDENTITY"
          
          # Get provisioning profile
          PROFILE_PATH=$(ls ~/Library/MobileDevice/Provisioning\ Profiles/*.mobileprovision | head -1)
          echo "Using profile: $PROFILE_PATH"
          
          # Extract entitlements from provisioning profile
          ENTITLEMENTS_PATH="$RUNNER_TEMP/app.entitlements"
          PROFILE_PLIST="$RUNNER_TEMP/profile.plist"
          
          # Decode profile and extract entitlements
          security cms -D -i "$PROFILE_PATH" > "$PROFILE_PLIST"
          plutil -extract Entitlements xml1 -o "$ENTITLEMENTS_PATH" "$PROFILE_PLIST"
          
          echo "Extracted entitlements:"
          cat "$ENTITLEMENTS_PATH"
          
          # Embed profile into app
          APP_PATH="build/ios/iphoneos/Runner.app"
          cp "$PROFILE_PATH" "$APP_PATH/embedded.mobileprovision"
          
          # Sign frameworks
          echo "Signing frameworks..."
          for framework in "$APP_PATH/Frameworks"/*; do
            if [ -d "$framework" ]; then
              codesign --force --deep --sign "$SIGNING_IDENTITY" --timestamp=none "$framework"
            fi
          done
          
          # Sign app with extracted entitlements
          echo "Signing app..."
          codesign --force --deep --sign "$SIGNING_IDENTITY" --entitlements "$ENTITLEMENTS_PATH" --timestamp=none "$APP_PATH"
          
          # Verify signature
          echo "Verifying signature..."
          codesign -dvv "$APP_PATH"
          
          # Create IPA
          mkdir -p Payload
          cp -r "$APP_PATH" Payload/
          zip -r build/ios_build.ipa Payload

      - name: Upload IPA Artifact
        uses: actions/upload-artifact@v4
        with:
          name: ios-release
          path: build/ios_build.ipa
          retention-days: 30

      - name: Clean up Keychain
        if: ${{ always() }}
        run: security delete-keychain $RUNNER_TEMP/app-signing.keychain-db || true
```

### 配置 GitHub Secrets

在仓库的 `Settings -> Secrets and variables -> Actions` 中添加：

- `IOS_CERTIFICATE`：证书文件的 base64 编码
- `IOS_CERTIFICATE_PASSWORD`：证书密码
- `IOS_PROVISIONING_PROFILE`：描述文件的 base64 编码
- `KEYCHAIN_PASSWORD`：临时钥匙串密码（可自定义）

将证书转换为 base64：

**macOS/Linux:**

```bash
base64 -i your_certificate.p12 | pbcopy
base64 -i your_profile.mobileprovision | pbcopy
```

**Windows PowerShell:**

```powershell
[Convert]::ToBase64String((Get-Content 证书.p12 -Encoding Byte))
[Convert]::ToBase64String((Get-Content 描述文件.mobileprovision -Encoding Byte))
```

### 触发构建

推送代码或手动触发 workflow，等待构建完成。

### 下载构建产物

在 Actions 页面找到完成的 workflow，下载 `ios-release` 产物，解压得到 IPA 文件。

## 三、安装已签名的 IPA

### 方法一：使用 appuploader

将下载的 IPA 上传到 appuploader，安装到已连接的 iOS 设备。

### 方法二：使用爱思助手

1. 连接 iPhone 到电脑
2. 打开爱思助手
3. 选择「应用游戏」->「导入安装」
4. 选择下载的 IPA 文件进行安装

## 四、注意事项

### 证书有效期

开发者证书有效期为 **1 年**，到期后需要重新生成证书和描述文件，并更新 GitHub Secrets。

### 设备限制

Apple 开发者账号最多可注册 **100 台设备**（包括 iPhone、iPad、iPod touch），设备列表可在 appuploader 中管理。
