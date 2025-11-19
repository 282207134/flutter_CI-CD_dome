# Unity + Cursor + MCP 自动化开发完整指南

## 📁 第一步：项目初始化配置

### 1.1 创建项目结构文档
```markdown
# PROJECT_STRUCTURE.md

## 游戏概述
**游戏类型**: 2048变体 - 数字合成消除游戏
**平台**: iOS + Android
**核心玩法**: 滑动合并相同数字，达到目标分数
**变现**: 激励广告 + 去广告内购

## Unity项目设置
- Unity版本: 2022.3 LTS
- 渲染管线: URP
- 输入系统: New Input System
- 架构模式: MVC

## 文件夹结构
Assets/
├── _Project/
│   ├── Scripts/
│   │   ├── Core/          # 核心游戏逻辑
│   │   ├── UI/            # UI控制器
│   │   ├── Managers/      # 单例管理器
│   │   ├── Data/          # 数据模型和SO
│   │   └── Utilities/     # 工具类
│   ├── Prefabs/
│   ├── Scenes/
│   ├── Materials/
│   ├── Audio/
│   └── Resources/
├── Plugins/               # 第三方插件
└── TextMesh Pro/
```

### 1.2 技术规范文档
```markdown
# TECH_SPECS.md

## 编码规范
- 命名: PascalCase类/方法, camelCase私有字段, _camelCase序列化字段
- 注释: 所有public方法必须有XML注释
- 架构: 使用事件系统解耦，避免GetComponent在Update

## 性能要求
- 目标帧率: 60 FPS
- 内存限制: < 150MB
- 启动时间: < 3秒

## 必用组件
- DOTween: 动画补间
- Unity Ads: 广告变现
- TextMesh Pro: 文本渲染

## Git忽略规则
.gitignore包含: Library/, Temp/, Logs/, UserSettings/
```

---

## 🤖 第二步：Cursor Prompt 模板库

### 2.1 初始化项目 Prompt
```
我正在用Unity 2022.3 LTS开发一款2048变体手游。请按照以下步骤初始化项目：

## 项目背景
参考文档：@PROJECT_STRUCTURE.md @TECH_SPECS.md

## 立即执行任务
1. 创建完整的文件夹结构（如PROJECT_STRUCTURE.md所示）
2. 生成核心管理器基类：
   - GameManager.cs（单例，游戏状态机）
   - AudioManager.cs（音效/音乐管理）
   - UIManager.cs（UI栈管理）
   - DataManager.cs（存档/配置）

3. 每个管理器需要：
   - 单例模式（线程安全）
   - XML文档注释
   - 基础事件系统
   - 初始化/清理方法

4. 创建游戏状态枚举：
   public enum GameState { MainMenu, Playing, Paused, GameOver, Loading }

请逐个生成完整代码，每个文件包含完整命名空间和using语句。
```

### 2.2 核心玩法开发 Prompt
```
现在开发2048游戏的核心逻辑。参考 @TECH_SPECS.md 的性能要求。

## 需要生成的脚本

### 1. GridSystem.cs
- 4x4网格数据结构（使用int[,]数组）
- 方法：InitializeGrid(), GetTile(x,y), SetTile(x,y,value)
- 空格检测：HasEmptySpace()
- 可合并检测：CanMerge()

### 2. TileController.cs (MonoBehaviour)
- 属性：int value, Vector2Int gridPosition
- 方法：SetValue(), AnimateMove(target), AnimateMerge(), AnimateSpawn()
- 使用DOTween实现动画（duration 0.2s, Ease.OutQuad）

### 3. InputHandler.cs
- 使用New Input System
- 检测滑动方向（上下左右）
- 触摸灵敏度：50像素最小距离
- 事件：OnSwipe(Vector2Int direction)

### 4. MoveProcessor.cs
- ProcessMove(direction) 处理一次滑动
- 瓦片移动逻辑（先移动再合并）
- 返回：移动的瓦片列表 + 得分增量
- 算法要高效，避免重复遍历

### 5. ScoreSystem.cs
- 当前分数、最高分、连击数
- CalculateScore(mergedValue, comboCount)
- 事件：OnScoreChanged, OnNewHighScore

要求：
- 所有类都要有详细注释
- 性能优化：对象池复用TileController
- 使用UnityEvent解耦
- 代码遵循 @TECH_SPECS.md 规范
```

### 2.3 UI系统开发 Prompt
```
开发游戏UI系统。参考 @PROJECT_STRUCTURE.md 的UI文件夹结构。

## 需要的UI脚本

### 1. BasePanel.cs（抽象基类）
- 方法：Show(), Hide(), OnShow(), OnHide()
- 使用CanvasGroup控制显示（alpha + interactable）
- DOTween渐入渐出动画

### 2. MainMenuPanel.cs : BasePanel
- 按钮：开始游戏、继续游戏、设置、商店
- 显示最高分
- 检查是否有存档决定"继续"按钮状态

### 3. GameplayPanel.cs : BasePanel
- 实时显示：当前分数、最高分、移动次数
- 按钮：暂停、撤销（可选）
- 连击特效文字（"Great!" "Perfect!" "Combo x3"）

### 4. PausePanel.cs : BasePanel
- 半透明背景（Color 0,0,0,180）
- 按钮：继续、重新开始、主菜单
- 显示当前游戏统计

### 5. GameOverPanel.cs : BasePanel
- 显示最终分数、最高分、评价星级
- 按钮：看广告重生、重新开始、分享
- 星级计算逻辑（1000分=1星，2000=2星，4000=3星）

### 6. UIManager.cs 扩展
- PanelStack管理（后进先出）
- ShowPanel<T>(), HidePanel<T>(), HideAllPanels()
- 面板转场动画

要求：
- 所有UI使用TextMesh Pro
- 按钮添加音效触发
- 使用对象池优化特效文字
- 响应式布局（支持不同分辨率）
```

### 2.4 数据持久化 Prompt
```
实现存档和数据持久化系统。参考 @TECH_SPECS.md。

## 需要生成

### 1. GameData.cs（可序列化数据类）
```csharp
[System.Serializable]
public class GameData
{
    public int highScore;
    public int currentScore;
    public int[,] gridState;
    public int moveCount;
    public long lastPlayTime; // Unix时间戳
    public bool soundEnabled;
    public bool musicEnabled;
    public bool adsRemoved;
}
```

### 2. SaveLoadSystem.cs
- SaveGame(GameData data)：JSON序列化 -> PlayerPrefs或文件
- LoadGame()：返回GameData或null
- DeleteSave()：清除存档
- HasSave()：检查是否有存档
- 使用JsonUtility或Newtonsoft.Json

### 3. DataManager.cs 扩展
- CurrentGameData属性
- AutoSave功能（每次移动后自动保存）
- 配置管理（音效/音乐开关）

### 4. PlayerPrefs键名常量
```csharp
public static class SaveKeys
{
    public const string GAME_DATA = "GameData";
    public const string HIGH_SCORE = "HighScore";
    public const string SOUND_ENABLED = "Sound";
    public const string MUSIC_ENABLED = "Music";
}
```

要求：
- 加密敏感数据（防作弊）
- 异常处理（损坏的存档）
- 版本控制（数据结构变更兼容）
```

### 2.5 广告和内购 Prompt
```
集成Unity Ads和内购系统（纯代码，不涉及后台配置）。

## 生成脚本

### 1. AdsManager.cs : MonoBehaviour
```csharp
// 功能需求：
- 初始化Unity Ads（测试模式）
- ShowRewardedAd(Action<bool> callback) // 激励广告
- ShowInterstitialAd() // 插屏广告
- IsRewardedAdReady() // 检查广告是否准备好
- 广告频率控制（关卡间隔显示插屏）
```

### 2. IAPManager.cs
```csharp
// 产品ID定义
public static class ProductIDs
{
    public const string REMOVE_ADS = "com.yourgame.removeads";
    public const string COIN_PACK_SMALL = "com.yourgame.coins100";
}

// 功能：
- 初始化IAP（Unity IAP）
- PurchaseProduct(string productID, Action<bool> callback)
- RestorePurchases()
- HasPurchased(string productID) // 检查是否已购买
```

### 3. MonetizationController.cs
```csharp
// 整合广告和内购的高层逻辑
- ShowRewardedAdForContinue() // 游戏结束看广告重生
- ShowRewardedAdForHint() // 看广告获得提示
- CheckAdsRemoved() // 是否移除广告
- TrackAdRevenue() // 广告收入追踪（集成后）
```

要求：
- 优雅的回调处理
- 广告加载失败提示
- 去广告后隐藏所有广告按钮
- 防止广告频繁弹出（冷却时间）
```

### 2.6 音效系统 Prompt
```
完善AudioManager实现游戏音效和音乐。

## 扩展 AudioManager.cs

### 功能需求
1. 音效管理
   - PlaySFX(string clipName, float volume = 1f)
   - 音效类型：按钮点击、瓦片移动、合并、得分、游戏结束
   - 使用对象池管理AudioSource

2. 音乐管理
   - PlayMusic(string clipName, bool loop = true)
   - CrossFade切换音乐（渐入渐出）
   - StopMusic(), PauseMusic(), ResumeMusic()

3. 设置
   - SetSFXVolume(float volume) // 0-1
   - SetMusicVolume(float volume)
   - ToggleSFX(), ToggleMusic()
   - 设置持久化到PlayerPrefs

### 音效命名约定
```csharp
public static class SoundNames
{
    // SFX
    public const string BUTTON_CLICK = "btn_click";
    public const string TILE_MOVE = "tile_move";
    public const string TILE_MERGE = "tile_merge";
    public const string SCORE_UP = "score_up";
    public const string GAME_OVER = "game_over";
    
    // Music
    public const string MENU_MUSIC = "music_menu";
    public const string GAME_MUSIC = "music_game";
}
```

要求：
- 音效短促响亮
- 音乐循环无缝
- 音量淡入淡出自然
- Resources文件夹加载
```

### 2.7 特效和视觉反馈 Prompt
```
增强游戏视觉反馈和粒子特效。

## 生成脚本

### 1. ParticleManager.cs
```csharp
// 粒子效果池管理
- PlayEffect(string effectName, Vector3 position)
- 特效类型：
  * merge_particle（合并时爆炸）
  * score_particle（得分文字飘动）
  * spawn_particle（新瓦片生成）
  * clear_particle（清除特效）
```

### 2. FeedbackController.cs
```csharp
// 视觉/触觉反馈
- CameraShake(float intensity, float duration) // 相机震动
- HapticFeedback(HapticType type) // 手机震动
- FlashEffect(Color color) // 屏幕闪光
- SlowMotion(float duration, float scale) // 慢动作
```

### 3. TileAnimations.cs（扩展TileController）
```csharp
// 高级动画
- PunchScale() // 弹性缩放
- Rainbow() // 彩虹色过渡（高分瓦片）
- Glow() // 发光效果
- TrailEffect() // 移动拖尾
```

### 4. ComboEffects.cs
```csharp
// 连击特效
- ShowComboText(int combo, Vector3 position)
  * combo 2-3: "Good!"
  * combo 4-5: "Great!"
  * combo 6-7: "Awesome!"
  * combo 8+: "PERFECT!"
- ComboMultiplier动画（x2, x3显示）
```

要求：
- 使用TextMesh Pro生成浮动文字
- 粒子系统预制件化
- 特效对象池复用
- 性能优化（限制同屏粒子数）
```

---

## 🔄 第三步：迭代优化 Prompt

### 3.1 性能优化
```
分析当前代码性能瓶颈并优化。参考 @TECH_SPECS.md 性能要求。

## 优化检查清单

1. 内存优化
   - 检查是否有内存泄漏（事件未取消订阅）
   - 对象池使用情况（TileController, 粒子, AudioSource）
   - 纹理压缩和大小
   - 减少GC Alloc（避免频繁装箱、字符串拼接）

2. CPU优化
   - 移除Update中的GetComponent
   - 缓存Transform引用
   - 避免foreach（使用for循环）
   - 协程替代Update轮询

3. 渲染优化
   - 合并材质（Atlas）
   - 减少DrawCall
   - UI Canvas分层（静态/动态）
   - Overdraw检查

4. 启动优化
   - 异步场景加载
   - 延迟初始化非关键系统
   - 预加载优化

请生成优化后的代码和优化报告。
```

### 3.2 代码审查
```
对现有代码进行全面审查。参考 @TECH_SPECS.md 编码规范。

## 审查维度

1. 代码质量
   - 命名规范检查
   - 注释完整性
   - 单一职责原则
   - 代码复用

2. 架构问题
   - 耦合度（是否过度依赖）
   - 扩展性（新增功能容易吗）
   - 可测试性
   - 事件系统使用

3. 潜在Bug
   - 空引用检查
   - 边界条件处理
   - 并发问题
   - 状态机遗漏状态

4. 安全性
   - 存档加密
   - 防作弊措施
   - 内购验证

请生成详细审查报告和修复建议。
```

---

## 🚀 第四步：一键部署 Prompt

### 4.1 打包配置
```
配置Unity项目为iOS和Android打包做准备。

## 需要生成

### 1. BuildSettings.cs（编辑器脚本）
```csharp
// 自动化打包配置
- SetupIOSBuild() // iOS设置
  * Bundle ID
  * 版本号
  * 签名配置
  * Info.plist配置
  
- SetupAndroidBuild() // Android设置
  * Package Name
  * 版本号（versionCode + versionName）
  * Keystore配置
  * AndroidManifest权限

- BuildGame(BuildTarget target, string path)
  * 自动选择场景
  * 压缩设置
  * 剥离引擎代码
```

### 2. 版本管理
```csharp
public static class GameVersion
{
    public const string VERSION = "1.0.0";
    public const int BUILD_NUMBER = 1;
    public const string BUNDLE_ID = "com.yourstudio.2048pro";
}
```

### 3. 配置检查清单
- Player Settings完整性
- Ads/IAP ID配置
- Icon和Splash Screen
- 权限声明（相机、麦克风等）
```

---

## 📱 第五步：测试和调试 Prompt

### 5.1 单元测试生成
```
为核心系统生成单元测试。

使用Unity Test Framework生成测试：

1. GridSystemTests.cs
   - 测试网格初始化
   - 测试瓦片设置/获取
   - 测试空格检测
   - 测试合并检测

2. MoveProcessorTests.cs
   - 测试各方向移动
   - 测试合并逻辑
   - 测试得分计算
   - 边界情况测试

3. SaveLoadTests.cs
   - 测试保存功能
   - 测试加载功能
   - 测试损坏数据处理

要求：
- 使用NUnit框架
- 100%覆盖核心逻辑
- 包含边界和异常测试
```

### 5.2 调试工具
```
创建开发调试工具。

## 生成 DebugManager.cs

功能：
1. 作弊菜单（仅Editor和Development Build）
   - 设置分数
   - 生成指定数字瓦片
   - 清除网格
   - 解锁全部关卡

2. 性能监控
   - FPS显示
   - 内存使用
   - DrawCall计数
   - GC Alloc统计

3. 日志系统
   - 彩色日志（Info/Warning/Error）
   - 日志保存到文件
   - 上传Crash报告

4. 调试UI
   - On-screen console
   - 触摸热区显示
   - 网格调试可视化
```

---

## 💡 第六步：AI辅助内容生成 Prompt

### 6.1 关卡数据生成
```
生成100个关卡的配置数据。

## 关卡设计规则

### LevelData.json 结构
```json
{
  "levels": [
    {
      "id": 1,
      "targetScore": 1000,
      "maxMoves": 50,
      "initialTiles": [
        {"x": 0, "y": 0, "value": 2},
        {"x": 1, "y": 1, "value": 2}
      ],
      "difficulty": "easy",
      "starThresholds": [800, 1000, 1200]
    }
  ]
}
```

### 难度曲线
- Level 1-10: 简单（目标分1000-2000）
- Level 11-30: 普通（目标分2000-5000）
- Level 31-60: 困难（目标分5000-10000）
- Level 61-100: 专家（目标分10000+）

请生成完整的100关配置JSON，确保难度平滑上升。
```

### 6.2 文案生成
```
为游戏生成所有文案内容（中英文）。

## 需要的文案

### 1. UI文案
- 按钮文字（开始/继续/设置/商店等）
- 提示文字
- 教程文案
- 成就描述

### 2. 营销文案
- App Store描述（英文）
- 关键词（ASO优化）
- 更新日志模板
- 社交媒体分享文案

### 3. 本地化
- 生成LocalizationData.json
- 支持语言：中文、英文、日文、韩文
- 使用键值对结构

格式：
```json
{
  "ui.button.start": {
    "zh": "开始游戏",
    "en": "Start Game",
    "ja": "ゲーム開始",
    "ko": "게임 시작"
  }
}
```
```

---

## 🎨 第七步：美术资源 AI 生成指令

### 7.1 UI图标生成（Midjourney/DALL-E）
```
Midjourney Prompts：

1. 游戏Logo
"minimalist 2048 game logo, geometric number blocks, gradient blue to purple, modern flat design, white background, app icon style --ar 1:1 --v 6"

2. 数字瓦片
"set of colorful number tiles for puzzle game, 2 4 8 16 32 64 128 256, pastel gradient colors, rounded corners, subtle shadow, flat design --ar 4:1"

3. UI按钮
"game UI button set, play pause restart home icons, modern minimalist style, blue gradient, glass morphism effect --ar 4:1"

4. 背景
"abstract geometric pattern background, soft gradient, subtle grid lines, mobile game style, calming colors --ar 9:16"
```

### 7.2 音效生成（Eleven Labs/AIVA）
```
音效需求列表：

1. UI音效
- button_click.wav（清脆点击声，100ms）
- panel_open.wav（柔和展开音，200ms）
- panel_close.wav（快速关闭音，150ms）

2. 游戏音效
- tile_move.wav（滑动声，150ms）
- tile_merge.wav（合并爆裂声，200ms）
- score_up.wav（得分铃声，300ms）
- combo.wav（连击音效，500ms，上升音调）
- game_over.wav（失败音，1s）

3. 背景音乐
- menu_theme.mp3（轻松欢快，循环2分钟）
- game_theme.mp3（专注氛围，循环3分钟）

使用AI音乐生成工具（AIVA/Soundraw）参数：
- 风格：Casual Game / Puzzle
- 情绪：Playful, Energetic
- BPM：120-140
```

---

## 🔥 全自动开发超级 Prompt（直接复制使用）

```markdown
# 🎮 2048 游戏全自动开发任务

你是Unity高级开发专家。我需要你完全自动化开发一款2048变体手游。

## 项目上下文
- 游戏类型：2048数字合成益智游戏
- 平台：iOS + Android
- Unity版本：2022.3 LTS
- 架构：MVC + 事件驱动
- 参考文档：@PROJECT_STRUCTURE.md @TECH_SPECS.md

## 自动执行步骤

### Phase 1: 项目初始化（5分钟）
1. 创建完整文件夹结构
2. 生成所有管理器基类（GameManager, UIManager, AudioManager, DataManager）
3. 创建核心数据结构和枚举
4. 设置Git仓库和.gitignore

### Phase 2: 核心玩法（30分钟）
1. 实现GridSystem（4x4网格逻辑）
2. 实现TileController（瓦片显示和动画）
3. 实现InputHandler（滑动检测）
4. 实现MoveProcessor（移动和合并算法）
5. 实现ScoreSystem（计分系统）
6. 对象池优化

### Phase 3: UI系统（20分钟）
1. 实现所有UI面板（MainMenu, Gameplay, Pause, GameOver）
2. UI动画和转场效果
3. 响应式布局适配

### Phase 4: 数据持久化（10分钟）
1. 实现SaveLoadSystem
2. 实现自动存档
3. 配置管理

### Phase 5: 变现系统（15分钟）
1. 集成Unity Ads框架
2. 实现AdsManager
3. 实现IAPManager
4. 广告展示逻辑

### Phase 6: 音效和特效（15分钟）
1. 完善AudioManager
2. 实现ParticleManager
3. 实现FeedbackController
4. 连击特效

### Phase 7: 打包和优化（10分钟）
1. 性能优化（对象池、GC优化）
2. 打包配置
3. 生成Build脚本

### Phase 8: 内容生成（10分钟）
1. 生成100关配置JSON
2. 生成本地化文案
3. 生成测试数据

## 执行要求
- 每个文件都要完整可运行，包含所有using和命名空间
- 遵循C#最佳实践和Unity规范
- 所有public方法都要有XML注释
- 关键逻辑添加性能优化注释
- 代码要优雅、可读、可维护

## 输出格式
每完成一个Phase，输出：
1. 生成的文件清单
2. 关键代码片段
3. 使用说明
4. 潜在问题和解决方案

现在立即开始 Phase 1！
```

---

## 📋 使用检查清单

### 开发前
- [ ] 安装Unity 2022.3 LTS
- [ ] 安装Cursor + MCP
- [ ] 创建项目文档（PROJECT_STRUCTURE.md, TECH_SPECS.md）
- [ ] 准备美术AI工具账号

### 开发中
- [ ] 按Phase顺序执行Prompt
- [ ] 每个Phase完成后测试
- [ ] 及时提交Git（每个Phase一次commit）
- [ ] 记录遇到的问题和解决方案

### 开发后
- [ ] 完整性测试（所有功能）
- [ ] 性能测试（FPS、内存、启动时间）
- [ ] 真机测试（iOS + Android）
- [ ] 提交App Store审核

---

## 💡 高级技巧

### Cursor快捷操作
- `Cmd/Ctrl + K`：快速生成代码
- `Cmd/Ctrl + L`：多行编辑
- `@filename`：引用特定文件作为上下文
- `#selection`：对选中代码操作

### MCP最佳实践
- 使用MCP File System读取项目文件作为上下文
- 用MCP Git操作自动提交
- 用MCP Search检索Unity文档

### Prompt优化技巧
- 明确指定输出格式（JSON、代码、文档）
- 提供具体示例和参考
- 分阶段执行，避免一次性生成过多代码
- 使用"请检查XXX并修复"进行迭代优化

---

## 🎯 成功指标

完成后你应该有：
- ✅ 可运行的Unity项目（无编译错误）
- ✅ 完整的游戏循环（菜单→游戏→结束）
- ✅ 存档系统正常工作
- ✅ 广告和内购代码集成（待后台配置）
- ✅ 性能达标（60FPS，<150MB内存）
- ✅ 真机测试通过

预计开发时间：**2-3天**（使用AI辅助）
传统开发时间：2-3周

---

祝开发顺利！有问题随时调整Prompt继续迭代。🚀
