# GitHub Actions - Flutter CI/CD 完整指南

## 📖 简介

GitHub Actions 是 GitHub 原生的 CI/CD 服务，与 GitHub 仓库无缝集成。对于 Flutter 项目来说，它是最推荐的 CI/CD 解决方案之一。

### 优势

- ✅ **免费额度充足**：公开仓库无限免费，私有仓库每月 2000 分钟
- ✅ **无缝集成**：原生支持 GitHub，无需额外配置
- ✅ **生态丰富**：大量现成的 Actions 可供使用
- ✅ **多平台支持**：支持 Linux、macOS、Windows 构建机器
- ✅ **配置灵活**：强大的 YAML 配置和条件执行

### 劣势

- ⚠️ macOS 构建机器分钟数消耗 10 倍（构建 iOS 时）
- ⚠️ 大型项目可能需要优化以节省构建时间

## 🚀 快速开始

### 1. 创建工作流文件

在你的 Flutter 项目根目录创建 `.github/workflows/main.yml`：

```yaml
name: Flutter CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: 设置 Flutter
      uses: subosito/flutter-action@v2
      with:
        flutter-version: '3.16.0'
        channel: 'stable'
    
    - name: 获取依赖
      run: flutter pub get
    
    - name: 运行测试
      run: flutter test
    
    - name: 构建 APK
      run: flutter build apk --release
```

### 2. 提交代码

```bash
git add .github/workflows/main.yml
git commit -m "添加 GitHub Actions 工作流"
git push
```

### 3. 查看构建结果

访问你的 GitHub 仓库的 **Actions** 标签页，查看构建进度和结果。

## 📋 完整配置示例

### Android 应用完整构建

```yaml
name: Android CI

on:
  push:
    branches: [ main ]
    tags:
      - 'v*'
  pull_request:
    branches: [ main ]

jobs:
  build-android:
    runs-on: ubuntu-latest
    
    steps:
    - name: 检出代码
      uses: actions/checkout@v4
    
    - name: 设置 Java
      uses: actions/setup-java@v3
      with:
        distribution: 'zulu'
        java-version: '17'
    
    - name: 设置 Flutter
      uses: subosito/flutter-action@v2
      with:
        flutter-version: '3.16.0'
        channel: 'stable'
        cache: true
    
    - name: 获取依赖
      run: flutter pub get
    
    - name: 运行代码分析
      run: flutter analyze
    
    - name: 运行测试
      run: flutter test
    
    - name: 解码 Keystore
      run: |
        echo "${{ secrets.KEYSTORE_BASE64 }}" | base64 --decode > android/app/keystore.jks
    
    - name: 创建 key.properties
      run: |
        cat > android/key.properties << EOF
        storePassword=${{ secrets.STORE_PASSWORD }}
        keyPassword=${{ secrets.KEY_PASSWORD }}
        keyAlias=${{ secrets.KEY_ALIAS }}
        storeFile=keystore.jks
        EOF
    
    - name: 构建 APK
      run: flutter build apk --release
    
    - name: 构建 App Bundle
      run: flutter build appbundle --release
    
    - name: 上传 APK
      uses: actions/upload-artifact@v3
      with:
        name: release-apk
        path: build/app/outputs/flutter-apk/app-release.apk
    
    - name: 上传 App Bundle
      uses: actions/upload-artifact@v3
      with:
        name: release-aab
        path: build/app/outputs/bundle/release/app-release.aab
    
    - name: 创建 Release（仅标签）
      if: startsWith(github.ref, 'refs/tags/')
      uses: softprops/action-gh-release@v1
      with:
        files: |
          build/app/outputs/flutter-apk/app-release.apk
          build/app/outputs/bundle/release/app-release.aab
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### iOS 应用完整构建

```yaml
name: iOS CI

on:
  push:
    branches: [ main ]
    tags:
      - 'v*'

jobs:
  build-ios:
    runs-on: macos-latest
    
    steps:
    - name: 检出代码
      uses: actions/checkout@v4
    
    - name: 设置 Flutter
      uses: subosito/flutter-action@v2
      with:
        flutter-version: '3.16.0'
        channel: 'stable'
        cache: true
    
    - name: 获取依赖
      run: flutter pub get
    
    - name: 运行测试
      run: flutter test
    
    - name: 安装 CocoaPods
      run: |
        cd ios
        pod install
    
    - name: 导入证书和配置文件
      run: |
        # 创建临时 keychain
        security create-keychain -p "${{ secrets.KEYCHAIN_PASSWORD }}" build.keychain
        security default-keychain -s build.keychain
        security unlock-keychain -p "${{ secrets.KEYCHAIN_PASSWORD }}" build.keychain
        security set-keychain-settings -t 3600 -l ~/Library/Keychains/build.keychain
        
        # 导入证书
        echo "${{ secrets.CERTIFICATE_BASE64 }}" | base64 --decode > certificate.p12
        security import certificate.p12 -k build.keychain -P "${{ secrets.CERTIFICATE_PASSWORD }}" -T /usr/bin/codesign
        security set-key-partition-list -S apple-tool:,apple:,codesign: -s -k "${{ secrets.KEYCHAIN_PASSWORD }}" build.keychain
        
        # 安装配置文件
        mkdir -p ~/Library/MobileDevice/Provisioning\ Profiles
        echo "${{ secrets.PROVISIONING_PROFILE_BASE64 }}" | base64 --decode > ~/Library/MobileDevice/Provisioning\ Profiles/profile.mobileprovision
    
    - name: 构建 iOS
      run: flutter build ios --release --no-codesign
    
    - name: 构建 IPA
      run: |
        cd ios
        xcodebuild -workspace Runner.xcworkspace \
          -scheme Runner \
          -configuration Release \
          -archivePath $PWD/build/Runner.xcarchive \
          archive
        
        xcodebuild -exportArchive \
          -archivePath $PWD/build/Runner.xcarchive \
          -exportPath $PWD/build \
          -exportOptionsPlist ExportOptions.plist
    
    - name: 上传 IPA
      uses: actions/upload-artifact@v3
      with:
        name: release-ipa
        path: ios/build/*.ipa
```

### 多平台构建矩阵

```yaml
name: Multi-Platform CI

on:
  push:
    branches: [ main ]

jobs:
  build:
    strategy:
      matrix:
        platform: [android, ios, web, windows, linux, macos]
        include:
          - platform: android
            os: ubuntu-latest
            build-command: flutter build apk --release
            artifact-path: build/app/outputs/flutter-apk/app-release.apk
          
          - platform: ios
            os: macos-latest
            build-command: flutter build ios --release --no-codesign
            artifact-path: build/ios/iphoneos/Runner.app
          
          - platform: web
            os: ubuntu-latest
            build-command: flutter build web --release
            artifact-path: build/web
          
          - platform: windows
            os: windows-latest
            build-command: flutter build windows --release
            artifact-path: build/windows/runner/Release
          
          - platform: linux
            os: ubuntu-latest
            build-command: flutter build linux --release
            artifact-path: build/linux/x64/release/bundle
          
          - platform: macos
            os: macos-latest
            build-command: flutter build macos --release
            artifact-path: build/macos/Build/Products/Release/Runner.app
    
    runs-on: ${{ matrix.os }}
    
    steps:
    - uses: actions/checkout@v4
    
    - name: 安装 Linux 依赖
      if: matrix.platform == 'linux'
      run: |
        sudo apt-get update
        sudo apt-get install -y clang cmake ninja-build pkg-config libgtk-3-dev liblzma-dev
    
    - name: 设置 Flutter
      uses: subosito/flutter-action@v2
      with:
        flutter-version: '3.16.0'
        channel: 'stable'
        cache: true
    
    - name: 启用平台支持
      run: |
        flutter config --enable-web
        flutter config --enable-windows-desktop
        flutter config --enable-linux-desktop
        flutter config --enable-macos-desktop
    
    - name: 获取依赖
      run: flutter pub get
    
    - name: 构建 ${{ matrix.platform }}
      run: ${{ matrix.build-command }}
    
    - name: 上传构建产物
      uses: actions/upload-artifact@v3
      with:
        name: ${{ matrix.platform }}-release
        path: ${{ matrix.artifact-path }}
```

## 🔐 配置密钥和环境变量

### Android 签名配置

#### 1. 准备 Keystore

首先将 keystore 文件转换为 Base64：

```bash
base64 -i android/app/keystore.jks | pbcopy  # macOS
base64 -w 0 android/app/keystore.jks  # Linux
```

#### 2. 在 GitHub 设置 Secrets

在仓库的 **Settings > Secrets and variables > Actions** 中添加：

- `KEYSTORE_BASE64`: Keystore 的 Base64 编码
- `STORE_PASSWORD`: Store 密码
- `KEY_PASSWORD`: Key 密码
- `KEY_ALIAS`: Key 别名

#### 3. 配置 build.gradle

确保 `android/app/build.gradle` 包含：

```gradle
def keystoreProperties = new Properties()
def keystorePropertiesFile = rootProject.file('key.properties')
if (keystorePropertiesFile.exists()) {
    keystoreProperties.load(new FileInputStream(keystorePropertiesFile))
}

android {
    signingConfigs {
        release {
            keyAlias keystoreProperties['keyAlias']
            keyPassword keystoreProperties['keyPassword']
            storeFile keystoreProperties['storeFile'] ? file(keystoreProperties['storeFile']) : null
            storePassword keystoreProperties['storePassword']
        }
    }
    buildTypes {
        release {
            signingConfig signingConfigs.release
        }
    }
}
```

### iOS 证书配置

#### 1. 导出证书

```bash
# 导出证书为 .p12 文件
# 在 macOS Keychain Access 中右键证书 > 导出

# 转换为 Base64
base64 -i certificate.p12 | pbcopy
```

#### 2. 导出 Provisioning Profile

```bash
# 从 ~/Library/MobileDevice/Provisioning Profiles/ 找到配置文件
base64 -i profile.mobileprovision | pbcopy
```

#### 3. 在 GitHub 设置 Secrets

- `CERTIFICATE_BASE64`: 证书的 Base64 编码
- `CERTIFICATE_PASSWORD`: 证书密码
- `PROVISIONING_PROFILE_BASE64`: 配置文件的 Base64 编码
- `KEYCHAIN_PASSWORD`: 临时 Keychain 密码（自定义）

#### 4. 创建 ExportOptions.plist

在 `ios/ExportOptions.plist`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>method</key>
    <string>app-store</string>
    <key>teamID</key>
    <string>YOUR_TEAM_ID</string>
    <key>uploadBitcode</key>
    <false/>
    <key>compileBitcode</key>
    <false/>
    <key>uploadSymbols</key>
    <true/>
</dict>
</plist>
```

## 🎯 高级功能

### 缓存优化

```yaml
- name: 设置 Flutter 缓存
  uses: subosito/flutter-action@v2
  with:
    flutter-version: '3.16.0'
    cache: true

- name: 缓存 Pub 依赖
  uses: actions/cache@v3
  with:
    path: |
      ~/.pub-cache
      ${{ github.workspace }}/.dart_tool
    key: ${{ runner.os }}-pub-${{ hashFiles('**/pubspec.lock') }}
    restore-keys: |
      ${{ runner.os }}-pub-

- name: 缓存 Gradle
  uses: actions/cache@v3
  with:
    path: |
      ~/.gradle/caches
      ~/.gradle/wrapper
    key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
    restore-keys: |
      ${{ runner.os }}-gradle-
```

### 条件执行

```yaml
# 只在主分支运行发布构建
- name: 构建生产版本
  if: github.ref == 'refs/heads/main'
  run: flutter build apk --release

# 只在 PR 时运行测试
- name: 运行测试
  if: github.event_name == 'pull_request'
  run: flutter test

# 只在标签推送时创建 Release
- name: 创建 Release
  if: startsWith(github.ref, 'refs/tags/v')
  uses: softprops/action-gh-release@v1
```

### 并行任务

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
      - run: flutter test

  analyze:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
      - run: flutter analyze

  build:
    needs: [test, analyze]  # 等待测试和分析完成
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
      - run: flutter build apk
```

### 自动版本号

```yaml
- name: 获取版本号
  id: version
  run: |
    VERSION=$(grep 'version:' pubspec.yaml | sed 's/version: //')
    echo "version=$VERSION" >> $GITHUB_OUTPUT
    echo "build_number=${{ github.run_number }}" >> $GITHUB_OUTPUT

- name: 构建带版本号的 APK
  run: |
    flutter build apk --release \
      --build-name=${{ steps.version.outputs.version }} \
      --build-number=${{ steps.version.outputs.build_number }}
```

### 测试覆盖率报告

```yaml
- name: 运行测试并生成覆盖率
  run: flutter test --coverage

- name: 上传到 Codecov
  uses: codecov/codecov-action@v3
  with:
    files: ./coverage/lcov.info
    flags: unittests
    name: codecov-umbrella
```

## 🚀 自动部署

### 部署到 Google Play

```yaml
- name: 部署到 Google Play
  uses: r0adkll/upload-google-play@v1
  with:
    serviceAccountJsonPlainText: ${{ secrets.PLAY_SERVICE_ACCOUNT_JSON }}
    packageName: com.example.app
    releaseFiles: build/app/outputs/bundle/release/app-release.aab
    track: internal
    status: completed
```

需要的 Secret:
- `PLAY_SERVICE_ACCOUNT_JSON`: Google Play 服务账号 JSON

### 部署到 Firebase App Distribution

```yaml
- name: 部署到 Firebase
  uses: wzieba/Firebase-Distribution-Github-Action@v1
  with:
    appId: ${{ secrets.FIREBASE_APP_ID }}
    serviceCredentialsFileContent: ${{ secrets.FIREBASE_SERVICE_ACCOUNT }}
    groups: testers
    file: build/app/outputs/flutter-apk/app-release.apk
```

### 部署 Web 到 Firebase Hosting

```yaml
- name: 构建 Web
  run: flutter build web --release

- name: 部署到 Firebase Hosting
  uses: FirebaseExtended/action-hosting-deploy@v0
  with:
    repoToken: ${{ secrets.GITHUB_TOKEN }}
    firebaseServiceAccount: ${{ secrets.FIREBASE_SERVICE_ACCOUNT }}
    channelId: live
    projectId: your-project-id
```

### 部署到 App Store Connect

```yaml
- name: 上传到 App Store Connect
  run: |
    xcrun altool --upload-app \
      --type ios \
      --file "ios/build/Runner.ipa" \
      --username "${{ secrets.APPLE_ID }}" \
      --password "${{ secrets.APPLE_APP_SPECIFIC_PASSWORD }}"
```

## 📊 通知和报告

### Slack 通知

```yaml
- name: 发送 Slack 通知
  if: always()
  uses: 8398a7/action-slack@v3
  with:
    status: ${{ job.status }}
    text: '构建 ${{ job.status }}'
    webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

### 邮件通知

```yaml
- name: 发送邮件通知
  if: failure()
  uses: dawidd6/action-send-mail@v3
  with:
    server_address: smtp.gmail.com
    server_port: 465
    username: ${{ secrets.EMAIL_USERNAME }}
    password: ${{ secrets.EMAIL_PASSWORD }}
    subject: 'CI 构建失败: ${{ github.repository }}'
    to: your-email@example.com
    from: CI Bot
    body: '构建失败，请检查: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}'
```

## 💡 最佳实践

### 1. 优化构建时间

```yaml
# 使用缓存
- uses: subosito/flutter-action@v2
  with:
    cache: true

# 只在需要时运行完整构建
- name: 构建
  if: github.event_name == 'push' && github.ref == 'refs/heads/main'
  run: flutter build apk
```

### 2. 分离开发和生产环境

```yaml
jobs:
  build-dev:
    if: github.ref != 'refs/heads/main'
    steps:
      - run: flutter build apk --debug
  
  build-prod:
    if: github.ref == 'refs/heads/main'
    steps:
      - run: flutter build apk --release
```

### 3. 使用可复用工作流

创建 `.github/workflows/reusable-build.yml`：

```yaml
name: Reusable Build

on:
  workflow_call:
    inputs:
      platform:
        required: true
        type: string

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
      - run: flutter build ${{ inputs.platform }}
```

使用：

```yaml
jobs:
  android:
    uses: ./.github/workflows/reusable-build.yml
    with:
      platform: apk
```

### 4. 安全检查

```yaml
- name: 检查依赖漏洞
  run: |
    flutter pub outdated
    dart pub audit

- name: 代码安全扫描
  uses: github/codeql-action/analyze@v2
```

## 🐛 常见问题

### 问题 1: Flutter 版本不匹配

**解决方案**：明确指定 Flutter 版本

```yaml
- uses: subosito/flutter-action@v2
  with:
    flutter-version: '3.16.0'  # 使用具体版本号
    channel: 'stable'
```

### 问题 2: iOS 构建失败 - 证书问题

**解决方案**：确保证书和配置文件正确导入

```yaml
# 检查证书是否正确导入
- name: 列出证书
  run: security find-identity -v -p codesigning
```

### 问题 3: Android 签名失败

**解决方案**：确认 key.properties 文件创建正确

```yaml
- name: 验证签名配置
  run: |
    cat android/key.properties
    ls -la android/app/keystore.jks
```

### 问题 4: 构建超时

**解决方案**：增加超时时间并优化构建

```yaml
jobs:
  build:
    timeout-minutes: 60  # 默认是 360 分钟
    steps:
      # 使用缓存加速
      - uses: actions/cache@v3
```

### 问题 5: macOS 构建消耗太多分钟数

**解决方案**：只在必要时构建 iOS

```yaml
jobs:
  build-ios:
    if: github.ref == 'refs/heads/main' || startsWith(github.ref, 'refs/tags/')
    runs-on: macos-latest
```

## 📚 更多资源

- [GitHub Actions 官方文档](https://docs.github.com/actions)
- [Flutter Action](https://github.com/subosito/flutter-action)
- [Flutter 官方 CI/CD 文档](https://docs.flutter.dev/deployment/cd)
- [GitHub Actions Marketplace](https://github.com/marketplace?type=actions)

## 💰 费用优化

### 免费额度

- **公开仓库**：无限免费
- **私有仓库**：每月 2000 分钟
- **注意**：macOS 消耗 10 倍，Windows 消耗 2 倍

### 节省策略

1. **优化触发条件**：不要每次提交都运行完整构建
2. **使用缓存**：减少依赖下载时间
3. **并行构建**：利用矩阵策略
4. **条件构建**：只在必要时构建特定平台
5. **自托管 Runner**：对于频繁构建，考虑自托管

```yaml
# 示例：优化的触发配置
on:
  push:
    branches: [ main ]
    paths:
      - 'lib/**'
      - 'android/**'
      - 'ios/**'
      - 'pubspec.yaml'
  pull_request:
    branches: [ main ]
```

---

**提示**：这份文档会随着 GitHub Actions 和 Flutter 的更新而更新。建议定期查看官方文档获取最新信息。
