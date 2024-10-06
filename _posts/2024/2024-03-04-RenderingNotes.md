---
layout: post
title:  "渲染学习笔记"
date:   2024-03-04 16:16:00 +0800
category: Rendering
---

## 说明

在这里记录学习一些图形学中的概念、算法的学习笔记，记录完全随机。

---

## Normal Matrix(法线变换矩阵)

法线向量是定义在模型空间中的，而光照计算一般情况下都是在世界空间进行的，所以就需要一个矩阵来把法线向量从模型空间转换到世界空间，这个矩阵就叫做法线变化矩阵。

首先，法线向量它是一个向量，它并不是和模型顶点坐标一样代表的是空间中的一个位置，所以法线向量的齐次坐标为 `0`，这意味着平移变换对法线向量不会有任何影响。顶点坐标是通过模型矩阵 $\mathbf{M}$ 变换到世界空间的，那么使用忽略掉平移部分(只考虑对法线向量的旋转和缩放)的模型矩阵左上角 3x3 部分的矩阵来变换法线向量是不是就可以了？如果模型矩阵执行了统一缩放(X,Y和Z是相同的缩放值)，直接使用模型矩阵左上角3x3部分来变换法线向量是没有问题的，此时变换只改变了法线向量的长度，而没有改变它的方向，所以只需要做归一化操作就可以得到正确的计算光照所需的法线向量；但如果模型矩阵执行了非统一缩放，这个时候就会出问题，变换后的法线向量会不再垂直于表面，下图展示了模型矩阵进行了非统一缩放变换时，直接使用模型矩阵导致的错误结果：

![00_ununiform_scaling](/assets/images/2024/2024-03-04-RenderingNotes/00_ununiform_scaling.png)

所以需要推导出正确的法线变换矩阵，下面是推导过程：

![01_normal_triangle](/assets/images/2024/2024-03-04-RenderingNotes/01_normal_triangle.webp)

上图中，垂直于法线向量 $\mathbf{N}$ 的切线向量 $\mathbf{T}$ 可以由三角形边上的2个点 $P_{1}$ 和 $P_{2}$ 计算得到：

$$
\mathbf{T} = \mathbf{P_2} - \mathbf{P_1}
$$

考虑到切线向量是一个方向向量，而方向向量也可以用齐次坐标来表示(此时的 `w` 分量为0)，所以上面等式2边可以同时乘以 4x4 的模型矩阵 $\mathbf{M}$：

$$
\mathbf{M} * \mathbf{T} = \mathbf{M} * (\mathbf{P_2} - \mathbf{P_1})
$$

所以：

$$
\mathbf{M} * \mathbf{T} = \mathbf{M} * \mathbf{P_2} - \mathbf{M} * \mathbf{P_1}
$$

可以看到等式右边就是2个在模型空间的点 $\mathbf{P_2}$ 和 $\mathbf{P_1}$ 分别变换到世界空间后再继续相减，得到世界空间下的切线，而等式左边是用模型矩阵直接对模型空间下的切线向量(用齐次坐标来表示，此时 `w` 分量为 0)做矩阵变换，所以可以得到可以直接使用模型矩阵左上角 3x3 部分来变换切线向量。

知道这个以后，现在继续来推导正确的法线变换矩阵：

![02_normal_triangle_transformed](/assets/images/2024/2024-03-04-RenderingNotes/02_normal_triangle_transformed.webp)

因为法线向量和切线向量应该是相互垂直的，1个垂直于表面，1个平行于表面，所以变换到世界空间后的法线向量 $\mathbf{N}^{\prime}$ 和切线向量 $\mathbf{T}^{\prime}$ 也应该互相垂直：

$$
\mathbf{N}^{\prime} \cdot \mathbf{T}^{\prime} = 0
$$

下面使用 $\mathbf{M}$ 来表示 4x4 模型矩阵左上角 3x3 部分的矩阵，通过上面的推导可以知道这个矩阵可以正确的将模型空间的切线向量 $\mathbf{T}$ 变换到世界空间(后面涉及的变换矩阵都是 3x3 的矩阵，因为切线向量和法线向量都是方向向量，不受平移变换影响)，假设法线向量的变换矩阵是 $\mathbf{G}$，所以上面的等式可以变换为：

$$
\mathbf{N}^{\prime} \cdot \mathbf{T}^{\prime} = (\mathbf{G}\mathbf{N}) \cdot (\mathbf{M}\mathbf{T}) = 0
$$

而2个行向量的点乘可以通过把其中一个向量转置成列向量从而把点乘转换成矩阵的乘法(2个Vector3向量点乘，转换为1个3x1的矩阵和1个1x3的矩阵相乘)，即：

$$
(\mathbf{G}\mathbf{N}) \cdot (\mathbf{M}\mathbf{T}) = (\mathbf{G}\mathbf{N})^{T} * (\mathbf{M}\mathbf{T})
$$

又因为转置的性质，两个矩阵相乘后进行转置，等于每个矩阵转置后再相乘，且顺序相反，所以：

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

最后：

$$
\mathbf{G}^{T} * \mathbf{M} = \mathbf{I} \Leftrightarrow \mathbf{G} = (\mathbf{M^{-1}})^{T}
$$

可以看到要求得法线变换矩阵也就是：***模型矩阵左上角3x3部分的逆的转置***。

当模型矩阵只包含旋转和平移时，此时左上角 3x3 部分只包含旋转变换，而旋转矩阵是正交矩阵，正交矩阵的逆等于它的转置，此时：

$$
\mathbf{M^{-1}} = \mathbf{M}^{T} \Leftrightarrow \mathbf{G} = \mathbf{M}
$$

所以在模型矩阵只有旋转和平移变换时，可以直接使用模型矩阵左上角3x3部分的矩阵来正确变换法线向量。

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

可以知道如果模型变换中包含旋转，统一缩放和平移时，此时也可以直接使用模型矩阵左上角3x3部分的矩阵来变换法线向量，变换后的统一缩放系数影响的是法线向量的长度，而在计算光照时，需要的仅仅的法线向量的方向，也就是最终都会对法线向量进行归一化计算，它的长度变化并不需要关心。

### 总结

如果模型变换中包含旋转变换，统一变换，平移变换时，可以直接使用模型矩阵左上角 3x3 部分的矩阵来变换法线向量；而如果变换中包含了非统一缩放，此时就必须求解模型矩阵左上角 3x3 部分的矩阵的逆的转置来得到法线变换矩阵

### 参考

- [The Normal Matrix](http://www.lighthouse3d.com/tutorials/glsl-12-tutorial/the-normal-matrix/)
- [Unity Shaders Book Chapter 4](https://candycat1992.github.io/unity_shaders_book/unity_shaders_book_chapter_4.pdf)

---

## Alpha to Coverage

<!-- Alpha to Coverage一般简称A2C或者是ATOC，是对Alpha Test做抗锯齿的一种技术。 -->
<!-- Alpha Test是一种很常见的技术，它的原理也很简单，在像素着色器中，alpha值低于设定阈值的像素将会直接被丢弃，在游戏中经常用来渲染树叶、草和头发等物体。而Alpha Blend也可以是 -->
在渲染树叶、草和头发等这种部分透明部分不透明的物体时，常用的方法有2种，Alpha Test和Alpha Blend。Alpha Test原理很简单，在像素着色器种，alpha低于设定阈值的像素将直接被丢弃，而Alpha Blend就是常见的透明像素的颜色混合。下图是使用这2个方法渲染同一个草的对比图：

![03_alphatest_alphablend](/assets/images/2024/2024-03-04-RenderingNotes/03_alphatest_alphablend.png)

可以看到Alpha Test因为是根据一个阈值直接丢弃像素，所以边缘会出现一个个的锯齿，而Alpha Blend是做的颜色混合，所以效果上要比Alpha Test好很多，但是为什么实际使用中都是使用Alpha Test来渲染这些物体的呢？因为Alpha Test相比于Alpha Blend有以下几个优点：

- 因为Alpha Test实际渲染的还是不透明像素，它不会有OverDraw的问题
- 当渲染一些比较复杂的灌木丛时(茂密的灌木丛往往都是错综复杂，相互穿插生长)，Alpha Blend不能像不透明和Alpha Test那样使用深度进行排序，所以排序是个很大的问题，有些多边形相互交叉的情况很难排序，错误的渲染顺序会导致错误的颜色混合，最终也会渲染出错误的结果

而Alpha to Coverage就是针对解决Alpha Test渲染锯齿的问题。Alpha to Coverage一般简称为A2C或者ATOC(后续都称ATOC)，它就是利用MASS来对Alpha Test结果做抗锯齿的技术。

首先来简单说说MSAA，MSAA开启后，每个pixel会对应多个subsample，光栅化阶段会基于这些subsample的位置，去计算图元(例如三角形)的coverage情况，也就是计算当前图元覆盖了哪些subsample，只有那些被覆盖了的subsample的颜色，才会被更新为当前图元的颜色(shading的结果)。画完所有图元之后，最终每个pixel的颜色，是它对应的所有sample的颜色的平均值。

而开启ATOC之后，会基于像素着色器输出的alpha值生成一个coverage mask，这个生成的coverage mask会去跟原始流程得到的结果(被图元覆盖到的subsamples)做AND操作。举个例子，假设1个三角形覆盖了4个subsamples，启用ATOC且像素着色器输出的alpha值为0.5时，则最终只有一半的subsamples将储存颜色，最后pixel颜色计算就会变成本来颜色的 1/2，也就是有点类似Alpha Blend的方式。实际的coverage mask计算还是和具体硬件实现有关。

总结一下，ATOC使得MSAA针对图元边缘做抗锯齿的时候，会基于像素着色器输出的alpha值去修改图元颜色在1个pixel里面锁占的coverage，从某种程度上来说，是在修改这个图元在这个pixel里的“透明度”，实现了类似Alpha Blend的效果。下图展示了ATOC的效果：

![04_alpha_to_coverage](/assets/images/2024/2024-03-04-RenderingNotes/04_alpha_to_coverage.png)

### 参考

- [Anti-aliased Alpha Test: The Esoteric Alpha To Coverage](https://bgolus.medium.com/anti-aliased-alpha-test-the-esoteric-alpha-to-coverage-8b177335ae4f)
- [A Quick Overview of MSAA](https://therealmjp.github.io/posts/msaa-overview/)
- [Multisampling](https://www.khronos.org/opengl/wiki/Multisampling)
- [UE426终于实现了Alpha to Coverage](https://zhuanlan.zhihu.com/p/388513281)
- [Alpha compositing](https://developer.arm.com/documentation/102073/0100/Alpha-compositing)
- [Alpha to coverage](https://www.humus.name/index.php?page=3D&ID=61)
- [ShaderLab command: AlphaToMask](https://docs.unity3d.com/2023.2/Documentation/Manual/SL-AlphaToMask.html)

---

## Fast Approximate Anti-aliasing (FXAA)

FXAA 是一种基于图像处理的抗锯齿算法，它的优点就是性能开销极小，而且有着不错的抗锯齿效果。它主要是通过混合具有高对比度的相邻像素来实现抗锯齿，下面就来看看这个算法的具体实现。

一个像素的亮度是通过其 `R, G, B` 值来确定的，具体的算法如下：

```c++
float FXAALuma(float4 rgba)
{
    return dot(rgba.xyz, float3(0.299, 0.587, 0.114));
}
```

首先要做的，是找到那些与周围像素具有高对比度的像素，这是通过采样计算中心像素的亮度和此像素相邻的上、下、左和右四个像素的亮度来确定的，需要说明的是，中心像素指的是此像素着色器的当前像素。如下图：

![15_FXAA_N_S_E_W_M](/assets/images/2024/2024-03-04-RenderingNotes/15_FXAA_N_S_E_W_M.png)

采样这五个像素，并计算它们的亮度值，找到其中最大的亮度值和最小的亮度值，相减就可以得到中心像素周围像素的对比度，知道对比度后，通过过滤掉与相邻像素具有低对比度的像素，保留与相邻像素具有
高对比度的像素，这些保留下来的与相邻像素具有高对比度的像素也就是需要做抗锯齿处理的锯齿边缘像素：

![16_FXAA_HighContrastFilter](/assets/images/2024/2024-03-04-RenderingNotes/16_FXAA_HighContrastFilter.png)

下面是具体的代码实现：

```c++
// Center pixel
float2 posM = input.texcoord;

// Center sample
half4 rgbyM = SAMPLE_TEXTURE2D_LOD(_BlitTexture, sampler_LinearClamp, posM, 0.0);

// Luma of pixels around current pixel
float lumaM = FXAALuma(rgbyM);
float lumaS = FXAALuma(FXAATexOff(posM, int2(0, -1)));
float lumaE = FXAALuma(FXAATexOff(posM, int2(1, 0)));
float lumaN = FXAALuma(FXAATexOff(posM, int2(0, 1)));
float lumaW = FXAALuma(FXAATexOff(posM, int2(-1, 0)));

float maxSM = max(lumaS, lumaM);
float minSM = min(lumaS, lumaM);
float maxESM = max(lumaE, maxSM);
float minESM = min(lumaE, minSM);
float maxWN = max(lumaW, lumaN);
float minWN = min(lumaW, lumaN);
// Maximum luminance of pixels
float rangeMax = max(maxWN, maxESM);
// Minimum luminance of pixels
float rangeMin = min(minWN, minESM);
float rangeMaxScaled = rangeMax * FXAA_QUALITY__EDGE_THRESHOLD;
// Get the edge
float range = rangeMax - rangeMin;
// Skip the pixel that its neighborhood does not have a high enough contrast
float rangeMaxClamped = max(FXAA_QUALITY__EDGE_THRESHOLD_MIN, rangeMaxScaled);
if (range < rangeMaxClamped)
    return rgbyM;
```

![17_FXAA_Edge](/assets/images/2024/2024-03-04-RenderingNotes/17_FXAA_Edge.png)

<center> 当没有通过对比度阈值检测时输出 0.0，通过对比度阈值检测时输出 range，可以看到锯齿边缘被过滤了出来 </center>

<br/>

过滤出锯齿边缘后，现在需要知道锯齿边缘的方向，锯齿边缘的方向可能是水平的，也有可能是垂直的，锯齿边缘的方向决定了最终混合的方向。如下图可能的锯齿边缘方向：

![18_FXAA_BlendDirections](/assets/images/2024/2024-03-04-RenderingNotes/18_FXAA_BlendDirections.png)

通过额外采样中心像素的左上，右上，左下和右下这四个像素并计算它们的亮度值来确定锯齿边缘的方向，而且因为不同的像素位置，所以它们贡献的权重也是不同的：

额外的四个相邻的像素：

![19_FXAA_NW_NE_SW_SE](/assets/images/2024/2024-03-04-RenderingNotes/19_FXAA_NW_NE_SW_SE.png)

相邻像素的权重：

![20_FXAA_NeighborWeights](/assets/images/2024/2024-03-04-RenderingNotes/20_FXAA_NeighborWeights.png)

分别计算出水平和垂直的权重，比较它们的大小就可以判断锯齿边缘的方向，也就是后续需要混合的方向：

```c++
float horizontalWeight = abs((N + S) - 2.0 * M)) * 2.0 + abs((NE + SE) - 2.0 * E)) + abs((NW + SW) - 2.0 * W));
float verticalWeight = abs((W + E) - 2.0 * M)) * 2.0 + abs((NW + NE) - 2.0 * N)) + abs((SW + SE) - 2.0 * S));
```

代码如下：

```c++
float lumaNW = FXAALuma(FXAATexOff(posM, int2(-1, 1)));
float lumaSE = FXAALuma(FXAATexOff(posM, int2(1, -1)));
float lumaNE = FXAALuma(FXAATexOff(posM, int2(1, 1)));
float lumaSW = FXAALuma(FXAATexOff(posM, int2(-1, -1)));

float lumaNS = lumaN + lumaS;
float lumaWE = lumaW + lumaE;
float edgeHorz1 = (-2.0 * lumaM) + lumaNS;
float edgeVert1 = (-2.0 * lumaM) + lumaWE;

float lumaNESE = lumaNE + lumaSE;
float lumaNWNE = lumaNW + lumaNE;
float edgeHorz2 = (-2.0 * lumaE) + lumaNESE;
float edgeVert2 = (-2.0 * lumaN) + lumaNWNE;

float lumaNWSW = lumaNW + lumaSW;
float lumaSWSE = lumaSW + lumaSE;
float edgeHorz4 = abs(edgeHorz1) * 2.0 + abs(edgeHorz2);
float edgeVert4 = abs(edgeVert1) * 2.0 + abs(edgeVert2);

float edgeHorz3 = (-2.0 * lumaW) + lumaNWSW;
float edgeVert3 = (-2.0 * lumaS) + lumaSWSE;

float edgeHorz = abs(edgeHorz3) + edgeHorz4;
float edgeVert = abs(edgeVert3) + edgeVert4;

bool horzSpan = edgeHorz >= edgeVert;
```

可以看到锯齿边缘的方向被区分了出来：

![21_FXAA_HorizontalVertical](/assets/images/2024/2024-03-04-RenderingNotes/21_FXAA_HorizontalVertical.png)

抗锯齿的原理就是通过原像素和其相邻像素之一混合来实现的，现在知道了锯齿边缘的方向，也就是混合的方向，现在就要开始计算混合因子了，它有个名字叫做亚像素混合因子 (Subpixel Blend Factor)，计算混合因子是通过权重
求原像素和相邻像素的平均，最后计算这个平均值和原像素亮度的差值来得到的：

```c++
// Total luminance of the neighborhood according to neighbor weights
float subpixNSWE = lumaNS + lumaWE;
float subpixNWSWNESE = lumaNWSW + lumaNESE;
float subpixA = subpixNSWE * 2.0 + subpixNWSWNESE;

// Calculate average of all adjacent neighbors and get the contrast between the middle and this average
float subpixB = (subpixA * (1.0 / 12.0)) - lumaM;

// Normalized the contrast, clamp the result to a maximum of 1
float subpixRcpRange = 1.0 / range;
float subpixC = saturate(abs(subpixB) * subpixRcpRange);

// Make factor more smoother
float subpixD = smoothstep(0, 1, subpixC);
float subpixH = subpixD * subpixD * FXAA_QUALITY__SUBPIX;
```

得到了混合因子后，最后需要计算最终的具体混合方向，前面已经知道了锯齿边缘是水平的还是垂直的，但是水平还需要区分水平向右还是向左，垂直还需要区分向上还是向下，
首先混合的单位步长是一个像素的长度，根据锯齿边缘是水平还是垂直选择 `U` 方向还是 `V` 方向的单位像素长度：

```c++
// Step length: horizontal edge, 1.0 / screenHeightInPixels, vertical edge, 1.0 / screenWidthInPixels
float lengthSign = _SourceSize.z;
if (horzSpan)
    lengthSign = _SourceSize.w;
```

然后根据锯齿边缘方向两边的梯度变化来确定最终的混合方向，根据锯齿边缘的方向分别取不同的正方向和负方向，锯齿边缘是水平时，正方向为正上，负方向为正下；锯齿边缘是垂直时，正方形是正右，负方向是正左，再通过和
中心像素的亮度值相减来得到正负两个方向的梯度变化，取更大的梯度变化方向为最终的混合方向：

```c++
// Calculate the gradient of positive direction and negative direction
// N means positive direction and S means negative direction
// horizontal edge: positive = north, negative = south, vertical edge: positive = east, negative = west
if (!horzSpan)
{
    lumaN = lumaE;
    lumaS = lumaW;
}
float gradientN = lumaN - lumaM;
float gradientS = lumaS - lumaM;

// Two direction gradients indicate step length direction
bool pairN = abs(gradientN) >= abs(gradientS);
if (!pairN) lengthSign = -lengthSign;
```

最后，就可以做混合了：

```c++
if (!horzSpan) posM.x += subpixH * lengthSign;
if (horzSpan) posM.y += subpixH * lengthSign;
return half4(FXAATex(posM).rgb, 1.0);
```

下面是关闭和开启抗锯齿的效果对比：

![22_FXAA_SubpixelBlendFactor_Off](/assets/images/2024/2024-03-04-RenderingNotes/22_FXAA_SubpixelBlendFactor_Off.png)

![23_FXAA_SubpixelBlendFactor_On](/assets/images/2024/2024-03-04-RenderingNotes/23_FXAA_SubpixelBlendFactor_On.png)

观察抗锯齿之后的效果，发现斜向的锯齿，抗锯齿效果其实不太好。这是因为亚像素混合因子是在中心像素周围 3x3 的像素块内确认的，并且假设锯齿边缘是完全垂直或者水平的，但实际情况下锯齿边缘往往远远
大于 3 个像素的范围，像素可能位于这个倾斜锯齿边缘的某一部分，虽然局部来看是水平或者垂直的，但是真正的锯齿边缘是有一定长度和倾斜角度的。所以需要引入一个新的混合因子：边缘混合因子。

计算边缘混合因子的原理是，从中心像素开始，沿着锯齿边缘的方向，分别向正、负两个方向一步步移动搜索 (水平的锯齿边缘搜索的正、负方向分别是正右和正左；垂直的锯齿边缘搜索的正、负方向分别是正上和正下)，
当移动到的锯齿边缘上的某个像素和混合方向上的相邻像素的平均值与中心像素和混合方向上相邻像素的亮度平均值的梯度变化大于某个阈值时 (因为搜索次数是有限的，所以到达搜索上限也会停止继续移动搜索)，
认为是找到了此锯齿边缘在这个搜索方向上的边缘。找到两个方向上的边缘后，取更靠近的一个方向，根据移动的距离计算出边缘混合因子。下面来看具体的代码实现：

需要注意的是，利用纹理的双线性过滤的特性，可以把采样点移动到锯齿边缘上的像素和混合方向上相邻像素之间的边缘上，这样通过一次采样就可以得到它们之前的平均值。如下图，红色方框的像素，移动采样位置到绿色X处，这样一次采样
就可以得到红色方框像素和混合方向上相邻正上方像素的平均值：

![24_FXAA_SamplePoint](/assets/images/2024/2024-03-04-RenderingNotes/24_FXAA_SamplePoint.png)

根据锯齿边缘水平还是垂直，偏移 0.5 个像素来确定搜索采样的起始 UV 坐标，以及定义每一次搜索的偏移大小：

```c++
// Determine the start UV coordinates on the edge between pixels, which is half step length away from the original UV coordinates
float2 posB;
posB.x = posM.x;
posB.y = posM.y;
if (!horzSpan) posB.x += lengthSign * 0.5;
if (horzSpan) posB.y += lengthSign * 0.5;

// Search step offset
float2 offNP;
offNP.x = (!horzSpan) ? 0.0 : _SourceSize.z;
offNP.y = (horzSpan) ? 0.0 : _SourceSize.w;
```

确定判断找到边缘的阈值，在这里是正、负方向上梯度变化的最大值的四分之一：

```c++
// Gradient threshold for determining that searching has gone off the edge
float gradient = max(abs(gradientN), abs(gradientS));
float gradientScaled = gradient * 1.0 / 4.0;
```

第一次分别向正、负两个方向移动 UV 坐标并判断是否找到边缘：

```c++
// 根据设定好的步长，分别向正、负两个方向移动 UV 坐标
float2 posN;
posN.x = posB.x - offNP.x * FXAA_EDGE_SEARCH_STEP0;
posN.y = posB.y - offNP.y * FXAA_EDGE_SEARCH_STEP0;
float2 posP;
posP.x = posB.x + offNP.x * FXAA_EDGE_SEARCH_STEP0;
posP.y = posB.y + offNP.y * FXAA_EDGE_SEARCH_STEP0;

// 采样得到当前采样点两边的相邻像素的亮度平均值
float lumaEndN = FXAALuma(FXAATex(posN));
float lumaEndP = FXAALuma(FXAATex(posP));

// 中心像素与混合方向上相邻像素的和
float lumaNN = lumaN + lumaM;
float lumaSS = lumaS + lumaM;
if (!pairN)
    lumaNN = lumaSS;

// 计算与中心像素与其混合方向上相邻像素平均值的梯度变化
lumaEndN -= lumaNN * 0.5;
lumaEndP -= lumaNN * 0.5;

// 当梯度变化大于阈值时，确定找到边缘
bool doneN = abs(lumaEndN) >= gradientScaled;
bool doneP = abs(lumaEndP) >= gradientScaled;

// 是否有一个方向没有找到边缘
bool doneNP = (!doneN) || (!doneP);
```

![25_FXAA_Search](/assets/images/2024/2024-03-04-RenderingNotes/25_FXAA_Search.png)

持续搜索，具体的搜索次数和搜索步长取决于对性能上的考虑，更多的搜索次数以及更小的搜索步长会使结果更加准确，但同时也意味着更高的性能消耗：

```c++
UNITY_UNROLL
for (int i = 0; i < FXAA_EXTRA_EDGE_SEARCH_STEPS && doneNP; ++i)
{
    if (!doneN) posN.x -= offNP.x * SEARCH_STEPS[i];
    if (!doneN) posN.y -= offNP.y * SEARCH_STEPS[i];
    if (!doneP) posP.x += offNP.x * SEARCH_STEPS[i];
    if (!doneP) posP.y += offNP.y * SEARCH_STEPS[i];

    if (!doneN) lumaEndN = FXAALuma(FXAATex(posN));
    if (!doneP) lumaEndP = FXAALuma(FXAATex(posP));
    if (!doneN) lumaEndN -= lumaNN * 0.5;
    if (!doneP) lumaEndP -= lumaNN * 0.5;

    doneN = abs(lumaEndN) >= gradientScaled;
    doneP = abs(lumaEndP) >= gradientScaled;

    doneNP = (!doneN) || (!doneP);
}
```

搜索结束后，得到中心像素距离两个方向上边缘的长度。取距离更小的方向，根据这个方向上的距离来计算边缘混合因子。最后还有一项额外的检查，要确保中心像素在混合方向上亮度值的梯度变化和在边缘混合方向上亮度值的梯度变化是相反的，只有满足
这个才需要做边缘混合，看下图中的红色点标注的像素：

![26_FXAA_RedGreen_0](/assets/images/2024/2024-03-04-RenderingNotes/26_FXAA_RedGreen_0.png)

计算可以知道此时是一个水平的锯齿边缘，而且上方的绿色点标注的像素亮度值(1)是大于红色点像素的亮度值(0)的，所以此时的混合方向和搜索边缘的正、负两个方向如下：

![27_FXAA_RedGreen_1](/assets/images/2024/2024-03-04-RenderingNotes/27_FXAA_RedGreen_1.png)

在正方向上，1 次就找到了边缘，而在负方向上，搜索 7 次找到了边缘，取搜索次数更少的方向作为计算边缘混合因子的方向，即正方向，但此时，中心像素也就是红色点像素在混合方向上的亮度值变化梯度是 `0.0 - (0.0 + 1.0) * 0.5 = -0.5`，而
正方向边缘的亮度值变化梯度是 `(0.0 + 0.0) * 0.5 - (0.0 + 1.0) * 0.5 = -0.5`，因为两个梯度变化是相同的，都为负，所以此红色点像素不做边缘混合。

下面再看绿色点标注的像素，它的混合方向和搜索边缘的正、负两个方向如下：

![28_FXAA_RedGreen_2](/assets/images/2024/2024-03-04-RenderingNotes/28_FXAA_RedGreen_2.png)

绿色点像素也是在正方向上 1 次就找到了边缘，在负方向上 7 次找到了边缘，还是取正方向作为计算边缘混合因子的方向。此时，中心像素也就是绿色点像素在混合方向上亮度值变化的梯度是 `1.0 - (0.0 + 1.0) * 0.5 = 0.5`，正方向
边缘的亮度值变化梯度是 `(0.0 + 0.0) * 0.5 - (1.0 + 0.0) * 0.5 = -0.5`，两个梯度变化是相反的，所以应用边缘混合。最终抗锯齿的效果如下图：

![29_FXAA_RedGreen_3](/assets/images/2024/2024-03-04-RenderingNotes/29_FXAA_RedGreen_3.png)

总结一下就是，***只有满足中心像素亮度值与搜索起点的亮度值的梯度变化和边缘混合方向上的边缘点与搜索起点的亮度值的梯度变化相反时，才需要应用边缘混合。其中，搜索起点的亮度值也就是中心像素亮度值与混合方向上相邻像素亮度值的平均值***。

下面是完整的代码：

```c++
// 分别到两个方向边缘的距离
float dstN = posM.x - posN.x;
float dstP = posP.x - posM.x;
if (!horzSpan) dstN = posM.y - posN.y;
if (!horzSpan) dstP = posP.y - posM.y;

// 中心像素在混合方向上亮度变化梯度值
float lumaMM = lumaM - lumaNN * 0.5;

// 中心像素在混合方向上亮度变化梯度，是否小于0
bool lumaMLTZero = lumaMM < 0.0;

// 负方向上亮度变化梯度是否和中心像素在混合方向上亮度变化梯度相反
bool goodSpanN = (lumaEndN < 0.0) != lumaMLTZero;
// 正方向上亮度变化梯度是否和中心像素在混合方向上亮度变化梯度相反
bool goodSpanP = (lumaEndP < 0.0) != lumaMLTZero;

// 取距离边缘更近的方向作为边缘混合方向
bool directionN = dstN < dstP;
// 是否需要边缘混合
bool goodSpan = directionN ? goodSpanN : goodSpanP;

// 计算边缘混合因子
// edgeBlendFactor = 0.5 - min(dstN, dstP) / (dstN + dstP)
float spanLength = (dstP + dstN);
float spanLengthRcp = 1.0 / spanLength;
float dst = min(dstN, dstP);
float pixelOffset = (dst * (-spanLengthRcp)) + 0.5;
float pixelOffsetGood = goodSpan ? pixelOffset : 0.0;
```

最后，应用边缘混合因子和亚像素混合因为，取它们中更大的作为混合因子，偏移中心像素的 UV 坐标采样，得到抗锯齿的结果：

```c++
// Apply both edge and subpixel blending, use the largest blend factor of both
float pixelOffsetSubpix = max(pixelOffsetGood, subpixH);

if (!horzSpan) posM.x += pixelOffsetSubpix * lengthSign;
if (horzSpan) posM.y += pixelOffsetSubpix * lengthSign;

return half4(FXAATex(posM).rgb, 1.0);
```

可以看到应用了边缘混合因子后，抗锯齿的效果提高了不少：

![30_FXAA_SubpixelAndEdgeBlendFactor](/assets/images/2024/2024-03-04-RenderingNotes/30_FXAA_SubpixelAndEdgeBlendFactor.png)

### 参考

- [FXAA](https://catlikecoding.com/unity/tutorials/custom-srp/fxaa/#3.8)
- [Implementing FXAA](https://blog.simonrodriguez.fr/articles/2016/07/implementing_fxaa.html)
- [FXAA3_11](https://github.com/hghdev/NVIDIAGameWorks-GraphicsSamples/blob/master/samples/es3-kepler/FXAA/FXAA3_11.h)

---

## Normal Mapping without Precomputed Tangents

在使用法线纹理时，因为法线纹理中存储的法线方向是在切线空间中的，所谓的切线空间如下图所示，其中，原点对应了顶点坐标，红色向量是切线方向（Tangent），绿色向量是副切线方向（Bitangent），蓝色向量是法线方向（Normal），这 3 个向量是正交的，它们之前互相垂直：

![05_Tangent_Space](/assets/images/2024/2024-03-04-RenderingNotes/05_Tangent_Space.png)

一般情况下，模型文件的顶点数据中会存有法线和切线的信息，在实际的渲染中，首先将模型数据中存储的的法线和切线转换到世界空间，因为切线，副切线和法线是正交的，所以可以通过叉乘计算出世界空间下的副切线。在计算出副切线后，通过构建 TBN 矩阵就可以将法线纹理中存储在切线空间中的法线变换到世界空间中。有了世界空间下的法线，就可以进行光照计算了。

上面说的是一般情况下，如果没有切线信息呢？其实可以通过顶点和纹理坐标（纹理坐标也是定义在切线空间的）计算出切线，下面看下是如何推导的。

首先来看下面一张图，它展示了一个表面的 T，B 和 N 这 3 个向量：

![06_Tangent_Calculation_0](/assets/images/2024/2024-03-04-RenderingNotes/06_Tangent_Calculation_0.jpeg)

在这个表面上，取 3 个点， $P_{1}$ ， $P_{2}$ 和 $P_{3}$ ，如下图所示：

![07_Tangent_Calculation_1](/assets/images/2024/2024-03-04-RenderingNotes/07_Tangent_Calculation_1.jpeg)

其中， $\Delta{U_{2}}$ 与切线向量 `T` 方向相同， $\Delta{V_{2}}$ 与副切线向量 `B` 方向相同。也就是说，可以将上图中的三角形的边 $E_{1}$ 和 $E_{2}$ 表示成向量 `T` 和向量 `B` 的线性组合：

$$
E_{1} = \Delta{U_{1}}T + \Delta{V_{1}}B \\
E_{2} = \Delta{U_{2}}T + \Delta{V_{2}}B
$$

也可以写成这样：

$$
(E_{1x}, E_{1y}, E_{1z}) = \Delta{U_{1}}(T_{x}, T_{y}, T_{z}) + \Delta{V_{1}}(B_{x}, B_{y}, B_{z}) \\
(E_{2x}, E_{2y}, E_{2z}) = \Delta{U_{2}}(T_{x}, T_{y}, T_{z}) + \Delta{V_{2}}(B_{x}, B_{y}, B_{z})
$$

$E$ 是 2 个顶点坐标的差，而 $\Delta{U}$ 和 $\Delta{V}$ 是这 2 个顶点 `UV` 坐标的差。观察这 2 个等式，顶点坐标的差和顶点 `UV` 的差都是已知的，所以可以根据这 2 个等式来求需要的 2 个向量 `T` 和 `B`。上面的这 2 个等式可以写成矩阵的形式：

$$
\begin{bmatrix}
    E_{1x} & E_{1y} & E_{1z} \\
    E_{2x} & E_{2y} & E_{2z}
\end{bmatrix} =
\begin{bmatrix}
    \Delta{U_{1}} & \Delta{V_{1}} \\
    \Delta{U_{2}} & \Delta{V_{2}}
\end{bmatrix}
\begin{bmatrix}
    T_{x} & T_{y} & T_{z} \\
    B_{x} & B_{y} & B_{z}
\end{bmatrix}
$$

等式 2 边都乘以 $\Delta{U}\Delta{V}$ 矩阵的逆矩阵，因为一个矩阵乘以它的逆矩阵等于单位矩阵，所以可以得到：

$$
{\begin{bmatrix}
    \Delta{U_{1}} & \Delta{V_{1}} \\
    \Delta{U_{2}} & \Delta{V_{2}}
\end{bmatrix}}^{-1}
\begin{bmatrix}
    E_{1x} & E_{1y} & E_{1z} \\
    E_{2x} & E_{2y} & E_{2z}
\end{bmatrix} =
({\begin{bmatrix}
    \Delta{U_{1}} & \Delta{V_{1}} \\
    \Delta{U_{2}} & \Delta{V_{2}}
\end{bmatrix}}^{-1}
\begin{bmatrix}
    \Delta{U_{1}} & \Delta{V_{1}} \\
    \Delta{U_{2}} & \Delta{V_{2}}
\end{bmatrix})
\begin{bmatrix}
    T_{x} & T_{y} & T_{z} \\
    B_{x} & B_{y} & B_{z}
\end{bmatrix} =
\begin{bmatrix}
    T_{x} & T_{y} & T_{z} \\
    B_{x} & B_{y} & B_{z}
\end{bmatrix}
$$

对于 2x2 的小矩阵 $\Delta{U}\Delta{V}$，可以使用伴随矩阵法（把此矩阵变化为，1 除以此矩阵的行列式，再乘以它的伴随矩阵（Adjugate Matrix））求它的逆矩阵，最终可以得到表示要求的切线向量 `T` 和副切线向量 `B`：

$$
\begin{bmatrix}
    T_{x} & T_{y} & T_{z} \\
    B_{x} & B_{y} & B_{z}
\end{bmatrix} =
\frac{1}{\Delta{U_{1}}\Delta{V_{2}} - \Delta{V_{1}}\Delta{U_{2}}}
\begin{bmatrix}
    \Delta{V_{2}} & -\Delta{V_{1}} \\
    -\Delta{U_{2}} & \Delta{U_{1}}
\end{bmatrix}
\begin{bmatrix}
    E_{1x} & E_{1y} & E_{1z} \\
    E_{2x} & E_{2y} & E_{2z}
\end{bmatrix}
$$

最后是一个在 OpenGL 实时渲染中，通过顶点坐标和顶点 `UV` 坐标在 $x$ 和 $y$ 方向上梯度的变化计算切线向量和副切线向量的例子：

```glsl
vec3 getNormal()
{
    // 采样法线贴图，并将法线向量从 [0, 1] 转换到 [-1, 1]
    vec3 tangentNormal = texture(uNormalMap, fs_in.UV0).xyz * 2.0 - 1.0;

    // 计算当前像素世界坐标在 x 方向的梯度变化
    vec3 ddxPos = dFdx(fs_in.WorldPos);
    // 计算当前像素世界坐标在 y 方向的梯度变化
    vec3 ddyPos = dFdy(fs_in.WorldPos);

    // 计算当前像素 UV 坐标在 x 方向的偏梯度变化
    vec2 ddxUV = dFdx(fs_in.UV0);
    // 计算当前像素 UV 坐标在 y 方向的梯度变化
    vec2 ddyUV = dFdy(fs_in.UV0);

    // 法线
    vec3 N = normalize(fs_in.Normal);
    // 切线
    vec3 T = normalize(ddxPos * ddyUV.t - ddyPos * ddxUV.t);
    // 副切线
    vec3 B = normalize(ddyPos * ddxUV.s - ddyPos * ddyUV.s);

    // 构建 TBN 矩阵
    mat3 TBN = mat3(T, B, N);

    return normalize(TBN * tangentNormal);
}
```

---

## 通过一个四元数存储基础法线向量(0, 0, 1)、基础切线向量(1, 0, 0)与基础副切线向量(0, 1, 0)的旋转信息

今天在看 [flament](https://github.com/google/filament) 源码时，看到一个有意思的算法：通过模型数据中的切线，副切线，法线信息，构建一个表示旋转的 `TBN` 矩阵，将此矩阵转换为四元数表示，最后将此四元数传入着色器中，在着色器中，通过此四元数对初始法线向量 `(0, 0, 1)` 和初始切线向量 `(1, 0, 0)` 进行旋转，得到模型空间中的法线和切线向量信息。其中，这个四元数的 `w` 分量的符号（sign，正或者负）还代表了切线向量的方向，因为切线空间必须是右手坐标系统（Right-hand coord system），而如果 $(\mathbf{n} \times \mathbf{t}) \cdot \mathbf{b} \leq 0.0$ 则代表此时的切线空间并不是右手坐标系统，需要根据 `w` 分量的符号将切线向量取反。

表示旋转的 `TBN` 矩阵怎么理解呢？在加载模型时得到了切线、法线数据后（副切线可以通过法线和切线叉乘得到），将切线（t）、副切线（b）和法线向量（n）构建一个 `TBN` 矩阵：

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
\end{bmatrix}
=
\begin{bmatrix}
    \mathbf{n}_x \\
    \mathbf{n}_y \\
    \mathbf{n}_z
\end{bmatrix}
$$

可以看到基础法线向量与构建的 `TBN` 矩阵相乘也就得到了模型空间中的法线向量。切线和副切线同理。

构建 `TBN` 矩阵的主要的算法在 `packTangentFrame` 方法中：

```c++
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

其中 第一个 `3x3` 的矩阵 `m` 参数通过模型数据中的切线（tangent）、副切线（bitangent）和法线（normal）构成，返回的四元数传入顶点数据中：

```c++
quatf q = filament::math::details::TMat33<float>::packTangentFrame({tangent, bitangent, normal});
asset.tangents.push_back(packSnorm16(q.xyzw));
```

第二个参数是为了计算四元数 `w` 分量的偏差，为了保证 `w` 分量不为 `0`，这是因为 GPU 不一定能表示 `-0` 这种数据。 `storageSize` 默认的大小是 `sizeof(int16_t)`。

下面来看看 `packTangentFrame` 函数：

- `TQuaternion<T> q = TMat33<T>{ m[0], cross(m[2], m[0]), m[2] }.toQuaternion();`，首先将 `TBN` 矩阵转换为四元数表示，其中 `m[0]` 是模型中的切线，`m[2]` 是法线，`cross(m[2], m[0])` 是副切线。
- `q = positive(normalize(q));`，先归一化四元数 `q`，并通过 `w` 分量的符号，来决定是否取反四元数 `q`，保证其 `w` 分量为正值，用以后续偏移的计算。需要注意的是： **当四元数是表示旋转时，对其进行取反，`q` 与 `-q` 是等价的，两者表示的都是相同的旋转**。而在这里，`TBN` 矩阵表示的就是一个旋转矩阵，所以 `q` 和 `-q` 是等价的。
- 将 `w` 分量限制在最小正值 `bias`，并保证 `q.xyz` 向量的模还是 `1`，也就是单位向量。
- 最后，根据模型中切线，副切线和法线之间的方向关系来确定 `w` 分量的符号。当 `dot(cross(m[0], m[2]), m[1]) <= T(0)` 时，表示构建 `TBN` 的三个向量并不是在右手坐标系统下，此时再次将四元数 `q` 取反，在后续着色器的计算中，通过 `w` 分量的符号来保证 `TBN` 矩阵的正确性。

在着色器中，需要做的是通过这个四元数 `q` 来解析出模型空间的切线、副切线和法线数据，其中 `q` 就是计算好的代表旋转的四元数：

```c++
/**
 * Extracts the normal vector of the tangent frame encoded in the specified quaternion.
 */
void toTangentFrame(const highp vec4 q, out highp vec3 n) {
    n = vec3( 0.0,  0.0,  1.0) +
        vec3( 2.0, -2.0, -2.0) * q.x * q.zwx +
        vec3( 2.0,  2.0, -2.0) * q.y * q.wzy;
}

/**
 * Extracts the normal and tangent vectors of the tangent frame encoded in the
 * specified quaternion.
 */
void toTangentFrame(const highp vec4 q, out highp vec3 n, out highp vec3 t) {
    toTangentFrame(q, n);
    t = vec3( 1.0,  0.0,  0.0) +
        vec3(-2.0,  2.0, -2.0) * q.y * q.yxw +
        vec3(-2.0,  2.0,  2.0) * q.z * q.zwx;
}
```

---

## 计算法线变换矩阵的另一种方法

<!-- TODO -->

<!-- * Note that the inverse-transpose of a matrix is equal to its cofactor matrix divided by its
* determinant:
*
*     transpose(inverse(M)) = cof(M) / det(M)
*
* The cofactor matrix is faster to compute than the inverse-transpose, and it can be argued
* that it is a more correct way of transforming normals anyway. Some references from Dale
* Weiler, Nathan Reed, Inigo Quilez, and Eric Lengyel:
*
*   - https://github.com/graphitemaster/normals_revisited
*   - http://www.reedbeta.com/blog/normals-inverse-transpose-part-1/
*   - https://www.shadertoy.com/view/3s33zj
*   - FGED Volume 1, section 1.7.5 "Inverses of Small Matrices"
*   - FGED Volume 1, section 3.2.2 "Transforming Normal Vectors"
*
* In "Transforming Normal Vectors", Lengyel notes that there are two types of transformed
* normals: one that uses the transposed adjugate (aka cofactor matrix) and one that uses the
* transposed inverse. He goes on to say that this difference is inconsequential, except when
* mirroring is involved. -->

在之前的笔记中，有说到[法线变换矩阵](#normal-matrix法线变换矩阵)，经过推导使用的是 `transpose(inverse(M))` 矩阵。

- https://github.com/graphitemaster/normals_revisited
- http://www.reedbeta.com/blog/normals-inverse-transpose-part-1/
- https://www.shadertoy.com/view/3s33zj
- FGED Volume 1, section 1.7.5 "Inverses of Small Matrices"
- FGED Volume 1, section 3.2.2 "Transforming Normal Vectors"

---

## SSR

<!-- TODO -->

---

## TAA

<!-- TODO -->

---
