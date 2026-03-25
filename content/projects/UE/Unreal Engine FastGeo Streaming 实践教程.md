
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

### Step 4：创建 World Partition 关卡

#### 4a. 新建 Open World 关卡

1. 在编辑器菜单栏点击 **File → New Level**
2. 在弹出的模板窗口中选择 **Open World**（不是 Basic 或 Empty Level）
3. 点击 Create，引擎会生成一个默认带 Landscape 的关卡，World Partition 已自动启用
4. 按 **Ctrl+S** 保存关卡，给它起个名字，比如 `L_FastGeoTest`

> **如何确认 WP 已启用：** 点击 **Window → World Partition** 打开 World Partition 编辑器面板。如果能看到一个蓝色网格的俯视图，说明 WP 已经启用了。

#### 4b. 如果你用的是已有项目

如果你想在已有的非 WP 关卡上测试：

1. 打开关卡
2. 进入 **Tools → Convert Level...**
3. 选择 **Convert to World Partition**
4. 引擎会创建一个转换后的副本

**注意：** 转换是单向的，建议先备份原始关卡。

### Step 5：准备静态网格体资产

你需要一些静态网格体来填充场景。有几种获取方式：

#### 方式 A：使用 Starter Content（最快）

1. 如果项目创建时没有勾选 Starter Content，可以现在添加：
    - **Content Browser** 空白处右键 → **Add Feature or Content Pack**
    - 选择 **Content → Starter Content** → Add to Project
2. 添加后在 `Content/StarterContent/Props/` 路径下能找到桌椅、柱子、门等道具
3. 在 `Content/StarterContent/Architecture/` 下有墙壁、地板等建筑件

#### 方式 B：从 Fab（Epic 的资产商城）下载

1. 在编辑器中打开 **Fab**（Edit → Plugins 确保 Fab 插件已启用）
2. 搜索免费资产包，推荐关键词：`modular building`、`medieval props`、`environment pack`
3. 下载后资产会出现在 Content Browser 中

#### 方式 C：使用引擎自带基本形状（最简方案）

即使不下载任何额外资产，你也可以用引擎自带的几何体测试：

1. 在 **Place Actors** 面板（左侧边栏或 Window → Place Actors）中
2. 选择 **Shapes** 分类
3. 将 Cube、Cylinder、Sphere 等拖入场景

### Step 6：在场景中大量放置静态网格体

这是最关键的一步——你需要让场景足够"密集"才能看到 FastGeo 的效果。

#### 6a. 手动放置（少量测试）

1. 从 Content Browser 中将静态网格体资产 **拖拽** 到视口中
2. 放置后在 Details 面板中确认以下属性：
    - **Mobility** 必须设为 **Static**（这是默认值，一般不需要改）
    - 确保该 Actor 不是蓝图 Actor（不含额外逻辑）
3. 重复放置，在场景中散布至少 50-100 个

#### 6b. 批量复制（推荐，效率更高）

1. 先放置一个 StaticMeshActor
2. 选中它，按住 **Alt 键 + 拖拽** 可以快速复制
3. 或者选中一个 Actor 后按 **Ctrl+D** 复制，然后移动到新位置
4. 选中多个 Actor（框选或 Ctrl+点击），再 Alt+拖拽可以一次复制一批

#### 6c. 使用散布工具（最高效，几百上千个）

如果你需要大量放置（几百个以上），可以使用 Foliage Tool：

1. 点击工具栏上的 **模式选择器**（Modes），切换到 **Foliage**（树叶图标）
2. 将你想散布的 StaticMesh 资产从 Content Browser 拖到 Foliage Type 列表中
3. 设置笔刷大小和密度
4. 在视口中"刷"地面，就会自动散布大量实例

> **注意：** Foliage Tool 生成的是 Instanced Static Mesh (ISM) 组件，与手动放置的 StaticMeshActor 在 FastGeo 处理上可能有所不同。两种都可以测试。

#### 6d. 使用 PCG（程序化生成，进阶方案）

如果你想同时测试 PCG + FastGeo 的联动：

1. 在 Content Browser 右键 → **Miscellaneous → PCG Graph**
2. 双击打开 PCG Graph 编辑器
3. 拖入 **Surface Sampler** 节点（在 Landscape 表面采样点位）
4. 连接 **Static Mesh Spawner** 节点，指定你想生成的网格体
5. 在场景中放置一个 **PCG Volume**，将 Graph 赋给它
6. 在 PCG Volume 的 Details 面板中点击 **Generate** 即可生成

### Step 7：确保 Actor 分布跨越多个流送单元格

FastGeo 的优化是以流送单元格（Streaming Cell）为单位的，所以你的 Actor 必须分散在足够大的区域内。

1. 打开 **Window → World Partition** 面板查看网格
2. 默认网格大小通常是 12800（约 128 米）或 25600（约 256 米）
3. 你放置的 Actor 应该覆盖 **至少 4-6 个不同的网格单元格**
4. 简单做法：在 X 方向上放一排，间隔超过一个 Cell Size，在 Z 方向上也放一排，形成一个 L 形或方形分布

> **调整网格大小：** 在 **World Settings → World Partition → Runtime Hash → Grids** 中可以查看和修改 Cell Size。测试时可以把 Cell Size 调小一些（比如 6400），这样不需要太大的场景就能看到多个 Cell 的效果。

### Step 8：检查你的场景是否符合 FastGeo 要求

在进入下一步配置 Transformer 之前，做一个 checklist：

- ✅ 关卡已启用 World Partition
- ✅ 场景中有大量 StaticMeshActor（或包含 StaticMeshComponent 的 Actor）
- ✅ 这些 Actor 的 Mobility 都是 **Static**
- ✅ 这些 Actor **不包含**蓝图逻辑、Tick、网络复制等
- ✅ Actor 分布 **跨越了多个** World Partition 网格单元格
- ✅ DefaultEngine.ini 中已配置好 `p.Chaos.EnableAsyncInitBody=true`

全部满足后，就可以进入第四部分配置 FastGeo Transformer 了。

### Step 8（补充）：关于 World Partition 网格的可视化技巧

在视口中你可以直接看到网格线来辅助判断 Actor 分布：

1. 在视口左上角点击 **Show**（眼睛图标）
2. 找到 **Developer → World Partition** 选项并勾选
3. 视口中会叠加显示 World Partition 的网格线
4. 不同颜色的网格线对应不同的 Runtime Grid

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