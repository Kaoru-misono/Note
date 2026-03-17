---
title: "PBRT4 学习总览"
date: 2026-03-17
tags: [pbrt, rendering, graphics, physically-based-rendering]
category: study-notes
description: "Physically Based Rendering: From Theory to Implementation (4th Edition) 学习总览与章节规划"
status: in-progress
source: "https://pbr-book.org/"
---

# PBRT4 学习总览

## 关于本书

**Physically Based Rendering: From Theory to Implementation** 第四版（PBRT4）是计算机图形学中基于物理渲染的经典教材，由 Matt Pharr、Wenzel Jakob 和 Greg Humphreys 合著。本书将渲染理论与完整的渲染器实现相结合，第四版全书代码以 literate programming 风格呈现，可在线免费阅读。

## 前置知识

- **线性代数**：向量、矩阵变换、齐次坐标
- **微积分**：积分、概率密度函数、蒙特卡洛方法基础
- **C++ 基础**：模板、智能指针、现代 C++ 特性（本书使用 C++17）
- **光学 / 辐射度量学基础**：了解光的物理模型有助于理解 radiometry 章节

## 章节规划

### Part I — 基础设施

- [[introduction]] - 渲染与 pbrt 系统概述
- [[monte-carlo-integration]] - Monte Carlo 积分方法
- [[geometry-and-transformations]] - 几何基元与坐标变换
- [[radiometry-spectra-and-color]] - 辐射度量、光谱与色彩

### Part II — 图像生成

- [[cameras-and-film]] - 相机模型与胶片
- [[shapes]] - 几何形状（球、三角形、曲线等）
- [[primitives-and-intersection-acceleration]] - 图元与加速结构（BVH、kd-tree）
- [[sampling-and-reconstruction]] - 采样理论与图像重建

### Part III — 光与材质

- [[reflection-models]] - 反射模型（BRDF/BSDF）
- [[textures-and-materials]] - 纹理系统与材质
- [[volume-scattering]] - 体积散射与参与介质
- [[light-sources]] - 光源类型与采样

### Part IV — 光线传输

- [[light-transport-surface-reflection]] - 光线传输 I：表面反射（Path Tracing）
- [[light-transport-volume-rendering]] - 光线传输 II：体积渲染
- [[wavefront-rendering-on-gpus]] - GPU 上的 Wavefront 渲染

## 学习策略

1. **跟着书走代码**：每章对应 pbrt 源码，理论与实现对照阅读
2. **概念笔记联动**：遇到核心概念（如 [[Monte Carlo Method]]、[[BSDF]]、[[BVH]]）单独建概念笔记
3. **练习与实验**：利用 pbrt 渲染器跑场景，修改参数观察效果
4. **循序渐进**：Part I → II → III → IV，不跳章节
