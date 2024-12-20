---
layout: post
title:  "数学笔记"
date:   2018-08-05 16:16:00 +800
category: Math
---

- [1. 求简单矩阵(2x2，3x3)的逆矩阵](#1-求简单矩阵2x23x3的逆矩阵)
  - [1.1. 矩阵的行列式](#11-矩阵的行列式)
  - [1.2. 伴随矩阵（Adjugate Matrix）](#12-伴随矩阵adjugate-matrix)
- [2. 导数（Derivative）](#2-导数derivative)
  - [2.1. 在图形学中的应用场景](#21-在图形学中的应用场景)
- [3. `dot` 、 `cross` 和 `reflect` 的数学计算公式](#3-dot--cross-和-reflect-的数学计算公式)

## 1. 求简单矩阵(2x2，3x3)的逆矩阵

对于简单的矩阵（`2x2`，`3x3`），可以通过伴随矩阵法来求它的逆矩阵。例如当前有个矩阵 `M`，它的逆矩阵可以表示为：

$$
M^{-1} = \frac{1}{\det(A)} \cdot \text{adj}(M)
$$

### 1.1. 矩阵的行列式

行列式是一个方阵（即行数和列数相等的矩阵）所对应的一个标量。它可以用来判断矩阵是否可逆，以及描述线性变换的某些性质。

行列式为零表示矩阵是奇异的（不可逆），而非零行列式表示矩阵是非奇异的（可逆）。

**行列式的绝对值为 1 的矩阵叫做正交矩阵，正交矩阵乘以它的转置等于单位矩阵，正交矩阵的逆矩阵等于它的转置矩阵**。

对于一个 2x2 的矩阵：

$$
A = \begin{bmatrix}
    a & b \\
    c & d
\end{bmatrix}
$$

其行列式定义为：

$$
\det(A) = ad - bc
$$

对于一个 3x3 的矩阵：

$$
B = \begin{bmatrix}
a & b & c \\
d & e & f \\
g & h & i \\
\end{bmatrix}
$$

其行列式计算公式为：

$$
\det(B) = aei + bfg + cdh - ceg - bdi - afh
$$

### 1.2. 伴随矩阵（Adjugate Matrix）

给定一个 `n x n` 的方阵 `A`，其伴随矩阵（记作 $\text{adj}(A)$ ）是由 `A` 的代数余子式（Cofactor）组成的新矩阵的转置。

具体来说，伴随矩阵的元素 $\text{adj}(A)_{ij}$ 是原矩阵 `A` 的 `(i, j)` 元素的代数余子式 `C`，然后将这些代数余子式组成一个新的矩阵，并对这个矩阵进行转置。

代数余子式 $C_{ij}$ 定义为：

$$
C_{ij} = (-1)^{i + j}M_{ij}
$$

其中， $M_{ij}$ 是原矩阵 `A` 去掉第 `i` 行和第 `j` 列后的子矩阵的行列式。

看下面一个例子，求下面这个矩阵的逆矩阵：

$$
M =
{\begin{bmatrix}
    \Delta{U_{1}} & \Delta{V_{1}} \\
    \Delta{U_{2}} & \Delta{V_{2}}
\end{bmatrix}}
$$

1. 首先，计算此矩阵的行列式 $\det(M)$：

$$
\det(M) = \Delta{U_{1}}\Delta{V_{2}} - \Delta{V_{1}}\Delta{U_{2}}
$$

2. 计算伴随矩阵 $\text{adj}(M)$：
  - $N_{00}$ 等于原矩阵 `M` 去掉第 `0` 行和第 `0` 列后的子矩阵的行列式：也就是 $(-1)^{(0 + 0)}\Delta{V_{2}} = \Delta{V_{2}}$。
  - $N_{01}$ 等于原矩阵 `M` 去掉第 `0` 行和第 `1` 列后的子矩阵的行列式：也就是 $(-1)^{(0 + 1)}\Delta{U_{2}} = -\Delta{U_{2}}$。
  - $N_{10}$ 等于原矩阵 `M` 去掉第 `1` 行和第 `0` 列后的子矩阵的行列式：也就是 $(-1)^{(1 + 0)}\Delta{V_{1}} = -\Delta{V_{1}}$。
  - $N_{11}$ 等于原矩阵 `M` 去掉第 `1` 行和第 `1` 列后的子矩阵的行列式：也就是 $(-1)^{(1 + 1)}\Delta{U_{1}} = \Delta{U_{1}}$。
  - 组成新的矩阵：

  $$
  N =
  \begin{bmatrix}
      \Delta{V_{2}} & -\Delta{U_{2}} \\
      -\Delta{V_{1}} & \Delta{U_{1}}
  \end{bmatrix}
  $$

  - 最后对新的矩阵进行转置：

  $$
  \text{adj}(M) =
  (N)^{T} =
  \begin{bmatrix}
      \Delta{V_{2}} & -\Delta{V_{1}} \\
      -\Delta{U_{2}} & \Delta{U_{1}}
  \end{bmatrix}
  $$

3. 最终矩阵 `M` 的逆矩阵是：

  $$
  M^{-1} =
  \frac{1}{\det(M)}
  \text{adj}(M) =
  \frac{1}{\Delta{U_{1}}\Delta{V_{2}} - \Delta{V_{1}}\Delta{U_{2}}}
  \begin{bmatrix}
      \Delta{V_{2}} & -\Delta{V_{1}} \\
      -\Delta{U_{2}} & \Delta{U_{1}}
  \end{bmatrix}
  $$

## 2. 导数（Derivative）

导数是多元函数中一个重要的概念，它用于描述函数在某一特定变量方向上的变化率。

在图形学（特别是计算机图形学）中，`ddx` 和 `ddy` 通常指的是对图像或纹理坐标进行导数的计算。这些操作在着色器编程中非常常见，尤其是在处理纹理映射、抗锯齿、法线贴图等效果时。这些函数通常在像素着色器（fragment shader）中使用，用于获取某个变量在屏幕空间 x 方向或 y 方向上的变化率。它们的主要作用是帮助实现各种图形效果，确保渲染结果更加平滑和准确。

### 2.1. 在图形学中的应用场景

1. **纹理映射**：在纹理映射过程中，使用 `ddx` 和 `ddy` 可以计算纹理坐标的导数，从而进行各向异性过滤（anisotropic filtering），根据 `uv` 在 `x` 和 `y` 方向的变化率计算纹理映射的 `mipmap` 采样级别，提高纹理的细节表现。
2. **抗锯齿**：在抗锯齿处理中，导数可以用于估计像素覆盖率，从而平滑边缘，减少锯齿现象。
3. **法线贴图**：在法线贴图中，`ddx` 和 `ddy` 可以用于计算法线的变化，从而正确地模拟光照效果。
4. **屏幕空间效果**：如屏幕空间环境光遮蔽（SSAO）和屏幕空间反射（SSR），这些效果通常需要计算屏幕空间中的梯度。

## 3. `dot` 、 `cross` 和 `reflect` 的数学计算公式

1. 点积 `dot` ：

    ```cpp
    inline float dot(const vec3& v0, const vec3& v1)
    {
        return v0.x * v1.x + v0.y * v1.y + v0.z * v1.z;
    }
    ```

2. 叉积 `cross` ：

    ```cpp
    inline vec3 cross(const vec3& v0, const vec3& v1)
    {
        return vec3(v0.y * v1.z - v0.z * v1.y,
                v0.z * v1.x - v0.x * v1.z,
                v0.x * v1.y - v0.y * v1.x);
    }
    ```

3. 反射 `reflect`

    ```cpp
    inline vec3 reflect(const vec3& v, const vec3& n) {
        return v - 2 * dot(v, n) * n;
    }
    ```

    需要注意的是，其中 `v` 是入射向量，指向反射点。 `n` 是反射点的法线向量。
