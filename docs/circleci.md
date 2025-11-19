# CircleCI - Flutter CI/CD 完整指南

## 📖 简介

CircleCI 是一个现代化的云端 CI/CD 平台，提供高性能、可扩展的持续集成和部署解决方案。它以灵活的配置、强大的缓存机制和出色的 Docker 支持而闻名。

### 优势

- ✅ **免费额度充足**：每月 6000 分钟构建时间（免费计划）
- ✅ **高性能**：快速的构建速度和启动时间
- ✅ **Docker 原生支持**：可使用任何 Docker 镜像
- ✅ **强大的缓存系统**：多级缓存机制
- ✅ **并行执行**：支持工作流和并行任务
- ✅ **云端和自托管**：支持 SaaS 和本地部署

### 适用场景

- 需要高性能 CI/CD 的中型团队
- 使用 Docker 容器化的项目
- 需要复杂工作流和并行任务
- 希望快速构建和部署

## 🚀 快速开始

### Step 1: 连接项目

1. 登录 [CircleCI](https://circleci.com)
2. 使用 GitHub/GitLab/Bitbucket 账号登录
3. 选择 **Projects** > **Set Up Project**
4. 选择你的 Flutter 项目

### Step 2: 创建配置文件

在项目根目录创建 `.circleci/config.yml`：

```yaml
version: 2.1

jobs:
  build:
    docker:
      - image: cirrusci/flutter:stable
    steps:
      - checkout
      - run:
          name: 安装依赖
          command: flutter pub get
      - run:
          name: 运行测试
          command: flutter test
      - run:
          name: 构建 APK
          command: flutter build apk --release
      - store_artifacts:
          path: build/app/outputs/flutter-apk/app-release.apk

workflows:
  version: 2
  build-and-test:
    jobs:
      - build
```

### Step 3: 提交并触发

```bash
mkdir -p .circleci
git add .circleci/config.yml
git commit -m "添加 CircleCI 配置"
git push
```

CircleCI 会自动检测配置并开始构建。

## 📋 完整配置示例

### Android 应用构建

```yaml
version: 2.1

orbs:
  android: circleci/android@2.3.0

jobs:
  test:
    docker:
      - image: cirrusci/flutter:stable
    steps:
      - checkout
      - restore_cache:
          keys:
            - flutter-pub-cache-{{ checksum "pubspec.lock" }}
            - flutter-pub-cache-
      - run:
          name: 获取依赖
          command: flutter pub get
      - save_cache:
          key: flutter-pub-cache-{{ checksum "pubspec.lock" }}
          paths:
            - ~/.pub-cache
      - run:
          name: 代码分析
          command: flutter analyze
      - run:
          name: 运行测试
          command: flutter test --coverage
      - run:
          name: 生成覆盖率报告
          command: |
            if [ ! -d "coverage" ]; then
              echo "没有覆盖率数据"
              exit 0
            fi
      - store_test_results:
          path: test-results
      - store_artifacts:
          path: coverage

  build_apk:
    docker:
      - image: cirrusci/flutter:stable
    steps:
      - checkout
      - restore_cache:
          keys:
            - flutter-pub-cache-{{ checksum "pubspec.lock" }}
      - run: flutter pub get
      - run:
          name: 解码 Keystore
          command: |
            echo "$ANDROID_KEYSTORE_BASE64" | base64 -d > android/app/keystore.jks
      - run:
          name: 创建 key.properties
          command: |
            cat > android/key.properties <<EOF
            storePassword=$ANDROID_KEYSTORE_PASSWORD
            keyPassword=$ANDROID_KEY_PASSWORD
            keyAlias=$ANDROID_KEY_ALIAS
            storeFile=keystore.jks
            EOF
      - restore_cache:
          keys:
            - gradle-cache-{{ checksum "android/build.gradle" }}
            - gradle-cache-
      - run:
          name: 构建 APK
          command: flutter build apk --release
      - run:
          name: 构建 App Bundle
          command: flutter build appbundle --release
      - save_cache:
          key: gradle-cache-{{ checksum "android/build.gradle" }}
          paths:
            - ~/.gradle
      - store_artifacts:
          path: build/app/outputs/flutter-apk/app-release.apk
          destination: app-release.apk
      - store_artifacts:
          path: build/app/outputs/bundle/release/app-release.aab
          destination: app-release.aab
      - persist_to_workspace:
          root: build/app/outputs
          paths:
            - flutter-apk/app-release.apk
            - bundle/release/app-release.aab

workflows:
  version: 2
  test-and-build:
    jobs:
      - test
      - build_apk:
          requires:
            - test
          filters:
            branches:
              only:
                - main
                - develop
```

### iOS 应用构建

```yaml
version: 2.1

jobs:
  build_ios:
    macos:
      xcode: 15.0.0
    environment:
      FLUTTER_VERSION: "3.16.0"
    steps:
      - checkout
      - run:
          name: 安装 Flutter
          command: |
            git clone https://github.com/flutter/flutter.git -b stable
            echo 'export PATH="$PATH:`pwd`/flutter/bin"' >> $BASH_ENV
            source $BASH_ENV
            flutter --version
      - restore_cache:
          keys:
            - flutter-pub-cache-{{ checksum "pubspec.lock" }}
      - run:
          name: 获取依赖
          command: flutter pub get
      - save_cache:
          key: flutter-pub-cache-{{ checksum "pubspec.lock" }}
          paths:
            - ~/.pub-cache
      - restore_cache:
          keys:
            - pods-cache-{{ checksum "ios/Podfile.lock" }}
      - run:
          name: 安装 CocoaPods
          command: |
            cd ios
            pod install
      - save_cache:
          key: pods-cache-{{ checksum "ios/Podfile.lock" }}
          paths:
            - ios/Pods
      - run:
          name: 设置证书
          command: |
            echo "$IOS_CERTIFICATE_BASE64" | base64 -d > certificate.p12
            echo "$IOS_PROVISIONING_PROFILE_BASE64" | base64 -d > profile.mobileprovision
            
            security create-keychain -p "$KEYCHAIN_PASSWORD" build.keychain
            security default-keychain -s build.keychain
            security unlock-keychain -p "$KEYCHAIN_PASSWORD" build.keychain
            security set-keychain-settings -t 3600 -l ~/Library/Keychains/build.keychain
            
            security import certificate.p12 -k build.keychain -P "$IOS_CERTIFICATE_PASSWORD" -T /usr/bin/codesign
            security set-key-partition-list -S apple-tool:,apple: -s -k "$KEYCHAIN_PASSWORD" build.keychain
            
            mkdir -p ~/Library/MobileDevice/Provisioning\ Profiles
            cp profile.mobileprovision ~/Library/MobileDevice/Provisioning\ Profiles/
      - run:
          name: 构建 iOS
          command: flutter build ios --release --no-codesign
      - run:
          name: 打包 IPA
          command: |
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
      - store_artifacts:
          path: ios/build/*.ipa

workflows:
  version: 2
  ios-workflow:
    jobs:
      - build_ios:
          filters:
            branches:
              only: main
```

### 多平台并行构建

```yaml
version: 2.1

executors:
  flutter:
    docker:
      - image: cirrusci/flutter:stable
  flutter-macos:
    macos:
      xcode: 15.0.0

jobs:
  test:
    executor: flutter
    steps:
      - checkout
      - run: flutter pub get
      - run: flutter test

  build_android:
    executor: flutter
    steps:
      - checkout
      - run: flutter pub get
      - run: flutter build apk --release
      - store_artifacts:
          path: build/app/outputs/flutter-apk/app-release.apk

  build_ios:
    executor: flutter-macos
    steps:
      - checkout
      - run:
          name: 安装 Flutter
          command: |
            git clone https://github.com/flutter/flutter.git -b stable
            export PATH="$PATH:`pwd`/flutter/bin"
            flutter --version
      - run: flutter pub get
      - run: cd ios && pod install
      - run: flutter build ios --release --no-codesign
      - store_artifacts:
          path: build/ios/iphoneos

  build_web:
    executor: flutter
    steps:
      - checkout
      - run: flutter config --enable-web
      - run: flutter pub get
      - run: flutter build web --release
      - store_artifacts:
          path: build/web

workflows:
  version: 2
  multi-platform:
    jobs:
      - test
      - build_android:
          requires:
            - test
      - build_ios:
          requires:
            - test
      - build_web:
          requires:
            - test
```

## 🔧 Orbs（可重用配置）

CircleCI Orbs 是预打包的可重用配置模块。

### 使用 Flutter Orb

```yaml
version: 2.1

orbs:
  flutter: circleci/flutter@1.0.0

workflows:
  build-and-test:
    jobs:
      - flutter/test
      - flutter/build:
          platform: android
          requires:
            - flutter/test
```

### 自定义 Orb

创建可复用的命令：

```yaml
version: 2.1

commands:
  setup_flutter:
    description: "安装并配置 Flutter"
    parameters:
      version:
        type: string
        default: "stable"
    steps:
      - run:
          name: 安装 Flutter
          command: |
            git clone https://github.com/flutter/flutter.git -b << parameters.version >>
            export PATH="$PATH:`pwd`/flutter/bin"
            flutter --version

jobs:
  build:
    docker:
      - image: ubuntu:22.04
    steps:
      - setup_flutter:
          version: "3.16.0"
      - run: flutter build apk
```

## 🔐 环境变量和密钥管理

### 在 CircleCI 中配置

1. 进入项目设置
2. 选择 **Environment Variables**
3. 点击 **Add Variable**
4. 输入名称和值

### 常用变量

```
ANDROID_KEYSTORE_BASE64
ANDROID_KEYSTORE_PASSWORD
ANDROID_KEY_PASSWORD
ANDROID_KEY_ALIAS
IOS_CERTIFICATE_BASE64
IOS_CERTIFICATE_PASSWORD
IOS_PROVISIONING_PROFILE_BASE64
KEYCHAIN_PASSWORD
FIREBASE_TOKEN
```

### 在配置中使用

```yaml
- run:
    name: 使用环境变量
    command: |
      echo "$ANDROID_KEYSTORE_BASE64" | base64 -d > keystore.jks
      echo "Keystore 密码: $ANDROID_KEYSTORE_PASSWORD"
```

### 使用 Context（跨项目共享变量）

```yaml
workflows:
  version: 2
  build:
    jobs:
      - build_android:
          context: android-signing  # 使用 context
```

创建 Context：
1. Organization Settings > Contexts
2. Create Context
3. 添加环境变量

## 🎯 高级功能

### 缓存优化

```yaml
# 依赖缓存
- restore_cache:
    keys:
      - pub-cache-v1-{{ checksum "pubspec.lock" }}
      - pub-cache-v1-

- run: flutter pub get

- save_cache:
    key: pub-cache-v1-{{ checksum "pubspec.lock" }}
    paths:
      - ~/.pub-cache

# Gradle 缓存
- restore_cache:
    keys:
      - gradle-cache-v1-{{ checksum "android/build.gradle" }}
      - gradle-cache-v1-

- save_cache:
    key: gradle-cache-v1-{{ checksum "android/build.gradle" }}
    paths:
      - ~/.gradle
      - android/.gradle

# CocoaPods 缓存
- restore_cache:
    keys:
      - pods-cache-v1-{{ checksum "ios/Podfile.lock" }}

- save_cache:
    key: pods-cache-v1-{{ checksum "ios/Podfile.lock" }}
    paths:
      - ios/Pods
```

### 工作区（Workspace）

在任务之间传递文件：

```yaml
jobs:
  build:
    steps:
      - run: flutter build apk
      - persist_to_workspace:
          root: build/app/outputs
          paths:
            - flutter-apk/app-release.apk

  deploy:
    steps:
      - attach_workspace:
          at: /tmp/workspace
      - run: ls /tmp/workspace/flutter-apk
```

### 并行测试

```yaml
jobs:
  test:
    parallelism: 4
    docker:
      - image: cirrusci/flutter:stable
    steps:
      - checkout
      - run: flutter pub get
      - run:
          name: 运行测试
          command: |
            TEST_FILES=$(circleci tests glob "test/**/*_test.dart" | circleci tests split --split-by=timings)
            flutter test $TEST_FILES
```

### 条件执行

```yaml
workflows:
  version: 2
  build-workflow:
    jobs:
      - test
      - build_android:
          requires:
            - test
          filters:
            branches:
              only:
                - main
                - develop
      - deploy:
          requires:
            - build_android
          filters:
            branches:
              only: main
            tags:
              only: /^v.*/
```

### 定时任务

```yaml
workflows:
  version: 2
  nightly-build:
    triggers:
      - schedule:
          cron: "0 0 * * *"  # 每天午夜
          filters:
            branches:
              only: main
    jobs:
      - build_android
```

### 审批步骤

```yaml
workflows:
  version: 2
  build-and-deploy:
    jobs:
      - build_android
      - hold:
          type: approval  # 手动审批
          requires:
            - build_android
      - deploy:
          requires:
            - hold
```

## 🚀 自动部署

### 部署到 Google Play

```yaml
jobs:
  deploy_google_play:
    docker:
      - image: cirrusci/flutter:stable
    steps:
      - attach_workspace:
          at: /tmp/workspace
      - run:
          name: 安装 Fastlane
          command: |
            gem install fastlane
      - run:
          name: 部署到 Google Play
          command: |
            cd android
            fastlane supply \
              --aab /tmp/workspace/bundle/release/app-release.aab \
              --track internal \
              --json_key ../play-store-credentials.json

workflows:
  version: 2
  build-and-deploy:
    jobs:
      - build_android
      - deploy_google_play:
          requires:
            - build_android
          filters:
            branches:
              only: main
```

### 部署到 Firebase

```yaml
- run:
    name: 部署到 Firebase App Distribution
    command: |
      npm install -g firebase-tools
      firebase appdistribution:distribute \
        build/app/outputs/flutter-apk/app-release.apk \
        --app $FIREBASE_APP_ID \
        --token $FIREBASE_TOKEN \
        --groups "testers"
```

### 部署 Web 到 Firebase Hosting

```yaml
jobs:
  deploy_web:
    docker:
      - image: node:latest
    steps:
      - checkout
      - run: flutter build web --release
      - run:
          name: 部署到 Firebase Hosting
          command: |
            npm install -g firebase-tools
            firebase deploy --only hosting --token $FIREBASE_TOKEN
```

## 📊 测试和报告

### 测试结果报告

```yaml
- run:
    name: 运行测试
    command: |
      flutter test --machine > test-results.json
- store_test_results:
    path: test-results
- store_artifacts:
    path: test-results
```

### 代码覆盖率

```yaml
- run:
    name: 生成覆盖率
    command: |
      flutter test --coverage
      apt-get update && apt-get install -y lcov
      genhtml coverage/lcov.info -o coverage/html
- store_artifacts:
    path: coverage/html
    destination: coverage-report
```

### 上传到 Codecov

```yaml
- run:
    name: 上传覆盖率到 Codecov
    command: |
      bash <(curl -s https://codecov.io/bash)
```

## 💡 最佳实践

### 1. 使用执行器（Executors）

```yaml
executors:
  flutter-executor:
    docker:
      - image: cirrusci/flutter:stable
    environment:
      GRADLE_OPTS: -Xmx1536m
    resource_class: large

jobs:
  build:
    executor: flutter-executor
    steps:
      - checkout
      - run: flutter build apk
```

### 2. 资源类优化

```yaml
jobs:
  build:
    docker:
      - image: cirrusci/flutter:stable
    resource_class: large  # small, medium, large, xlarge
    steps:
      - run: flutter build apk
```

### 3. 使用工作流过滤器

```yaml
workflows:
  version: 2
  build-workflow:
    jobs:
      - build_android:
          filters:
            branches:
              only:
                - /feature-.*/
                - main
            tags:
              only: /^v.*/
```

### 4. 模块化配置

将配置拆分为多个文件：

```yaml
version: 2.1

setup: true

orbs:
  continuation: circleci/continuation@0.3.1

workflows:
  setup-workflow:
    jobs:
      - continuation/continue:
          configuration_path: .circleci/continue-config.yml
```

## 🐛 常见问题

### 问题 1: 构建超时

**解决方案**：增加超时时间

```yaml
- run:
    name: 构建 APK
    command: flutter build apk --release
    no_output_timeout: 30m
```

### 问题 2: 内存不足

**解决方案**：使用更大的资源类

```yaml
jobs:
  build:
    resource_class: large  # 或 xlarge
    docker:
      - image: cirrusci/flutter:stable
```

### 问题 3: 缓存未命中

**解决方案**：检查缓存键

```yaml
- restore_cache:
    keys:
      - v1-pub-{{ checksum "pubspec.lock" }}
      - v1-pub-  # 后备缓存
```

### 问题 4: macOS 构建慢

**解决方案**：使用缓存和优化依赖

```yaml
- restore_cache:
    keys:
      - pods-{{ checksum "ios/Podfile.lock" }}
- run: cd ios && pod install --repo-update
- save_cache:
    paths:
      - ios/Pods
    key: pods-{{ checksum "ios/Podfile.lock" }}
```

## 💰 费用优化

### 免费计划
- 每月 6000 分钟构建时间
- 1 个并发任务
- Linux、Docker、macOS、Windows 环境

### 优化策略

1. **使用缓存**：减少依赖下载时间
2. **并行执行**：多任务同时运行
3. **条件构建**：只在需要时构建
4. **资源类优化**：根据需求选择合适大小

```yaml
# 示例：条件构建
workflows:
  version: 2
  build:
    jobs:
      - test:
          filters:
            branches:
              ignore: /feature-.*/
      - build_android:
          filters:
            branches:
              only:
                - main
                - develop
```

## 📚 更多资源

- [CircleCI 官方文档](https://circleci.com/docs/)
- [CircleCI Orbs 注册表](https://circleci.com/developer/orbs)
- [Flutter CI/CD 最佳实践](https://docs.flutter.dev/deployment/cd)
- [CircleCI 配置参考](https://circleci.com/docs/configuration-reference/)

---

**提示**：CircleCI 提供强大的性能和灵活性，合理使用缓存、并行执行和资源类可以显著提高构建效率。
