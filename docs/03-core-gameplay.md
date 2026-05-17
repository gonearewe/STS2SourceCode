# 核心功能逻辑

## 1. 战斗系统 (Combat)

### 1.1 CombatState

`CombatState` 是战斗的核心数据对象，负责管理：

- **阵营**: `CombatSide.Player` / `CombatSide.Enemy`
- **生物列表**: `Allies`, `Enemies`, `Creatures`
- **回合计数**: `RoundNumber`
- **所有卡牌**: 战斗中存在的所有卡牌引用

### 1.2 回合流程

```
CombatManager.StartCombatInternal()
  │
  └─ StartTurn() [循环]
       │
       ├─ Player 回合开始:
       │   ├─ Creature.BeforeTurnStart()
       │   ├─ Hook.BeforeSideTurnStart()
       │   ├─ PlayerActionsDisabled = false
       │   ├─ 敌人准备下一回合 (PrepareForNextTurn)
       │   ├─ Creature.AfterTurnStart()
       │   ├─ Hook.AfterBlockCleared()
       │   ├─ SetupPlayerTurn() → 抽牌 → 获得能量
       │   ├─ Hook.AfterSideTurnStart()
       │   ├─ OrbQueue.AfterTurnStart() (Defect 职业)
       │   └─ 等待所有玩家就绪 (SetReadyToEndTurn)
       │
       ├─ Enemy 回合开始:
       │   ├─ 敌人执行 Intent
       │   └─ 攻击/施加状态
       │
       └─ 切换到对方回合
            ├─ CombatState.CurrentSide 切换
            ├─ CheckWinCondition()
            └─ StartTurn() [下一回合]
```

### 1.3 卡牌打出流程

```
PlayCardAction.ExecuteAction()
  ├─ 验证: CanPlay() + IsValidTarget()
  ├─ SpendResources() → 消耗能量/星能
  ├─ CardModel.OnPlayWrapper()
  │    ├─ Hook.BeforeCardPlayed()
  │    ├─ CardModel.OnPlay() → 实际效果
  │    ├─ 目标选择 → 创建 AttackCommand
  │    │    ├─ Hook.BeforeAttack()
  │    │    ├─ 伤害计算 (StrengthPower, WeakPower 等)
  │    │    │    ├─ PowerModel.ModifyDamageAdditive()
  │    │    │    └─ PowerModel.ModifyDamageMultiplicative()
  │    │    ├─ 实际扣血/格挡
  │    │    └─ Hook.AfterAttack()
  │    └─ Hook.AfterCardPlayed()
  └─ 卡牌进入弃牌堆/消耗
```

### 1.4 伤害计算管道

```
AttackCommand 构建
  ├─ 基础伤害 (卡牌定义)
  ├─ 加成修正 (StrengthPower, DexterityPower...)
  │    └─ PowerModel.ModifyDamageAdditive()
  ├─ 乘数修正 (VulnerablePower, WeakPower...)
  │    └─ PowerModel.ModifyDamageMultiplicative()
  ├─ 格挡抵消
  └─ 最终 HP 扣除
```

### 1.5 战斗结束

```
Enemy 全灭 → CombatManager.IsEnding = true
  ├─ Hook.AfterCombatVictory() → 遗物触发 (BurningBlood 等)
  ├─ 奖励生成 (Rewards)
  └─ CombatManager.CombatWon 事件

玩家全灭 → PendingLossState
  └─ CombatManager.CombatEnded 事件 → GameOver
```

---

## 2. 卡牌系统

### 2.1 卡牌数据模型

```
AbstractModel
  └─ CardModel (抽象基类)
       ├─ Title / Description (LocString 本地化)
       ├─ CardType (Attack / Skill / Power / Status / Curse)
       ├─ TargetType (None / AnyEnemy / AnyAlly / AllEnemies / AllAllies)
       ├─ CardRarity (Basic / Common / Uncommon / Rare / Special)
       ├─ EnergyCost / StarCost
       ├─ Keywords / Tags
       ├─ Pool (所属卡牌池)
       ├─ Owner (Player)
       ├─ Pile (当前所在牌堆: Draw / Hand / Discard / Exhaust / etc.)
       └─ UpgradeLevel (升级)
            ├─ OnPlay() → 打出效果 (虚方法)
            ├─ OnUpgrade() → 升级效果
            └─ CanPlay() → 是否可打出
```

### 2.2 卡牌定义

所有具体卡牌定义在 `src/Core/Models/Cards/` 目录下。示例：

```csharp
// Ironclad 的燃烧之血遗物
public sealed class BurningBlood : RelicModel
{
    public override RelicRarity Rarity => RelicRarity.Starter;

    public override async Task AfterCombatVictory(CombatRoom _)
    {
        if (!base.Owner.Creature.IsDead)
        {
            Flash();
            await CreatureCmd.Heal(base.Owner.Creature, base.DynamicVars.Heal.BaseValue);
        }
    }
}
```

### 2.3 卡牌牌堆管理

每个玩家有以下牌堆，由 `CardPile` 管理：

- **抽牌堆** (Draw Pile)
- **手牌** (Hand)
- **弃牌堆** (Discard Pile)
- **消耗堆** (Exhaust Pile)
- **创建中** (Being Created)

牌堆操作通过 `CardPileCmd` 实现，并触发 Hook：

```
CardModel.MoveToPile(newPile)
  └─ Hook.AfterCardChangedPiles(runState, combatState, card, oldPile, source)
```

---

## 3. 能力系统 (Powers)

### 3.1 能力模型

```csharp
AbstractModel
  └─ PowerModel
       ├─ Type (Buff / Debuff / Neutral)
       ├─ StackType (Counter / Intensity / Duration / Minion)
       ├─ Amount (堆叠数)
       ├─ Owner (所属 Creature)
       └─ Hook 方法 (可重写)
            ├─ ModifyDamageAdditive()    # 伤害加成 (如力量)
            ├─ ModifyDamageMultiplicative() # 伤害乘算 (如易伤)
            ├─ ModifyBlockGained()       # 格挡加成
            ├─ OnTurnStart() / OnTurnEnd()
            ├─ OnCardPlayed() / OnCardDrawn()
            └─ ...
```

### 3.2 能力示例

```csharp
// 力量: 攻击伤害 + 力量值
public sealed class StrengthPower : PowerModel
{
    public override decimal ModifyDamageAdditive(Creature target, decimal amount, ValueProp props, Creature dealer, CardModel cardSource)
    {
        if (base.Owner != dealer) return 0;
        if (!props.IsPoweredAttack()) return 0;
        return base.Amount;  // 每层力量 +1 伤害
    }
}

// 易伤: 伤害 x 1.5
// Fragile: 格挡 x 0.75
```

### 3.3 能力触发机制

能力通过 `Hook` 系统触发。`CombatState.IterateHookListeners()` 收集所有注册了 Hook 的模型：

```csharp
// 所有 AbstractModel 均可接收 Hook
public abstract class AbstractModel {
    public abstract bool ShouldReceiveCombatHooks { get; }
    // ... 虚方法: BeforeAttack, AfterAttack, OnTurnStart, etc.
}
```

---

## 4. 遗物系统 (Relics)

### 4.1 设计模式

遗物同样继承 `AbstractModel` → `RelicModel`，通过覆写 Hook 方法实现效果：

```csharp
public abstract class RelicModel : AbstractModel
{
    public abstract RelicRarity Rarity { get; }
    public virtual RelicStatus Status { get; }  // Enabled / Disabled / Locked

    // 可覆写的 Hook 方法与 Powers 类似
    public virtual Task AfterCombatVictory(CombatRoom room) => Task.CompletedTask;
    public virtual Task OnTurnStart() => Task.CompletedTask;
    // ...
}
```

### 4.2 激活与闪亮

```csharp
// 遗物闪亮效果
public void Flash() { ... }

// 遗物可通过 Hook 条件触发
public override async Task AfterCombatVictory(CombatRoom _) {
    if (!base.Owner.Creature.IsDead) {
        Flash();
        await CreatureCmd.Heal(base.Owner.Creature, ...);
    }
}
```

---

## 5. 药水系统 (Potions)

设计模式与卡牌/遗物类似，继承链为：

```
AbstractModel → PotionModel
```

药水效果在 `OnUse()` 方法中定义，使用方式和卡牌类似。

---

## 6. 地图系统 (Map)

### 6.1 地图生成

- `ActMap` / `StandardActMap` - 地图生成算法
- `MapPoint` / `MapCoord` - 地图节点与坐标
- `MapDrawing` - 地图绘制

### 6.2 房间类型

| 房间类型 | 类 | 场景 |
|---------|-----|------|
| 战斗 | `CombatRoom` | `combat_room.tscn` |
| 事件 | `EventRoom` | `event_room.tscn` |
| 商人 | `MerchantRoom` | `merchant_room.tscn` |
| 休息 | `RestSiteRoom` | `rest_site_room.tscn` |
| 宝箱 | `TreasureRoom` | `treasure_room.tscn` |
| 地图 | `MapRoom` | `map_room.tscn` |

---

## 7. Run 流程

```
MainMenu → CharacterSelectScreen
  └─ RunManager.StartRun()
       ├─ 创建 RunState (包含 RNG、角色、难度)
       ├─ NRun.Create(state)
       ├─ SetCurrentRoom(MapRoom) → 显示地图
       ├─ 玩家选择节点 → 进入对应房间
       ├─ 房间循环:
       │    ├─ CombatRoom → 战斗 → 奖励
       │    ├─ EventRoom → 选择 → 结果
       │    ├─ MerchantRoom → 买卖
       │    ├─ RestSiteRoom → 休息/锻造
       │    └─ TreasureRoom → 拾取
       ├─ Act 结束 → Boss 战斗 → 下一幕
       └─ 胜利/死亡 → GameOverScreen
```

### 存档点

Run 状态通过 `SaveManager` 持久化，包含：
- `SerializableRun` - 可序列化的 Run 数据
- `RoomSet` - 房间集合
- 支持云存档同步
