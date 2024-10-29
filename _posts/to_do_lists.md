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

<!-- TODO
### RenderTexture.memorylessMode
<!-- https://discussions.unity.com/t/rendertexture-memorylessmode/661299 -->

<!-- #### **球谐函数（Spherical Harmonics）** -->

<!-- TODO
### Variance Shadow Maps（VSMs） -->
