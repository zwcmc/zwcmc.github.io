---
layout: post
title:  "OpenGL学习笔记"
date:   2024-07-31 16:16:00
category: OpenGL
---

语言为 C++，因为要兼容 macOS，所以 OpenGL 版本为 4.1

- [std140](#std140)
- [伽马校正（Gamma Correction）](#伽马校正gamma-correction)
  - [OpenGL 中怎么进行伽马校正](#opengl-中怎么进行伽马校正)
  - [sRGB 纹理](#srgb-纹理)
    - [对 sRGB 纹理转换到线性空间的方法](#对-srgb-纹理转换到线性空间的方法)
- [GL\_TRIANGLE\_STRIP](#gl_triangle_strip)
- [全屏渲染时，使用 3 个顶点的三角形覆盖整个屏幕](#全屏渲染时使用-3-个顶点的三角形覆盖整个屏幕)
- [天空盒（Skybox）渲染中的一些技巧分析](#天空盒skybox渲染中的一些技巧分析)
  - [Radiance HDR 渲染到立方体纹理](#radiance-hdr-渲染到立方体纹理)

## std140

`std140` 是 OpenGL 中统一缓冲区块（Uniform Block）数据的对齐和布局的一种标准化格式，统一缓冲区块是一种用于在 CPU 和着色器之间高效传递一组统一变量的机制，它的数据存储在统一缓冲区对象（Uniform Buffer Object，UBO）中。统一缓冲区块的内容实际是存储在 GPU 上的一块全局内存，这块内存并不会保存它所持有的数据类型的信息，所以通过 `glBufferSubData` 方法更新对象数据时，需要告诉 OpenGL 这块内存的哪些部分对应着色器中的哪些统一变量，也就是设置每个变量的大小（以字节为单位，in bytes）以及它们在内存块开始位置的偏移量。

每个元素的大小都是在 OpenGL 中有清楚的声明的，而且是直接对应 C++ 数据类型，其中向量和矩阵是由 `n` 个 `float` 组成的数组，然而 OpenGL 没有声明这些变量之间的间距，这允许硬件根据需要在它认为合适的位置放置变量，例如，一些硬件可能会将一个 `vec3` 放置在 `float` 边上，在附加这个 `float` 之前，先将 `vec3` 填充（Padding）为一个包含 4 个 `float` 的数组。

而使用 `std140` 布局，它通过一套规则声明了每个变量的偏移量，明确规定了每个变量类型的内存布局，这样在使用时，可以通过手动计算出每个变量的偏移量。在 `std140` 布局规则下，每个变量都有一个基准对齐量（Base Alignment），它等于这个变量在统一缓冲区块中所占用的空间大小（包括填充），对于每个变量，需要计算它的对齐偏移量（Aligned Offset），对齐偏移量描述的是这个变量从统一缓冲区块开始位置的字节偏移量。一个变量的对齐偏移量**必须等于其基准对齐量的倍数**。

布局规则的原文可以在 OpenGL 的统一缓冲区对象规范[这里](http://www.opengl.org/registry/specs/ARB/uniform_buffer_object.txt)找到。下面是一些常见变量的规则：

1. **标量类型（如 `float`、`int`、`bool`）**：
   - 基准对齐量：4 字节。
   - 大小：4 字节。
2. **向量类型（如 `vec2`、`vec3`、`vec4`）**：
   - `vec2` 的基准对齐量是 8 字节，大小为 8 字节。
   - `vec3` 和 `vec4` 的基准对齐量是 16 字节，`vec3` 的大小为 12 字节，但会填充到 16 字节。`vec4` 的大小为 16 字节。
3. **矩阵类型（如 `mat2`、`mat3`、`mat4`）**：
   - 矩阵按列主序存储，每列看作一个向量。
   - `mat2` 的基准对齐量是每列 1 个 `vec2`，即每列 8 个字节，总大小为 `2 * 8` 字节。
   - `mat3` 的基准对齐量是每列 1 个 `vec4`，即每列 16 个字节，总大小为 `3 * 16` 字节。
   - `mat4` 的基准对齐量是每列 1 个 `vec4`，即每列 16 个字节，总大小为 `4 * 16` 字节。
4. **数组类型**：
   - 数组元素的基准对齐量与其基础类型相同，但每个元素的大小按基础类型的基准对齐量对齐。
   - 例如，`float` 数组，虽然每个 `float` 只占 4 个字节，但 `float` 数组的每个元素大小对齐到 16 字节。
5. **结构体**：
   - 结构体的基准对齐量是其所有成员中基准对齐量最大的那个。
   - 结构体的每个成员按其自身的基准对齐量对齐。
   - 结构体的大小填充到其基准对齐量的倍数。

下面展示了一个使用 `std140` 布局的例子：

```glsl
layout (std140) uniform ExampleBlock
{
                     // 基准对齐量        // 对齐偏移量
    float value;     // 4               // 0 
    vec3 vector;     // 16              // 16
    mat4 view;       // 16              // 32  (列 0)
                     // 16              // 48  (列 1)
                     // 16              // 64  (列 2)
                     // 16              // 80  (列 3)
    float values[3]; // 16              // 96  (values[0])
                     // 16              // 112 (values[1])
                     // 16              // 128 (values[2])
    bool boolean;    // 4               // 144
    int integer;     // 4               // 148
};
```

上面例子的具体分析：

1. **float value**：
   - 基准对齐量：4 字节
   - 对齐偏移量：0（统一缓冲区块开始位置）
   - 大小：4 字节
2. **vec3 vector**：
   - 基准对齐量：16 字节（`vec3` 按 `vec4` 对齐）
   - 对齐偏移量：16（`0 + 4 + 12 = 16`， `value` 占据了 4 字节，从偏移 0 到 3，根据 `std140` 的对齐规则，接下来的 `vec3` 需要在 16 字节对齐的边界上开始，因此，从偏移 4 到偏移 15 之前需要填充 12 字节的空间，以确保 `vec3` 的起始偏移是 16 的倍数）
   - 大小：12 字节
3. **mat4 view**：
   - 基准对齐量：16 字节（矩阵按列存储，每列是一个 `vec4`）
   - 对齐偏移量：32（`16 + 16 = 32`，`vector` 是 16 字节对齐，3 个 `float` 占用 12 字节，再填充 4 字节达到 16 字节边界）
   - 总大小：64 字节（4 个 `vec4`，每个 16 字节）
4. **float values[3]**：
   - 基准对齐量：16 字节（数组元素至少 16 字节对齐）
   - 对齐偏移量：96（`32 + 64 = 96`，`view` 占用 64 字节，所以下一个元素从 96 字节开始）
   - 总大小：48 字节（每个 `float` 按 16 字节对齐，共 3 个元素）
5. **bool boolean**：
   - 基准对齐量：4 字节（`bool` 按 `int` 对齐）
   - 对齐偏移量：144（`96 + 48 = 144`，`values` 占用 48 字节，所以下一个元素从 144 字节开始）
   - 大小：4 字节
6. **int integer**：
   - 基准对齐量：4 字节
   - 对齐偏移量：148（`144 + 4 = 148` ，`boolean` 占用 4 字节，所以下一个元素从 148 字节开始）
   - 大小：4 字节

最终，整个统一缓冲区块的大小为：`148 + 4 = 152` 字节。

---

## 伽马校正（Gamma Correction）

数字成像早期，大多数显示器都是阴极射线（CRT）显示器，它具有非线性亮度响应特性，输入电压加倍并不会导致亮度加倍，而是遵循一个大约 `2.2` 的指数关系（`2.2` 的伽马值是一个默认值，广泛应用于大多数现代显示器）。而人类对亮度的感知也遵循类似的指数关系，这与CRT显示器的伽马特性巧合地相匹配。下图是人眼感知的亮度与真实物理亮度的差异对比：

![00_gamma_correction_brightness.jpeg](/assets/images/2024/2024-08-02-OpenGLNotes/00_gamma_correction_brightness.jpeg)

可以看出来物理上亮度的加倍会返回正确的物理亮度，但人眼对不同的亮度感知却是不同的（对暗色变化更敏感），会感觉物理上亮度的变化有些奇怪。所以现代显示器通过指数关系来输出颜色，将原始的物理亮度颜色映射到人眼感知的非线性亮度颜色是很有必要的，这样的输出效果更符合人眼的感知。

这种显示器的非线性映射虽然更符合人眼的感知，但在图形渲染时存在一个问题：**在应用程序中配置的所有颜色和亮度都是基于从显示器中感知到的亮度，所以这些颜色和亮度都是非线性的，它们并不物理正确**。看下面的图：

![01_gamma_correction_gamma_curves](/assets/images/2024/2024-08-02-OpenGLNotes/01_gamma_correction_gamma_curves.jpeg)

上图中虚线表示的是线性空间中的颜色值，实线表示显示器显示的颜色空间。如果在线性空间中将一个值加倍，其结果确实是原值的 2 倍，举个例子，线性空间的一个值 `0.5`，将其加倍后，是 `1.0`，然而原始值 `0.5` 在显示器上显示的值为 `0.218`，也就是说这个值在显示器上变得亮度超过 4.5 倍（因为显示器和线性空间的起点和终点是相同的，也就是线性空间中的 `1.0` 在显示器显示的也是 `1.0`），这很明显是不对的。

因为渲染要保证物理正确，所以都是在线性空间计算。那么怎么让物理正确的渲染结果正确的显示在显示器上呢？答案就是伽马校正。

伽马校正的原理就是，将最终的输出颜色显示到显示器之前，应用显示器伽马的逆函数。还是上图的那个例子，线性空间的 `0.5` 值，在输出到显示器之前，对其应用伽马校正，也就是将其应用 $0.5^{\frac{1}{2.2}}$ 次方，伽马校正后的值传递到显示器，显示器显示的颜色值为： ${(0.5^{\frac{1}{2.2}})}^{2.2} = 0.5$ ，可以看到显示器最终显示的值和渲染时计算的值保持了一致。

### OpenGL 中怎么进行伽马校正

1. 通过 `glEnable(GL_FRAMEBUFFER_SRGB)` 告诉 OpenGL 在将颜色存储到颜色缓冲区之前，先对颜色（从sRGB颜色空间）进行伽马校正，sRGB是一种大致对应于2.2伽马值的颜色空间，是大多数设备的标准。在启用 `GL_FRAMEBUFFER_SRGB` 之后，OpenGL会自动对所有后续帧缓冲区（包括默认帧缓冲区）在每次片元着色器运行后进行伽马校正。
   - 优点：实现简单，交给硬件去做，性能消耗几乎可以忽略不计；
   - 缺点：控制少，无法自定义伽马校正过程；如果渲染过程中使用了多个帧缓冲区，那么就要确保只有最后一个帧缓冲区在发送到显示器之前应用伽马校正，因为伽马校正会将颜色从线性空间转换到非线性空间，如果不是最后一个帧缓冲区应用伽马校正，那么帧缓冲区之前传递的中间结果就不在线性空间了，会导致后续的对颜色的操作都基于不正确的值；
2. 在片元着色器中自己进行伽马校正。
   - 优点：对伽马校正的完全控制；
   - 缺点：每个片元着色器都需要做伽马校正，当每个片元着色器中都有伽马校正的计算逻辑时，对性能也会有一定的影响；

一般做法是，如果渲染流程中有后处理，可以在做完后处理后的最后一个帧缓冲区应用伽马校正，如果没有后处理，也可以创建一个帧缓冲区，先把所有的物体都渲染到这个帧缓冲区，最后在将此帧缓冲区发送到显示器之前对其应用伽马校正（Blit操作）。

### sRGB 纹理

由于显示器的颜色是应用了伽马值的，所以艺术家们创建或编辑的所有纹理都不是在线性空间的，而是在 sRGB 空间。因为纹理是基于显示器上看到的内容创建和编辑的，实际上对纹理的颜色值已经进行了伽马校正，以确保它在显示器上看起来正确，而在渲染时，如果有使用纹理，最终再次对颜色进行伽马校正后，就相当于对纹理中的颜色值进行了 2 次伽马校正。所以在对纹理中的颜色值进行任何计算之前，需要将这些 sRGB 纹理重新转换到线性空间。

#### 对 sRGB 纹理转换到线性空间的方法

1. 在着色器中采样纹理后，额外进行转换：

   ```glsl
   float gamma = 2.2;
   vec3 albedo = pow(texture(uTexture, uv0).rgb, vec3(gamma));
   ```

2. 通过 `glTexImage2D` 设置其内部格式为 `GL_SRGB` 或者 `GL_SRGB_ALPHA`：

   ```glsl
   glTexImage2D(GL_TEXTURE_2D, 0, GL_SRGB, width, height, 0, GL_RGB, GL_UNSIGNED_BYTE, data);
   // include alpha compenent
   glTexImage2D(GL_TEXTURE_2D, 0, GL_SRGB_ALPHA, width, height, 0, GL_RGBA, GL_UNSIGNED_BYTE, data);
   ```

最后还有一点要注意，并不是所有的纹理都需要指定为 sRGB 空间，基本上只有和颜色相关的纹理才会需要指定为 sRGB 空间（例如漫反射纹理，自发光纹理），而例如法线纹理，金属度/光滑度纹理都是在线性空间。

---

## GL_TRIANGLE_STRIP

在OpenGL中，`GL_TRIANGLE_STRIP` 是一种绘图模式，用于渲染一系列相连的三角形。与使用单独三角形（`GL_TRIANGLES`）相比，它减少了需要发送到 GPU 的数据量，所以它可以更高效地绘制三角形。

使用 `GL_TRIANGLE_STRIP` 时，前三个顶点定义第一个三角形。每一个后续顶点与前一个三角形的最后两个顶点共同定义一个新的三角形。这意味着每增加一个顶点，就会形成一个新的三角形，并与前一个三角形共享两个顶点。例如下面一组顶点数据：

```cpp
std::vector<vec3> quadVertices =
{
   vec3(-1.0f, 1.0f, 0.0f),   // 顶点1
   vec3(-1.0f, -1.0f, 0.0f),  // 顶点2
   vec3(1.0f, 1.0f, 0.0f),    // 顶点3
   vec3(1.0f, -1.0f, 0.0f)    // 顶点4
};
```

1. 第一个三角形由顶点（1, 2, 3）形成。
2. 第二个三角形由顶点（2, 3, 4）形成。

这样只需要 4 组顶点数据就能绘制 2 个三角形。

---

## 全屏渲染时，使用 3 个顶点的三角形覆盖整个屏幕

当在做全屏渲染时，比如离屏渲染、后处理渲染时，需要做 `Blit` 操作，正常情况下是绘制一个覆盖整个屏幕空间的四边形网格（2 个三角形）。然而，可以通过渲染一个包含整个屏幕的三角形来实现相同的效果，这样对性能有一定的优化。先看下图：

![02_quad_block_rendering](/assets/images/2024/2024-08-02-OpenGLNotes/02_quad_block_rendering.png)

使用单个三角形进行屏幕覆盖 `Blit` 操作时，对比上图中展示的四边形网格有如下优势：

1. **减少顶点数量**：顶点数从 6 个减少到 3个，甚至都不需要向 GPU 中传输顶点数据；
2. **消除对角线**：在GPU中，片段是并行处理的，通常以小块（如2x2或4x4）的形式处理。当使用两个三角形覆盖屏幕时，对角线两侧的片段块会被多次渲染，导致不必要的计算；
3. **更好的缓存一致性**：单个三角形的渲染可以更好地利用GPU的本地缓存，因为它避免了跨越对角线的片段块切换，从而提高了缓存命中率；

下面来看看单个三角形是怎么做到覆盖整个屏幕的。

在光栅化渲染中，顶点是经过一系列的变换最终映射到屏幕上的，首先是 M-V-P 变换，将顶点从模型空间转换到裁剪空间，而根据 P 的不同（正交或透视），透视会多一个透视除法，将顶点的齐次坐标的 `w` 分量变为 `1`，最终变换到 NDC 空间（ `x`，`y` 和 `z` 都在 `[-1, 1]` 的一个空间）。最后 `x` 和 `y` 经过视口变换，映射到屏幕上。

现在就知道可以用一个三角形覆盖整个 NDC 空间就可以覆盖整个屏幕，看下面的图：

![03_triangle_covers_clipspace](/assets/images/2024/2024-08-02-OpenGLNotes/03_triangle_covers_clipspace.png)

只考虑 `x` 和 `y` 2个方向（最终屏幕的宽和高就是这 2 个方向），黄色的区域就是 NDC 空间，左下角是 `-1`，和 NDC 空间的左下角对应，这也就是三角形的第 1 个顶点 `(-1.0, -1.0, 0.0)`，以下说的顶点都是顺时针情况下考虑的其它顶点，而另外 2 个顶点分别是 `(3.0, -1.0, 0.0)` 和 `(-1.0, 3.0, 0.0)` ，这样整个 NDC 空间的 4 个顶点（(-1.0, -1.0, 0.0)，(1.0, -1.0, 0.0)，(1.0, 1.0, 0.0)，(-1.0, 1.0, 0.0)）都包含在三角形内了。

而 `UV` 坐标呢？为了使黄色 NDC 空间的 4 个顶点的 `UV` 坐标都在三角形内（(0.0, 0.0)，(1.0, 0.0)，(0.0, 1.0)，(1.0, 1.0)）覆盖 `0-1`，三角形三个顶点对应的 `UV` 坐标就使用 `(0.0, 0.0)`，`(2.0, 0.0)` 和 `(0.0, 2.0)`。

最后是在顶点着色器中的实现：

```glsl
#version 410 core

out vec2 UV0;

void main()
{
    UV0 = vec2(
        gl_VertexID == 1 ? 2.0 : 0.0,
        gl_VertexID == 2 ? 2.0 : 0.0
    );
    gl_Position = vec4(
        gl_VertexID == 1 ? 3.0 : -1.0,
        gl_VertexID == 2 ? 3.0 : -1.0,
        0.0,
        1.0
    );
}
```

根据 `gl_VertexID` 输出具体的裁剪空间的坐标和 `UV` 坐标就可以了，而且 `gl_Position` 的 `w` 分量为 1，透视除法也不会影响 `x` 和 `y` 的坐标。

而在渲染时，也不需要设置顶点数据，只需要创建一个临时的 `VAO` 就可以了：

```cpp
// 切换回默认 framebuffer
glBindFramebuffer(GL_FRAMEBUFFER, 0);
// 设置屏幕尺寸
glViewport(0, 0, m_RenderSize.x, m_RenderSize.y);
// 清理缓存
glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT | GL_STENCIL_BUFFER_BIT);

// 设置材质
m_BlitMat->addOrSetTexture(source);
m_BlitMat->use();

// 创建临时的 VAO，并渲染三角形的 3 个顶点
GLuint VAO;
glGenVertexArrays(1, &VAO);
glBindVertexArray(VAO);
glDrawArrays(GL_TRIANGLES, 0, 3);
glBindVertexArray(0);
```

---

## 天空盒（Skybox）渲染中的一些技巧分析

天空盒通常被用来渲染场景的背景，通过渲染一个 `[-1, 1]` 的单位立方体，然后使用它的模型空间坐标去采样一个立方体贴图（Cubemap），最终渲染出来。

先来看渲染天空盒的顶点着色器：

```glsl
#version 410 core

layout (location = 0) in vec3 vPosition;

#include "common/Uniforms.glsl"

out vec3 WorldPos;

void main()
{
    WorldPos = vPosition;

    vec4 clipPos = projectionMatrix * mat4(mat3(viewMatrix)) * vec4(vPosition, 1.0);

    gl_Position = clipPos.xyww;
}
```

首先，传输到片元着色器的 `WorldPos` 就是这个单位立方体模型空间的坐标。再来看看最终输出的裁剪空间坐标：

```glsl
vec4 clipPos = projectionMatrix * mat4(mat3(viewMatrix)) * vec4(vPosition, 1.0);

gl_Position = clipPos.xyww;
```

没有模型变换矩阵，意味着是天空盒是一个中心在世界原点 `(0.0, 0.0, 0.0)` 的 `[-1, 1]` 的单位立方体，然后乘以矩阵 `mat4(mat3(viewMatrix))`，先通过 `mat3(viewMatrix)` 只保留 4x4 的观察矩阵的左上角 3x3 的子矩阵，剔除了观察矩阵平移的部分，最后再将 3x3 的矩阵转换回 4x4 的矩阵用以视图变换。这样操作后，相机的旋转会影响天空盒，但是相机的位置平移不会对天空盒造成影响。最后再乘以透视矩阵将顶点坐标转换到裁剪空间。再对 `gl_Position` 赋值时，将裁剪空间坐标的 `z` 分量改为 `w` 分量，这样做能保证透视除法后，在 NDC 空间下，天空盒的深度永远为 1.0。

为什么要将天空盒的深度固定设置成 1.0 呢？天空盒是当作场景的背景来渲染的，照理说背景应该最先渲染，然后再渲染其它物体。但是这样做就造成了 Over Draw 的问题，屏幕上的像素先渲染了天空盒，然后又渲染不透明物体覆盖掉天空盒，造成了性能浪费。而将天空盒的深度固定设置成 1.0，然后来利用深度测试，先渲染不透明物体，然后再渲染天空盒，可以避免 Over Draw 的问题。

在 OpenGL 中，深度缓冲区用于存储每个像素的深度值，这些深度值通常在 0.0 到 1.0 之间，表示从近到远的距离。使用函数 `glClear(GL_DEPTH_BUFFER_BIT)` 可以在每帧渲染前清除深度缓冲区，深度缓冲区会使用 `glClearDepth` 指定的值来设置其所有值，这个值默认就是 1.0，也就是最远的距离。

在渲染天空盒的顶点着色器中，将输出的天空盒深度控制在 1.0，也就是最大的深度。然后在渲染的过程中，首先渲染不透明物体，此时深度测试开启，深度测试函数为 `GL_LESS`，也就是说只有小于深度缓冲中深度值（此时默认为最大，1.0）的片段才会通过深度测试并渲染。渲染完所有不透明物体后，再渲染天空盒，此时将深度测试函数改为 `GL_LEQUAL`，这样就能保证天空盒只在所有物体后面渲染，而且因为天空盒的深度是 1.0，屏幕上被不透明物体覆盖的像素的深度此时都小于 1.0，渲染天空盒时这些像素的深度测试都不会通过（因为 1.0 大于这些像素的深度值），这样天空盒就只会在不透明物体没有覆盖的像素上渲染了，这样也就避免了先渲染天空盒再渲染不透明物体所造成的 Over Draw 的问题了。

还有一点需要注意的是，采样天空盒后，需不需要做 `sRGB to Linear` 的操作？**只有天空盒的立方体贴图是从 sRGB 图像（比如 JPEG 或 PNG 格式）加载的，才需要对 立方体贴图采样的结果做 `sRGB to Linear` 的转换**。

### Radiance HDR 渲染到立方体纹理

Radiance HDR 图像格式（通常文件扩展名为 `.hdr` 或 `.pic`），也被称为 RGBE 图像格式，是一种用于存储高动态范围（HDR）图像的文件格式。最初由 Greg Ward 为 Radiance 光照模拟系统开发。

RGBE 图像相较于普通的 8 位的 SDR（Standard Dynamic Range）图像（如 JPEG 或 PNG）：

1. 色深：
   - RGBE：每个像素使用 32 位数据表示，其中 R、G、B 三个颜色通道各用 8 位表示，E 通道用 8 位表示指数；
   - SDR：每个像素使用 24 位数据表示，其中 R、G、B 三个颜色通道各用 8 位；PNG 支持多种色深，常见的有8位和16位。在 PNG 使用 8 位色深的情况下，如果还带有 Alpha 通道，则每个像素使用 32 位数据表示，除了 R、G、B 三个颜色通道各用 8 位表示外，Alpha 通道使用额外 8 位表示；
2. 动态范围：
   - RGBE：RGBE 格式通过指数部分扩展了动态范围，可以表示从非常暗到非常亮的细节，这使得它特别适合于高动态范围（HDR）的场景。假设存储的值为 `(R, G, B, E)`，实际的颜色可以表示为： $\text{实际的颜色}=(R/256 \times 2^{E - 128}, G/256 \times 2^{E - 128}, B/256 \times 2^{E - 128})$ ；
   - SDR：只能表示有限的动态范围，通常是 0 到 255 的颜色值。16 位 PNG 可以表示更大的色彩范围；

在图形学中，可以利用 Radiance HDR 全景图像来创建环境贴图 Cubemap。Radiance HDR 全景图像结合了 HDR 和全景技术，通常以球形或柱状投影的形式存储。等距矩形投影图（Equirectangular Map）是最常见的一种表示方法：它将球面上的点映射到一个矩形平面上，通常用于 360° 全景图像和环境贴图，下图展示了一个等距矩形投影图：

![04_equirectangular_map](/assets/images/2024/2024-08-02-OpenGLNotes/04_equirectangular_map.jpeg)

等距矩形投影图是一种将球面坐标（经度和纬度）映射到二维矩形图像的投影方式。它通常是一个宽高比为 2:1 的矩形图像。水平轴（X 轴）表示经度，范围通常是 0° 到 360° ，或者 -180° 到 180° ，垂直轴（Y 轴）表示纬度，范围通常是 -90° 到 90° 。通过等距矩形投影图渲染到立方体纹理就是在这里要讨论的技术。

首先，创建一个立方体纹理，然后渲染一个单位立方体，然后处于立方体内部的中心，分别面向立方体的每个面渲染一次，通过立方体的模型空间坐标，将空间坐标转换到等距矩形投影图的纹理 UV 坐标，采样等距矩形投影图，输出采样的值，将每次渲染的结果存储在立方体纹理的每个面。通过这 6 个面的渲染，也就将等距矩形投影图渲染到了一个立方体纹理，而这个立方体纹理就可以拿来用作渲染天空盒。下面来看看具体的原理：

首先是顶点着色器，因为和渲染天空盒的原理是一样的，所以可以和渲染天空盒的顶点着色器保持一致，这里就不展示出来了。

然后是渲染流程代码，因为在实际的渲染过程中，只需要最终的立方体纹理，所以等距矩形投影图渲染到立方体纹理的过程并不需要每帧的渲染中都进行一次，只需要在程序启动时渲染一次后，保存最终的立方体纹理以提供给后续渲染中使用就可以了。具体的渲染流程代码：

```cpp
// 加载等距矩形投影图
Texture2D::Ptr environmentMap = AssetsLoader::loadHDRTexture("uEnvMap", cubemapPath);

// 创建新的立方体纹理
TextureCube::Ptr cubemap = TextureCube::New("uCubemap");
cubemap->defaultInit(512, 512, GL_RGB32F, GL_RGB, GL_FLOAT);

glm::u32vec2 size = cubemap->getSize();

// 创建帧缓冲对象记录结果
GLuint frameBufferID;
glGenFramebuffers(1, &frameBufferID);

// 绑定帧缓冲对象
glBindFramebuffer(GL_FRAMEBUFFER, frameBufferID);

// 创建渲染缓冲对象用以存储深度信息
GLuint cubemapDepthRenderBufferID;
glGenRenderbuffers(1, &cubemapDepthRenderBufferID);

// 绑定渲染缓冲对象
glBindRenderbuffer(GL_RENDERBUFFER, cubemapDepthRenderBufferID);
// 设置渲染缓冲对象的存储格式，创建一个深度缓冲区
glRenderbufferStorage(GL_RENDERBUFFER, GL_DEPTH_COMPONENT24, size.x, size.y);

// 将渲染缓冲对象附加到帧缓冲对象的深度附件
glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT, GL_RENDERBUFFER, cubemapDepthRenderBufferID);

// 检查帧缓冲对象的完整性
if (glCheckFramebufferStatus(GL_FRAMEBUFFER) != GL_FRAMEBUFFER_COMPLETE)
{
   std::cerr << "FrameBuffer is not complete!" << std::endl;
}
// 解绑帧缓冲对象
glBindFramebuffer(GL_FRAMEBUFFER, 0);

// 渲染时的投影矩阵，fov 为 90° 以捕捉整个面
glm::mat4 captureProjection = glm::perspective(glm::radians(90.0f), (float)size.x / size.y, 0.1f, 10.0f);

// 渲染时的观察矩阵，分别是处于立方体中心，面对立方体的 6 个面
glm::mat4 captureViews[] =
{
   glm::lookAt(glm::vec3(0.0f, 0.0f, 0.0f), glm::vec3(1.0f,  0.0f,  0.0f), glm::vec3(0.0f, -1.0f,  0.0f)),
   glm::lookAt(glm::vec3(0.0f, 0.0f, 0.0f), glm::vec3(-1.0f,  0.0f,  0.0f), glm::vec3(0.0f, -1.0f,  0.0f)),
   glm::lookAt(glm::vec3(0.0f, 0.0f, 0.0f), glm::vec3(0.0f,  1.0f,  0.0f), glm::vec3(0.0f,  0.0f,  1.0f)),
   glm::lookAt(glm::vec3(0.0f, 0.0f, 0.0f), glm::vec3(0.0f, -1.0f,  0.0f), glm::vec3(0.0f,  0.0f, -1.0f)),
   glm::lookAt(glm::vec3(0.0f, 0.0f, 0.0f), glm::vec3(0.0f,  0.0f,  1.0f), glm::vec3(0.0f, -1.0f,  0.0f)),
   glm::lookAt(glm::vec3(0.0f, 0.0f, 0.0f), glm::vec3(0.0f,  0.0f, -1.0f), glm::vec3(0.0f, -1.0f,  0.0f))
};

// 将视口大小设置为每个面的大小
glViewport(0, 0, size.x, size.y);

// 开始渲染
// 绑定帧缓冲对象
glBindFramebuffer(GL_FRAMEBUFFER, frameBufferID);

// 创建渲染材质
Material::Ptr capMat = Material::New("HDR_to_Cubemap", "glsl_shaders/Skybox.vs", "glsl_shaders/HDRToCubemap.fs");
// 设置渲染时使用的等距矩形投影图纹理
capMat->addOrSetTexture("uEnvMap", envMap);
m_Skybox->setOverrideMaterial(capMat);

// 和渲染天空盒一样，顶点着色器中输出的裁剪空间坐标为 gl_Postion = clipPos.xyww，也就是说输出的深度为最大的 1.0，为了能通过深度测试，所以将深度测试函数设置为 less&equal
glDepthFunc(GL_LEQUAL);

// 因为是在立方体内部中心向立方体的 6 个面做渲染，所以要关闭面剔除
glDisable(GL_CULL_FACE);

// 绑定统一缓冲区
glBindBuffer(GL_UNIFORM_BUFFER, m_GlobalUniformBufferID);
// 设置投影矩阵数据
glBufferSubData(GL_UNIFORM_BUFFER, 64, 64, &(captureProjection[0].x));

// 6 个面的渲染
for (unsigned int i = 0; i < 6; ++i)
{
   // 每次渲染，设置当前面的观察矩阵
   glBufferSubData(GL_UNIFORM_BUFFER, 0, 64, &(captureViews[i][0].x));

   // 将立方体纹理的每一个面设置为帧缓对象的颜色附件
   glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_CUBE_MAP_POSITIVE_X + i, cubemap->getTextureID(), 0);

   // 检查帧缓冲对象的完整性
   if (glCheckFramebufferStatus(GL_FRAMEBUFFER) != GL_FRAMEBUFFER_COMPLETE)
   {
      std::cerr << "FrameBuffer is not complete!" << std::endl;
   }

   // 每次渲染前清除颜色缓冲和深度缓冲
   glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);

   // 渲染立方体
   drawNode(m_Skybox);
}

// 解绑帧缓冲对象
glBindFramebuffer(GL_FRAMEBUFFER, 0);
// 解绑统一缓冲区
glBindBuffer(GL_UNIFORM_BUFFER, 0);

// 将深度测试函数和面剔除恢复初始值
glDepthFunc(GL_LESS);
glEnable(GL_CULL_FACE);

// 将视口大小恢复为初始值
glViewport(0, 0, m_RenderSize.x, m_RenderSize.y);
```

最后来看看片元着色器中的代码：

```glsl
#version 410 core
out vec4 FragColor;

uniform sampler2D uEnvMap;

in vec3 WorldPos;

const vec2 invAtan = vec2(0.1591,0.3183);
vec2 SampleSphericalMap(vec3 v)
{
    vec2 uv = vec2(atan(v.z, v.x), asin(v.y));
    uv *= invAtan;
    uv += 0.5;
    return uv;
}

void main()
{
    vec2 uv = SampleSphericalMap(normalize(WorldPos));
    vec3 color = texture(uEnvMap, uv).rgb;
    FragColor = vec4(vec3(color), 1.0);
}
```

其中函数 `SampleSphericalMap` 做的就是将单位立方体 6 个面上的坐标转换到球面坐标，然后将其映射到等距矩形投影图上的 UV 坐标。原理：

球面坐标系中的每个点可以用 2 个角度来表示，方位角（Azimuthal Angle，通常记作： $\phi$ ）和极角（Polar Angle，通常记作： $\theta$ ），如下图：

![05_spherical_coordinates](/assets/images/2024/2024-08-02-OpenGLNotes/05_spherical_coordinates.png)

其中， $\theta$ 范围通常是 $[0, \pi]$ ， $\phi$ 范围通常是 $[0, 2\pi]$ 。

有一个三维向量，可以通过如下公式计算出其对应的球面坐标：

$$
\phi = \arctan(v.z, v.x); \\
\theta = \arcsin(v.y);
$$

而为了将球面坐标映射到 UV 坐标，则需要将角度转换到 [0, 1] 的范围：

$$
u = \frac{\phi}{2\pi} + 0.5; \\
v = \frac{\theta}{\pi} + 0.5;
$$

知道了这些就可以很好的理解片元着色器中，函数 `SampleSphericalMap` 的代码了，首先将立方体面上的坐标转换到球面坐标：

```glsl
const vec2 invAtan = vec2(0.1591, 0.3183);
vec2 SampleSphericalMap(vec3 v)
{
    // 将单位立方体面上的坐标转换到单位球面坐标(1, ϕ, θ)
    vec2 uv = vec2(atan(v.z, v.x), asin(v.y));

    // 将球面坐标映射到 uv 坐标，其中 invAtan.x 就是 1 / (2π) = 0.1591，而 invAtan.y 就是 1 / π = 0.3183
    uv *= invAtan;
    // 将 uv 坐标从 [-0.5, 0.5] 转换到 [0, 1]
    uv += 0.5;
    return uv;
}
```

最后就是采样等距矩形投影图纹理并输出结果了。

---
