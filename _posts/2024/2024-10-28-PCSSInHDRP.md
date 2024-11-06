---
layout: post
title:  "HDRP 中的 PCSS"
date:   2024-10-28 16:16:00 +800
category: Unity
---

本篇笔记是关于 HDRP 中方向光 PCSS 算法的分析和理解， HDRP 的版本是 16.0.6 。

- [PCSS（Percentage-Closer Soft Shadows）](#pcsspercentage-closer-soft-shadows)
- [HDRP 中的 PCSS](#hdrp-中的-pcss)
  - [斐波那契螺旋盘采样（Fibonacci Spiral Disk Sampling）](#斐波那契螺旋盘采样fibonacci-spiral-disk-sampling)
  - [角直径（Angular diameter）](#角直径angular-diameter)
  - [在搜索 Blocker 和 PCF 采样时对着色点深度的偏移](#在搜索-blocker-和-pcf-采样时对着色点深度的偏移)
  - [PCSS 效果对比](#pcss-效果对比)

## PCSS（Percentage-Closer Soft Shadows）

PCSS 是一种基于 PCF 的改进型软阴影算法，它的主要步骤是：

1. **搜索 Blocker（Blocker search）** ：首先，在阴影贴图上搜素一定范围内的纹素，计算出 **此范围内距离光源比着色点 $d_{Receiver}$ 更近（也就是“遮挡”了着色点）的所有深度的平均值** ，即 $d_{Blocker}$

    ![01_blocker_search](/assets/images/2024/2024-10-28-PCSSInHDRP/01_blocker_search.jpg)

2. **计算半影大小（Penumbra estimation）** ：根据搜索到的 Blocker 的深度 $d_{Blocker}$ 、着色点的深度 $d_{Receiver}$ 和光源的大小 $w_{Light}$ 来计算出半影的宽度 $w_{Penumbra}$

    ![02_penumbra_estimation](/assets/images/2024/2024-10-28-PCSSInHDRP/02_penumbra_estimation.jpg)

3. **PCF 采样（PCF Filtering）** ：根据 $w_{Penumbra}$ ，使用不同的 PCF 核的大小（ $w_{Penumbra}$ 越大， PCF 的核越大）来进行 PCF 采样

## HDRP 中的 PCSS

### 斐波那契螺旋盘采样（Fibonacci Spiral Disk Sampling）

在 HDRP 的 PCSS 算法中，使用了斐波那契螺旋盘生成对阴影贴图随机采样的坐标。

在原文章[Spherical Fibonacci Point Sets for Illumination Integrals](https://people.irisa.fr/Ricardo.Marques/articles/2013/SF_CGF.pdf) 中，生成的是半球上的球面坐标 $\theta$ 和 $\phi$ ，核心算法如下：

![03_spherical_fibonacci_point_set_algorithm](/assets/images/2024/2024-10-28-PCSSInHDRP/03_spherical_fibonacci_point_set_algorithm.jpg)

HDRP 通过预计算的方式，计算了 64 次采样数量下的 $\phi$ 坐标的 $\cos\phi$ 值和 $\sin\phi$ 值，存储在 `static const float2 fibonacciSpiralDirection[DISK_SAMPLE_COUNT]` 中，因为 $\phi$ 是方位角，范围是 $[0,2\pi]$ 也就是一个圆盘。预计算的代码如下：

```js
var res = "";
for (var i = 0; i < 64; ++i)
{
    var a = Math.PI * (3.0 - Math.sqrt(5.0));
    var b = a / (2.0 * Math.PI);
    var c = i * b;
    var theta = (c - Math.floor(c)) * 2.0 * Math.PI;
    res += "float2 (" + Math.cos(theta) + ", " + Math.sin(theta) + "),\n";
}
```

在 PCSS 的计算时，只需要通过圆盘的半径分别乘以 $\cos\phi$ 值和 $\sin\phi$ 值就可以得到基于斐波那契螺旋盘分布的坐标了，而螺旋构造的半径与采样样本 index 有关。下面分别是用于搜索 Blocker 和最终 PCF Filter 时生成斐波那契螺旋盘坐标点的函数：

```hlsl
// 基于斐波那契螺旋盘生成在中心区域分布更密集的随机采样坐标点，用于 Blocker 的搜索
float2 ComputeFibonacciSpiralDiskSampleClumped(const in int sampleIndex, const in float sampleCountInverse, out float sampleDistNorm)
{
    // 在这里不使用 sampleBias 是因为中心点 (0, 0) 的采样结果对于 Blocker 的搜索很重要
    sampleDistNorm = (float)sampleIndex * sampleCountInverse;

    // pow(r, 3.0) ， 越靠近中心的区域，采样点分布更密集，更有利于 Blocker 的搜索
    sampleDistNorm = sampleDistNorm * sampleDistNorm * sampleDistNorm;

    // 半径乘以 \cos\phi 和 \sin\phi 得到采样坐标点
    return fibonacciSpiralDirection[sampleIndex] * sampleDistNorm;
}

// 基于斐波那契螺旋盘生成随机采样坐标点
float2 ComputeFibonacciSpiralDiskSampleUniform(const in int sampleIndex, const in float sampleCountInverse, const in float sampleBias, out float sampleDistNorm)
{
    // sampleBias 是为了防止当 sampleIndex = 0 时，生成出的采样坐标点是 (0, 0)，采样坐标点为 (0, 0) 时，最终采样不会受到 Jitter 的影响，并且会造成明显的边缘 artifact
    sampleDistNorm = (float)sampleIndex * sampleCountInverse + sampleBias;

    // 圆盘的面积与半径的平方成正比，这意味着半径的增量在距离中心较远的地方会覆盖更大的面积。每个均匀半径增量将包含相同数量的点，这会导致靠近中心区域较小面积的面盘上的采样点更密集
    // 通过使用 sqrt(r) 使采样点在整个圆盘上均匀分布
    sampleDistNorm = sqrt(sampleDistNorm);

    // 半径乘以 \cos\phi 和 \sin\phi 得到采样坐标点
    return fibonacciSpiralDirection[sampleIndex] * sampleDistNorm;
}
```

上面两个函数中，用于搜索 Blocker 的采样点在更靠近中心 (0, 0) 的区域，分布更密集，这种分布更有利于搜索 Blocker；而最终 PCF Filter 的采样点分布则是均匀分布，这种分布更有利于生成软阴影。

### 角直径（Angular diameter）

方向光源通常表示的是太阳这类光源，它距离场景无限远，并且影响的是场景中的每个物体。对于方向光源，很难使用一个固定的光源大小 $w_{Light}$ 来描述，在 HDRP 中，使用 **角直径（Angular diameter）** 来描述方向光源的大小。

下图中很清晰明了了表现了在观察太阳时，什么是角直径：

![04_angular_diameter](/assets/images/2024/2024-10-28-PCSSInHDRP/04_angular_diameter.jpg)

使用直角径描述的方向光源，每个着色点就是一个观察点，从观察点看向光源，是一个无限延伸下去的圆锥体（Cone），观察点距离光源越远，则表示光源的大小越大，更大的光源大小也会影响最终 PCF 采样的核的大小。这正符合了 PCSS 的思想，距离光源越远的着色点，它的阴影就应该更“软”！

![05_angular_diameter_formula](/assets/images/2024/2024-10-28-PCSSInHDRP/05_angular_diameter_formula.jpeg)

角直径的计算也很简单，它与光源的直径 $d$ 和观察点（也就是着色点）距离光源的距离 $D$ 有关。具体公式如下：

$$
\delta = 2 \arctan{(\frac{d}{2D})}
$$

### 在搜索 Blocker 和 PCF 采样时对着色点深度的偏移

因为 PCSS 的计算是基于在着色点以一个固定角直径看向光源的圆锥体，而在阴影贴图上的采样样本是位于圆锥体的底面上。和下图中红色的四棱锥类似，只不过我们这说的是圆锥体：

![06_cone](/assets/images/2024/2024-10-28-PCSSInHDRP/06_cone.jpeg)

在计算对阴影贴图上的偏移采样时，所有的偏移都是基于这个角直径计算出来的，为了符合角直径的设定，在每次采样阴影贴图做深度比较时，都应该对着色点的深度进行一定的偏移（想象一个圆锥体，圆锥体的顶点，也就是着色点，距离底面距离最大）。也就是说， **只有在该圆锥体内的 Blocker 才会对阴影产生影响** 。

```hlsl
#if UNITY_REVERSED_Z
    #define Z_OFFSET_DIRECTION 1
#else
    #define Z_OFFSET_DIRECTION (-1)
#endif

// 将着色点的深度进行偏移, 以符合使用直角径来定义方向光的设定 , 这对于消除自遮挡很重要
float radialOffset = filterSize * sampleDistNorm;
float zoffset = radialOffset * (radialOffset < minFilterRadius ? minFilterRadial2DepthScale : radial2DepthScale);
float coordz = shadowCoord.z + (Z_OFFSET_DIRECTION) * zoffset;
```

### PCSS 效果对比

最后，放上一张 PCSS 与 PCF 阴影的效果对比图：

![07_pcf_pcss](/assets/images/2024/2024-10-28-PCSSInHDRP/07_pcf_pcss.jpeg)
