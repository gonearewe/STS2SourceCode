# 架构分析

## 1. 总体分层架构

系统采用**四层架构**，数据从底层到顶层单向流动：

```
┌──────────────────────────────────────────────────────────────┐
│  场景层 (Scenes / Visual Layer)                              │
│  .tscn 场景文件 + N* Node 脚本（NGame, NRun, NCreature...）   │
│  职责: 渲染、输入处理、动画播放                               │
├──────────────────────────────────────────────────────────────┤
│  房间/屏幕层 (Rooms / Screens / State Layer)                  │
│  AbstractRoom → CombatRoom, EventRoom...                     │
│  AbstractScreen → NMainMenu, NMapScreen...                   │
│  职责: 房间逻辑、房间切换、全局 UI 管理                       │
├──────────────────────────────────────────────────────────────┤
│  实体/模型层 (Entities / Models / Data Layer)                │
│  AbstractModel → CardModel, RelicModel, PowerModel...        │
│  Entity: Creature, Player, CardPile...                       │
│  职责: 游戏数据定义、运行时状态、Hook 监听                   │
├──────────────────────────────────────────────────────────────┤
│  系统层 (Systems / Logic Layer)                               │
│  CombatManager, RunManager, ActionExecutor, Hook...          │
│  职责: 核心游戏逻辑、事件调度、动作队列                       │
└──────────────────────────────────────────────────────────────┘
  基础设施: Multiplayer, Saves, Settings, Modding, Animation, VFX
```

---

## 2. 核心类关系

### 2.1 启动流程

```
project.godot
  └─ run/main_scene = game.tscn
       └─ NGame._EnterTree()
            ├─ SentryService.Initialize()
            └─ TaskHelper.RunSafely(GameStartupWrapper())
                 ├─ InitializePlatform()          # Steam 初始化
                 └─ GameStartup()
                      ├─ 数据迁移
                      ├─ CloudSave 初始化
                      ├─ GamePools 初始化 (卡牌/遗物/药水)
                      ├─ OneTimeInitialization
                      ├─ 显示/音频设置应用
                      ├─ 排行榜/Steam 初始化
                      ├─ 存档加载
                      └─ LaunchMainMenu() → 过渡到主菜单
```

### 2.2 Run 生命周期

```
MainMenu → CharacterSelect → NRun.Create(RunState)
                                │
                          NRun._Ready()
                          ├─ GlobalUi.Initialize()
                          ├─ RunMusicController
                          └─ NSceneContainer → Room
                                │
                          SetCurrentRoom()
                          ├─ CombatRoom  → CombatManager.SetUpCombat()
                          ├─ EventRoom   → EventSystem
                          ├─ MerchantRoom → MerchantSystem
                          ├─ RestSiteRoom → RestSiteLogic
                          ├─ MapRoom     → MapNavigation
                          └─ TreasureRoom → TreasureLogic
```

### 2.3 战斗系统关系图

```
CombatManager (Singleton)
  │
  ├─ SetUpCombat(CombatState)
  │    ├─ 创建 Creature 列表
  │    ├─ 设置玩家战斗状态
  │    └─ 触发 CombatSetUp 事件
  │
  ├─ StartCombatInternal()   (async)
  │    └─ StartTurn() → 回合循环
  │         ├─ Player 回合:
  │         │   ├─ SetupPlayerTurn() (抽牌+获得能量)
  │         │   ├─ 等待玩家动作 (PlayCardAction / EndTurn)
  │         │   └─ EndPlayerTurn()
  │         └─ Enemy 回合:
  │             ├─ 执行怪物 AI (Intent)
  │             └─ StartTurn() → 回到玩家回合
  │
  └─ 事件驱动
       ├─ CombatSetUp / CombatEnded / CombatWon
       ├─ TurnStarted / TurnEnded
       └─ CreaturesChanged
```

### 2.4 动作执行系统

```
玩家输入 (点击卡牌)
  └─ PlayCardAction (GameAction 子类)
       ├─ 入队 (ActionQueueSet)
       └─ ActionExecutor.ExecuteActions()
            ├─ 等待 unpause
            ├─ 调用 GameAction.Execute()
            │    └─ PlayCardAction.ExecuteAction()
            │         ├─ SpendResources() (消耗能量)
            │         ├─ CardModel.OnPlayWrapper()
            │         │    ├─ Hook.BeforeCardPlayed()
            │         │    ├─ card.OnPlay() (卡牌效果)
            │         │    └─ Hook.AfterCardPlayed()
            │         └─ 收尾
            └─ 触发 AfterActionExecuted 事件
```

---

## 3. 重要设计决策

### 3.1 单例 vs 实例化

**使用单例的类**:
- `NGame.Instance` - 根控制器
- `CombatManager.Instance` - 战斗管理器
- `RunManager.Instance` - Run 管理器
- `SaveManager.Instance` - 存档管理器
- `LocManager.Instance` - 本地化管理器
- `NRun.Instance` - 当前 Run (代理到 NGame)

**使用 Factory 模式创建的类**:
- `NRun.Create(RunState)` - 创建 Run 场景
- `NCombatRoom` 通过构造函数构建
- `CombatState` 通过构造函数构建

### 3.2 场景管理

`NSceneContainer` 实现了**单场景栈**模式：

```csharp
// 清除所有子节点，设置新场景
public void SetCurrentScene(Control node) {
    foreach (Node child in GetChildren()) {
        RemoveChildSafely(child);
        child.QueueFreeSafely();
    }
    CurrentScene = node;
    AddChildSafely(node);
}
```

- 每个容器只含有一个活动场景
- 用于房间切换（`NRun._roomContainer`）和主菜单/Logo 切换（`NGame.RootSceneContainer`）
- 叠加层使用 `NOverlayStack`（用于 Game Over、检查界面等）

### 3.3 异步编程模式

大量使用 `async/await` 配合 Godot 的 SceneTreeTimer：

```
Cmd.Wait(seconds)          → 等待实际时间
Cmd.CustomScaledWait(f,s)  → 根据 FastMode 选择等待时间
```

钩子系统（Hook）完全异步：

```csharp
public static async Task BeforeAttack(CombatState combatState, AttackCommand command) {
    foreach (AbstractModel model in combatState.IterateHookListeners()) {
        await model.BeforeAttack(command);
        model.InvokeExecutionFinished();
    }
}
```

动作执行器使用异步队列：

```csharp
private async Task ExecuteActions() {
    GameAction readyAction = _actionQueueSet.GetReadyAction();
    while (readyAction != null) {
        await WaitForUnpause();
        await readyAction.Execute();
        readyAction = _actionQueueSet.GetReadyAction();
    }
}
```

### 3.4 网络同步

多人模式通过在本地动作上叠加 `Net*` 包装器实现同步：

- `PlayCardAction` → `NetPlayCardAction`
- `EndPlayerTurnAction` → `NetEndPlayerTurnAction`
- `UsePotionAction` → `NetUsePotionAction`

多人服务分为四种模式：
- `NetSingleplayerGameService` - 单机
- `NetHostGameService` - 主机
- `NetClientGameService` - 客户端
- `NetReplayGameService` - 回放

### 3.5 资源加载

`PreloadManager.Cache` 和 `NAssetLoader` 提供后台资源加载：

```csharp
NRun nRun = PreloadManager.Cache
    .GetScene("res://scenes/run.tscn")
    .Instantiate<NRun>(PackedScene.GenEditState.Disabled);
```
