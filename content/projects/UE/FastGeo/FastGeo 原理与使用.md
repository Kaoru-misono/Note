---
title: "FastGeo 原理与使用"
date: 2026-03-27
tags: [UE, FastGeo, Streaming, World-Partition, Performance]
category: projects
description: "FastGeo（Fast Geometry Streaming）的内部原理、配置方法、使用场景与限制，以及与 World Partition / Nanite / ISM / PCG 的关系"
status: done
source: "https://dev.epicgames.com/documentation/en-us/unreal-engine/API/PluginIndex/FastGeoStreaming"
aliases: [FastGeo, Fast Geometry Streaming]
---

# FastGeo 原理与使用

## 概述

FastGeo（Fast Geometry Streaming）是 UE 5.6 引入的**实验性**插件，由 Epic Games（Ari Arnbjornsson，前 Housemarque Returnal 主程）与 CD Projekt RED 合作开发，在 Unreal Fest Orlando 2025 的 Witcher 4 技术演示中首次展示（PS5 上 60FPS + 光追）。

**核心思路**：对于纯静态装饰几何体，完全绕过传统 Actor/Component 体系，直接以轻量级 scene primitive 的方式注册到渲染和物理场景，从而大幅降低 streaming 开销。

## 传统 AddToWorld 的开销

当一个 level 流入时，`AddToWorld()` 在 Game Thread 上依次执行：

1. `FLevelUtils::ApplyLevelTransform` — 应用 level transform
2. `ULevel::ApplyWorldOffset` — 调整 level 位置
3. **`ULevel::IncrementalUpdateComponents`** — 逐步注册所有 component（受 `s.GLevelStreamingComponentsRegistrationGranularity` 控制）
4. `ULevel::InitializeNetworkActors` — 网络初始化
5. **`ULevel::RouteActorInitialize()`** — 执行 BeginPlay 等初始化逻辑
6. `ULevel::SortActorList` — 排序 Actor 列表

其中**步骤 3-5 是最大开销**：每个 Actor 需要作为 UObject 生成，每个 Component 需要向引擎子系统（渲染、物理、导航等）注册，每个都需要运行初始化逻辑。

## FastGeo 的核心机制

### 绕过了什么

FastGeo 消除了以下步骤：

- **不生成 AActor UObject** — 没有 Actor 生命周期开销
- **不创建和注册 UActorComponent** — 不向引擎子系统注册 Component
- **不执行 BeginPlay / InitializeComponent** — 这些是纯静态几何体，无 gameplay 影响
- **不在传统 UObject 系统中分配 per-actor/per-component 内存**

取而代之，FastGeo 直接以**轻量级 scene primitive** 注册到渲染和物理场景，走一条快速路径。

### Runtime Cell Transformer 架构

FastGeo 通过 World Partition 的 **Runtime Cell Transformer (RCT)** 框架工作：

1. **RCT 不在运行时执行。** 虽然名字里有 Runtime，但实际只在 editor 的 streaming generation 阶段运行（PIE 或 Cook 时）
2. 主要 transformer 类是 `FastGeoWorldPartitionRuntimeCellTransformer`，通过 World Settings > World Partition Setup 添加
3. **两阶段验证**：
   - **Actor 级验证** (`CanTransformActor`)：检查 `AllowedActorClasses` / `DisallowedActorClasses`（支持精确匹配和继承），检查 Actor 标签（`NoFastGeo` 或 `CellTransformer_IgnoreActor` 会被排除）
   - **Component 级验证** (`CanTransformComponent`)：只处理 `UPrimitiveComponent` 的子类，通过 `FastGeo::GetFastGeoComponentType` 映射支持的类型

### 支持的 Component 类型

- `StaticMeshComponent`
- `InstancedStaticMeshComponent`
- `SkinnedMeshComponent`
- `InstancedSkinnedMeshComponent`

### 不支持的 Component 类型（内置禁止）

- `InteractiveFoliageComponent`
- `LandscapeNaniteComponent`
- `SplineMeshComponent`
- Water 相关 mesh component

### 转换结果

转换后的 Actor 在 Outliner 中变为 `[Unnamed]`——不再是传统 UObject Actor。几何数据存储在 `FastGeoContainer` 中。**转换是非破坏性的**——只在 PIE/Cook 时生效，不修改保存的 level 数据。

## 配置方法

### 1. 启用插件

Edit > Plugins，搜索 "FastGeo" 或 "Fast Geo"，启用 **Fast Geo Streaming** 插件。

### 2. 必需的 INI 配置

在 `DefaultEngine.ini` 中（只读 CVar，不可通过控制台设置）：

```ini
[SystemSettings]
p.Chaos.EnableAsyncInitBody=true
LevelStreaming.AllowIncrementalPreRegisterComponents=true
```

不设置 `p.Chaos.EnableAsyncInitBody` 会报错：*"FastGeoStreaming Cell Transformer requires 'p.Chaos.EnableAsyncInitBody' to be enabled."*

### 3. 添加 Transformer

World Settings > World Partition Setup > Runtime Cell Transformers，添加 `FastGeoWorldPartitionRuntimeCellTransformer`。

可选叠加 `WorldPartitionRuntimeCellTransformerISM`（放在 FastGeo 之前），先将相同 mesh 的 StaticMeshActor 合并为 ISM，再由 FastGeo 处理。

### 4. 配置允许/禁止的 Actor

在 Transformer 属性上配置：
- **AllowedActorClasses** — 白名单（自定义蓝图 Actor 需要显式添加）
- **DisallowedActorClasses** — 黑名单
- **IgnoredRemainingComponentClasses** — 允许部分组件无法转换时仍进行转换

**单个 Actor 排除**：给 Actor 添加标签 `NoFastGeo` 或 `CellTransformer_IgnoreActor`。

### 5. Streaming 预算相关 CVar

```
s.LevelStreamingActorsUpdateTimeLimit        -- AddToWorld 基础时间预算（秒）
s.PriorityLevelStreamingActorsUpdateExtraTime -- 优先级加载的额外预算
s.GLevelStreamingComponentsRegistrationGranularity -- 每 tick 注册的 Component 数量
s.UseUnifiedTimeBudgetForStreaming 1          -- 统一 async loading + level streaming 预算
```

### 6. PCG FastGeo Interop (UE 5.7)

对于 PCG GPU 集成：
1. 启用 **PCG FastGeo Interop** 插件
2. 设置 CVar：`pcg.RuntimeGeneration.ISM.ComponentlessPrimitives = 1`

### 7. 调试命令

| 命令 | 用途 |
|------|------|
| `show ActorColoration FastGeo` | 蓝色 = FastGeo 内容，红色 = 非 FastGeo |
| `FastGeo.Show 0/1` | 隐藏/显示 FastGeo 图元 |
| `FastGeo.EnableTransformerDebugMode` | 选中 Actor 时输出转换详情日志 |

> 已知 Bug（UE-356401）：开启 PostProcess 时 `show ActorColoration FastGeo` 颜色显示为黑色。

## 使用场景

### FastGeo 收益显著的场景

| 场景 | 原因 |
|------|------|
| **大量独立 StaticMeshActor** | 每个 Actor 都需要走注册流程，FastGeo 直接绕过 |
| **密集城市环境** | Witcher 4 Kovir 城市演示是典型案例 |
| **岩石、道具密集场景** | 大量静态装饰物 |
| **不可交互的纯视觉内容** | 无 gameplay 影响的装饰几何体 |

### FastGeo 收益不明显的场景

| 场景 | 原因 |
|------|------|
| **PCG InstancedStaticMeshActor** | ISM 已合批减少 Actor 数量，注册开销本就不大 |
| **需要 gameplay 交互的 Actor** | 可交互物品、载具、NPC、触发器等 |
| **Landscape** | `LandscapeStreamingProxy` 明确不可转换 |
| **动态/可移动 Actor** | FastGeo 面向不可变静态几何体 |
| **SplineMesh、InteractiveFoliage、Water** | 明确禁止 |
| **有 Mesh Paint 数据的 Actor** | 纹理绘制数据在 ISM/FastGeo 转换中丢失 |

### 性能测试数据（本项目）

详见 [[FastGeo 性能测试]]。

| 场景类型 | FastGeo 收益 |
|---------|-------------|
| 独立 StaticMeshActor | AddToWorld -73%，总 Streaming -50%，卡顿帧 -80% |
| PCG InstancedStaticMeshActor | 仅 AddToWorld 峰值 -21%，其余持平 |

**结论：FastGeo 的价值与场景中独立 Actor 的数量正相关。**

## 与其他系统的关系

### World Partition

FastGeo **不是** World Partition 的替代品。它工作在 World Partition 的 streaming generation 管线内部，作为 Runtime Cell Transformer。World Partition 仍然管理 grid cell、streaming source、loading range、Data Layer 和整个 streaming 生命周期。FastGeo 优化的是 cell 内几何体在运行时的**表示方式**。

### Nanite

互补关系。Nanite 处理 LOD 和虚拟化几何体渲染，FastGeo 处理 streaming 注册效率。FastGeo 可以转换启用了 Nanite 的 StaticMeshComponent。但 `LandscapeNaniteComponent` 明确禁止。

### ISM/HISM

`WorldPartitionRuntimeCellTransformerISM` 可以与 FastGeo 叠加。ISM Transformer 先将同一 cell 内共享相同 mesh 的多个 StaticMeshActor 合并为 `WorldPartitionAutoInstancedActor`，然后 FastGeo 再处理这些 ISM Actor。如果 ISM 已经显著减少了 Actor 数量（如 PCG 森林），FastGeo 的边际收益就很小。

### PCG

- UE 5.7 新增 **PCG FastGeo Interop** 插件，让 PCG GPU 利用 FastGeo 组件
- 移除了对 partition actor 的依赖，直接创建本地 PCG 组件
- PCG GPU + FastGeo 实现 "componentless primitives"
- Epic 报告 5.7 vs 5.5 PCG 性能提升约 2 倍

## UE 5.7 相对 5.6 的改进

- **FastGeoContainer PSO 预缓存** 从 `PostLoad`（Game Thread）移至**异步执行**，消除资产加载时的主线程阻塞
- **GPU Fast Geo Interop** 改进了多平台 GPU 计算路径
- Mutators 支持更多 actor description 属性修改（如参与 HLOD 分配）

## 关键 API 汇总

| 名称 | 类型 | 说明 |
|------|------|------|
| `FastGeoWorldPartitionRuntimeCellTransformer` | Transformer 类 | 主要 FastGeo 转换器 |
| `WorldPartitionRuntimeCellTransformerISM` | Transformer 类 | ISM 合并转换器（可与 FastGeo 叠加） |
| `FastGeoWorldSubsystem` | World Subsystem | 运行时 FastGeo 管理（蓝图可通过 `GetFastGeoWorldSubsystem` 访问） |
| `FastGeoContainer` | 数据容器 | 存储转换后的几何数据 |
| `FastGeo::GetFastGeoComponentType` | 函数 | Component 类型到 FastGeo 类型的映射 |
| `CanTransformActor` / `CanTransformComponent` | 验证函数 | Actor/Component 级别的资格检查 |
| `WorldPartitionAutoInstancedActor` | 生成 Actor | ISM Transformer 创建的合并 Actor |

## 参考资料

- [FastGeo Streaming API (UE 5.7)](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/PluginIndex/FastGeoStreaming)
- [PCG FastGeo Interop API (UE 5.7)](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/PluginIndex/PCGFastGeoInterop)
- [[Streaming Improvements for Dense Worlds]] — Unreal Fest Orlando 2025 演讲笔记
- [UE 5.6 Performance Highlights — Tom Looman](https://tomlooman.com/unreal-engine-5-6-performance-highlights/)
- [UE 5.7 Performance Highlights — Tom Looman](https://tomlooman.com/unreal-engine-5-7-performance-highlights/)
- [Fast Geo Notes — hzFishy](https://notes.hzfishy.fr/Unreal-Engine/Engine--and--Editor/Content-Streaming/Fast-Geo)
- [Level Streaming Optimization — Petr Leontev](https://peterleontev.com/blog/level_streaming_optimization/)
- [Unreal World Partition Internals — xbloom.io](https://xbloom.io/2025/10/24/unreals-world-partition-internals/)
