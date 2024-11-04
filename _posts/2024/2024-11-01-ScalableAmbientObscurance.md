---
layout: post
title:  "Scalable Ambient Obscurance"
date:   2024-11-01 16:16:00
category: Rendering
---

- [(1) 简单介绍一下 SSAO](#1-简单介绍一下-ssao)
- [(2) SAO](#2-sao)
  - [(2.1) Hi-Z （ Hierarchical z ）](#21-hi-z--hierarchical-z-)
    - [(2.1.1) Hi-Z 中存储的深度值](#211-hi-z-中存储的深度值)
    - [(2.1.2) Mipmap 的过滤方式](#212-mipmap-的过滤方式)
    - [(2.1.3) Hi-Z 实现](#213-hi-z-实现)
  - [(2.2) 根据线性深度重建观察空间的坐标与法线向量](#22-根据线性深度重建观察空间的坐标与法线向量)
    - [(2.2.1) 重建观察空间的坐标](#221-重建观察空间的坐标)
    - [(2.2.2) 重建观察空间的法线向量](#222-重建观察空间的法线向量)
  - [(2.3) AO 的采样计算](#23-ao-的采样计算)
    - [(2.3.1) 圆盘的半径大小](#231-圆盘的半径大小)
    - [(2.3.2) 每个样本在圆盘上旋转的角度以及与中心点的距离](#232-每个样本在圆盘上旋转的角度以及与中心点的距离)
  - [(3) 总结](#3-总结)
- [(4) 参考](#4-参考)

Scalable Ambient Obscurance （以下简称为 SAO ）是 2012 年由 Morgan McGuire, Michael Mara 和 David Luebke 在 [Scalable Ambient Obscurance](https://research.nvidia.com/publication/2012-06_scalable-ambient-obscurance) 这篇文章中提出一种屏幕空间环境光遮蔽（Screen Space Ambient Occlusion, SSAO）的算法。最近在自己的 [Tiny RP](https://github.com/zwcmc/TinyRenderPipeline) 工程中实现了此算法，此篇笔记的目的是为了对这个算法做一下总结，都是我自己的理解，可能有遗漏或者理解错误的地方～

## (1) 简单介绍一下 SSAO

顾名思义，屏幕空间环境光遮蔽指的是基于屏幕空间计算环境光遮蔽的算法，最早由 Crytek 在 2007 年发布，并在其游戏孤岛危机（Crysis）中使用了此技术。

传统的 SSAO 算法对前向渲染管线不友好，这是因为它需要额外的深度缓冲和法线缓冲数据（也有的实现还需要坐标信息），所以一般情况下此算法更适合用于延迟渲染管线中。

SSAO 的实现原理可以总结为：

1. 从深度缓冲中获取深度信息 $z$ ，从法线缓冲中获取法线向量 $\mathbf{n}$ ，从坐标信息中获取坐标 $\mathbf{p}$ （如果没有坐标信息的话，通过深度信息、屏幕空间坐标以及投影矩阵重建计算得到）
2. 将 $z$ 转换到线性深度，也就是观察空间中的深度值，将 $\mathbf{n}$ 和 $\mathbf{p}$ 都转换到相同的观察空间中
3. 在沿着 $\mathbf{n}$ 方向的单位半球中生成 N 个随机方向向量，根据此随机方向向量生成新的位置坐标 $\mathbf{p}_i$
4. 将每个样本 $\mathbf{p}_i$ 转换到屏幕空间并从深度缓冲中得到深度值 $z_i$ ，将 $z_i$ 与 $z$ 做比较（大于代表对 AO 有贡献，反之则没有贡献）
5. 根据每个样本比较的结果加权计算 AO （通常距离原像素更近权重越大）
6. 对得到的 AO 图进行模糊滤波

## (2) SAO

SAO 相较于传统的 SSAO 算法主要有两个方面的改进：

- 仅需要深度缓冲数据，对前向渲染管线更友好（需要额外的一个 Depth Prepass ）
- 通过 Hi-Z 将深度缓冲分层（Mipmap），提高读取深度缓冲的缓存效率，减少实际读取 DRAM 的次数，带宽消耗更低，性能表现更优。这种优化在高渲染分辨率和更大的 AO 搜索半径下更加明显

### (2.1) Hi-Z （ Hierarchical z ）

在对深度缓冲进行 Mipmap 分级时，需要考虑两个方面的问题：

- Hi-Z 中存储的是线性还是非线性的深度值
- Mipmap 的过滤方式

#### (2.1.1) Hi-Z 中存储的深度值

因为在后续计算 AO 的过程中，我们是通过观察空间中的线性深度来重建观察空间的坐标和法线向量的，所以在 Hi-Z 中需要存储观察空间中的线性深度，这使得在计算 AO 的过程中，只需要一次采样就可以得到观察空间中的线性深度，而不需要额外的转换计算。

#### (2.1.2) Mipmap 的过滤方式

原文中对几种不同 Mipmap 过滤方式做了总结，如下图所示：

![00_mipmap_filtering](/assets/images/2024/2024-11-01-ScalableAmbientObscurance/00_mipmap_filtering.jpg)

在这里我选择了原文的结论，使用效果最优的旋转网格子采样（Rotated Grid Subsample）来进行 Mipmap 过滤。

#### (2.1.3) Hi-Z 实现

对于 Mipmap 0 ，在读取深度缓冲中的深度值后，将其转换到观察空间中的线性深度，并写入到 Mipmap 0 中。而对于其它的 Mipmap 层级，则使用旋转网格子采样来进行 Mipmap 过滤。具体的 Compute Shader 代码如下：

```hlsl
#pragma kernel CSCopyMipmap0Depth
#pragma kernel CSMipmapDepth

Texture2D<float> _PrevMipDepth;
RWTexture2D<float> _CurrMipDepth;

float4 _ZBufferParams;
float LinearEyeDepth(float depth)
{
    return 1.0 / (_ZBufferParams.z * depth + _ZBufferParams.w);
}

[numthreads(8, 8, 1)]
void CSCopyMipmap0Depth(uint3 id : SV_DispatchThreadID)
{
    // Calculate the linear depth and store in mipmap 0
    _CurrMipDepth[id.xy] = LinearEyeDepth(_PrevMipDepth.Load(uint3(id.xy, 0)));
}

[numthreads(8, 8, 1)]
void CSMipmapDepth(uint3 id : SV_DispatchThreadID)
{
    // Rotated Grid Subsample
    uint2 prevId = id.xy << 1;
    uint2 currId = prevId + uint2(id.y & 1, id.x & 1);
    _CurrMipDepth[id.xy] = _PrevMipDepth.Load(uint3(currId, 0));
}
```

### (2.2) 根据线性深度重建观察空间的坐标与法线向量

#### (2.2.1) 重建观察空间的坐标

首先来看看观察空间中的 $xy$ 坐标转换到屏幕空间 $xy$ 坐标的过程：

1. 通过投影矩阵从观察空间转换到裁剪空间
2. 透视除法（除以齐次坐标的 $w$ 分量，也就是观察空间中的线性深度 $z$ ）转换到 $[-1,1]$ 的 NDC 坐标下
3. 将 $[-1,1]$ 映射到 $[0,1]$ ，也就是屏幕空间的 $xy$ 坐标

而重建的原理也就是逆向上面的转换步骤：

1. 首先，将屏幕空间坐标转换到 $[-1,1]$
2. 然后，乘以观察空间中的线性深度 $z$ 得到裁剪空间中的 $xy$ 坐标
3. 最后通过投影矩阵的逆矩阵将裁剪空间中的 $xy$ 坐标转换到观察空间

具体的实现代码如下：

```hlsl
float3 ReconstructViewSpacePositionFromDepth(float2 uv, float linearDepth)
{
    return float3((uv - 0.5) * _PositionParams.xy * linearDepth, linearDepth);
}
```

其中 `_PositionParams` 的 `x` 和 `y` 分量分别是：

```csharp
Matrix4x4 projectionMatrix = GL.GetGPUProjectionMatrix(renderingData.camera.projectionMatrix, true);
var invProjection = projectionMatrix.inverse;
_PositionParams.x = invProjection.m00 * 2.0f;
_PositionParams.y = invProjection.m11 * 2.0f;
```

#### (2.2.2) 重建观察空间的法线向量

观察空间的法线向量根据观察空间中的坐标相对于屏幕空间 $x$ 坐标和 $y$ 坐标的偏导数的叉积得到。代码如下：

```hlsl
float3 ReconstructViewSpaceNormal(float2 uv, float3 C, float2 texelSize)
{
    float2 dx = float2(texelSize.x, 0.0);
    float2 dy = float2(0.0, texelSize.y);

    float2 uvdx = uv + dx;
    float2 uvdy = uv + dy;
    float3 px = ReconstructViewSpacePositionFromDepth(uvdx, SampleMipmapDepthLod(uvdx));
    float3 py = ReconstructViewSpacePositionFromDepth(uvdy, SampleMipmapDepthLod(uvdy));

    float3 dpdx = px - C;
    float3 dpdy = py - C;
    return normalize(cross(dpdx, dpdy));
}
```

### (2.3) AO 的采样计算

随机采样的样本，是基于屏幕空间的螺旋圆盘分布，具体的分布主要由 3 个变量控制，分别是圆盘的半径大小、每个样本在圆盘上旋转的角度以及每个样本与中心点距离（从中心到圆盘边缘螺旋扩大）。

#### (2.3.1) 圆盘的半径大小

圆盘的半径大小是根据单位大小为 1 m 的物体在距离相机为 1 m 的位置覆盖屏幕像素范围大小（以像素为单位）的估计值：

```csharp
// 屏幕的宽高(以屏幕像素为单位)
Vector2Int screenSizeInPixels = new Vector2Int(saoDescriptor.width, saoDescriptor.height);
// 投影矩阵将 xy 转换到 [-1,1] , 也就是 2 的范围 , 所以这里乘以 0.5
float projectionScale = Mathf.Min(0.5f * projectionMatrix.m00 * screenSizeInPixels.x, 0.5f * projectionMatrix.m11 * screenSizeInPixels.y);
// 计算经过投影矩阵缩放后的屏幕空间内范围大小(以屏幕像素为单位)
float projectionScaledRadius = projectionScale * radius;
```

其中， `radius` 是自定义的搜索范围大小。在 Shader 中计算 AO 时，还需要除以当前像素的线性深度，保证距离相机越远，采样的范围就越小：

```hlsl
float ssDiskRadius = -(_PositionParams.z / origin.z);
```

#### (2.3.2) 每个样本在圆盘上旋转的角度以及与中心点的距离

因为是螺旋圆盘分布，所以采样点的分布是从中心到边缘螺旋式的扩大。

每次采样的半径计算如下：

```hlsl
float TapRadius(float tapIndex, float jitter)
{
    float r = (tapIndex + jitter + 0.5) * _StepTapRadius;
    return r * r
}
```

其中， `_StepTapRadius` 是 `1.0f / (sampleCount - 0.5f)` ， `jitter` 是随机抖动，而 `r * r` 是为了让采样点分布在靠近中心的区域更密集。

而每次采样的旋转角度通过在 CPU 端计算好角度的 $\cos$ 和 $\sin$ 值，并在 Shader 中构造一个 `2x2` 的矩阵，每次采样通过这个矩阵旋转采样坐标：

```csharp
const float spiralTurns = 6.0f;
const float stepTapRadius = 1.0f / (sampleCount - 0.5f);
// 每次采样旋转的角度
float stepTapAngle = stepTapRadius * spiralTurns * (2.0f * Mathf.PI);
// 计算这个旋转角度的 cos 和 sin 值, 并在后续的 Shader 计算中构建一个 2x2 的旋转矩阵
Vector2 angleIncCosSin = new Vector2(Mathf.Cos(stepTapAngle), Mathf.Sin(stepTapAngle));
```

### (3) 总结

以上就是我对 SAO 这个算法的一些总结。还有一些实现细节我这里没有做说明，例如怎么计算 AO 贡献、使用双边滤波模糊对 AO 图进行降噪、为了节省带宽将高精度的线性深度存储在 RGB8 的两个通道中存储传递等。 SAO 算法可以在比较少的采样点下得到还不错的效果，最后展示一下我在自己电脑上运行的效果吧，电脑是 Apple M1 Max Mackbook Pro，采样次数是 9 ，采样范围是 0.3 ，分辨率是 2560x1440 ，耗时 1.0 ms 左右：

![01_sao](/assets/images/2024/2024-11-01-ScalableAmbientObscurance/01_sao.jpg)

## (4) 参考

- [1] [Scalable Ambient Obscurance](https://research.nvidia.com/publication/2012-06_scalable-ambient-obscurance)
- [2] [filament](https://github.com/google/filament)
