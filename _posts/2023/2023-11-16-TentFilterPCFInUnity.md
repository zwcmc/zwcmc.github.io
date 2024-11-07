---
layout: post
title:  "Unity 中的 Tent Filter PCF"
date:   2023-11-16 16:16:00 +800
category: Unity
---

- [1. Bilinear PCF](#1-bilinear-pcf)
- [2. 更大的 PCF 内核](#2-更大的-pcf-内核)
- [3. 双线性插值（Bilinear Interpolation）](#3-双线性插值bilinear-interpolation)
- [4. Unity 中的 Tent Filter](#4-unity-中的-tent-filter)
  - [4.1. `SampleShadow_GetTriangleTexelArea`](#41-sampleshadow_gettriangletexelarea)
  - [4.2. `SampleShadow_GetTexelAreas_Tent_3x3`](#42-sampleshadow_gettexelareas_tent_3x3)
  - [4.3. `SampleShadow_GetTexelWeights_Tent_3x3`](#43-sampleshadow_gettexelweights_tent_3x3)
  - [4.4. `SampleShadow_ComputeSamples_Tent_3x3`](#44-sampleshadow_computesamples_tent_3x3)
    - [4.4.1. 计算 Group 的权重](#441-计算-group-的权重)
    - [4.4.2. 计算 Group 的采样坐标](#442-计算-group-的采样坐标)
    - [4.4.3. `SampleShadow_ComputeSamples_Tent_3x3` 产生的阴影效果](#443-sampleshadow_computesamples_tent_3x3-产生的阴影效果)
  - [4.5. `SampleShadow_ComputeSamples_Tent_5x5` 与 `SampleShadow_ComputeSamples_Tent_7x7`](#45-sampleshadow_computesamples_tent_5x5-与-sampleshadow_computesamples_tent_7x7)
- [5. 参考](#5-参考)

PCF（Percentage Closer Filtering）是一种用于实时渲染中的软阴影生成的基本算法。它通过对阴影贴图中的多个样本进行采样并平均化结果，从而产生柔和的阴影边缘效果。

## 1. Bilinear PCF

Bilinear PCF 是最基础的 PCF 算法。通过采样点周围 `2x2` 的 4 个纹素，并将 4 次深度测试比较的结果通过 **双线性插值** 的方式进行混合，得到最终的阴影结果。在早期时代，我们需要手动进行 4 次 Point Filter 采样并进行深度测试比较，然后将比较的结果插值混合。现代的硬件直接提供了此算法的支持，只需要一次双线性过滤的特殊采样（例如 HLSL 中提供的函数：[SampleCmpLevelZero](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-to-samplecmplevelzero)）就能得到最终的阴影结果。

下图展示了 Bilinear PCF 的阴影效果：

![00_bilinear_pcf](/assets/images/2023/2023-11-16-TentFilterPCFInUnity/00_bilinear_pcf.jpg)

可以看出来， Bilinear PCF 使得阴影边缘有微弱的过渡，不过锯齿感还是比较明显。

## 2. 更大的 PCF 内核

想要降低阴影边缘的锯齿感，使其过渡更加平滑，首先想到的就是扩大 PCF 的内核，来使阴影的边缘更加的“模糊”。

![01_pcf_3x3](/assets/images/2023/2023-11-16-TentFilterPCFInUnity/01_pcf_3x3.jpeg)

如上图所示，将蓝色给定采样点的坐标往 4 个对角方向各自偏移 0.5 个纹素，并在偏移后的 4 个红色点上分别使用 Bilinear PCF 采样 1 次并将结果平均。该算法的代码如下：

```hlsl
// 4 tap bilinear PCF 3x3
real SampleShadow_PCF_Bilinear_4Tap_3x3(TEXTURE2D_SHADOW_PARAM(shadowMap, sampler_shadowMap), float3 coord)
{
    float2 offset = _MainLightShadowmapTexture_TexelSize.xy * 0.5;

    real4 result;
    result.x = SampleShadow_PCF_Bilinear(shadowMap, sampler_shadowMap, float3(coord.xy + int2(-1, -1) * offset, coord.z));
    result.y = SampleShadow_PCF_Bilinear(shadowMap, sampler_shadowMap,float3(coord.xy + int2(-1, 1) * offset, coord.z));
    result.z = SampleShadow_PCF_Bilinear(shadowMap, sampler_shadowMap,float3(coord.xy + int2(1, 1) * offset, coord.z));
    result.w = SampleShadow_PCF_Bilinear(shadowMap, sampler_shadowMap,float3(coord.xy + int2(1, -1) * offset, coord.z));
    return real(dot(result, 0.25));
}
```

通过这种方法，我们将 1 次 Bilinear PCF 采样覆盖的 `2x2` 范围内的 4 个纹素扩大到 4 次 Bilinear PCF 采样覆盖的 `3x3` 范围内的 9 个纹素。使用该算法产生的阴影如下图：

![02_4tap_bilinear_pcf](/assets/images/2023/2023-11-16-TentFilterPCFInUnity/02_4tap_bilinear_pcf.jpg)

从上图中可以看到，虽然阴影“模糊”的范围变大了，但是锯齿感还是非常的明显。

## 3. 双线性插值（Bilinear Interpolation）

首先我们知道，阴影贴图中的纹素存储的是一个个 **离散的** 深度值，在使用 Bilinear PCF 对阴影贴图进行采样时，其实是将采样点周围的 4 个纹素看作 4 个点，并且根据采样点与周围 4 个纹素之间的距离来进行线性插值得到采样的结果。

下面有一个一维的例子：

![03_discrete_value](/assets/images/2023/2023-11-16-TentFilterPCFInUnity/03_discrete_value.jpeg)

对于离散的取值 ABC ，使用线性插值后会变成：

![04_linear_interpolation](/assets/images/2023/2023-11-16-TentFilterPCFInUnity/04_linear_interpolation.jpeg)

可以看到，通过线性插值，虽然 AB 和 AC 之间的取值过渡变得平滑，但是取值在 A 点却是突然的跳变，这说明线 **性插值仅仅是一阶连续的，其导数是不连续的** 。下图展示了一个例子，对于离散的信号，分别使用最近相邻插值（Nearest-neighbor interpolation）和线性插值（Linear interpolation）对其进行采样的结果：

![05_sampling](/assets/images/2023/2023-11-16-TentFilterPCFInUnity/05_sampling.jpg)

通过上图可以看出来， **线性插值本质上是一个离散的 Tent Filter ，它将每个纹素看作一个点，只计算了这个点相对于纹理坐标变化的权重，而不是根据纹素的整个面积来计算其所占据的权重** 。

## 4. Unity 中的 Tent Filter

在 Unity 的 `ShadowSamplingTent.hlsl` 源码中， Unity 通过计算线性插值的 Tent Filter 在每个纹素所占据的面积来计算这个纹素的权重。通过这种方式，将离散的 Tent Filter 转换为一个连续的 Tent Filter ，通过此连续的 Tent Filter 来对阴影贴图进行采样滤波，就可以得到一个平滑过渡的阴影边缘。

### 4.1. `SampleShadow_GetTriangleTexelArea`

```hlsl
// Assuming a isoceles right angled triangle of height "triangleHeight" (as drawn below).
// This function return the area of the triangle above the first texel.
//
// |\      <-- 45 degree slop isosceles right angled triangle
// | \
// ----    <-- length of this side is "triangleHeight"
// _ _ _ _ <-- texels
real SampleShadow_GetTriangleTexelArea(real triangleHeight)
{
    return triangleHeight - 0.5;
}
```

该函数计算的是等腰直角三角形在第一个纹素范围中的面积。如下图所示的等腰直角三角形：

![06_isoceles_right_angled_triangle](/assets/images/2023/2023-11-16-TentFilterPCFInUnity/06_isoceles_right_angled_triangle.png)

假设它的直角边边长为 `triangleHeight = h` ，那么此等腰直角三角形在第一个纹素范围中的面积（也就是上图中蓝色的区域）可以通过相似三角形很容易求出来：

$$
\begin{align*}
    S &= \frac{h^2}{2} - \frac{(h-1)^2}{2} \\
    &= \frac{h^2 - (h^2 - 2h + 1)}{2} \\
    &= \frac{2h-1}{2} \\
    &= h - 0.5
\end{align*}
$$

### 4.2. `SampleShadow_GetTexelAreas_Tent_3x3`

```hlsl
// Assuming a isoceles triangle of 1.5 texels height and 3 texels wide lying on 4 texels.
// This function return the area of the triangle above each of those texels.
//    |    <-- offset from -0.5 to 0.5, 0 meaning triangle is exactly in the center
//   / \   <-- 45 degree slop isosceles triangle (ie tent projected in 2D)
//  /   \
// _ _ _ _ <-- texels
// X Y Z W <-- result indices (in computedArea.xyzw and computedAreaUncut.xyzw)
void SampleShadow_GetTexelAreas_Tent_3x3(real offset, out real4 computedArea, out real4 computedAreaUncut)
{
    // Compute the exterior areas
    real offset01SquaredHalved = (offset + 0.5) * (offset + 0.5) * 0.5;
    computedAreaUncut.x = computedArea.x = offset01SquaredHalved - offset;
    computedAreaUncut.w = computedArea.w = offset01SquaredHalved;

    // Compute the middle areas
    // For Y : We find the area in Y of as if the left section of the isoceles triangle would
    // intersect the axis between Y and Z (ie where offset = 0).
    computedAreaUncut.y = SampleShadow_GetTriangleTexelArea(1.5 - offset);
    // This area is superior to the one we are looking for if (offset < 0) thus we need to
    // subtract the area of the triangle defined by (0,1.5-offset), (0,1.5+offset), (-offset,1.5).
    real clampedOffsetLeft = min(offset,0);
    real areaOfSmallLeftTriangle = clampedOffsetLeft * clampedOffsetLeft;
    computedArea.y = computedAreaUncut.y - areaOfSmallLeftTriangle;

    // We do the same for the Z but with the right part of the isoceles triangle
    computedAreaUncut.z = SampleShadow_GetTriangleTexelArea(1.5 + offset);
    real clampedOffsetRight = max(offset,0);
    real areaOfSmallRightTriangle = clampedOffsetRight * clampedOffsetRight;
    computedArea.z = computedAreaUncut.z - areaOfSmallRightTriangle;
}
```

该函数是计算一个底为 3 个纹素，高为 1.5 个纹素的等腰直角三角形的 Tent Filter 在 4 个纹素中所占的面积分别是多少，也就是这个 Tent Filter 在其覆盖的 4 个纹素中的权重。

需要注意的是，这里的计算是一个一维的，而我们在计算纹理采样时，需要分别计算 `u` 方向上的 4 个权重 $w_{u1}, w_{u2}, w_{u3}, w_{u4}$ 以及 `v` 方向上的 4 个权重 $w_{v1}, w_{v2}, w_{v3}, w_{v4}$ ，并将它们相乘，也就得到了每个纹素的权重。

下图展示了一个在 `u` 方向上计算 Tent Filter 所覆盖的 4 个纹素的例子，要计算的也就是下图中 `x,y,z,w` 4 个区域的面积：

![07_isoceles_triangle](/assets/images/2023/2023-11-16-TentFilterPCFInUnity/07_isoceles_triangle.jpeg)

其中 `offset` 指的是 Tent Filter 顶点相对于坐标原点的偏移量，而此坐标原点是距离采样点最近的整数纹素坐标的位置，所以 `offset` 的取值范围应该是 $[-0.5, 0.5]$ 。计算的代码如下：

```hlsl
// 将纹理坐标 [0,1] 转换到纹素坐标 [0, width] / [0, height]
real2 tentCenterInTexelSpace = coord.xy * shadowMapTexture_TexelSize.zw;
// 计算出最近的纹素坐标
real2 centerOfFetchesInTexelSpace = floor(tentCenterInTexelSpace + 0.5);
// 计算相对于最近纹素坐标的偏移 offset ，offset 的范围应该是 [-0.5, 0.5]
real2 offsetFromTentCenterToCenterOfFetches = tentCenterInTexelSpace - centerOfFetchesInTexelSpace;
```

`w` 与 `x` 区域的面积都很好计算：

```hlsl
w = (offset + 0.5) * (offset + 0.5) * 0.5;
x = (0.5 - offset) * (0.5 - offset) * 0.5 = w - offset;  // 简化为 w - offset
```

对于 `y` 的面积，当 `offset` 为正时，如下图所示：

![08_offset_positive_y](/assets/images/2023/2023-11-16-TentFilterPCFInUnity/08_offset_positive_y.jpg)

此时 `y` 的面积也就是 `x` 和 `y` 区域组成的小的等腰直角三角形在第一个纹素中的面积，可以直接通过函数 `SampleShadow_GetTriangleTexelArea` 来计算，也就是：

```hlsl
y = SampleShadow_GetTriangleTexelArea(1.5 - offset);
```

而当 `offset` 为负时，此时通过此函数计算的是一个更大的等腰直角三角形的面积，如下图所示：

![09_offset_negative_y](/assets/images/2023/2023-11-16-TentFilterPCFInUnity/09_offset_negative_y.jpg)

所以还需要减去右上角小的等腰直角三角形的面积，这个小的等腰直角三角形的面积可以表示为：

```hlsl
areaMin = abs(offset) * (2.0 * abs(offset)) * 0.5;
```

也就是函数 `SampleShadow_GetTexelAreas_Tent_3x3` 中的如下代码：

```hlsl
real clampedOffsetLeft = min(offset,0);
real areaOfSmallLeftTriangle = clampedOffsetLeft * clampedOffsetLeft;
computedArea.y = computedAreaUncut.y - areaOfSmallLeftTriangle;
```

`z` 区域的面积计算与求 `y` 区域的面积的思路一样，这里就不再赘述。

### 4.3. `SampleShadow_GetTexelWeights_Tent_3x3`

```hlsl
// Assuming a isoceles triangle of 1.5 texels height and 3 texels wide lying on 4 texels.
// This function return the weight of each texels area relative to the full triangle area.
void SampleShadow_GetTexelWeights_Tent_3x3(real offset, out real4 computedWeight)
{
    real4 dummy;
    SampleShadow_GetTexelAreas_Tent_3x3(offset, computedWeight, dummy);
    computedWeight *= 0.44444;//0.44 == 1/(the triangle area)
}
```

最后需要对计算出的权重归一化，因为这 4 个权重是根据底为 3 个纹素，高为 1.5 个纹素的等腰直角三角形计算出来的，所以最后需要除以这个等腰直角三角形的面积来保证能量守恒。

### 4.4. `SampleShadow_ComputeSamples_Tent_3x3`

```hlsl
// 3x3 Tent filter (45 degree sloped triangles in U and V)
void SampleShadow_ComputeSamples_Tent_3x3(real4 shadowMapTexture_TexelSize, real2 coord, out real fetchesWeights[4], out real2 fetchesUV[4])
{
    // tent base is 3x3 base thus covering from 9 to 12 texels, thus we need 4 bilinear PCF fetches
    real2 tentCenterInTexelSpace = coord.xy * shadowMapTexture_TexelSize.zw;
    real2 centerOfFetchesInTexelSpace = floor(tentCenterInTexelSpace + 0.5);
    real2 offsetFromTentCenterToCenterOfFetches = tentCenterInTexelSpace - centerOfFetchesInTexelSpace;

    // find the weight of each texel based
    real4 texelsWeightsU, texelsWeightsV;
    SampleShadow_GetTexelWeights_Tent_3x3(offsetFromTentCenterToCenterOfFetches.x, texelsWeightsU);
    SampleShadow_GetTexelWeights_Tent_3x3(offsetFromTentCenterToCenterOfFetches.y, texelsWeightsV);

    // each fetch will cover a group of 2x2 texels, the weight of each group is the sum of the weights of the texels
    real2 fetchesWeightsU = texelsWeightsU.xz + texelsWeightsU.yw;
    real2 fetchesWeightsV = texelsWeightsV.xz + texelsWeightsV.yw;

    // move the PCF bilinear fetches to respect texels weights
    real2 fetchesOffsetsU = texelsWeightsU.yw / fetchesWeightsU.xy + real2(-1.5,0.5);
    real2 fetchesOffsetsV = texelsWeightsV.yw / fetchesWeightsV.xy + real2(-1.5,0.5);
    fetchesOffsetsU *= shadowMapTexture_TexelSize.xx;
    fetchesOffsetsV *= shadowMapTexture_TexelSize.yy;

    real2 bilinearFetchOrigin = centerOfFetchesInTexelSpace * shadowMapTexture_TexelSize.xy;
    fetchesUV[0] = bilinearFetchOrigin + real2(fetchesOffsetsU.x, fetchesOffsetsV.x);
    fetchesUV[1] = bilinearFetchOrigin + real2(fetchesOffsetsU.y, fetchesOffsetsV.x);
    fetchesUV[2] = bilinearFetchOrigin + real2(fetchesOffsetsU.x, fetchesOffsetsV.y);
    fetchesUV[3] = bilinearFetchOrigin + real2(fetchesOffsetsU.y, fetchesOffsetsV.y);

    fetchesWeights[0] = fetchesWeightsU.x * fetchesWeightsV.x;
    fetchesWeights[1] = fetchesWeightsU.y * fetchesWeightsV.x;
    fetchesWeights[2] = fetchesWeightsU.x * fetchesWeightsV.y;
    fetchesWeights[3] = fetchesWeightsU.y * fetchesWeightsV.y;
}
```

上面的 [SampleShadow_GetTexelWeights_Tent_3x3](#43-sampleshadow_gettexelweights_tent_3x3) 等函数计算了在一维情况下，一个底为 3 个纹素，高为 1.5 个纹素的 Tent Filter 覆盖的 4 个纹素中每个纹素的权重。而图片是二维的，要想得到范围内每个纹素的权重，那么就先根据采样点的二维偏移坐标，分别求出 `u` 和 `v` 方向上的一维的 Tent Filter 的权重，然后把两者相乘就得到了每个纹素的权重了。

因为 Bilinear PCF 可以同时对 `2x2` 的纹素做采样比较，因此我们现在需要把纹素分成 `2x2` 的一组（Group），这样每 `2x2` 个纹素只需要一次采样。以 `SampleShadow_ComputeSamples_Tent_3x3` 为例， Tent Filter 一共覆盖 `3x3` 个纹素，所以一共需要 4 次 Bilinear PCF 采样，也就是将纹素分成 4 个 Group 。

#### 4.4.1. 计算 Group 的权重

首先，每个 Group 中纹素的权重应该是所有纹素的权重之和。所以，对于 `u` 和 `v` 方向上分别计算出的 4 个纹素权重，将其分别分成 `x+y` 和 `z+w` 这两组：

```hlsl
// 4 个纹素的权重 x,y,z,w ，将其分为两组 (x+y) 和 (z+w)
real2 fetchesWeightsU = texelsWeightsU.xz + texelsWeightsU.yw;
real2 fetchesWeightsV = texelsWeightsV.xz + texelsWeightsV.yw;
```

在 `u` 方向上的两组权重是 `fetchesWeightsU.x` 和 `fetchesWeightsU.y` ，在 `v` 方向上的两组权重是 `fetchesWeightsV.x` 和 `fetchesWeightsV.y` 。将它们两两相乘就得到了 4 次 Bilinear PCF 采样的权重：

```hlsl
// 计算 4 次 Bilinear PCF 采样的权重
fetchesWeights[0] = fetchesWeightsU.x * fetchesWeightsV.x;
fetchesWeights[1] = fetchesWeightsU.y * fetchesWeightsV.x;
fetchesWeights[2] = fetchesWeightsU.x * fetchesWeightsV.y;
fetchesWeights[3] = fetchesWeightsU.y * fetchesWeightsV.y;
```

#### 4.4.2. 计算 Group 的采样坐标

现在我们有了在 `u` 和 `v` 方向上 Tent Filter 分别覆盖的 4 个纹素的权重，那么根据线性插值可以计算出 Group 的采样坐标。下面以一维的 `x+y` 这一组为例来说明，在 Group 内部， `x` 的坐标为 `-1.5` ， `y` 的坐标为 `-0.5` ，根据它们的权重 `x` 和 `y` 可以计算出偏移量为：

```hlsl
offset = -1.5 + y / (x + y);
```

下面的代码是根据权重计算 4 次 Bilinear PCF 的采样坐标：

```hlsl
// 分别计算 u 和 v 方向上 2 组 2x2 Group 的坐标偏移
real2 fetchesOffsetsU = texelsWeightsU.yw / fetchesWeightsU.xy + real2(-1.5,0.5);
real2 fetchesOffsetsV = texelsWeightsV.yw / fetchesWeightsV.xy + real2(-1.5,0.5);
fetchesOffsetsU *= shadowMapTexture_TexelSize.xx;
fetchesOffsetsV *= shadowMapTexture_TexelSize.yy;

// 以 Tent Filter 坐标原点为基准进行偏移
real2 bilinearFetchOrigin = centerOfFetchesInTexelSpace * shadowMapTexture_TexelSize.xy;

// 计算 4 次 Bilinear PCF 采样的坐标
fetchesUV[0] = bilinearFetchOrigin + real2(fetchesOffsetsU.x, fetchesOffsetsV.x);
fetchesUV[1] = bilinearFetchOrigin + real2(fetchesOffsetsU.y, fetchesOffsetsV.x);
fetchesUV[2] = bilinearFetchOrigin + real2(fetchesOffsetsU.x, fetchesOffsetsV.y);
fetchesUV[3] = bilinearFetchOrigin + real2(fetchesOffsetsU.y, fetchesOffsetsV.y);
```

#### 4.4.3. `SampleShadow_ComputeSamples_Tent_3x3` 产生的阴影效果

![10_tent_filter_pcf_3x3](/assets/images/2023/2023-11-16-TentFilterPCFInUnity/10_tent_filter_pcf_3x3.jpg)

上图展示了同样是 4 次 Bilinear PCF 采样，使用离散的 Tent Filter 和使用连续的 Tent Filter 产生的阴影的对比。可以看到，阴影的过渡效果更加平滑了。

### 4.5. `SampleShadow_ComputeSamples_Tent_5x5` 与 `SampleShadow_ComputeSamples_Tent_7x7`

`5x5` 的 Tent Filter 可以覆盖 `5x5` ～ `6x6` 范围内的纹素。如下图所示：

![11_tent_filter_pcf_5x5](/assets/images/2023/2023-11-16-TentFilterPCFInUnity/11_tent_filter_pcf_5x5.png)

和 `3x3` 的 Tent Filter 类似，我们需要计算 `Ax,Ay,Az,Bx,By,Bz` 这 6 个纹素下 Tent Filter 所占的面积。由于等腰直角三角形的优良性质，我们可以直接利用 `SampleShadow_GetTexelAreas_Tent_3x3` 的计算结果，然后补足剩余的面积来得到 `SampleShadow_GetTexelWeights_Tent_5x5` 的结果。

上图中，三个红、蓝、绿小三角形都是和 `3x3` Tent Filter 相同的等腰直角三角形。以 `Az` 为例，其面积等于蓝色三角形 `y` 区域（ `3x3` Tent Filter 等腰直角三角形的 `x,y,z,w` 四个区域中的 `y` 区域）的面积加上底下的一个面积为 1 的正方形的面积。依此类推，可以求出剩余的区域的面积。

`7x7` 的 Tent Filter 的情况类似，也是先计算 `SampleShadow_GetTexelAreas_Tent_3x3` ，然后利用 `3x3` 的 Tent Filter 来推导 `7x7` 的 Tent Filter 。

## 5. 参考

- [1] [阴影的PCF采样优化算法](https://github.com/wlgys8/SRPLearn/wiki/PCFSampleOptimize)
- [2] [Upsampling and Interpolation](https://www.cs.toronto.edu/~guerzhoy/320/lec/upsampling.pdf)
- [3] [unity的ShadowSamplingTent源码](https://www.legendsmb.com/2022/10/13/unity-PCF-sourcecode/)
