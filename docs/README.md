# Slay the Spire 2 - 代码分析文档

> 本文档基于 Slay the Spire 2 (STS2) 的 Godot 4.5 + C# 源码，分析其架构设计、核心逻辑、设计模式与潜在问题。
> 目的：供人类开发者学习，同时为后续 AI 代码索引提供结构化参考。

## 文档索引

| 文件 | 内容 |
|------|------|
| [01-project-overview.md](./01-project-overview.md) | 项目概览、技术栈、目录结构逐层说明 |
| [02-architecture.md](./02-architecture.md) | 分层架构、核心类关系、控制流 |
| [03-core-gameplay.md](./03-core-gameplay.md) | 战斗系统、卡牌系统、遗物/药水/能力系统、地图与 Run 流程 |
| [04-design-patterns.md](./04-design-patterns.md) | 使用的设计模式详解（命令模式、Hook 系统、Model 体系等） |
| [05-potential-issues.md](./05-potential-issues.md) | 潜在问题与改进建议 |

## 快速导航

- **入口**: `scenes/game.tscn` -> `src/Core/Nodes/NGame.cs`
- **战斗核心**: `src/Core/Combat/CombatManager.cs`
- **Run 管理**: `src/Core/Runs/RunManager.cs`
- **数据模型基类**: `src/Core/Models/AbstractModel.cs`
- **Room 基类**: `src/Core/Rooms/AbstractRoom.cs`
- **事件系统**: `src/Core/Hooks/Hook.cs`
- **命令系统**: `src/Core/Commands/Cmd.cs`
- **动作系统**: `src/Core/GameActions/GameAction.cs`

## 核心统计数据

| 指标 | 数值 |
|------|------|
| C# 源文件 | 约 3,279 个 |
| GDScript 文件 | 48 个 |
| Godot 场景 (.tscn) | 907 个 |
| VFX 场景 | 152 个 |
| 生物视觉场景 | 127 个 |
| 能力 (Power) 模型 | 约 260+ 种 |
| 遗物 (Relic) 模型 | 约 290+ 种 |
| 多人游戏服务模式 | 4 种 (单机/主机/客户端/回放) |
