---
title: "Streaming Improvements for Dense Worlds - 字幕翻译"
date: 2026-03-23
tags: [UE5, streaming, translation, unreal-fest]
category: projects
description: "Unreal Fest Orlando 2025 演讲逐段字幕翻译，标注关键技术术语"
status: in-progress
source: "https://www.youtube.com/watch?v=BdopUm1_1_E"
---

# Streaming Improvements for Dense Worlds — 字幕翻译

> 格式说明：每段先给出英文原文，再给出中文翻译。**加粗**标注关键技术术语。

---

## Opening — 开场

> Hi everyone, thanks for joining the streaming improvements for dense world presentation. I'm Richard Malo, **principal engine programmer** at **Epic Games**. And I'm Jarosław Fruczki. I am **Core Tech engineer** at **CD Projekt Red**. Before we continue, I would like to give a special thanks to Sebastian Lucier and Ben Ziggler from Epic Games who also contributed to this initiative.

大家好，感谢参加"密集世界的流送改进"演讲。我是 Richard Malo，Epic Games 首席引擎程序员。我是 Jarosław Fruczki，CD Projekt Red 核心技术工程师。在继续之前，我想特别感谢同样为此做出贡献的 Epic Games 的 Sebastian Lucier 和 Ben Ziggler。

---

## Agenda — 议程

> We have quite a big agenda today. We will start with a short **streaming overview** and then go through all the changes starting with **mutators**, **cell transformers**, new **hash sets** and new features that allowed us to get much better results like **async physics state creation**, the **unified streaming budget**, to finish with **Fast Geo** and simplified **mesh and texture streaming** that are new development and experimental.

今天议程很满。我们将从简短的**流送概览**开始，然后逐一介绍所有改进，包括 **mutators（变换器）**、**cell transformers（单元格转换器）**、新的 **hash sets（哈希集合）**，以及带来更好效果的新特性如**异步物理状态创建**、**统一流送预算**，最后以 **Fast Geo** 和简化的**网格与纹理流送**收尾——后两者是新开发的实验性功能。

---

## Background — 合作背景

> We work together because there is no better way to build new things and push boundaries than working on something that tries to do things that we weren't able to achieve before. At CDPR, we partner up with our friends at Epic with an ambitious goal of building the **tech demo** that will showcase the **Unreal Engine** running **open world experiences** at **60fps** on **current gen consoles**.

我们合作是因为，没有比一起尝试做到以前做不到的事情更好的方式来突破边界了。在 CDPR，我们与 Epic 的朋友合作，目标是打造一个**技术 Demo**，展示 **Unreal Engine** 在**次世代主机**上以 **60fps** 运行**开放世界体验**的能力。

---

## 1. Streaming Generation Overview — 流送生成概览

> Before diving into the streaming improvements, let's begin with a quick overview of how the **world partition streaming generation process** works. The process is designed to break down large worlds containing vast numbers of actors and sub levels into efficiently **streamable cells**. The process runs both during **map cooking** and whenever **PIE** is launched. So it must be efficient.

在深入流送改进之前，先快速了解 **World Partition 流送生成过程**的工作原理。该过程将包含大量 Actor 和子关卡的大型世界拆分为高效的**可流送单元格（streamable cells）**。此过程在 **map cooking（地图烘焙）** 和每次启动 **PIE（Play In Editor）** 时都会运行，因此必须高效。

> Since world partition uses **levels** as containers for actors and relies on **level streaming** to load and unload them, the process needs to generate levels. It takes as input the world, the **world partition setup** including the **grid setup** and a subset of the actor's information such as the **actor streaming bounds**, **runtime grid**, **spatially loaded flag** and **runtime data layers**. This information is accessible to the **asset registry** and is updated each time an actor is saved. So we call this information the **world partition actor descriptor**.

由于 World Partition 使用 **levels（关卡）** 作为 Actor 的容器，并依赖 **level streaming（关卡流送）** 来加载和卸载它们，因此需要生成 levels。输入包括：世界本身、**World Partition 设置**（含 **grid 设置**），以及 Actor 信息的子集，如 **actor streaming bounds（流送边界）**、**runtime grid（运行时网格）**、**spatially loaded flag（空间加载标志）** 和 **runtime data layers（运行时数据层）**。这些信息可通过 **asset registry（资产注册表）** 访问，并在每次 Actor 保存时更新。我们称之为 **World Partition Actor Descriptor（世界分区 Actor 描述符）**。

> When launching PIE or cooking a partition map, the streaming generation process starts by building a list of all the world partition actor descriptors. It then performs a **validation pass** to detect, report and correct errors, ensuring the content is valid for the **partitioning phase**. Then the **runtime hash** handles partitioning using the world partition setup and the provided actor descriptors. It will create the necessary **world partition cells** and distribute the actor descriptors in the generated cells. A level will be created for each cell.

启动 PIE 或烘焙分区地图时，流送生成过程首先构建所有 World Partition Actor Descriptor 的列表。然后执行**验证 pass**，检测、报告和纠正错误，确保内容对**分区阶段**有效。接着 **runtime hash（运行时哈希）** 使用 WP 设置和提供的 Actor Descriptor 进行分区，创建必要的 **World Partition cells** 并分配 Actor Descriptor。每个 cell 会创建一个 level。

---

## What We Want from Streaming — 我们对流送的期望

> The streaming has to be **flexible** because we all have different use cases. It has to be **performant** so we can ensure that the players can move around freely without any distractions. The execution has to run within the **budget** and the budget we assign for the streaming has to be respected in all cases. If not, players will encounter **traversal stutter** and/or we have to plan for **loading screens**.

流送必须**灵活**，因为不同项目有不同用例。必须**高性能**，确保玩家自由移动不受干扰。执行必须在**预算**内运行，分配给流送的预算必须在所有情况下被严格遵守。否则，玩家会遭遇**遍历卡顿（traversal stutter）**，或者我们不得不设计**加载画面**。

> We have to ensure that we have tools to build with higher **granularity** and allow iterating without breaking what we built before. We would like to remove as much of the things from the **critical path**. Things like streaming don't contribute to that final frame image. And we hope for a renewed look at simple **primitives** so we can have **spawning** and adding objects to the scene cheap again.

我们需要确保拥有更高**粒度**的构建工具，并允许迭代而不破坏已有成果。我们希望尽可能多地将东西从**关键路径（critical path）** 移除——流送本身不参与最终帧画面的生成。我们希望重新审视简单 **primitives（图元）**，让**生成（spawning）** 和添加对象到场景再次变得廉价。

---

## 2. Actor Descriptor Mutators (UE 5.4+)

> Starting with the 5.4 release, we introduced **actor descriptor mutators** in the **pre-partitioning phase**. They enable modification of actor descriptors without having to **check out**, modify or submit content. This process is **non-destructive** and **code-driven**, which makes it much more reliable and robust than manual editing. The **mutable properties** are the **spatially loaded flag** and the **runtime grid**.

从 5.4 版本开始，我们在**预分区阶段**引入了 **Actor Descriptor Mutators**。它们允许修改 Actor Descriptor，无需 **check out（签出）**、修改或提交内容。这个过程是**非破坏性的**且**代码驱动**的，比手动编辑更可靠和健壮。**可变属性**包括 **spatially loaded flag（空间加载标志）** 和 **runtime grid（运行时网格）**。

> A common use case for actor descriptor mutators is automatically assigning actors to a runtime grid based on criteria such as **actor location**, **bounds**, **name**, **actor tags**. In fact, any combination of actor descriptor properties can be used here.

Actor Descriptor Mutators 的常见用例是根据 **Actor 位置**、**边界（bounds）**、**名称**、**Actor 标签**等条件自动将 Actor 分配到运行时网格。实际上可以使用 Actor Descriptor 属性的任意组合。

> At CD Projekt Red we heavily utilize the **folder structure** to separate **architecture** from **decoration**, **vegetation** and so on. With vanilla approach, we would have to do a manual pass to assign proper grids. With mutators, we can automate this process. We can very easily split existing structure and separate decoration into even more detailed grids depending on the **size** or the **draw distance**.

在 CD Projekt Red，我们大量利用**文件夹结构**将**建筑**与**装饰**、**植被**等分离。使用原始方法，我们需要手动分配网格。有了 Mutators，我们可以自动化这个过程。可以非常容易地拆分现有结构，根据**尺寸**或**绘制距离（draw distance）** 将装饰细分到更精细的网格。

> Thanks to mutators, we can easily split our world into very detailed grid setup. We can load **fewer primitives** at one time. It significantly impacts the **memory consumption** but also the performance. With mutators, we increased **streaming granularity**. We split the big chunk into smaller sections that we stream in different frames, and also in different physical spaces.

借助 Mutators，我们可以轻松地将世界拆分为非常精细的网格设置。一次加载**更少的 primitives**。这显著影响了**内存消耗**和性能。Mutators 增加了**流送粒度**。我们将大块拆分为更小的部分，在不同帧和不同物理空间中流送。

---

## 3. New Runtime Hash Set (UE 5.4+) — 新运行时哈希集合

> In the **partitioning phase**, starting with 5.4, we introduced a new **runtime hash set** to replace the original **runtime spatial hash** from Unreal 5.0. This new hash supports **2D** and full **3D streaming** and contains a list of **partition objects**. Actors are mapped to their corresponding partition object based on their **runtime grid property**.

在**分区阶段**，从 5.4 开始，我们引入了新的 **runtime hash set**，替代 Unreal 5.0 的原始 **runtime spatial hash（运行时空间哈希）**。新哈希支持 **2D** 和完整的 **3D 流送**，包含一组 **partition objects（分区对象）**。Actor 根据其 **runtime grid 属性**映射到对应的 partition object。

> We provide two **partition objects**: the **loose hierarchical grid** which is the default when creating a new map, and the **level streaming partition** which can mimic standard **level streaming** or even **world composition** setup with automatic support for **distance-based streaming**, **HLOD** and **runtime data layer** support.

我们提供两种 **partition objects**：**loose hierarchical grid（松散层级网格）**——创建新地图时的默认选项；以及 **level streaming partition（关卡流送分区）**——可模拟标准的 **level streaming** 甚至 **World Composition** 设置，自动支持**基于距离的流送**、**HLOD** 和**运行时数据层**。

> This new hash effectively **decouples** the partitioning from the common logic like handling runtime data layers and makes it easy to write new custom partition objects.

新哈希有效地将分区逻辑与通用逻辑（如处理 runtime data layers）**解耦**，并使编写新的自定义 partition object 变得容易。

---

## 4. Runtime Cell Transformers (UE 5.5+) — 运行时单元格转换器

> In the **post-partitioning phase**, starting with 5.5, **runtime cell transformers** were added to allow transforming the generated world partition cells. This is a **non-destructive** process and it's **data-driven**. It can be configured using a **runtime cell transformer stack**. A world partition cell is transformed by each transformer as it progresses through the stack.

在**后分区阶段**，从 5.5 开始，添加了 **runtime cell transformers**，允许变换已生成的 World Partition cells。这是一个**非破坏性**的、**数据驱动**的过程。可以通过 **runtime cell transformer stack（转换器堆栈）** 配置。World Partition cell 依次经过堆栈中的每个 transformer 进行变换。

> You can modify the world partition cell level's content like adding, removing, modifying any actor or component, or create **embedded assets** inside that level. But you cannot modify the **persistent level** content or generate new world partition cells, and you cannot create **public assets**.

你可以修改 World Partition cell level 的内容，如添加、删除、修改任何 Actor 或 Component，或在该 level 内创建**嵌入式资产（embedded assets）**。但不能修改 **persistent level（持久关卡）** 的内容或生成新的 World Partition cell，也不能创建**公共资产（public assets）**。

> We provide the **ISM runtime cell transformer** which automatically transforms **static mesh actors** into **instanced static mesh components**. You can specify a minimum number of **mesh instances** required to allow the transformation and define a list of allowed and disallowed actor classes. This transformer doesn't support **replicated actors**. The **root component mobility** needs to be static and **child actor components** are not supported.

我们提供了 **ISM runtime cell transformer**，自动将 **Static Mesh Actors** 转换为 **Instanced Static Mesh Components（实例化静态网格组件）**。可以指定允许转换所需的最少**网格实例数**，并定义允许和禁止的 Actor 类列表。该 transformer 不支持 **replicated actors（复制 Actor）**。**Root Component Mobility（根组件可移动性）** 必须为 Static，**Child Actor Components** 不受支持。

> We encourage you to experiment with and implement your own runtime cell transformer based on your project-specific needs. But be careful — this is a powerful tool but it comes with risks and no validation is performed at this stage. However, it only affects **runtime behavior**. **Editor content remains unaffected**. Your source assets and level data are safe.

我们鼓励你根据项目特定需求实验和实现自己的 runtime cell transformer。但要小心——这是一个强大的工具，但有风险，此阶段不执行验证。不过，这只影响**运行时行为**。**编辑器内容不受影响**。你的源资产和关卡数据是安全的。

> Once again, we see fewer primitives as we merge components together and reduction in memory. This time we get real performance savings due to fewer components. We have fewer **loads**, **post loads**, **spawns**, and **destroys**.

我们再次看到合并 Components 后 primitives 更少，内存减少。这次我们获得了真正的性能节省：更少的 **loads（加载）**、**post loads（后加载）**、**spawns（生成）** 和 **destroys（销毁）**。

---

## 5. Async Physics State Creation (UE 5.6+, Experimental) — 异步物理状态创建

> With **component reduction** due to conversion to ISMs, we minimize the cost of registering those components. What we find instead is that the majority of the cost comes from creating the **physics state** of the assets.

通过转换为 ISM **减少 Components** 后，我们最小化了注册这些 Components 的成本。但我们发现，大部分成本来自创建资产的**物理状态（physics state）**。

> This is why starting with 5.6, we implemented **asynchronous creation and destruction of the physics state**. In addition to changes made in **Chaos** to support this, the **level streaming**, more specifically **AddToWorld** and **RemoveFromWorld**, was updated to support **asynchronous task management**. We added **incremental pre-register and pre-unregister component phases** to handle the asynchronous creation and destruction of the physics state.

因此从 5.6 开始，我们实现了**物理状态的异步创建和销毁**。除了在 **Chaos（物理引擎）** 中做的修改外，**level streaming**（具体是 **AddToWorld** 和 **RemoveFromWorld**）也更新以支持**异步任务管理**。我们添加了**增量 pre-register 和 pre-unregister component 阶段**来处理物理状态的异步创建和销毁。

> The supported components are the **static mesh component**, the **ISM component**, and the **landscape height field collision component**.

支持的组件包括 **Static Mesh Component**、**ISM Component** 和 **Landscape Height Field Collision Component**。

> We added support for adding and removing **multiple levels concurrently**. Multiple AddToWorld and RemoveFromWorld calls were initiated within the same **game thread time budget**. None of them actually completed during that frame. Instead, they each kicked off **asynchronous tasks** and most of them will likely finish on the next frame.

我们增加了**并发添加和移除多个 levels** 的支持。多个 AddToWorld 和 RemoveFromWorld 调用在同一**游戏线程时间预算**内发起。它们都没有在那一帧内完成。相反，每个都启动了**异步任务**，大部分可能在下一帧完成。

> The main gain comes from moving heavy **physics creation and destruction** from the **critical path** to the **worker thread**. As long as you have **spare cycles** on your worker thread, you're going to see significant improvement in streaming.

主要收益来自将繁重的**物理创建和销毁**从**关键路径**移至 **worker thread（工作线程）**。只要你的 worker thread 有**空闲周期**，就能看到流送的显著改善。

---

## 6. Unified Streaming Budget — 统一流送预算

> Going back to our mission statement, we wanted to build a demo that runs at **60fps**. The most important concept is **budgeting**. In open world experiences, we need to ensure in every single frame that we keep a **budget for streaming**, a buffer. If there is no buffer when you need to stream something, you will encounter **traversal stutter**.

回到我们的目标，我们要构建一个 **60fps** 运行的 Demo。最重要的概念是**预算管理**。在开放世界体验中，我们需要确保每一帧都保留**流送预算**——一个缓冲区。如果需要流送时没有缓冲区，就会遇到**遍历卡顿**。

> We do not stream in every single frame. Making this budget big will waste cycles in frames where we just idle. But if the buffer is too small, we may stream too slow. With base implementation, we need to worry about three budgets: **AddToWorld**, **RemoveFromWorld** and **ProcessAsyncLoading**, as they all might execute in exactly the same single frame. We initially had to reserve **2.5 milliseconds** — 15% of every single frame.

我们不是每帧都在流送。预算太大会在空闲帧浪费周期。但缓冲区太小，流送可能太慢。基础实现需要管理三个预算：**AddToWorld**、**RemoveFromWorld** 和 **ProcessAsyncLoading**，它们可能在同一帧内执行。我们最初需要预留 **2.5 毫秒**——每帧的 15%。

> Let's see what happens when we load a small bit of the level. Everything starts with a very cheap request to **async loading** to load all required **serialized actor component information**. It takes five frames to load data from drive. But even as those assets are being loaded, we need to run **post loads** executed on the **game thread**. With a tight budget of 0.5 milliseconds, it takes **29 frames or around 500 milliseconds** to process all that data. After that, we start adding loaded actors and components to the world — with a 1 millisecond budget, it takes **24 frames, 400 milliseconds**.

看看加载一小块 level 时会发生什么。一切从向 **async loading（异步加载）** 发出请求开始，加载所有需要的**序列化 Actor/Component 信息**。从硬盘加载数据需 5 帧。但资产加载的同时需要在**游戏线程**上运行 **post loads（后加载）**。以 0.5 毫秒的紧凑预算，处理所有数据需要 **29 帧 / 约 500 毫秒**。之后，开始将加载的 Actor 和 Component 添加到世界——以 1 毫秒预算，需要 **24 帧 / 400 毫秒**。

> There is **51 frames of delay** between our request and the final update. The worst part is in most frames when we ProcessAsyncLoading we waste the budget for AddToWorld, and when adding to the world we waste the budget for ProcessAsyncLoading. We are utilizing just a fraction of the budgets we assigned.

从请求到最终更新有 **51 帧的延迟**。最糟糕的是，大多数帧中 ProcessAsyncLoading 时浪费了 AddToWorld 的预算，AddToWorld 时又浪费了 ProcessAsyncLoading 的预算。我们只利用了分配预算的一小部分。

> We implemented an optional **unified streaming budget**. We calculate the streaming budget as a sum of existing budgets but still execute ProcessAsyncLoading with a **guaranteed minimum budget**. Then with the unified budget left, we start **UpdateLevelStreaming** that will execute add and remove under a **single time budget**. If UpdateLevelStreaming didn't use all the budget, we try to run ProcessAsyncLoading again to prepare the next batch of assets for the next frame.

我们实现了可选的**统一流送预算**。将流送预算计算为现有预算之和，但仍以**保证的最低预算**执行 ProcessAsyncLoading。然后用剩余的统一预算启动 **UpdateLevelStreaming**，在**单一时间预算**下执行 add 和 remove。如果 UpdateLevelStreaming 没用完预算，我们会再次运行 ProcessAsyncLoading，为下一帧准备下一批资产。

> We could reduce the budget from **2.5 to 1.5 milliseconds** and still got a **40% reduction** in streaming response delay. Any **hitch** in ProcessAsyncLoading will be **amortized** in that specific frame by reduction in time in UpdateLevelStreaming.

我们将预算从 **2.5 降至 1.5 毫秒**，仍获得流送响应延迟 **40% 的减少**。ProcessAsyncLoading 中的任何**卡顿（hitch）** 都会被 UpdateLevelStreaming 的时间减少在同一帧内**摊销（amortized）**。

---

## 7. Fast Geo Streaming Plugin (UE 5.6+, Experimental)

### 动机 — Motivation

> Streaming **static geometry** such as **HLOD actors** or even the transformed **ISM actors** go through the same **registration process** as more complex actors like gameplay actors. However static geometry is much simpler. It doesn't need that kind of overhead. So what we should aim for is to remove the static geometry from the regular **actor component registration process** and stream it outside of the **critical path**.

流送**静态几何体**（如 **HLOD Actors** 或转换后的 **ISM Actors**）经历与复杂 gameplay Actor 相同的**注册过程**。然而静态几何体简单得多，不需要这种开销。我们的目标是将静态几何体从常规的 **Actor/Component 注册过程**中移除，在**关键路径之外**进行流送。

> This should have no impact on the **editor workflow** and should preserve full **world partition functionality**. This **fast path** should be easy to test in game and support quick iteration with minimal setup. This is what led to the development of the new experimental **Fast Geo streaming plugin**.

这不应影响**编辑器工作流**，且应保留完整的 **World Partition 功能**。这条**快速路径（fast path）** 应易于在游戏中测试，支持最小设置下的快速迭代。这就是新的实验性 **Fast Geo streaming 插件**的开发动因。

### 内容生成 — Content Generation

> To generate the Fast Geo content, this process should first extract the static geometry and convert it into a **lightweight non-UObject data structure**. That content should be stored in the same level as the **non-transformed content** as we need both to stream at the same time. The process should be **non-destructive** and shouldn't require an **offline process** like a builder.

生成 Fast Geo 内容时，首先提取静态几何体并转换为**轻量级非 UObject 数据结构**。该内容应与**未转换内容**存储在同一 level 中，因为两者需要同时流送。过程应是**非破坏性的**，不需要像 builder 那样的**离线过程**。

> This is a perfect use case for **runtime cell transformers**. Fast Geo comes with its own runtime cell transformer. Actors can be **partially or fully transformed**. Fully transformed actors are simply removed from the level. Note that this transformer doesn't automatically convert **static mesh actors** into **ISM components**. To do so, simply add the provided **ISM cell transformer** before the Fast Geo transformer in the **transformer stack**.

这是 **runtime cell transformers** 的完美用例。Fast Geo 自带 runtime cell transformer。Actor 可以**部分或完全转换**。完全转换的 Actor 直接从 level 中移除。注意该 transformer 不会自动将 **Static Mesh Actors** 转换为 **ISM Components**。要实现这一点，只需在 **transformer stack** 中将 **ISM cell transformer** 放在 Fast Geo transformer 之前。

### 运行时流送 — Runtime Streaming

> When **adding a level to the world**, Fast Geo streaming is notified, finds its Fast Geo content inside the level and starts **asynchronous tasks** to create the **physics and render state**. While these tasks are running, the non-transformed actors and components are registered using the regular registration process. A new **delegate** was added to allow external systems like Fast Geo to monitor their internal tasks and report whether they have completed. **AddToWorld** will combine that result with its own internal state to determine when it has completed.

当**将 level 添加到世界**时，Fast Geo streaming 收到通知，找到 level 内的 Fast Geo 内容并启动**异步任务**来创建**物理和渲染状态**。这些任务运行时，未转换的 Actor 和 Component 通过常规注册过程注册。添加了新的 **delegate（委托）**，允许 Fast Geo 等外部系统监控其内部任务并报告完成状态。**AddToWorld** 将该结果与自身内部状态合并，以确定何时完成。

> Several systems were adapted to support **decoupling from actors and components**. On the **rendering** side, we built on **render proxy decoupling**. On the **physics** side, we decoupled via async physics state creation and provided a **physics body instance owner interface** on the **hit result** and the **overlap result**. We also decoupled the **HLOD subsystem** from HLOD actors. And the **render asset streaming manager** was modified to work without **primitive components**.

多个系统进行了适配以支持**与 Actor 和 Component 解耦**。**渲染**方面，基于 **render proxy（渲染代理）解耦**。**物理**方面，通过异步物理状态创建解耦，并在 **hit result（命中结果）** 和 **overlap result（重叠结果）** 上提供了 **physics body instance owner interface（物理体实例所有者接口）**。还将 **HLOD 子系统**从 HLOD Actor 解耦。**渲染资产流送管理器**也被修改为无需 **primitive components** 即可工作。

### 限制 — Limitations

> Fast Geo currently has some limitations. **Replicated actors** are not supported. **Blueprint actors** with logic are excluded from transformation to avoid potential gameplay issues. Actors referenced by non-transformed or **partially transformed actors** are also excluded as this would break **actor references**. **Actor component mobility** needs to be static.

Fast Geo 目前有一些限制。不支持 **replicated actors（复制 Actor）**。有逻辑的 **Blueprint Actors** 被排除以避免潜在的 gameplay 问题。被未转换或**部分转换的 Actor** 引用的 Actor 也被排除，因为这会破坏 **actor references（Actor 引用）**。**Actor Component Mobility（可移动性）** 必须为 Static。

### 调试工具 — Debugging Tools

> We added a **Fast Geo actor coloration mode** which shows in red the **non-transformed primitives**. A console command `FastGeo.Show` allows to hide or show the Fast Geo content — it's an easy way to isolate the non-transformed content. We also added **debug modes** to report all the details of the transformation and the reason why some actors were not transformed. There are three debug modes: **selection-based**, **PIE cell loading**, and a **cook-time** console variable mode.

我们添加了 **Fast Geo Actor 着色模式**，用红色显示**未转换的 primitives**。控制台命令 `FastGeo.Show` 可隐藏/显示 Fast Geo 内容——轻松隔离未转换内容。还添加了**调试模式**，报告转换的所有细节和某些 Actor 未被转换的原因。三种调试模式：**基于选择的**、**PIE cell 加载时的**、以及 **cook 时的**控制台变量模式。

### 结果 — Results

> We've been able to convert **91% of all components** in our demo. **86% of actors** were fully transformed and 1% was transformed partially. The things that were not converted are **skeletal meshes**, gameplay elements, and things without **static mobility**.

在我们的 Demo 中，成功转换了 **91% 的所有 components**。**86% 的 actors** 被完全转换，1% 被部分转换。未转换的是 **skeletal meshes（骨骼网格）**、gameplay 元素和没有 **static mobility** 的对象。

> Fast Geo greatly impacts performance because we don't have to **load**, **post-load** and create all the components, run all the registration for the environment. In the final demo, the whole budget for the streaming was just **0.8 milliseconds**.

Fast Geo 极大地影响了性能，因为我们不必**加载**、**后加载**和创建所有环境的 Components，不必运行所有注册。最终 Demo 中，整个流送预算仅 **0.8 毫秒**。

### City Sample 测试 — City Sample Test

> We enabled Fast Geo in **City Sample** without modifying the content. **92% of components** were transformed. **94% of actors** fully transformed and 3% partially transformed.

我们在 **City Sample** 中启用了 Fast Geo，未修改内容。**92% 的 Components** 被转换。**94% 的 Actors** 完全转换，3% 部分转换。

> We ran an **automated test** on **PlayStation 5** with a **drone** flying through the map at around **215 kilometers per hour**. As expected, **AddToWorld** and **RemoveFromWorld** now take a very small amount of time on the **game thread**. Since a big part of the map was transformed into Fast Geo content, we end up with a lot less **UObjects** which translates to less objects being processed by both the **garbage collection** and ProcessAsyncLoading. Both the **peaks** and the **duration** of garbage collection were significantly reduced.

我们在 **PlayStation 5** 上运行了**自动化测试**，**无人机**以约 **215 km/h** 飞越地图。如预期，**AddToWorld** 和 **RemoveFromWorld** 在**游戏线程**上只占用很少时间。由于地图大部分被转换为 Fast Geo 内容，**UObjects** 大幅减少，**垃圾回收（garbage collection）** 和 ProcessAsyncLoading 处理的对象也随之减少。GC 的**峰值**和**持续时间**都显著降低。

> We then increased the drone speed to **540 kilometers per hour** with a tight budget of just **2 milliseconds**. Without Fast Geo, the game pauses while streaming catches up. With Fast Geo enabled, **streaming stays ahead of the game**, keeping everything smooth and uninterrupted.

然后我们将无人机速度提升到 **540 km/h**，预算仅 **2 毫秒**。没有 Fast Geo，游戏会暂停等待流送追赶。启用 Fast Geo 后，**流送始终领先于游戏**，一切流畅不中断。

---

## 8. Simple Streamable Asset Manager — 简化流送资源管理器

> **Texture and mesh streaming** is a legacy system that streams **mips** and **LODs** for static and skeletal meshes. Not to be confused with **virtual textures** and **Nanite** as a virtual geometry system.

**纹理和网格流送**是一个遗留系统，为静态和骨骼网格流送 **mips（多级渐远纹理层级）** 和 **LODs（细节级别）**。不要与 **virtual textures（虚拟纹理）** 和 **Nanite**（作为虚拟几何体系统）混淆。

> It is a very **CPU-intensive** process. Each time we load a level, the system has to extract all actors, then all components, then textures from materials — quite a lot of **nested loops**. For each asset it has to keep track of the **bounds** of all components using that specific asset to estimate **screen space projection** so it can load the proper **mip** with respect to the **texture pool size**. The costly registration is on the **critical path** on the game thread.

这是一个非常 **CPU 密集**的过程。每次加载 level，系统需要从所有 Actor 提取所有 Components，再从 Components 中提取材质的纹理——大量**嵌套循环**。对每个资产，需跟踪使用该资产的所有 Component 的 **bounds**，以估算**屏幕空间投影（screen space projection）**，从而根据 **texture pool size（纹理池大小）** 加载适当的 **mip**。昂贵的注册在游戏线程的**关键路径**上。

> We are using **multi-layered materials** and each material can have over **30 reference textures**. We have textures referenced by **thousands of components** in case of foliage or hundreds for architecture. Not every use case can be efficiently supported by **virtual textures**. But I recommend using them whenever possible.

我们使用**多层材质**，每个材质可有超过 **30 个引用纹理**。植被的纹理可能被**数千个 Components** 引用，建筑的则被数百个引用。并非所有用例都能被 **virtual textures** 高效支持，但我建议尽可能使用它们。

> We implemented a **texture sampling streaming cache** to break one of the most expensive loops. We added **parallel processing** but not the whole algorithm was susceptible to parallel execution. Even with additional CPU power, we had a hard time reducing spikes below **1 millisecond**.

我们实现了 **texture sampling streaming cache（纹理采样流送缓存）** 来打破最昂贵的循环之一。添加了**并行处理**，但不是所有算法都适合并行执行。即使增加 CPU 算力，也难以将尖峰降至 **1 毫秒**以下。

> We decided to implement the **Simple Streamable Asset Manager** — an additional module to the **render asset streaming manager**. It replaces the component-based registration process with a process relying on a **scene proxy** but it can be fed with any arbitrary data. The **cost of registration was removed from the critical path**. All processing is executed **asynchronously in the background**. Due to its simplified nature, the asset manager has **better memory access patterns** and can process data more quickly.

我们决定实现 **Simple Streamable Asset Manager** —— **渲染资产流送管理器**的附加模块。它用基于 **scene proxy（场景代理）** 的过程替代了基于 Component 的注册过程，且可以接收任意数据。**注册成本从关键路径中移除**。所有处理**在后台异步执行**。由于简化的特性，资产管理器具有**更优的内存访问模式**，处理数据更快。

> I highly recommend checking this module in your project even if you are not using Fast Geo.

即使不使用 Fast Geo，我也强烈建议检查你项目中的这个模块。

---

## 9. Additional UE 5.6 Improvements — 其他 5.6 流送改进

> The **World Partition Update Streaming State** now supports running **asynchronously** — the process where World Partition decides which cells need to be loaded, unloaded, activated, deactivated.

**World Partition 更新流送状态**现在支持**异步运行**——即 World Partition 决定哪些 cell 需要加载、卸载、激活、停用的过程。

> Similar to **BeginPlay**, **EndPlay** can now be done **incrementally** during **RemoveFromWorld**. This allows to better respect the **game thread time budget**.

与 **BeginPlay** 类似，**EndPlay** 现在可以在 **RemoveFromWorld** 时**增量执行**。这有助于更好地遵守**游戏线程时间预算**。

> We added the possibility for **primitives to be added to the scene outside of the game thread** while registering the component in the game thread.

我们添加了在游戏线程注册 Component 时，**在游戏线程外向场景添加 primitives** 的可能性。

> We added **caching of ISM component bounds** at **cook time**, which reduces the **component registration time**.

我们添加了 **cook 时缓存 ISM Component bounds** 的功能，减少了 **Component 注册时间**。

---

## 10. Future Research — 未来研究方向

> **Fast Geo** remains in active development with ongoing improvements like support for more **primitive types** or support for **movable objects**.

**Fast Geo** 仍在积极开发中，计划支持更多**图元类型**和**可移动对象**。

> **Mutators** might allow us to mutate more properties — like maybe they could resolve the **HLOD assignment process** for us as well.

**Mutators** 可能允许变换更多属性——也许还能为我们解决 **HLOD 分配过程**。

> With more advanced **transformers**, we might be able to separate the **collision representation** and **graphical representation** of the level and put them into two separate grids to **stream them separately**.

通过更高级的 **transformers**，我们可能能够分离 level 的**碰撞表示**和**图形表示**，放入两个独立的网格中**分别流送**。

> Despite our improvements, the **texture streaming** process is still very CPU heavy. The estimation for **wanted mips** is very optimistic. With **sampler feedback**, we might get much more detailed information that would allow us to be more aggressive with **memory pool sizes**.

尽管我们做了改进，**纹理流送**过程仍然非常 CPU 密集。对 **wanted mips（所需 mip 级别）** 的估算非常乐观。通过 **sampler feedback（采样器反馈）**，我们可能获得更详细的信息，从而更积极地管理**内存池大小**。

---

## Summary — 总结

> We implemented **mutators** so we can automate World Partition. The **runtime hash set** to stream in 3D and allow us to experiment with new strategies. **Transformers** will help us optimize content we build. **Asynchronous physics state creation** moves all the heavy processing to the workers. **Unified streaming budget** allows us to worry about fewer budgets and use them efficiently. **Fast Geo** to stream static geometry more efficiently. And finally **Simple Streamable Asset Manager** to execute in the background so we don't have to worry about it.

我们实现了 **mutators** 来自动化 World Partition。**Runtime hash set** 支持 3D 流送并允许实验新策略。**Transformers** 帮助优化已构建的内容。**异步物理状态创建**将所有繁重处理移至 worker。**统一流送预算**让我们减少预算管理负担并高效使用。**Fast Geo** 更高效地流送静态几何体。最后 **Simple Streamable Asset Manager** 在后台执行，无需担忧。

---

## 术语表 — Key Terminology

| 英文术语 | 中文翻译 | 简要说明 |
|----------|----------|----------|
| World Partition | 世界分区 | UE5 大世界管理系统 |
| Streaming Generation | 流送生成 | 将大世界拆分为可流送 cell 的过程 |
| Actor Descriptor | Actor 描述符 | Actor 的流送相关元信息 |
| Spatially Loaded Flag | 空间加载标志 | 控制 Actor 是否按空间位置流送 |
| Runtime Grid | 运行时网格 | Actor 所属的流送网格 |
| Runtime Data Layers | 运行时数据层 | 可按需切换的内容层 |
| Partitioning | 分区 | 将 Actor 分配到 cell 的过程 |
| Mutators | 变换器/修改器 | 代码驱动修改 Actor Descriptor |
| Cell Transformers | 单元格转换器 | 后分区阶段转换 cell 内容 |
| ISM (Instanced Static Mesh) | 实例化静态网格 | 合并相同 mesh 为实例渲染 |
| HLOD (Hierarchical LOD) | 层级 LOD | 远距离合并渲染的 LOD 系统 |
| Critical Path | 关键路径 | 帧内必须完成的串行处理链 |
| Traversal Stutter | 遍历卡顿 | 移动时流送跟不上导致的卡顿 |
| AddToWorld | 添加到世界 | Level 加载后注册 Actor 到场景 |
| RemoveFromWorld | 从世界移除 | 卸载 level 时注销 Actor |
| ProcessAsyncLoading | 处理异步加载 | 游戏线程处理异步加载完成的资产 |
| Post Load | 后加载 | 资产加载后的初始化处理 |
| Physics State | 物理状态 | 物理引擎中对象的碰撞/模拟表示 |
| Chaos | Chaos 物理引擎 | UE5 的物理引擎 |
| Budget | 预算 | 分配给某操作的帧时间上限 |
| Hitch | 卡顿/尖峰 | 单帧超时导致的掉帧 |
| Amortize | 摊销 | 将开销分摊到多帧 |
| Fast Geo | 快速几何体 | 绕过 Actor 注册流程的静态几何体流送 |
| UObject | UObject | UE 的基础对象类型，有 GC 开销 |
| Render Proxy | 渲染代理 | 渲染线程的对象表示 |
| Scene Proxy | 场景代理 | Primitive 在场景中的渲染端表示 |
| Garbage Collection (GC) | 垃圾回收 | UObject 内存回收机制 |
| Mip / Mipmap | 多级渐远纹理 | 不同分辨率的纹理层级 |
| Texture Pool | 纹理池 | GPU 纹理内存预算池 |
| Sampler Feedback | 采样器反馈 | GPU 报告实际使用的纹理区域 |
| Virtual Textures | 虚拟纹理 | 按需流送纹理块的系统 |
| Nanite | Nanite 虚拟几何体 | UE5 虚拟化微多边形系统 |
| PIE (Play In Editor) | 编辑器内运行 | 在编辑器中测试游戏 |
| Cook / Cooking | 烘焙 | 将资产打包为目标平台格式 |
| Embedded Asset | 嵌入式资产 | 内嵌在 level 内的资产 |
| Persistent Level | 持久关卡 | 始终加载的主关卡 |
| Delegate | 委托 | UE 的事件回调机制 |
| Replicated Actor | 复制 Actor | 需要网络同步的 Actor |
| Component Mobility | 组件可移动性 | Static/Stationary/Movable 三种模式 |
