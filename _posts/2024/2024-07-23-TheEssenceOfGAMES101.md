---
layout: post
title:  "GAMES101知识精炼"
date:   2024-07-23 16:16:00
category: Rendering
---

总结 [GAMES101](https://sites.cs.ucsb.edu/~lingqi/teaching/games101.html) 的知识点，以及一些扩展知识点。

- [1 点积（Dot Product）](#1-点积dot-product)
- [2 叉积（Cross Product）](#2-叉积cross-product)
- [3 矩阵（Matrix）](#3-矩阵matrix)
  - [3.1 三维空间中的变换](#31-三维空间中的变换)
  - [3.2 罗德里格斯公式（Rodrigues’ Rotation Formula）](#32-罗德里格斯公式rodrigues-rotation-formula)
- [4 视图变换（View Transformation）](#4-视图变换view-transformation)
- [5 投影变换（Projection Transformation）](#5-投影变换projection-transformation)
- [6 视口变换（Viewport Transform）](#6-视口变换viewport-transform)
- [7 三角形图元（Triangle Primitive）](#7-三角形图元triangle-primitive)
- [8 光栅化（Rasterization）](#8-光栅化rasterization)
  - [8.1 光栅化的一些加速优化](#81-光栅化的一些加速优化)
    - [8.1.1 轴对齐包围盒（Axis-aligned Bounding Box, AABB）](#811-轴对齐包围盒axis-aligned-bounding-box-aabb)
    - [8.1.2 将屏幕像素视为一行一行像素的集合](#812-将屏幕像素视为一行一行像素的集合)
- [9 抗锯齿（Antialiasing）](#9-抗锯齿antialiasing)
  - [9.1 MSAA](#91-msaa)
- [10 深度缓冲算法（Z-buffering）](#10-深度缓冲算法z-buffering)
- [11 重心坐标插值（Barycentric Interpolation）](#11-重心坐标插值barycentric-interpolation)
- [12 透视纠正插值（Perspective Correct Interpolation）](#12-透视纠正插值perspective-correct-interpolation)
- [13 纹理过滤（Texture Filtering）](#13-纹理过滤texture-filtering)
  - [13.1 纹理过滤的方法](#131-纹理过滤的方法)
    - [13.1.1 最邻近插值（Nearest-neighbor Interpolation）](#1311-最邻近插值nearest-neighbor-interpolation)
    - [13.1.2 双线性过滤（Bilinear Filtering）](#1312-双线性过滤bilinear-filtering)
    - [13.1.3 双三次插值（Bicubic Interpolation）](#1313-双三次插值bicubic-interpolation)
    - [13.1.4 Mipmap](#1314-mipmap)
    - [13.1.5 三线性过滤(Trilinear Filtering)](#1315-三线性过滤trilinear-filtering)
    - [13.1.6 各向异性过滤（Anisotropic Filtering）](#1316-各向异性过滤anisotropic-filtering)
- [14 几何的基本表示方法](#14-几何的基本表示方法)
  - [14.1 隐式几何](#141-隐式几何)
  - [14.2 隐式几何的表示方法](#142-隐式几何的表示方法)
    - [14.2.1 代数曲面（Algebraic Surfaces）](#1421-代数曲面algebraic-surfaces)
    - [14.2.2 构造立体几何（Constructive Solid Geometry, CSG）](#1422-构造立体几何constructive-solid-geometry-csg)
    - [14.2.3 距离函数（Distance Function）](#1423-距离函数distance-function)
    - [14.2.4 水平集方法（Level Set Method, LSM）](#1424-水平集方法level-set-method-lsm)
    - [14.2.5 分型几何（Fractals）](#1425-分型几何fractals)
  - [14.3 显式几何](#143-显式几何)
  - [14.4 显式几何的表示方法](#144-显式几何的表示方法)
    - [14.4.1 点云（Point Cloud）](#1441-点云point-cloud)
    - [14.4.2 多边形网格（Polygon Mesh）](#1442-多边形网格polygon-mesh)
- [15 ⻉塞尔曲线（Bézier Curve）](#15-塞尔曲线bézier-curve)
  - [15.1 de Casteljau 算法（de Casteljau Algorithm）](#151-de-casteljau-算法de-casteljau-algorithm)
    - [15.1.1 二次贝塞尔曲线（Quadratic Bézier Curve）](#1511-二次贝塞尔曲线quadratic-bézier-curve)
    - [15.1.2 三次贝塞尔曲线（Cubic Bézier Curve）](#1512-三次贝塞尔曲线cubic-bézier-curve)
    - [15.1.3 贝塞尔曲线的一般化代数表示](#1513-贝塞尔曲线的一般化代数表示)
  - [15.2 分段贝塞尔曲线（Piecewise Bézier Curve）](#152-分段贝塞尔曲线piecewise-bézier-curve)
    - [15.2.1 $C^0$ 连续（Continuity）](#1521-c0-连续continuity)
    - [15.2.2 $C^1$ 连续（Continuity）](#1522-c1-连续continuity)
  - [贝塞尔曲面（Bézier Surface）](#贝塞尔曲面bézier-surface)
- [16 网格细分（Mesh Subdivision）](#16-网格细分mesh-subdivision)
  - [16.1 Loop 细分（Loop Subdivision）](#161-loop-细分loop-subdivision)
    - [16.1.1 生成更多的网格](#1611-生成更多的网格)
    - [16.1.2 更新新生成的顶点和老的顶点的位置](#1612-更新新生成的顶点和老的顶点的位置)
  - [16.2 Catmull-Clark 细分（Catmull-Clark Subdivision）](#162-catmull-clark-细分catmull-clark-subdivision)
    - [16.2.1 生成更多的网格](#1621-生成更多的网格)
    - [16.2.2 更新新生成的顶点和老的顶点的位置](#1622-更新新生成的顶点和老的顶点的位置)
- [17 网格简化（Mesh Simplification）](#17-网格简化mesh-simplification)
  - [17.1 边坍缩（Edge Collapsing）](#171-边坍缩edge-collapsing)
- [18 光线追踪（Ray Tracing）](#18-光线追踪ray-tracing)
  - [18.1 光线（Light Ray）的基本定义](#181-光线light-ray的基本定义)
  - [18.2 光线投射算法（Ray Casting）](#182-光线投射算法ray-casting)
  - [18.3 Whitted-Style 光线追踪（Whitted-Style Ray Tracing）](#183-whitted-style-光线追踪whitted-style-ray-tracing)
  - [18.4 光线与表面求交（Ray-Surface Intersection）](#184-光线与表面求交ray-surface-intersection)
    - [18.4.1 光线方程（Ray Equation）](#1841-光线方程ray-equation)
    - [18.4.2 光线与隐式几何求交](#1842-光线与隐式几何求交)
    - [18.4.3 光线与显示几何求交](#1843-光线与显示几何求交)
    - [18.4.4 Möller Trumbore 算法（Möller Trumbore Algorithm）](#1844-möller-trumbore-算法möller-trumbore-algorithm)
  - [18.5 轴对齐包围盒（Axis-Aligned Bounding Box, AABB）](#185-轴对齐包围盒axis-aligned-bounding-box-aabb)
    - [光线与 AABB 求交的核心思想](#光线与-aabb-求交的核心思想)
  - [18.6 使用 AABB 加速光线追踪](#186-使用-aabb-加速光线追踪)
    - [18.6.1 均匀网格空间划分（Uniform Spatial Partitions (Grids) ）](#1861-均匀网格空间划分uniform-spatial-partitions-grids-)
    - [18.6.2 空间划分（Spatial Partitions）](#1862-空间划分spatial-partitions)
      - [八叉树（Oct-Tree）](#八叉树oct-tree)
      - [KD树（KD-Tree）](#kd树kd-tree)
    - [18.6.3 层次包围盒（Bounding Volume Hierarchy, BVH）](#1863-层次包围盒bounding-volume-hierarchy-bvh)
    - [18.6.4 空间划分（Spatial Partitions）与物体划分（Object Partitions）的比较](#1864-空间划分spatial-partitions与物体划分object-partitions的比较)
- [19 辐射度量学（Basic radiometry）](#19-辐射度量学basic-radiometry)
  - [19.1 基本的物理量](#191-基本的物理量)
  - [19.2 辐射能量（Radiant Energy）和辐射通量（Radiant Flux）](#192-辐射能量radiant-energy和辐射通量radiant-flux)
  - [19.3 立体角（Solid Angle）](#193-立体角solid-angle)
  - [19.4 微分立体角（Differential Solid Angles）](#194-微分立体角differential-solid-angles)
  - [19.5 辐射强度（Radiant Intensity）](#195-辐射强度radiant-intensity)
  - [19.6 辐射照度（Irradiance）](#196-辐射照度irradiance)
  - [19.7 辐射亮度（Radiance）](#197-辐射亮度radiance)
  - [19.8 辐射照度（Irradiance）与辐射亮度（Radiance）](#198-辐射照度irradiance与辐射亮度radiance)
- [20 BRDF 、反射方程（The Reflection Equation）和渲染方程（The Rendering Equation）](#20-brdf-反射方程the-reflection-equation和渲染方程the-rendering-equation)
  - [20.1 BRDF](#201-brdf)
  - [20.2 反射方程（The Reflection Equation）](#202-反射方程the-reflection-equation)
  - [20.3 渲染方程（The Rendering Equation）](#203-渲染方程the-rendering-equation)
- [21 概率论基础与蒙特卡罗积分](#21-概率论基础与蒙特卡罗积分)
  - [21.1 随机变量](#211-随机变量)
  - [21.2 概率密度函数（Probability Distribution Function, PDF）](#212-概率密度函数probability-distribution-function-pdf)
  - [21.3 期望值（Expected Value）](#213-期望值expected-value)
  - [21.5 蒙特卡罗积分](#215-蒙特卡罗积分)

## 1 点积（Dot Product）

![00_dot_product](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/00_dot_product.jpeg)

两个向量的点积可以表示为：

$$ \vec{a} \cdot \vec{b} = |\vec{a}| \, |\vec{b}| \, \cos{\theta} $$

对于两个单位向量，因为单位向量的长度为 1 ，所以可以表示为：

$$ \hat{a} \cdot \hat{b} = \cos{\theta} $$

点积满足的性质：

- 交换律： $\vec{a} \cdot \vec{b} = \vec{b} \cdot \vec{a}$ ；
- 分配律： $\vec{a} \cdot (\vec{b} + \vec{c}) = \vec{a} \cdot \vec{b} + \vec{a} \cdot \vec{c}$ ；
- 结合律： $(k \vec{a}) \cdot \vec{b} = \vec{a} \cdot (k \vec{b}) = k(\vec{a} \cdot \vec{b})$ ；

点积在笛卡尔坐标系下（三维空间下）的计算：

$$
\vec{a} \cdot \vec{b} = \begin{bmatrix}
    x_a \\
    y_a \\
    z_a
\end{bmatrix}
\cdot
\begin{bmatrix}
    x_b \\
    y_b \\
    z_b
\end{bmatrix}
= x_a x_b + y_a y_b + z_a z_b
$$

点积在图形学中的应用：

- 用以计算两个单位向量之间夹角的大小，这在光照计算中非常常见；
- 通过点积将一个向量投影到另外一个向量；
- 根据 $\cos{\theta}$ 的正负性来判断两个向量是否是同向的；

## 2 叉积（Cross Product）

两个向量的叉积返回一个新的向量，这个新的向量与原来的两个向量都垂直，新的向量的方向与坐标系的定义有关（基于右手定则或左手定则）。

叉积满足以下性质：

$$
\begin{align*}
\vec{a} \times \vec{b} &= -\vec{b} \times \vec{a} \\
\vec{a} \times \vec{a} &= \vec{0} \\
\vec{a} \times (\vec{b} + \vec{c}) &= \vec{a} \times \vec{b} + \vec{a} \times \vec{c} \\
\vec{a} \times (k\vec{b}) &= k(\vec{a} \times \vec{b})
\end{align*}
$$

叉积在笛卡尔坐标系下（三维空间下）的计算：

$$
\vec{a} \times \vec{b} = \begin{bmatrix}
    y_a z_b - y_b z_a \\
    z_a x_b - x_a z_b \\
    x_a y_b - y_a x_b
\end{bmatrix}
$$

也可以写成矩阵与向量相乘的形式：

$$
\vec{a} \times \vec{b} = A * \vec{b} = \begin{bmatrix}
    0 & -z_a & y_a \\
    z_a & 0 & -x_a \\
    -y_a & x_a & 0
\end{bmatrix}
\begin{bmatrix}
    x_b \\
    y_b \\
    z_b
\end{bmatrix}
$$

根据叉积结果的正负性，可以判断两个向量的左右关系。举个例子，在右手法则的笛卡尔坐标系下，对于下图的两个向量 $\vec{a}, \vec{b}$ ：

![01_cross_product](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/01_cross_product.png)

如果 $\vec{a} \times \vec{b} > 0$ ，则表示向量 $\vec{b}$ 在向量 $\vec{a}$ 的左侧；如果叉积结果小于 0 ，则表示在右侧。

继续扩展一下，叉积还可以判断一个点是否在三角形内。看下图的例子，对于三角形 ABC ，怎么判断点 P 是否在三角形内：

![02_cross_product](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/02_cross_product.jpeg)

当以下三个叉积的结果同符号（同为正或同为负）时：

$$
\vec{AB} \times \vec{AP} \\
\vec{BC} \times \vec{BP} \\
\vec{CA} \times \vec{CP}
$$

则表示向量 $\vec{AP}, \vec{BP}, \vec{CP}$ 都在三角形三条边向量 $\vec{AB}, \vec{BC}, \vec{CA}$ 的左侧，也就说明点 P 在三角形 ABC 内。需要注意的是，三角形三条边的向量需要保证沿着同一方向来定义，顺时针或逆时针。

## 3 矩阵（Matrix）

矩阵相乘满足的性质：

- **不满足交换律** ， 对于两个矩阵 $A$ 和 $B$ ， $AB \neq BA$ ；
- 结合律： $(AB)C = A(BC)$ ；
- 分配律： $A(B+C) = AB + AC$ ；

**矩阵的转置（Transpose）** 是将矩阵的行和列互换：

$$
{\begin{bmatrix}
    1 & 2 \\
    3 & 4 \\
    5 & 6
\end{bmatrix}}^T =
\begin{bmatrix}
    1 & 3 & 5 \\
    2 & 4 & 6
\end{bmatrix}
$$

对于两个矩阵 $A$ 和 $B$ ，满足： $(AB)^T = B^T A^T$ 。

**单位矩阵（Identity Matrix）** 是一种特殊矩阵，它在矩阵乘法中起到类似于数字 1 在数值乘法中的作用。通常用 $I$ 来表示。下面是一个 3x3 的单位矩阵：

$$
I_{3x3} = \begin{bmatrix}
    1 & 0 & 0 \\
    0 & 1 & 0 \\
    0 & 0 & 1
\end{bmatrix}
$$

单位矩阵具有以下重要性质：

1. **乘法恒等性**：对于任何 $n \times n$ 的矩阵 $A$ ，有 $AI = IA = A$ 。这意味着单位矩阵在矩阵乘法中起到类似于数字1在数值乘法中的作用；
2. **转置**：单位矩阵的转置仍然是单位矩阵，即 $I_n^T = I_n$ ；
3. **逆矩阵**：单位矩阵的逆矩阵仍然是单位矩阵，即 $I_n^{-1} = I_n$ ；
4. **[行列式（Determinant）](https://en.wikipedia.org/wiki/Determinant)**：单位矩阵的行列式为1，即 $\det(I_n) = 1$ ；

**矩阵的逆（Inverse）** ，对于一个矩阵 $A$ ，如果存在一个矩阵 $B$ 使得 $AB = BA = I$ ，其中 $I$ 是单位矩阵，那么我们称矩阵 $B$ 是矩阵 $A$ 的逆矩阵，记作： $A^{-1}$ 。对于两个矩阵 $A$ 和 $B$ ，满足： $(AB)^{-1} = B^{-1} A^{-1}$ 。

**齐次坐标（Homogenous Coordinates）** ，对于三维空间中的一个点 $(x,y,z)$ ，一个 3x3 的矩阵 $M$ 可以表示对这个点的旋转（Rotation）和缩放（Scale）变换，但是不能表示平移（Translation）变换，通过引入齐次坐标，使得一个矩阵可以同时表示平移，旋转和缩放变换，这种变换也叫做 **仿射变换（Affine Transformations）** ，对于三维空间中的一个点，一个仿射变换可以表示为：

$$
\begin{bmatrix}
    x' \\
    y' \\
    z' \\
    1
\end{bmatrix}=
\begin{bmatrix}
    a & b & c & t_x \\
    d & e & f & t_y \\
    g & h & i & t_z \\
    0 & 0 & 0 & 1
\end{bmatrix}
\begin{bmatrix}
    x \\
    y \\
    z \\
    1
\end{bmatrix}
$$

其中矩阵左上角 3x3 部分表示的是线性的旋转缩放变换，矩阵最后一列表示的是平移变换。

**需要注意的是变换的顺序很重要，一定是先进行线性变换（旋转缩放）再进行平移变换** 。通过上面仿射变换的矩阵运算也能看出来保证了这个顺序，它可以表示成：

$$
\begin{bmatrix}
    x' \\
    y' \\
    z'
\end{bmatrix}=
\begin{bmatrix}
    a & b & c \\
    d & e & f \\
    g & h & i \\
\end{bmatrix}
\begin{bmatrix}
    x \\
    y \\
    z
\end{bmatrix}+
\begin{bmatrix}
    t_x \\
    t_y \\
    t_z
\end{bmatrix}
$$

### 3.1 三维空间中的变换

缩放：

$$
S(s_x,s_y,s_z)=
\begin{bmatrix}
    s_x & 0 & 0 & 0 \\
    0 & s_y & 0 & 0 \\
    0 & 0 & s_z & 0 \\
    0 & 0 & 0 & 1
\end{bmatrix}
$$

平移：

$$
T(t_x,t_y,t_z) = \begin{bmatrix}
    1 & 0 & 0 & t_x \\
    0 & 1 & 0 & t_y \\
    0 & 0 & 1 & t_z \\
    0 & 0 & 0 & 1
\end{bmatrix}
$$

分别绕着 $x$ 轴， $y$ 轴和 $z$ 轴的旋转：

$$
R_x(\alpha) = \begin{bmatrix}
    1 & 0 & 0 & 0 \\
    0 & \cos{\alpha} & -\sin{\alpha} & 0 \\
    0 & \sin{\alpha} & \cos{\alpha} & 0 \\
    0 & 0 & 0 & 1
\end{bmatrix}
$$

$$
R_y(\alpha) = \begin{bmatrix}
    \cos{\alpha} & 0 & \sin{\alpha} & 0 \\
    0 & 1 & 0 & 0 \\
    -\sin{\alpha} & 0 & \cos{\alpha} & 0 \\
    0 & 0 & 0 & 1
\end{bmatrix}
$$

$$
R_z(\alpha) = \begin{bmatrix}
    \cos{\alpha} & -\sin{\alpha} & 0 & 0 \\
    \sin{\alpha} & \cos{\alpha} & 0 & 0 \\
    0 & 0 & 1 & 0 \\
    0 & 0 & 0 & 1
\end{bmatrix}
$$

任意的一个三维的旋转可以写成分别绕 $x$ 轴， $y$ 轴和 $z$ 轴旋转的组合：

$$ R_{xyz}(\alpha,\beta,\gamma) = R_x(\alpha)R_y(\beta)R_z(\gamma) $$

这三个角度 $\alpha,\beta,\gamma$ 也被称为 **欧拉角（Euler angles）** ，分别叫做 `yaw` ， `pitch` 和 `roll` 。

### 3.2 罗德里格斯公式（Rodrigues’ Rotation Formula）

罗德里格斯公式是计算三维空间中，一个向量绕着任意单位旋转轴一定角度后得到一个新的向量的计算公式，而且可以改写成矩阵的形式。

给定一个三维向量 $\vec{v}$ 和一个单位旋转轴 $\hat{k}$ ，以及一个旋转角度 $\theta$ ，罗德里格斯公式计算绕 $\hat{k}$ 轴旋转角度 $\theta$ 后的新向量 $\vec{v'}$ 。公式如下：

$$
\vec{v'} = \cos \theta \vec{v} + (1 - \cos \theta) \hat{k} (\hat{k} \cdot \vec{v}) + \sin \theta (\hat{k} \times \vec{v})
$$

将旋转表示成矩阵的形式为：

$$
R(\hat{k}, \theta) = \cos{\theta} I + (1 - \cos{\theta}) \hat{k} (\hat{k})^T + \sin{\theta} \begin{bmatrix}
    0 & -z_k & y_k \\
    z_k & 0 & -x_k \\
    -y_k & x_k & 0
\end{bmatrix}
$$

罗德里格斯公式推导如下：

![03_rrf_s](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/03_rrf_s.jpeg)

![04_rrf_s](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/04_rrf_s.jpeg)

表示成矩阵的形式的推导：

![05_rrf_s](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/05_rrf_s.jpeg)

## 4 视图变换（View Transformation）

首先，场景中的一个摄像机是怎么定义的：

- 摄像机的位置： $\vec{e}$ ；
- 摄像机看向的方向： $\hat{g}$ ；
- 向上的方向： $\hat{t}$ ，垂直于 $\hat{g}$ ；

视图变换是将世界空间中的物体转换到观察空间（摄像机空间）的矩阵变换，它实际上做的就是把场景中摄像机变换到原点位置，向上方向 $\hat{t}$ 变换到 $+y$ 轴方向，摄像机看向的方向 $\hat{g}$ 变换到 $-z$ 轴方向（这是基于右手法则的坐标系下，而对于有些基于左手法则的坐标系，则是转换到 $+z$ 轴方向）。同时对场景中的物体也应用此变换，这样能保证摄像机和场景内物体的相对位置不会发生变化，也就是说摄像机看到的场景内容不变。下图展示了视图变换的过程：

![06_view_transformation](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/06_view_transformation.jpeg)

视图变换矩阵怎么计算出来的呢？视图变换的过程是这样的：

- 将摄像机位置 $\vec{e}$ 平移到原点；
- 将摄像机看向的方向 $\hat{g}$ 旋转到 $-z$ 轴方向；
- 将摄像机向上的方向 $\hat{t}$ 旋转到 $+y$ 轴方向；
- 最后将 $\hat{g} \times \hat{t}$ 旋转到 $+x$ 轴方向；

所以视图变换可以表示为：

$$M_{view} = R_{view}T_{view}$$

对于平移变换，很容易写出来：

$$
T_{view} =
\begin{bmatrix}
    1 & 0 & 0 & -x_e \\
    0 & 1 & 0 & -y_e \\
    0 & 0 & 1 & -z_e \\
    0 & 0 & 0 & 1
\end{bmatrix}
$$

而对于将 $\hat{g}$ 旋转到 $-z$ 轴方向、 $\hat{t}$ 旋转到 $+y$ 轴方向、 $\hat{g} \times \hat{t}$ 旋转到 $+x$ 轴方向的变换很难写出来，不过我们知道一个矩阵的逆表示的就是这个矩阵变换的逆变换，那么可以考虑写出将 $+x$ 轴方向旋转到 $\hat{g} \times \hat{t}$ 、 $+y$ 轴方向旋转到 $\hat{t}$ ，以及 $z$ 轴方向旋转到 $-\hat{g}$ 的变换矩阵，这个矩阵很好写出来：

$$
R_{view}^{-1}=
\begin{bmatrix}
    x_{\hat{g} \times \hat{t}} & x_t & x_{-g} & 0 \\
    y_{\hat{g} \times \hat{t}} & y_t & y_{-g} & 0 \\
    z_{\hat{g} \times \hat{t}} & z_t & z_{-g} & 0 \\
    0 & 0 & 0 & 1
\end{bmatrix}
$$

举个例子验证一下，将 $+x$ 轴方向 $(1,0,0,0)$ 与此矩阵相乘，得到的就是 $\hat{g} \times \hat{t}$ 。

而我们知道这个矩阵 $R^{-1}_{view}$ 是一个正交矩阵，那么它的逆矩阵就是它的转置，所以：

$$
R_{view} = \begin{bmatrix}
    x_{\hat{g} \times \hat{t}} & y_{\hat{g} \times \hat{t}} & z_{\hat{g} \times \hat{t}} & 0 \\
    x_t & y_t & z_t & 0 \\
    x_{-g} & y_{-g} & z_{-g} & 0 \\
    0 & 0 & 0 & 1
\end{bmatrix}
$$

结合这两个矩阵可以得到视图变换为：

$$
M_{view} = R_{view}T_{view} = \begin{bmatrix}
    x_{\hat{g} \times \hat{t}} & y_{\hat{g} \times \hat{t}} & z_{\hat{g} \times \hat{t}} & 0 \\
    x_t & y_t & z_t & 0 \\
    x_{-g} & y_{-g} & z_{-g} & 0 \\
    0 & 0 & 0 & 1
\end{bmatrix}
\begin{bmatrix}
    1 & 0 & 0 & -x_e \\
    0 & 1 & 0 & -y_e \\
    0 & 0 & 1 & -z_e \\
    0 & 0 & 0 & 1
\end{bmatrix}=
\begin{bmatrix}
    x_{\hat{g} \times \hat{t}} & y_{\hat{g} \times \hat{t}} & z_{\hat{g} \times \hat{t}} & -x_e \\
    x_t & y_t & z_t & -y_e \\
    x_{-g} & y_{-g} & z_{-g} & -z_e \\
    0 & 0 & 0 & 1
\end{bmatrix}
$$

## 5 投影变换（Projection Transformation）

投影变换有两种不同的投影方式， **正交投影（Orthographic Projection）** 和 **透视投影（Perspective Projection）** 。

**视锥体（Frustum）** 是指从摄像机（或观察点）出发的一个锥形体（对于透视投影）或柱形体（对于正交投影），它定义了摄像机能看到的空间范围。

对于正交投影的视锥体，它由以下几个参数来定义：

1. **左剪裁面（Left Clipping Plane）** ；
2. **右剪裁面（Right Clipping Plane）** ；
3. **下剪裁面（Bottom Clipping Plane）** ；
4. **上剪裁面（Top Clipping Plane）** ；
5. **近剪裁面（Near Clipping Plane）** ；
6. **远剪裁面（Far Clipping Plane）** ；

而对于透视投影的视锥体，它由以下几个参数来定义：

1. **视场角（Field of View, FOV）**：这个角度决定了视锥体的开口大小。通常在垂直方向上定义。
2. **纵横比（Aspect Ratio）**：视锥体在水平和垂直方向上的比例，通常等于屏幕的宽高比。
3. **近剪裁面（Near Clipping Plane）**：距离摄像机最近的剪裁面，任何在这个平面之前的物体都会被裁剪掉。
4. **远剪裁面（Far Clipping Plane）**：距离摄像机最远的剪裁面，任何在这个平面之后的物体也会被裁剪掉。

下图展示了正交投影和透视投影的视锥体的区别：

![07_orthographic_perspective](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/07_orthographic_perspective.jpeg)

投影变换矩阵的生成依赖于视锥体的参数，通过投影变换，将观察空间中的点变换到 $[-1,1]^3$ 的标准设备坐标（Normalized Device Coordinates, NDC）空间。

**需要注意的是** ，在有些图形 API 中，标准设备坐标空间的 $z$ 轴坐标范围并不是 $[-1,1]$ （OpenGL），而是 $[0,1]$ （DirectX， Metal， Vulkan） 。

对于正交投影变换，它需要做的也就是：

- 平移视锥体的中心到原点 $(0,0,0)$ ；
- 将视锥体缩放到 $[-1,1]^3$ ，也就是将长宽高缩放到 2（-1到1，长度为2）；

正交投影的矩阵也很好写出来：

$$
M_{ortho} =
\begin{bmatrix}
    2/(r-l) & 0 & 0 & 0 \\
    0 & 2/(t-b) & 0 & 0 \\
    0 & 0 & 2/(n-f) & 0 \\
    0 & 0 & 0 & 1
\end{bmatrix}
\begin{bmatrix}
    1 & 0 & 0 & -(r+l)/2 \\
    0 & 1 & 0 & -(t+b)/2 \\
    0 & 0 & 1 & -(n+f)/2 \\
    0 & 0 & 0 & 1
\end{bmatrix}
$$

需要注意的是，因为观察空间摄像机面向的是 $-z$ 轴的方向，所以 $z$ 的坐标都是负的，也就是说近平面的 $z$ 值是大于远平面的 $z$ 值，有一点反直觉，这也就是为什么 OpenGL 在标准化设备空间使用的是左手坐标系（按照惯例，OpenGL使用的是右手坐标系，而DirectX使用的是左手坐标系）。

而对于透视投影，它的视锥体是一个锥形体，所以相对于正交投影，它首先要将锥形体“挤压”成一个长方体，然后再像正交投影那样，先将长方体的中心平移到原点，然后再将长方体缩放到 $[-1,1]^3$ 。可以表示为：

$$
M_{persp} = M_{ortho}M_{persp \to ortho}
$$

具体对矩阵 $M_{persp \to ortho}$ 的推导这里就不再详细说明，主要思想就是利用相似三角形求出 $x,y$ 的变化规律，再通过近平面远平面上两个特殊的点来确定 $z$ 的变化规律。需要注意的是，对于透视投影变换，变换以后的齐次坐标坐标不为 1，而是 $z$ ，此时是将观察空间的点变换到了 **裁剪空间（Clip Space）** ，最后还需要对裁剪空间的齐次坐标做 **透视除法（Perspective Division）** 才能变换到标准设备坐标空间。

最后写出透视投影的矩阵：

$$
M_{persp}=
M_{ortho}M_{persp \to ortho}=
\begin{bmatrix}
    2/(r-l) & 0 & 0 & 0 \\
    0 & 2/(t-b) & 0 & 0 \\
    0 & 0 & 2/(n-f) & 0 \\
    0 & 0 & 0 & 1
\end{bmatrix}
\begin{bmatrix}
    1 & 0 & 0 & -(r+l)/2 \\
    0 & 1 & 0 & -(t+b)/2 \\
    0 & 0 & 1 & -(n+f)/2 \\
    0 & 0 & 0 & 1
\end{bmatrix}
\begin{bmatrix}
    n & 0 & 0 & 0 \\
    0 & n & 0 & 0 \\
    0 & 0 & n+f & -nf \\
    0 & 0 & 1 & 0
\end{bmatrix}
$$

## 6 视口变换（Viewport Transform）

投影变换之后，要做的就是将标准设备坐标空间变换到实际的屏幕坐标空间了。视口变换将 $[-1,1]^3$ 的标准立方体的 $xy$ 变换到 $[0, \text{width}] \times [0, \text{height}]$ 的屏幕上，而 $z$ 将从 $[-1,1]$ 转换到 $[0,1]$ 并存储在深度缓冲中用于深度测试。

视口变换矩阵：

$$
M_{viewport}=
\begin{bmatrix}
    \frac{\text{width}}{2} & 0 & 0 & \frac{\text{width}}{2} \\
    0 & \frac{\text{height}}{2} & 0 & \frac{\text{height}}{2} \\
    0 & 0 & 1 & 0 \\
    0 & 0 & 0 & 1 \\
\end{bmatrix}
$$

## 7 三角形图元（Triangle Primitive）

三角形是最基本形状基元，广泛用于构建复杂的几何形状，是表示三维模型的基础。它有很多不错的特性：

- 三角形是最基础的多边形；
- 任何其它多边形都可以拆分成多个三角形；
- 三角形保证是一个平面；
- 三角形内部外部定义清晰；
- 定义了三角形三个顶点的属性，可以通过 **重心坐标** 使用[重心坐标插值](#重心坐标插值barycentric-interpolation)使三角形内部各个点的属性有一个平滑的过渡；

三维的模型都是通过一个个三角形图元来定义的，下面是一个三维的球体的例子：

![09_triangle_meshes](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/09_triangle_meshes.jpeg)

## 8 光栅化（Rasterization）

我们知道，经过视口变换后，标准设备坐标空间中的对象被转换到实际的屏幕坐标空间。而下一步要做的，就是光栅化。

光栅化的本质是将矢量图形（即由几何形状如点，线，曲线和多边形定义的图形）转换为栅格图像（即由像素组成的图像，比如显示设备）。简单点来说光栅化就是：遍历显示屏幕上的每个像素，计算每个像素覆盖了哪些三角形图元，将此像素的颜色设置为被覆盖的三角形图元的颜色。

根据上面的描述不难理解，光栅化本质上是一种采样（Sampling）的过程。用伪代码可以表示为：

```cpp
for (int x = 0; x < width; ++x)
    for (int y = 0; y < height; ++y)
        image[x][y] = inside(triangle, x + 0.5, y + 0.5);
```

需要注意的是，在光栅化的过程中，对于一个 $1 \times 1$ 的像素，我们将其视为空间中的一个点，而不是一个 $1 \times 1$ 的区域。一般认为是这个像素的中心，也就是屏幕上像素坐标 $x,y$ 加上 0.5 的坐标。

对于 `inside(triangle, x + 0.5, y + 0.5)` 函数，怎么判断一个点在不在一个三角形图元中，在上面 [叉积（Cross Product）](#叉积cross-product) 有提到。

下图就是一个光栅化的例子：

![08_rasterization](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/08_rasterization.jpeg)

有一种特殊的情况，采样的像素点在两个三角形图元公共边上，如下图：

![10_sample_point](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/10_sample_point.jpeg)

这种情况要取决于不同的标准，要么不做处理，要么做特殊处理。在一些图形API上比如 OpenGL 和 DirectX ，它们会定义落在三角形上边和左边的点在三角形内，在三角形右边和下边的点在三角形外。

### 8.1 光栅化的一些加速优化

#### 8.1.1 轴对齐包围盒（Axis-aligned Bounding Box, AABB）

计算出每个三角形图元的包围盒，找到其中最小和最大的 $x, y$ 坐标。在光栅化的过程中，只遍历计算被包围盒覆盖的像素。如下图所示：

![11_rasterization_aabb](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/11_rasterization_aabb.jpeg)

#### 8.1.2 将屏幕像素视为一行一行像素的集合

将屏幕像素视为一行一行像素的集合，找出每一行像素被三角形图元覆盖的最小和最大 $x$ 坐标。在光栅化的过程中，只需要遍历计算每行像素中与被三角形图元覆盖的像素。如下图所示：

![12_rasterization_rows](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/12_rasterization_rows.jpeg)

在这里做一个扩展，怎么计算每一行像素与三角形图元三条边的交点呢？

首先，每个三角形图元的顶点属性（坐标、颜色、深度等）我们是知道的，而且每行像素的 $y$ 坐标也是已知的。计算每一行像素被三角形图元覆盖的像素可以理解为：

- 通过线段与线段求交的算法计算出每一行的线段与三角形图元每条边的交点；
- 将这些交点通过从左到右（也就是 $x$ 轴坐标从小到大 ）的顺序排序；
- 这样就得到了每一行像素被三角形图元覆盖的最小和最大 $x$ 坐标；

线段与线段求交的算法：

看下面的图，线段 $p1p2$ 表示每一行像素，线段 $q1q2$ 表示三角形图元的一条边，现在要求交点出的 $x$ 坐标：

![13_line_line_intersection](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/13_line_line_intersection.jpeg)

求解此线段相交的主要思想是通过点-斜率（Point-slope）的形式来表示线段：

- 对于三角形图元的边 $q1q2$ ，可以求出它的斜率（Slope）是： $m = \frac{q2.y - q1.y}{q2.x - q1.x}$ 。那么对于此三角形图元边上的一个点，这个点的 $y$ 坐标可以表示为： $y = m(x - q1.x) + q1.y$ ，其中 $x$ 是这个点的 $x$ 坐标；
- 如果某一行像素 $p1p2$ 与三角形图元的这条边 $q1q2$ 有交点，那么可以表示为： $p1.y = m(x - q1.x) + q1.y$ ，因为屏幕上的每一行像素的 $y$ 坐标是已知的，也就是说 $p1.y$ 是已知的，三角形图元边的斜率 $m$ 也是已知的，所以 $x$ 就可以求解出来。

求交的一些特殊情况：

对于求出来交点，必须要在三角形图元的包围盒内，因为通过点-斜率表示的是一条无限长的直线。如下图所示，对于三角形图元下面的这条边，交点在三角形图元外：

![14_intersection_special_0](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/14_intersection_special_0.jpeg)

三角形图元的一条边的斜率为 0，也就是这条边是水平的。对于这种情况，可以直接忽略掉这条边，因为三角形图元的另外两条边保证会与此行像素有一个交点。如下图：

![15_intersection_special_1](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/15_intersection_special_1.jpeg)

三角形图元的一条边是完全垂直的，这种情况使用边上任意一点的 $x$ 坐标。如下图所示：

![16_intersection_special_2](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/16_intersection_special_2.jpeg)

## 9 抗锯齿（Antialiasing）

前面我们说过，光栅化本质上是一种采样，采样点是一个个离散的像素的中心点，而采样的信号是连续的网格数据。当采样点不够密集时，就会造成 **锯齿走样（Aliasing Artifacts）** 的问题。下图展示了锯齿走样的现象，对于连续的三角形图元，采样出来的结果是离散的：

![17_aliasing](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/17_aliasing.jpeg)

出现锯齿走样问题的本质就是：数据信号变化的太快（信号变化的快的都是高频的信号）而采样的频率太低。

解决锯齿走样问题的方法：

- 提高采样频率，使用更大分辨率的屏幕，更大的帧缓冲区（Framebuffer）等；
- 做抗锯齿，在采样之前对信息做 **过滤（Filtering）** ，过滤掉高频的信息。一般通过 **卷积（Convolution）** 来过滤高频信息，然后再采样；

对于第二种抗锯齿的方法，原理就是根据像素被三角形图元覆盖的比例来做着色。下图分别展示了不同覆盖率下的最终像素的结果：

![18_antialiasing](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/18_antialiasing.jpeg)

### 9.1 MSAA

MSAA 的原理就是对于每个像素，不再使用一个点来采样，而是采样 $N$ 个点，分别计算它们是否被三角形图元覆盖，最终根据被覆盖的数量与总的采样点数量的比例来决定最终像素的颜色。下图很好的展示了 MSAA 的原理：

![19_msaa_0](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/19_msaa_0.jpeg)

![20_msaa_1](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/20_msaa_1.jpeg)

## 10 深度缓冲算法（Z-buffering）

在实际光栅化的过程中，一个像素可能覆盖了很多个三角形图元，对于这种情况，像素最终的颜色应该由哪个三角形图元来确定呢？这就要使用到深度缓冲算法。这个算法的灵感来自画家算法（Painter’s Algorithm）：画家在创作一幅作品时，会根据从远到近的顺序来画出作品中的每个元素。

深度缓冲算法中的深度表示的是什么呢？对于标准设备坐标空间中的坐标，它的 $x,y,z$ 坐标的范围都是 $[-1,1]$，对于 $z$ 坐标的值，将其从 $[-1,1]$ 的范围映射到 $[0,1]$ 的范围中，映射后的值就是 **深度值（Depth Value 或者 Z Value）** 。需要注意的是，对于非 OpenGL 的图形 API，例如 DirectX，Metal 和 Vulkan，它们的标准坐标空间中的 $z$ 坐标的范围是 $[0,1]$ ，所以不需要这种额外的映射操作。

我们知道，在光栅化的过程中，每个像素的颜色存储在 **帧缓冲（Frame Buffer）** ，也有叫做 **颜色缓冲（Color Buffer）** 中。深度缓冲算法的原理就是，创建一个额外的缓冲区来存储每个像素覆盖的所有三角形图元中的最小深度值（也就是离摄像机最近的三角形图元的深度值），这个额外的缓冲区叫做 **深度缓冲（Depth Buffer 或者 Z Buffer）** 。下图展示了一个三维场景中深度缓冲的样子：

![21_depth_buffer](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/21_depth_buffer.jpeg)

深度缓冲算法的核心逻辑：创建一个与帧缓冲同等分辨率的深度缓冲（每个像素的深度值都初始化为一个无限大的值），也就是说深度缓冲中的每个像素与帧缓冲中的每个像素一一对应，深度缓冲中每个像素存储的就是帧缓冲中对应像素的最小深度值。在光栅化的过程中，如果一个像素覆盖了某一个三角形图元，将深度缓冲中存储的深度值与此三角形图元的深度值做比较，只有当此三角形图元的深度值小于深度缓冲中存储的深度值时，将此像素的颜色更新为此三角形图元的颜色，并将深度缓冲中此像素的深度值更新为此三角形图元的深度。如果此三角形图元的深度值大于等于深度缓冲中存储的深度值时，跳过这次采样处理。这种比较深度的操作叫做 **深度测试（Depth Test）** 。深度缓冲算法用伪代码可以表示为：

```cpp
for (each triangle T)
    for (each sample(x,y,z) in T)
        if (T.z < zbuffer[x,y])       // closest sample so far
            framebuffer[x,y] = T.rgb; // update color
            zbuffer[x,y] = T.z        // update depth
        else
            // do nothing, this sample is occluded
```

下图展示了深度缓冲算法的原理：

![22_depth_buffer_algorithm](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/22_depth_buffer_algorithm.jpeg)

对于 $n$ 个三角形图元，深度缓冲算法的算法复杂度是 $O(n)$ ，因为它记录的永远是三角形图元最小的深度值。

## 11 重心坐标插值（Barycentric Interpolation）

首先来说说什么是 **重心坐标（Barycentric Coordinates）** ：对于一个三角形，其顶点分别为 A 、 B 、 C ，任意一点 P 在三角形内部或者边界上的位置都可以用三个重心坐标 $\alpha$ 、 $\beta$ 、 $\gamma$ 来表示，这些重心坐标满足以下条件：

$$ P = \alpha A + \beta B + \gamma C $$

并且：

$$ \alpha + \beta + \gamma = 1 $$

重心坐标插值就是利用重心坐标来对三角形内部或边界上任意一点的属性的差值计算。利用重心坐标插值可以根据顶点的坐标、颜色、纹理坐标、法线、深度值等属性插值计算出三角形内或边界上任意一点的坐标、颜色、纹理坐标、法线、深度值等属性。

下面来看看重心坐标是怎么计算的，对于下面的一个三角形 $P_1 P_2 P_3$ ，它的内部有一个点 $P$ ：

![23_barycentric_coordinates](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/23_barycentric_coordinates.jpeg)

其中：

- 三角形 $P_1 P_2 P_3$ 的面积为 $S$ ；
- 三角形 $P P_2 P_3$ 的面积为 $S_1$ ；
- 三角形 $P P_3 P_1$ 的面积为 $S_2$ ；
- 三角形 $P P_1 P_2$ 的面积为 $S_3$ ；

那么重心坐标可以表示为：

$$
\alpha = \frac{S_1}{S} ,\  \beta = \frac{S_2}{S} ,\  \gamma = \frac{S_3}{S}
$$

也就是说：

$$ P = \frac{S_1}{S} P_1 + \frac{S_2}{S} P_2 + \frac{S_3}{S} P_3 $$

那么三角形的面积怎么求呢？

首先我们知道， **两个向量叉积得到的新的向量的长度等于由这两个向量构成的平行四边形的面积** 。这个很好证明，先看图：

![24_cross_product_cal_area](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/24_cross_product_cal_area.jpeg)

对于向量 $\vec{a}$ 和向量 $\vec{b}$ 构成的平行四边形，它的面积可以表示为：

$$
S = |\vec{a}| h = |\vec{a}| \ |\vec{b}| \sin{\theta}
$$

而我们知道，向量叉积的模和向量的长度以及夹角有如下关系：

$$
|\vec{a} \times \vec{b}| = |\vec{a}| \ |\vec{b}| \sin{\theta}
$$

所以说两个向量叉积得到的新的向量的长度等于由这两个向量构成的平行四边形的面积。 **那么两个向量构成三角形的面积就是等于构成平行四边形的面积的一半** 。

回到上面的三角形的例子，三角形 $P_1 P_2 P_3$ 的面积就可以表示为：

$$
S_{P_1 P_2 P3} = 0.5 * \text{length}(\text{corss}(P_1 - P_2, P_3 - P_2))
$$

小的三角形 $S_1, S_2, S_3$ 的面积计算方法同理。

有了重心坐标，那么就很容易计算出三角形内或边界上任意一点的属性了，比如三角形内 $P$ 点的颜色可以这样计算：

$$
P.color = P_{1}.color * \frac{S_1}{S} + P_{2}.color * \frac{S_2}{S} + P_{3}.color * \frac{S_3}{S}
$$

## 12 透视纠正插值（Perspective Correct Interpolation）

在光栅化的过程中，像素的颜色、法线、纹理坐标等属性通常是通过重心坐标对其覆盖的三角形图元的三个顶点的颜色、法线、纹理坐标属性插值而来。在三维空间中，每个属性的值在每个三角形图元上线性变化。然而， **在三维顶点通过透视投影被投影到二维屏幕上之后，这种属性值在三维空间中的线性变化并不会转换为屏幕空间中的类似线性变化** 。而我们知道重心坐标是基于屏幕空间的坐标计算出来的，因此，直接使用重心坐标对这些属性进行线性插值会得到错误的结果：

![25_perspective_correct_interpolation_0](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/25_perspective_correct_interpolation_0.jpeg)

为了更容易说明，上图展示了二维空间中的一条线段被投影到一维图像平面上的例子，但同样的结论也可以应用于光栅化中三维线性三角形图元被投影到二维屏幕上的情况。在此图中：

- 二维空间中线段 $AB$ 的顶点 $A$ 和 $B$ 分别被投影到一维平面上的点 $a$ 和 $b$ ，顶点处的属性值是强度值（intensity） ；
- 在顶点 $A$ 和顶点 $B$ 处，强度值分别为 0.0 和 1.0 ，因此，在顶点 $a$ 和顶点 $b$ 处，强度值分别为 0.0 和 1.0 ；
- 假设点 $c$ 是一维平面上 $a$ 和 $b$ 之间的中点，那么在平面中对 $a$ 和 $b$ 的强度值进行线性插值，那么 $c$ 处的强度值应为 0.5 ；
- 然而，如果我们将点 $c$ 反投影到线段 $AB$ 上的 $C$ 点，可以看到点 $C$ 不一定是 $A$ 和 $B$ 之间的中点，因为强度值在 $A$ 和 $B$ 之间是线性变化的，如果 $C$ 不是 $A$ 和 $B$ 之间的中点，那么 $C$ 处的强度值不应该是 0.5 ；

**透视纠正插值** 是将顶点属性值与顶点 **深度值（观察空间中的线性深度值）** 相关联，在插值计算中考虑深度的影响。通过透视纠正插值，可以根据屏幕空间的重心坐标插值得到正确的线性属性。

下面还是以上图为例子，展示怎么具体推导出屏幕空间中正确插值的深度值 $Z$ 。如下图所示，摄像机位于观察空间中的原点 $(0, 0)$ ，摄像机朝向 $+z$ 方向，一维平面位于摄像机前方距离为 $d$ 的位置， $s$ 是一维平面中的插值参数，而 $t$ 是二维线段中的插值参数：

![26_perspective_correct_interpolation_1](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/26_perspective_correct_interpolation_1.jpeg)

首先，根据相似三角形，我们很容易得到如下关系：

$$ \frac{X_1}{Z_1} = \frac{u_1}{d} \Rightarrow X_1 = \frac{u_1 Z_1}{d} \tag{1} $$

$$ \frac{X_2}{Z_2} = \frac{u_2}{d} \Rightarrow X_2 = \frac{u_2 Z_2}{d} \tag{2} $$

$$ \frac{X_t}{Z_t} = \frac{u_s}{d} \Rightarrow Z_t = \frac{d X_t}{u_s} \tag{3} $$

在一维平面中， $u_s$ 可以通过线性插值表示为：

$$ u_s = (1-s)u_1 + s u_2 = u_1 + s(u_2 - u_1) \tag{4} $$

对于处于摄像机二维空间中线段 $AB$ 上的点 $C$ ，其坐标 $X_t$ 和深度值 $Z_t$ 通过线性插值表示为：

$$ X_t = (1-t) X_1 + t X_2 = X_1 + t (X_2 - X_1) \tag{5} $$

$$ Z_t = (1-t) Z_1 + t Z_2 = Z_1 + t (Z_2 - Z_1) \tag{6} $$

将式（4）和式（5）带入式（3）可以得到：

$$ Z_t = \frac{d (X_1 + t (X_2 - X_1))}{u_1 + s(u_2 - u_1)} \tag{7} $$

再将式（1）和式（2）带入式（7）可以得到：

$$ Z_t = \frac{d (\frac{u_1 Z_1}{d} + t (\frac{u_2 Z_2}{d} - \frac{u_1 Z_1}{d}))}{u_1 + s(u_2 - u_1)} $$

$$ = \frac{u_1 Z_1 + t (u_2 Z_2 - u_1 Z_1)}{u_1 + s (u_2 - u_1)} \tag{8} $$

式（6）带入式（8）：

$$ Z_1 + t (Z_2 - Z_1) = \frac{u_1 Z_1 + t (u_2 Z_2 - u_1 Z_1)}{u_1 + s (u_2 - u_1)} \tag{9} $$

通过简化这个式子，我们可以得到一维平面中的插值参数 $s$ 与二维线段中的插值参数 $t$ 以及二维线段端点深度值的关系：

![27_t_s](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/27_t_s.jpeg)

也就是：

$$ t = \frac{s Z_1}{s Z_1 + (1-s) Z_2} \tag{10} $$

将式（10）带入式（6），得到 $Z_t$ 通过一维平面中的插值参数 $s$ 正确插值的结果了：

$$
Z_t = Z_1 + t (Z_2 - Z_1) \\
= Z_1 + \frac{s Z_1}{s Z_1 + (1-s) Z_2} (Z_2 - Z_1) \tag{11}
$$

上式（11）可以简化成：

$$
Z_t = \frac{1}{\frac{1}{Z_1} + s (\frac{1}{Z_2} - \frac{1}{Z_1})} \tag{12}
$$

通过最终的式子（12）可以得出： **一维平面上点 $c$ 的深度值可以通过在 $\frac{1}{Z_1}$ 和 $\frac{1}{Z_1}$ 之间进行线性插值，然后计算插值结果的导数来正确的推导出来** 。

对于其它的属性值我们也可以使用这种策略来正确的插值。对于某个属性 $I$ ：

$$ I_t = (1-t) I_1 + t I_2 = I_1 + t (I_2 - I_1) $$

将式（10）带入上式，可以得到：

$$ I_t = I_1 + \frac{s Z_1}{s Z_1 + (1-s) Z_2} (I_2 - I_1) $$

重新排列为：

![28_t_s_1](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/28_t_s_1.jpeg)

也就是：

$$
I_t = \frac{\frac{I_1}{Z_1} + s (\frac{I_2}{Z_2} - \frac{I_1}{Z_1})}{\frac{1}{Z_1} + s (\frac{1}{Z_2} - \frac{1}{Z_1})}
$$

可以看到上式中的分母项也就是式子（12）中的分母项，所以最终的表达式为：

$$
I_t = \frac{\frac{I_1}{Z_1} + s (\frac{I_2}{Z_2} - \frac{I_1}{Z_1})}{\frac{1}{Z_t}}
$$

**总结** ：一维平面上的点 $c$ 的准确属性值 $I_t$ 可以通过在 $\frac{I_1}{Z_1}$ 和 $\frac{I_2}{Z_2}$ 之间进行线性插值，然后将插值结果除以 $\frac{1}{Z_t}$ 来正确的推导出来，其中 $\frac{1}{Z_t}$ 可以通过在屏幕空间中的线性插值得到，如式（12）所示。

**现在将此结论扩展到重心坐标插值** ：

对于三维场景中的三角形图元 $ABC$ ，通过投影变换投影到屏幕空间后的三角形图元为 $abc$ ，在光栅化的过程中，根据屏幕空间像素的坐标以及三角形图元 $abc$ 的顶点坐标计算出的重心坐标为 $\alpha , \beta , \gamma$ 。那么对于某一个属性 $I$ ，像素坐标上得到的正确插值结果为：

$$
I_{\text{pixel}} = \frac{\alpha \frac{I_A}{Z_A} + \beta \frac{I_B}{Z_B} + \gamma \frac{I_C}{Z_C}}{\alpha \frac{1}{Z_A} + \beta \frac{1}{Z_B} + \gamma \frac{1}{Z_C}}
$$

## 13 纹理过滤（Texture Filtering）

为了区分屏幕上的像素和纹理中的像素，我们将纹理的像素称为 **纹素（Texel）** 。

在对屏幕上每个像素进行着色的过程中，我们使用重心坐标对此像素覆盖的三角形图元的三个顶点进行线性插值来确定此像素在纹理内的纹理坐标，该纹理坐标不可能完美的落在纹理的纹素网格上。而且纹理是要贴（映射）到三维物体表面上的，由于纹理表面相对于摄像机可能处于任意距离和方向，屏幕上的一个像素通常不会对应一个纹素。

当使用纹理过滤时，所使用的纹理通常会被 **放大（Magnification）** 或 **缩小（Minification）** 。假设一个方形纹理映射到场景中的一个方形表面，对于场景中的一个透视摄像机，当方形表面距离摄像机很近时，每个纹素比屏幕像素要大，此时屏幕上多个像素映射到同一个纹素上，这是就需要将纹素相应的放大，也就是 **纹理放大（Texture Magnification）** ；当方形表面距离摄像机很远时，纹素比屏幕像素要小，此时屏幕上一个像素覆盖多个纹素，一个像素的最终值会从其覆盖的多个纹素的值计算而出，这就是 **纹理缩小（Texture Minification）** 。

下图展示了纹理放大与纹理缩小，从左到右表示的是离摄像机的位置从近到远，灰色格子表示的是纹理上一个个的纹素，蓝色四边形表示的是屏幕上的一个像素映射到纹理上覆盖的纹素范围：

![29_texture_mag_min](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/29_texture_mag_min.jpeg)

### 13.1 纹理过滤的方法

#### 13.1.1 最邻近插值（Nearest-neighbor Interpolation）

最邻近插值使用离像素重心最近的纹素颜色作为像素颜色。下图展示了最邻近插值的结果：

![30_texture_filtering_nearest](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/30_texture_filtering_nearest.jpeg)

这种方法粗暴简单，但是会产生大量的伪影（Artifact）：在纹理放大时会出现纹理“像素化”，这在有些像素风游戏中比较常用；在纹理缩小时会出现混叠和闪烁。这种方法在纹理放大时速度很快，只需要对纹理坐标四舍五入取整，但是在纹理缩小时，由于内存访问跨度变得任意大，往往会比 Mipmap 效率更低。

#### 13.1.2 双线性过滤（Bilinear Filtering）

在使用双线性过滤时，采样的是离像素重心最近的 4 个纹素（2x2），并根据距离通过两次线性插值来计算最终的像素颜色。下图展示了双线性过滤的结果：

![31_texture_filtering_bilinear](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/31_texture_filtering_bilinear.jpeg)

具体的算法如下图所示：

![32_bilinear_interpolation](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/32_bilinear_interpolation.jpeg)

首先做两次水平方向上的插值计算出 $u_0$ 和 $u_1$ 值：

$$
u_0 = lerp(s, u_{00}, u_{10}) \\
u_1 = lerp(s, u_{01}, u_{11})
$$

最后再根据 $u_0$ 和 $u_1$ 值在垂直方向上插值计算出红色采样点的值：

$$
u_{sample} = lerp(t, u_0, u_1)
$$

双线性过滤消除了纹理放大中的“像素化”的问题，因为现在的像素颜色从一个纹素到相邻的纹素呈现出平滑的渐变，而不是越过纹素边界时突然的跳变。双线性过滤在纹理放大时很常见，在纹理缩小时，通常与 Mipmap 结合使用。

#### 13.1.3 双三次插值（Bicubic Interpolation）

双三次插值时一种更复杂的插值方法，它考虑了更大范围内的纹素（通常是 4x4 的纹素网格，共 16 个纹素），并通过三次函数来计算插值结果，从而在图像放大和缩小时提供更平滑的过渡和更少的伪影。

#### 13.1.4 Mipmap

在纹理缩小时，屏幕上一个像素可以覆盖多个纹素，在远离摄像机的位置，屏幕上一个像素甚至会覆盖整个纹理，此时像素的颜色需要考虑到这些所有被覆盖的纹素的颜色。对于这个像素来说，使用更多的采样点来采样所有被覆盖的纹素并结合它们的值来确定像素颜色是一种对性能消耗极大的操作，而 Mipmap 技术通过预过滤纹理并将其存储为从大到小的一系列尺寸，每一级的尺寸都是上一级的一半，直至单个像素，在像素采样的过程中，根据像素覆盖的纹素的大小来决定其采样哪个尺寸的纹理，这样通过一次采样就可以得到像素覆盖的整个纹素区域的颜色平均值。

Mipmap 的特点：

- 速度快；
- 采样的结果是近似、不准确的；
- 只能做近似正方形的范围查询；
- 额外只增加了原图 $1/3$ 的存储占用

原分辨率为 128x128 的纹理，生成出从 0 到 7 的 8 个层级的 Mipmap，每一层级的分辨率都是上一层级的一半。 对于一个分辨率为 $N \times N$ 的纹理， Mipmap 的层级一共是 $log2(N) + 1$ ，下图展示了一个生成 Mipmap 的例子：

![33_mipmap](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/33_mipmap.jpeg)

**采样纹理时怎么确定应该采样哪一级的 Mipmap** ：

计算出当前像素的纹理坐标 $uv$ 在屏幕坐标 $x$ 和 $y$ 方向上的梯度变化，也就是纹理坐标在这两个方向上的变化率，取最大的变化率用以计算采样的 Mipmap 层级。如下图所示：

![34_cal_mipmap_level](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/34_cal_mipmap_level.jpeg)

用公式表示为：

$$
L = max(\sqrt{(\frac{du}{dx})^2 + (\frac{dv}{dx})^2}, \sqrt{(\frac{du}{dy})^2 + (\frac{dv}{dy})^2}) \\
D = log_{2} L
$$

用 glsl 代码可以表示为：

```glsl
const vec4 MipColors[8] = vec4[8]
(
    vec4(1.0, 0.0, 0.0, 1.0),
    vec4(0.0, 1.0, 0.0, 1.0),
    vec4(0.0, 0.0, 1.0, 1.0),
    vec4(1.0, 0.0, 1.0, 1.0),
    vec4(0.0, 1.0, 1.0, 1.0),
    vec4(1.0, 1.0, 0.0, 1.0),
    vec4(1.0, 1.0, 1.0, 1.0),
    vec4(0.0, 0.0, 0.0, 1.0)
);

void main()
{
    vec2 uv = fs_in.UV0;

    // Texture size in pixels
    ivec2 size = textureSize(uBaseMap, 0);
    // Texcoord
    vec2 texcoords = uv * size;
    // Gradient of texcoords in the x direction
    vec2 dx = dFdx(texcoords);
    // Gradient of texcoords in the y direction
    vec2 dy = dFdy(texcoords);
    // Use the maximum square of the gradients in the x and y directions to compute the mip map level
    float L = max(dot(dx, dx), dot(dy, dy));
    // Max mip level
    float maxLevel = log2(max(size.x, size.y));
    // Mipmap level, clamp to [0, maxLevel]
    float D = clamp(0.5 * log2(L), 0.0, maxLevel); // log2(sqrt(L)) = log2((L)^0.5) = 0.5 * log2(L)
    // Rounded to nearest integer level
    int mipLevel = int(D);

    FragColor = MipColors[mipLevel];
}
```

其中 $log2(sqrt(L))$ 的计算简化为 $0.5 * log2(L)$ 。

最终显示的结果如下：

![35_mip_debug](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/35_mip_debug.jpeg)

#### 13.1.5 三线性过滤(Trilinear Filtering)

当纹理采样从一个 Mipmap 层级切换到下一个层级时，采样的结果会出现突然而非常明显的变化，三线性过滤通过对两个最接近的 Mipmap 层级（ `D` 和 `D + 1` ）进行双线性过滤查找，然后根据计算出的连续的层级 D 来进行线性插值去解决这个问题。具体来说，三线性过滤的步骤如下：

- 对较高质量的 Mipmap 层级进行双线性过滤采样；
- 对下一层级的 Mipmap 进行双线性过滤采样；
- 使用连续的 Mipmap 层级 D（也就是不再四舍五入取整） 对上述两个结果进行线性插值，得到最终的纹理颜色；

![36_trilinear](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/36_trilinear.jpeg)

连续 Mipmap 层级的可视化：

![37_continuous_mipmap_level](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/37_continuous_mipmap_level.jpeg)

代码如下：

```glsl
const vec4 MipColors[8] = vec4[8]
(
    vec4(1.0, 0.0, 0.0, 1.0),
    vec4(0.0, 1.0, 0.0, 1.0),
    vec4(0.0, 0.0, 1.0, 1.0),
    vec4(1.0, 0.0, 1.0, 1.0),
    vec4(0.0, 1.0, 1.0, 1.0),
    vec4(1.0, 1.0, 0.0, 1.0),
    vec4(1.0, 1.0, 1.0, 1.0),
    vec4(0.0, 0.0, 0.0, 1.0)
);

void main()
{
    vec2 uv = fs_in.UV0;

    // Texture size in pixels
    ivec2 size = textureSize(uBaseMap, 0);
    // Texcoord
    vec2 texcoords = uv * size;
    // Gradient of texcoords in the x direction
    vec2 dx = dFdx(texcoords);
    // Gradient of texcoords in the y direction
    vec2 dy = dFdy(texcoords);
    // Use the maximum square of the gradients in the x and y directions to compute the mip map level
    float L = max(dot(dx, dx), dot(dy, dy));
    // Max mip level
    float maxLevel = log2(max(size.x, size.y));
    // Mipmap level, clamp to [0, maxLevel]
    float D = clamp(0.5 * log2(L), 0.0, maxLevel); // log2(sqrt(L)) = 0.5 * log2(L)
    // Rounded to nearest integer level
    int mipLevel = int(D);

    // Mipmap level D+1
    int D1 = min(mipLevel + 1, int(maxLevel));
    vec4 mipDColor = MipColors[mipLevel];
    vec4 mipD1Color = MipColors[D1];
    // Linear interpolation based on continuous D value
    vec4 color = mix(mipDColor, mipD1Color, max(0.0, D - mipLevel));

    FragColor = color;
}
```

#### 13.1.6 各向异性过滤（Anisotropic Filtering）

当一个表面相对于摄像机呈斜角时，此时屏幕上的一个像素映射到纹理上的区域不再近似为方形，如下图所示：

![38_anisotropic_filtering_0](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/38_anisotropic_filtering_0.jpeg)

在 **各向同性（Isotropic）** 的 Mipmap 技术中，每个层级的降采样会同时在每个轴上减半分辨率。因此，当出现上图中的情况时，会使用更大的 $y$ 轴方向上覆盖的纹素数量用以计算出更低分辨率的 Mipmap 采样层级，但此时在 $x$ 轴方向上像素覆盖的纹素其实很小，这也就使 $x$ 轴被错误的“模糊”了，最终导致模糊的现象。

使用各向异性过滤的 Mipmap 时，分辨率为 $256 \times 256$ 的纹理不仅会被降采样到 $128 \times 128$ ，还会被降采样到其他非方形分辨率，例如 $256 \times 128$ 和 $32 \times 128$ 。当纹理映射的图像频率在每个纹理轴上不同步时，可以查找这些各向异性降采样的图像。这样，一个轴不会因为另一个轴的屏幕频率而模糊，同时仍然避免了混叠。下图是各向同性 Mipmap 和各向异性 Mipmap 图像存储的一个对比示例：

![40_anisotropic_filtering_2](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/40_anisotropic_filtering_2.jpg)
![39_anisotropic_filtering_1](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/39_anisotropic_filtering_1.jpeg)

## 14 几何的基本表示方法

几何分为 **显式几何（Explicit Geometry）** 和 **隐式几何（Implicit Geometry）** 。

### 14.1 隐式几何

隐式几何不告诉你每个点的具体坐标，而是描述每个点满足的关系，也就是几何的函数表达式：

![41_implicit_geometry](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/41_implicit_geometry.jpeg)

优点：

- 通过一个函数就可以表示，简洁
- 对于几何内外，到几何表面的距离，查询简单
- 隐式表面很容易做和光线的求交
- 用于简单的形状，精确的描述
- 易于处理拓扑结构的变化

缺点：

- 难以对复杂形状进行建模

### 14.2 隐式几何的表示方法

#### 14.2.1 代数曲面（Algebraic Surfaces）

![42_algebraic_surfaces](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/42_algebraic_surfaces.jpeg)

#### 14.2.2 构造立体几何（Constructive Solid Geometry, CSG）

构造立体几何指的是可以对各种不同的几何做布尔运算，如下图对两个几何体的并（Union）、交（Intersection）和差（Difference）运算：

![43_constructive_solid_geometry](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/43_constructive_solid_geometry.jpeg)

#### 14.2.3 距离函数（Distance Function）

距离函数并不直接表示几何表面，而是通过描述空间中任意一点到几何表面的距离来表示几何。最常见的一种距离函数是 **符号距离函数（Signed Distance Function, SDF）** ，对于空间中的任意一点， SDF 定义的是此点到最近的几何表面点的最小距离，最小距离小于 0 ，表示点在几何内部，等于 0 表示在几何表面，大于 0 表示在几何外。

可以通过距离函数来得到几何形体混合（Blend）的效果：

![44_distance_function](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/44_distance_function.jpeg)

#### 14.2.4 水平集方法（Level Set Method, LSM）

水平集的表示方法与距离函数类似，也是表示到集合表面的最小距离，但不像距离函数那样对空间中的每一个点都有一种严格的数学定义，而是将空间划分为一个个的区域，每个区域描述了距离几何表面的距离。和采样纹理一样，通过对每个区域值的插值可以得到距离为 0 的坐标，也就是几何表面。这种方法特别适合在笛卡尔坐标系下使用，将空间划分为一个个的小立方体。

![45_level_set](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/45_level_set.jpeg)

#### 14.2.5 分型几何（Fractals）

分型几何是指许多自相似的形体最终所组成的几何形体。

![46_fractals](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/46_fractals.jpeg)

### 14.3 显式几何

显式几何直接给出几何表面上所有的点，或者可以通过映射关系直接得到。

![47_explicit_geometry](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/47_explicit_geometry.jpeg)

### 14.4 显式几何的表示方法

#### 14.4.1 点云（Point Cloud）

通过表面所有点的列表来表示几何，可以轻松表示任何类型的几何形状。在渲染时通常转换为多边形网格。

![48_point_cloud](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/48_point_cloud.jpeg)

#### 14.4.2 多边形网格（Polygon Mesh）

通过顶点和多边形（通常是三角形或四方形）来表示几何。在图形学中，多边形网格是最常用的一种表示几何的方式。

![49_polygon_mesh](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/49_polygon_mesh.jpeg)

下面是一个使用 Wavefront Object（.obj） 文件定义的几何：

![50_obj](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/50_obj.jpeg)

其中， `v` 开头的行是表示顶点坐标， `vn` 开头的行是对应顶点的法线向量， `vt` 开头的行是对应顶点的纹理坐标， `f` 开头的行是定义的每个三角形面。

## 15 ⻉塞尔曲线（Bézier Curve）

⻉塞尔曲线是一种显式的几何表示方法，它是通过一组控制点来定义的，这些控制点决定了⻉塞尔曲线的形状。下图展示了通过 $p_0 , p_1, p_2, p_3$ 这 4 个点控制的贝塞尔曲线：

![51_bezier_curves](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/51_bezier_curves.jpeg)

贝塞尔曲线的性质：

- 必定经过起点与终点；
- 必定与起始线段和终止线段相切；
- 对控制点做仿射变换，仿射变换后的所有控制点控制的贝塞尔曲线与变换前保持不变。 **需要注意的是对控制点做投影变换后新的贝塞尔曲线会发生改变** ；
- 凸包（Convex Hull）性质，曲线一定不会超出所有控制点构成的多边形范围；

### 15.1 de Casteljau 算法（de Casteljau Algorithm）

de Casteljau 算法是一种用于计算贝塞尔曲线的递归算法，算法核心是线性插值与递归，由法国数学家 Paul de Casteljau 在 1959 年发明。

#### 15.1.1 二次贝塞尔曲线（Quadratic Bézier Curve）

下面以 3 个点控制的二次贝塞尔曲线为例来描述一下此算法：

![52_quadratic_bezier](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/52_quadratic_bezier.jpeg)

如上图所示，有 3 个控制点 $\mathbf{b}_0, \mathbf{b}_1, \mathbf{b}_2$ ：

- 首先，选定一个线性插值参数 $t \in [0, 1]$ ；
- 在 $\mathbf{b}_0 \mathbf{b}_1$ 线段上使用 $t$ 值进行线性插值得到点 $\mathbf{b}_0^1$ ；
- 在 $\mathbf{b}_1 \mathbf{b}_2$ 线段上使用 $t$ 值进行线性插值得到点 $\mathbf{b}_1^1$ ；
- 再在前 2 次插值得到的 2 个新的点 $\mathbf{b}_0^1, \mathbf{b}_1^1$ 组成的线段上使用 $t$ 值进行线性插值得到点 $\mathbf{b}_0^2$ ；
- 对所有的 $t \in [0, 1]$ 都重复上述过程，将所有得到的点 $\mathbf{b}_0^2$ 连起来就得到了这个贝塞尔曲线；

用代数表达式可以表示为：

$$ \mathbf{b}_0^1(t)=(1-t)\mathbf{b}_0 + t\mathbf{b}_1 \tag{13}$$

$$ \mathbf{b}_1^1(t)=(1-t)\mathbf{b}_1 + t\mathbf{b}_2 \tag{14}$$

$$ \mathbf{b}_0^2(t)=(1-t)\mathbf{b}_0^1 + t\mathbf{b}_1^1 \tag{15}$$

将式（13）和式（14）带入式（15）可以得到：

$$ \mathbf{b}_0^2(t) = (1-t)^2 \mathbf{b}_0 + 2t(1-t)\mathbf{b}_1 + t^2 \mathbf{b}_2 $$

#### 15.1.2 三次贝塞尔曲线（Cubic Bézier Curve）

4 个点控制的三次贝塞尔曲线的计算过程和二次贝塞尔曲线类似，只是比二次贝塞尔曲线多一次递归计算：

![53_cubic_bezier_curve](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/53_cubic_bezier_curve.jpeg)

#### 15.1.3 贝塞尔曲线的一般化代数表示

对于 $n$ 次贝塞尔曲线，它的代数表达式可以写成：

<!-- $$\sum_{i=0}^{n} {\mathbf{b}}_i B_{i}^{n}(t)$$ -->
$$ b^n(t) = b_0^n (t) = \sum_{i=0}^{n} b_i B_i^n(t)$$

其中， $t \in [0,1]$ 。表达式中的 $B_i^n(t)$ 又称作 $n$ 次的 **伯恩斯坦多项式（Bernstein Polynomial）** ：

$$
B_i^n(t)=
\begin{pmatrix}
    n \\
    i
\end{pmatrix}
t^i
(1-t)^{n-i}  \ , \ i = 0, 1, \cdots , n
$$

其中， $t^0 = 1, (1-t)^0 = 1$ 。伯恩斯坦多项式中的：

$$
\begin{pmatrix} n \\ i \end{pmatrix}
$$

叫做 **二项式系数（Binomial Coefficient）** ，它表示的是：

$$
\begin{pmatrix}
    n \\
    i
\end{pmatrix}=
\frac{n!}{i!(n-i)!}
$$

### 15.2 分段贝塞尔曲线（Piecewise Bézier Curve）

分段贝塞尔曲线是一种通过连接多个低次贝塞尔曲线来构建更复杂更高次的贝塞尔曲线的方法，通常使用三次贝塞尔曲线来构建。每个贝塞尔曲线片段由一组控制点定义，并且这些片段可以无缝的连接在一起，形成一条连续的曲线。

![54_piecewise_bezier_curve](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/54_piecewise_bezier_curve.jpeg)

分段贝塞尔曲线的关键在于如何在每个贝塞尔曲线片段之间进行平滑连接。根据不同的条件，分段贝塞尔曲线之间的连接有着不同的连续性定义。

#### 15.2.1 $C^0$ 连续（Continuity）

相邻贝塞尔曲线片段公用同一个点（前一个贝塞尔曲线片段的终点等于相邻贝塞尔曲线片段的起点），这种连续性称为 $C^0$ 连续。

![55_c0](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/55_c0.jpeg)

#### 15.2.2 $C^1$ 连续（Continuity）

相邻贝塞尔曲线片段在连接点处的切线方向相同（前一个贝塞尔曲线片段的终点和上一个点组成的线段与相邻贝塞尔曲线片段的起点与下一个点组成的线段共线且方向相反，并且长度相等），这种连续性称为 $C^1$ 连续。

![56_c1](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/56_c1.jpeg)

### 贝塞尔曲面（Bézier Surface）

![57_bezier_surface](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/57_bezier_surface.jpeg)

以 $4\times 4$ 个控制点的贝塞尔曲面为例：

![58_4x4_bezier_surface](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/58_4x4_bezier_surface.jpeg)

- 使用第一个插值参数 $u \in [0,1]$ 通过 de Casteljau 算法计算出 4 行贝塞尔曲线（每行有 4 个点，通过这 4 个点计算出这一行的贝塞尔曲线，一共 $4 \times 4$ 个控制点 ）上的 4 个点（上图中蓝色的 4 个点，灰色是 4 行计算出的贝塞尔曲线）；
- 使用第二个插值参数 $v \in [0,1]$ 通过  de Casteljau 算法计算出这 4 个蓝色点控制的贝塞尔曲线（上图中蓝色的贝塞尔曲线）；
- 遍历所有的 $u, v$ 值就可以成功得到一个贝塞尔曲面；

这篇文章[Making MathBox](https://acko.net/blog/making-mathbox/)有很生动的演示动画，对于理解很有帮助。

## 16 网格细分（Mesh Subdivision）

网格细分将粗糙的多边形网格细化，生成更为平滑和细致的曲面。网格细分的主要目的是通过反复细分多边形表面，使其逐步接近光滑的极限曲面。网格细分主要做的是：

- 首先，生成更多的网格（也就是生成更多的顶点）
- 然后，调整顶点的位置使曲面更光滑

![59_mesh_subdivision](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/59_mesh_subdivision.jpeg)

### 16.1 Loop 细分（Loop Subdivision）

Loop 细分用于三角形网格。

#### 16.1.1 生成更多的网格

对于每个三角形网格，取这个三角形网格三条边的中点作为新的顶点，连接这些新的顶点，将一个三角形网格分割成四个三角形网格：

![60_loop_subdivision_0](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/60_loop_subdivision_0.jpeg)

#### 16.1.2 更新新生成的顶点和老的顶点的位置

对于新生成的顶点，它一定在三角形网格的一条边上，这条边被两个相邻的三角形网格所共享。新生成的白色点位置的更新算法如下图所示：

![60_loop_subdivision_1](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/60_loop_subdivision_1.jpeg)

对于下图中白色的老的顶点：

![60_loop_subdivision_2](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/60_loop_subdivision_2.jpeg)

其中 $n$ 表示的是这个 **顶点的度（Vertex Degree）** ，顶点的度表示的是这个顶点连接的边的数量。

### 16.2 Catmull-Clark 细分（Catmull-Clark Subdivision）

Catmull-Clark 细分可以用于三角形网格或四边形网格。对于四边形网格，称为 **四边形面（Quad Face）** ，而对于三角形网格，称其为 **非四边形面（Non-quad Face）** 。顶点的度不为 4 的顶点，叫做 **奇异点（Extraordinary Vertex）** 。

#### 16.2.1 生成更多的网格

对于每一个网格，取它每条边上的中点和这个网格的中点，将这些点连接起来。

![61_catmull_clark_subdivision_1](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/61_catmull_clark_subdivision_1.jpeg)
![61_catmull_clark_subdivision_2](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/61_catmull_clark_subdivision_2.jpeg)

#### 16.2.2 更新新生成的顶点和老的顶点的位置

![61_catmull_clark_subdivision_0](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/61_catmull_clark_subdivision_0.jpeg)

## 17 网格简化（Mesh Simplification）

网格简化要做的是在尽量保持整体形状外观和几何特征的前提下，尽量减少网格的数量。

![62_mesh_simplification](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/62_mesh_simplification.jpeg)

### 17.1 边坍缩（Edge Collapsing）

找到一条边，把这条边的二个顶点压缩到一起成为一个顶点，这就叫做边坍缩。

![63_edge_collapsing](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/63_edge_collapsing.jpg)

在选择坍缩哪条边时，我们希望尽量保持原来的整体形状，所以需要要找到哪些边是不重要的，可以把它坍缩到一块；那些边是重要的，不能坍缩它。

通过使用 **二次误差（Quadric Error）** 来评估应该优先坍缩哪条边。**所谓二次误差，就是一个顶点和原本几个和它都有关系的面的距离的平方和** ：

![64_quadric_error](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/64_quadric_error.jpg)

边坍缩算法的步骤：

- 对每一条边打一个分数，分数就是他坍缩后的二次误差度量；
- 对分数最低（误差最小）的边做边坍缩（使用 Dijkstra 的 **最短路径算法** ）；
- 重新执行第一步直到完成整个模型的边坍缩；

## 18 光线追踪（Ray Tracing）

### 18.1 光线（Light Ray）的基本定义

- 光线沿直线传播；
- 光线之间不会碰撞；
- 光线从光源发出，在场景中不断碰撞，最终进入人的眼睛（通过光线传播的可逆性，也可以理解为眼睛发射感知的光线，通过空间中物体的反射最终打到光源上去）；

### 18.2 光线投射算法（Ray Casting）

从相机位置通过屏幕上每个像素投射一条光线（Eye Ray），计算这条光线与场景中物体（光线第一个碰撞到的物体）的交点，根据物体属性和光照情况对该像素做着色。

![65_ray_casting](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/65_ray_casting.jpg)

### 18.3 Whitted-Style 光线追踪（Whitted-Style Ray Tracing）

Whitted-Style 光线追踪是一种 **递归的（Recursive）** 光线追踪算法，由计算机图形学先驱 Turner Whitted 在 1980 年提出。这种方法扩展了基本的光线投射算法（Ray Casting），能够更真实地模拟光线与物体的交互作用，如反射、折射和阴影。

Whitted-Style 光线追踪算法：

- **光线与物体相交** ：从相机位置通过每个像素投射一条初始光线，计算这条光线与场景中物体（光线第一个碰撞到的物体）的交点，对于每个交点，计算其光照；
- **计算阴影** ：从此交点向光源投射一条光线，计算这条光线是否被其它物体遮挡，以确定该点是否在阴影中；
- **反射** ：如果相交的物体是反射性的，则生成一条反射光线，并递归的追踪这条反射光线；
- **折射** ：如果相交的物体是透明或折射性的，则生成一条折射光线，递归的追踪这条折射光线；

![66_whitted_style_ray_tracing](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/66_whitted_style_ray_tracing.jpg)

Whitted-Style 光线追踪算法如上图所示，其中 **主光线（Primary Ray）** 表示从相机位置投射出的光线， **次级光线（Secondary Ray）** 表示经过反射或折射投射出的光线， **阴影光线（Shadow Ray）** 表示投射向光源的光线。

### 18.4 光线与表面求交（Ray-Surface Intersection）

#### 18.4.1 光线方程（Ray Equation）

一条光线通过一个起点和光线的方向来定义。

![67_ray_equation](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/67_ray_equation.jpg)

光线经过时间 $t$ 到达点 $\mathbf{P}$ ，可以表示为：

$$
\mathbf{P} = \mathbf{o} + t \mathbf{d}
$$

其中 $\mathbf{o}$ 是光线的起点， $\mathbf{d}$ 是光线的方向。

#### 18.4.2 光线与隐式几何求交

对于隐式几何，直接使用光线方程带入隐式几何表达式，求解新的表达式时间 $t$ 的 **实数** 和 **正数** 解。下面是一个光线与隐式几何表示的球体求交例子：

![68_ray_intersection_sphere](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/68_ray_intersection_sphere.jpg)

#### 18.4.3 光线与显示几何求交

1. 光线与平面（Plane）求交

    平面通过平面上的一个点和这个平面的法线向量来定义：

    ![69_define_plane](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/69_define_plane.jpg)

    对于上图中的由点 $\mathbf{p}'$ 与平面法线向量 $\mathbf{N}$ 定义的一个平面，如果一个点 $\mathbf{p}$ 满足： $(\mathbf{p} - \mathbf{p}') \cdot \mathbf{N} = 0$ ，则表示这个点 $\mathbf{p}$ 在平面上。那么只需要将点 $\mathbf{p}$ 通过光线方程的形式表示并带入此表达式求解时间 $t$ 的实数和正数解，即可计算光线与平面的交点：

    ![70_ray_plane](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/70_ray_plane.jpg)

1. 光线与三角形（Triangle）求交

    三角形在一个平面上，首先求光线与三角形所在平面的交点，然后再计算这个交点是否在三角形内。

#### 18.4.4 Möller Trumbore 算法（Möller Trumbore Algorithm）

Möller Trumbore 算法是一种利用三角形重心坐标来计算光线与三角形相交的算法，由 Tomas Möller 和 Ben Trumbore 在 1997 年提出。

![71_moller_trumbore_algorithm](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/71_moller_trumbore_algorithm.jpg)

### 18.5 轴对齐包围盒（Axis-Aligned Bounding Box, AABB）

AABB 是一个长方体盒子，它的每条边有坐标轴对齐， **一个 AABB 可以看作是三对无限大平面的交集** 。

![72_aabb](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/72_aabb.jpg)

场景中的每个物体都有一个自己的 AABB ，如果光线没有进入物体的 AABB ，那么表示光线和物体一定没有交点。

#### 光线与 AABB 求交的核心思想

- 当光线进入了所有三对无限大平面之间，则表示光线进入了此 AABB ；
- 当光线离开了任意一对无限大平面，则表示光线离开了此 AABB ；

下面以一个二维的 AABB 的例子来说明怎么计算光线与 AABB 求交：

![73_intersection_2d_aabb](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/73_intersection_2d_aabb.jpg)

- 首先计算光线进入以及离开与 $x$ 轴对齐的一对平面之间的时间 $t_{min1}$ 和 $t_{max1}$ ；
- 再计算光线进入以及离开与 $y$ 轴对齐的一对平面之间的时间 $t_{min2}$ 和 $t_{max2}$ ；
- 那么光线进入这个二维 AABB 的时间是 $t_{enter} = max(t_{min1}, t_{min2})$ ，离开这个二维 AABB 的时间是 $t_{exit} = max(t_{max1}, t_{max2})$ ；
- 当 $t_{enter} < t_{exit}$ 时，则表示光线与这个二维 AABB 有交点；
- 特殊情况：
    - 当 $t_{exit} < 0$ 时，此时表明这个二维 AABB 在光线的后方，这种情况下光线与 AABB 没有交点；
    - 当 $t_{exit} \geq 0 \And t_{enter} < 0$ 时，此时表明光线的起点在 AABB 内部，这种情况下光线与 AABB 有交点；

三维情况下也是同样的计算，多一对与 $z$ 轴对齐的的平面的计算。

通过 **轴对齐（Axis-Aligned）** 可以简化光线与平面的求交计算：

![74_axis_aligned](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/74_axis_aligned.jpg)

### 18.6 使用 AABB 加速光线追踪

使用 AABB 加速光线追踪的核心思想是：在做光线追踪之前，预处理场景，构建加速结构。

#### 18.6.1 均匀网格空间划分（Uniform Spatial Partitions (Grids) ）

![75_uniform_spatial_partitions](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/75_uniform_spatial_partitions.jpg)

对于一个场景，均匀网格空间划分算法预处理场景的步骤：

- 首先计算出包含场景内所有物体的 AABB ；
- 将场景的 AABB 划分为均匀的三维网格，每个网格称为 **体素（Voxel）** ；
- 遍历场景中的每个物体，根据物体的 AABB 将其分配到相应的体素中，一个物体可能会落在多个体素中；

在光线追踪的过程中，光线沿着其方向逐步穿过体素，当穿过包含物体的体素时，才计算光线与此体素所包含所有物体的求交。

![76_usp_intersection](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/76_usp_intersection.jpg)

当三维网格划分的太少时，加速效果不明显，而当三维网格划分的太多时，也会影响光线追踪的效率。一般情况下三维网格的划分数量为：

```cpp
#cells = C * #objs
```

其中，在三维空间中， $C \approx 27$ 。

#### 18.6.2 空间划分（Spatial Partitions）

不同于均匀网格空间划分，空间划分是将场景划分为最优的不均匀的三维网格。

##### 八叉树（Oct-Tree）

递归地将空间横竖二次划分为八个子空间（体素），直到达到一定的深度或每个体素包含的物体数量低于某个阈值。光线追踪时，从根节点开始，递归地测试光线与子空间的相交情况。

![77_oct_tree](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/77_oct_tree.jpg)

##### KD树（KD-Tree）

KD树是一种二叉树结构，它递归地将空间划分为两个子空间（体素），每次沿着 $x$ 、 $y$ 、 $z$ 轴循环划分，尽量让划分更均匀，直到达到一定的深度或每个体素包含的物体数量低于某个阈值，叶子节点包含实际的几何物体。光线追踪时，从根节点开始，从根节点（Root Node）开始往下递归计算光线与子节点的相交情况，如果是和叶子节点相交则与叶子节点下所有物体求交点。

![78_kd_tree](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/78_kd_tree.jpg)

KD树的数据结构：

- **中间节点（Internal Node）**
    - 当前节点时沿着拿个轴划分的
    - 划分的位置
    - 对于中间节点，会存储它的子节点
    - 中间节点不存储场景中物体的信息
- **叶子节点（Leaf Node）**
    - 存储场景中物体的列表

#### 18.6.3 层次包围盒（Bounding Volume Hierarchy, BVH）

**层次包围盒不再从空间上做划分，而是通过场景内的物体来做划分** 。

![79_bvh](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/79_bvh.jpg)

层次包围盒划分过程：

- 首先计算出包含场景内所有物体的 AABB ；
- 然后，递归的将场景内所有的物体划分成两个子集，并分别计算这两个子集的 AABB ；
- 直到达到一定的深度或每个子集包含的物体数量低于某个阈值；
- 将子集物体的列表存储在叶子节点；

划分的一些技巧：

- 每次沿着 $x$ 、 $y$ 、 $z$ 轴循环划分，尽量让划分更均匀；
- 选择 AABB 中范围最大的轴去划分；
- 在中值物体的位置拆分节点，使两个子集的物体数量尽量接近；

层次包围盒的数据结构：

- **中间节点（Internal Node）**
    - 存储 AABB
    - 存储子节点指针
- **叶子节点（Leaf Node）**
    - 存储 AABB
    - 存储场景中物体的列表

层次包围盒的遍历算法的伪代码：

![80_bvh_traversal](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/80_bvh_traversal.jpeg)

```cpp
Intersect(Ray ray, BVH node)
{
    if (ray misses node.bbox) return;
    if (node is a leaf node)
        test intersection with all objs;
    return closest intersection;
    hit1 = Intersect(ray, node.child1);
    hit2 = Intersect(ray, node.child2);
    return the closer of hit1, hit2;
}
```

#### 18.6.4 空间划分（Spatial Partitions）与物体划分（Object Partitions）的比较

空间划分(例如：KD-Tree)

- 将空间划分为非重叠区域；
- 一个物体可能在多个区域；

物体划分(例如：BVH)

- 将所有物体划分为不相交的子集；
- 每个子集的包围盒可能相交；

## 19 辐射度量学（Basic radiometry）

### 19.1 基本的物理量

| 中文名 | 英文名 | 单位 | 符号 |
|:------:|:--------:|:-------:|:-------:|
| 辐射能量  |  Radiant Energy   | 焦耳（J）  |  $Q$  |
| 辐射通量  |  Radiant Flux   | 瓦特（W）  |  $\phi$  |
| 辐射强度  |  Radiant Intensity   | 坎德拉（[Candela](https://en.wikipedia.org/wiki/Candela), W / sr）  |  $I$  |
| 辐射照度  |  Irradiance   | W / m^2  |  $E$  |
| 辐射亮度  |  Radiance   | W / (sr⋅m^2)  |  $L$  |

下面来具体依次对它们进行介绍。

### 19.2 辐射能量（Radiant Energy）和辐射通量（Radiant Flux）

**光源辐射出来的总能量，叫做辐射能量** 。它以焦耳（J）为单位进行测量，并用符号 $Q$ 表示。

**每单位时间的辐射能量，叫做辐射通量** 。它的单位是瓦特（W），使用符号 $\phi$ 表示：

$$ \phi = \frac{\mathrm{d}Q}{\mathrm{d}t} $$

### 19.3 立体角（Solid Angle）

要介绍辐射强度，就要先介绍 **立体角（Solid Angle）** 。什么是立体角呢？先引用一段维基百科的描述：

```text
立体角，是一个物体对特定点的三维空间的角度，是平面角在三维空间中的类比。它描述的是站在某一点的观察者测量到的物体大小的尺度。
```

我们先来看看二维空间的平面角是怎么定义的，平面角指的是这个角对应的圆上的弧长和圆半径的比值。看下图这个平面角的例子：

![81_angle](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/81_angle.jpeg)

在上面这个二维圆中，角度 $\theta$ 对应在圆上的弧长为 $l$ ，圆的半径为 $r$ ，那么这个角用弧度可以表示为：

$$ \theta = \frac{l}{r} $$

整个圆的弧度则为： $\frac{2\pi r}{r} = 2\pi$ 。

而立体角就是三维空间中的 “平面角 $\theta$ ” ，立体角是三维空间中 **球面上的面积与球体半径的平方的比值** ，立体角使用符号 $\Omega$ 来表示，它的单位是 **球面度（[Steradian](https://en.wikipedia.org/wiki/Steradian), sr）** 。下图是一个立体角的示例：

![82_solid_angle](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/82_solid_angle.jpeg)

上图中的立体角可以表示为：

$$ \Omega = \frac{A}{r^2} $$

整个球对应的立体角是： $\frac{4\pi r^2}{r^2} = 4\pi$ 。

### 19.4 微分立体角（Differential Solid Angles）

假设球的半径为 $r$ ，对于使用球面坐标定义的一个角度 $(r, \phi, \theta)$ ，如下图所示：

![83_spherical_coordinates](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/83_spherical_coordinates.png)

其中 $\phi$ 是方位角，范围是 $[0, 2\pi]$ ， $\theta$ 是极角，范围是 $[0, \pi]$ 。对于方位角和极角上的极小差分变化 $\phi + \mathrm{d}\phi$ 和 $\theta + \mathrm{d}\theta$ 所对应的立体角微元怎么表示呢？

下图展示了这种极小差分变化下对应的球面上的面积微元 $\mathrm{d}A$ 的推导计算过程：

![84_solid_angle_dA](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/84_solid_angle_dA.jpg)

需要注意的是， $a$ 和 $b$ 对应的是球面上的弧长，所以我们可以利用平面角的定义来计算它们。

那么这种极小差分变化对应的立体角微元 $\mathrm{d}\omega$ 也就可以表示为：

$$
\mathrm{d}\omega = \frac{\mathrm{d}A}{r^2} = \frac{r^2 \sin{\theta} \mathrm{d}\theta \mathrm{d}\phi}{r^2} = \sin{\theta} \mathrm{d}\theta \mathrm{d}\phi
$$

这种极小差分变化对应的立体角微元也叫做 **单位立体角（Unit Solid Angle）** 。

对于整个球的立体角也就是将所有的单位立体角积分起来：

$$
\Omega = \int_{S^2} \mathrm{d}\omega = \int_{0}^{2\pi} \int_{0}^{\pi} \sin{\theta} \mathrm{d}\theta \mathrm{d}\phi = 4\pi
$$

### 19.5 辐射强度（Radiant Intensity）

**每单位立体角的辐射通量，叫做辐射强度** 。它的单位是 `W / sr` ，使用符号 $I$ 表示：

$$ I(\omega) = \frac{\mathrm{d}\phi}{\mathrm{d}\omega} $$

对于三维空间中一个均匀向四周辐射能量的点光源，它在任意一个方向上单位立体角的能量就是它的辐射强度。这个点光源在单位时间内辐射的能量（即辐射通量）为 $\phi$ ， $\phi$ 可以表示为对整个球上所有方向上的辐射强度做积分，也就是：

$$ \phi = \int_{S^2} I \mathrm{d}\omega $$

我们知道，将一个球面上所有单位立体角积分起来的结果就是这个球的立体角，也就是 $4\pi$ ，所以：

$$ \phi = 4\pi I $$

那么对于任意方向上的辐射强度 $I$ ，可以表示为：

$$ I = \frac{\phi}{4\pi} $$

### 19.6 辐射照度（Irradiance）

**每单位面积所接收到的辐射通量** ，叫做辐射照度。它的单位是 `W / m^2` ，使用符号 $E$ 来表示：

$$ E = \frac{\mathrm{d}\phi}{dA} $$

需要注意的是，在实际计算接收到辐射照度时，需要考虑到表面法线与光线入射方向的夹角大小：

![85_irradiance_cos_theta](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/85_irradiance_cos_theta.jpg)

可以看到，当接收的表面与光线垂直时，此时接收到所有的能量，但当光线斜着照射到表面时，表面接收到的能量就会变少。通常情况下，接收到的辐射照度与表面与光线夹角的余弦乘比例关系，也就是：

$$ E = \frac{\phi}{A} \cos{\theta} $$

通过辐射照度，还可以解释为什么距离光源越远接收到的能量就越少，也就是辐射照度是有衰减的：

![86_irradiance_falloff](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/86_irradiance_falloff.jpeg)

如上图所示，因为辐射通量 $\phi$ 是固定的，而随着距离光源越来越远，单位面积上所接收到的能量也就越来越少。

### 19.7 辐射亮度（Radiance）

**每单位立体角每单位垂直面积的辐射通量** ，叫做辐射亮度。它的单位是 `W / (sr⋅m^2)` ，使用符号 $L$ 来表示：

$$
L = \frac{\mathrm{d}^2 \phi}{\mathrm{d}\omega \mathrm{d}A \cos{\theta}}
$$

**辐射亮度同时指定了光的方向与照射到的表面所接收到的能量** 。在这里有一个细微的差别，在辐射照度中定义的 **每单位面积** ，而在辐射亮度中，为了更好的使其称为描述一条光线传播中的能量，且在传播过程当中大小不随方向改变，所以在定义中关于接收面积的部分是 **每单位垂直面积** ，这一点的不同也正解释了上式中分母中的 $\cos{\theta}$ ，如下图所示：

![87_radiance_cos_theta](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/87_radiance_cos_theta.png)

上图中， $\mathrm{d}A$ 是辐射照度中所定义的单位面积，而 $\mathrm{d}A^{\perp}$ 才是辐射亮度中所定义的单位垂直面积，$\mathrm{d}A^{\perp}$ 可以通过 $\mathrm{d}A$ 与光线与表面法线之间夹角的余弦来表示，也就是： $\mathrm{d}A^{\perp} = \mathrm{d}A \cos{\theta}$ 。

### 19.8 辐射照度（Irradiance）与辐射亮度（Radiance）

再次复习一下：

- 辐射亮度（Radiance）描述的是 **单位立体角单位垂直面积的辐射通量**
- 而辐射照度（Irradiance）描述的是 **单位面积所接收到的辐射通量** ，也就是**单位面积从整个单位半球 $\Omega^{+}$ 接收到的所有辐射通量**

![88_indicent_radiance](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/88_indicent_radiance.jpg)

那么，对于上图所示的 **从某个单位立体角入射的辐射亮度（Radiance）** 可以表示为：

$$
L_i(p,\omega) = \frac{\mathrm{d}E(p)}{\mathrm{d}\omega \cos{\theta}}
$$

其中， $p$ 是物体表面上的某一点， $\omega$ 是入射方向的单位立体角， $\theta$ 是入射方向与物体表面法线的夹角。

也就是说， **从某个单位立体角入射的辐射照度（Irradiance）** 是：

$$
\mathrm{d}E(p) = L_i(p,\omega) \cos{\theta} \mathrm{d}\omega
$$

对单位半球 $\Omega^{+}$ 所有入射单位立体角积分就可以得到单位面积所接收到的辐射通量，也就是辐射照度（Irradiance） $E(p)$ ：

$$
E(p) = \int_{\Omega^{+}} L_i(p,\omega) \cos{\theta} \mathrm{d}\omega
$$

## 20 BRDF 、反射方程（The Reflection Equation）和渲染方程（The Rendering Equation）

### 20.1 BRDF

BRDF 全称是 Bidirectional Reflectance Distribution Function ，中文叫做： **双向反射分布函数** 。 BRDF 描述的是物体表面上的一个点，接收了某一个方向上的辐射照度（Irradiance）后，将其反射到另外一个方向上的比例。也就是 **反射光的辐射亮度（Radiance）与某个方向入射光的辐射照度（Irradiance）的比值** 。BRDF 表示为：

$$
f_r(\omega_{i} \to \omega_{r}) = \frac{\mathrm{d}L_r(\omega_{r})}{\mathrm{d}E_i(\omega_{i})} = \frac{\mathrm{d}L_r(\omega_{r})}{L_i(\omega_{i}) \cos{\theta_{i}} \mathrm{d}\omega_{i}}
$$

### 20.2 反射方程（The Reflection Equation）

对于某一个入射方向 $\omega_{i}$ 的入射光，对反射方向 $\omega_{r}$ 的辐射亮度（Radiance）的贡献为：

$$
\mathrm{d}L_r{\omega_{r}} = f_r(\omega_{i} \to \omega_{r}) L_i(\omega_{i}) \cos{\theta_{i}} \mathrm{d}\omega_{i}
$$

那么，将整个单位半球 $\Omega^{+}$ 上所有入射光对反射方向 $\omega_{r}$ 的辐射亮度（Radiance）的贡献积分出来就是最终反射光的辐射亮度（Radiance），这就是反射方程。对于物体表面某个点 $p$ ，反射方向为 $\omega_{r}$ ，反射方程可以表示为：

$$
L_r(p,\omega_{r}) = \int_{\Omega^{+}} f_r(p,\omega_{i} \to \omega_{r}) L_i(p,\omega_{i}) \cos{\theta_{i}} \mathrm{d}\omega_{i}
$$

### 20.3 渲染方程（The Rendering Equation）

![89_the_rendering_equation](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/89_the_rendering_equation.jpeg)

## 21 概率论基础与蒙特卡罗积分

### 21.1 随机变量

随机变量，使用 $X$ 来表示，随机变量描述的是 **随机事件中潜在值的分布** 。

随机变量有两种类型： **离散型（Discrete）** 和 **连续型（Continuous）** 。当随机变量是离散型时，随机的结果只能取一系列确切的值（随机的结果是 **有限的** 或 **可数无限的** 多个值），例如投骰子（ 1 点 到 6 点）、抛硬币（正面或反面）等。当随机变量是连续型时，随机的结果落在一个连续区间范围内，无法列举所有的可能性，例如温度测量。

### 21.2 概率密度函数（Probability Distribution Function, PDF）

概率密度函数，简称为 PDF ，是用于描述连续型随机变量的概率分布的函数。 PDF 的值表示的是连续型随机变量在某个确定的取值点附近的可能性大小。对于一个连续型随机变量 $X$ ，概率密度函数 $p(x)$ 满足：

- 非负性：对于所有的 $x$ ，有 $p(x) \geq 0$
- 归一化：整个实数集上的积分等于 1，即 $\int_{-\infty}^{+\infty} p(x) \mathrm{d}x = 1$
- 对于任意两个实数 $a$ 和 $b$ ，随机变量 $X$ 落在区间 $[a,b]$ 内的概率可以通过积分计算： $P(a \leq X \leq b) = \int_{a}^{b} p(x) \mathrm{d}x$

对于 **均匀分布** 的连续型随机变量，它的概率密度函数是一个 **常数** 。

### 21.3 期望值（Expected Value）

如果从随机变量中反复抽取样本，所获得的平均值称为 **期望值** ，使用符号 $E$ 来表示。具体来说， **期望值是对随机变量取值的加权平均，权重是每个可能值的概率** ，它提供了一个关于随机变量取值的集中趋势的度量。

对于一个随机变量 $X$ ，它从具有 $n$ 个离散值 $x_i$ 的样本空间中随机抽取结果，样本空间中每个离散值的概率密度是 $p_i$ 。那么，期望值 $E[X]$ 可以表示为：

$$
E[X] = \sum_{i=1}^{n} x_i p_i
$$

对于 **连续型随机变量** $X$ ：

$$ X \sim p(x) $$

它的概率密度函数满足：

$$ p(x) \geq 0 $$

并且将连续区间内的概率密度积分起来结果为 1 ：

$$ \int p(x)\mathrm{d}x = 1 $$

连续型随机变量 $X$ 的期望值 $E[X]$ 是：

$$
E[X] = \int x p(x) \mathrm{d}x
$$

随机变量 $X$ 的函数 $Y$ 也是一个随机变量：

$$ X \sim p(x) $$

$$ Y = f(X) $$

函数 $Y$ 的期望可以表示为：

$$ E[Y] = E[f(X)] = \int f(x)p(x) \mathrm{d}x $$

### 21.5 蒙特卡罗积分

蒙特卡罗积分是通过 **使用蒙特卡罗法来估计积分值** 。如下图所示，要计算函数 $f(x)$ 在区间 $[a,b]$ 上的积分值：

![90_continuous_function](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/90_continuous_function.jpg)

我们知道求函数在某个区间的积分也就是求函数在此区间的面积，也就是上图中蓝色的部分,蒙特卡罗积分的主要思想就是：在区间 $[a,b]$ 之间，随机采样 $N$ 个样本，将每次的样本计算的函数面积加起来并平均，得到的就是此函数在此区间所求积分的估计值。原理的示意图如下：

![91_monte_carlo_integration](/assets/images/2024/2024-07-23-TheEssenceOfGAMES101/91_monte_carlo_integration.jpg)

蒙特卡罗积分的一般公式可以表示为：

$$
F_N = \frac{1}{N} \sum_{i=1}^{N} \frac{f(X_i)}{p(X_i)}
$$

其中，随机变量 $X_i$ 的概率密度函数为 $p(x)$ ：

$$
X_i \sim p(x)
$$

在这里证明蒙特卡罗积分的正确性：

$$
\begin{align*}
E[F_N] &= E\Big[\frac{1}{N} \sum_{i=1}^{N} \frac{f(X_i)}{p(X_i)}\Big]\\
&= \frac{1}{N} \sum_{i=1}^{N} E\Big[\frac{f(X_i)}{p(X_i)}\Big]\\
&= \frac{1}{N} \sum_{i=1}^{N} \int_{a}^{b} \frac{f(x)}{p(x)} p(x) \mathrm{d}x\\
&= \frac{1}{N} \sum_{i=1}^{N} \int_{a}^{b} f(x)\mathrm{d}x\\
&= \int_{a}^{b} f(x)\mathrm{d}x
\end{align*}
$$

最后举一个例子加强理解。使用均匀分布的随机变量样本的蒙特卡罗积分来估算 $f(x)$ 在 $[a,b]$ 区间上的积分值。

首先概率密度函数在区间 $[a,b]$ 之间的积分为 1 ，也就是：

$$
\int_{a}^{b} p(x) \mathrm{d}x = 1
$$

而 **对于一个均匀分布的随机变量，它的概率密度函数是一个常数** ，如下图所示：

$$
X_i \sim p(x) = C (constant)
$$

所以：

$$
\int_{a}^{b} C \mathrm{d}x = 1
$$

也就得到这个常数是：

$$
C = \frac{1}{b-a}
$$

最终使用蒙特卡罗积分估算 $f(x)$ 在区间 $[a,b]$ 的积分值为：

$$
F_N = \frac{1}{N} \sum_{i=1}^{N} \frac{f(X_i)}{p(X_i)} = \frac{1}{N} \sum_{i=1}^{N} \frac{f(X_i)}{\frac{1}{b-a}} = \frac{b-a}{N} \sum_{i=1}^{N} f(X_i)
$$
