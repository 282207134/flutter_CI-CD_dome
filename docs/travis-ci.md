# Travis CI - Flutter CI/CD 指南

## 📖 简介

Travis CI 是早期应用广泛的云端持续集成平台，对开源项目提供免费支持，并与 GitHub 集成紧密。虽然相较于更新平台略显传统，但其配置简单、生态成熟，仍然适合大量 Flutter 项目。

### 优势

- ✅ **开源项目永久免费**
- ✅ **配置简单**：使用 `.travis.yml` 配置
- ✅ **与 GitHub 集成良好**
- ✅ **支持 Linux、macOS、Windows**
- ✅ **自定义脚本灵活**

### 不足

- ⚠️ 免费额度仅适用于开源项目，私有仓库需付费
- ⚠️ 构建速度较新平台略慢
- ⚠️ Windows/macOS 构建需要指定 plan

## 🚀 快速开始

### Step 1: 启用 Travis CI

1. 登录 [Travis CI](https://travis-ci.com)（或 .org，视仓库而定）
2. 使用 GitHub 账号登录
3. 在 **Accounts** → **Repositories** 中启用目标仓库

### Step 2: 创建 `.travis.yml`

```yaml
language: dart

os:
  - linux
  - windows

jobs:
  include:
    - stage: test
      os: linux
      script:
        - flutter pub get
        - flutter analyze
        - flutter test
    - stage: build
      os: linux
      script:
        - flutter pub get
        - flutter build apk --release

stages:
  - test
  - name: build
    if: branch = main

before_install:
  - git clone https://github.com/flutter/flutter.git -b stable
  - export PATH="$PATH:`pwd`/flutter/bin"
```

### Step 3: 提交并触发

```bash
git add .travis.yml
git commit -m "添加 Travis CI 配置"
git push
```

Travis CI 会自动检测新配置并开始构建。

## 📋 配置详解

### 1. 指定语言和环境

```yaml
language: generic  # 官方没有 Flutter 专用语言，可使用 generic 或 dart
os:
  - linux
  - osx
  - windows
```

### 2. 安装 Flutter

```yaml
before_install:
  - git clone https://github.com/flutter/flutter.git -b stable
  - export PATH="$PATH:`pwd`/flutter/bin"
  - flutter --version
```

### 3. 缓存依赖

```yaml
cache:
  directories:
    - $HOME/.pub-cache
    - $HOME/.gradle
    - $HOME/.cache/flutter
```

### 4. 完整示例：Android + iOS

```yaml
os: linux
language: generic

stages:
  - analyze
  - test
  - build_android
  - build_ios

jobs:
  include:
    - stage: analyze
      script:
        - flutter pub get
        - flutter analyze
    
    - stage: test
      script:
        - flutter pub get
        - flutter test --coverage
      after_success:
        - bash <(curl -s https://codecov.io/bash)
    
    - stage: build_android
      script:
        - flutter pub get
        - echo "$ANDROID_KEYSTORE_BASE64" | base64 -d > android/app/keystore.jks
        - |
          cat > android/key.properties <<EOF
          storePassword=$ANDROID_KEYSTORE_PASSWORD
          keyPassword=$ANDROID_KEY_PASSWORD
          keyAlias=$ANDROID_KEY_ALIAS
          storeFile=keystore.jks
          EOF
        - flutter build appbundle --release
      deploy:
        provider: releases
        api_key: $GITHUB_TOKEN
        file_glob: true
        file:
          - build/app/outputs/bundle/release/app-release.aab
        skip_cleanup: true
        on:
          tags: true
    
    - stage: build_ios
      os: osx
      osx_image: xcode15
      script:
        - flutter pub get
        - cd ios && pod install && cd ..
        - flutter build ios --release --no-codesign
      deploy:
        provider: releases
        api_key: $GITHUB_TOKEN
        file_glob: true
        file: build/ios/iphoneos/*.app
        skip_cleanup: true
        on:
          branch: main

before_install:
  - git clone https://github.com/flutter/flutter.git -b stable
  - export PATH="$PATH:`pwd`/flutter/bin"
```

## 🔐 环境变量和密钥管理

### 设置步骤
1. 进入 Travis CI 项目页面
2. 点击 **More options → Settings**
3. 在 **Environment Variables** 中添加变量
4. 选择是否在 Pull Request 中可见

### 常用变量
```
ANDROID_KEYSTORE_BASE64
ANDROID_KEYSTORE_PASSWORD
ANDROID_KEY_PASSWORD
ANDROID_KEY_ALIAS
IOS_CERTIFICATE_BASE64
IOS_CERTIFICATE_PASSWORD
IOS_PROVISIONING_PROFILE_BASE64
APPLE_ID
APPLE_APP_SPECIFIC_PASSWORD
FIREBASE_TOKEN
```

### 使用方式

```yaml
script:
  - echo "$ANDROID_KEYSTORE_BASE64" | base64 -d > android/app/keystore.jks
```

## 🎯 测试与分析

```yaml
script:
  - flutter analyze
  - flutter test --coverage
after_success:
  - bash <(curl -s https://codecov.io/bash)
```

### 严格模式：失败即中断

```yaml
script:
  - set -e
  - flutter analyze
  - flutter test
```

## 🚀 发布

### 自动创建 GitHub Release

```yaml
deploy:
  provider: releases
  api_key: $GITHUB_TOKEN
  file:
    - build/app/outputs/flutter-apk/app-release.apk
    - build/app/outputs/bundle/release/app-release.aab
  skip_cleanup: true
  on:
    tags: true
```

`GITHUB_TOKEN` 可使用 GitHub Personal Access Token，添加到 Travis 环境变量中。

### 部署到 Firebase

```yaml
script:
  - curl -sL https://firebase.tools | bash
  - firebase appdistribution:distribute build/app/outputs/flutter-apk/app-release.apk --app $FIREBASE_APP_ID --token $FIREBASE_TOKEN
```

## 🧰 多平台构建

### Linux + macOS + Windows

```yaml
os:
  - linux
  - osx
  - windows

jobs:
  include:
    - name: Linux 测试
      script:
        - flutter test
    - name: macOS 构建
      os: osx
      script:
        - flutter build ios --no-codesign
    - name: Windows 构建
      os: windows
      script:
        - flutter build windows --release
```

### 使用矩阵

```yaml
matrix:
  include:
    - os: linux
      env: TARGET=apk
    - os: linux
      env: TARGET=web
    - os: osx
      env: TARGET=ios

script:
  - flutter pub get
  - if [ "$TARGET" = "apk" ]; then flutter build apk --release; fi
  - if [ "$TARGET" = "web" ]; then flutter build web --release; fi
  - if [ "$TARGET" = "ios" ]; then flutter build ios --release --no-codesign; fi
```

## 🔄 缓存与优化

```yaml
cache:
  directories:
    - $HOME/.pub-cache
    - $HOME/.gradle
    - $HOME/Library/Caches/CocoaPods
    - android/.gradle

# 仅在依赖变化时重新安装
before_cache:
  - rm -rf $HOME/.gradle/caches/*/fileHashes/
```

### 并行 Stage

```yaml
stages:
  - name: lint
  - name: test
  - name: build

jobs:
  include:
    - stage: lint
      script: flutter analyze
    - stage: test
      script: flutter test
    - stage: build
      script: flutter build apk --release
```

### 条件执行

```yaml
jobs:
  include:
    - stage: build
      if: branch = main AND type = push
      script: flutter build appbundle --release
```

## 🐛 常见问题

| 问题 | 解决 |
| ---- | ---- |
| 找不到 Flutter 命令 | 确保在 `before_install` 中设置 PATH |
| macOS 构建失败 | 指定 `osx_image`，并确保配额充足 |
| 缓存不生效 | 确认路径正确，避免缓存临时文件 |
| Pull Request 无法访问密钥 | 在环境变量中取消 `Display value in build log` 并允许 PR |
| 构建超时 | 启用缓存，优化脚本，或使用 `travis_wait` |
| Gradle 内存不足 | 设置 `GRADLE_OPTS: -Xmx1536m` |

```yaml
env:
  global:
    - GRADLE_OPTS=-Xmx2048m
```

## 💰 费用与优化

### 免费额度
- 开源项目：无限制
- 私有项目：根据 Plan（早期旧计划较受限）

### 节省策略
1. 仅在关键分支运行构建
2. 使用缓存加速
3. 使用 `if` 条件过滤 PR/分支
4. 压缩日志输出

```yaml
branches:
  only:
    - main
    - develop
    - /^release\/.*$/
```

## 📚 更多资源

- [Travis CI 官方文档](https://docs.travis-ci.com/)
- [Travis CI 与 GitHub 集成](https://docs.travis-ci.com/user/tutorial/)
- [Flutter 官方 CI/CD 文档](https://docs.flutter.dev/deployment/cd)
- [Travis CI 构建矩阵](https://docs.travis-ci.com/user/build-matrix/)

---

**提示**：如果你主要维护开源 Flutter 项目，Travis CI 仍然是稳定可靠的选择。通过缓存与阶段化配置，可以获得可接受的构建速度和清晰的流水线结构。
