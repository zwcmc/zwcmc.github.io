---
layout: post
title:  "Unity中的LinearEyeDepth和Linear01Depth推导"
date:   2023-07-03 16:16:00 +0800
category: Unity
---

- [Unity 中的这两个函数](#unity-中的这两个函数)
- [LinearEyeDepth](#lineareyedepth)
- [Linear01Depth](#linear01depth)

### Unity 中的这两个函数

先看这2个函数：

```c
// Z buffer to linear 0..1 depth (0 at camera position, 1 at far plane).
// Does NOT work with orthographic projections.
// Does NOT correctly handle oblique view frustums.
// zBufferParam = { (f-n)/n, 1, (f-n)/n*f, 1/f }
float Linear01Depth(float depth, float4 zBufferParam)
{
    return 1.0 / (zBufferParam.x * depth + zBufferParam.y);
}

// Z buffer to linear depth.
// Does NOT correctly handle oblique view frustums.
// Does NOT work with orthographic projection.
// zBufferParam = { (f-n)/n, 1, (f-n)/n*f, 1/f }
float LinearEyeDepth(float depth, float4 zBufferParam)
{
    return 1.0 / (zBufferParam.z * depth + zBufferParam.w);
}
```

其中，`zBufferParam` 是 `_ZBufferParams` ，定义在 UnityInput.hlsl 文件：

```c
// Values used to linearize the Z buffer (http://www.humus.name/temp/Linearize%20depth.txt)
// x = 1-far/near
// y = far/near
// z = x/far
// w = y/far
// or in case of a reversed depth buffer (UNITY_REVERSED_Z is 1)
// x = -1+far/near
// y = 1
// z = x/far
// w = 1/far
float4 _ZBufferParams;
```

可以知道，最主要的就是这个 `_ZBufferParams` 。下面来推导它的四个分量分别是怎么计算出来的。

### LinearEyeDepth

深度纹理中存储的是经过透视变换和透视除法后的 NDC 空间中的 `z` 值。

下面以 OpenGL 为例，转换后的 `z` 值区间为 $[-1, 1]$ ，再使用下面的公式把 `z` 映射到 $[0, 1]$ ，最终存储在深度纹理中：

$$
z_d = 0.5 \cdot z_{ndc} + 0.5
$$

首先，透视投影矩阵为，其中 f 代表远平面，n 代表近平面，后面都这么表示：

$$
M_{persp}=
\begin{bmatrix}
\frac{\cot{(\frac{FOV}{2})}}{Asepct} & 0 & 0 & 0\\
0 & \cot{(\frac{FOV}{2})} & 0 & 0\\
0 & 0 & -\frac{f + n}{f - n} & -\frac{2  \cdot  n  \cdot  f}{f - n}\\
0 & 0 & -1 & 0
\end{bmatrix}
$$

假设经过观察矩阵变换后的 `z` 值为 $z$，则经过透视变换后：

$$
\begin{bmatrix}
\frac{\cot{(\frac{FOV}{2})}}{Asepct} & 0 & 0 & 0\\
0 & \cot{(\frac{FOV}{2})} & 0 & 0\\
0 & 0 & -\frac{f + n}{f - n} & -\frac{2  \cdot  n  \cdot  f}{f - n}\\
0 & 0 & -1 & 0
\end{bmatrix}
\cdot
\begin{bmatrix}
x\\
y\\
z\\
1
\end{bmatrix}
$$

得到：

$$
z_{persp} = -z \cdot \frac{f + n}{f - n} - \frac{2 \cdot n \cdot f}{f - n}
$$

此时的 `w` 分量为 $-z$，所以经过透视除法后，NDC 空间下的 `z` 为：

$$
z_{ndc} = \frac{f + n}{f - n} + \frac{2 \cdot n \cdot f}{z \cdot (f - n )}
$$

继续做映射，把 `z` 的范围映射到 $[0, 1]$，推导如下：

![00_z_to_0_1_opengl](/assets/images/2023/2023-07-03-LinearEyeDepthAndLinear01DepthInUnity/00_z_to_0_1_opengl.jpg)

得到经过投影变换和透视除法后的非线性存储在深度纹理中的深度 $z_{d}$ 为：

$$
z_{d} = \frac{f + \frac{nf}{z}}{f - n}
$$

所以，观察空间的 `z` 可以用深度纹理中存储的非线性深度 $z_{d}$ 表示为：

$$
z = \frac{nf}{z_{d} \cdot (f - n) - f} = \frac{1}{z_{d} \cdot \frac{f - n}{nf} - \frac{1}{n}}
$$

由于在 Unity 使用的视角空间中，摄像机正向对应的 `z` 值为负值，因此为了得到深度值的正数表示，需要对上面的结果取反，最后得到的结果为：

$$
z = \frac{1}{z_{d} \cdot \frac{n - f}{nf} + \frac{1}{n}}
$$

所以，`LinearEyeDepth` 计算的就是上面公式的内容，采样出深度纹理中的深度后，转换到观察空间中的正的深度值，`_ZBufferParams` 的 `z` 和 `w` 分量分别存储了 $\frac{n - f}{nf}$ 和 $\frac{1}{n}$ 的值。

这是在 OpenGL 的环境下的推导结果，在 DirectX，Vulkan，Metal 环境下，NDC空间下的 `z` 范围为 $[0, 1]$，但在上面的推导过程中，已经把 OpenGL 下 $[-1, 1]$ 的深度值转换到 $[0, 1]$ 了，所以最终计算的结果是一样的。推导过程如下：

![01_z_to_0_1_not_opengl](/assets/images/2023/2023-07-03-LinearEyeDepthAndLinear01DepthInUnity/01_z_to_0_1_not_opengl.jpeg)

还有一种情况就是反向Z。反向Z其实就是在NDC空间，`z` 的值是 $[1, 0]$ ，也就是在Near处 `z` 为 1，而在Far处，`z` 为 0。可以理解为：

$$
1.0 - z_{ndc} = \frac{-f}{n - f} - \frac{nf}{(n - f) \cdot z}
$$

推导如下：

![02_z_to_1_0_not_opengl](/assets/images/2023/2023-07-03-LinearEyeDepthAndLinear01DepthInUnity/02_z_to_1_0_not_opengl.jpg)

可见和源码中的`_ZBufferParams` 的 `z` 和 `w` 分量注释是一样的，推导正确。

### Linear01Depth

上面推导的 LinearEyeDepth 其实就是把经过非线性的投影变换和透视除法后的 `z` 变换到观察空间中的 `z`，此时它的取值范围是 [0, f]，此时，近平面的 `z` 为 $-n$，而远平面的 `z` 为 $-f$，而相机的 `z` 为 0。而想要转换到 [0, 1]，即把上面得到的结果除以 f 就可以了。

所以，没有使用反向Z时：

$$
z_{01} = \frac{1}{z_{d} \cdot \frac{n - f}{n} + \frac{f}{n}}
$$

使用反向Z时：

$$
z_{01} = \frac{1}{z_{d} \cdot \frac{f- n}{n} + 1}
$$

这2个值分别存储在了 `_ZBufferParams` 的 `x` 和 `y` 分量，也可以看到和注释中是一样的。
