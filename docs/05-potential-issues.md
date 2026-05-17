# 潜在问题与改进建议

## 1. 架构层面的潜在问题

### 1.1 单例滥用

**问题**: `NGame`, `CombatManager`, `RunManager`, `SaveManager`, `LocManager` 等大量使用单例模式。

**风险**:
- 全局可变状态，难以追踪修改来源
- 单元测试困难（mock 单例需要特殊处理）
- 隐式依赖关系 — 代码中随处可见 `CombatManager.Instance.xxx`，模块间耦合度高

**改进建议**:
- 引入依赖注入容器 (Dependency Injection)
- 或使用服务定位器 (Service Locator) + 接口抽象
- 关键路径的 Manager 应该定义接口 (`ICombatManager`, `IRunManager`)

### 1.2 Hook 系统性能开销

**问题**: `Hook.xxx()` 每次调用都遍历全部 Hook 监听器：

```csharp
public static async Task BeforeAttack(CombatState state, AttackCommand cmd) {
    foreach (var model in state.IterateHookListeners())
        await model.BeforeAttack(cmd);
}
```

**风险**:
- 每张卡牌打出、每次伤害都会触发大量虚方法调用
- `IterateHookListeners()` 收集所有模型 → 过滤 → 遍历
- 可能有上百个 PowerModel/RelicModel 同时活跃

**改进建议**:
- 增加 Hook 注册/注销机制，而非每次都遍历
- 按事件类型分类监听器（例如 `damageHooks`, `turnHooks`, `cardPlayHooks`）
- 性能影响可能不明显，但在复杂场景（多玩家、大量 Buff）可能成为瓶颈

### 1.3 异步异常处理

**问题**: 使用 `TaskHelper.RunSafely()` 包装异步方法。

```csharp
TaskHelper.RunSafely(GameStartupWrapper());
```

**风险**:
- 部分异步路径可能吞掉异常（取决于 `RunSafely` 实现）
- 非预期的 `async void` 用法
- 协程中途取消 (`CancellationToken`) 的处理路径可能不完整

**改进建议**:
- 统一使用 `TaskHelper.RunSafely` 虽好，但应检查所有 `async void` 事件处理器
- 确保 `CancellationToken` 传递到所有异步路径

### 1.4 ActionQueue 的 Pause/Resume 并发

```csharp
private async Task ExecuteActions() {
    while (readyAction != null) {
        await WaitForUnpause();  // 忙等 unpause
        // ...
    }
}
```

**风险**:
- `WaitForUnpause` 可能使用轮询或信号量实现，需要确认实现方式
- 如果有多个 Action 同时入队，竞争条件是否安全

---

## 2. 代码层面的具体问题

### 2.1 `Creature.cs` 属性安全检查

```csharp
public int Block {
    set {
        if (value < 0) throw new ArgumentException("Block must be positive");
    }
}
```

**问题**: `CurrentHp` 也有 `>= 0` 检查。在战斗中大量设置这些属性会有小幅性能开销。更重要的是，这些检查在某些边缘情况（如 `value` 由于浮点误差变为负数）可能导致异常。

### 2.2 大量枚举与扩展方法

`Entities/Cards/` 目录下有 36 个文件，大部分是枚举 + 扩展方法：

```
CardType.cs + CardTypeExtensions.cs
CardRarity.cs + CardRarityExtensions.cs
CardScope.cs (无 Extensions)
...
```

**问题**: 枚举和扩展方法分散在大量小文件中，可能降低导航效率。部分枚举的扩展方法较多，可以考虑集中管理。

### 2.3 `Lock` 对象用于同步

```csharp
private readonly Lock _playerReadyLock = new Lock();
```

**问题**: C# 的 `Lock` 类是 .NET 9 新增，在游戏主循环中使用锁可能引入性能问题。由于 Godot 是单线程主循环，理论上不应需要锁。这里的锁可能是为多人模式设计。

### 2.4 魔法数字

```csharp
public const int baseHandDrawCount = 5;
await Cmd.CustomScaledWait(0.5f, 0.8f);  // 战斗动画等待时间
await Cmd.CustomScaledWait(0.5f, 1f);    // 开场动画等待时间
```

**问题**: 部分数值（如等待时间）是硬编码的。对于 Mod 开发者来说，这些值应该可配置。

### 2.5 场景与 C# 的混合管理

- 907 个 `.tscn` 文件分散在 `scenes/` 中
- C# 节点脚本在 `src/Core/Nodes/` 中
- 部分逻辑在 GDScript 中 (48 个 `.gd` 文件)

**问题**: 混合语言 (C# + GDScript) 增加认知负担。GDScript 主要用于插件和测试工具，但仍需注意跨语言调用的边界。

### 2.6 网络序列化

```csharp
public ModelId CombatTargetId { get; set; }
```

**问题**: 多人模式使用 `ModelId` 进行网络同步，需要验证序列化/反序列化是否高效，以及 `ModelId` 在不同客户端是否一致（需要依赖 `ModelDb` 注册顺序一致）。

---

## 3. 可维护性问题

### 3.1 文件数量膨胀

| 类别 | 文件数 |
|------|--------|
| 能力模型 | 523 |
| 遗物模型 | 580 |
| VFX 节点 | ~290 |
| 生物视觉 | 127 |
| 屏幕场景 | 48+ |

**问题**: 每个能力/遗物一个文件。对于 500+ 能力、580+ 遗物，查找特定文件变困难。虽然没有单个文件过大问题，但导航成本高。

### 3.2 代码重复风险

每个 PowerModel 子类平均只有 10-30 行代码，但存在模式重复：

```csharp
public override decimal ModifyDamageAdditive(...) => base.Amount;  // 多处重复
public override Task OnTurnStart() { Flash(); ... }  // 多处重复
```

**改进建议**: 考虑使用配置驱动 (JSON/YAML) + 通用处理器的方式来减少重复子类。不过对于游戏内容的灵活性而言，当前方式也有其优势。

---

## 4. 性能建议

### 4.1 对象池覆盖

已有基本的对象池，但需要确认：
- 卡牌对象是否池化 (频繁创建/销毁)
- VFX 对象是否池化
- UI 元素是否复用

### 4.2 资源加载

`NAssetLoader` 后台加载资源。需要确认：
- 是否所有场景都预加载了关键资源
- 合理的内存预算管理

### 4.3 着色器复杂度

49 个着色器文件 + 大量 VFX 场景。移动端或低配设备可能需要简化。

---

## 5. 新学建议

### 5.1 从哪开始学

推荐按以下顺序阅读：

1. **入口**: `NGame.cs` → `GameStartup()` → 理解启动流程
2. **Run 流程**: `NRun.cs` → `RunManager.cs` → 理解游戏会话
3. **战斗核心**: `CombatManager.cs` → `CombatState.cs` → 战斗循环
4. **数据模型**: `AbstractModel.cs` → `CardModel.cs` → `PowerModel.cs` → `RelicModel.cs`
5. **Hook 系统**: `Hook.cs` → 游戏事件总线
6. **一个具体能力**: `StrengthPower.cs` → 理解如何 Hook 进游戏
7. **一个具体遗物**: `BurningBlood.cs` → 理解事件监听
8. **动作系统**: `GameAction.cs` → `ActionExecutor.cs` → `PlayCardAction.cs`

### 5.2 查询索引

| 关键词 | 搜索路径 |
|--------|----------|
| 战斗初始化 | `CombatManager.SetUpCombat` |
| 开始回合 | `CombatManager.StartTurn` |
| 打出一张牌 | `PlayCardAction.ExecuteAction` |
| 伤害计算 | `AttackCommand` → `PowerModel.ModifyDamage*` |
| 牌堆变化 | `CardModel.MoveToPile` → `Hook.AfterCardChangedPiles` |
| 能力添加 | `Creature.AddPower` |
| 场景切换 | `NSceneContainer.SetCurrentScene` |
| 存档保存 | `SaveManager` → `SerializableRun` |
| 多人动作 | `NetPlayCardAction` (继承 `PlayCardAction`) |
| 地图生成 | `StandardActMap` |
| 事件系统 | `EventModel` → `EventRoom` |
