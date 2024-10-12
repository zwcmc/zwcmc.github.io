---
layout: post
title:  "阴影贴图的原理以及相关的软阴影技术"
date:   2024-08-21 16:16:00
category: Rendering
---

- [(1) 阴影贴图（Shadow mapping）原理](#1-阴影贴图shadow-mapping原理)
  - [1.1 阴影贴图的一些常见问题](#11-阴影贴图的一些常见问题)
    - [1.1.1 透视锯齿（Perspective Aliasing）](#111-透视锯齿perspective-aliasing)
    - [1.1.2 投影锯齿（Projective Aliasing）](#112-投影锯齿projective-aliasing)
    - [1.1.3 阴影暗斑（Shadow Acne）](#113-阴影暗斑shadow-acne)
    - [1.1.4 彼得潘效应（Peter Panning）](#114-彼得潘效应peter-panning)
  - [1.2 改善阴影贴图的一些技术](#12-改善阴影贴图的一些技术)
    - [1.2.1 基于斜率的深度偏移（Slope-Scale Depth Bias）](#121-基于斜率的深度偏移slope-scale-depth-bias)
    - [1.2.2 提高深度缓冲区的精度或提高阴影贴图的分辨率](#122-提高深度缓冲区的精度或提高阴影贴图的分辨率)
    - [1.2.3 计算合适的光源投影大小（Calculating a Tight Projection）](#123-计算合适的光源投影大小calculating-a-tight-projection)
    - [1.2.4 计算适合的光源投影的近平面和远平面（Calculating the Near Plane and Far Plane）](#124-计算适合的光源投影的近平面和远平面calculating-the-near-plane-and-far-plane)
    - [1.2.5 以纹素大小的增量来移动光源（Moving the Light in Texel-Sized Increments）](#125-以纹素大小的增量来移动光源moving-the-light-in-texel-sized-increments)
- [(2) 级联阴影（Cascaded Shadow Maps）](#2-级联阴影cascaded-shadow-maps)
  - [2.1 分割摄像机视锥体的方法](#21-分割摄像机视锥体的方法)
  - [2.2 摄像机视锥体覆盖的场景区域 Z 轴范围的大小影响级联区间的划分](#22-摄像机视锥体覆盖的场景区域-z-轴范围的大小影响级联区间的划分)
  - [2.3 光源与摄像机的方向之间的关系会影响级联之间的重叠](#23-光源与摄像机的方向之间的关系会影响级联之间的重叠)
  - [2.4 创建子视锥体](#24-创建子视锥体)
  - [2.5 渲染级联阴影贴图](#25-渲染级联阴影贴图)
  - [2.6 采样级联阴影贴图](#26-采样级联阴影贴图)
- [(3) 硬阴影（Hard Shadows）与软阴影（Soft Shadows）](#3-硬阴影hard-shadows与软阴影soft-shadows)
  - [3.1 Percentage Closer Filtering（PCF）](#31-percentage-closer-filteringpcf)
    - [3.1.1 双线性 PCF （Bilinear PCF）](#311-双线性-pcf-bilinear-pcf)
    - [3.1.2 更大内核的 PCF](#312-更大内核的-pcf)
- [(4) 参考](#4-参考)

## (1) 阴影贴图（Shadow mapping）原理

阴影贴图（Shadow mapping）是实时渲染中常用的一种用以渲染场景阴影的技术，它最初是由 Lance Williams 于 1978 年在 “Casting curved shadows on curved surfaces” 这篇论文中提出的。阴影贴图的原理如下：

- **生成阴影贴图**：首先，从光源的视角渲染一次场景，输出一张深度贴图（Depth Map），这张深度图存储的是场景中每个可见像素到光源的距离，也就是这个可见像素的深度值。生成的阴影贴图如下：

  ![00_shadow_map](/assets/images/2024/2024-08-21-ShadowMapping/00_shadow_map.png)

- **应用阴影贴图**：从摄像机的视角渲染场景，在渲染场景时，将场景中的像素从世界空间坐标转换到光源的剪裁空间（Clip Space），并将此像素的深度值与深度贴图中存储的深度值做比较，如果此像素的深度值大于深度贴图中存储的深度值（也就是以光源为视角渲染场景时，看不到这个像素），则意味着此像素在阴影中。下图形象的展示了阴影贴图的原理：

  ![01_use_shadow_map](/assets/images/2024/2024-08-21-ShadowMapping/01_use_shadow_map.png)

  在上图中，首先在渲染阴影贴图时，阴影贴图中存储了可见像素 `A` 和 `C` 距离光源的深度 $\text{z}_a^{\*}$ 和 $\text{z}_c^{\*}$ ，在后续渲染场景的过程中，将像素 `A` 和像素 `B` 从世界坐标转换到光源的裁剪空间后，将像素 `A` 和像素 `B` 的在光源裁剪空间中的深度值和深度贴图中存储的深度值做对比，发现像素 `A` 的深度值 $\text{z}_a$ 与阴影贴图中存储的深度值 $\text{z}_a^{\*}$ 相等，所以说明像素 `A` 不在阴影中（光源照到了像素 `A` ），而像素 `B` 的深度值 $\text{z}_b$ 大于阴影贴图中存储的深度值 $\text{z}_c^{\*}$ ，说明像素 `B` 在阴影中。

需要注意的是，不同的光源使用不同的投影矩阵：

- 对于方向光（Directional Light），它是一个方向，使用正交的投影矩阵（Orthographic Projection）。
- 对于聚光灯光源（Spot Light），使用透视的投影矩阵（Perspective Projection）。
- 而对于点光源（Point Light），需要使用透视的投影矩阵，而且要渲染点光源的正上、正下，正左、正右、正前和正后 6 个方向上的阴影贴图。一般使用一个立方体贴图才存储 6 个方向的阴影贴图。

### 1.1 阴影贴图的一些常见问题

#### 1.1.1 透视锯齿（Perspective Aliasing）

方向光通常模拟太阳光，单个方向光即可照亮整个场景，这意味着方向光的阴影贴图需要覆盖摄像机视锥体（Frustum）覆盖的场景的部分。**透视锯齿（Perspective Aliasing）** 问题是指靠近摄像机的阴影贴图像素看起来比远处的像素更大块，如下图所示，靠近摄像机位置的阴影比远处的阴影更大块，这就是透视锯齿的问题：

![02_perspective_aliasing](/assets/images/2024/2024-08-21-ShadowMapping/02_perspective_aliasing.jpg)

出现这个问题的原因是因为在靠近摄像机的位置，像素彼此更靠近，导致许多像素映射到相同的阴影贴图纹素上。为了更形象的说明此问题出现的原因，下面举一个简单的例子，想象一个场景，方向光是从上往下照射，摄像机方向是水平的，如下图所示：

![03_simple_scene_perspective_aliasing](/assets/images/2024/2024-08-21-ShadowMapping/03_simple_scene_perspective_aliasing.png)

那么这个从上往下照射的方向光的阴影贴图覆盖的区域应该是：

![04_simple_scene_shadowmap_cover](/assets/images/2024/2024-08-21-ShadowMapping/04_simple_scene_shadowmap_cover.png)

可以看到，对于摄像机视锥体远端被阴影贴图的 20 个纹素覆盖，而近端仅被 4 个纹素覆盖，但是在屏幕上，两端却显示的是相同的大小（投影变换和透视除法最终会将整个视锥体变换成归一化设备坐标空间），也就是说，对于靠近摄像机的阴影区域，阴影贴图的分辨率比远离摄像机阴影区域的阴影贴图分辨率要低的多。

**级联阴影贴图（CSMs）** 是处理透视锯齿问题的最流行技术，在后面会介绍这个技术。

#### 1.1.2 投影锯齿（Projective Aliasing）

当几何体的切平面与光线方向平行时，会出现 **投影锯齿（Projective Aliasing）** 的问题。下图展示了立方体的侧切面与光线方向平行时出现的投影锯齿问题：

![05_projective_aliasing](/assets/images/2024/2024-08-21-ShadowMapping/05_projective_aliasing.jpg)

用于减轻透视锯齿问题的技术也可以缓解投影锯齿的问题。而且当几何体的切平面与光线方向平行时，也就表示此表面法线和光线方向垂直，在光照计算时可以通过 `NdotL = 0` 来规避投影锯齿的问题。

#### 1.1.3 阴影暗斑（Shadow Acne）

**阴影暗斑（Shadow Acne）** 也叫做 **错误自阴影（Erroneous Self-Shadowing）** 。阴影暗斑的问题是由于阴影贴图的离散性质而引起的，阴影贴图中每个纹素存储的是离散的深度值，而实际场景中几何体的表面是连续的，可能实际场景中几何体表面上的很多个连续的像素采样的是阴影贴图中的同一个纹素，这就导致在做深度对比时，有的像素的深度是大于阴影贴图中的深度，而有些像素的深度是小于阴影贴图中存储的深度，这也就导致了阴影暗斑的问题。下图展示了阴影暗斑的问题：

![07_shadow_acne_example](/assets/images/2024/2024-08-21-ShadowMapping/07_shadow_acne_example.jpg)

下面结合一张示意图来具体说明出现这个问题的原因：

![06_shadow_acne](/assets/images/2024/2024-08-21-ShadowMapping/06_shadow_acne.jpg)

如上图所示，绿色线条代表的是阴影贴图中一个纹素覆盖的场景中的几何体，对于场景中的一个平面的一部份，黄色线条和灰色线条映射的是阴影贴图中的同一个纹素，而在采样阴影贴图做深度比较时，可以发现黄色线条在光源空间（Light Space）的深度都是小于阴影贴图中对应纹素存储的深度值，所以它不在阴影中，而灰色线条在光源空间的深度都是大于阴影贴图中对应纹素存储的深度值，所以对比的结果显示它在阴影中，这样也就造成了阴影暗斑的问题，实际上黄色线条和黑色线条都应该不在阴影中。

另一个导致阴影暗斑的原因是，转换到光源空间的像素的深度值与阴影贴图中存储的深度值非常接近，以至于精度错误导致深度测试结果的错误，这种**错误也叫做 `z-fighting`** 。造成这种精度差异的一个原因是，深度贴图是由固定功能（Fixed-Function）的光栅化硬件计算的，而与深度贴图中存储的深度做比较的深度是在着色器中计算的。

最后，投影锯齿也会造成阴影暗斑的问题。

通过对像素的深度值或者阴影贴图中存储的深度值做一个固定的 **深度偏移（Depth Bias）** 可以有效的解决阴影暗斑的问题。还是对上面出现阴影暗斑问题的渲染结果，对像素的深度值添加一个 `0.001` 的深度偏移，也就是 `shadowCoord.z -= 0.001;` ，可以看出几乎解决了阴影暗斑的问题：

![08_depth_bias](/assets/images/2024/2024-08-21-ShadowMapping/08_depth_bias.jpg)

下图也展示了深度偏移的原理：

![09_depth_bias_alg](/assets/images/2024/2024-08-21-ShadowMapping/09_depth_bias_alg.jpg)

上图中展示的是对阴影贴图中存储的深度值做了一个固定的深度偏移，也就是红色线条的部分。可以看到，经过深度偏移后，之前做深度测试错误的被判断为在阴影中的像素现在能正确的被判断出不在阴影中了。

但是过大的深度偏移，又会造成一个新的问题，这个问题叫做 **Peter Panning**，如下图所示（例子使用了 `0.007` 的深度偏移）：

![10_peter_panning](/assets/images/2024/2024-08-21-ShadowMapping/10_peter_panning.jpg)

#### 1.1.4 彼得潘效应（Peter Panning）

**彼得潘效应（Peter Panning）** 这个术语源自一本儿童书籍中的角色，该角色的影子脱离了他并且他能飞。这种瑕疵问题（Artifact）使得缺失阴影的物体看起来像是与地面分离并漂浮在空中一样。在这种情况下，深度偏移导致深度测试错误的通过。当阴影贴图精度不足时，彼得潘效应会加剧。

计算紧密的近剪裁面和远剪裁面也有助于避免彼得潘效应。

### 1.2 改善阴影贴图的一些技术

#### 1.2.1 基于斜率的深度偏移（Slope-Scale Depth Bias）

对于固定的深度偏移，偏移值太小还是会有阴影暗斑的问题，而偏移值太大又会造成彼得潘效应。此外，相对于光源倾斜角度较大的像素比倾斜度较小的像素更容易受到投影锯齿的影响。因此，每个深度贴图中存储的深度值需要根据像素相对于光源的倾斜角度应用不同的深度偏移量，这就是 **基于斜率的深度偏移（Slope-Scale Depth Bias）**。下图展示了基于斜率的深度偏移的原理：

![11_slope_scale_depth_bias](/assets/images/2024/2024-08-21-ShadowMapping/11_slope_scale_depth_bias.jpg)

上图中红色线条仍然是对阴影贴图中存储的绿色线条深度值做了一个深度偏移后深度值，可以看到基于不同像素相对于光源倾斜的角度，每个深度偏移值也不一样，对于正对着光源的像素，几乎不做深度偏移，而对于倾斜角度很大的像素（比如右下角的像素），深度偏移值就比较大。

在 OpenGL 中，可以通过 `glPolygonOffset` 来应用基于斜率的深度偏移，其原理是通过屏幕坐标 `x` 和 `y` 的导数来得到 `x` 和 `y` 方向上的变化率，根据变化率计算出像素的倾斜程度，最终得到不同的深度偏移，使用的代码如下：

```cpp
glEnable(GL_POLYGON_OFFSET_FILL);

// Usually the following works well
glPolygonOffset(1.1f, 4.0f);

// Render shadow map...

glDisable(GL_POLYGON_OFFSET_FILL);
```

`Direct3D` 中也有类似的技术，在 `Direct3D 10` 之后，硬件能够根据多边形相对于视角方向的倾斜角度来调整深度偏移量。这种做法的效果是，对那些边缘面向光源方向的多边形应用较大的偏移量，而对于直接面向光源的多边形则不应用任何偏移。下图展示了当对同一个未偏移的倾斜角度进行测试时，两个相邻像素如何在阴影和非阴影之间交替变化。

#### 1.2.2 提高深度缓冲区的精度或提高阴影贴图的分辨率

深度缓冲区精度可以是 16 位、24 位或 32 位，值介于 0 和 1 之间，通常是固定点格式（Fixed-point Format）。使用更高精度的深度缓冲区可以改善阴影的质量，同时，使用更高分辨率的阴影贴图也可以改善阴影的质量。

#### 1.2.3 计算合适的光源投影大小（Calculating a Tight Projection）

紧密的将光源的投影大小适配到摄像机的视锥体大小可以增加阴影贴图的覆盖范围。下图展示了不同投影大小对阴影贴图覆盖的影响，其中观察的视角来自光源，梯形代表的是摄像机的视锥体，网格代表的是阴影贴图，从对比可以看出，当光源的投影大小更紧密的适配场景时，相同分辨率的阴影贴图会对场景产生更多的纹素覆盖：

![12_tight_projection](/assets/images/2024/2024-08-21-ShadowMapping/12_tight_projection.png)

计算适合的光源投影大小的方法也很简单，将摄像机视锥体的 8 个顶点转换到光源空间，接下来，找到 `X` 和 `Y` 方向上的最小值和最大值，这些值构成了光源正交投影的边界。下图展示了光源投影大小正确适配摄像机的视锥体：

![13_fit_frustum](/assets/images/2024/2024-08-21-ShadowMapping/13_fit_frustum.png)

也可以将摄像机视锥体裁剪到场景的轴对齐边界包围盒（Axis-Aligned Bounding Box，AABB）大小以获得更紧密的边界。然而，并不是在所有情况下都建议这么做，这可能导致光源的投影大小每帧都在变化，这种变化会导致阴影计算结果的不稳定（某一个像素上一帧在阴影中，下一帧可能因为光源投影大小的变化导致它的计算结果又是不在阴影中）。

#### 1.2.4 计算适合的光源投影的近平面和远平面（Calculating the Near Plane and Far Plane）

深度缓冲区的精度也由近平面与远平面的比例决定。使用尽可能紧密的近平面/远平面比例可以提高深度缓冲区的精度，在使用紧密的近平面/远平面的情况下还可以使用16位深度缓冲区，16位深度缓冲区可以减少内存使用，同时提高处理速度。

下面介绍一些常见的计算光源投影近平面/远平面的方法：

1. 基于场景的轴对齐边界包围盒来计算近平面/远平面（AABB-Based Near Plane and Far Plane）

    通过将场景的包围盒转换到光源空间，取最小的 `Z` 坐标值为近平面，最大的 `Z` 坐标值为远平面。对于一般的情况，这种方法可以使远近平面更紧凑，但是当场景很大而摄像机的视锥体很小时，这种计算并不是最理想的（仍然导致阴影贴图上少量的纹素覆盖了整个摄像机视锥体），如下图展示的情况：

    ![14_near_far_scene_aabb](/assets/images/2024/2024-08-21-ShadowMapping/14_near_far_scene_aabb.png)

2. 基于摄像机视锥体来计算近平面/远平面（Frustum-Based Near Plane and Far Plane）

    此方法是通过将摄像机视锥体转换到光源空间，然后分别使用 `Z` 方向上最小值和最大值作为近平面和远平面。下图也展示了这个方法的一些局限性，首先，当视锥体超出场景几何体时，这种计算过于保守。其次，当视锥体太小时，计算出的近/远平面又太过于紧密。下图展示了这两种情况：

    ![15_near_far_frustum_based](/assets/images/2024/2024-08-21-ShadowMapping/15_near_far_frustum_based.png)

3. 光线与场景相交以计算近平面/远平面（Light Frustum Intersected with Scene to Calculate Near and Far Planes）

    此方法是最适合（Proper）计算远/近平面的方法。通过将摄像机视锥体转换到光源空间，取 `X` 和 `Y` 方向上的最小值和最大值作为光源投影大小，知道光源投影大小后，使用光源锥体的 4 个锥体面将场景的包围盒进行裁剪，从新裁剪的空间的边界中找到最小和最大的 `Z` 值，它们也就是要求的近平面和远平面了。下图是这个方法的示意图：

    ![16_light_frustum_intersected_scene](/assets/images/2024/2024-08-21-ShadowMapping/16_light_frustum_intersected_scene.png)

    微软在他们的 [DirectX 的 CascadedShadowMaps11 示例中](https://github.com/walbourn/directx-sdk-samples) 实现了这个算法。核心思路是，首先将场景包围盒的转换到光源空间，光源空间中的场景包围盒每个四边形面可以表示为 2 个三角形，所以光源空间中的场景包围盒可以用 12 个三角形来表示。将这些三角形裁剪到光源视锥体的空间中，所有被裁减后的三角形中的最小和最大的 `Z` 值就是近平面和远平面了。

#### 1.2.5 以纹素大小的增量来移动光源（Moving the Light in Texel-Sized Increments）

使用阴影贴图技术还有一个常见的问题就是 **闪烁边缘效应（shimmering edge effect）**。当摄像机移动时，阴影边缘的像素会变亮或者变暗。下图展示了这个问题出现的样子：

![17_shimmering_shadow_edges](/assets/images/2024/2024-08-21-ShadowMapping/17_shimmering_shadow_edges.png)

发生闪烁边缘效应的原因是因为随着摄像机的移动，光源投影矩阵都需要重新计算。以下所有因素都有可能影响包围场景的光源投影视锥体：

- 摄像机视锥体的大小
- 摄像机视锥体的方向
- 光源的位置
- 摄像机的位置

对于方向光，解决这个问题的方法是将光源投影视锥大小的最小/最大 `X` 和 `Y` 值四舍五入到纹素大小的增量，实现的代码如下：

```cpp
vLightCameraOrthographicMin /= vWorldUnitsPerTexel;
vLightCameraOrthographicMin = XMVectorFloor( vLightCameraOrthographicMin );
vLightCameraOrthographicMin *= vWorldUnitsPerTexel;
vLightCameraOrthographicMax /= vWorldUnitsPerTexel;
vLightCameraOrthographicMax = XMVectorFloor( vLightCameraOrthographicMax );
vLightCameraOrthographicMax *= vWorldUnitsPerTexel;
```

`vWorldUnitsPerTexel` 值的计算方法是取视锥体的边界并除以缓冲区大小：

```cpp
FLOAT fWorldUnitsPerTexel = fCascadeBound / (float)m_CopyOfCascadeConfig.m_iBufferSize;
vWorldUnitsPerTexel = XMVectorSet( fWorldUnitsPerTexel, fWorldUnitsPerTexel, 0.0f, 0.0f );
```

将视锥体的最大尺寸作为边界会导致正交投影的贴合度更松散。

需要注意的是，当使用这种技术时，纹理的宽度和高度会增加 1 个像素。这可以防止阴影坐标索引到阴影贴图之外。

## (2) 级联阴影（Cascaded Shadow Maps）

在前面的阴影贴图中的一些问题中，有提到 [透视锯齿](#透视锯齿perspective-aliasing) 的问题，出现透视锯齿问题的原因是靠近摄像机的阴影贴图分辨率不足（阴影贴图中少量的纹素覆盖了场景中太大的范围）。而解决透视锯齿的问题，目前来说最佳的方案就是使用 **级联阴影**。

级联阴影的基本思想是将摄像机视锥体分割成多个子视锥体，为每个视锥体渲染一张阴影贴图，在实际渲染的过程中，根据像素与摄像机的距离，判断像素处于哪个子视锥体，并使用所在的子视锥体的阴影贴图来计算阴影。下图展示了将摄像机视锥体分割成 3 个子视锥体，并为每一个子视锥体渲染一张阴影贴图（需要注意的是，一般情况下每张阴影贴图的分辨率应该是一样的，下图中展示的网格密度的不同只是为了展示场景中靠近摄像机的区域也覆盖了足够分辨率的阴影贴图纹素数量）：

![18_cascade_shadow_maps](/assets/images/2024/2024-08-21-ShadowMapping/18_cascade_shadow_maps.png)

### 2.1 分割摄像机视锥体的方法

通过计算摄像机视锥体在 `Z` 方向上的区间，将此区间按固定的间隔划分为一个个的小区间，每个小区间代表的就是一个子视锥体，每个子视锥体根据划分的 `Z` 方向上的区间都有一个近平面和远平面，下图展示了一个对摄像机视锥体进行分割的例子：

![19_partition_view_frustum](/assets/images/2024/2024-08-21-ShadowMapping/19_partition_view_frustum.jpg)

一般在实际渲染中，普遍使用的分割摄像机视锥体做法是：针对每个不同的场景，使用一组静态的级联区间（Cascades Intervals）（每帧重新计算动态的级联区间会导致阴影边缘闪烁），级联区间中的每个值描述的是沿 `Z` 轴的间隔，通过这个间隔来表示分割出的一个个子视锥体。常见的做法是使用一组固定的级联区间，这样在像素着色器中，可以通过像素的 `Z` 值来计算出当前像素具体处于哪个级联阴影中。

### 2.2 摄像机视锥体覆盖的场景区域 Z 轴范围的大小影响级联区间的划分

对于场景中的几何体，摄像机的方向会影响级联区间的选择。当摄像机视锥体覆盖的场景中的几何体的 `Z` 轴范围特别大时，需要划分更多的级联，而当场景中大部分的几何体都集中在摄像机视锥体的一个小部分时，此时只需要少量的级联。下图展示了摄像机视锥体覆盖的场景中几何体在 `Z` 轴范围大小不同时，不同的级联区间的划分。可以看到最左边的图中，覆盖的几何体在 `Z` 轴上有很大的范围时，这时需要大量的级联，而当覆盖的几何体在 `Z` 轴上有中等范围（右图）、低范围（中图）时，这时可以划分出少量的级联：

![20_scene_geometry](/assets/images/2024/2024-08-21-ShadowMapping/20_scene_geometry.jpg)

### 2.3 光源与摄像机的方向之间的关系会影响级联之间的重叠

每个级联的投影矩阵都紧密的适配其对应的子视锥体。当摄像机方向与光源方向正交（互相垂直）时，每个级联可以很紧密的适配其对应的子视锥体，且级联之间重叠很少；当摄像机方向与光源方向趋于平行时，每个级联之前的重叠很多；而当摄像机方向和光源方向几乎平行时，这种情况被称为 **对抗视锥体（Dueling Frusta）** ，对于大多数阴影算法来说，这是一个非常难处理的情况，所以需要尽量避免这种情况的发生，级联阴影在这种情况下的表现优于许多其它算法。下图展示了随着摄像机方向与光线方向越来越趋于平行时，级联之间的重叠会越来越明显：

![21_cascades_overlap](/assets/images/2024/2024-08-21-ShadowMapping/21_cascades_overlap.jpg)

### 2.4 创建子视锥体

创建子视锥体一般有两种方法： **适应场景（Fit to Scene）** 和 **适应级联（Fit to Cascade）** 。

- **适应场景（Fit to Scene）** ：所有的子视锥体使用相同的近平面来创建。这种方法会导致级联之间会有重叠。下图展示了使用适应场景的方法创建的子视锥体：

  ![22_fit_to_scene](/assets/images/2024/2024-08-21-ShadowMapping/22_fit_to_scene.jpg)

- **适应级联（Fit to Cascade）** ：通过实际划分的区间作为近平面和远平面来创建子视锥体。这种方法会使得每个级联的投影矩阵更紧密的适配子视锥体，但是在对抗视锥体（摄像机方向与光源方向几乎平行时）的情况下，这种方法会退化成与适应场景一样的计算方式。下图展示了使用适应级联的方法创建的子视锥体：

  ![23_fit_to_cascade](/assets/images/2024/2024-08-21-ShadowMapping/23_fit_to_cascade.jpg)

适应级联的方法计算出的子视锥体之间没有重叠，与适应场景的方法相比更充分的利用了分辨率有限的阴影贴图（有重叠区域意味着浪费了阴影贴图上的纹素），但是当摄像机移动旋转时，通过适应级联方法计算出的子视锥体都需要重新计算，这会导致阴影闪烁的问题。

### 2.5 渲染级联阴影贴图

一种方法将所有级联阴影贴图渲染到一张大的阴影贴图纹理上（通过在渲染每一个级联阴影时改变视口（Viewport）的偏移和大小），阴影贴图上的每个部分代表了一个级联阴影贴图，下图展示了在一张 `2048 x 2048` 的阴影贴图上渲染了 4 个级联阴影贴图：

![24_four_cascades](/assets/images/2024/2024-08-21-ShadowMapping/24_four_cascades.jpg)

还有一种方法是使用 **纹理数组（Textures Array）**，数组中的每一张纹理代表了一个级联阴影贴图。

### 2.6 采样级联阴影贴图

采样级联阴影贴图最重要的一点就是确定当前像素着色器需要采样哪一级的级联阴影贴图。一般有如下 2 种方法：

- **根据深度间隔来确定级联** ： 通过级联区间划分与摄像机视锥体的深度 `Z` 将每一级联的深度范围计算出来并传入 `GPU` ，在着色器中，将像素坐标转换到摄像机的视图空间，根据该像素在摄像机视图空间的深度坐标 `Z` 与所有级联的深度范围做对比，这样就能确定该像素所属的级联。找到所属级联后，再应用此级联的纹理坐标的缩放（Scale）和偏移（Offset），也就确定了该像素在大的阴影贴图上需要采样的纹理坐标了。
- **根据每个级联的视口缩放和偏移来确定级联**： 此方法的核心思想是：首先将像素转换到光源的视图空间，然后遍历每个级联的投影矩阵，当计算出的 `X` 和 `Y` 坐标在纹理上时（范围属于 `[0, 1]`，实际计算中会偏移一个纹素的大小 ），也就说明找到了所属的级联。最后，同样需要再应用此级联的纹理坐标的缩放（Scale）和偏移（Offset），最终计算出该像素在大的阴影贴图上需要采样的纹理坐标。

## (3) 硬阴影（Hard Shadows）与软阴影（Soft Shadows）

首先，阴影贴图的核心原理就是通过 **将像素在光源空间中的深度值与该像素在阴影贴图中对应纹素所存储的深度值做对比来判断该像素在光源空间中能不能被光源看到（能看到就是不在阴影中，不能看到就是在阴影中，因为前方有遮挡）** 。

由于阴影贴图具有固定的分辨率，也就是说阴影贴图中存储的深度信息是离散的，而在采样阴影贴图的过程中，场景中的多个像素采样的是阴影贴图中的同一个纹素，也就是说场景中的这些像素会采样到相同的深度值，在做深度测试时，因为对比的是同一个深度值，所以会产生锯齿状的块状边缘，这种阴影被称为 **硬阴影（Hard Shadows）**。增加阴影贴图的分辨率可以改善锯齿的大小。下图展示了不同分辨率的阴影贴图下的硬阴影：

![25_hard_shadows](/assets/images/2024/2024-08-21-ShadowMapping/25_hard_shadows.png)

### 3.1 Percentage Closer Filtering（PCF）

PCF的基本思想是从阴影贴图中多次采样，每次使用略有不同的采样纹理坐标，最终根据所有通过深度测试的采样数与总的采样数的比例来计算像素处于阴影中的百分比。如下图所示，一个像素通过了 4 个深度测试中的 1 个，所以它处于 25% 的阴影中：

![26_pcf](/assets/images/2024/2024-08-21-ShadowMapping/26_pcf.png)

通过这种方式计算出的阴影称为 **软阴影（Soft Shadows）** 。通过扩大 PCF 内核的大小（采样点周围 NxN 的像素），可以是阴影边缘产生更平滑的渐变效果。

#### 3.1.1 双线性 PCF （Bilinear PCF）

这是最基础的 PCF `2x2` 内核，通过采样像素映射到阴影贴图中纹理坐标周围的 4 个纹素，并将每次采样的深度测试的结果通过 **双线性插值（Bilinear Interpolation）** 的方式进行混合，得到最终该像素的阴影值。

![27_bilinear_pcf](/assets/images/2024/2024-08-21-ShadowMapping/27_bilinear_pcf.png)

上图中像素映射到阴影贴图中的纹理坐标是红色的点 `asked point` ，双线性 PCF 通过采样周围的 4 个纹素（texel1 - texel4），对每个采样的结果与像素在光源空间的深度值做深度测试，得到每个采样纹素的深度测试结果，最后根据映射的纹理坐标与 4 个采样纹素的纹理坐标做双线性插值，计算出映射点的深度测试结果。

在早期的硬件中，需要手动进行 4 次**点过滤（Point Filtering）**采样，并做双线性插值才能得到最终的阴影结果。在现代的硬件中都集成了此算法，只需要 1 次采样就可以得到同样的结果。下面举一个在 OpenGL 中使用的例子：

```cpp
// 设置阴影贴图的过滤方式为双线性
glTexParameteri(m_Target, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
glTexParameteri(m_Target, GL_TEXTURE_MAG_FILTER, GL_LINEAR);

// 设置阴影贴图的比较模式
glTexParameteri(m_Target, GL_TEXTURE_COMPARE_MODE, GL_COMPARE_REF_TO_TEXTURE);
glTexParameteri(m_Target, GL_TEXTURE_COMPARE_FUNC, GL_LEQUAL);
```

在着色器中，只需要一次采样就可以得到双线性 PCF 的结果：

```cpp
uniform sampler2DShadow uShadowMap;

...

float shadow = texture(uShadowMap, shadowCoord.xyz);
```

下图展示了使用双线性 PCF 的阴影结果，阴影贴图的分辨率是 `512x512` ：

![28_bilinear_pcf_512](/assets/images/2024/2024-08-21-ShadowMapping/28_bilinear_pcf_512.png)

#### 3.1.2 更大内核的 PCF

通过使用更大的加权内核进行采样，例如使用 `NxN` 的加权内核（所有权重总和为 1 ），可以使阴影更加柔和，不过更大的内核也意味着更高的性能消耗，因为需要更多的采样。也有使用 **非对称的（Irregular）** 内核来进行 PCF 过滤（例如使用泊松圆盘采样（Poisson Disk Sampling）算法生成的内核），通过非对称的 PCF 内核进行采样可以减少采样次数，节省带宽的使用，不过非对称的 PCF 内核会使阴影噪声（Noise）变多。下图展示了普通 `NxN` PCF 内核与非对称 PCF 内核的阴影效果对比：

![29_regular_irregular](/assets/images/2024/2024-08-21-ShadowMapping/29_regular_irregular.png)

需要注意的是，过大的 PCF 内核也会造成[阴影暗斑](#阴影暗斑shadow-acne)的问题。

PCF 的主要思想是：对于场景中的某个转换到光源空间后的像素的深度 `Z` 值，PCF 算法不仅会采样此像素映射到阴影贴图中的纹素，并将采样的结果与 `Z` 值做深度测试比较，而且还会采样映射纹素周围 `NxN` 的纹素，并将采样的结果分别与 `Z` 值做深度测试比较，最终根据通过深度测试的样本数量与总的采样数量的比值来确定此像素的阴影值。通过这个主要思想就能发现一个问题： **对于阴影贴图中的每个纹素，都应该分别使用其映射到光源空间中像素的深度 `Z` 值与其存储的深度值做深度测试比较，而不是像上面描述的那样使用同一个光源空间中的像素的深度 `Z` 值与所有这些 `NxN` 纹素中存储的深度值做深度测试比较**。

一种解决此问题的方法是，利用当前像素在光源空间中坐标的导数（Derivative），计算出相邻像素深度 `Z` 值在 `X` 和 `Y` 轴方向上的变化率，在 PCF 过滤采样相邻纹素的过程中，基于相邻纹素纹理坐标在 `X` 和 `Y` 轴上的偏移，计算出深度偏差并将其应用到深度测试比较中。具体的流程如下：

1. 通过求导函数 `ddx` 和 `ddy` 计算出当前像素在光源空间中 xyz 坐标的导数，通过倒数的 `x` 和 `y` 构建一个 `float2x2` 的矩阵，此矩阵用来将屏幕空间的邻近纹素转换为光源空间的斜率。
2. 通过对这个矩阵求逆，得到一个将光源空间邻近像素转换为屏幕空间斜率的矩阵。
3. 然后将光源空间中，向上偏移 1 个像素 和 向右偏移 1 个像素的向量转换到屏幕空间斜率，也就得到了屏幕空间中向上+y和向右+x的相邻像素的斜率。
4. 因为最终要对比的是深度值，所以这2个斜率还要乘以深度 z 的倒数，这样就得到了屏幕空间相邻纹素相对的深度的斜率。
5. 在 PCF 计算时，在应用 xy 偏移的同时，同时根据这 2 个斜率计算出深度偏差。

在 OpenGL 中实现的代码如下：

```cpp
void CalculateRightAndUpTexelDepthDeltas(in vec3 texShadowView, in mat3 shadowProjection, out float upTextDepthWeight, out float rightTextDepthWeight)
{
    vec3 vShadowTexDDX = dFdx(texShadowView);
    vec3 vShadowTexDDY = dFdy(texShadowView);

    vShadowTexDDX = shadowProjection * vShadowTexDDX;
    vShadowTexDDY = shadowProjection * vShadowTexDDY;

    mat2 matScreenToShadow = mat2(vShadowTexDDX.xy, vShadowTexDDY.xy);

    // 求逆矩阵
    float fDeterminant = determinant(matScreenToShadow);
    float fInvDeterminant = 1.0 / fDeterminant;
    mat2 matShadowToScreen = mat2(
        matScreenToShadow[1][1] * fInvDeterminant, matScreenToShadow[0][1] * -fInvDeterminant,
        matScreenToShadow[1][0] * -fInvDeterminant, matScreenToShadow[0][0] * fInvDeterminant
    );

    vec2 vRightShadowTexelLocation = vec2(1.0/2048.0, 0.0);
    vec2 vUpShadowTexelLocation = vec2(0.0, 1.0/2048.0);

    vec2 vRightTexelDepthRatio = matShadowToScreen * vRightShadowTexelLocation;
    vec2 vUpTexelDepthRatio = matShadowToScreen * vUpShadowTexelLocation;

    upTextDepthWeight = vUpTexelDepthRatio.x * vShadowTexDDX.z + vUpTexelDepthRatio.y * vShadowTexDDY.z;
    rightTextDepthWeight = vRightTexelDepthRatio.x * vShadowTexDDX.z + vRightTexelDepthRatio.y * vShadowTexDDY.z;
}
```

由于这种技术在计算上比较复杂，因此只有在 GPU 有足够的计算周期可供使用时才应该使用它。当使用非常大的 PCF 内核时，这可能是唯一一种可以去除自阴影伪影而不引起彼得潘效应的技术。

<!-- TODO
### Variance Shadow Maps（VSMs） -->

<!-- TODO
### Percentage-Closer Soft Shadows（PCSS）
- https://developer.download.nvidia.cn/whitepapers/2008/PCSS_Integration.pdf
- https://developer.download.nvidia.cn/shaderlibrary/docs/shadow_PCSS.pdf
- https://www.legendsmb.com/2022/10/13/unity-PCF-sourcecode/
- https://zhuanlan.zhihu.com/p/369761748 -->

## (4) 参考

- [1] [https://graphics.stanford.edu/~mdfisher/Shadows.html](https://graphics.stanford.edu/~mdfisher/Shadows.html)
- [2] [https://learn.microsoft.com/en-us/windows/win32/dxtecharts/common-techniques-to-improve-shadow-depth-maps](https://learn.microsoft.com/en-us/windows/win32/dxtecharts/common-techniques-to-improve-shadow-depth-maps)
- [3] [https://docs.google.com/presentation/d/10FKu5BF7HPC8hRKp1h3ZdO1tkUXagOq3ZLpHgJvDIqg/pub](https://docs.google.com/presentation/d/10FKu5BF7HPC8hRKp1h3ZdO1tkUXagOq3ZLpHgJvDIqg/pub)
- [4] [https://docs.unity3d.com/2021.3/Documentation/Manual/ShadowPerformance.html](https://docs.unity3d.com/2021.3/Documentation/Manual/ShadowPerformance.html)
- [5] [https://github.com/walbourn/directx-sdk-samples/blob/main/CascadedShadowMaps11](https://github.com/walbourn/directx-sdk-samples/blob/main/CascadedShadowMaps11)
- [6] [https://learn.microsoft.com/en-us/windows/win32/dxtecharts/cascaded-shadow-maps](https://learn.microsoft.com/en-us/windows/win32/dxtecharts/cascaded-shadow-maps)
- [7] [https://developer.download.nvidia.com/SDK/10.5/opengl/src/cascaded_shadow_maps/doc/cascaded_shadow_maps.pdf](https://developer.download.nvidia.com/SDK/10.5/opengl/src/cascaded_shadow_maps/doc/cascaded_shadow_maps.pdf)
- [8] [https://developer.download.nvidia.com/presentations/2008/GDC/GDC08_SoftShadowMapping.pdf](https://developer.download.nvidia.com/presentations/2008/GDC/GDC08_SoftShadowMapping.pdf)
- [9] [https://cg.informatik.uni-freiburg.de/course_notes/graphics_07_shadows.pdf](https://cg.informatik.uni-freiburg.de/course_notes/graphics_07_shadows.pdf)
- [10] [https://developer.download.nvidia.cn/assets/gamedev/docs/GDC01_Shadows.pdf](https://developer.download.nvidia.cn/assets/gamedev/docs/GDC01_Shadows.pdf)
