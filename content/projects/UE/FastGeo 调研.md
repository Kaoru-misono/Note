# Unreal Engine FastGeo Streaming 实践教程

> **适用版本：** UE 5.6（实验性）/ UE 5.7（实验性，功能更完善）  
> **前置条件：** 你的项目已启用 **World Partition**

---

## 第一部分：理解 FastGeo 的核心机制

在动手之前，先理解一个关键概念：

FastGeo 的工作不是发生在"运行时"，而是发生在 **Cook / 进入 PIE** 时。它通过 World Partition 的 **Runtime Cell Transformer**（运行时单元格转换器）机制，在流送生成阶段将符合条件的 Actor **替换**为一种不需要 Actor/Component 的轻量图元。

也就是说：

- 你在编辑器里照常摆放 StaticMeshActor
- FastGeo 的 Transformer 在 Cook/PIE 时自动识别并转换它们
- 运行时加载的不再是完整的 Actor，而是极轻量的 FastGeo Primitive

---

## 第二部分：环境准备

### Step 1：创建 World Partition 项目

如果你还没有 World Partition 项目：

1. **File → New Level → Open World**（默认已启用 World Partition）
2. 或者在现有项目中，**World Settings → Enable World Partition** 打勾

FastGeo 只能在 World Partition 环境下工作，因为它依赖 WP 的流送单元格系统。

### Step 2：启用 FastGeo Streaming 插件

1. **Edit → Plugins**
2. 搜索 **"Fast Geometry Streaming"** 或 **"FastGeo"**
3. 勾选启用（位于 `Engine/Plugins/Experimental/FastGeoStreaming`）
4. **重启编辑器**

### Step 3：配置必要的 CVar

FastGeo 要求启用异步物理体初始化，否则 Cook 会报错。

打开项目的 `Config/DefaultEngine.ini`，添加以下内容：

```ini
[SystemSettings]
p.Chaos.EnableAsyncInitBody=true
LevelStreaming.AllowIncrementalPreRegisterComponents=true
```

> **重要：** `p.Chaos.EnableAsyncInitBody` 是只读 CVar，不能通过控制台命令在运行时修改，必须在 INI 文件中设置。你可以在控制台输入 `p.Chaos.EnableAsyncInitBody`（不带值）来确认当前值和来源。

---

## 第三部分：搭建测试场景

### Step 4：创建一个简单的开放世界测试关卡

在你的 World Partition 关卡中摆放大量静态网格体来模拟密集场景：

1. 从 Starter Content 或 Fab 下载一些资产（建筑、石头、树桩等）
2. 在场景中散布 **大量 StaticMeshActor**（至少几百个）
3. 分散在较大的区域内，确保它们分布在不同的 World Partition 网格单元中

> **关键点：** 只有"不可变的静态几何体"才是 FastGeo 的目标——即没有 Gameplay 逻辑、不需要交互、不需要 Tick 的纯装饰性物体。

### Step 5：确认 World Partition 网格设置

1. **World Settings → World Partition → Runtime Hash**
2. 确保你有合理的网格大小（Grid Cell Size），比如默认的 12800 或 25600
3. FastGeo 将以这些单元格为单位进行转换

---

## 第四部分：配置 FastGeo Transformer

### Step 6：在 World Settings 中添加 FastGeo Transformer

FastGeo 通过 **Runtime Cell Transformer** 工作。Transformer 是在 World Settings 中配置的类，它在流送单元格生成时被调用，对单元格中的 Actor 进行转换。

1. 打开 **World Settings**
2. 找到 **World Partition → Runtime Cell Transformers**（或类似的数组属性）
3. 添加 FastGeo 提供的 Transformer 类

> 具体的类名可能因引擎版本有所不同，在 5.6/5.7 中你可以在 Class Picker 里搜索包含 "FastGeo" 的 Transformer 类。

### Step 7：标记哪些内容参与 FastGeo 转换

FastGeo Transformer 有判定逻辑来决定哪些 Actor 可以被转换。通常满足以下条件的 Actor 会被自动识别：

- 是 StaticMeshActor 或含有 StaticMeshComponent 的 Actor
- 没有绑定 Gameplay 逻辑（无蓝图逻辑、无 Tick）
- Mobility 设为 Static
- 不处于 Persistent Level（持久关卡中的 Actor 不参与流送）

如果你需要手动排除某些 Actor，可以通过 Data Layer 或标签的方式进行控制。

---

## 第五部分：测试与验证

### Step 8：在 PIE 中验证 FastGeo

进入 PIE（Play in Editor）时，FastGeo Transformer 会自动运行。

使用以下控制台命令来验证：

```
show ActorColoration FastGeo
```

这会将场景中的内容按颜色区分：

- **蓝色** = 已被 FastGeo 处理的内容
- **红色** = 仍然以传统 Actor 形式存在的内容

如果你看到大量蓝色区域，说明 FastGeo 已经在工作了。

### Step 9：其他调试命令

```
FastGeo.Show
```

隐藏/显示所有 FastGeo 图元，方便你判断哪些内容属于 FastGeo。

```
FastGeo.EnableTransformerDebugMode
```

启用后，在 PIE 中选中一个 Actor 或在 Cook 时，会输出该 Actor 的 FastGeo 转换详情日志，包括是否被转换、转换原因等。

### Step 10：性能对比

要真正看到效果，你需要做前后对比：

1. **禁用 FastGeo 插件** → 进入 PIE → 用 `stat unit` 或 Unreal Insights 观察 GameThread 在场景流送时的耗时
2. **启用 FastGeo 插件** → 同样操作 → 对比 GameThread 耗时

重点关注以下指标：

- `GameThread` 帧时间
- `AddToWorld` / `RemoveFromWorld` 的耗时
- 快速移动时是否出现帧率尖刺（hitch）

---

## 第六部分：进阶 — 结合 PCG 使用

如果你希望用 PCG 程序化生成大量散布物（草地、碎石等）并享受 FastGeo 的流送优化，需要额外配置。

### Step 11：启用 PCG FastGeo Interop 插件

1. **Edit → Plugins**
2. 搜索 **"PCG FastGeo Interop"**
3. 勾选启用
4. 重启编辑器

### Step 12：设置 Componentless Primitives CVar

在 `Config/DefaultEngine.ini` 中添加：

```ini
[ConsoleVariables]
pcg.RuntimeGeneration.ISM.ComponentlessPrimitives=1
```

或者在控制台中输入：

```
pcg.RuntimeGeneration.ISM.ComponentlessPrimitives 1
```

这个 CVar 告诉 PCG 在运行时生成 ISM（Instanced Static Mesh）时，不再创建传统的 ISM Component 和 Partition Actor，而是直接使用 FastGeo 的无组件图元。

### Step 13：设置 PCG 运行时生成

1. 创建一个 PCG Graph（在 Content Browser 右键 → Procedural Content Generation → PCG Graph）
2. 在 Graph 中设置你的散布逻辑（Surface Sampler → Static Mesh Spawner 等）
3. 在场景中放置 PCG Volume 或 PCG Component
4. 在 PCG Component 上启用：
    - **Is Partitioned** = true
    - **Generate at Runtime** = true
    - **Use Hierarchical Generation** = true
5. 确保场景中有 **PCG World Actor** 且启用了运行时生成

### Step 14：验证 PCG + FastGeo 的效果

进入 PIE 后：

- PCG 内容应在玩家周围动态生成
- 使用 `show ActorColoration FastGeo` 验证散布物显示为蓝色
- 快速移动时观察是否有流送卡顿

---

## 第七部分：Cook 与打包

### Step 15：Cook 构建

FastGeo 的转换在 Cook 阶段也会执行（而且这是它最重要的使用场景）。

执行正常的 Cook/Package 流程即可。如果配置正确，Cook 日志中会包含 FastGeo 相关的信息。

如果 Cook 报错提到 `AsyncInitBody not enabled`，请回到 Step 3 确认 INI 配置正确。

---

## 常见问题排查

|问题|解决方案|
|---|---|
|Cook 报错 "AsyncInitBody not enabled"|在 `DefaultEngine.ini` 的 `[SystemSettings]` 下设置 `p.Chaos.EnableAsyncInitBody=true`|
|`show ActorColoration FastGeo` 全是红色|检查：1) 插件是否启用 2) Transformer 是否配置 3) Actor 是否为静态 Mobility|
|PIE 中看不到 FastGeo 效果|确保关卡使用了 World Partition，且 Actor 分布在流送网格中|
|PCG 生成的内容没走 FastGeo|确认 PCG FastGeo Interop 插件已启用，且 CVar `pcg.RuntimeGeneration.ISM.ComponentlessPrimitives` = 1|
|运行时某些物体消失|FastGeo 只处理静态不可变几何体，带蓝图/动画的 Actor 可能不兼容|

---

## 推荐的参考资源

- **Unreal Fest 演讲：** "Streaming Improvements for Dense Worlds in The Witcher 4 UE5 Tech Demo"  
    https://dev.epicgames.com/community/learning/talks-and-demos/KWGD/
    
- **Tom Looman 性能分析：**  
    https://tomlooman.com/unreal-engine-5-6-performance-highlights/  
    https://tomlooman.com/unreal-engine-5-7-performance-highlights/
    
- **官方 API 文档：**  
    https://dev.epicgames.com/documentation/en-us/unreal-engine/API/PluginIndex/FastGeoStreaming
    
- **World Partition Internals 深度分析（含 Transformer 原理）：**  
    https://xbloom.io/2025/10/24/unreals-world-partition-internals/
    

---

## 总结

FastGeo 的使用流程可以用一句话概括：

> **你正常做内容，FastGeo 在 Cook/PIE 时自动帮你优化流送。**

它的强大之处在于对工作流的侵入极小——你不需要改变内容制作方式，只需要启用插件、配好几个 CVar、确保场景使用 World Partition，FastGeo 就会自动识别并转换适合优化的静态几何体。

但别忘了：这仍然是 **实验性功能**，建议在正式项目中充分测试后再大规模采用。