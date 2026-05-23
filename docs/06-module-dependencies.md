# 模块依赖与运行时调用顺序

## 1. 模块概览

`src/Core/` 下共 **46 个模块目录**，按职责分为五类：

| 分类 | 模块 | 职责 |
|------|------|------|
| **入口/UI** | `Nodes` | Godot 场景控制器 (NGame, NRun, NCreature...)，UI 组件 |
| **系统层** | `Runs`, `Combat`, `GameActions`, `Commands`, `Saves` | 核心游戏逻辑编排 |
| **数据层** | `Models`, `Entities`, `Rooms`, `Map`, `Rewards`, `Timeline` | 游戏数据定义与运行时状态 |
| **基础设施** | `Assets`, `Multiplayer`, `Localization`, `Platform`, `Modding` | 跨切面服务 |
| **工具层** | `Helpers`, `Logging`, `Random`, `Extensions`, `ValueProps` | 通用工具与数学 |

---

## 2. 分层依赖架构

系统呈现**单向分层依赖**，上层依赖下层，下层不依赖上层：

```
┌──────────────────────────────────────────────────────────────────────────┐
│  Layer 0: 入口/UI (Nodes)                                                │
│  依赖: 全部下层模块                                                       │
├──────────────────────────────────────────────────────────────────────────┤
│  Layer 1: 系统层 (Runs, Combat, GameActions, Commands, Saves, Hooks)     │
│  依赖: 数据层 + 基础设施 + 工具层                                          │
├──────────────────────────────────────────────────────────────────────────┤
│  Layer 2: 数据层 (Models, Entities, Rooms, Map, Rewards, Timeline...)   │
│  依赖: 基础设施 + 工具层                                                  │
├──────────────────────────────────────────────────────────────────────────┤
│  Layer 3: 基础设施 (Assets, Multiplayer, Localization, Platform,        │
│                     Modding, Achievements, Leaderboard, Daily...)        │
│  依赖: 工具层                                                            │
├──────────────────────────────────────────────────────────────────────────┤
│  Layer 4: 工具层 (Helpers, Logging, Random, Extensions, Settings,       │
│                     ValueProps, Exceptions, TestSupport)                 │
│  依赖: 无 (纯工具类/枚举)                                                 │
└──────────────────────────────────────────────────────────────────────────┘
```

### 依赖流向规则

- **不允许**下层模块引用上层模块
- 跨层引用通过 `Hook` 事件系统解耦（数据模型向系统层发事件）
- 模块间通过**静态单例**访问（无 DI 容器）

---

## 3. 核心依赖关系

### 3.1 系统层内部依赖

```
RunManager (Run 管理器)
  ├── CombatManager → 战斗循环
  ├── SaveManager → 存档
  ├── ActionExecutor + ActionQueueSet → 动作队列
  ├── Multiplayer synchronizers → 多人同步
  └── Hook → 事件调度

CombatManager (战斗管理器)
  ├── GameActions → 玩家动作 (PlayCardAction 等)
  ├── Commands → 效果命令 (DamageCmd, CreatureCmd 等)
  ├── Hooks → 战斗事件分发
  └── Rooms → 房间状态

Commands (命令层)
  ├── Assets, Audio → 资源/音效
  ├── Models, Entities → 数据操作
  ├── Hooks → 事件触发
  └── ValueProps → 属性计算
```

### 3.2 数据层内部依赖

```
Models (数据定义)
  ├── Entities (运行时包装器)
  ├── Rooms (房间逻辑)
  └── Rewards (奖励定义)

Entities (运行时状态)
  ├── Models (引用数据定义)
  └── (无其他模块依赖)

Multiplayer synchronizers
  ├── Runs → 同步 Run 状态
  ├── Combat → 同步战斗状态
  └── SaveManager → 保存/加载
```

### 3.3 基础设施依赖

```
Assets (资源管理)
  ├── PreloadManager → 分阶段加载
  ├── NAssetLoader → 后台加载
  └── AtlasManager → 图集管理

Multiplayer (多人系统)
  ├── NetSingleplayerGameService (单机)
  ├── NetHostGameService (主机)
  ├── NetClientGameService (客户端)
  └── NetReplayGameService (回放)
       └── 均依赖 synchronizers (CombatState, PlayerChoice, Event, Reward...)
```

---

## 4. 运行时调用顺序

### 4.1 游戏启动流程

```
project.godot (run/main_scene = game.tscn)
  │
  ├── Autoload 注册:
  │   ├── SentryInit (错误追踪)
  │   ├── OneTimeInitialization (首次初始化)
  │   ├── AssetLoader (后台资源加载)
  │   ├── DevConsole (开发者控制台)
  │   ├── CommandHistory (命令历史)
  │   ├── MemoryMonitor (内存监控)
  │   └── FmodManager (FMOD 音频)
  │
  └── NGame._EnterTree()
        └── GameStartup()
              ├── (1) Platform 初始化
              │     └── SteamInitializer, GitHelper
              ├── (2) 数据迁移
              │     └── SaveManager 初始化 + 云存档同步
              ├── (3) 对象池初始化
              │     └── NCard.InitPool(), NGridCardHolder.InitPool()
              ├── (4) 核心子系统初始化 (OneTimeInitialization)
              │     ├── ModManager.Initialize()
              │     ├── LocManager.Initialize()
              │     ├── ModelDb.Init() + InitIds()
              │     ├── AtlasManager.LoadEssentialAtlases()
              │     └── ModelIdSerializationCache.Init()
              ├── (5) 显示/音频设置
              │     ├── InitializeGraphicsPreferences()
              │     └── AudioManager.SetVolumes()
              ├── (6) Steam 服务
              │     ├── LeaderboardManager.Initialize()
              │     └── SteamStatsManager.Initialize()
              ├── (7) 存档加载 (等待云同步)
              │     ├── SaveManager.Instance.InitProfileId()
              │     ├── SaveManager.Instance.InitProgressData()
              │     └── SaveManager.Instance.InitPrefsData()
              ├── (8) Logo 动画 (可选)
              │     └── NLogoAnimation → Spine 动画播放
              ├── (9) 主菜单
              │     ├── NMainMenu.Create()
              │     └── RootSceneContainer.SetCurrentScene()
              └── (10) 后台资源加载
                    ├── OneTimeInitialization.ExecuteDeferred()
                    └── PreloadManager.LoadCommonAndMainMenuAssets()
```

### 4.2 Run 启动流程

```
MainMenu → CharacterSelect → NGame.StartNewSingleplayerRun()
  │
  ├── (1) RunState 创建
  │     └── RunState.CreateForNewRun(character, ascension)
  │
  ├── (2) RunManager 初始化
  │     ├── SetUpNewSinglePlayer()
  │     ├── InitializeShared()
  │     │     ├── NetService (NetSingleplayerGameService)
  │     │     ├── ChecksumTracker, RunLocationTargetedBuffer
  │     │     ├── FlavorSynchronizer
  │     │     ├── ActionQueueSet + ActionExecutor
  │     │     ├── 同步器群: ActionQueueSynchronizer,
  │     │     │   PlayerChoiceSynchronizer, MapSelectionSynchronizer,
  │     │     │   ActChangeSynchronizer, EventSynchronizer,
  │     │     │   RewardSynchronizer, RestSiteSynchronizer,
  │     │     │   OneOffSynchronizer, TreasureRoomRelicSynchronizer
  │     │     ├── CombatReplayWriter
  │     │     ├── AscensionManager
  │     │     └── HoveredModelTracker, InputSynchronizer
  │     ├── InitializeNewRun()
  │     │     ├── 填充遗物抽奖袋
  │     │     ├── 应用 Ascension 修改器
  │     │     └── GenerateRooms() → 为每幕生成遭遇集
  │     └── FinalizeStartingRelics()
  │
  ├── (3) 资源加载
  │     ├── PreloadManager.LoadRunAssets(characters)
  │     └── PreloadManager.LoadActAssets(act 0)
  │
  ├── (4) Run 场景创建
  │     ├── NRun.Create(state)
  │     └── RootSceneContainer.SetCurrentScene(nRun)
  │
  ├── (5) 进入第 0 幕
  │     ├── GenerateMap() → ActMap.CreateMap() → Hook 修改 → 显示
  │     └── EnterRoom(MapRoom) → 玩家看到地图
  │
  └── (6) 房间循环
        ├── 玩家点击地图节点 → EnterMapCoord()
        ├── ExitCurrentRooms() + CreateRoom()
        └── EnterRoom(room) → 按房间类型进入不同逻辑
```

### 4.3 战斗生命周期

```
CombatRoom.Enter()
  │
  ├── CombatManager.SetUpCombat(state)
  │     ├── StateTracker.SetState(state)
  │     ├── player.ResetCombatState()
  │     ├── player.PopulateCombatState() → 初始抽牌
  │     ├── AddCreature (所有敌方生物)
  │     └── CombatSetUp 事件
  │
  ├── NCombatRoom._Ready()
  │     └── CombatManager.AfterCombatRoomLoaded()
  │
  └── CombatManager.StartCombatInternal()
        │
        ├── 战前: AfterCreatureAdded → Hook.BeforeCombatStart → Banner
        │
        └── StartTurn() [回合循环]
              │
              ├── 玩家回合:
              │     ├── BeforeTurnStart (Creature)
              │     ├── Hook.BeforeSideTurnStart
              │     ├── SetupPlayerTurn() → 抽牌 + 能量
              │     ├── Hook.AfterSideTurnStart
              │     ├── PlayPhase (等待玩家动作):
              │     │     ├── PlayCardAction → CardModel.OnPlay()
              │     │     │     ├── Hook.BeforeCardPlayed
              │     │     │     ├── CardModel.OnPlay() (卡牌效果)
              │     │     │     │     └── DamageCmd/CreatureCmd/PowerCmd/RelicCmd/VfxCmd/SfxCmd
              │     │     │     │           └── Hook.BeforeAttack/Hook.AfterAttack
              │     │     │     └── Hook.AfterCardPlayed
              │     │     ├── EndPlayerTurnAction → 回合结束
              │     │     └── UsePotionAction → 使用药水
              │     └── 切换: 弃牌 → 清理 → SwitchSides → Enemy
              │
              └── 敌人回合:
                    ├── Hook.BeforeSideTurnStart
                    ├── ExecuteEnemyTurn() → 执行所有怪物 Intent
                    │     └── NCreature.PerformIntent() + enemy.TakeTurn()
                    │           └── (内部同样通过 Commands + Hook)
                    ├── 清理 → SwitchSides → Player
                    └── StartTurn() [下一回合]
              │
              └── 战斗结束:
                    ├── CheckWinCondition() → 敌人全灭
                    ├── EndCombatInternal()
                    │     ├── Hook.AfterCombatEnd
                    │     ├── Hook.AfterCombatVictory (遗物触发)
                    │     ├── CombatReplayWriter.WriteReplay()
                    │     ├── SaveManager.Instance.SaveRun()
                    │     └── 成就检查
                    └── Room rewards → 返回地图
```

### 4.4 动作执行系统调用链

```
玩家输入 (点击卡牌)
  │
  └── PlayCardAction (入列 ActionQueueSet)
        │
        └── ActionExecutor.ExecuteActions() [循环]
              │
              ├── WaitForUnpause()
              ├── readyAction.Execute()
              │     └── PlayCardAction.ExecuteAction()
              │           ├── Validate: CanPlay() + IsValidTarget()
              │           ├── SpendResources() → 消耗能量
              │           ├── CardModel.OnPlayWrapper()
              │           │     ├── Hook.BeforeCardPlayed (所有 Hook 监听器)
              │           │     ├── CardModel.OnPlay() (卡牌具体效果)
              │           │     │     ├── z.B. DamageCmd.Attack()
              │           │     │     │     └── AttackCommand
              │           │     │     │           ├── Hook.BeforeAttack
              │           │     │     │           ├── 伤害计算管道:
              │           │     │     │           │     ├── PowerModel.ModifyDamageAdditive()
              │           │     │     │           │     └── PowerModel.ModifyDamageMultiplicative()
              │           │     │     │           ├── 实际扣血
              │           │     │     │           └── Hook.AfterAttack
              │           │     │     └── CreatureCmd.Heal() / PowerCmd.Add() / ...
              │           │     └── Hook.AfterCardPlayed
              │           └── 卡牌进入弃牌堆/消耗
              │
              └── Trigger AfterActionExecuted 事件
```

---

## 5. 关键架构特征

### 5.1 依赖汇聚点

- **`Models`** 是被依赖最多的模块 (28+ 模块引用) — 所有游戏内容都从这里派生
- **`Commands`** 是所有游戏效果的出口 — 任何卡牌/遗物/能力最终都通过 Commands 执行
- **`Hooks`** 是解耦核心 — 数据模型通过 Hook 向系统层发事件，避免反向依赖

### 5.2 无 DI 容器

所有依赖通过**静态单例**解析：

```csharp
// 典型用法 — 任何地方直接引用单例
RunManager.Instance.ActionExecutor.EnqueueAction(action);
CombatManager.Instance.IsInProgress;
SaveManager.Instance.Progress;
NGame.Instance.RootSceneContainer;
LocManager.Instance.GetString(key);
ModelDb.GetId(typeof(StrengthPower));
```

### 5.3 多人同步模式

每个关键子系统有对应同步器：

| 子系统 | 同步器 |
|--------|--------|
| 战斗状态 | `CombatStateSynchronizer` |
| 玩家选择 | `PlayerChoiceSynchronizer` |
| 动作队列 | `ActionQueueSynchronizer` |
| 地图选择 | `MapSelectionSynchronizer` |
| 幕切换 | `ActChangeSynchronizer` |
| 事件 | `EventSynchronizer` |
| 奖励 | `RewardSynchronizer` |
| 休息处 | `RestSiteSynchronizer` |
| 一次性同步 | `OneOffSynchronizer` |
| 宝箱遗物 | `TreasureRoomRelicSynchronizer` |

### 5.4 数据流向

```
Models (静态数据定义)
  → Entities (运行时实例 + 可变状态)
    → Hook 事件 (数据 → 系统)
      → Commands (系统 → 数据/UI)
        → Nodes (UI 更新 + 动画)
```

---

## 6. 模块引用关系一览

| 模块 | 被引用次数 | 核心引用方 |
|------|-----------|-----------|
| Models | 28+ | 几乎所有模块 |
| Runs | 18+ | Combat, Commands, Nodes, Rooms, Rewards |
| Commands | 16+ | Combat, GameActions, Nodes, Rooms |
| Saves | 16+ | Runs, Nodes, Combat, Multiplayer |
| Nodes | 14+ | Combat, Commands, Rooms, Runs |
| Helpers | 14+ | 几乎所有模块 |
| Entities | 13+ | Models, Combat, Commands, Hooks |
| Hooks | 12+ | Commands, Combat, GameActions, Rooms |
| Combat | 11+ | Runs, Commands, Nodes, Rooms |
| Rooms | 11+ | Commands, Runs, Nodes |
