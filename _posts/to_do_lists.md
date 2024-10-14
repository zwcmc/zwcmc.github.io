<!-- ## 计算法线变换矩阵的另一种方法 -->

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

<!-- 在之前的笔记中，有说到[法线变换矩阵](#normal-matrix法线变换矩阵)，经过推导使用的是 `transpose(inverse(M))` 矩阵。

- https://github.com/graphitemaster/normals_revisited
- http://www.reedbeta.com/blog/normals-inverse-transpose-part-1/
- https://www.shadertoy.com/view/3s33zj
- FGED Volume 1, section 1.7.5 "Inverses of Small Matrices"
- FGED Volume 1, section 3.2.2 "Transforming Normal Vectors" -->

<!-- ## SSR -->

<!-- TODO -->

<!-- ## TAA -->

<!-- TODO -->

<!-- --- -->

<!-- ### VisibleLight.localToWorldMatrix 矩阵的第3列和第4列的意义 -->

<!-- TODO -->
<!-- // For directional light, lightPos is a direction, and in light's local space, it's forward direction in local space is (0,0,1),
// after is multiplied by light's localToWorld matrix:
// localToWorldMatrix * (0,0,1,0), a direction has 0 in homogeneous coordinate;
// it returns the column 2 of the light's localToWorld matrix, and in lighting calculation in Shader,
// the light directional vector needs to point to light, so negative the direction here

// For point light and spot light, lightPos is a position in world space, it's original position in local space is (0,0,0),
// after is multiplied by light's localToWorld matrix:
// localToWorldMatrix * (0,0,0,1), a position has 1 in homogeneous coordinate;
// it returns the column 3 of the light's localToWorld matrix

// For spot light's direction, and in light's local space, it's forward direction in local space is (0,0,1),
// after is multiplied by light's localToWorld matrix:
// localToWorldMatrix * (0,0,1,0), a direction has 0 in homogeneous coordinate;
// it returns the column 2 of the light's localToWorld matrix, and in lighting calculation in Shader,
// the spot directional vector needs to point to light, so negative the direction here -->

<!-- ---

### RenderTexture.memorylessMode

<!-- TODO -->
<!-- https://discussions.unity.com/t/rendertexture-memorylessmode/661299 -->

<!-- **TODO：在这里只是列出了实现的代码，但是具体 GGX 采样点生成的推导过程还需要详细的学习分析。下图展示了主要思路和一些文章参考** ：

![39_GGX_ImportanceSampling](/assets/images/2024/2024-08-12-PhysicallyBasedRendering/39_GGX_ImportanceSampling.png)

- https://blog.tobias-franke.eu/2014/03/30/notes_on_importance_sampling.html
- https://blog.selfshadow.com/publications/s2013-shading-course/#course_content
- https://placeholderart.wordpress.com/2015/07/28/implementation-notes-runtime-environment-map-filtering-for-image-based-lighting/ -->

<!-- #### **球谐函数（Spherical Harmonics）** -->

<!-- TODO
### Variance Shadow Maps（VSMs） -->

<!-- TODO
### Percentage-Closer Soft Shadows（PCSS）
- https://developer.download.nvidia.cn/whitepapers/2008/PCSS_Integration.pdf
- https://developer.download.nvidia.cn/shaderlibrary/docs/shadow_PCSS.pdf
- https://www.legendsmb.com/2022/10/13/unity-PCF-sourcecode/
- https://zhuanlan.zhihu.com/p/369761748 -->

## IMR & TBR & TBDR

- https://developer.samsung.com/galaxy-gamedev/resources/articles/gpu-framebuffer.html
- https://docs.qq.com/slide/DUWFsekNSZllIVHpT
- https://mp.weixin.qq.com/s/-ueKhxbsJOnUtV1SC5eyBQ
- https://gitlab.freedesktop.org/mesa/mesa