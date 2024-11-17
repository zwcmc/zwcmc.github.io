---
layout: post
title:  "Screen Space Reflection"
date:   2024-11-12 16:16:00 +800
category: Rendering
---

本篇笔记是对我在 [Tiny RP](https://github.com/zwcmc/TinyRenderPipeline) 项目中实现屏幕空间反射（ Screen Space Reflection, SSR 以下简称为 SSR ）的总结，主要包含一些技术细节和原理的分析。

- [1. SSR 简介](#1-ssr-简介)
- [2. 实现 SSR 的总结](#2-实现-ssr-的总结)
  - [2.1. Depth prepass 与模版缓冲 Mask](#21-depth-prepass-与模版缓冲-mask)
  - [2.2. Hi-Z 缓冲（ Hi-Z buffer ）](#22-hi-z-缓冲-hi-z-buffer-)
  - [2.3. SSR 计算](#23-ssr-计算)
  - [2.4. 高斯模糊](#24-高斯模糊)
  - [2.5. 采样反射光照纹理并与环境反射光照混合](#25-采样反射光照纹理并与环境反射光照混合)
- [3. 参考](#3-参考)

## 1. SSR 简介

SSR是一种在实时渲染中模拟反射效果的技术，它通过利用场景的深度缓冲、法线缓冲和颜色缓冲生成反射结果。

SSR 的算法核心可以总结为：对于屏幕上的每个像素，根据观察向量和法线向量计算出反射光线，沿着反射光线方向以一定的步长进行光线 **步进（Marching）** ，当反射光线深度小于（离相机越近深度值越大时）存储在深度缓冲中的场景深度时，表示我们找到了反射光线与场景中几何体的交点，使用交点的屏幕空间坐标读取颜色缓冲中的场景颜色，该颜色也就是反射的颜色。下图展示了 SSR 算法核心的示意图：

![00_ssr_overview](/assets/images/2024/2024-11-12-ScreenSpaceReflections/00_ssr_overview.jpg)

因为 SSR 中所有的计算都是利用屏幕空间中的信息（深度、法线和颜色），所以它也存在一些问题。对于在场景中被遮挡的物体是无法被反射的，如下图中龙的脚底：

![01_hide_ssr](/assets/images/2024/2024-11-12-ScreenSpaceReflections/01_hide_ssr.jpeg)

而对处于屏幕边缘的几何体，其在屏幕外的部分也会被生硬的直接切断，如下图所示：

![02_cutoff_ssr](/assets/images/2024/2024-11-12-ScreenSpaceReflections/02_cutoff_ssr.jpeg)

另外，因为 SSR 还涉及光线步进计算，所以如何选择一个合适的步长也非常重要，如果步长太小，这意味着要频繁采样深度缓冲，对性能消耗过大，而如果步长太大，则有可能跳过深度缓冲中的一些像素（也就是跳过了场景中几何体的信息），从而导致采样不足而造成阶梯状的走样现象。

尽管如此，因为 SSR 可以带来实时的、高质量的反射效果，并且为了解决优化它存在的问题以及加速它的求交计算，也衍生出了不少优秀的加速优化算法，所以 SSR 在现在的实时渲染中变的越来越受欢迎。

## 2. 实现 SSR 的总结

对于我在 Tiny RP 项目中实现的 SSR ，主要包含以下步骤：

- Depth prepass 生成深度缓冲以及在模版缓冲中标记屏幕中需要计算 SSR 的像素
- 生成 Hi-Z 缓冲用以加速反射光线求交计算
- 基于屏幕空间像素的 SSR 计算，生成出反射光照纹理
- 基于高斯模糊生成反射光照纹理的 Mipmap
- 在前向光照计算阶段，根据物体的粗糙度采样反射光照纹理的不同 Mipmap 层级，并将结果与环境反射混合

### 2.1. Depth prepass 与模版缓冲 Mask

因为我们的项目暂时是前向渲染管线，而 SSR 需要场景的深度信息，所以首先通过一个 Depth prepass 来生成深度缓冲，而且为了节省性能，减少不必要的计算，同时输出一个模版缓冲 Mask ，这个模版缓冲 Mask 将屏幕上需要计算 SSR 的像素标记出来。

模版缓冲区中的每个像素存储的是一个 8 位整数值，在这里我们使用最低位作为 SSR 的标识：

![03_ssr_stencil_mask](/assets/images/2024/2024-11-12-ScreenSpaceReflections/03_ssr_stencil_mask.jpg)

### 2.2. Hi-Z 缓冲（ Hi-Z buffer ）

Hi-Z 缓冲也被称为分层 Z 缓冲（ Hierarchical-Z buffer ），它是将深度缓冲像 Mipmap 那样进行分层，其中 Hi-Z Mip 0 是原始深度缓冲，记录的是每个像素的场景深度，而从 Hi-Z Mip 1 开始每一层 Mip 中的每个像素对应的是上一层 Mip 中 4 个像素的左下角位置的中的的像素，并且存储的是这 4 个像素的最小值或最大值：

![04_hi_z](/assets/images/2024/2024-11-12-ScreenSpaceReflections/04_hi_z.jpg)

下图展示了一个简单的 Hi-Z 缓冲的示意图，其中蓝色是靠近相机的区域，而红色是远离相机的区域，灰色的正方形代表一个像素：

![05_z_pyramid](/assets/images/2024/2024-11-12-ScreenSpaceReflections/05_z_pyramid.jpg)

Hi-Z 缓冲最大的作用就是加速光线步进的计算，通过在高层级的 Hi-Z Mip 上采样一次，就可以快速查询场景中一大片区域的最小或最大深度值，这样就可以跳过场景中对光线步进求交计算没有贡献的大块区域，从而减少相当一部分的采样和光线步进求交计算的开销。

在实际的应用中，我们利用 Hi-Z 动态决定每次光线步进的步长，具体的算法是：初始使用较小的步长并采样较低层级的 Hi-Z Mip （这样初始步进距离比较短，检测求交更精确），如果没有检测到交点就增大步长并采样更高 1 级的 Hi-Z Mip ；如果检测到交点，则使用更小的步长并采样低 1 级的 Hi-Z Mip ，直到步进次数达到上限或检测到交点且此时采样的 Hi-Z Mip 层级为 0 。下面这张动图演示了这种算法具体的原理：

![06_hi_z_marching](/assets/images/2024/2024-11-12-ScreenSpaceReflections/06_hi_z_marching.gif)

另外， SSR 中的光线步进求交算法意味着我们需要在深度缓冲中采样查询很大一部分像素的深度值，也就是说 On-chip memory 很有可能是不够用的，需要频繁的从 DRAM 中读取深度缓冲的数据，这会造成更高的带宽消耗。而通过 Hi-Z 缓冲可以提高读取深度缓冲的缓存效率，减少实际读取 DRAM 的次数（因为高层级的 Hi-Z Mip 上的一个像素代表的就是原始深度缓冲中一个范围内像素的最小或最大值），降低带宽消耗，性能表现自然也更好。

当然，额外的 Mip 层级也带来了额外的 1/3 的内存存储需求，同时生成 Hi-Z 缓冲也需要一定的性能消耗，然而，与其优点相比，这些缺点是完全可以接受的。

生成 Hi-Z 缓冲的 Compute Shader 如下，因为有 `REVERSED_Z` ，所以这里选择最大值， `_CurrMipDepth` 是当前 Hi-Z Mip 层级， `_PrevMipDepth` 是上一层级的 Hi-Z Mip ：

```hlsl
[numthreads(8, 8, 1)]
void PyramidMin(uint3 dispatchThreadId : SV_DispatchThreadID)
{
    uint2 prevId = dispatchThreadId.xy << 1;

    float maxDepth = _PrevMipDepth.Load(uint3(prevId.xy, 0)).r;
    // 右
    maxDepth = max(maxDepth, _PrevMipDepth.Load(uint3(prevId.xy + int2(1, 0), 0)).r);
    // 上
    maxDepth = max(maxDepth, _PrevMipDepth.Load(uint3(prevId.xy + int2(0, 1), 0)).r);
    // 右上
    maxDepth = max(maxDepth, _PrevMipDepth.Load(uint3(prevId.xy + int2(1, 1), 0)).r);

    _CurrMipDepth[dispatchThreadId.xy] = maxDepth;
}
```

生成出 Hi-Z 缓冲后，就可以开始进行 SSR 的计算了。

### 2.3. SSR 计算

SSR 的计算是在屏幕空间中进行的，最终输出一张基于屏幕空间的反射纹理。需要注意的是，这里我使用的是 Compute Shader 来计算 SSR 。

首先，根据 Depth prepass 中写入的模版缓冲 Mask ，跳过不需要计算 SSR 的像素，则直接返回 0.0 ：

```hlsl
// 当前像素的屏幕坐标
uint2 positionSS = dispatchThreadId.xy;
// 读取 stencil buffer
uint stencilValue = GetStencilValue(_StencilBuffer.Load(int3(positionSS, 0)));
// 对于不需要计算 SSR 的像素, 直接返回 0.0
bool doesntReceiveSSR = (stencilValue & _SsrStencilBit) == 0;
UNITY_BRANCH
if (doesntReceiveSSR)
{
    _SsrLightingTexture[positionSS] = float4(0.0, 0.0, 0.0, 0.0);
    return;
}
```

然后，根据深度缓冲中当前像素的深度值以及观察投影变换矩阵的逆矩阵重建出世界空间坐标，有了世界空间坐标就可以得到世界空间的观察向量，而对于世界空间的法线向量，因为使用的是前向渲染管线，没有法线缓冲，所以使用[[2]](#ref2)中的算法通过深度缓冲重建出来（文章[[3]](#ref3)中还提供了一种改进的算法，使重建出的法线向量更准确，当然计算也更复杂），最终根据世界空间的观察向量和法线向量计算出反射方向向量：

```hlsl
// 当前像素中心的 uv 坐标
float2 uv = positionSS * _ScreenSize.zw + (0.5 * _ScreenSize.zw);
// 采样深度值
float deviceDepth = _DepthPyramidTexture.SampleLevel(sampler_PointClamp, uv, 0).r;
// 重建世界空间坐标
float3 positionWS = ReconstructWorldSpacePosition(uv, deviceDepth, _InvViewProjection);
// 重建世界空间法线向量
float3 N = ReconstructWorldSpaceNormal(uv, positionWS, _ScreenSize.zw);
// 计算反射方向向量
float3 V = GetWorldSpaceNormalizeViewDir(positionWS);
float3 R = reflect(-V, N);
```

其中，重建世界空间坐标和重建世界空间法线向量的方法如下：

```hlsl
// World space position reconstruction
float3 ReconstructWorldSpacePosition(float2 uv, float deviceDepth, float4x4 invViewProjMatrix)
{
    // 构建裁剪空间坐标
    float4 positionCS = float4(uv * 2.0 - 1.0, deviceDepth, 1.0);

    // 如果是 Y-down 的平台, 反转裁剪空间 y 坐标
#if UNITY_UV_STARTS_AT_TOP
    positionCS.y = -positionCS.y;
#endif

    // 通过 VP 的逆矩阵将裁剪空间坐标变换到世界空间坐标
    float4 hpositionWS = mul(invViewProjMatrix, positionCS);

    // 透视除法
    return hpositionWS.xyz / hpositionWS.w;
}

// World space normal reconstruction
float3 ReconstructWorldSpaceNormal(float2 uv, float3 origin, float2 texelSize)
{
    float2 dx = float2(texelSize.x, 0.0);
    float2 dy = float2(0.0, texelSize.y);

    float2 uvdx = uv + dx;
    float2 uvdy = uv + dy;
    float depth0 = _DepthPyramidTexture.SampleLevel(sampler_PointClamp, uvdx, 0).r;
    float depth1 = _DepthPyramidTexture.SampleLevel(sampler_PointClamp, uvdy, 0).r;
    float3 px = ComputeWorldSpacePosition(uvdx, depth0, _InvViewProjection);
    float3 py = ComputeWorldSpacePosition(uvdy, depth1, _InvViewProjection);

    float3 dpdx = px - origin;
    float3 dpdy = py - origin;

    return normalize(cross(dpdy, dpdx));
}
```

有一点需要注意的是，不同平台的的纹理坐标是有区别的，对于 Direct3D 、 Metal 和 Vulkan 等平台，纹理的左上角是坐标 $(0,0)$ 的位置，而对于 OpenGL 和 OpenGL ES ，纹理的左下角是坐标 $(0,0)$ 的位置，当渲染目标是一张 **渲染纹理（Render Texture）**（我们在做屏幕空间效果时，一般情况下都是将结果渲染到一张渲染纹理中），并且在 Direct3D 、 Metal 和 Vulkan 这些平台时， Unity 在内部会通过修改投影矩阵的方式将裁剪空间的 `y` 轴坐标反转从而将纹理坐标统一成 OpenGL 的布局（Unity 的文档 [Writing shaders for different graphics APIs - Render Texture coordinates](https://docs.unity3d.com/Manual/SL-PlatformDifferences.html) 中对此也有做说明），所以我们在通过屏幕 `uv` 坐标和深度值构建裁剪空间坐标时，如果当前平台的纹理坐标是 Direct3D 这种 Y-down 的情况时，需要对裁剪空间的 `y` 坐标进行反转。

现在我们计算出了反射方向，剩下要做的在屏幕空间进行光线步进找到与场景中几何体的交点了，在这里主要对光线步进的核心算法进行分析说明。

首先要声明的是，每次都是从起点进行步进，通过改变每次步进的步长 `t` 来进行求交计算。

每次步进得到终点在原始屏幕空间中的坐标位置为 `rayPos` ，并且为了避免步进后的原始屏幕坐标处于像素边缘不利于采样深度缓冲，对其应用一个极小的偏移：

```hlsl
// 每次都是从起点步进 t 距离, 计算出此次步进后的终点在原始屏幕空间中的坐标位置
rayPos = rayOrigin + t * rayDir;

// 为了避免 rayPos 处于屏幕像素的边缘, 对其应用一个微小的偏移量
float2 sgnEdgeDist = round(rayPos.xy) - rayPos.xy;
float2 satEdgeDist = clamp(raySign.xy * sgnEdgeDist + SSR_TRACE_EPS, 0, SSR_TRACE_EPS);
rayPos.xy += raySign.xy * satEdgeDist;
```

计算出此次步进的终点坐标后，下面要做的就是采样 Hi-Z 缓冲了：

```hlsl
float4 bounds;
// 将原始屏幕空间的坐标转换到当前 Hi-Z Mip 层级中的坐标
int2 mipCoord  = (int2)rayPos.xy >> mipLevel;
// 在第 n 层级的 Hi-Z Mip 上步进 1 次, 相当于在原始屏幕空间步进了 2^n 次
// 例如, 原始屏幕坐标是 (12,12) , mipCoord = 12 >> 2 = 3, (3 + 1) << 2 = 16, 也就是说在 Mip 2 层级步进 1 次, 相当于在原始屏幕空间步进了 4 次
bounds.xy = (mipCoord + rayStep) << mipLevel;
// 采样当前 Hi-Z Mip 层级中的深度值, 可以理解为几何体表面深度值
bounds.z = _DepthPyramidTexture.Load(int3(mipCoord, mipLevel)).r;
// 应用几何体厚度后的深度值, 可以理解为几何体背面深度值
bounds.w = bounds.z * thickness_scale + thickness_bias;
```

我们根据当前 Hi-Z 层级, 计算出在此次步进终点采样 Hi-Z 能覆盖的原始屏幕空间中最远的屏幕坐标位置。下面来结合一张图来详细解释一下这句话的意思：

![07_mip_mincoord](/assets/images/2024/2024-11-12-ScreenSpaceReflections/07_mip_mincoord.jpg)

首先，在我们的 Hi-Z 缓冲中，每一层 Mip 中每个像素对应的是上一层 Mip 中 4 个像素的左下角位置的中的像素，并且存储的是这 4 个像素中的最大值，如上图中所示， Mip 1 中的像素 $(1,1)$ 存储的是 Mip 0 中对应的 $(2,2), (3,2), (2,3), (3,3)$ 这 4 个像素中的最大值（具体的 Hi-Z 生成代码可以看上面的 [Hi-Z 缓冲（ Hi-Z buffer ）](#22-hi-z-缓冲-hi-z-buffer-) ）。

那么，在我们计算出反射光线步进的终点后，根据 Mip 层级，我们就可以计算出采样当前 Mip 层级可以覆盖的相对于步进起点的最远距离的坐标，还是这个例子，假设步进后在 Mip 0 中的坐标是 $(2,2)$ ，而此时的 Mip 层级为 1 ，那么计算出的 `mipCoord` 就是 $(2,2) >> 1 = (1,1)$ ，采样 Mip 1 中的 $(1,1)$ 坐标的深度值就代表了我们获取到的最远距离是 $((1,1)+1) << 1 = (4,4)$ ，也就是上图中 Mip 0 的右上角位置。

另外需要注意的是，当步进方向是负方向（也就是上图中往左下角进行步进）时，此时就不需要有 $+1$ 的操作了，也就是上面的最远距离计算变为 $((1+1)+0) << 1 = (2,2)$ ，因为当步进方向是负方向时，此时采样点表示的就是相对于起点的最远距离坐标，这也是由我们生成 Hi-Z 缓冲的方式而决定的。在实际的代码中， `rayStep` 会根据步进的方向设定不同的值，正方向时为 `1` ，负方向时为 `0` ：

```hlsl
int2 rayStep = int2(rcpRayDir.x >= 0 ? 1 : 0,
                    rcpRayDir.y >= 0 ? 1 : 0);
```

下面就是根据终点的深度与深度缓冲中的深度来判断反射光线是否与场景中的几何体有交点了。具体看一下代码：

```hlsl
// 是否在几何体表面以下
bool belowFloor = rayPos.z < bounds.z;
// 是否在几何体背面以上
bool aboveBase = rayPos.z >= bounds.w;
// 在几何体内部
bool insideFloor = belowFloor && aboveBase;
```

然后，我们计算此次步进终点距离起点的距离与此次步进步长 `t` 的关系，并判断此次步进是否落在场景内几何体的表面上：

```hlsl
// 在原始屏幕空间上, 此次步进终点能覆盖的最远坐标位置相对于起点的距离变化
float4 dist = bounds * rcpRayDir.xyzz - (rayOrigin.xyzz * rcpRayDir.xyzz);
float distWall  = min(dist.x, dist.y);
float distFloor = dist.z;
// 当每次的步长 t 与 t <= distFloor <= distWall , 则认为交点正好在几何体表面
bool hitFloor = (t <= distFloor) && (distFloor <= distWall);
```

判断是否此次步进是否找到与场景中几何体的交点，其中 `belowMip0` 默认值为 `false` ，当反射光线已经进入几何体表面以下并且此时的 Hi-Z Mip 层级为 0，我们就认为不需要再进行交点，直接将 `belowMip0` 设置为 `true` ：

```hlsl
// 如果 belowMip0 为 true , 表示上次步进时, Mip 已经为 0 且没有找到交点, 则认为没有找到交点, 跳出步进
miss = belowMip0;
// 当前在 Mip 0 且交点在几何体表面或几何体内部则认为找到交点, 跳出步进
hit = (mipLevel == 0) && (hitFloor || insideFloor);

// 当反射光线深度在几何体表面以下, 且此时采样的 Hi-Z Mip 为 0 , 则 belowMip0 为 true , belowMip0 默认为 false
belowMip0 = (mipLevel == 0) && belowFloor;
```

如果交点在几何体表面, 则减小步长, 使用 `distFloor` 作为下一次步进的步长, 并在后续对 Hi-Z Mip -1 ；如果 Hi-Z Mip 不为 0 且反射光线深度值小于物体表面深度, 则保持当前步长并在后续对 Hi-Z Mip -1 ；否则的话, 增大步长, 使用 `distWall` 作为下一次步长, 并在后续对 Hi-Z Mip +1 ：

```hlsl
// 如果交点在几何体表面, 则减小步长, 使用 distFloor 作为下一次步进的步长
// 如果 Hi-Z Mip 不为 0 且反射光线深度值小于物体表面深度, 则保持当前步长并在后续对 Hi-Z Mip -1
// 否则, 增大步长, 使用 distWall 作为下一次步长, 并在后续对 Hi-Z Mip +1
t = hitFloor ? distFloor : (((mipLevel != 0) && belowFloor) ? t : distWall);

// 交点在几何体表面, 或者在几何体表面以下, 则下次步进采样低 1 级的 Hi-Z , 否则下次步进采样高 1 级的 Hi-Z
mipLevel += (hitFloor || belowFloor) ? -1 : 1;
mipLevel  = clamp(mipLevel, 0, maxMipLevel);
```

上面就是关于光线步进的核心算法的解析了。最后，就是生成出反射光照纹理了，根据检测到交点的反射光线终点，计算出当前帧的屏幕 `uv` 坐标，并投影到上一帧的屏幕 `uv` 坐标，采样上一帧的颜色缓冲，得到最终的反射光照纹理：

```hlsl
if (hit)
{
    // 根据最终反射光线屏幕空间坐标计算 uv 坐标
    float2 hitPositionNDC = floor(rayPos.xy) * _ScreenSize.zw + (0.5 * _ScreenSize.zw);

    if (max(hitPositionNDC.x, hitPositionNDC.y) == 0.0)
    {
        // Miss.
        _SsrLightingTexture[positionSS] = float4(0.0, 0.0, 0.0, 0.0);
        return;
    }

    // 重投影到上一帧的屏幕 uv 坐标
    float4 q = mul(_HistoryReprojection, float4(hitPositionNDC, deviceDepth, 1.0));
    float2 historyUV = (q.xy * (1.0 / q.w)) * 0.5 + 0.5;

    if (any(historyUV < float2(0.0, 0.0)) || any(historyUV > float2(1.0, 1.0)))
    {
        // Off-Screen.
        _SsrLightingTexture[positionSS] = float4(0.0, 0.0, 0.0, 0.0);
        return;
    }

    // 采样上一帧的颜色缓冲
    float3 color = _SsrHistoryColorTexture.SampleLevel(sampler_LinearClamp, historyUV, 0).rgb;

    // 检查上一帧中的颜色缓冲中无效的值
    uint3 intCol = asuint(color);
    bool isPosFin = Max3(intCol.r, intCol.g, intCol.b) < 0x7F800000;

    // 对处于屏幕边缘的反射做过度, 避免反射结果贴图直接被截断
    const float screenFadeDistance = 0.1;
    const float _SsrEdgeFadeRcpLength = min(1.0 / screenFadeDistance, FLT_MAX);
    float opacity = EdgeOfScreenFade(historyUV, _SsrEdgeFadeRcpLength);

    color = isPosFin ? color : 0.0;
    opacity = isPosFin ? opacity : 0.0;

    _SsrLightingTexture[positionSS] = float4(color, 1.0) * opacity;
}
else
{
    _SsrLightingTexture[positionSS] = float4(0.0, 0.0, 0.0, 0.0);
}
```

对于处于屏幕边缘的反射结果，我们根据它的屏幕 uv 坐标做一下平滑的淡化，并且将淡化值 `fade` 存储在反射光照纹理的 `a` 通道，后面利用这个淡化值将反射光照结果与环境反射结果进行混合。 `EdgeOfScreenFade` 函数如下：

```hlsl
// Performs fading at the edge of the screen.
float EdgeOfScreenFade(float2 coordNDC, float fadeRcpLength)
{
    float2 coordCS = coordNDC * 2 - 1;
    float2 t = Remap10(abs(coordCS), fadeRcpLength, fadeRcpLength);
    return Smoothstep01(t.x) * Smoothstep01(t.y);
}
```

最终的反射光照纹理如下图：

![08_ssr_lighting](/assets/images/2024/2024-11-12-ScreenSpaceReflections/08_ssr_lighting.jpg)

### 2.4. 高斯模糊

为了模拟不同粗糙度的表面的反射，我们对生成出的反射光照纹理进行模糊处理，和生成 Hi-Z 缓冲类似，也是生成反射光照纹理的不同 Mipmap 层级，这里直接参考了 HDRP 中 Bloom 的降采样模糊方案。完整的代码就不写出来了，这里主要是贴一下添加了注释的 kernel 函数吧：

```hlsl
[numthreads(8, 8, 1)]
void CSMain(uint2 groupId : SV_GroupID, uint2 groupThreadId : SV_GroupThreadID, uint2 dispatchThreadId : SV_DispatchThreadID)
{
    // 因为每个线程组 group 包含 8x8 个线程, 每个线程代表 RT 上的一个像素
    // groupId.x 表示的是 RT 在 u 方向上的 线程组 index, groupId.y 表示的是 RT 在 v 方向上的 线程组 index
    // groupThreadId.xy 表示的是每个线程组内每个像素在 u 和 v 方向上的 index, 范围是 [0,7]

    // 下面这行代码的意思是将 RT 中的像素分成一个个 2x2 的像素块, 返回的是这个 2x2 像素块的左下角的像素的在屏幕范围上的 index
    int2 threadUL = (groupThreadId << 1) + (groupId << 3) - 4;
    float2 offset = float2(threadUL);

    // 2x2 像素块的 4 个像素的坐标
    float2 c00 = (offset + 0.5) * _DestinationSize.xy;
    float2 c10 = (offset + float2(1.0, 0.0) + 0.5) * _DestinationSize.xy;
    float2 c01 = (offset + float2(0.0, 1.0) + 0.5) * _DestinationSize.xy;
    float2 c11 = (offset + float2(1.0, 1.0) + 0.5) * _DestinationSize.xy;

    // 通过双线性过滤采样上 1 级的 Mip
    float4 p00 = _SourcePyramid.SampleLevel(sampler_LinearClamp, c00, 0.0);
    float4 p10 = _SourcePyramid.SampleLevel(sampler_LinearClamp, c10, 0.0);
    float4 p01 = _SourcePyramid.SampleLevel(sampler_LinearClamp, c01, 0.0);
    float4 p11 = _SourcePyramid.SampleLevel(sampler_LinearClamp, c11, 0.0);

    // Store the 4 downsampled pixels in LDS
    // LDS 是指 本地数据共享(Local Data Share), LDS 是一种用于在同一线程组内的着色器线程之间共享数据的高速存储器, 它通常用于需要在多个线程之间快速交换数据的场合
    // 将采样的每 2 个像素的颜色存储到 组共享内存(Group Shared Memory) 中, 这 2 个像素的每个通道从 32 位转换为 16 位, 并分别存储在一个 32 位的 uint 的低 16 位 和 高 16 位中
    // 每个线程组有 64 个线程, 每个线程采样了 4 个像素, 会将 2 个像素的同一个通道存储在一个 uint 中, 所以一共需要 (64 * 4 / 2 = 128) 个 uint
    uint destIdx = groupThreadId.x + (groupThreadId.y << 4u);
    Store2Pixels(destIdx , p00, p10);
    Store2Pixels(destIdx + 8u, p01, p11);

    // 两个作用:
    // 1. 确保所有内存操作（读或写）都已经完成，并且结果对同一线程组中的所有线程可见
    // 2. 阻塞当前线程，直到同一线程组中的所有线程都到达此同步点
    GroupMemoryBarrierWithGroupSync();

    // 横向模糊
    uint row = groupThreadId.y << 4u;
    BlurHorizontally(row + (groupThreadId.x << 1u), row + groupThreadId.x + (groupThreadId.x & 4u));

    GroupMemoryBarrierWithGroupSync();

    // 纵向模糊, 并将最终的结果输出
    BlurVertically(dispatchThreadId, (groupThreadId.y << 3u) + groupThreadId.x);
}
```

### 2.5. 采样反射光照纹理并与环境反射光照混合

现在要做的就是在前向光照计算阶段，将 SSR 的结果应用到最终的光照结果中了。我们将反射光照纹理中的 `a` 通道的值作为 SSR 的权重与环境反射混合，并且根据粗糙度计算采样反射光照纹理的 Mipmap 层级，主要逻辑如下：

```hlsl
// Ssr lighting
float mipLevel = lerp(0, 8, brdfData.perceptualRoughness);  // simple lerp
half4 ssrLighting = SAMPLE_TEXTURE2D_LOD(_SsrLightingTexture, sampler_SsrLightingTexture, inputData.normalizedScreenSpaceUV, mipLevel);
float envWeight = 1.0 - ssrLighting.a;
iblFr = iblFr * envWeight + (E * ssrLighting);
```

最终的 SSR 效果如下：

![09_ssr_result](/assets/images/2024/2024-11-12-ScreenSpaceReflections/09_ssr_result.jpeg)

## 3. 参考

- <a id="ref1"></a>[1] "Hi-Z Screen-Space Cone-Traced Reflections", Yasin Uludag, GPU Pro 5
- <a id="ref2"></a>[2] [Improved normal reconstruction from depth](https://wickedengine.net/2019/09/improved-normal-reconstruction-from-depth/)
- <a id="ref3"></a>[3] [Accurate Normal Reconstruction from Depth Buffer](https://atyuwen.github.io/posts/normal-reconstruction/#fn:1)
- [4] [Unity HDRP](https://docs.unity3d.com/Packages/com.unity.render-pipelines.high-definition@17.0/manual/index.html)
