# Flutter CI/CD 完整指南

这个仓库包含了使用各种CI/CD服务构建和部署Flutter应用的完整中文指南。

## 📚 文档目录

### 推荐服务

- **[GitHub Actions](./docs/github-actions.md)** ⭐ 最推荐
  - 公开仓库完全免费
  - 私有仓库每月2000分钟免费
  - 与GitHub无缝集成

- **[Codemagic](./docs/codemagic.md)** ⭐ Flutter专用
  - 专为Flutter设计
  - 每月500分钟免费
  - 配置最简单

### 其他优质服务

- **[GitLab CI/CD](./docs/gitlab-ci.md)**
  - 每月400分钟免费
  - 可自建runner无限使用

- **[Cirrus CI](./docs/cirrus-ci.md)**
  - 开源项目完全免费
  - 支持多平台

- **[CircleCI](./docs/circleci.md)**
  - 每月6000分钟免费
  - 配置灵活

- **[Travis CI](./docs/travis-ci.md)**
  - 开源项目免费
  - GitHub集成良好

- **[Bitrise](./docs/bitrise.md)**
  - 移动应用友好
  - 有免费套餐

## 🎯 如何选择

### 如果你的代码在GitHub上
👉 使用 **GitHub Actions** - 免费额度充足，配置灵活，社区支持好

### 如果你需要专业的移动端CI/CD
👉 使用 **Codemagic** - Flutter原生支持，配置最简单，专为移动应用优化

### 如果你的代码在GitLab上
👉 使用 **GitLab CI/CD** - 集成无缝，可以自建runner

### 如果你是开源项目
👉 使用 **Cirrus CI** 或 **Travis CI** - 开源项目完全免费

## 📖 快速开始

1. 选择一个适合你的CI/CD服务
2. 点击上面的链接查看详细文档
3. 按照文档中的步骤配置你的项目
4. 提交代码，自动构建开始！

## 💡 通用最佳实践

### 1. 代码签名和密钥管理
- 永远不要将密钥提交到代码仓库
- 使用CI/CD平台的加密环境变量功能
- 定期更新和轮换密钥

### 2. 构建优化
- 使用缓存加速构建（依赖缓存、Gradle缓存等）
- 只在需要时构建特定平台
- 使用条件构建（如：只有tag推送时才发布）

### 3. 测试策略
- 在构建前运行单元测试
- 考虑集成测试和UI测试
- 使用测试覆盖率报告

### 4. 版本管理
- 使用语义化版本号
- 自动从git tag生成版本号
- 在每个构建中记录版本信息

## 🔧 前置要求

所有CI/CD方案都需要：

- Flutter项目已配置好
- 了解基本的YAML语法
- 对应平台的账号（GitHub/GitLab等）
- 用于签名的证书和密钥（发布生产版本时）

### Android发布需要
- Keystore文件
- Key密码和Store密码
- Key别名

### iOS发布需要
- Apple Developer账号
- 证书（Certificate）
- 配置文件（Provisioning Profile）
- 或者使用Fastlane Match管理证书

## 📱 支持的平台

所有文档都涵盖以下平台的构建：

- ✅ Android（APK和AAB）
- ✅ iOS（IPA）
- ✅ Web
- ✅ macOS
- ✅ Windows
- ✅ Linux

## 🤝 贡献

欢迎提交问题和改进建议！

## 📄 许可

本文档基于Flutter官方文档编写，遵循相同的许可协议。

---

**最后更新时间：** 2024年
