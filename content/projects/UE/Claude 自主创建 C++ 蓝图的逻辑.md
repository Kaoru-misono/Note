---
title: "Claude 自主创建 C++ 蓝图的逻辑"
date: 2026-03-27
tags: [UE, C++, Blueprint, FastGeo, Streaming, Benchmark]
category: projects
description: "记录 Claude 从零为纯蓝图 UE 项目创建 C++ 模块并实现 Spline 自动移动 + Streaming 性能对比测试的完整决策过程"
status: done
---

# Claude 自主创建 C++ 蓝图的逻辑

## 1. 理解需求

原始需求：

> Content 目录下有一张 Newmap，里面放了一个很多树的场景，我现在希望能够测试 FastGeo 的 Streaming 消耗表现和没有 FastGeo 的对比，因此需要 PIE 启动并控制 player 移动触发 streaming，为了数据一致性，我需要 player 验证相同的路径进行移动

提取三个核心约束：

- **A/B 对比测试** — FastGeo on vs off，需要两组可比数据
- **路径一致性** — 两次测试 player 必须走完全相同的路径，否则数据不可比
- **PIE 环境** — 在编辑器内运行，不是 standalone

## 2. 探索项目现状

在写任何代码之前，先用 Explore agent 摸清了项目：

- **纯蓝图项目** — 没有 Source 目录，没有 C++ 代码
- **UE 5.7** — 引擎版本决定了 API 和模块结构
- **FirstPerson 模板** — 已有 `BP_FirstPersonCharacter`、`BP_FirstPersonGameMode`
- **FastGeoStreaming / PCGFastGeoInterop** — 作为引擎插件启用
- **NewMap.umap** — 目标测试地图
- **World Partition** — 这个后来成了一个坑

这一步的目的是避免盲目编码。比如如果项目已经有 C++ 代码，就不需要从零创建模块；如果不是 World Partition，后面也不会遇到 Actor 查找失败的问题。

## 3. 技术方案选择

"控制 player 沿相同路径移动"有几种实现方式：

| 方案 | 优点 | 缺点 |
|------|------|------|
| Sequencer 录制/回放 | UE 内置 | 需要手动录制，回放精度依赖序列化 |
| Demo Recording | 引擎原生 | 网络相关，PIE 下行为不一定一致 |
| 录制坐标+回放 | 灵活 | 需要先手动跑一次录制，代码量大 |
| **Spline 路径** | **可视化编辑、完全确定性、代码简单** | 需要手动摆控制点 |
| 蓝图实现 | 不需要 C++ | 帧级数据采集和 CSV 写入在蓝图里很繁琐 |

**决策：选 Spline + C++**

理由：
- Spline 在编辑器里可视化编辑，直观
- 每帧位置由 `距离 = 速度 × 时间累积` 计算，完全确定性，两次运行路径 100% 一致
- C++ 可以方便地做帧级数据采集、CSV 写入、控制台命令、CSV Profiler 集成
- 蓝图做这些会很笨重

## 4. 架构设计 — 三个类的拆分

核心问题是"谁负责什么"：

### ATestPathSpline — 纯数据，不包含逻辑

- 持有 `USplineComponent`（路径定义）
- 持有配置参数（`MoveSpeed`、`WarmupDelay`、`RunLabel`、`bAutoStart`）
- 为什么独立成 Actor？因为路径需要在编辑器里可视化编辑，必须是场景中的实体

### AAutoMovePlayerController — 所有运行时逻辑

- 为什么用 PlayerController 而不是 Component？因为需要 `SetControlRotation` 控制相机朝向，这是 PlayerController 的职责。如果用 Component 挂在 Pawn 上，还得拿到 Controller 的引用，多一层间接
- 为什么不直接修改 Character？因为要保持和现有 `BP_FirstPersonCharacter` 的兼容性，不侵入用户的蓝图资产

### AStreamingTestGameMode — 胶水层

- 唯一目的：把 `PlayerControllerClass` 设为 `AutoMovePlayerController`
- 为什么需要它？因为 UE 的 PlayerController 是由 GameMode 创建的，用户在 World Settings 里设置 GameMode Override 就能一键切换整个测试系统

## 5. 从零创建 C++ 模块

项目是纯蓝图的，需要创建完整的 C++ 模块结构：

```
Source/
  TestFastGeo.Target.cs          ← Game target
  TestFastGeoEditor.Target.cs    ← Editor target
  TestFastGeo/
    TestFastGeo.Build.cs          ← 模块依赖声明
    TestFastGeoModule.h/cpp       ← 模块入口
    Public/                       ← 头文件
    Private/                      ← 实现文件
```

同时修改 `.uproject` 添加 `Modules` 数组。这是从蓝图项目转混合项目的标准步骤。

依赖只声明了 `Core`, `CoreUObject`, `Engine`, `InputCore` — 最小依赖原则，只引入实际用到的模块。

## 6. 移动逻辑的关键决策

### 怎么移动 Pawn？

直接 `SetActorLocationAndRotation` + `ETeleportType::TeleportPhysics`，而不是用 `AddMovementInput`。

理由：
- `AddMovementInput` 通过 `CharacterMovementComponent` 走物理模拟，会受碰撞、重力影响，路径不可控
- 直接设位置是确定性的，每帧位置完全由 Spline 距离决定
- `TeleportPhysics` 告诉物理引擎这是传送，不要产生碰撞响应

### 为什么要 DisableMovement？

`CharacterMovementComponent` 每帧会根据速度/重力更新位置。如果不禁用，它会在 Teleport 之后又把 Pawn 拉回去或者施加重力，导致位置抖动。

### 碰撞处理的迭代

这里经历了三轮迭代，因为需求在对话中逐步明确：

1. 最初没处理碰撞 → 你提出树有碰撞会卡住
2. 加了 `SetActorEnableCollision(false)` → 你指出要保留树的物理碰撞体来测 streaming 开销
3. 撤回 → 你说 player 可以不接受碰撞
4. **最终方案**：`Capsule->SetCollisionResponseToAllChannels(ECR_Overlap)`

这个方案的精妙之处：树的碰撞体完整存在（streaming 物理开销如实反映），但 Player 的胶囊体对所有碰撞通道改为 Overlap — 穿过去但不被挡住。物理世界里碰撞检测仍然在发生（有开销），只是响应方式从 Block 变成了 Overlap。

## 7. World Partition 踩坑

第一次 PIE 测试时，日志显示：

```
[Benchmark] No TestPathSpline found in level.
```

GameMode 生效了，但 `UGameplayStatics::GetActorOfClass` 在 `BeginPlay` 里找不到 `TestPathSpline`。

**原因**：NewMap 使用 World Partition，Actor 是按 streaming cell 动态加载的。Controller 的 `BeginPlay` 执行时，`TestPathSpline` 所在的 cell 可能还没加载。

**解决方案**：把查找逻辑从 `BeginPlay` 移到定时器回调，延迟 2 秒后查找，找不到则每秒重试最多 5 次：

```cpp
void BeginPlay() {
    // 不在这里找了
    GetWorldTimerManager().SetTimer(..., &FindSplineAndAutoStart, 2.0f);
}

void FindSplineAndAutoStart() {
    PathSpline = GetActorOfClass(...);
    if (!PathSpline && SplineSearchRetries < 5) {
        // 重试
        SetTimer(..., 1.0f);
        return;
    }
    // 找到了，开始
}
```

## 8. DefaultPawnClass 不可编辑问题

在 World Settings 里发现 Default Pawn Class 是灰色不可编辑的。这是因为 C++ GameMode 的构造函数设了 `PlayerControllerClass` 但没设 `DefaultPawnClass`，UE 的 World Settings UI 在 GameMode 是 C++ 类时不允许单独覆盖 Pawn。

**解决**：在 `StreamingTestGameMode` 构造函数里用 `ConstructorHelpers::FClassFinder` 直接引用 `BP_FirstPersonCharacter`：

```cpp
static ConstructorHelpers::FClassFinder<APawn> PawnClassFinder(
    TEXT("/Game/FirstPerson/Blueprints/BP_FirstPersonCharacter"));
if (PawnClassFinder.Succeeded())
    DefaultPawnClass = PawnClassFinder.Class;
```

## 9. 数据采集 — 两层策略

### 自定义 CSV（简单、可控）

- 每帧记录 `DeltaTime`、位置、距离
- `FinishBenchmark` 时同步写入 `FFileHelper::SaveStringToFile`
- 计算统计摘要（Avg/P95/P99）

### UE CSV Profiler（全量引擎数据）

- 第一版用 `FCsvProfiler::Get()->BeginCapture/EndCapture` C++ API → 文件没写入磁盘
- **原因**：`EndCapture` 是异步的，PIE 结束时后台写入线程可能被打断
- 第二版改用控制台命令 `csvprofile start` / `csvprofile stop` → 成功

控制台命令方式更可靠，因为引擎内部处理了异步写入的生命周期。

## 10. 编译环境问题

机器上有 VS 18 和 VS 2022 两个版本。VS 18 的 v143 工具链缺少一个 props 文件导致 MSBuild 编译失败。

解决：没有去修 VS 安装文件（那是侵入性操作），而是直接用 UE 的 `Build.bat`，它通过 UBT 而不是 MSBuild 编译 C++，自动选择了 VS 2022 的工具链，绕过了问题。

## 11. 迭代总结

整个过程不是一次写完的，而是根据反馈迭代了多轮：

| 轮次 | 变更 | 触发原因 |
|------|------|----------|
| 1 | 创建完整 C++ 模块 + 三个类 | 初始需求 |
| 2 | 用 `Build.bat` 编译 | VS 18 编译失败 |
| 3 | `ConstructorHelpers` 引用 BP Pawn | `DefaultPawnClass` 不可编辑 |
| 4 | 碰撞改 Overlap | 树碰撞 + player 穿过的需求 |
| 5 | `FindSpline` 延迟重试 | World Partition Actor 加载延迟 |
| 6 | `FCsvProfiler` → `csvprofile` 命令 | CSV Profiler 文件未写入 |
| 7 | 开启 GC/LevelStreamingDetail category | 需要更多 streaming 指标 |

每轮迭代都是"发现问题 → 分析原因 → 最小改动修复"，没有大规模重写。
