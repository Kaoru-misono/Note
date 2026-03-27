---
title: "Streaming Improvements for Dense Worlds in The Witcher 4 UE5 Tech Demo"
date: 2026-03-23
tags: [UE5, streaming, world-partition, fast-geo, optimization, witcher4, unreal-fest]
category: projects
description: "Unreal Fest Orlando 2025 演讲笔记：Epic Games 与 CD Projekt Red 合作改进 UE5 World Partition 流送系统，实现次世代主机 60fps 开放世界"
status: in-progress
source: "https://www.youtube.com/watch?v=BdopUm1_1_E"
---

# Streaming Improvements for Dense Worlds in The Witcher 4 UE5 Tech Demo

> **演讲者：** Richard Malo (Principal Engine Programmer, Epic Games) & Jarosław Fruczki (Core Tech Engineer, CD Projekt Red)
> **活动：** Unreal Fest Orlando 2025
> **特别感谢：** Sebastian Lucier, Ben Ziggler (Epic Games)

## 背景与目标

Epic Games 与 CDPR 合作，目标是构建一个技术 Demo，展示 Unreal Engine 在**次世代主机上以 60fps 运行开放世界体验**的能力。

![Open World 60fps Goal](assets/streaming-improvements/02-open-world-60fps.jpg)

![Agenda](assets/streaming-improvements/01-agenda.jpg)

---

## 1. Streaming Generation 概览

![Streaming Generation Overview](assets/streaming-improvements/03-streaming-generation.jpg)

World Partition 的流送生成过程（Streaming Generation Process）将大型世界拆分为可高效流送的 cell。该过程在 **map cooking** 和 **PIE 启动**时运行，因此必须高效。

### 流程

1. **构建 Actor Descriptor 列表** — 收集所有 World Partition Actor 的流送相关信息（streaming bounds、runtime grid、spatially loaded flag、runtime data layers）
2. **验证 Pass** — 检测、报告和修正错误
3. **分区（Partitioning）** — Runtime Hash 使用 WP 设置和 Actor Descriptors 创建 cell 并分配 Actor
4. **生成 Level** — 为每个 cell 创建 Level，Actor 被加载并移入其中

### 对 Streaming 的期望

![Streaming Goals 1](assets/streaming-improvements/04-streaming-goals-1.jpg)

![Streaming Goals 2](assets/streaming-improvements/05-streaming-goals-2.jpg)

| 需求 | 说明 |
|------|------|
| **灵活可配** | 不同用例需要不同策略 |
| **高性能** | 玩家自由移动无卡顿 |
| **可配置预算** | 预算必须被严格遵守 |
| **高粒度** | 系统不应被大量对象压垮 |
| **易迭代** | Grid 分配、HLOD 设置需可在后期调整 |

---

## 2. Actor Descriptor Mutators（UE 5.4+）

Mutators 在 **pre-partitioning 阶段**引入，允许以代码驱动方式**非破坏性地修改 Actor Descriptors**，无需 checkout、修改或提交内容。

![Mutators Intro](assets/streaming-improvements/06-mutators-intro.jpg)

### 核心特性

- 可修改的属性：`Spatially Loaded Flag`、`Runtime Grid`（未来将支持更多）
- 修改后会运行**第二次验证 pass**
- 支持基于 actor 位置、bounds、name、tags 等任意属性组合进行自动分配

![Mutators Properties](assets/streaming-improvements/07-mutators-properties.jpg)

### CDPR 的使用方式

CDPR 利用文件夹结构分离 architecture、decoration、vegetation 等内容。通过 Mutators **自动化 grid 分配**：

![Mutators Code Example](assets/streaming-improvements/08-mutators-code.jpg)

- 可轻松拆分已有结构，将 decoration 按 size、draw distance 等细分到更精细的 grid
- 快速迭代比较结果

### 性能影响

- **更少的 primitives 同时加载** → 显著降低内存消耗和性能开销
- 大 level chunk 被拆分为**更小的 section**，分散到不同帧和不同物理空间加载

![Mutators Performance](assets/streaming-improvements/09-mutators-performance.jpg)

---

## 3. 新 Runtime Hash Set（UE 5.4+）

新的 Runtime Hash Set 替代了 UE 5.0 的原始 Runtime Spatial Hash。

![Runtime Hash Set](assets/streaming-improvements/10-runtime-hash-set.jpg)

### 特性

- 支持 **2D 和 3D 流送**
- 包含 Partition Object 列表，Actor 根据 `Runtime Grid` 属性映射到对应的 Partition Object
- 提供两种 Partition Object：
  - **Loose Hierarchical Grid**（默认，新建地图时使用）
  - **Level Streaming Partition**（模拟标准 Level Streaming / World Composition）
- 将分区逻辑与通用逻辑（如 Runtime Data Layers 处理）**解耦**，易于编写自定义 Partition Object

---

## 4. Runtime Cell Transformers（UE 5.5+）

在 **post-partitioning 阶段**引入，允许**变换已生成的 World Partition Cells**。

![Cell Transformers](assets/streaming-improvements/11-cell-transformers.jpg)

### 核心特性

- 非破坏性、数据驱动
- 通过 World Partition Setup 中的 **Runtime Cell Transformer Stack** 配置
- Cell 依次通过 stack 中的每个 transformer

### 可以做 vs 不可以做

![Cell Transformers Limitations](assets/streaming-improvements/12-cell-transformers-limits.jpg)

| 可以做 | 不可以做 |
|--------|----------|
| 修改 cell level 内容（增删改 Actor/Component） | 修改 Persistent Level 内容 |
| 在 level 内创建 embedded asset | 生成新的 World Partition Cell |
| | 创建 public asset |

### 内置 ISM Transformer

自动将 **Static Mesh Actors 转换为 Instanced Static Mesh Components**：

- 可指定最小实例数阈值
- 可定义允许/禁止的 Actor 类
- 限制：不支持 replicated actors、root component 必须 static、不支持 child actor component

![Implement your own Transformers](assets/streaming-improvements/13-implement-transformers.jpg)

### ISM 转换效果

![ISM Transformer Results](assets/streaming-improvements/40-ism-transformer-results.jpg)

- **更少的 primitives**（合并 components）
- **减少内存使用**
- 真正的性能收益：更少的 load、postload、spawn、destroy

---

## 5. Async Physics State Creation（UE 5.6+，实验性）

经过 ISM 转换减少 component 数量后，发现**大部分 streaming 开销来自物理状态的创建和销毁**。

![Physics State Problem](assets/streaming-improvements/physics-state-problem.jpg)

### 实现

![Async Physics](assets/streaming-improvements/15-async-physics.jpg)

- Level Streaming 的 AddToWorld / RemoveFromWorld 支持异步任务管理
- 新增 **incremental pre-register / pre-unregister component 阶段**处理物理状态的异步创建/销毁
- 支持的组件：`StaticMeshComponent`、`ISMComponent`、`LandscapeHeightFieldCollisionComponent`

### 并发 Add/Remove

![Concurrent Levels](assets/streaming-improvements/16-concurrent-levels.jpg)

为最大化时间预算利用率，支持**同时 add 和 remove 多个 level**：
- 在同一游戏线程时间预算内启动多个 AddToWorld / RemoveFromWorld
- 异步任务在下一帧完成

### 性能效果

![Combined Performance Graph](assets/streaming-improvements/17-combined-perf-graph.jpg)

- 主要收益：**重物理创建/销毁从关键路径移至 worker thread**
- 只要 worker thread 有空闲周期，就能看到显著改进
- 流送系统更容易跟上角色移动速度

---

## 6. Unified Streaming Budget（统一流送预算）

### 问题

60fps 开放世界最重要的概念是**预算管理**。基础实现需要管理三个独立预算：

![Unified Budget Intro](assets/streaming-improvements/18-unified-budget-intro.jpg)

- **AddToWorld** 预算
- **RemoveFromWorld** 预算
- **ProcessAsyncLoading** 预算

这三个预算可能在同一帧内执行。Demo 初始需要预留 **2.5ms**（占 16.6ms 帧的 15%），大量预算在空闲帧被浪费。

### 流送延迟分析（60fps 下）

![Budget 51 Frames Analysis](assets/streaming-improvements/21-budget-51-frames.jpg)

加载一个 level 小块的完整过程：
1. **AsyncLoading**（异步加载线程）：5 帧加载资源
2. **ProcessAsyncLoading**（GameThread）：29 帧 / ~500ms 处理 postload
3. **AddToWorld**（GameThread）：24 帧 / ~400ms 添加到世界

总计 **51 帧延迟**，且大部分帧只利用了一个预算，其余预算被浪费。

### 解决方案

![Unified Budget CVars and Code](assets/streaming-improvements/22-unified-budget-cvars.jpg)

与 Ben Ziggler 实现的 **统一流送预算**：

1. 计算统一预算 = 三个预算之和
2. 先执行 ProcessAsyncLoading（保证最低预算）
3. 剩余预算分配给 AddToWorld + RemoveFromWorld（单一时间预算下执行）
4. 如果 Add/Remove 未用完预算，再次运行 ProcessAsyncLoading 为下一帧准备数据

### 效果

![Unified Budget Results — 40% Reduction](assets/streaming-improvements/24-unified-budget-results.jpg)

- 预算从 **2.5ms 降至 1.5ms**
- 流送响应延迟**减少 40%**（52 帧 → 32 帧）
- ProcessAsyncLoading 在不 add/remove 的帧中可使用全部预算
- ProcessAsyncLoading 中的 hitch 被同帧内 UpdateLevelStreaming 的时间减少所**摊销**

---

## 7. FastGeo Streaming Plugin（UE 5.6+，实验性）

### 动机

静态几何体（HLOD actors、ISM actors）经历与复杂 gameplay actor 相同的注册过程，但静态几何体**不需要这种开销**。目标：

- 将静态几何体从常规 Actor/Component 注册流程中移除
- 在**关键路径之外**流送
- 对编辑器工作流无影响，保留完整 World Partition 功能

### 内容生成 — Runtime Cell Transformer

![FastGeo Content Generation](assets/streaming-improvements/27-fastgeo-content-gen.jpg)

FastGeo 自带 Runtime Cell Transformer：

- 提取静态几何体，转换为**轻量级非 UObject 数据结构**
- 内容存储在与非转换内容相同的 level 中（同步流送）
- 非破坏性，无需离线 builder
- Actor 可**部分或完全转换**，完全转换的 Actor 直接从 level 移除

![FastGeo Cell Transformer Config](assets/streaming-improvements/fastgeo-cell-transformer-config.jpg)

> **注意：** FastGeo transformer 不自动将 StaticMeshActor 转换为 ISM。需要在 transformer stack 中将 ISM Cell Transformer 放在 FastGeo Transformer 之前。

### 运行时流送

![FastGeo Runtime Streaming — Decoupling](assets/streaming-improvements/fastgeo-runtime-decoupling.jpg)

依赖的系统适配：
- **渲染**：基于 Render Proxy 解耦
- **物理**：异步物理状态创建/销毁 + Physics Body Instance Owner Interface
- **HLODs**：HLOD Subsystem 从 HLOD Actors 解耦
- **Render Asset Streaming Manager**：无需 Primitive Components

![FastGeo Runtime Streaming — Flow Chart](assets/streaming-improvements/fastgeo-runtime-flow.jpg)

AddToWorld 时的流程：
1. FastGeo 被通知，找到 level 中的 FastGeo 内容
2. 启动**异步任务**创建物理和渲染状态
3. 同时，非转换的 Actor/Component 通过常规注册流程处理
4. 通过新 delegate 监控内部任务完成状态
5. AddToWorld 合并 FastGeo 和常规注册的状态来判断是否完成

### 限制

![FastGeo Limitations](assets/streaming-improvements/31-fastgeo-limitations.jpg)

- 不支持 Replicated Actors
- 不支持有逻辑的 Blueprint Actors
- 被非转换/部分转换 Actor 引用的 Actor 也会被排除
- Actor Component Mobility 必须为 Static
- 不支持 Child Actors、Non-Static Mobility RootComponent、Editor-only Actors

### 调试工具

![FastGeo Debug Tools](assets/streaming-improvements/fastgeo-debug-tools-clean.jpg)

| 工具 | 说明 |
|------|------|
| `show ActorColoration FastGeo` | 蓝色 = FastGeo 内容，红色 = 未转换的 primitives |
| `FastGeo.Show` | 隐藏/显示 FastGeo 内容，隔离非转换内容 |
| `FastGeo.EnableTransformerDebugMode` | 日志输出转换详情和排除原因 |

### Console Variables

![FastGeo Console Variables](assets/streaming-improvements/49-ps5-drone-metrics.jpg)

| CVar | 用途 |
|------|------|
| `FastGeo.Enable` | 启用功能（PIE/Cook） |
| `FastGeo.EnableTransformerDebugMode` | 启用 Transformer 日志 |
| `FastGeo.Show` | 隐藏/显示 FastGeo 内容 |
| `FastGeo.AsyncRenderStateTask.ParallelWorkerCount` | 控制 worker 数量 |
| `FastGeo.AsyncRenderStateTask.TimeBudgetMS` | 控制 worker 预算 |
| `FastGeo.AsyncRenderStateTask.MaxNumComponentsToProcess` | 单次处理最大 component 数 |

### Demo 结果

![FastGeo Demo Scene — FastGeo.Show 0](assets/streaming-improvements/fastgeo-demo-scene.jpg)

**Witcher 4 Tech Demo：**
- **91% 的 components** 被转换
- **86% 的 actors** 完全转换，1% 部分转换
- 未转换内容：SkeletalMesh、gameplay 元素、非 static mobility

![FastGeo Memory Comparison](assets/streaming-improvements/fastgeo-memory-comparison.jpg)

![FastGeo Performance — 0.8ms Budget](assets/streaming-improvements/fastgeo-perf-graph-08ms.jpg)

最终 Demo 全部流送预算仅 **0.8ms**。

### City Sample 测试（未修改内容）

![City Sample Stats — 92%/94%/3%](assets/streaming-improvements/city-sample-stats.jpg)

- **92% components** 转换，**94% actors** 完全转换，3% 部分转换

![City Sample PS5 Performance Intro](assets/streaming-improvements/city-sample-perf-intro.jpg)

PS5 自动化测试（无人机 215 km/h 飞行）：

![PS5 Drone Metrics Table](assets/streaming-improvements/ps5-drone-metrics-table.jpg)

- AddToWorld: 3224ms → 240ms (**92.55% gain**)
- RemoveFromWorld: 1696ms → 112ms (**93.34% gain**)
- UObject 大幅减少 → GC 和 ProcessAsyncLoading 负担减轻

![PS5 AddToWorld Comparison Graph](assets/streaming-improvements/ps5-addtoworld-graph.jpg)

### 极限测试

![540 km/h Drone Stress Test](assets/streaming-improvements/drone-540kmh-intro.jpg)

无人机速度提升至 **540 km/h**，预算仅 2ms：
- 无 FastGeo：游戏暂停等待流送追赶
- **有 FastGeo：流送始终保持领先，画面流畅无中断**

---

## 8. Texture & Mesh Streaming 改进

### 问题

![Texture & Mesh Streaming Intro](assets/streaming-improvements/texture-streaming-intro.jpg)

传统 Texture & Mesh Streaming 是 CPU 密集型过程：

![Texture Streaming Registration](assets/streaming-improvements/texture-streaming-registration.jpg)

- 每次加载 level 需从所有 Actor → Component → Material → Texture 逐层提取信息（大量嵌套循环）
- 需要跟踪所有使用特定资源的 component 的 bounds 以估算屏幕空间投影
- 注册过程在**游戏线程关键路径**上

### 优化尝试

![Texture Streaming with Cache](assets/streaming-improvements/texture-streaming-cache.jpg)

实现了 texture sampling streaming cache 和部分并行处理，但仍难以将 spike 降至 1ms 以下。

![Texture Streaming — Can we do better?](assets/streaming-improvements/texture-streaming-questions.jpg)

### Simple Streamable Asset Manager

![Simple Streamable Asset Manager](assets/streaming-improvements/simple-streamable-manager.jpg)

**Simple Streamable Asset Manager** — 渲染资源流送管理器的附加模块：
- 用 **Scene Proxy** 替代基于 Component 的注册过程
- 可接收任意数据提供必要信息（支持 FastGeo 的非 UPrimitiveComponent 对象）
- **注册成本从关键路径移除**
- 所有处理**异步在后台执行**
- 更优的内存访问模式，处理速度更快

> 即使不使用 FastGeo，也推荐在项目中检查此模块。

启用方式：`s.StreamableAssets.UseSimpleStreamableAssetManager=True`

---

## 9. UE 5.6 其他流送改进

![Additional Streaming Improvements](assets/streaming-improvements/additional-improvements.jpg)

| 改进 | CVar |
|------|------|
| **WP Update Streaming State 异步化** | `wp.Runtime.UpdateStreaming.EnableAsyncUpdate = true` |
| **EndPlay 增量执行** | `s.LevelStreamingRouteActorEndPlayForRemoveFromWorldGranularity = <value>` |
| **Game Thread 外添加 Primitives** | `LevelStreaming.AsyncRegisterLevelContext.Enabled = true` |
| **ISM Component Bounds Cook-time 缓存** | 减少 ISM 注册时间 |

---

## 10. 未来研究方向

![Future Research Ideas](assets/streaming-improvements/future-research.jpg)

- **FastGeo 持续开发**：支持更多 primitive 类型、支持可移动对象、打通 FastGeo 和 Turbo Streaming
- **Mutators 扩展**：可能允许 mutate 更多属性，如 HLOD 分配
- **高级 Transformers**：分离碰撞与图形表示到不同 grid 独立流送（Stream physics separately）
- **GPU Sampler Feedback**：类似 Virtual Textures，利用采样器反馈驱动纹理流送，大幅降低 CPU 开销

---

## 总结

![Key Takeaways](assets/streaming-improvements/summary-key-takeaways.jpg)

| 技术 | 一句话总结 |
|------|-----------|
| **Mutators** | 自动化 World Partition grid 分配 |
| **Runtime Hash Set** | 支持 3D 流送，允许实验新策略 |
| **Cell Transformers** | 优化已构建的内容（SM → ISM 等） |
| **Async Physics State** | 将重物理处理移至 worker thread |
| **Unified Streaming Budget** | 合并预算，更高效利用帧时间 |
| **FastGeo** | 高效流送静态几何体，绕过 Actor/Component 开销 |
| **Simple Streamable Asset Manager** | 后台异步执行纹理/网格流送管理 |

---

## 参考链接

- [YouTube 视频](https://www.youtube.com/watch?v=BdopUm1_1_E)
- [Epic Dev Community 页面](https://dev.epicgames.com/community/learning/talks-and-demos/KWGD/)
- [FastGeo Streaming API 文档](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/PluginIndex/FastGeoStreaming)
- [UE Public Roadmap - Fast Geometry Streaming Plugin](https://portal.productboard.com/epicgames/1-unreal-engine-public-roadmap/c/2022-fast-geometry-streaming-plugin-experimental-)
- [hzFishy 社区笔记 - FastGeo](https://notes.hzfishy.fr/Unreal-Engine/Engine--and--Editor/Content-Streaming/Fast-Geo)
