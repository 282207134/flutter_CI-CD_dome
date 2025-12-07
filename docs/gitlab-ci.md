# GitLab CI/CD - Flutter 完整指南

## 📖 简介

GitLab CI/CD 是 GitLab 内置的持续集成和持续部署工具，通过 `.gitlab-ci.yml` 文件配置。它与 GitLab 仓库无缝集成，是代码托管在 GitLab 的项目的首选 CI/CD 方案。

### 优势

- ✅ **无缝集成**：原生支持 GitLab，无需额外配置
- ✅ **免费额度**：每月 400 分钟（共享 Runner）
- ✅ **自托管 Runner**：可搭建私有 Runner 实现无限使用
- ✅ **强大的 Pipeline**：支持复杂的多阶段流水线
- ✅ **Docker 支持**：可使用任意 Docker 镜像作为构建环境
- ✅ **内置容器注册表**：方便存储 Docker 镜像

### 适用场景

- 代码托管在 GitLab 上的项目
- 需要复杂多阶段流水线的项目
- 希望自建 Runner 以节省成本
- 需要完全控制构建环境

## 🚀 快速开始

### Step 1: 创建配置文件

在项目根目录创建 `.gitlab-ci.yml`：

```yaml
image: cirrusci/flutter:stable

stages:
  - test
  - build

variables:
  GET_SOURCES_ATTEMPTS: 3

before_script:
  - flutter --version
  - flutter pub get

test:
  stage: test
  script:
    - flutter analyze
    - flutter test
  only:
    - merge_requests
    - main

build_apk:
  stage: build
  script:
    - flutter build apk --release
  artifacts:
    paths:
      - build/app/outputs/flutter-apk/app-release.apk
  only:
    - main
    - tags
```

### Step 2: 提交代码

```bash
git add .gitlab-ci.yml
git commit -m "添加 GitLab CI/CD 配置"
git push
```

### Step 3: 查看 Pipeline

访问 GitLab 项目页面的 **CI/CD > Pipelines** 查看构建进度。

## 📋 完整配置示例

### Android 完整构建

```yaml
image: cirrusci/flutter:stable

stages:
  - prepare
  - test
  - build
  - deploy

variables:
  ANDROID_COMPILE_SDK: "33"
  ANDROID_BUILD_TOOLS: "33.0.0"
  ANDROID_SDK_TOOLS: "9477386"

cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - .pub-cache/
    - .gradle/

before_script:
  - export PATH="$PATH:$HOME/.pub-cache/bin"
  - flutter --version
  - flutter doctor -v

flutter_test:
  stage: test
  script:
    - flutter pub get
    - flutter analyze
    - flutter test --coverage
  coverage: '/lines......: \d+\.\d+\%/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura.xml
  only:
    - merge_requests
    - main

build_apk_debug:
  stage: build
  script:
    - flutter pub get
    - flutter build apk --debug
  artifacts:
    paths:
      - build/app/outputs/flutter-apk/app-debug.apk
    expire_in: 1 week
  only:
    - merge_requests
    - develop

build_apk_release:
  stage: build
  script:
    - flutter pub get
    # 解码 keystore
    - echo "$ANDROID_KEYSTORE_BASE64" | base64 -d > android/app/keystore.jks
    # 创建 key.properties
    - |
      cat > android/key.properties << EOF
      storePassword=$ANDROID_KEYSTORE_PASSWORD
      keyPassword=$ANDROID_KEY_PASSWORD
      keyAlias=$ANDROID_KEY_ALIAS
      storeFile=keystore.jks
      EOF
    - flutter build apk --release
    - flutter build appbundle --release
  artifacts:
    paths:
      - build/app/outputs/flutter-apk/app-release.apk
      - build/app/outputs/bundle/release/app-release.aab
    expire_in: 30 days
  only:
    - main
    - tags

deploy_internal:
  stage: deploy
  script:
    - echo "部署到内部测试轨道"
    # 这里可以添加部署到 Google Play 的脚本
  dependencies:
    - build_apk_release
  only:
    - tags
  when: manual
```

### iOS 完整构建

```yaml
build_ios:
  stage: build
  tags:
    - macos  # 需要 macOS Runner
  before_script:
    - export PATH="$PATH:$HOME/flutter/bin"
    - flutter --version
    - flutter pub get
    - cd ios
    - pod install
    - cd ..
  script:
    # 配置证书和配置文件
    - echo "$IOS_CERTIFICATE_BASE64" | base64 -d > certificate.p12
    - echo "$IOS_PROVISIONING_PROFILE_BASE64" | base64 -d > profile.mobileprovision
    
    # 创建临时 keychain
    - security create-keychain -p "$KEYCHAIN_PASSWORD" build.keychain
    - security default-keychain -s build.keychain
    - security unlock-keychain -p "$KEYCHAIN_PASSWORD" build.keychain
    - security set-keychain-settings -t 3600 -l ~/Library/Keychains/build.keychain
    
    # 导入证书
    - security import certificate.p12 -k build.keychain -P "$IOS_CERTIFICATE_PASSWORD" -T /usr/bin/codesign
    - security set-key-partition-list -S apple-tool:,apple: -s -k "$KEYCHAIN_PASSWORD" build.keychain
    
    # 安装配置文件
    - mkdir -p ~/Library/MobileDevice/Provisioning\ Profiles
    - cp profile.mobileprovision ~/Library/MobileDevice/Provisioning\ Profiles/
    
    # 构建
    - flutter build ios --release --no-codesign
    - cd ios
    - xcodebuild -workspace Runner.xcworkspace -scheme Runner -configuration Release -archivePath $PWD/build/Runner.xcarchive archive
    - xcodebuild -exportArchive -archivePath $PWD/build/Runner.xcarchive -exportPath $PWD/build -exportOptionsPlist ExportOptions.plist
  artifacts:
    paths:
      - ios/build/*.ipa
    expire_in: 30 days
  only:
    - main
    - tags
```

### 多平台矩阵构建

```yaml
.build_template: &build_template
  stage: build
  script:
    - flutter pub get
    - flutter build $PLATFORM $BUILD_FLAGS
  artifacts:
    paths:
      - build/
    expire_in: 1 week

build_android:
  <<: *build_template
  variables:
    PLATFORM: apk
    BUILD_FLAGS: --release

build_web:
  <<: *build_template
  variables:
    PLATFORM: web
    BUILD_FLAGS: --release

build_linux:
  <<: *build_template
  image: cirrusci/flutter:stable
  before_script:
    - apt-get update
    - apt-get install -y clang cmake ninja-build pkg-config libgtk-3-dev liblzma-dev
    - flutter config --enable-linux-desktop
    - flutter pub get
  variables:
    PLATFORM: linux
    BUILD_FLAGS: --release

build_windows:
  stage: build
  tags:
    - windows
  before_script:
    - flutter config --enable-windows-desktop
    - flutter pub get
  script:
    - flutter build windows --release
  artifacts:
    paths:
      - build/windows/runner/Release/
    expire_in: 1 week
```

## 🔐 环境变量和密钥管理

### 在 GitLab 中配置密钥

1. 进入项目的 **Settings > CI/CD**
2. 展开 **Variables** 部分
3. 点击 **Add variable**
4. 勾选 **Mask variable**（隐藏日志输出）
5. 勾选 **Protect variable**（仅保护分支可用）

### Android 密钥配置

需要配置的变量：

```
ANDROID_KEYSTORE_BASE64       # Keystore 的 Base64 编码
ANDROID_KEYSTORE_PASSWORD     # Store 密码
ANDROID_KEY_PASSWORD          # Key 密码
ANDROID_KEY_ALIAS            # Key 别名
```

生成 Base64：

```bash
base64 -w 0 android/app/keystore.jks > keystore_base64.txt
```

在 Pipeline 中使用：

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

### iOS 证书配置

需要配置的变量：

```
IOS_CERTIFICATE_BASE64              # 证书的 Base64
IOS_CERTIFICATE_PASSWORD            # 证书密码
IOS_PROVISIONING_PROFILE_BASE64     # 配置文件的 Base64
KEYCHAIN_PASSWORD                   # 临时 Keychain 密码
APPLE_ID                            # Apple ID
APPLE_APP_SPECIFIC_PASSWORD         # App 专用密码
```

### 使用 CI/CD 变量组

创建变量组以便在多个项目间共享：

1. 进入 **Settings > CI/CD > Variables**
2. 创建 Group-level variables
3. 在项目中引用

## 🎯 高级功能

### 使用自定义 Docker 镜像

```yaml
image: YOUR_REGISTRY/flutter:custom

before_script:
  - flutter --version
```

创建自定义镜像的 Dockerfile：

```dockerfile
FROM ubuntu:22.04

# 安装依赖
RUN apt-get update && apt-get install -y \
    curl git unzip xz-utils zip libglu1-mesa \
    openjdk-11-jdk wget

# 安装 Flutter
RUN git clone https://github.com/flutter/flutter.git -b stable /flutter
ENV PATH="/flutter/bin:${PATH}"

# 预下载依赖
RUN flutter doctor
RUN flutter precache

WORKDIR /app
```

### 多阶段 Pipeline

```yaml
stages:
  - prepare
  - analyze
  - test
  - build
  - deploy
  - notify

prepare_dependencies:
  stage: prepare
  script:
    - flutter pub get
  artifacts:
    paths:
      - .dart_tool/
      - .packages
    expire_in: 1 hour

code_analyze:
  stage: analyze
  dependencies:
    - prepare_dependencies
  script:
    - flutter analyze --no-fatal-infos

unit_test:
  stage: test
  dependencies:
    - prepare_dependencies
  script:
    - flutter test --coverage

integration_test:
  stage: test
  script:
    - flutter test integration_test

build_production:
  stage: build
  dependencies:
    - unit_test
    - integration_test
  script:
    - flutter build apk --release
  only:
    - tags
```

### 并行任务

```yaml
test:android:
  stage: test
  script:
    - flutter test test/android_test.dart

test:ios:
  stage: test
  script:
    - flutter test test/ios_test.dart

test:web:
  stage: test
  script:
    - flutter test test/web_test.dart
```

### 条件执行

```yaml
# 只在主分支执行
deploy_production:
  script:
    - echo "部署到生产环境"
  only:
    - main

# 只在标签时执行
release:
  script:
    - echo "创建发布版本"
  only:
    - tags

# 只在 MR 时执行
test_mr:
  script:
    - flutter test
  only:
    - merge_requests

# 排除特定分支
build:
  script:
    - flutter build apk
  except:
    - develop
```

### 使用 Rules（推荐）

```yaml
build_apk:
  script:
    - flutter build apk --release
  rules:
    - if: '$CI_COMMIT_TAG'                 # 有标签时
      when: always
    - if: '$CI_COMMIT_BRANCH == "main"'    # 主分支时
      when: always
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'  # MR 时
      when: manual                         # 手动触发
    - when: never                          # 其他情况不执行
```

### 动态版本号

```yaml
build_with_version:
  script:
    - |
      VERSION=$(grep 'version:' pubspec.yaml | sed 's/version: //' | cut -d'+' -f1)
      BUILD_NUMBER=$CI_PIPELINE_IID
      echo "Building version $VERSION+$BUILD_NUMBER"
      flutter build apk --release --build-name=$VERSION --build-number=$BUILD_NUMBER
```

### 缓存优化

```yaml
cache:
  key:
    files:
      - pubspec.lock
      - android/build.gradle
  paths:
    - .pub-cache/
    - .dart_tool/
    - android/.gradle/
    - ios/Pods/
  policy: pull-push

# 只读缓存（加快速度）
test_job:
  cache:
    policy: pull
  script:
    - flutter test
```

## 🚀 自动部署

### 部署到 Google Play

```yaml
deploy_google_play:
  stage: deploy
  image: ruby:latest
  before_script:
    - gem install fastlane
  script:
    - cd android
    - fastlane supply --aab ../build/app/outputs/bundle/release/app-release.aab --track internal --json_key ../play-store-credentials.json
  only:
    - tags
  when: manual
```

### 部署到 Firebase App Distribution

```yaml
deploy_firebase:
  stage: deploy
  image: node:latest
  before_script:
    - npm install -g firebase-tools
  script:
    - firebase appdistribution:distribute build/app/outputs/flutter-apk/app-release.apk --app $FIREBASE_APP_ID --token $FIREBASE_TOKEN --groups "testers"
  only:
    - tags
```

### 部署 Web 到 GitLab Pages

```yaml
pages:
  stage: deploy
  script:
    - flutter build web --release --base-href /$CI_PROJECT_NAME/
    - mkdir -p public
    - cp -r build/web/* public/
  artifacts:
    paths:
      - public
  only:
    - main
```

访问地址：`https://YOUR_USERNAME.gitlab.io/PROJECT_NAME/`

### 部署到自定义服务器

```yaml
deploy_server:
  stage: deploy
  before_script:
    - apt-get update && apt-get install -y sshpass
  script:
    - sshpass -p "$SERVER_PASSWORD" scp -o StrictHostKeyChecking=no build/app/outputs/flutter-apk/app-release.apk $SERVER_USER@$SERVER_HOST:/var/www/apps/
  only:
    - tags
  when: manual
```

## 🏃 自建 GitLab Runner

### 为什么自建？

- 无限构建时间
- 更快的构建速度（本地网络）
- 完全控制构建环境
- 可以访问内网资源

### 安装 Runner（Linux）

```bash
# 下载
sudo curl -L --output /usr/local/bin/gitlab-runner https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-amd64

# 添加执行权限
sudo chmod +x /usr/local/bin/gitlab-runner

# 创建用户
sudo useradd --comment 'GitLab Runner' --create-home gitlab-runner --shell /bin/bash

# 安装
sudo gitlab-runner install --user=gitlab-runner --working-directory=/home/gitlab-runner
sudo gitlab-runner start
```

### 注册 Runner

```bash
sudo gitlab-runner register

# 输入：
# - GitLab URL: https://gitlab.com/
# - Registration token: (从项目设置获取)
# - Description: flutter-runner
# - Tags: flutter,linux
# - Executor: shell
```

### 安装 Flutter 到 Runner

```bash
# 切换到 gitlab-runner 用户
sudo su - gitlab-runner

# 安装 Flutter
git clone https://github.com/flutter/flutter.git -b stable
echo 'export PATH="$PATH:$HOME/flutter/bin"' >> ~/.bashrc
source ~/.bashrc

# 验证
flutter doctor
```

### 配置 Android SDK（可选）

```bash
# 下载 Android Command Line Tools
mkdir -p ~/android-sdk/cmdline-tools
cd ~/android-sdk/cmdline-tools
wget https://dl.google.com/android/repository/commandlinetools-linux-9477386_latest.zip
unzip commandlinetools-linux-9477386_latest.zip
mv cmdline-tools latest

# 配置环境变量
echo 'export ANDROID_HOME=$HOME/android-sdk' >> ~/.bashrc
echo 'export PATH=$PATH:$ANDROID_HOME/cmdline-tools/latest/bin:$ANDROID_HOME/platform-tools' >> ~/.bashrc
source ~/.bashrc

# 安装必要的包
sdkmanager "platform-tools" "platforms;android-33" "build-tools;33.0.0"
```

### 在 CI 中使用自建 Runner

```yaml
build_with_self_hosted:
  tags:
    - flutter
    - linux
  script:
    - flutter build apk --release
```

## 📊 监控和通知

### 生成测试覆盖率报告

```yaml
test:
  script:
    - flutter test --coverage
    - |
      apt-get update && apt-get install -y lcov
      genhtml coverage/lcov.info -o coverage/html
  coverage: '/lines......: \d+\.\d+\%/'
  artifacts:
    paths:
      - coverage/
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura.xml
```

### Slack 通知

```yaml
notify_slack:
  stage: notify
  image: curlimages/curl:latest
  script:
    - |
      curl -X POST -H 'Content-type: application/json' \
      --data "{\"text\":\"Pipeline $CI_PIPELINE_ID finished with status: $CI_JOB_STATUS\"}" \
      $SLACK_WEBHOOK_URL
  when: always
```

### Email 通知

GitLab 自动发送邮件通知，可在 **Settings > Integrations > Emails on push** 配置。

## 💡 最佳实践

### 1. 使用缓存

```yaml
cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - .pub-cache/
    - .dart_tool/
    - android/.gradle/
```

### 2. 优化触发条件

```yaml
# 只在需要时运行
build:
  script:
    - flutter build apk
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      when: never
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_COMMIT_TAG'
```

### 3. 使用 include 模块化配置

创建 `.gitlab-ci-templates/flutter-test.yml`：

```yaml
.flutter_test:
  script:
    - flutter pub get
    - flutter test
```

在主配置中引用：

```yaml
include:
  - local: '.gitlab-ci-templates/flutter-test.yml'

test_job:
  extends: .flutter_test
  stage: test
```

### 4. 使用变量简化配置

```yaml
variables:
  FLUTTER_VERSION: "3.16.0"
  BUILD_TYPE: "release"

before_script:
  - flutter --version | grep $FLUTTER_VERSION
```

## 🐛 常见问题

### 问题 1: Flutter 命令未找到

**解决方案**：确保使用正确的镜像或安装 Flutter

```yaml
image: cirrusci/flutter:stable

# 或者
before_script:
  - git clone https://github.com/flutter/flutter.git -b stable
  - export PATH="$PATH:`pwd`/flutter/bin"
```

### 问题 2: 缓存不生效

**解决方案**：检查缓存键和路径

```yaml
cache:
  key:
    files:
      - pubspec.lock  # 根据依赖文件生成缓存键
  paths:
    - .pub-cache/
```

### 问题 3: iOS 构建失败（Linux Runner）

**解决方案**：iOS 构建需要 macOS Runner

```yaml
build_ios:
  tags:
    - macos  # 使用 macOS Runner
  script:
    - flutter build ios
```

### 问题 4: 权限错误

**解决方案**：给予必要的权限

```bash
# 在 Runner 上
sudo chmod -R 755 /home/gitlab-runner
```

### 问题 5: 构建超时

**解决方案**：增加超时时间

```yaml
build:
  timeout: 2h  # 默认 1 小时
  script:
    - flutter build apk
```

## 💰 成本优化

### 免费额度
- 共享 Runner：每月 400 分钟（私有项目）
- 自建 Runner：无限制

### 节省策略
1. **使用自建 Runner**：完全免费
2. **优化缓存**：减少下载时间
3. **条件执行**：只在必要时构建
4. **并行任务**：提高效率

```yaml
# 示例：条件优化
test:
  script:
    - flutter test
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'
    - changes:
        - lib/**/*
        - test/**/*
```

## 📚 更多资源

- [GitLab CI/CD 官方文档](https://docs.gitlab.com/ee/ci/)
- [GitLab Runner 文档](https://docs.gitlab.com/runner/)
- [Flutter CI/CD 最佳实践](https://docs.flutter.dev/deployment/cd)
- [GitLab CI YAML 参考](https://docs.gitlab.com/ee/ci/yaml/)

---

**提示**：GitLab CI/CD 功能强大且灵活，建议根据团队需求逐步优化配置。自建 Runner 可以显著节省成本并提高构建速度。
