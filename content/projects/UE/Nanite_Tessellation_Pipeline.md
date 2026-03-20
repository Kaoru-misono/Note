# Nanite Tessellation Pipeline 完整流程分析

> 基于 NeonEngine560 (UE 5.6) 源码分析
> 主要涉及文件均位于 `Engine/Source/Runtime/Renderer/Private/Nanite/` 和 `Engine/Shaders/Private/Nanite/`

---

## 目录

1. [整体架构概览](#1-整体架构概览)
2. [CPU 侧：Dispatch 构建与分流](#2-cpu-侧dispatch-构建与分流)
3. [GPU Pass 1：SW Rasterize (Tessellated) — 初始评估与即时 Dice](#3-gpu-pass-1sw-rasterize-tessellated--初始评估与即时-dice)
4. [GPU Pass 2：PatchSplit — 递归细分](#4-gpu-pass-2patchsplit--递归细分)
5. [GPU Pass 3：SW Rasterize (Patches) — 细分后光栅化](#5-gpu-pass-3sw-rasterize-patches--细分后光栅化)
6. [TessFactor 计算详解](#6-tessfactor-计算详解)
7. [LowTess 远距离衰减机制](#7-lowtess-远距离衰减机制)
8. [Displacement Fade 与 Fallback 机制](#8-displacement-fade-与-fallback-机制)
9. [Tessellation Table（预计算查找表）](#9-tessellation-table预计算查找表)
10. [核心数据结构](#10-核心数据结构)
11. [性能分析与优化建议](#11-性能分析与优化建议)
12. [RenderDoc 中的对应关系](#12-renderdoc-中的对应关系)

---

## 1. 整体架构概览

Nanite Tessellation 是 UE 5.5+ 引入的 Compute Shader 驱动的 tessellation 管线，**完全绕过传统的 Hull/Domain Shader 固定管线**，通过 SW rasterizer 实现。

### 核心设计理念

- **不使用硬件 Tessellation 单元**：全部在 Compute Shader 中完成细分 + 光栅化
- **基于 Nanite Cluster DAG**：在 Nanite 现有的 LOD 层级上叠加 displacement tessellation
- **预计算 Tessellation Table**：通过 LUT 避免运行时的 remesh 计算
- **两级细分策略**：低 TessFactor 即时 dice，高 TessFactor 递归 PatchSplit

### 总体流程图

```
┌──────────────────────────────────────────────────────────────────┐
│                  CPU 侧 (NaniteCullRaster.cpp)                   │
│                                                                  │
│  遍历 RasterBin → 检查 NANITE_MATERIAL_FLAG_DISPLACEMENT         │
│       ↓ Yes                          ↓ No                        │
│  Dispatches_SW_Tessellated     Dispatches_SW_Triangles           │
│  (永不走 HW Rasterize)         + Dispatches_HW_Triangles         │
└────────────┬─────────────────────────────────────────────────────┘
             │
             ▼
┌──────────────────────────────────────────────────────────────────┐
│  GPU Pass 1: SW Rasterize (Tessellated)  [Patches=false]         │
│  ── NaniteRasterizer.usf::ClusterRasterize() ──                  │
│                                                                  │
│  对每个 tessellated cluster 的每个三角形:                        │
│    1. Vertex Fetch + Transform + WPO                             │
│    2. 计算 TessFactor (基于屏幕空间边长)                         │
│    3. 判断 bCanDice:                                             │
│       ┌─ TessFactor ≤ IMMEDIATE_SIZE ──→ 即时 Dice + Rasterize   │
│       └─ TessFactor > IMMEDIATE_SIZE ──→ 写入 SplitWorkQueue     │
└────────────┬───────────────┬─────────────────────────────────────┘
             │               │
    (低TessFactor,           (高TessFactor,
     远景三角形)               近景三角形)
             │               │
             │               ▼
             │  ┌──────────────────────────────────────────────────┐
             │  │  GPU Pass 2: PatchSplit                          │
             │  │  ── NaniteSplit.usf::PatchSplit() ──             │
             │  │                                                  │
             │  │  递归处理 SplitWorkQueue:                        │
             │  │    读取 patch → 计算子 TessFactor                │
             │  │    ├─ TessFactor ≤ TABLE_SIZE → VisiblePatches   │
             │  │    └─ TessFactor > TABLE_SIZE → 继续 Split       │
             │  └──────────────────┬───────────────────────────────┘
             │                     │
             │                     ▼
             │  ┌──────────────────────────────────────────────────┐
             │  │  GPU Pass 3: SW Rasterize (Patches)[Patches=true]│
             │  │  ── NaniteRasterizer.usf::PatchRasterize() ──    │
             │  │                                                  │
             │  │  读取 VisiblePatches → Dice + Displacement       │
             │  │  → 最终 SW 光栅化                                │
             │  └──────────────────────────────────────────────────┘
             │
             ▼
        ┌────────────────┐
        │ 最终 VisBuffer │  (所有路径的输出汇合)
        └────────────────┘
```

---

## 2. CPU 侧：Dispatch 构建与分流

**文件**: `NaniteCullRaster.cpp`

### 2.1 FDispatchContext 结构 (Line 3185)

```cpp
struct FDispatchContext
{
    FDispatchList Dispatches_HW_Triangles;      // 硬件光栅化 (非 displacement)
    FDispatchList Dispatches_SW_Triangles;      // 软件光栅化 (非 displacement)
    FDispatchList Dispatches_SW_Tessellated;    // 软件光栅化 (tessellation)  ← 关键
    // ...
    bool HasTessellated() const;
    void DispatchSW(..., bool bPatches) const;  // bPatches 控制走哪个 shader 入口
};
```

### 2.2 材质级别的分流 (Line 5557-5568)

```cpp
const int32 PassIndex = Context.RasterizerPasses.Num() - 1;

if (RasterizerPass.bDisplacement)  // Line 5559
{
    // Displaced meshes 永远不走 HW 路径
    Context.Dispatches_SW_Tessellated.Indirections.Emplace(PassIndex);
}
else
{
    Context.Dispatches_SW_Triangles.Indirections.Emplace(PassIndex);
    Context.Dispatches_HW_Triangles.Indirections.Emplace(PassIndex);
}
```

**关键点**: `bDisplacement` 来自材质的 `NANITE_MATERIAL_FLAG_DISPLACEMENT` 标志 (Line 5151)。只要材质开启了 displacement，该材质的**所有 cluster（无论远近）**都进入 `Dispatches_SW_Tessellated`，**永远不走 HW Rasterize 路径**。

### 2.3 FRasterizerPass 结构 (Line 2291)

```cpp
struct FRasterizerPass
{
    TShaderRef<FMicropolyRasterizeCS> ClusterComputeShader;  // Patches=false 时使用
    TShaderRef<FMicropolyRasterizeCS> PatchComputeShader;    // Patches=true 时使用
    bool bDisplacement = false;
    // ...
};
```

### 2.4 三次 Dispatch 调用

**Dispatch 1 — SW Rasterize (Tessellated)** (Line 5762-5783):
```cpp
DispatchContext.DispatchSW(
    RHICmdList,
    DispatchContext.Dispatches_SW_Tessellated,
    SceneView, PSOCollectorIndex,
    *ClusterPassParameters,
    false /* Patches */   // ← ClusterRasterize 入口
);
```

**Dispatch 2 — PatchSplit** (Line 5892-5989):
```cpp
// InitPatchSplitArgs → 初始化递归 split 参数
// PatchSplit CS → 递归细分
```

**Dispatch 3 — SW Rasterize (Patches)** (Line 5865-5883):
```cpp
DispatchContext.DispatchSW(
    RHICmdList,
    DispatchContext.Dispatches_SW_Tessellated,
    SceneView, PSOCollectorIndex,
    *PatchPassParameters,
    true /* Patches */    // ← PatchRasterize 入口
);
```

### 2.5 Tessellation Table 全局资源 (Line 3029-3059)

```cpp
class FTessellationTableResources : public FRenderResource
{
    FByteAddressBuffer Offsets;          // Pattern → (TableOffset, NumVerts, NumTris) 查找表
    FByteAddressBuffer VertsAndIndexes;  // 预计算的 barycentric 顶点 + 三角形索引
};

TGlobalResource<FTessellationTableResources> GTessellationTable;
```

在 Shader 参数中绑定 (Line 5715-5716):
```cpp
RasterPassParameters->TessellationTable_Offsets = GTessellationTable.Offsets.SRV;
RasterPassParameters->TessellationTable_VertsAndIndexes = GTessellationTable.VertsAndIndexes.SRV;
```

---

## 3. GPU Pass 1：SW Rasterize (Tessellated) — 初始评估与即时 Dice

**文件**: `NaniteRasterizer.usf`，函数 `ClusterRasterize()`，`NANITE_TESSELLATION` 路径 (Line 420-528)

### 3.1 初始化

```hlsl
// Line 420-424
#if NANITE_TESSELLATION
    float LowTessDistance = 0.0f;
#if USES_DISPLACEMENT
    LowTessDistance = CalcDisplacementLowTessDistance(PrimitiveData, InstanceData, NaniteView);
#endif
```

### 3.2 逐三角形处理

```hlsl
// Line 426-430
uint TriIndex = TriRange.Start + GroupThreadIndex;
bool bTriValid = GroupThreadIndex < TriRange.Num;
```

对 cluster 中的每个三角形：

#### Step 1: 计算 View 空间位置
```hlsl
// Line 438+ : 顶点 Fetch + Transform
uint3 VertIndexes = DecodeTriangleIndices(Cluster, TriIndex);
// 变换到 view space
float3 TriPointView[3] = { ... };
```

#### Step 2: 计算 TessFactor
```hlsl
// Line 454
float3 TessFactors = GetTessFactors(NaniteView, TriPointView, LowTessDistance);
```

#### Step 3: 判断 bCanDice
```hlsl
// Line 458
bool bCanDice = max3(TessFactors.x, TessFactors.y, TessFactors.z)
                <= NANITE_TESSELLATION_TABLE_IMMEDIATE_SIZE;
```

- **`NANITE_TESSELLATION_TABLE_IMMEDIATE_SIZE`**: 即时 dice 表的最大 TessFactor（约 14）
- **`NANITE_TESSELLATION_TABLE_SIZE`**: 完整 tessellation 表的最大 TessFactor（约 14）

#### Step 4a: 即时 Dice 路径 (bCanDice = true)

```hlsl
// Line 460-490
if (bCanDice)
{
    FDiceTask DiceTask;
    DiceTask.Init(TessFactors, VisibleIndex, TriIndex);
    NumVerts = DiceTask.TessellatedPatch.GetNumVerts();
    NumTris  = DiceTask.TessellatedPatch.GetNumTris();

    // 统计
    WaveInterlockedAdd(OutStatsBuffer[0].NumDicedTrianglesClusters, NumTris);
    WaveInterlockedAddScalar(OutStatsBuffer[0].NumImmediatePatches, 1);

    // 分配工作到 wave 线程
    DistributeWork(DiceTask, GroupThreadIndex, NumTris);
}
```

DiceTask 内部流程 (`NaniteDice.ush` Line 141-283):
1. `TessellatedPatch.GetVert(LocalItemIndex)` → 获取 barycentric 坐标
2. `Shader.EvaluateDomain(UVDensities, Barycentrics)` → **采样 displacement + 材质**
3. 转换到屏幕空间 subpixel 坐标
4. `RasterizeDicedTri()` → **直接光栅化到 VisBuffer**

#### Step 4b: Enqueue Split 路径 (bCanDice = false)

```hlsl
// Line 509-527
else if (bTriValid && !bCanDice)
{
    uint WriteOffset = SplitWorkQueue.Add();
    if (WriteOffset < SplitWorkQueue.Size)
    {
        uint4 Encoded;
        Encoded.x = (VisibleIndex << 7) | TriIndex;   // cluster + tri 索引
        Encoded.y = BarycentricMax;                     // 初始 barycentric (整个三角形)
        Encoded.z = BarycentricMax << 16;
        Encoded.w = 0;

        SplitWorkQueue.DataBuffer_Store4(WriteOffset * 16, Encoded);
    }
}
```

**这就是为什么 SW Rasterize (Tessellated) 在 RenderDoc 中只显示远景**：
- 远景三角形 TessFactor 低 → `bCanDice=true` → 即时光栅化 → **有像素输出**
- 近景三角形 TessFactor 高 → `bCanDice=false` → 仅写入 SplitWorkQueue → **无像素输出**
- 但 GPU time 包含了对**所有三角形**的评估开销

---

## 4. GPU Pass 2：PatchSplit — 递归细分

**文件**: `NaniteSplit.usf`

### 4.1 FSplitTask 结构 (Line 52-72)

```hlsl
struct FSplitTask
{
    // Load/Store/Clear methods for SplitWorkQueue interaction
    void Load(uint QueueOffset);
    void Store(uint QueueOffset);
};
```

### 4.2 递归 Split 逻辑 (Line 117-290)

```hlsl
void FSplitTask::Run()
{
    // 1. 从 SplitWorkQueue 读取 encoded patch
    uint4 Encoded = SplitWorkQueue.DataBuffer_Load4(QueueOffset * 16);

    // 2. 解码 FSplitPatch
    FSplitPatch SplitPatch;
    SplitPatch.Decode(Encoded);

    // 3. 获取 cluster 信息, 计算 view 空间位置
    FVisibleCluster VisibleCluster = GetVisibleCluster(SplitPatch.VisibleClusterIndex);
    // ...

    // 4. 计算 TessFactor
    float3 TessFactors = GetTessFactors(NaniteView, CornersView, LowTessDistance);

    // 5. 判断是否需要继续 split
    bool bNeedsSplitting = max3(TessFactors.x, TessFactors.y, TessFactors.z)
                           > NANITE_TESSELLATION_TABLE_SIZE;

    if (!bNeedsSplitting)
    {
        // 写入 VisiblePatches → 后续由 SW Rasterize (Patches) 光栅化
        uint WriteOffset = RWVisiblePatchesArgs.Add();

        // PATCH_REFS 模式：存储引用
        RWVisiblePatches.Store2(WriteOffset * 8,
            uint2(QueueOffset, TessellatedPatch.GetPattern()));

        // 非 PATCH_REFS 模式：存储完整 encoded data
        RWVisiblePatches.Store4(WriteOffset * 16, Encoded);
    }
    else
    {
        // 继续递归 split → 生成 4 个子三角形
        // CreateChild() → 新 patch 写回 SplitWorkQueue
    }
}
```

### 4.3 子三角形生成 (Line 292-343)

```hlsl
void FSplitTask::CreateChild()
{
    // 将父三角形沿中点二等分为 4 个子三角形
    // 变换 barycentric 坐标
    // 写入 SplitWorkQueue 的下一层
    uint WriteOffset = SplitWorkQueue.Add();
    SplitWorkQueue.DataBuffer_Store4(WriteOffset * 16, ChildEncoded);

    // 更新下一层级的 indirect args
    RWPatchSplitArgs.InterlockedAdd(...);
}
```

### 4.4 PatchSplit 入口 (Line 345-378)

```hlsl
[numthreads(THREADGROUP_SIZE, 1, 1)]
void PatchSplit(uint GroupID, uint GroupIndex)
{
    // 多层级递归处理
    // 每层读取 SplitWorkQueue → Run() → 输出到 VisiblePatches 或继续 split
    FSplitTask Task;
    Task.Load(QueueOffset);
    Task.Run();
    DistributeWork(Task, GroupIndex, NumChildTris);
}
```

---

## 5. GPU Pass 3：SW Rasterize (Patches) — 细分后光栅化

**文件**: `NaniteRasterizer.usf`，函数 `PatchRasterize()` (Line 776-995)

### 5.1 入口

```hlsl
// Line 1726 (主入口选择)
#if PATCHES
    PatchRasterize(GroupID, GroupIndex);
#else
    ClusterRasterize(GroupID, GroupIndex);
#endif
```

### 5.2 Patch 数据加载

```hlsl
// Line 785-817
const uint TotalPatches = RasterBinMeta[GetRasterBin()].BinSWCount;

// 从 VisiblePatches 读取
#if NANITE_TESSELLATION_PATCH_REFS
    // 引用模式: 读取 SplitWorkQueue 的 offset + pattern
    const uint2 VisiblePatch = VisiblePatches.Load2(VisibleIndex * 8);
    Patches_EncodedPatch = SplitWorkQueue.DataBuffer_Load4(VisiblePatch.x * 16);
#else
    // 直接模式: 读取完整的 encoded patch
    Patches_EncodedPatch = VisiblePatches.Load4(VisibleIndex * 16);
#endif
```

### 5.3 Patch 设置

```hlsl
// Line 820-876
Patches_SplitPatch.Decode(Patches_EncodedPatch);

// 获取 cluster + primitive 信息
FVisibleCluster VisibleCluster = GetVisibleCluster(Patches_SplitPatch.VisibleClusterIndex);
FCluster Cluster = GetCluster(VisibleCluster.PageIndex, VisibleCluster.ClusterIndex);

// 顶点 fetch + transform + WPO
const uint3 VertIndexes = DecodeTriangleIndices(Cluster, Patches_SplitPatch.TriIndex);
Patches_Verts = FetchTransformedNaniteVertex(..., VertIndexes[PatchCornerIndex], bEvaluateWPO);

// 计算 TessFactor (基于子 patch 的 view 空间大小)
const float3 TessFactors = GetTessFactors(NaniteView, CornersView, LowTessDistance);
Patches_TessellatedPatch.Init(TessFactors, Patches_EncodedPatch.yzw, false);
```

### 5.4 Dice + 光栅化循环

```hlsl
// Line 879-992
for (uint i = 0; i < NumPatches; i++)
{
    // 读取当前 patch 的数据 (wave broadcast)
    const FSplitPatch SplitPatch = WaveReadLaneAt(Patches_SplitPatch, PatchStartLane);
    const FTessellatedPatch TessellatedPatch = WaveReadLaneAt(Patches_TessellatedPatch, PatchStartLane);

    // 遍历 tessellated patch 的所有顶点
    for (uint v = GroupThreadIndex; v < TessellatedPatch.GetNumVerts(); v += ThreadGroupSize)
    {
        // 获取 barycentric → 变换到父三角形空间
        FBarycentrics Barycentrics = TessellatedPatch.GetVert(v);
        Barycentrics = SplitPatch.TransformBarycentrics(Barycentrics);

        // 评估 domain (采样 displacement)
        float3 PointClip = MaterialShader.EvaluateDomain(UVDensities, Barycentrics).xyz;

        // 缓存到 GroupVerts (LDS)
        GroupVerts[v] = PointClip;
    }

    GroupMemoryBarrierWithGroupSync();

    // 遍历所有 tessellated 三角形 → 光栅化
    for (uint t = GroupThreadIndex; t < TessellatedPatch.GetNumTris(); t += ThreadGroupSize)
    {
        uint3 Indexes = TessellatedPatch.GetIndexes(t);
        float3 Verts[3] = { GroupVerts[Indexes.x], GroupVerts[Indexes.y], GroupVerts[Indexes.z] };

        FRasterTri Tri = SetupTriangle(Raster.ScissorRect, Verts);
        RasterizeDicedTri(Tri, Raster, Shader, ...);
    }
}
```

---

## 6. TessFactor 计算详解

**文件**: `NaniteTessellation.ush` Line 62-101

### 6.1 GetTessFactors()

```hlsl
float3 GetTessFactors(FNaniteView NaniteView, float3 PointView[3], float LowTessDistance)
{
    // 判断投影类型
    bool bOrtho = NaniteView.ViewToClip[3][3] >= 1;

    float3 TessFactors;
    bool3 bDistant;

    for (int i = 0; i < 3; i++)
    {
        int i0 = i;
        int i1 = (i + 1) % 3;

        // 边长 (view 空间 3D 距离)
        float EdgeLength = distance(PointView[i0], PointView[i1]);

        // 缩放因子 (透视校正)
        float EdgeScale;
        if (bOrtho)
            EdgeScale = 1.0f;
        else
            EdgeScale = max(min(PointView[i0].z, PointView[i1].z), NearPlane);

        // 屏幕空间边长 (像素)
        TessFactors[i] = EdgeLength / EdgeScale;

        // 距离判断 (用于 LowTess 衰减)
        bDistant[i] = EdgeScale > LowTessDistance;
    }

    // 最终 TessFactor = 屏幕边长 × LODScale × DiceRate × LowTess衰减
    const float LowTessMultiple = 0.25f;
    TessFactors *= NaniteView.LODScale * InvDiceRate
                 * select(bDistant, LowTessMultiple, 1.0f);

    return TessFactors;
}
```

### 6.2 计算原理

```
TessFactor[edge] = (ViewSpaceEdgeLength / ViewDepth) × LODScale × InvDiceRate × LowTessMultiple

其中:
  ViewSpaceEdgeLength = 三角形边在 view 空间的 3D 长度
  ViewDepth           = 边两端点中较近的 Z 值 (透视校正)
  LODScale            = 全局 LOD 缩放 (受分辨率、FOV 影响)
  InvDiceRate         = 1.0 / DiceRate (受 r.Nanite.MaxPixelsPerEdge 影响)
  LowTessMultiple     = 0.25 (远距离) 或 1.0 (近距离)
```

本质上就是：**三角形边在屏幕上投影的像素长度**，决定了需要细分成多少段。

---

## 7. LowTess 远距离衰减机制

**文件**: `NaniteTessellation.ush` Line 49-60, 96-97

### 7.1 CalcDisplacementLowTessDistance()

```hlsl
float CalcDisplacementLowTessDistance(
    FPrimitiveSceneData PrimitiveData,
    FInstanceSceneData InstanceData,
    FNaniteView NaniteView)
{
    const float LowTessSize = PrimitiveData.MaterialDisplacementFadeOutSize;
    if (LowTessSize == 0.0f) return 0.0f;  // 禁用 LowTess

    const float MaxDisplacement = GetAbsMaxMaterialDisplacement(PrimitiveData);
    return (MaxDisplacement * InstanceData.NonUniformScale.w * NaniteView.LODScale) / LowTessSize;
}
```

### 7.2 机制说明

```
LowTessDistance = (MaxDisplacement × UniformScale × LODScale) / FadeOutSize

当三角形边端点的 view depth > LowTessDistance 时:
  TessFactor *= 0.25  (降低 4 倍)

效果:
  - 远处 displacement 在屏幕上投影很小时，减少 tessellation 密度
  - 但不会完全消除 (仍然 × 0.25，不是 × 0)
  - 只有 Displacement Fallback 才能完全消除 (见第 8 节)
```

**这解释了为什么远处仍然有 tessellation**：LowTess 只是降低了 TessFactor，但远处的 cluster 仍然走 `SW Rasterize (Tessellated)` 管线。

---

## 8. Displacement Fade 与 Fallback 机制

### 8.1 DisplacementFadeRange (CPU 侧)

**文件**: `NaniteCullRaster.cpp` Line 3062-3075

```cpp
static void CalcDisplacementFadeSizes(
    const FDisplacementFadeRange& Range,
    float& FadeSizeStart, float& FadeSizeStop)
{
    const float EdgesPerPixel = 1.0f / CVarNaniteMaxPixelsPerEdge.GetValueOnRenderThread();

    if (!Range.IsValid())
    {
        FadeSizeStart = FadeSizeStop = 0.0f;  // 不启用 fade
    }
    else
    {
        FadeSizeStop  = EdgesPerPixel * Max(Range.EndSizePixels, KINDA_SMALL_NUMBER);
        FadeSizeStart = EdgesPerPixel * Max(Range.StartSizePixels, Range.EndSizePixels + KINDA_SMALL_NUMBER);
    }
}
```

在材质参数中绑定 (Line 5143-5146):
```cpp
CalcDisplacementFadeSizes(
    *RasterMaterialCache.DisplacementFadeRange,
    BinMeta.MaterialDisplacementParams.FadeSizeStart,
    BinMeta.MaterialDisplacementParams.FadeSizeStop
);
```

### 8.2 Fallback 到 HW Rasterize

当 displacement 投影小于阈值时，cluster 可以 fallback 到普通 HW Rasterize：

**Culling Flag**: `NANITE_CULLING_FLAG_FALLBACK_RASTER`
- 在 culling shader 中设置
- 标记该 cluster 不需要 tessellation

**FixedDisplacementFallback**: (Line 5058, 5101)
```cpp
const bool bFixedDisplacementFallback = RasterizerPass.RasterPipeline.bFixedDisplacementFallback;
PermutationVectorCS_Cluster.Set<FMicropolyRasterizeCS::FFixedDisplacementFallbackDim>(bFixedDisplacementFallback);
```

**Per-Cluster Fallback**: `PRIMITIVE_SCENE_DATA_FLAG_PER_CLUSTER_DISPLACEMENT_FALLBACK_RASTER`
- 允许逐 cluster 判断是否 fallback
- 更精细的控制

### 8.3 三种距离状态

```
近距离 (depth < LowTessDistance):
  → TessFactor × 1.0 → 完整 tessellation

中距离 (depth > LowTessDistance, displacement 投影 > FadeOutSize):
  → TessFactor × 0.25 → 降低 tessellation

远距离 (displacement 投影 < FadeOutSize):
  → Fallback 到 HW Rasterize → 完全跳过 tessellation 管线
```

---

## 9. Tessellation Table（预计算查找表）

### 9.1 结构

Tessellation Table 是 CPU 端预计算的 LUT，存储了所有可能的 (TessFactorX, TessFactorY, TessFactorZ) 组合对应的：
- **顶点 barycentric 坐标**
- **三角形索引**

### 9.2 Pattern 编码 (NaniteTessellation.ush Line 175-179)

```hlsl
// Pattern = 三维 TessFactor 组合的线性索引
Pattern = TessFactors.x
        + TessFactors.y * TABLE_PO2_SIZE
        + TessFactors.z * TABLE_PO2_SIZE * TABLE_PO2_SIZE
        - (1 + TABLE_PO2_SIZE + TABLE_PO2_SIZE²);
```

### 9.3 旋转与翻转优化 (Line 152-173)

为减少 LUT 大小，强制 `TessFactors.x >= TessFactors.y >= TessFactors.z`：

```hlsl
// 旋转: 最大 factor 放到 x 位置
if (TessFactors.y is max) → rotate yzx
if (TessFactors.z is max) → rotate zxy

// 翻转: 确保 y >= z
if (TessFactors.y < TessFactors.z) → swap(y, z), bFlipWinding = true
```

翻转标志编码在 Pattern 的 bit 12: `Pattern |= 0x1000`

### 9.4 两种表

- **Immediate Table**: 用于 `SW Rasterize (Tessellated)` 中的即时 dice
  - 最大 TessFactor: `NANITE_TESSELLATION_TABLE_IMMEDIATE_SIZE`
  - 存储偏移: `offset + TABLE_PO2_SIZE³`
- **Full Table**: 用于 `SW Rasterize (Patches)` 中的 patch dice
  - 最大 TessFactor: `NANITE_TESSELLATION_TABLE_SIZE`
  - 存储偏移: `offset + 0`

### 9.5 数据读取

```hlsl
// 读取顶点 barycentric (Line 199-204)
float3 GetVert(uint VertIndex)
{
    uint Encoded = TessellationTable_VertsAndIndexes.Load(TableOffset + VertIndex * 4);
    return DecodeBarycentrics(Encoded);
}

// 读取三角形索引 (Line 206-222)
uint3 GetIndexes(uint TriIndex)
{
    // 3 × 10-bit 索引打包在 uint 中
    uint Packed = TessellationTable_VertsAndIndexes.Load(TableOffset + VertOffset + TriIndex * 4);
    uint3 Indexes;
    Indexes.x = (Packed >>  0) & 0x3FF;
    Indexes.y = (Packed >> 10) & 0x3FF;
    Indexes.z = (Packed >> 20) & 0x3FF;

    // 如果有翻转, 交换 y/z 恢复正确 winding
    if (bFlipWinding) Swap(Indexes.y, Indexes.z);
    return Indexes;
}
```

---

## 10. 核心数据结构

### 10.1 FTessellatedPatch (NaniteTessellation.ush Line 135-266)

```hlsl
struct FTessellatedPatch
{
    uint TableOffset;                // Tessellation Table 中的偏移
    uint Pattern_NumVerts_NumTris;   // 打包: Pattern(13b) | NumVerts(9b) | NumTris(10b)

    void Init(float3 TessFactors, uint3 VertData, bool bImmediateTable);
    void Init(uint Pattern, bool bImmediateTable);

    uint GetPattern()   { return Pattern_NumVerts_NumTris & 0x1FFF; }
    uint GetNumVerts()  { return (Pattern_NumVerts_NumTris >> 13) & 0x1FF; }  // max 512
    uint GetNumTris()   { return Pattern_NumVerts_NumTris >> 22; }             // max 1024

    float3 GetVert(uint VertIndex);    // 返回 barycentric
    uint3 GetIndexes(uint TriIndex);   // 返回 3 个顶点索引
    float3 GetTessFactors();           // 从 pattern 反算 TessFactor
};
```

### 10.2 FSplitPatch (NaniteTessellation.ush Line 278-309)

```hlsl
struct FSplitPatch
{
    uint VisibleClusterIndex;   // 25 bits, 引用 VisibleCluster 表
    uint TriIndex;              // 7 bits, cluster 内三角形索引 (max 128)
    float3 Barycentrics[3];    // 3 个 barycentric (定义子三角形在父三角形中的位置)

    void Decode(uint4 Encoded)
    {
        VisibleClusterIndex = Encoded.x >> 7;
        TriIndex = Encoded.x & 0x7F;
        for (int i = 0; i < 3; i++)
            Barycentrics[i] = DecodeBarycentrics(Encoded[i + 1]);
    }

    // 将 local barycentric 变换到父三角形的 barycentric 空间
    float3 TransformBarycentrics(float3 Local)
    {
        return Barycentrics[0] * Local.x
             + Barycentrics[1] * Local.y
             + Barycentrics[2] * Local.z;
    }
};
```

### 10.3 Barycentric 编码 (NaniteTessellation.ush Line 15-43)

```hlsl
const uint BarycentricMax = 32768;  // 2^15, 15-bit 定点数

uint EncodeBarycentrics(float3 Barycentrics)
{
    Barycentrics = round(Barycentrics * BarycentricMax);
    // 修正舍入误差: 强制最大分量 = BarycentricMax - 其他两个
    // 打包: x[0:15] | y[16:31], z = 1 - x - y
    return uint(Barycentrics.x) | (uint(Barycentrics.y) << 16);
}

float3 DecodeBarycentrics(uint Encoded)
{
    float x = float(Encoded & 0xFFFF) / BarycentricMax;
    float y = float(Encoded >> 16)    / BarycentricMax;
    return float3(x, y, 1.0 - x - y);
}
```

### 10.4 SplitWorkQueue (NaniteTessellation.ush Line 329-337)

```hlsl
// 用于传递需要进一步 split 的 patch
// 每个条目 16 bytes (uint4):
//   .x = (VisibleClusterIndex << 7) | TriIndex
//   .y = EncodedBarycentrics[0]
//   .z = EncodedBarycentrics[1]
//   .w = EncodedBarycentrics[2]
```

### 10.5 VisiblePatches (NaniteTessellation.ush Line 352-357)

```hlsl
// PatchSplit 的输出, SW Rasterize (Patches) 的输入
// PATCH_REFS 模式: 每条目 8 bytes (uint2)
//   .x = SplitWorkQueue 中的 offset
//   .y = TessellatedPatch Pattern
// 非 PATCH_REFS 模式: 每条目 16 bytes (uint4) = encoded patch data
```

---

## 11. 性能分析与优化建议

### 11.1 为什么 SW Rasterize (Tessellated) 比 SW Rasterize (Patches) 慢

| 开销项目 | Tessellated (1266) | Patches (460) |
|---------|-------------------|---------------|
| 处理的 cluster/patch 数量 | **所有** tessellated cluster | 仅 PatchSplit 后的 visible patches |
| 逐三角形顶点 Fetch + Transform | ✅ 全部 | ✅ 仅 patch 角点 (3 个) |
| 材质 Shader 求值 | ✅ 全部 | ✅ 仅 patch |
| TessFactor 计算 | ✅ 全部 | ✅ 仅 patch |
| 即时 Dice + Rasterize | ✅ 远景 | ❌ |
| 写入 SplitWorkQueue | ✅ 近景 | ❌ |
| Dice + Displacement + Rasterize | ❌ | ✅ 全部 |

**核心原因**: `SW Rasterize (Tessellated)` 是**所有 tessellation cluster 的网关**，即使只有远景被光栅化，它仍然要评估所有 cluster 的所有三角形。

### 11.2 优化方向

#### 方向 1：启用 Displacement Fade Range

在 Landscape 材质上设置 `DisplacementFadeRange`:
- `StartSizePixels`: 开始衰减的像素大小
- `EndSizePixels`: 完全关闭的像素大小

使远处 cluster 的 displacement 投影低于阈值后 **fallback 到 HW Rasterize**，彻底跳出 tessellation 管线。

#### 方向 2：调整 r.Nanite.MaxPixelsPerEdge

```
r.Nanite.MaxPixelsPerEdge = N
```
增大此值会降低全局 DiceRate → 降低 TessFactor → 减少需要 split 的三角形数量。

#### 方向 3：Per-Cluster Displacement Fallback

确保 Landscape primitive 上启用了 `PRIMITIVE_SCENE_DATA_FLAG_PER_CLUSTER_DISPLACEMENT_FALLBACK_RASTER`，让 GPU 逐 cluster 判断是否需要 tessellation。

#### 方向 4：减少 MaxDisplacement

材质的 `AbsMaxMaterialDisplacement` 直接影响 `LowTessDistance`。如果实际 displacement 幅度远小于声明值，缩小此值可以让 LowTess 更早生效。

---

## 12. RenderDoc 中的对应关系

| RenderDoc Event | 对应 GPU 函数 | 输入 | 输出 | 说明 |
|----------------|-------------|------|------|------|
| **SW Rasterize (Tessellated)** | `ClusterRasterize()` Patches=false | 所有 tessellated cluster | VisBuffer (远景) + SplitWorkQueue (近景) | 即时 dice 远景; enqueue 近景 |
| **PatchSplit** | `PatchSplit()` | SplitWorkQueue | VisiblePatches | 递归细分到 table size 以内 |
| **SW Rasterize (Patches)** | `PatchRasterize()` Patches=true | VisiblePatches | VisBuffer (近景) | 对细分后的 patch dice + 光栅化 |
| **HW Rasterize (FixedFunction)** | 传统 VS/PS 管线 | 非 displacement cluster 或 fallback cluster | VisBuffer | 无 tessellation |

### RenderDoc 中你看到的现象解释

**第一张图 (SW Rasterize Tessellated — 只有远景)**：
- 只有 TessFactor ≤ IMMEDIATE_SIZE 的三角形被即时 dice 并光栅化
- 这些都是远处的三角形（TessFactor 经过 LowTess × 0.25 衰减后变得很小）
- 近处的三角形虽然被评估了，但只写入了 SplitWorkQueue，不产生像素输出
- **GPU time = 1266 包含了对所有 tessellated cluster 的评估开销**

**第二张图 (HW Rasterize FixedFunction — 大量地形)**：
- 包含已经 fallback 的 tessellation cluster（displacement 投影 < 阈值）
- 以及所有非 displacement 材质的 Nanite mesh
- 注意：如果 `DisplacementFadeRange` 未启用，远处 cluster 不会 fallback，全部留在 tessellation 管线中

---

## 附录：关键源码文件索引

| 文件 | 路径 | 职责 |
|------|------|------|
| NaniteCullRaster.cpp | `Engine/Source/Runtime/Renderer/Private/Nanite/` | CPU 侧 dispatch 构建、资源绑定、pass 调度 |
| NaniteRasterizer.usf | `Engine/Shaders/Private/Nanite/` | ClusterRasterize + PatchRasterize shader |
| NaniteTessellation.ush | `Engine/Shaders/Private/Nanite/` | TessFactor 计算、FTessellatedPatch、FSplitPatch |
| NaniteDice.ush | `Engine/Shaders/Private/Nanite/` | FDiceTask、FClusterSplitTask、DistributeWork |
| NaniteSplit.usf | `Engine/Shaders/Private/Nanite/` | PatchSplit 递归 compute shader |
| NaniteRasterBinning.usf | `Engine/Shaders/Private/Nanite/` | Raster bin 分类、InitVisiblePatchesArgs |
| NaniteClusterCulling.usf | `Engine/Shaders/Private/Nanite/` | Cluster culling、displacement fallback 标志 |
