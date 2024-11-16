---
layout: post
title:  "Temporal Antialiasing"
date:   2024-11-06 6:16:00 +800
category: Rendering
---

- [1. 时间抗锯齿（Temporal Antialiasing, TAA）](#1-时间抗锯齿temporal-antialiasing-taa)
- [2. 核心算法介绍](#2-核心算法介绍)
- [3. 实现 TAA 的关键技术解析](#3-实现-taa-的关键技术解析)
  - [3.1. 抖动偏移（Jitter offset）](#31-抖动偏移jitter-offset)
  - [3.2. 重投影（Reproject）](#32-重投影reproject)
  - [3.3. 验证（Validate）](#33-验证validate)
    - [3.3.1. 拒绝（Rejection）](#331-拒绝rejection)
    - [3.3.2. 修正（Rectification）](#332-修正rectification)
  - [3.4. 样本累积（Accumulate）](#34-样本累积accumulate)
  - [3.5. 采样滤波的选择](#35-采样滤波的选择)
  - [3.6. Mipmap bias](#36-mipmap-bias)
- [4. TAA 效果展示](#4-taa-效果展示)
- [5. 参考](#5-参考)

## 1. 时间抗锯齿（Temporal Antialiasing, TAA）

传统上， **时间抗锯齿（Temporal Antialiasing, TAA）（以下简称为 TAA ）** 是指用于减少 时间锯齿（例如旋转的车轮）的技术，但在实时渲染中， TAA 是指一种 **利用多个帧的数据进行空间抗锯齿** 的技术，所以它也被叫做 **时间均摊超级采样（Temporally-amortized Supersampling）** 。

## 2. 核心算法介绍

我们知道，锯齿走样产生的本质原因是每个像素采样率太低而跟不上原信号变化的频率。理论上，可以通过使用 **超采样（Supersampling）** 增加每个像素的采样率来改善锯齿走样的问题，但对实时渲染来说，每一帧都对每个像素进行超采样的成本太高了，而 TAA 的核心思想是： **将超采样的子像素样本分摊到多个帧上，并通过使用当前帧之前累积的子像素样本来实现超采样** 。

下面通过一张图来详细说明一下 TAA 算法的核心流程：

![00_amortize_supersampling](/assets/images/2024/2024-11-06-TemporalAntialiasing/00_amortize_supersampling.jpeg)

1. 假设在当前帧第 N 帧之前，为每个像素累积了一些子像素样本（黄色点），每个像素的子像素的样本的平均值存储在第 N-1 帧的历史缓冲区中（绿色点）
2. 对于当前帧中像素的样本（蓝色点），首先根据场景的运动信息将其中心位置（橙色点）映射到上一帧并采样历史缓冲区以获取该像素的历史颜色数据（也就是是该像素累积的子像素样本的平均值），然后在样本位置（蓝色点）采样当前帧的颜色缓冲区得到当前帧的颜色数据，最后将当前帧的颜色和历史颜色混合得到当前帧的最终输出像素颜色
3. 当前帧的最终输出像素颜色也就是第 N+1 帧该像素的历史颜色

## 3. 实现 TAA 的关键技术解析

在渲染管线中实现 TAA 的示意图如下，其中蓝色模块是 TAA ，绿色模块是渲染引擎：

![01_taa_in_render_pipeline](/assets/images/2024/2024-11-06-TemporalAntialiasing/01_taa_in_render_pipeline.jpg)

下面根据上面这张实现 TAA 的示意图来对一些关键的技术进行分析说明。

### 3.1. 抖动偏移（Jitter offset）

为了在每帧中的像素区域内采样不同位置的子像素样本，常见的做法是 **在每帧渲染场景之前，将屏幕像素抖动偏移添加到相机的投影矩阵中** ，在 Unity 中的实现代码：

```csharp
// 添加屏幕像素抖动偏移
float cameraTargetWidth = (float)cameraDescriptor.width;
float cameraTargetHeight = (float)cameraDescriptor.height;
projectionMatrix.m02 -= s_Jitter.x * (2.0f / cameraTargetWidth);
projectionMatrix.m12 -= s_Jitter.y * (2.0f / cameraTargetHeight);
```

而为了使每个像素都能被多个帧生成的样本均匀覆盖，并且这些样本位置要足够随机才可以快速收敛，抖动偏移通常来自某些低差异序列，如 Halton 或者 Sobol 序列。例如在 UE4 中，默认使用来自 Halton(2,3) 的 8 样本序列。

### 3.2. 重投影（Reproject）

通过 **反向重投影（Reverse reprojection）** 计算每个像素在上一帧的对应位置并采样历史缓冲区以获取历史像素数据（也就是历史累积的子像素样本的平均值）。

在 TAA 中通常的做法是：在每一帧光栅化的过程中，将场景中的几何体进行两次 MVP 变换（运动的物体 M 矩阵也会有变化），一次使用前一帧的 MVP 变换矩阵，一次使用当前帧的 MVP 变换矩阵，以此计算出屏幕空间中的偏移量并存储在一个 Motion Vector （也有的引擎中叫做 Velocity ）纹理中，后续通过使用该纹理计算每个像素的重投影历史缓冲区坐标。因为 Motion Vector 纹理中存储的是坐标的偏移量，所以 Motion Vector 纹理需要使用较高的精度纹理格式（R16G16_SFloat）来存储偏移量。

对于运动的物体，我们可以通过此方法计算 Motion Vector ，而对于静止物体，则不需要这么复杂的步骤，只需要通过使用深度缓冲区重建每个像素的裁剪空间坐标，并使用当前帧相机 VP 矩阵的逆矩阵和上一帧相机的 VP 矩阵投影到上一帧：

$$
\mathbf{p}_{n-1} = M_{n-1} M_{n}^{-1} \mathbf{p}_{n}
$$

$\mathbf{p}_{n}$ 是当前帧像素在裁剪空间的坐标：

$\mathbf{p}_{n} = (\frac{2x}{w} - 1, \frac{2y}{h} - 1, z, 1)$

其中 $x, y$ 是屏幕空间像素坐标（以像素为单位）， $w, h$ 是屏幕宽高（以像素为单位）， $z$ 是深度缓冲区中该像素的深度值，而 $M_{n-1}$ 和 $M_{n}$ 分别是上一帧相机的 VP 矩阵和当前帧相机的 VP 矩阵。

一些引擎出于性能上的考虑，将受移动物体影响的像素通过 Stencil Buffer 标记区分出来，对于受移动物体影响的像素计算其 Motion Vector ，而对于其他的像素，则通过使用深度缓冲区重建裁剪空间坐标的形式计算其在上一帧的位置。例如在 Unity 6 的 HDRP 中，受移动物体影响的像素会通过 `ObjectsMotionVector` 这个 Pass 计算其 Motion Vector ，而其它的像素则通过 `CameraMotionVectors` 这个 Pass 来计算其 Motion Vector ，所有计算出的结果都存储在 Motion Vector 纹理中，在 TAA 的计算过程中，只需要通过采样一次 Motion Vector 纹理就可以计算出像素在上一帧中的位置了。

在计算出重投影坐标并对历史缓冲区进行采样时，有 2 点需要注意：

1. 根据当前像素中心坐标计算出的上一帧重投影坐标并不一定正好在像素中心（因为抖动偏移），当前帧和上一帧之间不再是 1:1 的像素映射关系，所以在采样上一帧历史缓冲区时，需要进行重采样（Resampling），以确保不造成 失真伪影（ Distortion Artifact ），通常是使用硬件加速的双线性过滤或双三次纹理过滤（直接使用双三次纹理过滤对性能消耗还是太大了，通常使用的是简化的算法，例如这篇文章[[2]](https://vec3.ca/bicubic-filtering-in-fewer-taps/)里的算法）来进行重采样
2. 由于 Motion Vector 纹理无法进行抗锯齿处理，使用 Motion Vector 纹理进行重投影可能会在移动物体的边缘引入新的锯齿现象，一个简单的方法是对 Motion Vector 纹理中所有的前景物体进行一次膨胀（Dilate）操作，具体细节参考[[3]](https://de45xmedrsdbp.cloudfront.net/Resources/files/TemporalAA_small-59732822.pdf)

### 3.3. 验证（Validate）

从上一帧重投影的历史像素数据可能由于场景遮挡变化、光照变化或者着色变化而变的过时，并且与当前帧像素的数据不一致。在这种情况下，直接使用过时的历史像素数据会导致明显的 时间伪影（Temporal Artifact） ，例如 鬼影（Ghosting） 现象：

![03_ghosting](/assets/images/2024/2024-11-06-TemporalAntialiasing/03_ghosting.jpg)

所以要 **对历史像素数据进行验证** ，只有通过验证的历史像素数据才可以使用。常用的验证方法有 **拒绝（Rejection）** 和 **修正（Rectification）** 。

#### 3.3.1. 拒绝（Rejection）

对于过时或无效的历史像素数据，最简单的一种方法就是拒绝该数据。一般来说，有两个方面的数据可以用来验证历史像素数据的可信度： **几何数据（深度、法线、物体ID 和 Motion Vector）** 和 **颜色数据** 。

几何数据通常用于识别由于场景遮挡变化从而导致历史像素数据无效的情况，通过将当前帧深度与重投影的上一帧深度进行比较，并超过某个设定的小的阈值来判断场景遮挡变化的情况，为了是验证结果更准确还可以使用法线和物体ID 等信息作为额外的判断指标。而颜色数据对于由光照变化或着色变化导致的历史像素数据无效，或由于重采样误差和不正确的 Motion Vector 导致的 失真伪影（ Distortion Artifact ） 很有用。

需要注意的是，拒绝过时或无效的历史像素数据会重置每个像素样本累积的过程，可能会导致 时间伪影（Temporal Artifact） 增加，这会使 TAA 的效果大打折扣。一种更好的方法是 **修正** 历史像素数据，而不是直接拒绝该数据。

#### 3.3.2. 修正（Rectification）

修正的主要思想是使原本被拒绝的历史像素数据与当前像素数据更 **接近** ，从而产生视觉上可接受的结果。

因为当前帧的像素数据通常都是未经过 TAA 抗锯齿处理的，所以获取当前帧的像素数据时一般将每个像素的 3x3 或更大的邻域像素一起考虑进来：

![04_samples_3x3](/assets/images/2024/2024-11-06-TemporalAntialiasing/04_samples_3x3.jpg)

一种比较复杂的修正算法是计算当前帧相邻像素样本的凸包（Convex hull），这个凸包表示了我们期望的当前帧中心像素的颜色数据范围。如果历史像素数据在这个凸包内，则表示通过验证可以直接使用，而如果在凸包之外，则表示其需要被修正。修正的方法是计算历史像素数据与当前帧中心像素数据的连线与凸包的交点，交点处的颜色数据就是历史像素数据被修正后的结果：

![05_convex_hull](/assets/images/2024/2024-11-06-TemporalAntialiasing/05_convex_hull.jpg)

在实时渲染中，计算凸包以及连线与凸包的交点太过复杂，另一种近似的方法是通过当前帧相邻像素的最小值和最大值来计算 AABB ，并将历史像素数据 Clip 或者 Clamp 到 AABB 上。这种方法计算简单速度快，但是这种近似的方式会增加产生鬼影（ Ghosting ）现象的可能性，一种减少鬼影现象的方法是 **将 RGB 空间中的颜色转换到 YCoCg 空间** 进行计算，因为 YCoCg 空间将色度 (Co, Cg) 通道与亮度 (Y) 通道分离，通过亮度 (Y) 通道计算的 AABB 更加紧凑：

![06_aabb](/assets/images/2024/2024-11-06-TemporalAntialiasing/06_aabb.jpg)

对于非常明亮或者非常暗的异常值像素计算出的 AABB 可能会特别大，从而导致本应该接受修正的历史像素数据通过验证。有一种改进的方法是通过使用局部像素数据的均值 $\mu$ 和标准差 $\sigma$ 来代替最小值和最大值去计算 AABB ：

$$
\begin{align*}
    C_{min} = \mu - \gamma \sigma \\
    C_{max} = \mu + \gamma \sigma
\end{align*}
$$

其中， $\gamma$ 参数的取值范围通常是 $[0.75, 1.25]$ 。计算得到的 $C_{min}$ 和 $C_{max}$ 用于替代 Clip 历史像素数据的范围：

![07_variance](/assets/images/2024/2024-11-06-TemporalAntialiasing/07_variance.jpg)

最后展示一下上面介绍的这几种方法对历史像素数据修正的结果：

![08_rectification_results](/assets/images/2024/2024-11-06-TemporalAntialiasing/08_rectification_results.jpg)

### 3.4. 样本累积（Accumulate）

在历史颜色数据通过验证之后，就需要将历史数据 **累积（混合）** 到当前帧像素新着色的样本数据中，以获得像素最终的输出颜色数据。

理论上，我们需要存储每个像素在历史帧中累积的所有子像素样本数据，并与当前帧像素的样本结果进行混合得到最终输出的抗锯齿结果：

$$
\begin{equation}
s_t = \frac{1}{n} \sum_{k=0}^{n-1} x_{t-k} \tag{1} \label{eq:式子1}
\end{equation}
$$

其中， $x$ 是每个子像素样本的数据， $s$ 是当前帧最终的像素输出结果。对于颜色数据来说使用历史帧的 $n=2$ 个子像素样本数据可以得到简单的近似结果，而对于亮度数据来说使用历史帧的 $n=5$ 个子像素样本数据才可以得到简单的近似结果[[3]](https://de45xmedrsdbp.cloudfront.net/Resources/files/TemporalAA_small-59732822.pdf)。

在实时渲染中，这种存储每个像素在历史帧中累积的所有子像素样本数据的方法明显是不现实的。在大多数的 TAA 实现中，每个像素的子像素累积样本会被平均并存储为单个颜色，这个颜色是当前帧的像素最终的输出颜色数据，也是下一帧的历史颜色数据，迭代累积的过程可以表示为：

$$
s_t = \alpha x_t + (1 - \alpha) s_{t-1} \tag{2} \label{eq:式子2}
$$

其中， $s_t$ 是当前帧 $t$ 像素的最终颜色输出， $\alpha$ 是混合因子， $x_t$ 是当前帧 $t$ 的子像素样本颜色数据， $s_{t-1}$ 是使用重投影和重采样从上一帧获取的历史颜色数据。在[[3]](https://de45xmedrsdbp.cloudfront.net/Resources/files/TemporalAA_small-59732822.pdf)中有证明当 $\alpha$ 较小时， $\eqref{eq:式子1}$ 和 $\eqref{eq:式子2}$ 是等价的：

![09_equal](/assets/images/2024/2024-11-06-TemporalAntialiasing/09_equal.jpg)

在实时渲染中，物理上正确的后处理效果是在线性HDR色彩空间中进行计算的，为了避免 TAA 对一些高亮的后处理特效（例如辉光、镜头光晕）产生影响（下图 b））， TAA 应该在这些后处理特效之前。

另一方面，由于色调映射（Tone Mapping）是非线性的，它会对 TAA 抗锯齿的滤波操作产生影响（下图 c）），所以应该在色调映射后的颜色空间中进行 TAA 计算，以在显示器上产生正确的边缘抗锯齿效果。

通常的做法是在 TAA 计算之前，先进行一次可逆的色调映射计算（例如使用 Reinhard 色调映射），然后再进行 TAA 计算，最后再将输出的结果逆向色调映射回线性HDR颜色空间并进行后续的后处理特效计算（下图 b））。

![10_when_taa](/assets/images/2024/2024-11-06-TemporalAntialiasing/10_when_taa.jpg)

最后，为了避免色调映射时颜色的饱和度被降低的影响，[[3]](https://de45xmedrsdbp.cloudfront.net/Resources/files/TemporalAA_small-59732822.pdf)中提出将颜色的亮度考虑进色调映射中，这样色调映射和逆向色调映射就变为：

$$
\begin{align*}
    T(c) = \frac{c}{1 + L(c)} \\
    T^{-1}(c) = \frac{c}{1 - L(c)}
\end{align*}
$$

其中， $c$ 是颜色， $L(c)$ 是亮度。

### 3.5. 采样滤波的选择

在进行多个样本点的采样时，每个样本点的权重是一样的，这相当于是一个 Box filter ，而 Box filter 在运动的场景中表现不稳定，[[3]](https://de45xmedrsdbp.cloudfront.net/Resources/files/TemporalAA_small-59732822.pdf) 中提出使用 3.3 宽 Blackman-Harris 窗口的高斯拟合（Gaussian fit of a 3.3-wide Blackman-Harris window）权重来进行滤波：

![11_blackman_harris](/assets/images/2024/2024-11-06-TemporalAntialiasing/11_blackman_harris.jpg)

### 3.6. Mipmap bias

由于纹理通常使用 Mipmap 进行过滤，所以 TAA 可能会使输出的画面过度模糊，一般通过应用一个 Mipmap bias 来对纹理进行采样。 Mipmap bias 的计算与输入像素的有效样本数量有关，例如，假设每个输入像素的有效样本数为 4 ，那么 Mipmap bias 的值为 $-\frac{1}{2} \log_{2} 4 = -1.0$ ，一般为了避免影响纹理缓存效率， Mipmap bias 会是 -1.0 到 0.0 之间一个较小的偏差值。

下图展示了使用统一的混合权重 $\alpha$ 和相同的历史数据验证方式下， $\alpha$ 与像素的有效累积样本数量的关系：

![12_effective_number_accumulated](/assets/images/2024/2024-11-06-TemporalAntialiasing/12_effective_number_accumulated.jpg)

从上图中可以看到，假设使用 $\alpha = 0.1$ 的权重来进行混合，5 帧累积的结果相当于每个像素 2.2 个有效样本， 10 帧累积的结果相当于是每个像素 5.1 个有效样本。

## 4. TAA 效果展示

我在自己的 [Tiny RP](https://github.com/zwcmc/TinyRenderPipeline) 工程中实现了 TAA 这个算法，最后展示一下它的效果吧：

![13_noaa_fxaa_taa](/assets/images/2024/2024-11-06-TemporalAntialiasing/13_noaa_fxaa_taa.jpg)

## 5. 参考

- [1] [A Survey of Temporal Antialiasing Techniques](http://www.leiy.cc/publications/TAA/TemporalAA.pdf)
- [2] [Bicubic Filtering in Fewer Taps](https://vec3.ca/bicubic-filtering-in-fewer-taps/)
- [3] [High Quality Temporal Supersampling](https://de45xmedrsdbp.cloudfront.net/Resources/files/TemporalAA_small-59732822.pdf)
