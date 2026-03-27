---
title: "FastGeo Streaming 性能测试"
date: 2026-03-27
tags: [UE, FastGeo, Streaming, Benchmark, World-Partition]
category: projects
description: "FastGeo 开启/关闭的 A/B 对比测试，分别测试 StaticMeshActor 和 PCG InstancedStaticMeshActor 两种场景"
status: done
---

# FastGeo Streaming 性能测试

## 测试环境

- UE 5.7, i7-14700, D3D12 SM6, Ray Tracing ON
- 128m Grid Size, World Partition
- 相同 Spline 路径、相同移动速度（自动控制，确保路径一致性）
- 测试工具：UE CSV Profiler + 自定义帧级数据采集（见 [[Claude 自主创建 C++ 蓝图的逻辑]]）

## 测试一：StaticMeshActor 场景

场景内全部为 StaticMeshActor（手动放置的独立静态网格体）。

![[Pasted image 20260327105249.png]]

### Level Streaming 开销曲线

**FastGeo ON：**

![[Pasted image 20260327111906.png]]

**FastGeo OFF：**

![[Pasted image 20260327112030.png]]

### 数据对比

| 指标 | FastGeo ON | FastGeo OFF | 差异 |
|------|-----------|------------|------|
| 总 Streaming 开销 | 55.7 ms | 111.7 ms | **-50%** |
| 单帧峰值 | 2.0 ms | 5.6 ms | **-64%** |
| 帧 >= 1ms | 6 | 30 | **-80%** |
| 帧 >= 2ms | 1 | 19 | **-95%** |
| AddToWorld 峰值 | 1.67 ms | 5.39 ms | **-69%** |
| AddToWorld 总耗时 | 19.2 ms | 70.2 ms | **-73%** |

### 结论

**FastGeo 效果显著。** AddToWorld 是最大受益点——这正是 FastGeo 的核心价值：绕过 Actor/Component 的注册流程。不启用时 AddToWorld 峰值达到 5.39 ms，有 17 帧超过 2ms；启用后峰值降到 1.67 ms，没有任何帧超过 2ms。

综合流送成本（ULS + AddToWorld）总量从 111.7 ms 降到 55.7 ms（-50%），超过 1ms 的帧从 30 帧降到 6 帧（-80%）。

---

## 测试二：PCG InstancedStaticMeshActor 场景

场景内全部为 PCG 生成的 InstancedStaticMeshActor（PCG 树林，Partitioned 模式）。

**FastGeo ON：**

![[Pasted image 20260327194955.png]]

**FastGeo OFF：**

![[Pasted image 20260327195322.png]]

### 整体性能

| 指标 | FastGeo ON | FastGeo OFF | 差异 |
|------|-----------|------------|------|
| 总帧数 | 4268 | 4362 | +94 |
| 平均 FPS | ~118 | ~118 | 持平 |
| 平均帧时间 | ~8.5ms | ~8.5ms | 持平 |

### Level Streaming 开销对比

| 指标 | FastGeo ON | FastGeo OFF | 差异 |
|------|-----------|------------|------|
| AddToWorld 次数 | 22 帧 | 22 帧 | 持平 |
| AddToWorld 最大单帧 | 2.90ms | 3.68ms | **ON 低 21%** |
| AddToWorld 平均 | 1.02ms | 0.99ms | 持平 |
| AddToWorld 总耗时 | 22.36ms | 21.68ms | 持平 |
| NumLevelsLoading | 12 帧 | 14 帧 | ON 略少 |
| NumLevelsMakingVisible | 3 帧 | 2 帧 | - |
| NumLevelsMakingInvisible | 0 | 1 帧 | - |
| NumLevelsPendingPurge 最大 | 21 | 13 | ON 更多待清除 |
| UpdateStreamingState 总耗时 | 162.6ms | 162.3ms | 持平 |
| UpdateLevelStreaming 总耗时 | 22.3ms | 22.8ms | 持平 |
| RenderAssetStreaming 总耗时 | 324.9ms | 325.9ms | 持平 |

### 场景复杂度

| 指标 | FastGeo ON | FastGeo OFF |
|------|-----------|------------|
| PCGPartitionActor 平均 | 56.8 | 55.9 |
| PCGPartitionActor 最大 | 66 | 60 |
| 总 Actor 最大 | 302 | 286 |
| SceneCulling Added | 1322 | 1318 |
| SceneCulling Removed | 1403 | 1399 |

### 结论

PCG InstancedStaticMeshActor 场景下 **FastGeo 收益不明显**。AddToWorld 峰值降低了 21%（3.68ms → 2.90ms），但平均值和总耗时基本持平。FPS 无可感知差异。

原因推测：PCG 的 InstancedStaticMeshActor 已经通过实例化合批减少了 Actor 数量，FastGeo 绕过 Actor 注册的优势被稀释了。FastGeo 的核心价值在于减少**大量独立 Actor** 的注册开销，当 Actor 数量本身不多时（通过 ISM 合并），收益自然有限。

## 综合分析

| 场景类型 | FastGeo 收益 | 原因 |
|---------|-------------|------|
| 大量独立 StaticMeshActor | **显著**（-50% ~ -80%） | 每个 Actor 都需要走注册流程，FastGeo 直接绕过 |
| PCG InstancedStaticMeshActor | **微弱**（仅峰值 -21%） | ISM 已合批，Actor 数量少，注册开销本就不大 |

**结论：FastGeo 的价值与场景中独立 Actor 的数量正相关。** 对于已经使用 ISM/HISM 合批的场景，FastGeo 的边际收益递减。在规划场景时，应优先考虑 ISM 合批策略，FastGeo 作为补充手段处理无法合批的独立 Actor。
