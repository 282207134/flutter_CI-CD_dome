# Cirrus CI - Flutter CI/CD 指南

## 📖 简介

Cirrus CI 是面向开源项目的云端 CI/CD 服务，对公共仓库完全免费，支持 Linux、macOS、Windows、FreeBSD 等多种环境。它以灵活配置、快速启动和多云支持著称，尤其适合需要同时构建多平台的 Flutter 项目。

### 优势

- ✅ **开源仓库无限免费**
- ✅ **多平台支持**：Linux、macOS、Windows、FreeBSD、ARM 等
- ✅ **灵活配置**：使用 `.cirrus.yml` 定义任务
- ✅ **按需付费**：私有项目按分钟计费
- ✅ **内置缓存与依赖管理**
- ✅ **支持自托管 runner（Cirrus Workers）**

### 适用场景

- 开源 Flutter 项目，需要免费 CI
- 需要构建多平台（尤其是桌面）
- 想要在 macOS 或 Windows 上构建但又不想付费
- 希望快速启动、灵活配置

## 🏗 架构概览

Cirrus CI 通过 `.cirrus.yml` 文件描述任务，每个任务称为 `task`。可以在单个文件中定义多个任务，分别运行在不同平台上。

关键概念：

- `task`: 一个构建任务，可指定运行环境、命令、缓存等
- `matrix`: 通过矩阵一次定义多个任务（例如不同 Flutter 版本）
- `cache`: 缓存依赖
- `artifacts`: 上传构建结果
- `env`: 环境变量

## 🚀 快速开始

### Step 1: 启用 Cirrus CI

1. 登录 [Cirrus CI](https://cirrus-ci.com)
2. 使用 GitHub / GitLab / Gitee 登录
3. 授权访问仓库
4. 在项目设置中 **Enable** Cirrus CI

### Step 2: 创建 `.cirrus.yml`

```yaml
task:
  name: Flutter CI
  container:
    image: cirrusci/flutter:latest
  env:
    CIRRUS_CLONE_DEPTH: 1
  setup_script:
    - flutter doctor
  script:
    - flutter pub get
    - flutter test
    - flutter build apk --debug
```

### Step 3: 提交并推送

```bash
git add .cirrus.yml
git commit -m "添加 Cirrus CI"
git push
```

Cirrus CI 会自动检测配置并开始构建。

## 📋 配置详解

### 1. 多任务示例

```yaml
free_task:
  only_if: $CIRRUS_REPO_OWNER == 'your-org'
  container:
    image: cirrusci/flutter:stable
  setup_script:
    - flutter pub get

linux_task:
  name: Linux APK
  container:
    image: cirrusci/flutter:stable
  script:
    - flutter build apk --release
  artifacts:
    apk:
      path: build/app/outputs/flutter-apk/app-release.apk
      type: file
  depends_on:
    - free_task
  only_if: $CIRRUS_BRANCH == 'main'

mac_task:
  name: iOS Build
  osx_instance:
    image: ghcr.io/cirruslabs/macos-ventura-base:latest
  env:
    FLUTTER_VERSION: "3.16.0"
    APPLE_ID: ENCRYPTED[base64:...]
  script:
    - git clone https://github.com/flutter/flutter.git -b $FLUTTER_VERSION
    - export PATH="$PWD/flutter/bin:$PATH"
    - flutter pub get
    - flutter build ios --release --no-codesign
```

### 2. 矩阵构建

```yaml
task:
  name: Flutter ${{ matrix.flutter }} on ${{ matrix.os }}
  matrix:
    flutter: [stable, beta]
    os: [linux, mac]
  container:
    image: cirrusci/flutter:${{ matrix.flutter }}
  osx_instance:
    image: ghcr.io/cirruslabs/macos-ventura-base:latest
  only_if: ${{ matrix.os == 'mac' }}
  script:
    - flutter --version
    - flutter test
```

### 3. 并行任务

```yaml
linux_task:
  container:
    image: cirrusci/flutter:stable
  script:
    - flutter test test/linux_test.dart

web_task:
  container:
    image: cirrusci/flutter:stable
  script:
    - flutter build web --release

windows_task:
  windows_container:
    image: cirrusci/flutter:windows
  script:
    - flutter build windows --release
```

## 🔐 环境变量与密钥

Cirrus 使用 **Encrypted Environment Variables** 保护敏感信息。

### 设置步骤
1. 打开项目的 Cirrus CI 页面
2. 点击右上角 ⚙️ -> **Repository Settings**
3. 在 **Environment Variables** 中添加变量
4. 选择 **Keep secret** 并输入值

常用变量：
- `ANDROID_KEYSTORE_BASE64`
- `ANDROID_KEYSTORE_PASSWORD`
- `ANDROID_KEY_PASSWORD`
- `ANDROID_KEY_ALIAS`
- `APPLE_ID`
- `APPLE_APP_SPECIFIC_PASSWORD`
- `FIREBASE_TOKEN`

### 在 `.cirrus.yml` 中使用

```yaml
env:
  ANDROID_KEYSTORE_BASE64: ENCRYPTED[base64:...]  # 自动替换为密文
```

### 解码 keystore

```yaml
script:
  - echo "$ANDROID_KEYSTORE_BASE64" | base64 -d > android/app/keystore.jks
  - |
    cat > android/key.properties << EOF
    storePassword=$ANDROID_KEYSTORE_PASSWORD
    keyPassword=$ANDROID_KEY_PASSWORD
    keyAlias=$ANDROID_KEY_ALIAS
    storeFile=keystore.jks
    EOF
```

## 🛠 构建示例

### Android：APK + App Bundle

```yaml
android_release_task:
  container:
    image: cirrusci/flutter:stable
  env:
    GRADLE_USER_HOME: /tmp/gradle
  script:
    - flutter pub get
    - flutter build apk --release --dart-define=FLAVOR=prod
    - flutter build appbundle --release
  cache:
    gradle:
      folder: /tmp/gradle
  artifacts:
    apk:
      path: build/app/outputs/flutter-apk/app-release.apk
    aab:
      path: build/app/outputs/bundle/release/app-release.aab
```

### iOS：IPA

```yaml
ios_task:
  osx_instance:
    image: ghcr.io/cirruslabs/macos-ventura-base:latest
  env:
    FLUTTER_VERSION: 3.16.0
  setup_script:
    - git clone https://github.com/flutter/flutter.git -b $FLUTTER_VERSION
    - export PATH="$PWD/flutter/bin:$PATH"
    - flutter doctor
  script:
    - flutter pub get
    - cd ios && pod install && cd ..
    - flutter build ipa --release --export-options-plist=ios/ExportOptions.plist
  artifacts:
    ipa:
      path: build/ios/ipa/*.ipa
```

### Web 构建 + 部署

```yaml
web_task:
  container:
    image: cirrusci/flutter:stable
  script:
    - flutter channel stable
    - flutter config --enable-web
    - flutter build web --release --base-href=/
    - tar -czf web.tar.gz build/web
  artifacts:
    web:
      path: web.tar.gz
```

### 桌面平台

```yaml
linux_desktop_task:
  container:
    image: cirrusci/flutter:stable
  setup_script:
    - apt-get update
    - apt-get install -y clang cmake ninja-build pkg-config libgtk-3-dev liblzma-dev
    - flutter config --enable-linux-desktop
  script:
    - flutter build linux --release
  artifacts:
    linux_build:
      path: build/linux/x64/release/bundle

windows_desktop_task:
  windows_container:
    image: cirrusci/flutter:windows
  script:
    - flutter config --enable-windows-desktop
    - flutter build windows --release
  artifacts:
    windows_build:
      path: build/windows/runner/Release
```

## 📦 缓存

Cirrus 自带缓存机制，通过 `cache` 字段配置。

```yaml
cache:
  pub_cache:
    folder: ~/.pub-cache
  gradle_cache:
    folder: ~/.gradle
  pods_cache:
    folder: ios/Pods
    fingerprint_script: cat ios/Podfile.lock
```

### 缓存命中策略
- `folder`: 缓存目录
- `fingerprint_script`: 输出内容作为缓存 key，可自定义
- `repopulate_from`: 指定依赖缓存

## 🚀 部署

### Google Play（Fastlane）

```yaml
deploy_google_play_task:
  container:
    image: cirrusci/flutter:stable
  env:
    PLAY_SERVICE_JSON: ENCRYPTED[base64:...]
  script:
    - python3 -m pip install --upgrade google-api-python-client google-auth google-auth-httplib2 google-auth-oauthlib
    - echo "$PLAY_SERVICE_JSON" > play-account.json
    - bundle exec fastlane supply --track internal --json_key play-account.json --aab build/app/outputs/bundle/release/app-release.aab
  depends_on:
    - android_release_task
```

### Firebase App Distribution

```yaml
deploy_firebase_task:
  container:
    image: cirrusci/flutter:stable
  env:
    FIREBASE_TOKEN: ENCRYPTED[base64:...]
  script:
    - curl -sL https://firebase.tools | bash
    - firebase appdistribution:distribute build/app/outputs/flutter-apk/app-release.apk --app $FIREBASE_APP_ID --groups testers
  depends_on:
    - android_release_task
```

### App Store Connect

```yaml
app_store_task:
  osx_instance:
    image: ghcr.io/cirruslabs/macos-ventura-base:latest
  env:
    APP_STORE_CONNECT_API_KEY: ENCRYPTED[base64:...]
  script:
    - xcrun altool --upload-app --type ios --file build/ios/ipa/App.ipa --apiKey $APP_STORE_CONNECT_KEY --apiIssuer $APP_STORE_CONNECT_ISSUER
  depends_on:
    - ios_task
```

## 📊 通知与报告

### Slack 通知

```yaml
notify_task:
  container:
    image: curlimages/curl
  script:
    - >
      curl -X POST -H 'Content-type: application/json' \
      --data '{"text":"Cirrus CI 构建完成: $CIRRUS_TASK_NAME => $CIRRUS_TASK_STATUS"}' \
      $SLACK_WEBHOOK_URL
  allow_failures: true
  always_run: true
```

### GitHub 状态报告

Cirrus 会自动将状态回传到 GitHub/GitLab 的 PR/MR 页面。

### 测试覆盖率

```yaml
test_task:
  script:
    - flutter test --coverage
    - genhtml coverage/lcov.info -o coverage/html
  artifacts:
    coverage:
      path: coverage/html
```

## 🐛 常见问题

| 问题 | 解决 |
| ---- | ---- |
| 构建队列长 | Cirrus 免费计划共享资源，考虑付费 / 自托管 Worker |
| macOS 构建失败 | 确保使用 `osx_instance` 并安装依赖 |
| 缓存不生效 | 添加 `fingerprint_script` 或确保路径正确 |
| 任务过多耗时长 | 使用 `only_if` 控制执行条件 |
| 环境变量不生效 | 检查是否添加到仓库设置并加密 |

## 💰 成本 & 优化

### 免费额度
- 开源仓库：无限免费
- 私有仓库：按使用计费（按分钟）

### 节省技巧
1. 使用 `only_if` / `skip` 控制运行条件
2. 使用缓存减少重复安装依赖
3. 拆分任务并行执行
4. 多项目共享 `.cirrus.yml` 模板
5. 对私有项目使用自托管 Worker

```yaml
linux_task:
  only_if: $CIRRUS_BRANCH == 'main' || $CIRRUS_TAG != ''
```

## 🧰 自托管 Cirrus Worker

### 为什么使用
- 完全控制硬件
- 可访问私有网络资源
- 节省成本

### 安装步骤（Linux）

1. 下载 Worker
```bash
curl -o worker.tar.gz https://github.com/cirruslabs/cirrus-cli/releases/latest/download/linux_amd64.tar.gz
tar -xzf worker.tar.gz
sudo mv cirrus worker /usr/local/bin
```

2. 登录并注册
```bash
cirrus login
ec2-worker register --name flutter-runner
```

3. 运行 Worker
```bash
cirrus worker run --config worker.yaml
```

### `worker.yaml` 示例

```yaml
api:
  url: https://api.cirrus-ci.com

instance:
  image: ubuntu-2204
  cpu: 8
  memory: 16G
  disk: 100G
  docker:
    image: cirrusci/flutter:stable
```

## 📚 参考与资源

- [Cirrus CI 官方文档](https://cirrus-ci.org)
- [Flutter 官方 CI/CD 文档](https://docs.flutter.dev/deployment/cd)
- [Cirrus Docker 镜像](https://hub.docker.com/r/cirrusci/flutter)
- [Cirrus YAML 参考](https://cirrus-ci.org/guide/writing-tasks/)
- [Cirrus 缓存说明](https://cirrus-ci.org/guide/writing-tasks/#cache)

---

> **提示**：对于开源 Flutter 项目，Cirrus CI 是在 macOS/Windows 平台上构建的极佳免费方案。合理使用矩阵和缓存可以显著提升构建效率。
