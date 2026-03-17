---
title: "Chapter 2 术语表"
date: 2026-03-17
tags: [pbrt, glossary, monte-carlo]
category: study-notes
description: "PBRT4 第2章 Monte Carlo Integration 专业术语速查"
status: in-progress
---

# Chapter 2 术语表

> 收录本章中出现的专业术语，按类别分组。常见数学概念（如 variance、PDF）不重复收录，详见 [[C2.1-monte-carlo-basics]]。

---

## 渲染 / 图形学术语

| 术语 | 中文 | 说明 |
|------|------|------|
| integrand | 被积函数 | 积分号下的函数 $f(x)$，渲染中通常是 BSDF × 入射辐射 × 余弦 |
| incident radiance | 入射辐亮度 | 到达表面某点某方向的光能量密度 |
| BSDF / BxDF | 双向散射分布函数 | 描述表面如何散射光线，`BxDF` 是 pbrt 中的基类 |
| Fresnel coefficient | 菲涅尔系数 | 决定光线在界面处反射与折射的能量比例 |
| GGX | GGX 微表面分布 | 一种常用的微表面法线分布函数，用于建模粗糙表面高光 |
| cosine factor | 余弦因子 | Lambert 余弦定律中 $\cos\theta$ 项，光线越倾斜贡献越小 |
| shadow ray | 阴影光线 | 从着色点射向光源，测试是否被遮挡的光线 |
| area light | 面积光源 | 有面积的光源，产生软阴影（如矩形灯、球灯） |
| environment map | 环境贴图 | 用 2D 纹理描述全方向环境照明（如 HDR 天空） |
| direct illumination | 直接光照 | 光源直接到达着色点的分量，不经过其他表面反射 |
| progressive rendering | 渐进式渲染 | 逐步增加每像素采样数并实时更新画面 |
| firefly | 萤火虫噪点 | 极少数像素因权重过大而异常明亮的视觉瑕疵 |
| bounce / depth | 弹射 / 深度 | 光线在场景中反射或折射的次数 |
| path throughput | 路径吞吐量 | 光线沿路径累积的衰减因子，代码中的 `beta` |
| specular | 镜面的 | 理想光滑表面的反射/折射，方向唯一确定 |
| shading point | 着色点 | 光线与表面交点，在此处进行光照计算 |
| supersampling | 超采样 | 在每个像素内取多个样本以抗锯齿 |
| light transport equation | 光线传输方程 | 描述场景中光能平衡的积分方程，即渲染方程 |
| low-discrepancy sequence | 低差异序列 | 用数论方法生成的覆盖性优于纯随机的样本序列（Halton、Sobol） |
| bilinear interpolation | 双线性插值 | 在 2D 矩形四角之间的线性插值，`SampleBilinear` 的采样对象 |
| polar coordinates | 极坐标 | $(r, \theta)$ 表示 2D 点，Jacobian = $r$ |
| spherical coordinates | 球坐标 | $(\theta, \varphi)$ 表示方向，Jacobian = $\sin\theta$（单位球面） |

## 概率 / 统计术语

| 术语 | 中文 | 说明 |
|------|------|------|
| canonical uniform random variable | 标准均匀随机变量 | 记作 $\xi$，在 $[0,1)$ 上均匀分布，所有采样的起点 |
| unbiased / biased | 无偏 / 有偏 | 估计器期望值是否精确等于真实值 |
| consistent | 一致的 | $N \to \infty$ 时收敛到真实值（即使有偏） |
| convergence rate | 收敛速率 | 误差随样本数减小的速度，MC 为 $O(1/\sqrt{N})$ |
| variance reduction | 方差缩减 | 使每个样本更有效的技术总称 |
| stratum (pl. strata) | 层 | 分层采样中积分域的各个子区域 |
| inter-stratum variance | 组间方差 | 各层均值与全局均值之差，分层采样消除的部分 |
| termination probability | 终止概率 | 俄罗斯轮盘赌中路径被终止的概率 $q$ |
| one-sample model | 单样本模型 | MIS 中以概率选择一个策略并只从中采样 |
| piecewise constant | 分段常数 | 在不同区间取不同常数值的分布/函数 |
| quadrature | 数值求积法 | 传统确定性数值积分方法（梯形法、高斯求积等） |
| curse of dimensionality | 维度灾难 | 高维空间所需样本数随维度呈指数增长的困境 |
| inversion method | 反演法 | 通过反转 CDF 将均匀样本映射到目标分布，渲染中最重要的采样技术 |
| Jacobian (determinant) | 雅可比（行列式） | 多维变换的面积/体积缩放因子，$p_T(y) = p(x) / \|J_T\|$ |
| marginal density | 边缘密度 | 积分掉部分维度后得到的低维密度函数 |
| conditional density | 条件密度 | 给定一个维度的值后，另一个维度的密度函数 |
| bijection | 双射 | 一对一且满射的函数，变换公式要求 $T$ 必须是双射 |
| normalization | 归一化 | 将函数除以其积分使之成为合法 PDF（积分为 1） |

## pbrt 代码标识符

| 标识符 | 类型 | 说明 |
|--------|------|------|
| `Sample_f()` | 方法 | BxDF 的重要性采样接口，返回采样方向、BSDF 值和 PDF |
| `BSDFSample` | 结构体 | `Sample_f()` 的返回值，含 `.f` `.wi` `.pdf` 字段 |
| `BalanceHeuristic()` | 函数 | 计算 balance heuristic 权重（两分布版本） |
| `PowerHeuristic()` | 函数 | 计算 power heuristic 权重，$\beta=2$ |
| `ConductorBxDF` | 类 | 导体材质，按 GGX 分布重要性采样 |
| `DielectricBxDF` | 类 | 电介质材质，按 Fresnel 系数分配反射/折射 |
| `RandomWalkIntegrator` | 类 | 第 1 章教学积分器，球面均匀采样 PDF = $1/(4\pi)$ |
| `PathIntegrator` | 类 | 第 13 章路径追踪积分器，使用 MIS + 轮盘赌 |
| `StratifiedSampler` | 类 | 分层采样器 |
| `HaltonSampler` | 类 | Halton 低差异序列采样器 |
| `SobolSampler` | 类 | Sobol 低差异序列采样器 |
| `SampledSpectrum` | 类 | 采样光谱，表示多波长的光能量 |
| `MaxComponentValue()` | 方法 | 返回光谱各通道最大值，用于轮盘赌存活概率 |
| `Float` | 类型别名 | pbrt 中的浮点类型（通常为 `float` 或 `double`） |
| `SampleDiscrete()` | 函数 | 从非归一化权重中离散采样，$O(n)$ 线性搜索 |
| `AliasTable` | 类 | 多次离散采样的 $O(1)$ 替代方案，$O(n)$ 预处理 |
| `SampleLinear()` | 函数 | 从线性函数 $(1-x)a + xb$ 的分布中采样（CDF 反演） |
| `LinearPDF()` | 函数 | 计算线性分布的 PDF |
| `InvertLinearSample()` | 函数 | 线性采样的逆操作（= 计算 CDF） |
| `SampleBilinear()` | 函数 | 从双线性函数分布中 2D 采样（marginal-conditional） |
| `BilinearPDF()` | 函数 | 计算双线性分布的 PDF |
| `InvertBilinearSample()` | 函数 | 双线性采样的逆操作 |
| `NextFloatDown()` | 函数 | 返回比输入小的下一个浮点数，用于边界保护 |
| `OneMinusEpsilon` | 常量 | 小于 1 的最大浮点数，用于限制采样返回值在 $[0,1)$ |
| `Lerp()` | 函数 | 线性插值 $(1-t)a + tb$ |
