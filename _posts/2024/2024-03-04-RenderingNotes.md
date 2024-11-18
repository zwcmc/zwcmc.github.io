---
layout: post
title:  "渲染学习笔记"
date:   2024-03-04 16:16:00 +800
category: Rendering
---

- [1. 法线变换矩阵（Normal Matrix）](#1-法线变换矩阵normal-matrix)
- [2. Alpha To Coverage](#2-alpha-to-coverage)
  - [2.1. 参考](#21-参考)
- [3. 通过一个四元数存储基础法线向量 `(0,0,1)` 、基础切线向量 `(1,0,0)` 与基础副切线向量 `(0,1,0)`](#3-通过一个四元数存储基础法线向量-001-基础切线向量-100-与基础副切线向量-010)
- [4. 3D 空间中的平面方程](#4-3d-空间中的平面方程)
  - [4.1. 平面方程表达式](#41-平面方程表达式)
  - [4.2. 如何确定平面方程](#42-如何确定平面方程)
  - [4.3. 平面方程的应用](#43-平面方程的应用)

## 1. 法线变换矩阵（Normal Matrix）

法线向量是定义在模型空间中的，而光照计算一般情况下都是在世界空间进行的，所以就需要一个矩阵来把法线向量从模型空间转换到世界空间，这个矩阵就叫做**法线变化矩阵** 。

首先，法线向量它是一个向量，它并不是和模型顶点坐标一样代表的是空间中的一个位置，所以法线向量的齐次坐标为 `0`，这意味着平移变换对法线向量不会有任何影响。

顶点坐标是通过模型矩阵 $\mathbf{M}$ 变换到世界空间的，那么使用忽略掉平移部分的模型矩阵左上角 `3x3` 部分的矩阵（只考虑对法线向量的旋转和缩放）来变换法线向量是不是就可以了？如果模型矩阵执行了**统一缩放( `X` , `Y` 和 `Z` 是相同的缩放值)**，直接使用模型矩阵左上角3x3部分来变换法线向量是没有问题的，此时变换只改变了法线向量的长度，而没有改变它的方向，所以只需要再做一下归一化操作就可以得到正确的计算光照所需的法线向量。

但如果模型矩阵执行了非统一缩放，这个时候就会出问题，变换后的法线向量会不再垂直于表面，下图展示了模型矩阵进行了非统一缩放变换时，直接使用模型矩阵导致的错误结果：

![00_ununiform_scaling](/assets/images/2024/2024-03-04-RenderingNotes/00_ununiform_scaling.png)

所以需要推导出正确的法线变换矩阵。

![01_normal_triangle](/assets/images/2024/2024-03-04-RenderingNotes/01_normal_triangle.webp)

如上图所示，垂直于法线向量 $\mathbf{N}$ 的切线向量 $\mathbf{T}$ 可以由三角形边上的 2 个点 $P_{1}$ 和 $P_{2}$ 计算得到：

$$ \mathbf{T} = \mathbf{P_2} - \mathbf{P_1} $$

考虑到切线向量是一个方向向量，而方向向量也可以用齐次坐标来表示(此时的 `w` 分量为0)，所以上面等式2边可以同时乘以 4x4 的模型矩阵 $\mathbf{M}$：

$$ \mathbf{M} * \mathbf{T} = \mathbf{M} * (\mathbf{P_2} - \mathbf{P_1}) $$

所以：

$$ \mathbf{M} * \mathbf{T} = \mathbf{M} * \mathbf{P_2} - \mathbf{M} * \mathbf{P_1} $$

可以看到等式右边就是 2 个在模型空间的点 $\mathbf{P_2}$ 和 $\mathbf{P_1}$ 分别变换到世界空间后再继续相减，得到世界空间下的切线，而等式左边是用模型矩阵直接对模型空间下的切线向量(用齐次坐标来表示，此时 `w` 分量为 0)做矩阵变换，所以可以得到 **直接使用模型矩阵左上角 `3x3` 部分来变换切线向量是没有问题的** 。

知道这个以后，现在继续来推导正确的法线变换矩阵：

![02_normal_triangle_transformed](/assets/images/2024/2024-03-04-RenderingNotes/02_normal_triangle_transformed.webp)

因为法线向量和切线向量应该是相互垂直的，1个垂直于表面，1个平行于表面，所以变换到世界空间后的法线向量 $\mathbf{N}^{\prime}$ 和切线向量 $\mathbf{T}^{\prime}$ 也应该互相垂直：

$$ \mathbf{N}^{\prime} \cdot \mathbf{T}^{\prime} = 0 $$

下面使用 $\mathbf{M}$ 来表示 4x4 模型矩阵左上角 `3x3` 部分的矩阵，通过上面的推导可以知道这个矩阵可以正确的将模型空间的切线向量 $\mathbf{T}$ 变换到世界空间(后面涉及的变换矩阵都是 `3x3` 的矩阵，因为切线向量和法线向量都是方向向量，不受平移变换影响)，假设法线向量的变换矩阵是 $\mathbf{G}$，所以上面的等式可以变换为：

$$ \mathbf{N}^{\prime} \cdot \mathbf{T}^{\prime} = (\mathbf{G}\mathbf{N}) \cdot (\mathbf{M}\mathbf{T}) = 0 $$

而2个行向量的点乘可以通过把其中一个向量转置成列向量从而把点乘转换成矩阵的乘法( 2 个 `vec3` 向量点乘，转换为 1 个 `3x1` 的矩阵和 1 个 `1x3` 的矩阵相乘)，即：

$$ (\mathbf{G}\mathbf{N}) \cdot (\mathbf{M}\mathbf{T}) = (\mathbf{G}\mathbf{N})^{T} * (\mathbf{M}\mathbf{T}) $$

又因为转置的性质：**两个矩阵相乘后进行转置，等于每个矩阵转置后再相乘，且顺序相反** 。所以：

$$
(\mathbf{G}\mathbf{N}) \cdot (\mathbf{M}\mathbf{T}) = (\mathbf{G}\mathbf{N})^{T} * (\mathbf{M}\mathbf{T}) = \mathbf{N}^{T}\mathbf{G}^{T}\mathbf{M}\mathbf{T} = 0
$$

又因为 $\mathbf{N} \cdot \mathbf{T} = 0$，同样的把其中1个行向量转置成列向量可以把点乘转换成矩阵的乘法，也就是 $\mathbf{N}^{T} * \mathbf{T} = 0$，所以上式中只有满足：

$$
\mathbf{G}^{T} * \mathbf{M} = \mathbf{I}
$$

才能得到：

$$
\mathbf{N}^{\prime} \cdot \mathbf{T}^{\prime} = \mathbf{N} \cdot \mathbf{T} = 0
$$

所以，要求的法线变换矩阵是： **模型矩阵左上角 `3x3` 部分的子矩阵的逆的转置** ，表达式为：

$$ \mathbf{G} = (\mathbf{M^{-1}})^{T} $$

当模型矩阵只包含旋转和平移时，此时左上角 `3x3` 部分只包含旋转变换，而 **旋转矩阵是正交矩阵** ，正交矩阵有一个性质就是： **正交矩阵的逆等于它的转置** 。此时满足：

$$ \mathbf{M^{-1}} = \mathbf{M}^{T} $$

所以此时：

$$ \mathbf{G} = \mathbf{M} $$

也就是说，**在模型矩阵只有旋转和平移变换时，可以直接使用模型矩阵左上角 `3x3` 部分的子矩阵来正确变换法线向量** 。

而当模型矩阵包含旋转、统一缩放和平移变换时，可以把统一缩放系数 $\lambda$ 提取出来:

$$
\mathbf{G} = (\mathbf{M}^{-1})^{T} =
((\begin{bmatrix}
    \lambda & 0 & 0 \\
    0 & \lambda & 0 \\
    0 & 0 & \lambda
\end{bmatrix}
\mathbf{M}_{rotation})^{-1})^{T}
$$

上面的公式可以简化为：

$$
((\lambda \mathbf{M}_{rotation})^{-1})^{T}
$$

最终表示为：

$$
\frac{1}{\lambda} ((\mathbf{M}_{rotation})^{-1})^{T}
$$

这时 $\mathbf{M}_{rotation}$ 是只包含旋转变换的正交矩阵，所以此时：

$$
\mathbf{G} = \frac{1}{\lambda} \mathbf{M}_{rotation}
$$

可以知道 **当模型变换中包含旋转，统一缩放和平移时，此时也可以直接使用模型矩阵左上角 `3x3` 部分的子矩阵来正确变换法线向量** ，变换后的统一缩放系数影响的是法线向量的长度，而在计算光照时，需要的仅仅的法线向量的方向，也就是最终都会对法线向量进行归一化计算，它的长度变化并不需要关心。

**总结** ：

- 当模型变换中包含旋转变换，统一变换，平移变换时，可以直接使用模型矩阵左上角 `3x3` 部分的子矩阵来变换法线向量
- 而当变换中包含了非统一缩放，此时就必须求解模型矩阵左上角 `3x3` 部分的子矩阵的逆的转置来得到法线变换矩阵

**参考** ：

- [The Normal Matrix](http://www.lighthouse3d.com/tutorials/glsl-12-tutorial/the-normal-matrix/)
- [Unity Shaders Book Chapter 4](https://candycat1992.github.io/unity_shaders_book/unity_shaders_book_chapter_4.pdf)

## 2. Alpha To Coverage

Alpha Test 是一种常见的渲染技术，它的原理也很简单：在片元着色器中， `alpha` 值低于设定阈值的像素将直接被丢弃。下图是一个使用 Alpha Test 渲染一个灌木丛的例子：

![03_a2c_bushes](/assets/images/2024/2024-03-04-RenderingNotes/03_a2c_bushes.webp)

可以看到，因为 Alpha Test 是根据像素的 `alpha` 值直接丢弃掉像素，所以边缘会有很明显的锯齿走样。而 Alpha To Coverage （一般称作 A2C 或 ATOC ， 以下简称为 A2C ）就是一种通过 MSAA 来对 Alpha Test 做抗锯齿的技术。

首先简单介绍一下 MSAA ，在光栅化阶段， MSAA 对每个像素采样 N 个子样本（例如 MSAA 2x2 就是每个像素 4 个样本），在每个子样本上测试三角形图元的对这个像素的覆盖（Coverage）情况，只有在三角形图元至少覆盖了一个子采样点的情况下才会执行一次片元着色器（这意味着启用 MSAA 时，片元着色器的成本不会显著增加，这是 MSAA 相对于 SSAA 的主要优势），最后将被覆盖的子采样点的颜色更新为此次片元着色器输出的结果，如下图所示：

![05_msaa_subsamples](/assets/images/2024/2024-03-04-RenderingNotes/05_msaa_subsamples.webp)

而像素最终的输出颜色，会通过一个 1 像素宽的 box filter 进行过滤（也就是对像素内所有的子采样点进行平均）输出。

另外，在开启 MSAA 的情况下，因为每个子采样点可能被不同的三角形图元覆盖，所以渲染目标的每个像素需要支持存储 N 个子采样点的颜色；对于深度测试也是一样的，因为每个子采样点都需要进行深度测试，这意味着在开启 N 个样本的 MSAA 的情况下，深度缓冲区的大小是没有开启 MSAA 的情况下的 N 倍。

而在开启 A2C 之后，片元着色器输出的 `alpha` 值会改变 MSAA 中被覆盖子样本的颜色更新比例。举个例子，在开启 MSAA 的情况下， 假设 1 个三角形片元覆盖了一个像素的 4 个子样本，那么这 4 个子样本都会存储片元着色器的颜色输出；但如果开启了 A2C 并且片元着色器输出颜色的 alpha 为 0.5 ，那么只有一半的子样本会存储片元着色器的颜色输出，最终只有 1/2 的子样本对像素的颜色产生贡献，也就达到了类似 0.5 透明度的 Alpha Blend 效果。

那么这 4 个样本的颜色将被更新为片元着色器的颜色输出结果，而当开启 A2C 之后，且片元着色器的输出颜色的 alpha 为 0.5 时，那么只有一半可能的子样本的颜色会被更新，最终像素的输出颜色就是原本输出颜色的 1/2 ，这样达到了类似 Alpha Blend 的效果：

![04_alpha_to_coverage](/assets/images/2024/2024-03-04-RenderingNotes/04_alpha_to_coverage.jpeg)

从上图可以看到，虽然 A2C 使 Alpha Test 的边缘锯齿效果减轻了，但是最终的效果却很糟糕，这是因为 **A2C 对 MSAA 子样本覆盖率的修改只会生效一次，并且修改后的子样本覆盖率会影响后续所有三角形图元对此像素的覆盖率** 。还是上面的例子，假如 1 个三角形片元覆盖了一个像素的 4 个子样本，在开启 4xMSAA 并且此时片元着色器输出的颜色的 alpha 为 0.5 的情况下，此时片元着色器输出的颜色只会写入 2 个被覆盖的子样本，但是，后续所有三角形片元对此像素的子样本覆盖率都将变成 50% ，也就是说此时每个像素的通过 A2C 达到的透明度是离散的（例如，一个像素的 A2C 覆盖率是 0.5 ，相邻的像素的 A2C 覆盖率跳变为 0.25），这也就是最终效果如此糟糕的原因。

一种解决方案是通过将片元着色器输出颜色 alpha 值锐化（ Sharpen ）为单个像素的宽度，从而产生与 MSAA 在实际几何边缘上生成的类似抗锯齿边缘：

```hlsl
col.a = (col.a - _Cutoff) / max(fwidth(col.a), 0.0001) + 0.5;
```

其中 `fwidth(col.a)` 函数计算的是 alpha 值每个像素的变化，等同于 `abs(ddx(col.a)) + abs(ddy(col.a))` 。通过将片元着色器输出的 alpha 锐化，可以大大改善 A2C 的效果：

![06_a2c_sharpened](/assets/images/2024/2024-03-04-RenderingNotes/06_a2c_sharpened.jpg)

最后补充一个更影响 Alpha Test 的效果的问题，那就是 Mipmap，默认的 Mipmap 生成方式是对相邻的 4 个像素就平均，也就是一个 Box Filter，对于颜色来说，这样做没问题，但是对于 Alpha Test 中的 alpha 值来说却是有问题的，因为 Alpha Test 是根据 alpha 值来做 clip 的，当 alpha 值因为 Mipmap 被平均的越来越小时，会导致在某个 Mipmap 层级的 alpha 值因为小于 Alpha Test 的阈值而大片像素直接全部被 clip 掉：

![07_alpha_test_mipmap](/assets/images/2024/2024-03-04-RenderingNotes/07_alpha_test_mipmap.gif)

解决此问题的办法也很简单，就是在生成 Mipmap 时对 alpha 值单独进行处理，2010 年，Ignacio Castaño 在 [[4]](#a2c_ref1) 提出了一种重新计算每个 Mip 的 alpha 值的方法，在 Unity 中，可以通过纹理导入设置中的 `Preserve Coverage` 选项来使用此方法。

### 2.1. 参考

- [1] [Anti-aliased Alpha Test: The Esoteric Alpha To Coverage](https://bgolus.medium.com/anti-aliased-alpha-test-the-esoteric-alpha-to-coverage-8b177335ae4f)
- [2] [A Quick Overview of MSAA](https://therealmjp.github.io/posts/msaa-overview/)
- [3] [UE426终于实现了Alpha to Coverage](https://zhuanlan.zhihu.com/p/388513281)
- <a id="a2c_ref1"></a>[4] [Computing Alpha Mipmaps](http://the-witness.net/news/2010/09/computing-alpha-mipmaps/?source=post_page-----8b177335ae4f--------------------------------)

## 3. 通过一个四元数存储基础法线向量 `(0,0,1)` 、基础切线向量 `(1,0,0)` 与基础副切线向量 `(0,1,0)`

今天在看 [flament](https://github.com/google/filament) 源码时，看到一个有意思的算法：通过模型数据中的切线，副切线，法线信息，构建一个表示旋转的 `TBN` 矩阵，将此矩阵转换为四元数表示，最后将此四元数传入着色器中，在着色器中，通过此四元数对初始法线向量 `(0, 0, 1)` 和初始切线向量 `(1, 0, 0)` 进行旋转，得到模型空间中的法线和切线向量信息。其中，这个四元数的 `w` 分量的符号（sign，正或者负）还代表了切线向量的方向，因为切线空间必须是右手坐标系统（Right-hand coord system），而如果 $(\mathbf{n} \times \mathbf{t}) \cdot \mathbf{b} \leq 0.0$ 则代表此时的切线空间并不是右手坐标系统，需要根据 `w` 分量的符号将切线向量取反。

表示旋转的 `TBN` 矩阵怎么理解呢？在加载模型时得到了切线、法线数据后（副切线可以通过法线和切线叉积得到），将切线（t）、副切线（b）和法线向量（n）构建一个 `TBN` 矩阵：

$$
\text{TBN} =
\begin{bmatrix}
    \mathbf{t}_x & \mathbf{b}_x & \mathbf{n}_x \\
    \mathbf{t}_y & \mathbf{b}_y & \mathbf{n}_y \\
    \mathbf{t}_z & \mathbf{b}_z & \mathbf{n}_z
\end{bmatrix}
$$

将此矩阵视作对基础切线向量 `(1, 0, 0)`、基础副切线向量 `(0, 1, 0)` 和基础法线向量 `(0, 0, 1)` 的旋转矩阵，分别对基础向量应用此旋转矩阵后，得到模型空间的切线向量、副切线向量与法线向量。举个例子，对于基础法线向量 `(0, 0, 1)`，对其应用这个旋转矩阵后，得到：

$$
\begin{bmatrix}
    \mathbf{t}_x & \mathbf{b}_x & \mathbf{n}_x \\
    \mathbf{t}_y & \mathbf{b}_y & \mathbf{n}_y \\
    \mathbf{t}_z & \mathbf{b}_z & \mathbf{n}_z
\end{bmatrix}
\begin{bmatrix}
    0 \\
    0 \\
    1
\end{bmatrix}=
\begin{bmatrix}
    \mathbf{n}_x \\
    \mathbf{n}_y \\
    \mathbf{n}_z
\end{bmatrix}
$$

可以看到基础法线向量与构建的 `TBN` 矩阵相乘也就得到了模型空间中的法线向量。切线和副切线同理。

构建 `TBN` 矩阵的主要的算法在 `packTangentFrame` 方法中：

```cpp
constexpr QUATERNION<T> MATH_PURE positive(const QUATERNION<T>& q)
{
    return q.w < 0 ? -q : q;
}

/**
* Packs the tangent frame represented by the specified matrix into a quaternion.
* Reflection is preserved by encoding it as the sign of the w component in the
* resulting quaternion. Since -0 cannot always be represented on the GPU, this
* function computes a bias to ensure values are always either positive or negative,
* never 0. The bias is computed based on the specified storageSize, which defaults
* to 2 bytes, making the resulting quaternion suitable for storage into an SNORM16
* vector.
*/
template<typename T>
constexpr TQuaternion<T> TMat33<T>::packTangentFrame(const TMat33<T>& m, size_t storageSize) noexcept
{
    TQuaternion<T> q = TMat33<T>{ m[0], cross(m[2], m[0]), m[2] }.toQuaternion();
    q = positive(normalize(q));

    // Ensure w is never 0.0
    // Bias is 2^(nb_bits - 1) - 1
    const T bias = T(1.0) / T((1 << (storageSize * CHAR_BIT - 1)) - 1);
    if (q.w < bias) {
        q.w = bias;

        const T factor = (T)(std::sqrt(1.0 - (double)bias * (double)bias));
        q.xyz *= factor;
    }

    // If there's a reflection ((n x t) . b <= 0), make sure w is negative
    if (dot(cross(m[0], m[2]), m[1]) <= T(0)) {
        q = -q;
    }

    return q;
}
```

其中，第一个 `3x3` 的矩阵 `m` 参数通过模型数据中的切线（tangent）、副切线（bitangent）和法线（normal）构成，返回的四元数传入顶点数据中：

```cpp
quatf q = filament::math::details::TMat33<float>::packTangentFrame({tangent, bitangent, normal});
asset.tangents.push_back(packSnorm16(q.xyzw));
```

第二个参数是为了计算四元数 `w` 分量的偏差，为了保证 `w` 分量不为 `0`，这是因为 GPU 不一定能表示 `-0` 这种数据。 `storageSize` 默认的大小是 `sizeof(int16_t)`。

下面来看看 `packTangentFrame` 函数的具体实现：

```cpp
TQuaternion<T> q = TMat33<T>{ m[0], cross(m[2], m[0]), m[2] }.toQuaternion();
```

首先将 `TBN` 矩阵转换为四元数表示，其中 `m[0]` 是模型中的切线，`m[2]` 是法线，`cross(m[2], m[0])` 是副切线。

```cpp
q = positive(normalize(q));
```

归一化四元数 `q`，并通过 `w` 分量的符号，来决定是否取反四元数 `q`，保证其 `w` 分量为正值，用以后续偏移的计算。需要注意的是： **当四元数是表示旋转时，对其进行取反，`q` 与 `-q` 是等价的，两者表示的都是相同的旋转**。而在这里，`TBN` 矩阵表示的就是一个旋转矩阵，所以 `q` 和 `-q` 是等价的。

```cpp
// Ensure w is never 0.0
// Bias is 2^(nb_bits - 1) - 1
const T bias = T(1.0) / T((1 << (storageSize * CHAR_BIT - 1)) - 1);
if (q.w < bias) {
    q.w = bias;

    const T factor = (T)(std::sqrt(1.0 - (double)bias * (double)bias));
    q.xyz *= factor;
}
```

将 `w` 分量限制在最小正值 `bias`，并保证 `q.xyz` 向量的模还是 `1`，也就是单位向量。

```cpp
// If there's a reflection ((n x t) . b <= 0), make sure w is negative
if (dot(cross(m[0], m[2]), m[1]) <= T(0)) {
    q = -q;
}
```

最后，根据切线，副切线和法线之间的方向关系来确定 `w` 分量的符号。当 `dot(cross(m[0], m[2]), m[1]) <= T(0)` 时，表示构建 `TBN` 的三个向量并不是在右手坐标系统下，此时再次将四元数 `q` 取反，在后续着色器的计算中，通过 `w` 分量的符号来保证 `TBN` 矩阵的正确性。

在着色器中，需要做的是通过这个四元数 `q` 来解析出模型空间的切线、副切线和法线数据，其中 `q` 就是计算好的代表旋转的四元数：

```cpp
/**
 * Extracts the normal vector of the tangent frame encoded in the specified quaternion.
 */
void toTangentFrame(const highp vec4 q, out highp vec3 n) {
    n = vec3( 0.0,  0.0,  1.0) +  // base normal vector (0,0,1)
        vec3( 2.0, -2.0, -2.0) * q.x * q.zwx +
        vec3( 2.0,  2.0, -2.0) * q.y * q.wzy;
}

/**
 * Extracts the normal and tangent vectors of the tangent frame encoded in the
 * specified quaternion.
 */
void toTangentFrame(const highp vec4 q, out highp vec3 n, out highp vec3 t) {
    toTangentFrame(q, n);
    t = vec3( 1.0,  0.0,  0.0) +  // base tangent vector (1,0,0)
        vec3(-2.0,  2.0, -2.0) * q.y * q.yxw +
        vec3(-2.0,  2.0,  2.0) * q.z * q.zwx;
}
```

## 4. 3D 空间中的平面方程

### 4.1. 平面方程表达式

**平面方程** 是描述 3D 空间中一个平面的数学表达式。标准的平面方程形式是：

$$ Ax + By + Cz + D = 0 $$

其中：

- $(A,B,C)$ 是平面的法线向量，它垂直于平面
- $(x,y,z)$ 是平面上的任意一点的坐标
- $D$ 是平面方程的常数项

### 4.2. 如何确定平面方程

1. **已知法线向量和平面上的一个点** ：

    如果知道平面的法线向量 $\mathbf{n} = (A,B,C)$ 和平面上的一个点 $\mathbf{p_0} = (x_0, y_0, z_0)$ ，那么平面方程可以表示为：

    $$ A(x - x_0) + B(y - y_0) + C(z - z_0) = 0 $$

    展开后就是标准形式：

    $$ Ax + By + Cz + D = 0 $$

    其中， $D = -(Ax_0, By_0, Cz_0)$ ，即 $- \mathbf{n} \cdot \mathbf{p_0}$ 。

2. **已知三个不共线的点** ：

    如果知道平面上的三个不共线的点 $\mathbf{p_1} = (x_1, y_1, z_1)$ 、 $\mathbf{p_2} = (x_2, y_2, z_2)$ 、 $\mathbf{p_3} = (x_3, y_3, z_3)$ ，可以通过以下步骤确定平面方程：

    - 计算两个向量： $\mathbf{v_1} = p_2 - p1$ 和 $\mathbf{v_2} = p_3 - p_1$
    - 通过叉积计算平面的法线向量： $\mathbf{n} = \mathbf{v_1} \times \mathbf{v2}$
    - 带入其中一个点（如 $p_1$ ）和法线向量 $\mathbf{n}$ 到平面方程中，得到常数 $D$

    通过这种方向可以计算出某个三角形所在平面的平面方程。

### 4.3. 平面方程的应用

在实际应用中，可以通过将一个点的坐标带入平面方程中，来计算这个点与平面的位置关系：

- 如果 $ Ax + By + Cz + D > 0 $ ，则这个点在平面的正面
- 如果 $ Ax + By + Cz + D < 0 $ ，则这个点在平面的背面
- 如果 $ Ax + By + Cz + D = 0 $ ，则这个点在平面上

在做视锥体剔除时，视锥体可以看作是 6 个平面，通过计算某个点距离分别距离这 6 个平面的距离来判断这个点是否在视锥体内。通过这个方法可以扩展到视锥体与球体、AABB等的求交。
