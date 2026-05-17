# 项目概览与目录结构

## 1. 项目简介

**Slay the Spire 2** 是 Mega Crit Games 开发的卡牌 Roguelike 游戏续作，使用 **Godot 4.5** 引擎 + **C# (.NET 9.0)** 编写。

### 技术栈

- **引擎**: Godot 4.5
- **主语言**: C# (3,279 `.cs` 文件)
- **副语言**: GDScript (48 `.gd` 文件，主要用于插件/测试)
- **音频**: FMOD
- **错误追踪**: Sentry
- **本地化**: Weblate 社区翻译（11+ 语言）
- **多人游戏**: 点对点 (P2P) 联机
- **Mod 支持**: 内置

### 命名空间

所有核心代码位于 `MegaCrit.Sts2.Core` 命名空间下。

---

## 2. 目录结构详解

```
STS2SourceCode/
├── addons/                        # Godot 编辑器插件
│   ├── fmod/                      # FMOD 音频集成插件
│   │   ├── FmodManager.gd         # 运行时管理器 (Autoload)
│   │   ├── FmodPlugin.gd          # Godot 编辑器插件入口
│   │   └── tool/                  # 编辑器工具 (Bank Explorer, UI)
│   ├── sentry/                    # Sentry 错误追踪
│   │   └── SentryInit.gd          # 自动初始化
│   ├── mega_text/                 # 自定义富文本标签
│   ├── megacontentcreator/        # 内容创建工具
│   ├── atlas_generator/           # 纹理图集生成
│   └── dev_tools/                 # 开发者工具
│
├── animations/                    # 动画资源
│   ├── backgrounds/               # 背景动画
│   ├── characters/                # 角色动画
│   ├── monsters/                  # 怪物动画
│   ├── map/                       # 地图动画
│   └── vfx/                       # 视觉特效动画
│
├── banks/                         # FMOD 音频银行
│
├── fonts/                         # 字体文件
│
├── images/                        # 游戏美术资源
│   ├── atlases/                   # 纹理图集
│   ├── characters/                # 角色立绘
│   ├── monsters/                  # 怪物立绘
│   ├── cards/                     # 卡牌图像
│   ├── relics/                    # 遗物图标
│   ├── potions/                   # 药水图标
│   ├── powers/                    # 能力图标
│   ├── ui/                        # UI 元素
│   ├── vfx/                       # 特效纹理
│   └── packed/                    # 打包后的压缩纹理
│
├── localization/                  # 本地化文件
│   ├── eng/                       # 英文 (源语言)
│   ├── deu/esp/ita/jpn/...        # 其他语言
│   └── README.md
│
├── materials/                     # 材质与着色器材质
│   └── transitions/               # 场景过渡材质
│
├── models/                        # 3D 模型
│   └── vfx/                       # 3D 特效模型
│
├── scenes/                        # 【场景层】- 所有 Godot 场景
│   ├── game.tscn                  # ★ 主入口场景 (run/main_scene)
│   ├── run.tscn                   # ★ Run (游戏会话) 场景
│   ├── scene_container.tscn       # 场景容器
│   ├── asset_loader.tscn          # 资源加载器 (Autoload)
│   ├── one_time_initialization.tscn # 首次初始化 (Autoload)
│   │
│   ├── backgrounds/               # 背景场景
│   ├── cards/                     # 卡牌场景 (card.tscn, card_grid.tscn, etc.)
│   │   ├── card.tscn              # 卡牌主场景
│   │   ├── holders/               # 卡牌持有者
│   │   └── overlays/              # 卡牌覆盖层
│   ├── combat/                    # 战斗UI场景 (23 scenes)
│   │   ├── combat_ui.tscn         # 战斗主 UI
│   │   ├── creature.tscn          # 生物显示
│   │   ├── player_hand.tscn       # 手牌
│   │   ├── targeting_arrow.tscn   # 瞄准箭头
│   │   └── ...
│   ├── creature_visuals/          # 生物视觉 (127 个场景)
│   ├── encounters/                # 遭遇战定义 (25 场景)
│   ├── events/                    # 事件场景
│   ├── ftue/                      # 新手教程 (13 场景)
│   ├── merchant/                  # 商人场景
│   ├── orbs/                      # 球体系统 (Defect 职业)
│   ├── potions/                   # 药水 UI
│   ├── relics/                    # 遗物 UI
│   ├── rest_site/                 # 休息处场景
│   ├── rewards/                   # 奖励 UI
│   ├── rooms/                     # 房间场景 (6 种)
│   ├── screens/                   # 所有屏幕 (48+)
│   │   ├── main_menu.tscn
│   │   ├── character_select_screen.tscn
│   │   ├── map/
│   │   ├── settings_screen.tscn
│   │   ├── game_over_screen.tscn
│   │   ├── card_library/
│   │   ├── bestiary/
│   │   ├── deck_view_screen.tscn
│   │   └── ...
│   ├── ui/                        # 共用 UI 组件 (58 场景)
│   ├── vfx/                       # 视觉特效 (152 场景)
│   └── debug/                     # 调试工具 (16 场景)
│
├── shaders/                       # Godot 着色器 (49 文件)
│
├── src/                           # 【源码层】- 所有 C# 源码
│   └── Core/                      # 核心游戏代码
│       │
│       ├── Nodes/                 # ★ Godot 场景控制器
│       │   ├── NGame.cs           # 入口控制器 (Singleton)
│       │   ├── NRun.cs            # Run 控制器 (Factory)
│       │   ├── NSceneContainer.cs # 场景栈容器
│       │   ├── NAssetLoader.cs    # 资源加载器
│       │   ├── NTransition.cs     # 过渡动画
│       │   ├── Cards/             # 卡牌节点
│       │   ├── Combat/            # 战斗 UI 节点
│       │   │   ├── NCombatUi.cs
│       │   │   ├── NCreature.cs
│       │   │   ├── NPlayerHand.cs
│       │   │   ├── NTargetManager.cs
│       │   │   └── ...
│       │   ├── Rooms/             # 房间节点
│       │   ├── Screens/           # 屏幕节点
│       │   ├── CommonUi/          # 通用 UI (Input Manager, TopBar, etc.)
│       │   ├── Vfx/               # 特效节点 (~290)
│       │   ├── Potions/           # 药水节点
│       │   ├── Relics/            # 遗物节点
│       │   ├── Orbs/              # 球体节点
│       │   └── ...
│       │
│       ├── Models/                # ★ 数据模型 (Model 层)
│       │   ├── AbstractModel.cs   # 所有游戏数据模型基类
│       │   ├── ModelDb.cs         # 模型注册表
│       │   ├── CardModel.cs       # 卡牌模型
│       │   ├── RelicModel.cs      # 遗物模型
│       │   ├── PowerModel.cs      # 能力模型
│       │   ├── PotionModel.cs     # 药水模型
│       │   ├── EncounterModel.cs  # 遭遇模型
│       │   ├── CharacterModel.cs  # 角色模型
│       │   ├── MonsterModel.cs    # 怪物模型
│       │   ├── EventModel.cs      # 事件模型
│       │   ├── ActModel.cs        # 幕 (Act) 模型
│       │   ├── Cards/             # 具体卡牌定义 (数百个)
│       │   ├── Relics/            # 具体遗物定义 (~290)
│       │   ├── Powers/            # 具体能力定义 (~260)
│       │   ├── Potions/           # 具体药水定义
│       │   ├── Monsters/          # 具体怪物定义
│       │   ├── Characters/        # 具体角色定义
│       │   └── ...
│       │
│       ├── Combat/                # ★ 战斗系统
│       │   ├── CombatManager.cs   # 战斗管理器 (Singleton)
│       │   ├── CombatState.cs     # 战斗状态
│       │   ├── CombatSide.cs      # 战斗方枚举
│       │   └── History/           # 战斗历史记录
│       │
│       ├── Rooms/                 # ★ 房间状态层
│       │   ├── AbstractRoom.cs    # 房间基类
│       │   ├── CombatRoom.cs      # 战斗房间
│       │   ├── EventRoom.cs       # 事件房间
│       │   ├── MerchantRoom.cs    # 商人房间
│       │   ├── RestSiteRoom.cs    # 休息处房间
│       │   ├── TreasureRoom.cs    # 宝箱房间
│       │   └── MapRoom.cs         # 地图房间
│       │
│       ├── Runs/                  # ★ Run 管理系统
│       │   ├── RunManager.cs      # Run 管理器 (Singleton)
│       │   ├── RunState.cs        # Run 状态
│       │   └── History/           # Run 历史
│       │
│       ├── GameActions/           # ★ 玩家动作系统
│       │   ├── GameAction.cs      # 动作基类 (抽象)
│       │   ├── ActionExecutor.cs  # 动作执行器
│       │   ├── PlayCardAction.cs  # 出牌动作
│       │   ├── EndPlayerTurnAction.cs
│       │   ├── UsePotionAction.cs
│       │   └── Multiplayer/       # 多人同步动作
│       │
│       ├── Commands/              # ★ 命令系统（动画/效果/等待）
│       │   ├── Cmd.cs             # 通用命令 (Wait)
│       │   ├── DamageCmd.cs       # 伤害命令
│       │   ├── CardCmd.cs         # 卡牌命令
│       │   ├── CreatureCmd.cs     # 生物命令
│       │   ├── PowerCmd.cs        # 能力命令
│       │   ├── RelicCmd.cs        # 遗物命令
│       │   ├── VfxCmd.cs          # 特效命令
│       │   ├── SfxCmd.cs          # 音效命令
│       │   └── Builders/          # 命令构建器 (AttackCommand)
│       │
│       ├── Hooks/                 # ★ 事件钩子系统
│       │   ├── Hook.cs            # 所有 Hook 入口 (静态方法)
│       │   ├── ModifyDamageHookType.cs
│       │   └── ...
│       │
│       ├── Entities/              # 实体层 (运行时对象)
│       │   ├── Creatures/         # 生物实体
│       │   ├── Cards/             # 卡牌相关 (CardPile, etc.)
│       │   ├── Players/           # 玩家实体
│       │   ├── Powers/            # 能力枚举
│       │   ├── Relics/            # 遗物枚举
│       │   ├── Potions/           # 药水枚举
│       │   └── ...
│       │
│       ├── Map/                   # 地图生成系统
│       ├── Saves/                 # 存档系统
│       ├── Multiplayer/           # 多人游戏系统
│       ├── Localization/          # 本地化系统
│       ├── Settings/              # 设置系统
│       ├── Rewards/               # 奖励生成
│       ├── Modding/               # Mod 系统
│       ├── Achievements/          # 成就系统
│       ├── Timeline/              # 解锁时间线
│       ├── Unlocks/               # 解锁系统
│       ├── Random/                # RNG 系统
│       ├── Assets/                # 资源管理
│       ├── Animation/             # 动画辅助
│       ├── ValueProps/            # 响应式属性
│       ├── Logging/               # 日志系统
│       └── Platform/              # 平台抽象 (Steam, etc.)
│
├── project.godot                  # Godot 项目配置
├── sts2.csproj                    # C# 项目文件
├── sts2.sln                       # Visual Studio 方案
├── global.json                    # .NET SDK 配置 (9.0.303)
└── release_info.json              # 版本发布信息
```

---

## 3. 关键文件说明

### 入口与生命周期

| 文件 | 角色 |
|------|------|
| `project.godot` | 配置 `run/main_scene = scenes/game.tscn`，注册 Autoload |
| `scenes/game.tscn` | 根场景，挂载 `NGame.cs` |
| `src/Core/Nodes/NGame.cs` | 全局单例控制器，负责初始化所有子系统 |
| `scenes/run.tscn` | Run 会话场景，挂载 `NRun.cs` |
| `scenes/scene_container.tscn` | 场景容器，挂载 `NSceneContainer.cs` |
| `scenes/asset_loader.tscn` | Autoload，后台资源加载 |
| `scenes/one_time_initialization.tscn` | Autoload，首次初始化 |

### Autoload 注册

在 `project.godot` 中注册的 Autoload：
- `SentryInit` - 错误追踪
- `OneTimeInitialization` - 首次初始化
- `AssetLoader` - 后台资源加载
- `DevConsole` - 开发者控制台
- `CommandHistory` - 命令历史
- `MemoryMonitor` - 内存监控
- `FmodManager` - FMOD 音频管理
