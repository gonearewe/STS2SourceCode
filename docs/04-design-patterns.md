# 优秀设计分析

## 1. AbstractModel + Hook 事件系统 ★★★★★

这是整个游戏最具扩展性的设计。

### 原理

```csharp
public abstract class AbstractModel : IComparable<AbstractModel> {
    public ModelId Id { get; }
    public abstract bool ShouldReceiveCombatHooks { get; }

    // 虚方法 — 子类选择性覆写
    public virtual Task BeforeAttack(AttackCommand cmd) => Task.CompletedTask;
    public virtual Task AfterCardPlayed(PlayerChoiceContext ctx, CardPlay play) => Task.CompletedTask;
    public virtual decimal ModifyDamageAdditive(...) => 0;
    // ... 约 100+ 个可覆写方法
}

public static class Hook {
    public static async Task BeforeAttack(CombatState state, AttackCommand cmd) {
        foreach (var model in state.IterateHookListeners())
            await model.BeforeAttack(cmd);
    }
}
```

### 优点

- **所有游戏内容 (卡牌、遗物、能力、药水) 统一继承 AbstractModel**
- **任何模型均可监听任意游戏事件** — 只需覆写对应方法
- **新增遗物/能力只需写一个类，零配置** — ModelDb 自动注册
- **天然支持组合** — 多个 PowerModel 叠加修正伤害，无需额外代码

### 示例

```csharp
// 力量: 修改伤害
class StrengthPower : PowerModel {
    public override decimal ModifyDamageAdditive(...) => base.Amount;
}

// 残暴: 受伤时抽牌
class BrutalityPower : PowerModel {
    public override async Task AfterAttack(...) {
        if (被攻击者是自己的主人) await 抽牌();
    }
}

// 燃烧之血: 战斗结束回血
class BurningBlood : RelicModel {
    public override async Task AfterCombatVictory(CombatRoom _) {
        await CreatureCmd.Heal(owner.Creature, 6);
    }
}
```

---

## 2. 命令模式 (Command Pattern) ★★★★

### 结构

```
Cmd (静态入口) → DamageCmd.Attack(damage) → AttackCommand
               → CreatureCmd.Heal(creature, amount)
               → CardCmd.MoveToPile(card, pile)
               → VfxCmd.Play(vfxScene, position)
               → SfxCmd.Play(soundId)

AttackCommand → Hook.BeforeAttack()
             → 计算伤害 (累加 Power 加成)
             → 应用伤害
             → Hook.AfterAttack()
```

### 优点

- **分离意图与执行** — 调用者只需描述"做一个攻击"，复杂逻辑封装在 Command 内部
- **统一的 Hook 触发点** — 所有伤害必经 `BeforeAttack/AfterAttack`
- **可测试性** — Command 可单独测试
- **动画友好** — Command 天然支持等待/链式调用

---

## 3. 异步驱动的游戏循环 ★★★★

### 特点

- 整场战斗是 `async/await` 驱动的协程
- `Cmd.Wait()` / `Cmd.CustomScaledWait()` 封装 Godot Timer
- **FastMode** 支持: 正常/快速/瞬间三种速度模式
- 动作队列 (`ActionQueueSet` + `ActionExecutor`) 保证动作有序执行

```csharp
// 战斗回合流程 — 就是一段异步代码
private async Task StartTurn() {
    await Hook.BeforeSideTurnStart(state, side);
    await Cmd.CustomScaledWait(0.5f, 0.8f);  // 开场动画
    foreach (var c in creatures) await c.AfterTurnStart(...);
    await Hook.AfterSideTurnStart(state, side);
    // ...
}
```

### 对比传统状态机

传统做法: 有限状态机 (Idle → Playing → EnemyTurn → ...)  
本项目做法: 协程式异步流，**更直观、易维护**

---

## 4. ModelDb 自动注册系统 ★★★★

### 原理

```csharp
[GenerateSubtypes]  // Source Generator 自动生成子类型注册
public abstract class AbstractModel {
    protected AbstractModel() {
        if (ModelDb.Contains(GetType()))
            throw new DuplicateModelException(type);
        Id = ModelDb.GetId(type);  // 自动分配唯一 ID
    }
}
```

### 优点

- **零配置内容注册** — 每个 CardModel/RelicModel/PowerModel 子类自动注册
- **ID 系统** — `ModelId` 包含 Category + Entry，支持网络序列化
- **不可变/可变分离** — `IsCanonical` vs `IsMutable`，防止意外修改数据模板

---

## 5. 场景容器 (Scene Container) 模式 ★★★★

### 实现

`NSceneContainer` 是一个轻量场景栈：

```csharp
public void SetCurrentScene(Control node) {
    // 清除旧场景
    foreach (var child in GetChildren()) {
        RemoveChildSafely(child);
        child.QueueFreeSafely();
    }
    // 设置新场景
    CurrentScene = node;
    AddChildSafely(node);
}
```

### 用途

- **房间切换**: NRun 的 `_roomContainer` (Combat ↔ Event ↔ Merchant ...)
- **主菜单/Logo**: NGame 的 `RootSceneContainer`
- **覆盖层**: `NOverlayStack` 用于 GameOver、检查界面

### 优点

- 隔离场景生命周期，避免节点泄漏
- 安全的引用检查 (`GodotObject.IsInstanceValid`)
- 单一职责 — 每个容器只管理一个当前场景

---

## 6. 多人同步架构 ★★★

### 设计

- **本地动作 + Net 包装器** — 每个 `GameAction` 有对应的 `Net*` 版本
- **4 种服务模式**: 单机 / 主机 / 客户端 / 回放
- **同步器体系**: `CombatStateSynchronizer`, `MapSelectionSynchronizer` 等
- **Checksum 校验**: `ChecksumTracker` 保证状态一致

### 优点

- 单机和多人共享同一套战斗逻辑
- Net 包装器只处理同步，不重复业务逻辑

---

## 7. 本地化系统 ★★★

- `LocString` 封装本地化键值对
- 支持 BBCode 标签
- `DynamicVar` 支持运行时替换变量 (伤害值、卡牌名等)
- `LocManager` 单例管理所有翻译资源
- 11+ 语言支持

---

## 8. 性能与调试工具 ★★★★

### 内置调试功能

- **FastMode**: Normal / Fast / Instant — 一键加速游戏
- **DebugHotkey**: 隐藏 UI、加速/减速、跳过战斗
- **DevConsole**: 命令行调试
- **AutoSlay**: 自动战斗模式 (测试用)
- **VFX Playground**: 特效测试沙盒
- **HitStop / ScreenShake**: 打击感系统

### 对象池

- `src/Core/Nodes/Pooling/` — 复用频繁创建销毁的对象 (特效等)

---

## 9. 其他优秀实践

| 实践 | 说明 |
|------|------|
| **ValueProps** | 响应式属性系统，属性变化自动通知 UI |
| **HoverTips** | 统一的悬浮提示框架 |
| **VFX 模板化** | 152 个 VFX 场景 + ~290 个 NVfx 控制器 |
| **存档 Schema 迁移** | 内置迁移系统，支持版本升级 |
| **Source Generator** | 使用 C# Source Generator 自动生成重复代码 |
| **RNG 系统** | 可复现的随机数 (种子支持) |
| **Mod 支持** | 内置 ModManager |
