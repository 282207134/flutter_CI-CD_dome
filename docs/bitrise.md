# Bitrise - Flutter CI/CD 完整指南

## 📖 简介

Bitrise 是专注于移动应用开发的云端 CI/CD 平台，提供可视化工作流编辑器和丰富的移动端集成。它特别适合需要简单配置、快速上手的 iOS 和 Android 项目。

### 优势

- ✅ **移动应用专属**：为 iOS/Android 优化的构建环境
- ✅ **可视化编辑器**：通过图形界面配置工作流
- ✅ **丰富的 Steps**：内置大量移动端集成步骤
- ✅ **免费套餐**：适合小型项目和个人开发者
- ✅ **自动代码签名**：简化 iOS 证书管理
- ✅ **内置测试设备云**：可选设备测试服务

### 适用场景

- 移动应用（iOS/Android）开发团队
- 需要快速上手、无需编写复杂配置
- 希望使用可视化界面管理 CI/CD
- 需要自动化 iOS 签名的团队

## 🚀 快速开始

### Step 1: 连接项目

1. 登录 [Bitrise](https://www.bitrise.io)
2. 点击 **Add new app**
3. 选择代码托管平台（GitHub/GitLab/Bitbucket）
4. 授权并选择仓库
5. Bitrise 自动检测项目配置（Flutter、Android、iOS）

### Step 2: 配置工作流

Bitrise 自动生成基础工作流：

- **Primary Workflow**: 测试 + 构建
- **Deploy Workflow**: 发布流程

你可以通过 **Workflow Editor** 可视化编辑，或使用 `bitrise.yml` 文件。

### Step 3: 添加环境变量

在 Workflow Editor 中，选择 **Secrets** 标签，添加：

- `ANDROID_KEYSTORE_URL`
- `ANDROID_KEYSTORE_PASSWORD`
- `ANDROID_KEY_PASSWORD`
- `ANDROID_KEY_ALIAS`

### Step 4: 运行构建

- 点击 **Start/Schedule Build**
- 或提交代码自动触发

## 📋 `bitrise.yml` 配置

虽然 Bitrise 提供可视化编辑器，但你也可以直接编写 `bitrise.yml` 文件（在项目根目录）。

### 基础配置

```yaml
format_version: "11"
default_step_lib_source: https://github.com/bitrise-io/bitrise-steplib.git

workflows:
  primary:
    steps:
      - activate-ssh-key@4:
          run_if: '{{getenv "SSH_RSA_PRIVATE_KEY" | ne ""}}'
      
      - git-clone@6: {}
      
      - flutter-installer@0:
          inputs:
            - version: stable
      
      - flutter-analyze@0: {}
      
      - flutter-test@0:
          inputs:
            - additional_params: --coverage
      
      - flutter-build@0:
          inputs:
            - platform: android
            - ios_output_type: archive
```

### Android 完整构建

```yaml
workflows:
  android_release:
    envs:
      - BUILD_TYPE: release
    steps:
      - git-clone@6: {}
      
      - flutter-installer@0:
          inputs:
            - version: 3.16.0
      
      - cache-pull@2: {}
      
      - flutter-analyze@0: {}
      
      - flutter-test@0: {}
      
      - android-build@1:
          inputs:
            - project_location: android
            - module: app
            - variant: Release
      
      - sign-apk@1:
          inputs:
            - apk_path: $BITRISE_APK_PATH
      
      - deploy-to-bitrise-io@2: {}
      
      - cache-push@2: {}
```

### iOS 完整构建

```yaml
workflows:
  ios_release:
    steps:
      - git-clone@6: {}
      
      - flutter-installer@0:
          inputs:
            - version: stable
      
      - certificate-and-profile-installer@1: {}
      
      - flutter-build@0:
          inputs:
            - platform: ios
            - ios_output_type: archive
            - ios_scheme: Runner
      
      - xcode-archive@4:
          inputs:
            - project_path: ios/Runner.xcworkspace
            - scheme: Runner
            - export_method: app-store
      
      - deploy-to-itunesconnect-deliver@2:
          inputs:
            - itunescon_user: $APPLE_ID
            - password: $APPLE_APP_SPECIFIC_PASSWORD
      
      - deploy-to-bitrise-io@2: {}
```

### 多平台构建

```yaml
workflows:
  multi_platform:
    steps:
      - git-clone@6: {}
      
      - flutter-installer@0:
          inputs:
            - version: stable
      
      - flutter-analyze@0: {}
      
      - flutter-test@0: {}
      
      - flutter-build@0:
          inputs:
            - platform: both
            - ios_output_type: archive
      
      - android-build@1:
          inputs:
            - project_location: android
            - variant: Release
      
      - xcode-archive@4:
          inputs:
            - project_path: ios/Runner.xcworkspace
            - scheme: Runner
      
      - deploy-to-bitrise-io@2: {}
```

## 🔧 可视化工作流编辑器

### 常用 Steps

#### 1. Flutter Installer
安装指定版本的 Flutter SDK

```yaml
- flutter-installer@0:
    inputs:
      - version: stable  # 或具体版本号如 3.16.0
      - installation_bundle_url: ""
```

#### 2. Flutter Analyze
运行代码分析

```yaml
- flutter-analyze@0:
    inputs:
      - additional_params: --no-fatal-infos
```

#### 3. Flutter Test
运行测试

```yaml
- flutter-test@0:
    inputs:
      - additional_params: --coverage
      - project_location: $BITRISE_SOURCE_DIR
```

#### 4. Flutter Build
构建应用

```yaml
- flutter-build@0:
    inputs:
      - platform: both  # android, ios, both, web
      - ios_output_type: archive  # app, archive
      - android_output_type: appbundle  # apk, appbundle
```

#### 5. Android Sign
签名 Android 应用

```yaml
- sign-apk@1:
    inputs:
      - apk_path: $BITRISE_APK_PATH
      - keystore_url: $BITRISEIO_ANDROID_KEYSTORE_URL
      - keystore_password: $BITRISEIO_ANDROID_KEYSTORE_PASSWORD
      - keystore_alias: $BITRISEIO_ANDROID_KEYSTORE_ALIAS
      - private_key_password: $BITRISEIO_ANDROID_KEYSTORE_PRIVATE_KEY_PASSWORD
```

#### 6. iOS Certificate Installer
自动安装 iOS 证书和配置文件

```yaml
- certificate-and-profile-installer@1: {}
```

#### 7. Deploy to Bitrise.io
上传构建产物

```yaml
- deploy-to-bitrise-io@2:
    inputs:
      - notify_user_groups: none
      - is_enable_public_page: false
```

#### 8. Deploy to Google Play
发布到 Google Play

```yaml
- google-play-deploy@3:
    inputs:
      - service_account_json_key_path: $BITRISEIO_SERVICE_ACCOUNT_JSON_KEY_URL
      - package_name: com.example.app
      - track: internal
```

#### 9. Deploy to App Store Connect
发布到 App Store

```yaml
- deploy-to-itunesconnect-deliver@2:
    inputs:
      - itunescon_user: $APPLE_ID
      - password: $APPLE_APP_SPECIFIC_PASSWORD
      - app_id: $APPLE_APP_ID
```

## 🔐 密钥和证书管理

### Android 签名

#### 1. 上传 Keystore

在 **Workflow Editor > Code Signing** 中：

- 上传 `keystore.jks` 文件
- Bitrise 自动创建环境变量：
  - `BITRISEIO_ANDROID_KEYSTORE_URL`
  - `BITRISEIO_ANDROID_KEYSTORE_PASSWORD`
  - `BITRISEIO_ANDROID_KEYSTORE_ALIAS`
  - `BITRISEIO_ANDROID_KEYSTORE_PRIVATE_KEY_PASSWORD`

#### 2. 在工作流中使用

添加 **Android Sign** Step，它会自动使用上述变量。

### iOS 证书和配置文件

#### 方式 1: 手动上传

1. 在 **Code Signing** 中上传 `.p12` 证书
2. 上传 `.mobileprovision` 文件
3. 添加 **Certificate and Profile Installer** Step

#### 方式 2: 自动管理（推荐）

使用 **iOS Auto Provision with App Store Connect API** Step：

```yaml
- ios-auto-provision-appstoreconnect@0:
    inputs:
      - distribution_type: app-store
      - apple_developer_team_id: $APPLE_TEAM_ID
      - app_store_connect_api_key_path: $BITRISEIO_APP_STORE_CONNECT_API_KEY_URL
```

需要在 App Store Connect 创建 API Key 并上传。

## 🎯 高级功能

### 缓存

```yaml
- cache-pull@2: {}

# 构建步骤...

- cache-push@2:
    inputs:
      - cache_paths: |-
          $HOME/.pub-cache
          $HOME/.gradle
          ios/Pods
```

### 并行工作流

Bitrise 不直接支持并行，但可以通过 **Workflow Chaining** 实现：

```yaml
workflows:
  test:
    steps:
      - flutter-test@0: {}
  
  build_android:
    after_run:
      - test
    steps:
      - flutter-build@0:
          inputs:
            - platform: android
  
  build_ios:
    after_run:
      - test
    steps:
      - flutter-build@0:
          inputs:
            - platform: ios
```

### 条件执行

```yaml
- flutter-build@0:
    run_if: '{{enveq "BITRISE_GIT_BRANCH" "main"}}'
    inputs:
      - platform: android
```

### 触发条件

在 **Triggers** 中设置：

- Push 触发：指定分支
- Pull Request 触发
- Tag 触发

也可以在 `bitrise.yml` 中配置：

```yaml
trigger_map:
  - push_branch: main
    workflow: deploy
  - push_branch: develop
    workflow: primary
  - pull_request_source_branch: "*"
    workflow: primary
  - tag: "*"
    workflow: release
```

### 环境变量

全局变量：

```yaml
app:
  envs:
    - FLUTTER_VERSION: 3.16.0
    - BUILD_NUMBER: $BITRISE_BUILD_NUMBER
```

工作流级别：

```yaml
workflows:
  release:
    envs:
      - BUILD_TYPE: release
      - FLAVOR: production
```

### 自定义脚本

```yaml
- script@1:
    title: 自定义构建脚本
    inputs:
      - content: |
          #!/bin/bash
          set -ex
          flutter pub get
          flutter build apk --release --flavor production
```

## 🚀 部署

### Firebase App Distribution

```yaml
- firebase-app-distribution@0:
    inputs:
      - app: $FIREBASE_APP_ID
      - firebase_token: $FIREBASE_TOKEN
      - app_path: $BITRISE_APK_PATH
      - groups: testers
      - release_notes: Automatic build from Bitrise
```

### TestFlight

```yaml
- deploy-to-itunesconnect-deliver@2:
    inputs:
      - itunescon_user: $APPLE_ID
      - password: $APPLE_APP_SPECIFIC_PASSWORD
      - app_id: $APPLE_APP_ID
```

### 自定义部署

```yaml
- script@1:
    title: 部署到自定义服务器
    inputs:
      - content: |
          #!/bin/bash
          scp $BITRISE_APK_PATH user@server:/path/to/deploy
```

## 📊 通知和报告

### Slack 通知

添加 **Slack** Step：

```yaml
- slack@3:
    inputs:
      - webhook_url: $SLACK_WEBHOOK_URL
      - channel: "#ci-cd"
      - from_username: Bitrise
      - message: Build $BITRISE_BUILD_NUMBER finished
      - emoji: ":rocket:"
```

### Email 通知

添加 **Send Email** Step：

```yaml
- email-with-mailgun@1:
    inputs:
      - mailgun_api_key: $MAILGUN_API_KEY
      - mailgun_domain: $MAILGUN_DOMAIN
      - send_to: team@example.com
      - subject: Bitrise Build $BITRISE_BUILD_NUMBER
      - message: Build finished with status $BITRISE_BUILD_STATUS
```

### 测试报告

```yaml
- flutter-test@0:
    inputs:
      - additional_params: --machine > test-results.json

- deploy-to-bitrise-io@2:
    inputs:
      - deploy_path: test-results.json
```

## 💡 最佳实践

### 1. 使用 Stack 选择合适的构建环境

在 **Stack** 中选择：

- `Xcode 15.x on macOS 13.x`: iOS 构建
- `Linux/Android & Docker`: Android 和 Web 构建

### 2. 使用 Secret 环境变量

在 **Secrets** 中添加敏感信息，勾选 **Expose for Pull Requests** 谨慎启用。

### 3. 优化缓存

```yaml
- cache-push@2:
    inputs:
      - cache_paths: |-
          ~/.pub-cache
          ~/.gradle/caches
          ios/Pods
      - ignore_check_on_paths: |-
          ~/.pub-cache/bin
```

### 4. 版本号管理

```yaml
- script@1:
    title: 设置版本号
    inputs:
      - content: |
          #!/bin/bash
          BUILD_NUMBER=$BITRISE_BUILD_NUMBER
          VERSION=$(grep 'version:' pubspec.yaml | cut -d ' ' -f2 | cut -d '+' -f1)
          echo "Building version $VERSION+$BUILD_NUMBER"
          envman add --key VERSION_NAME --value $VERSION
          envman add --key VERSION_CODE --value $BUILD_NUMBER
```

### 5. 工作流复用

```yaml
workflows:
  _setup:
    steps:
      - git-clone@6: {}
      - flutter-installer@0: {}
      - cache-pull@2: {}
  
  test:
    before_run:
      - _setup
    steps:
      - flutter-test@0: {}
  
  build_android:
    before_run:
      - _setup
    after_run:
      - _deploy
    steps:
      - flutter-build@0:
          inputs:
            - platform: android
```

## 🐛 常见问题

### 问题 1: Flutter 版本不匹配

**解决方案**：在 Flutter Installer Step 中指定版本

```yaml
- flutter-installer@0:
    inputs:
      - version: 3.16.0
```

### 问题 2: iOS 签名失败

**解决方案**：确保证书和配置文件已上传，并使用 Certificate and Profile Installer Step

### 问题 3: Android 签名失败

**解决方案**：检查 Keystore 文件和密码是否正确

### 问题 4: 构建超时

**解决方案**：优化缓存，或升级到付费计划（更长超时时间）

### 问题 5: 依赖下载慢

**解决方案**：使用缓存，或配置国内镜像

```yaml
- script@1:
    inputs:
      - content: |
          export PUB_HOSTED_URL=https://pub.flutter-io.cn
          export FLUTTER_STORAGE_BASE_URL=https://storage.flutter-io.cn
          flutter pub get
```

## 💰 费用优化

### 免费计划
- 每月 200 构建分钟
- 1 个并发构建
- 适合小型项目

### 优化策略

1. **合并工作流**：减少重复步骤
2. **使用缓存**：加速依赖安装
3. **条件构建**：只在需要时运行
4. **优化触发条件**：避免不必要的构建

```yaml
trigger_map:
  - push_branch: main
    workflow: deploy
  - push_branch: develop
    workflow: test
  - pull_request_source_branch: "*"
    workflow: test
```

## 📚 更多资源

- [Bitrise 官方文档](https://devcenter.bitrise.io)
- [Bitrise Step Library](https://www.bitrise.io/integrations/steps)
- [Flutter on Bitrise 教程](https://devcenter.bitrise.io/en/getting-started/getting-started-with-flutter-apps.html)
- [Bitrise YAML 参考](https://devcenter.bitrise.io/en/references/bitrise-yml-reference.html)

---

**提示**：Bitrise 的可视化工作流编辑器非常适合快速上手，特别是对移动应用开发团队。合理使用 Steps 和缓存可以获得高效的构建体验。
