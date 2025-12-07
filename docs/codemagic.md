# Codemagic - Flutter 专用 CI/CD 指南

## 📖 简介

Codemagic 是由 Flutter 生态公司 Codemagic 提供的专门面向移动应用（尤其是 Flutter） 的 CI/CD 服务。它提供 UI + YAML 双模式配置，支持从构建、测试到发布全流程自动化。

### 平台优势

- ✅ **Flutter 原生支持**：几乎所有 Flutter 构建场景都开箱即用
- ✅ **免费额度友好**：每月 500 分钟 + 2 个并发任务
- ✅ **图形化界面**：零代码即可完成基本配置
- ✅ **多平台支持**：Android、iOS、Web、桌面
- ✅ **内置发布**：可直接推送到 Google Play、App Store、TestFlight、Firebase App Distribution 等

### 适用场景

- 想要最快开始 Flutter CI/CD 的个人或小团队
- 需要同时构建 Android 和 iOS 的跨平台应用
- 需要简单配置和内置集成（如测试、发布、通知）

## 🚀 快速开始

### Step 1：连接代码仓库
1. 登录 [Codemagic](https://codemagic.io)
2. 选择 **Add Application**
3. 连接你的 GitHub/GitLab/Bitbucket 仓库
4. 授权 Codemagic 访问代码

### Step 2：选择构建方式
- **UI 模式**：适合初学者，只需勾选选项
- **`codemagic.yaml` 模式**：适合需要版本控制和高级配置

本文重点介绍 `codemagic.yaml` 模式（推荐）。

### Step 3：创建 `codemagic.yaml`

在项目根目录添加 `codemagic.yaml` 文件。

```yaml
workflows:
  flutter_deploy:
    name: Flutter Release Workflow
    max_build_duration: 60
    environment:
      flutter: stable
      xcode: latest
      cocoapods: default
      vars:
        APP_PACKAGE_NAME: "com.example.app"
    triggering:
      events:
        - push
        - pull_request
      branch_patterns:
        - pattern: main
          include: true
    scripts:
      - name: 安装依赖
        script: flutter pub get
      - name: 运行测试
        script: flutter test --coverage
      - name: 构建 APK
        script: flutter build apk --release
      - name: 构建 App Bundle
        script: flutter build appbundle --release
      - name: 构建 iOS
        script: flutter build ipa --release
    artifacts:
      - build/**/outputs/**/*
      - build/**/ipa/*.ipa
    publishing:
      email:
        recipients:
          - team@example.com
```

### Step 4：提交并触发
```bash
git add codemagic.yaml
git commit -m "添加 Codemagic Workflow"
git push
```
Codemagic 将自动检测到配置并开始构建。

## ⚙️ YAML 结构详解

| 字段 | 说明 |
| ---- | ---- |
| `workflows` | 定义一个或多个工作流 |
| `environment` | 设置 Flutter 版本、Xcode 版本、环境变量 |
| `triggering` | 触发构建的分支、事件等 |
| `scripts` | 构建、测试、打包的命令序列 |
| `artifacts` | 指定输出文件（构建结果） |
| `publishing` | 构建完成后的发布/通知 |

### 多平台示例

```yaml
workflows:
  multi_platform:
    environment:
      flutter: 3.16.0
      node: latest
      vars:
        ANDROID_KEYSTORE: Encrypted(…)
    scripts:
      - name: 启用桌面平台
        script: |
          flutter config --enable-web
          flutter config --enable-macos-desktop
          flutter config --enable-linux-desktop
          flutter config --enable-windows-desktop
          flutter pub get
      - name: Web 构建
        script: flutter build web --release
      - name: Android 构建
        script: |
          printf "%s" "$ANDROID_KEYSTORE" | base64 --decode > android/app/keystore.jks
          flutter build appbundle --release
      - name: iOS 构建
        script: flutter build ipa --release
      - name: macOS 构建
        script: flutter build macos --release
    artifacts:
      - build/**
```

### 拆分 Job 提升效率

```yaml
workflows:
  test_job:
    scripts:
      - script: flutter analyze
      - script: flutter test
  build_job:
    scripts:
      - script: flutter build appbundle
      - script: flutter build ipa
    cache:
      cache_paths:
        - ~/.pub-cache
        - android/.gradle
```

## 🔐 环境变量 & 密钥管理

Codemagic 提供两种安全方式：

1. **Environment variables**：普通变量
2. **Secure variables**：加密变量，使用 `Encrypted(...)` 语法

### 设置步骤
1. 打开 Codemagic 应用设置
2. 选择 **Environment variables**
3. 输入变量名、类型、值
4. 可选择「Secure」保护

### 常用变量
- `ANDROID_KEYSTORE`：Keystore 的 Base64 编码
- `KEYSTORE_PASSWORD`、`KEY_PASSWORD`、`KEY_ALIAS`
- `APP_STORE_CONNECT_ISSUER_ID`
- `APP_STORE_CONNECT_KEY_IDENTIFIER`
- `APP_STORE_CONNECT_PRIVATE_KEY`
- `FIREBASE_TOKEN`

### 示例：解码 keystore
```yaml
scripts:
  - name: 准备 Keystore
    script: |
      printf "%s" "$ANDROID_KEYSTORE" | base64 --decode > android/app/keystore.jks
      cat > android/key.properties <<'EOF'
      storePassword=$KEYSTORE_PASSWORD
      keyPassword=$KEY_PASSWORD
      keyAlias=$KEY_ALIAS
      storeFile=keystore.jks
      EOF
```

## 📦 构建与测试

### Android 构建
```yaml
- name: 构建 Debug APK
  script: flutter build apk --debug

- name: 构建 Release AAB
  script: flutter build appbundle --release --build-name=1.0.0 --build-number=$BUILD_NUMBER
```

### iOS 构建
```yaml
- name: 安装 CocoaPods
  script: |
    cd ios
    pod install

- name: 构建 iOS IPA
  script: flutter build ipa --release --export-options-plist=ios/ExportOptions.plist
```

### Web & 桌面
```yaml
- name: Web 构建
  script: flutter build web --release --base-href=/app/

- name: Windows 构建
  instance_type: windows_x2
  script: flutter build windows --release
```

### 测试与分析
```yaml
- name: Dart 静态分析
  script: flutter analyze

- name: 单元测试 + 覆盖率
  script: flutter test --coverage

- name: 集成测试
  script: flutter test integration_test
```

## 🚀 自动发布

### Google Play
```yaml
publishing:
  google_play:
    credentials: Encrypted(…)
    track: internal
    in_app_update_priority: 0
```

### App Store Connect
```yaml
publishing:
  app_store_connect:
    apple_id: $APPLE_ID
    issuer_id: $APP_STORE_CONNECT_ISSUER_ID
    key_id: $APP_STORE_CONNECT_KEY_IDENTIFIER
    private_key: Encrypted(…)
    submit_to_testflight: true
```

### Firebase App Distribution
```yaml
publishing:
  firebase:
    firebase_token: $FIREBASE_TOKEN
    android:
      app_id: 1:1234567890:android:abc
      testers: tester@example.com
    ios:
      app_id: 1:1234567890:ios:def
      groups: qa-team
```

### 自定义通知（Slack）
```yaml
publishing:
  slack:
    channel: '#ci-cd'
    notify_on_build_start: false
    notify:
      success: true
      failure: true
    webhook_url: Encrypted(…)
```

## 🧰 内置功能

### 缓存
```yaml
cache:
  cache_paths:
    - ~/.pub-cache
    - /Users/builder/Library/Caches/CocoaPods
    - android/.gradle
```

### 构建矩阵（不同 Flutter 版本）
```yaml
workflows:
  matrix_build:
    environment:
      groups:
        - flutter_versions
    scripts:
      - script: flutter --version

# Environment groups
# flutter_versions:
#   - flutter: beta
#   - flutter: stable
```

### 自托管构建机
- Codemagic 允许使用私有构建机（macOS/Linux/Windows）
- 适用于需要访问内网资源或自定义依赖

## 💰 费用 & 优化

| 项目 | 免费额度 |
| ---- | ---- |
| 构建分钟 | 500 分钟/月 |
| 并发构建 | 2 个 |
| macOS 构建 | 可使用（按分钟计费） |

**节省技巧**
1. 合理设置触发条件（仅 main 分支 / 仅标签）
2. 使用缓存减少构建时间
3. 只构建必要平台（例如 PR 只构建 Android）
4. 使用 `max_build_duration` 防止卡死消耗分钟数

```yaml
triggering:
  events:
    - push
  branch_patterns:
    - pattern: main
      include: true
      source: true
    - pattern: "v*"
      include: true
```

## 🐛 常见问题排查

| 问题 | 原因 | 解决 |
| ---- | ---- | ---- |
| 找不到 Flutter 命令 | 未安装 Flutter | 在 environment 设置 `flutter: stable` |
| iOS 构建失败 | 证书配置问题 | 检查 `Distribution certificate` + `Provisioning profile` |
| Android 签名失败 | Keystore 未解码/路径错误 | 确保 `key.properties` 和 `keystore.jks` 正确 |
| 构建超时 | 脚本卡死或命令耗时长 | 设置 `max_build_duration` 并拆分脚本 |
| 依赖下载慢 | 每次重新下载 | 使用 `cache_paths` |

## 📚 资源 & 链接

- [Codemagic 官网](https://codemagic.io)
- [Codemagic 文档](https://docs.codemagic.io)
- [codemagic.yaml 参考](https://docs.codemagic.io/yaml/yaml-getting-started/)
- [Flutter 官方 CI/CD 文档](https://docs.flutter.dev/deployment/cd)
- [Codemagic + Flutter 教程](https://blog.codemagic.io)

---

> **Tips**：Codemagic 与 Flutter 配合非常紧密，建议优先使用其内置模板快速开始，再根据团队需求迁移到 `codemagic.yaml` 以便版本控制。
