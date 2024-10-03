---
layout: post
title:  "基于物理的渲染（PBR）理论"
date:   2024-08-12 16:16:00 +0800
category: Rendering
---

## PBR 核心理论总结

基于物理的渲染（Physically Based Rendering，PBR）是指使用基于物理原理和微平面理论建模的着色/光照模型，以及使用从真实世界中测量的表面参数来准确表示真实世界材质的渲染理念。

PBR 基础理念主要包含以下几点：

- **微平面理论（Microfacet Theory）**：微平面理论是将物体表面建模成无数微观尺度上有随机朝向的理想镜面反射的小平面（Microfacet）的理论。
- **能量守恒（Energy Conservation）**：出射光线的能量永远不能大于入射光线的能量。随着粗糙度的上升，镜面反射区域的面积会增加，作为平衡，镜面反射区域的平局亮度则会下降。
- **基于 F0 建模的菲涅尔反射（Fresnel Reflection）**：菲涅尔效应表示观察到的反射光线的量与视角相关的现象，且掠射角度（90度）下反射率最大。
- **线性空间光照（Linear Spcae Lighting）**：线性空间渲染为光照计算提供了正确的数学运算。在线性空间中，能够还原现实世界的光与物质交互的方式，所以颜色值的计算和颜色操作必须在线性空间中执行。在渲染中，有些贴图是在伽马空间创建编辑的（例如漫反射贴图，自发光贴图），这些贴图需要先转换到线性空间再参与着色计算，而描述物体表面属性的贴图如粗糙度、金属度、高光贴图等必须保证是线性空间。所以基于物理的渲染往往会涉及到线性空间和伽马空间之间的相互转换。
- **色调映射（Tone Mapping）**：也称色调复制（Tone Reproduction）。在图形学中，色调映射表示将高动态范围（HDR）的光照结果转换到低动态范围（LDR）。因为通过 HDR 渲染出来的亮度值会超过传统显示设备（如显示器、电视）能够显示的最大亮度，所以需要结合色调映射将光照结果能正常显示在传统显示设备上。
- **基于真实世界测量的材质参数（Real-World Measurement Based Substance Properties）**：PBR 的正统材质参数往往都基于真实世界测量。真实世界中的物质可以分为三大类：绝缘体（Insulator）、半导体（Semiconductor）和导体（Conductor）。在渲染和游戏领域，一般只对其中的两个感兴趣：导体（金属，Metal）和电介质（Dielectric，绝缘体/非金属）。菲涅尔反射率代表材质的镜面反射与强度，是真实世界材质的核心测量数值。其中电介质具有非彩色的镜面反射颜色，而金属具有彩色的镜面反射颜色，即电介质的 `F0` 是一个 `float`，而金属的 `F0` 是一个 `float3`。

## 一些基本的知识准备

### 人眼视觉

在现实生活中，人眼看到某一物体的颜色其实并不是这个物体本身的颜色，而是其反射和散射的颜色。也就是，**那些不能被物体吸收（Absorption）的颜色，也就是被反射或散射到人眼中的可见光波长代表的颜色，就是人眼能够感知到的物体的颜色**。

### 相速度与折射率

波动光学（Wave Optics）又称物理光学（Physical Optics）。在波动光学中，光被建模为一种电磁横波（Transverse Wave），即是电场和磁场垂直于其传播方向振荡的波。下图展示了一种最简单的光波，电场和磁场矢量以 90° 相互振荡并同时向传播方向振荡，它既是单色的（具有单一波长 $\lambda$ ）又是线性极化的（电场和磁场各自沿单向振荡）：

![00_transverse_wave](/assets/images/2024/2024-08-12-PhysicallyBasedRendering/00_transverse_wave.png)

光波的磁场和电场的振荡相互耦合，彼此相互生成。磁场矢量和电场矢量相互垂直，且两者都垂直于波的传播方向。

**相速度（Phase Velocity）** 描述了波的某一特定相位点（如波峰或波谷）在介质中的传播速度。相速度 $v_p$ 通常由波的频率（ $\text{f}$ ）和波长（ $\lambda$ ）决定，并通过以下公式表示：

$$
v_p = \frac{\lambda}{\text{T}} = \lambda \text{f}
$$

其中：

- $\lambda$ 是波长，即相邻两个相同相位点（如两个波峰）之间的距离。
- $\text{T}$ 是周期，即完成一个完整波长所需的时间。
- $\text{f}$ 是频率，即每秒钟完成的波动周期数。

相速度表示的是波的特定相位点在介质中传播的速度。它与介质的性质有关，例如光在不同的介质中的传播速度会不同。

**折射率（Index of Refraction）** 描述了光在介质中的传播速度相对于在真空中的传播速度，通常简写为 `IOR`，由字母 $\text{n}$ 表示。折射率定义为：

$$
\text{n} = \frac{\text{c}}{v_p}
$$

其中：

- $\text{c}$ 是光在真空中的速度，约为 $3 \times 10^8$ m/s。
- $v_p$ 是光在介质中的相速度。

### 复折射率

复折射率（Complex Index of Refraction），是一个用于描述介质光学性质的参数。它不仅考虑了光在介质中的传播速度（通过折射率），还考虑了介质对光的吸收性。复折射率通常表示为一个复数的形式： $\tilde{n} = n + ik$ ，其分为实部和虚部两部分：

- 实部：折射率 $\text{n}$ 描述了光在介质中的传播速度相对于光在真空中传播的速度，即相对于光在真空中传播速度减慢的度量。
- 虚部：消光系数 $\text{k}$ 描述了介质对光的吸收强度。光被吸收后，通常会转换成热能。非吸收性介质通常虚部很小，可以忽略不计。

特定材质对光的选择性吸收是因为所选择的 **光波频率与该材质原子中的电子振动的频率相匹配**。由于不同的原子和分子具有不同的固有振动频率，其将选择性地吸收不同频率的可见光。光的吸收对视觉效果有直接影响，因为会降低光的强度，并且如果吸收对某些可见波长具有选择性，也会改变光的颜色。如下图，光的吸收导致透明介质的不同颜色外观，从左到右分别是：清水，石榴糖浆，茶和咖啡：

![01_light_absorb](/assets/images/2024/2024-08-12-PhysicallyBasedRendering/01_light_absorb.jpg)

### 折射的发生条件

折射的发生条件主要包括：

1. **介质差异**：光波必须从一种介质进入另一种介质，这两种介质的折射率必须不同。折射率不同意味着光波在这两种介质中的相速度不同。
2. **界面存在**：必须有一个明确的介质界面，即两种介质之间的分界面。
3. **入射角非零**：入射光波与界面法线之间的夹角（入射角）不能为零。即光波不能垂直入射到界面上，如果光波垂直入射（入射角为零），光波的方向不会改变，只是相速度发生变化。

下图展示了入射光线在两种介质分界面发生的变化：

![02_reflection_refraction](/assets/images/2024/2024-08-12-PhysicallyBasedRendering/02_reflection_refraction.png)

**另外，折射率缓慢的逐渐变化不会导致光线的分离，而是导致其传播路径的弯曲**。当空气密度因温度而变化时，通常可以看到这种效果，例如海市蜃楼（Mirages）和热形变（Heat Distortion）。见下图：

![03_heat_distortion](/assets/images/2024/2024-08-12-PhysicallyBasedRendering/03_heat_distortion.jpg)

### 物质对光的吸收和散射的特性决定其外观

光与物质之间的两种相互作用为散射（Scattering）和吸收（Absorption）。其中：

- **散射决定介质的浑浊程度**。大多数情况下，固体和液体介质中的颗粒都比光的波长更大，并且倾向于均匀的散射所有可见波长的光。高散射会产生不透明的外观。
- **吸收决定材质的外观颜色**。几乎任何材质的外观颜色都是由其吸收的波长相关性引起的。

每种介质的外观是散射和吸收两种现象的综合结果：

![04_scattering_absorption](/assets/images/2024/2024-08-12-PhysicallyBasedRendering/04_scattering_absorption.jpeg)

另外， 散射和吸收都与观察尺度有关。在小场景中不产生任何明显散射的介质在较大尺度上可能具有非常明显的散射。例如，当观察房间中的一杯水时，在水对光的散射和对光的吸收都是不可见的，但是如果在海底，就可以很明显的看到水对光的吸收，想象一下深海是一片漆黑的。

**当然，除了折射的光线，材质的外观还与反射有关，所以，可以理解为材质的最终外观由镜面反射以及物质对折射光线的吸收和散射的特性组合综合决定**。

### 辐射度量学（Basic Radiometry）

#### **辐射能量（Radiant Energy）**

光源辐射出来的能量，叫做辐射能量，以 **焦耳（Joule）** 为单位进行测量。并用符号 $Q_e$ 来表示。

#### **辐射通量（Radiant Flux）**

单位时间的辐射能量叫做辐射通量，单位是瓦特，或者流明。 $\phi_e=\frac{dQ_e}{dt}$

#### **立体角（Solid Angle）**

先看一个二维的角，对于圆上的一个角，它的定义是这个角对应的圆上的弧长和圆半径的比值：

![21_angle](/assets/images/2024/2024-08-12-PhysicallyBasedRendering/21_angle.jpeg)

角 $\theta$ 的弧度就是： $\theta=\frac{l}{r}$ 。一个圆对应的弧度就是 $2\pi$ 。

而立体角就是二维角度在三维空间中的一个延申：**三维空间中一个球，从球心出发，有一个一定大小的锥，锥打到球面上截取的球面上一个区域的面积，这个区域的面积和球半径的平方的比值就叫做立体角**：

![22_solid_angle](/assets/images/2024/2024-08-12-PhysicallyBasedRendering/22_solid_angle.jpeg)

一个球对应的立体角是 $4\pi$ 。

#### **单位立体角（Unit Solid Angle）**

单位立体角则指的是一个球面上截取面积与球半径平方相等的部分所对应的立体角。

#### **辐射强度（Radiant Intensity）、辐照度（Irradiance）和辐射率（Radiance）**

![20_radiantintensity_irradiance_radiance](/assets/images/2024/2024-08-12-PhysicallyBasedRendering/20_radiantintensity_irradiance_radiance.jpeg)

- 辐射强度（Radiant Intensity）：光源在每单位立体角所辐射出去的辐射通量，单位是 `W/sr`。
- 辐照度（Irradiance）：在物体表面任何一个点（每单位面积上），所接收到的辐射通量。
- 辐射亮度（Radiance）：光源在每单位投影面积、每单位立体角所辐射出去的辐射通量。它是一个方向性量，描述了某个方向上的辐射强度。单位是 `W/(m²·sr)`。

## 微平面理论

大多数真实世界的表面不是光学上光滑的，具有比光波长大的多但比像素小的尺度的不规则性。这种微观几何变化导致每个表面点反射（和折射）不同方向的光，材质的部分外观是这些反射和折射方向的聚合结果。

![11_microscopic_view](/assets/images/2024/2024-08-12-PhysicallyBasedRendering/11_microscopic_view.jpeg)

微平面理论是将物体表面建模成无数微观尺度上有随机朝向的理想镜面反射的微平面（Microfacet）。一个实际表面在微观尺度上并不是完全平滑的，而是由许多微小的平面组成。光在与其交互时，表面上的每个微平面都会以略微不同的方向对入射光反射，最终的表面外观是这些微平面与光线交互的结果的聚合。下图展示了基于微平面理论的物体表面的可见光反射是来自具有不同方向的许多微平面的反射的总体结果：

![05_microfacet](/assets/images/2024/2024-08-12-PhysicallyBasedRendering/05_microfacet.png)

在微观尺度上，表面越粗糙，反射越模糊，这是因为这些微平面的取向各异，与整个宏观表面取向的偏离更大：

![06_roughntess_microfacet](/assets/images/2024/2024-08-12-PhysicallyBasedRendering/06_roughntess_microfacet.jpeg)

出于着色的目的，通常会用统计方法处理这种微观几何现象，将表面的反射视为每个点在多个方向上反射（和折射）光。下图展示了在宏观上，表面可以被视为在多个方向上反射（和折射）光：

![07_reflect_refract_dirs](/assets/images/2024/2024-08-12-PhysicallyBasedRendering/07_reflect_refract_dirs.png)

而对于折射的光，不同的介质与折射光有着不同的交互：

- 对于金属（Metal），折射光会立刻被吸收（能量被自由电子立即吸收）。
- 对于电介质（Dielectric，也称为非金属或绝缘体），一旦光在其内部折射，会表现出吸收和散射两种行为。

在金属中，所有的折射光立即被自由电子吸收：

![08_metallic_absorption](/assets/images/2024/2024-08-12-PhysicallyBasedRendering/08_metallic_absorption.png)

而在电介质中，折射光会进行散射，被吸收部分光之后，剩下的光从表面重新射出：

![09_dielectrics_scattering_absorption](/assets/images/2024/2024-08-12-PhysicallyBasedRendering/09_dielectrics_scattering_absorption.png)

另外，漫反射和次表面反射其实是相同的物理现象，本质都是折射光的次表面散射的结果。如果散射距离远小于像素大小，则次表面散射便可以近似为漫反射。也就是说，光的折射现象，建模为漫反射还是次表面散射，取决于观察的尺度。看下面这个图：

![10_subsurface_scattering_or_diffuse](/assets/images/2024/2024-08-12-PhysicallyBasedRendering/10_subsurface_scattering_or_diffuse.jpeg)

如果像素大小（绿色部分）远大于光线离开表面之前所经过的距离（左上角的示意图），在这种情况下，次表面散射可以当作漫反射来处理；如果像素大小小于散射距离，则不能忽略这些距离的存在，需当作次表面散射现象进行处理。

### 光与介质边界的交互类型总结

下面是一张经典的光与表面的交互示意图：

![12_interact_with_light](/assets/images/2024/2024-08-12-PhysicallyBasedRendering/12_interact_with_light.png)

当一束光线入射到物体表面时，由于物体表面与空气两种介质之间折射率的变化，光线会发生反射和折射：

- **反射（Reflection）**：**光线在两种介质交界处的直接反射即镜面反射（Specular）**。金属的镜面反射颜色为三通道的彩色，而电介质的镜面反射颜色为单通道的单色。
- **折射（Refraction）**：从表面折射进入介质的光，会发生吸收（Absorption）和散射（Scattering），而介质的整体外观由其散射和吸收特性的组合决定。其中：
  - **散射（Scattering）**：折射率的变化引起散射，光的方向会改变（分裂成多个方向），但是光的总量或光谱分布不会改变。散射最终被视作的类型（漫反射或次表面散射）与观察尺度有关：
    - **次表面散射（Subsurface Scattering）**：观察像素小于散射距离，散射被视作次表面散射。
    - **漫反射（Diffuse）**：观察像素大于散射距离，散射被视作蛮烦谁。
    - **透射（Transmission）**：入射光经过折射穿过物体后的出射现象。透射为次表面散射的特例。
  - **吸收（Absorption）**：具有复折射率的物质区域会引起吸收，具体原理是光波频率与该材质原子中的电子振动的频率相匹配。复折射率（complex number）的虚部（imaginary part）确定了光在传播时是否被吸收（转换成其他形式的能量）。发生吸收的介质的光量会随传播的距离而减小（如果吸收优先发生于某些波长，则可能也会改变光的颜色），而光的方向不会因为吸收而改变。任何颜色色调通常都是由吸收的波长相关性引起的。

### 不同物质与光的交互总结

金属和电介质与光的交互总结如下：

- **金属（Metal）**：**金属的外观主要取决于光线在两种介质交界处的直接反射（即镜面反射，Specular）**。金属的镜面反射颜色为三通道的彩色，R、G、B 各不相同。而折射入金属内部的光线几乎立即全部被自由电子吸收，且折射入金属的光线不存在散射。
- **电介质（Dielectrics，非金属或绝缘体）**：整体外观主要由其吸收和散射的特性组合决定。同样，电介质与光的交互分为反射和折射两部分。而折射按介质类型的散射和吸收特性，分为多类：
  - **反射（Reflection）**：电介质的镜面反射颜色为单通道单色，即 R=G=B。
  - **折射（Refraction）**：光从表面折射入电介质介质，则介质的整体外观由其散射和吸收的特性组合决定。不同的介质类型的散射和吸收特性不一：
    - **均匀介质（Homogeneous Media）**：主要为透明介质，无折射率变化。不存在散射，光总以直线传播并且不会改变方向。存在吸收，光的强度会通过吸收减少，传播距离越远，吸收量越高。
    - **非均匀介质（Nonhomogeneous Media）**：通常可以建模为具有嵌入散射粒子的均匀介质。具有折射率变化，分为几类：
      - **浑浊介质（Cloudy Media）**：混浊介质具有弱散射，散射方向略微随机化。根据组成的不同，具有复数折射率的物质区域引起吸收。
      - **半透明介质（Translucent Media）**：半透明介质具有强散射，散射方向完全随机化。根据组成的不同，具有复数折射率的物质区域引起吸收。
      - **不透明介质（Opaque Media）**：不透明介质和半透明介质一致。具有强散射，散射方向完全随机化。根据组成的不同，具有复数折射率的物质区域引起吸收。

## 菲涅尔反射（Fresnel Reflectance）

### 菲涅尔效应（Fresnel Effect）

菲涅尔效应是光学中的一个现象，描述了光在不同介质的界面上发生反射和折射时，其反射率随入射角的变化规律。这个效应由法国物理学家奥古斯丁·菲涅尔（Augustin-Jean Fresnel）的名字命名。菲涅尔效应指出，反射率（即反射光的强度相对于入射光的强度）会随着入射角的增大而增加，通过这个反射率和能量守恒原则，又可以直接获得折射率（光总的能量为 1，菲涅尔效应描述了反射的部分，剩下的就是折射的部分了）。如下图：

![13_fresnel_effect](/assets/images/2024/2024-08-12-PhysicallyBasedRendering/13_fresnel_effect.jpeg)

简单来说，菲涅尔效应即描述视线垂直于表面时反射较弱，而当视角非垂直于表面时，与表面法线夹角越大，反射越明显的一种现象。比如说，站在海边低头看脚下的海水，会发现水是透明的，反射不会特别强烈；而如果看远处的海面时，会发现水并不透明，而且反射非常强烈。

**另外需要注意的是，在宏观层面看到的菲涅尔效应实际上是微观层面所有微平面菲涅尔效应的平均值。也就是说，影响菲涅尔效应的关键参数在于每个微平面的法向量和入射光线的角度，而不是宏观平面的法向量和入射光线的角度**。即：

- 当从接近平行于表面的视线方向进行观察，所有光滑表面都会变得 100% 的反射性。
- 对于粗糙表面来说，在接近平行方向的高光反射也会增强，但不够达到 100% 的强度。

不同材质的菲涅尔效应的强弱是不同的，金属的菲涅尔效应一般很弱，主要是因为金属本身的反射率就已经很强，就拿铝来说，其反射率在所有角度下几乎都保持86%以上，随角度变化很小。而电介质的菲涅尔效应就很明显，比如折射率为 1.5 的玻璃，在表面法向量方向的反射率仅为 4%，但当视线与表面法向量夹角很大的时候，反射率可以接近 100%，这一现象也导致了金属与电介质外观上的不同。

### 菲涅尔方程（Fresnel Equation）

菲涅尔方程用以描述光在两种不同介质的界面上反射和折射时，反射光的强度如何随入射角的变化而变化。菲涅尔方程描述的光的反射现象称之为 **菲涅尔反射**。

另外，菲涅尔方程其实和 **麦克斯韦方程组（Maxwell's Equations）** 有很深的渊源。根据物理学，麦克斯韦方程组可以在折射率变化时计算光的行为。对于空气中通常的物体表面而言，物体的表面便是空气折射率和物体折射率的交界面，对此特殊的折射率交界面而言，麦克斯韦方程组的解被我们称为菲涅尔方程，也就是说 **菲涅尔反射的方程可以通过麦克斯韦方程组推导出来**，因为本质上讲菲涅尔反射其实是使用的波动理论来解释光的反射现象。

完整的菲涅尔方程有点复杂，至少可以说艺术家所需的材质参数（在可见光谱上密集采样的复折射率）并不方便。而通过完整的菲涅尔方程，可以拟合出更简化的近似表达式（如 Schlick[1994]的 Fresnel近似），以及抽象出描述物体表面属性的菲涅尔反射率 `F0` 。

![14_fresnel_equation](/assets/images/2024/2024-08-12-PhysicallyBasedRendering/14_fresnel_equation.png)

上图展示了各种物质的菲涅尔反射率（Y 轴）作为入射角（X 轴）的函数。由于铜（copper）和铝（aluminum）在可见光谱上的反射率有明显变化，所以它们的反射率显示为 R、G 和 B 的三条独立曲线，铜的 R 曲线最高，其次是 G，最后是 B（因此它的红色）；铝的 B 曲线最高，其次是 G，最后是 R。上图选择的材质代表了各种各样的材质，虽然每条曲线值都不同，但是它们也有一些共同的地方：**对于每种材质来说， 在入射角为 0° 和 45° 之间时，反射率几乎是恒定的；在入射角为 45° 和 75° 之间时，反射率变化明显（通常但不总是有所增加）；最后，在入射角为 75° 和 90° 之间时，反射率总是迅速变为 1**。由此，可以将 `F0` , 即 0° 角入射时的菲涅尔反射率作为材质的特征反射率，用它来对该材质的反射属性进行建模。

### F0：0° 角入射时的菲涅尔反射率

当光线垂直（以 0° 角）入射当表面时，该光线被反射为镜面反射光的比率被称为 `F0`，即 `F0` 是 0° 角入射时的菲涅尔反射率，而又因为能量守恒原则，所以折射率则为 `1.0 - F0`。

![15_f0_grazing_angle](/assets/images/2024/2024-08-12-PhysicallyBasedRendering/15_f0_grazing_angle.jpeg)

上图展示了对于光滑的电介质表面，在 0° 角入射（`F0`）将反射 2-5% 的光，而在掠射角入射下将反射 100% 的光。

大多数常见电介质的 `F0` 范围为 `0.02-0.05`（线性值），而对于金属，`F0` 范围为 `0.5-1.0`。下图展示了大多数物质的 `F0`：

![16_f0_s](/assets/images/2024/2024-08-12-PhysicallyBasedRendering/16_f0_s.png)

### 折射率与 F0 的关系

通用的折射率与 `F0` 的关系式如下：

$$
\text{F0} = (\frac{n_1 - n_2}{n_1 + n_2})^2
$$

其中， $n_1$ 和 $n_2$ 分别为两种介质的折射率。通常假设 $n_1 = 1$近似于空气的折射率，并用 $n$ 替换 $n_2$ ，所以上式可以简化为：

$$
\text{F0} = (\frac{n - 1}{n + 1})^2
$$

这就是通常看到的 `F0` 和折射率之间的转换公式。这里[pixelandpoly.com](https://pixelandpoly.com/ior.html)可以查询各种物质折射率数值，对于一般仅有实数部，虚数部可以忽略不计的电介质而言，可以通过查询到的物质折射率和上面的公式，计算出其 `F0`。

## 渲染方程与反射方程

### 渲染方程（The Rendering Equation）

渲染方程（The Rendering Equation）作为渲染领域中的重要理论，其描述了光线在场景中的传播和相互作用。渲染方程由詹姆斯·卡吉亚（James Kajiya）在1986年提出。渲染方程在理论上给出了一个完美的结果，而各种各样的渲染技术，都是对这个理想结果的一个近似。

渲染方程的物理基础是能量守恒定律。在一个特定的位置和方向，出射光 $L_o$ 是自发光 $L_e$ 与反射光线之和，反射光线本身是在半球上各个方向的入射光 $L_i$ 之和乘以表面反射率及入射角。某一点 $\text{p}$ 的渲染方程可以表示为：

$$
L_o(\mathbf{p}, \omega_o) = L_e(\mathbf{p}, \omega_o) + \int_{\Omega}^{}L_i(\mathbf{p}, \omega_i)f_r(\mathbf{p}, \omega_i, \omega_o)(n \cdot \omega_i)d\omega_i
$$

其中， $\omega_o$ 是出射方向，也可以理解为观察方向； $\omega_i$ 是光线的入射方向：

- $L_o(\mathbf{p}, \omega_o)$ 是点 $\mathbf{p}$ 的最终出射光亮度。
- $L_e(\mathbf{p}, \omega_o)$ 是点 $\mathbf{p}$ 的自发光的亮度。
- $L_i(\mathbf{p}, \omega_i)$ 是入射光的亮度。
- $f_r(\mathbf{p}, \omega_i, \omega_o)$ 描述了入射光在 $\mathbf{p}$ 点如何反射成出射光。即是 `BxDF`（Bidirectional Scattering Distribution Function，双向散射分布函数），一般为 `BRDF`（Bidirectional Reflectance Distribution Function，双向反射分布函数）。
- $(n \cdot \omega_i)$ 是入射角带来的入射光衰减。
- $\int_{\Omega}^{} ···d\omega_i$ 是入射方向半球的积分（可以理解为无穷小的累加和）。

![17_rendering_equation](/assets/images/2024/2024-08-12-PhysicallyBasedRendering/17_rendering_equation.jpeg)

### 反射方程 BxDF

BxDF 一般而言是对 BRDF、BTDF、BSDF、BSSDF等几种双向分布函数的一个统一表示。它就是渲染方程中的 $f_r(\mathbf{p}, \omega_i, \omega_o)$ 项。

- **BRDF**：双向反射分布函数(Bidirectional Reflectance Distribution Function , BRDF)
- **BSDF**：双向散射分布函数(Bidirectional scattering distribution function, BSDF)
- **BTDF**：双向透射分布函数(Bidirectional Transmittance Distribution Function, BTDF)
- **BSSRDF**：双向散射表面反射分布函数(Bidirectional Scattering-Surface Reflectance Distribution Function, BSSRDF)

其中，BSDF 可以看作是 BRDF 和 BTDF 的超集和广义化概念，而且 BSDF = BRDF + BTDF：

![18_BTDF](/assets/images/2024/2024-08-12-PhysicallyBasedRendering/18_BTDF.jpeg)

而 BSSRDF 和 BRDF 的不同之处在于，BSSRDF 扩展了 BRDF 的概念，它还考虑到了光线在表面下的散射和扩散的现象，例如次表面散射（Subsurface Scattering，SSS）:

![19_BRDF_BSSRDF](/assets/images/2024/2024-08-12-PhysicallyBasedRendering/19_BRDF_BSSRDF.jpeg)

在上述这些 BxDF 中，BRDF 最为简单，也最为常用。因为游戏和电影中的大多数物体都是不透明的，用 BRDF 就完全足够。而 BSDF、BTDF、BSSRDF 往往更多用于半透明材质和次表面散射材质。

## 迪士尼原则的 BRDF（Disney Principled BRDF）

基于物理的渲染，其实早在20世纪就已经在图形学业界有了一些讨论，2010年在 SIGGRAPH 上就已经有公开讨论的 Course 《SIGGRAPH 2010 Course: Physically-Based Shading Models in Film and Game Production》，而直到 2012~2013 年，才正式进入大众的视野，渐渐被电影和游戏业界广泛使用。

Disney 动画工作室则是这次 PBR 革命的重要推动者。Disney 的 Brent Burley 于 SIGGRAPH 2012 上进行了著名的 Talk《Physically-based shading at Disney》，提出了 Disney 原则的 BRDF（Disney Principled BRDF）， 由于其高度的通用性，将材质复杂的物理属性，用非常直观的少量变量表达了出来（如金属度 metallic 和粗糙度 roughness），在电影业界和游戏业界引起了不小的轰动。从此，基于物理的渲染正式进入大众的视野。Disney 原则的 BRDF 奠定了后续游戏行业和电影行业 PBR 的方向和标准。

### Disney 采用的 BRDF 可视化方案和工具

在BRDF可视化方面，Disney 在分享中提出了三个方面的工具与资源，可以总结如下：

- **MERL 100 BRDF材质库**。Matusik等人[Matusik et al.2003]捕获的一组100个各向同性BRDF材质样本库。涵盖了各种材质，包括油漆，木材，金属，织物，石材，橡胶，塑料和其他合成材质。对学术与研究免费授权。
  - MERL BRDF主站：[http://merl.com/brdf](http://merl.com/brdf)
  - Database地址：[https://people.csail.mit.edu/wojciech/BRDFDatabase/](https://people.csail.mit.edu/wojciech/BRDFDatabase/)
- **BRDF Explorer**。Disney 为分析、比较和新开发BRDF模型而开发的可视化工具。该工具在分析测量材质，比较现有模型，以及开发新模型方面具有无可估量的价值。
  - GitHub地址：[https://github.com/wdas/brdf](https://github.com/wdas/brdf)
- **BRDF Image Slice切片**。将 $\theta h$ 与 $\theta d$ 作为横轴和纵轴，对观察到的材质的BRDF进行建模的2D图像切片。

下图展示了红色塑料和镜面红色塑料的 BRDF 图像切片以及 “切片空间（Slice Space）” 示意图：

![23_brdf_image_slice](/assets/images/2024/2024-08-12-PhysicallyBasedRendering/23_brdf_image_slice.jpeg)

### Disney 对 MERL 材质数据库的观察结论

Disney 对数据库大量的材质进行观察和理论分析，总结了以下 6 个部分的结论：

1. **漫反射（Diffuse）项的观察结论**
   - 漫反射表示折射到表面，经过散射和部分吸收，最终重新出射的光。
   - 被着色的电介质材质的任意出射部分都可视为漫反射。
   - 通过观察得出，很少有材质的漫反射表现和朗伯反射模型（Lambert 反射模型，也有叫朗伯特反射模型）相吻合，即需要更准确的漫反射模型。
   - 通过观察得出掠射逆反射（Grazing Retroreflection）有明显的着色现象，即可以将掠射逆反射也看作一种漫反射现象。
      - 掠射逆反射是指光线以接近平行于表面（即掠射）的角度入射，并在入射方向的反向反射回来的现象。
   - 粗糙度会对菲涅尔折射造成影响，而一般的漫反射模型（如 Lambert 模型）忽略了这种影响。

   下图展示了，红色塑料、镜面红色塑料和 Lambert 漫反射的点光源响应：

   ![24_diffuse_result](/assets/images/2024/2024-08-12-PhysicallyBasedRendering/24_diffuse_result.jpeg)

2. **高光（Specular）的法线分布项（Normal Distribution Function, NDF，D 项）的观察结论**
   - 微观分布函数 $D(\theta h)$ 可以通过测量材质的逆反射（Retroreflection）响应观察而得到。
   - 绝大多数 MERL 材质都有镜面波瓣（Specular Lobe），且尾部比传统的镜面模型长的多，即反射的法线分布项需要更宽的尾部。
   - GGX 比其它分布具有更长的尾部，但仍然无法捕捉铬（[gè]，Chromium）金属样本的闪亮亮点。

   下图展示了铬金属与几个镜面法线分布的比较。其中左图是镜面波峰的对数比例图：黑色曲线表示 MERL 铬金属，红色曲线表示 GGX 分布（ $\alpha=0.006$ ）,绿色曲线表示 Beckmann 分布（ $m=0.013$ ），蓝色曲线表示 Blinn-Phong（ $n=12000$ ），其中，绿色曲线和蓝色曲线基本重合，通过这个图可以看出来 GGX 分布相较于 Beckmann 和 Blinn-Phong 有着更长的尾巴（Tail），它更慢的衰减到 0（这意味着高光衰减的更平滑，有类似光晕的现象），而 MERL 铬金属则有着更长的尾巴。右图是铬金属、GGX 和 Beckmann 分布的点光源响应，可以看到有更长的尾巴意味着更平滑的高光衰减。

   ![25_specular_d](/assets/images/2024/2024-08-12-PhysicallyBasedRendering/25_specular_d.jpeg)

3. **高光（Specular）的菲涅尔反射系数项（Fresnel Reflection Coefficient，F 项）的观察结论**
   - 菲涅尔反射系数 `F` 描述了光的方向和观察方向的夹角越大，镜面反射的强度越强。
   - 光滑表面在切线方向入射（掠射角）入射时有将近 100% 的镜面反谁。
   - 对于粗糙表面，无法实现 100% 的镜面反谁，但反射率随着光的方向和观察方向的夹角越来越大，仍会变得越来越高。
   - 每种材质在掠射角附近都显示出一些反射率的增加。
   - 掠射角入射附近的许多曲线的陡度已经大于菲涅尔效应的预测值。
4. **高光（Specular）的几何项（Geometry Function，G 项）的观察结论**
   - 几何项的影响可以间接的看作其对定向反照率（Directional Albedo）的影响。定向反照率是一种描述物体表面在特定方向上的反射能力的参数，它是反照率（Albedo）的一个特定形式，反映了表面在特定入射角度和观察角度下的反射特性。
   - 大多数材质的定向反照率对于前 70° 是相对平坦的，并且切线方向（掠射角）入射处的反照率与表面粗糙度密切相关。
   - 几何项的选择会对反射率产生影响，继而又会对表面外观产生影响。
   - 完全省略几何项和 $\frac{1}{\cos{\theta_l}\cos{\theta_v}}$ 项的模型，被称为 “No G” 模型，这种模型会导致在掠射角处过暗的问题。
5. **布料（Fabric）材质的观察结论**
   - 许多布料样本在掠射角处呈现出镜面反射的色调，并且有着比十分粗糙的材质更强的菲涅尔波峰。
   - 布料具有有色的掠射反射，可以理解为是其轮廓附近获取到材质颜色的透射纤维（Transmissive Fibers）造成的。
   - 布料在掠射角处的额外光泽增加，超出了普通微平面模型的预测范围。
6. **彩虹色（Iridescence）的观察结论**
   - 变色涂料（color-changing-paint）在 $(\theta h, \theta d)$ 空间上显示出连续的色块，切对 $\phi d$ 的依赖性最小。
   - 彩虹色远离镜面峰值的反射率非常小，所以可以将彩虹色理解为一种镜面反射现象。
   - 可以将镜面色调调制为 $\theta h$ 和 $\theta d$ 的函数，配合小尺寸纹理贴图对彩虹色进行建模。

### Disney 原则的 BRDF 的理念

在 2012 年 Disney 原则的BRDF被提出之前，基于物理的渲染都需要大量复杂而不直观的参数，此时PBR的优势，并没有那么明显。2012 年 Disney 提出，他们的着色模型是艺术导向（Art Directable）的，而不一定要是完全物理正确(Physically Correct) 的，并且对微平面BRDF的各项都进行了严谨的调查，并提出了清晰明确而简单的解决方案。

Disney 的理念是开发一种“原则性”的易用模型，而不是严格的物理模型。正因为这种艺术导向的易用性，能让美术同学用非常直观的少量参数，以及非常标准化的工作流，就能快速实现涉及大量不同材质的真实感的渲染工作。而这对于传统的着色模型来说，是不可能完成的任务。

Disney 原则的BRDF（Disney Principled BRDF）核心理念如下：

- 应使用直观的参数，而不是物理类的晦涩参数。
- 参数应尽可能少。
- 参数在其合理范围内应该为0到1。
- 允许参数在有意义时超出正常的合理范围。
- 所有参数组合应尽可能健壮和合理。

而从本质上而言，Disney 原则的 BRDF 模型是金属和电介质的混合型模型，最终结果是基于金属度（Metallic）在金属 BRDF 和电介质 BRDF 之间进行线性插值：

![26_metallic_lerp_dielectric](/assets/images/2024/2024-08-12-PhysicallyBasedRendering/26_metallic_lerp_dielectric.jpeg)

因为这套新的渲染理念统一了金属和电介质的材质表述，可以仅通过少量的参数来涵盖自然界中绝大多数的材质，并可以得到非常逼真的渲染品质。也正因如此，在 PBR 的金属/粗糙度工作流中，固有色（baseColor）贴图才会同时包含金属和电介质的材质数据：

- 对金属而言，是反射率值。
- 对电介质而言，是漫反射颜色。

### Disney 原则的 BRDF 的参数

以上述理念为基础，Disney 动画工作室对每个参数的添加进行了把关，最终得到了一个颜色参数（baseColor）和下面描述的十个标量参数：

- **baseColor（基础色）**：表面颜色，通常由纹理贴图提供。
- **subsurface（次表面）**：使用次表面近似控制漫反射形状。
- **metallic（金属度）**：金属度（0 = 电介质，1 = 金属）。这是两种不同模型之间的线性混合。金属模型没有漫反射成分，并且还具有等于基础色的着色入射镜面反射。
- **specular（镜面反射强度）**：入射镜面反射量。用于取代折射率。
- **specularTint（镜面反射颜色）**：对美术控制的让步，用于对基础色（basecolor）的入射镜面反射进行颜色控制。掠射镜面反射仍然是非彩色的。
- **roughness（粗糙度）**：表面粗糙度，控制漫反射和镜面反射。
- **anisotropic（各向异性强度）**：各向异性程度。用于控制镜面反射高光的纵横比。（0 = 各向同性，1 = 最大各向异性。）
- **sheen（光泽度）**：一种额外的掠射分量（grazing component），主要用于布料。
- **sheenTint（光泽颜色）**：对sheen（光泽度）的颜色控制。
- **clearcoat（清漆强度）**：有特殊用途的第二个镜面波瓣（specular lobe）。
- **clearcoatGloss（清漆光泽度）**：控制透明涂层光泽度，0 = “缎面（satin）”外观，1 = “光泽（gloss）”外观。

下图展示了 Disney 原则的 BRDF 参数：

![27_disney_principled_brdf_params](/assets/images/2024/2024-08-12-PhysicallyBasedRendering/27_disney_principled_brdf_params.jpeg)

### Disney 原则的 BRDF 的着色模型

#### **核心 BRDF 模型**

Disney 采用了通用的 Microfacet Cook-Torrance BRDF 着色模型：

$$
f_r(l, v) = f_d + \frac{D(\theta_h)F(\theta_d)G(\theta_l, \theta_v)}{4\cos{\theta_l}\cos{\theta_v}}
$$

其中：

- $f_d$ 为漫反射项。
- $\frac{D(\theta_h)F(\theta_d)G(\theta_l, \theta_v)}{4\cos{\theta_l}\cos{\theta_v}}$ 为镜面反射项。
- $\theta_l$ 表示的是入射光线 $\mathbf{l}$ 与法线 $\mathbf{n}$ 之间的的夹角， $\theta_v$ 表示的是观察方向 $\mathbf{v}$ 与法线 $\mathbf{n}$ 之间的的夹角， $\theta_h$ 表示的是法线 $\mathbf{n}$ 与半程向量 $\mathbf{h}$ 之间的夹角，而 $\theta_d$ 表示的是入射光线 $\mathbf{l}$ 与半程向量（Half Vector， $\mathbf{h} = \text{normalize}(\mathbf{l} + \mathbf{v})$ ）之间的的夹角。

#### **漫反射项（Diffuse）：Disney Diffuse**

Disney 表示，Lambert漫反射模型在边缘上通常太暗，而通过尝试添加菲涅尔因子以使其在物理上更合理，但会导致其更暗。所以，根据对 Merl 100 材质库的观察，Disney 开发了一种用于漫反射的新的经验模型，以在光滑表面的漫反射菲涅尔阴影和粗糙表面之间进行平滑过渡。思路方面，Disney 使用了 Schlick Fresnel 近似，并对掠射逆反射（Grazing Retroreflection）进行了修改，以根据粗糙度来确定其特定反射强度，而不是简单的将其设为 0（在传统的漫反射模型中，掠射角度下的反射通常被简单地设为 0。然而，现实世界中许多材料在掠射角度下会表现出显著的反射。为了解决这个问题，Disney 的 BRDF 模型引入了粗糙度参数来调整掠射逆反射的强度）。

Disney Diffuse漫反射模型的公式为：

$$
f_d = \frac{baseColor}{\pi}(1 + (F_{D90} - 1.0)(1.0 - \cos{\theta_l})^5)(1.0 + (F_{D90} - 1.0)(1.0 - \cos{\theta_v})^5)
$$

其中， $F_{D90}$ 是：

$$
F_{D90} = 0.5 + 2.0 \times \text{roughness} \times \cos^2{\theta_d}
$$

以下为上述 Disney Diffuse 的 Shader 代码实现：

```c
// [Burley 2012, "Physically-Based Shading at Disney"]
float3 Diffuse_Burley_Disney(float3 baseColor, float roughness, float NdotV, float NdotL, float LdotH)
{
    float Fd90 = 0.5 + 2.0 * LdotH * LdotH * roughness;
    float FdL = 1.0 + (Fd90 - 1.0) * Pow5(1.0 - NdotL);
    float FdV = 1.0 + (Fd90 - 1.0) * Pow5(1.0 - NdotV);
    return baseColor * (1.0 / PI) * FdL * FdV;
}
```

#### **法线分布项（Specular D）：GTR**

在流行的模型中，GGX 拥有最长的尾部。而 GGX 其实与 Blinn（1977）推崇的 Trowbridge-Reitz（1975）分布函数模型的分布相同。然而，对于许多材质而言，即便是 GGX 分布，仍然没有足够长的尾部。

GGX（Trowbridge-Reitz） 模型（TR）的公式为：

$$
D_{TR} = \text{c} / (\text{a}^{2}\cos^2{\theta_h} + \sin^2{\theta_h})^2
$$

其中：

- $\text{c}$ 为缩放常数（Scaling Constant）。
- $\text{a}$ 为粗糙度参数，其值在 0 和 1 之间： $\text{a} = 0$ 产生完全平滑的分布， $\text{a} = 1$ 产生完全粗糙的分布。

而来自 Berry（1923）的分布函数模型和 GGX（Trowbridge-Reitz） 分布具有非常相似的形式，但指数为 1 而不是 2，从而导致了更长的尾部：

$$
D_{Berry} = \text{c} / (\text{a}^{2}\cos^2{\theta_h} + \sin^2{\theta_h})
$$

通过对 GGX（Trowbridge-Reitz） 分布和 Berry 分布的对比，Disney 发现其具有相似的形式，只是幂次不同，于是，Disney 将 GGX（Trowbridge-Reitz） 进行了 N 次幂的推广，并将其取名为 Generalized-Trowbridge-Reitz，简称为 GTR：

$$
D_{GTR} = \text{c} / (\text{a}^{2}\cos^2{\theta_h} + \sin^2{\theta_h})^{\gamma}
$$

可以发现，上式中：

- $\gamma = 1$ 时，GTR 即是 Berry 分布。
- $\gamma = 2$ 时，GTR 即是 GGX（Trowbridge-Reitz） 分布。

下图是为各种 $\gamma$ 值的 GTR 分布曲线与 $\theta_h$ 的关系图：

![28_GTR_Curves](/assets/images/2024/2024-08-12-PhysicallyBasedRendering/28_GTR_Curves.jpeg)

在 Disney 原则的 BRDF 中，Disney 使用了两个固定镜面反射波瓣（Specular Lobe），它们都使用 GTR 模型：

- 主波瓣（Primary Lobe）
  - 使用 $\gamma = 2$ 的 GTR 分布（即是 GGX（Trowbridge-Reitz） 分布）.
  - 代表基础材料（Base Material）的反射。
  - 可以是各向异性（Anisotropic）或各向同性（Isotropic）的金属或电介质。
- 次级波瓣（Secondary Lobe）
  - 使用 $\gamma = 1$ 的 GTR 分布（即 Berry 分布）。
  - 代表基础材料之上的清漆层（Clearcoat Layer）的反射。
  - 只能是各向同性和电介质材质。

**而对于粗糙度 $\text{a}$ ，Disney 发现映射 $\text{a} = \text{roughness}^2$ 能够在感知上更线性的改变粗糙度**。如果不进行这种映射，要表现光泽平滑的材质时需要非常小且不直观的粗糙度值，而且，在粗糙和光滑的材质之间进行插值时，总是会产生粗糙的结果。

最后，一个合理的法线分布函数，需要确保其在半球上积分为 1，从而保持能量守恒，而且为了高效渲染，分布函数还必须支持重要性采样。上面这些分布函数中的缩放常数 $\text{c}$ 就是用来规范化分布函数的。Disney 在 《Physically-based shading at Disney》的附录 B 中对 GTR 分布的 $\text{c}$ 进行了推导。最终归一化后的 GTR 分布函数表示如下：

- 在 $\gamma = 1$ 时：

$$
D_{GTR_1} = \frac{a^2 - 1.0}{\pi \log{a^2}} \frac{1.0}{(1.0 + (a^2 - 1.0)\cos^2{\theta_h})} = \frac{a^2 - 1.0}{\pi \log{a^2}(1.0 + (a^2 - 1.0)\cos^2{\theta_h})}
$$

- 在 $\gamma = 2$ 时：

$$
D_{GTR_2} = \frac{a^2}{\pi} \frac{1.0}{(1.0 + (a^2 - 1.0)\cos^2{\theta_h})^2} = \frac{a^2}{\pi (1.0 + (a^2 - 1.0)\cos^2{\theta_h})^2}
$$

- 以及各向异性的版本：

$$
D_{GTR_{2}aniso} = \frac{1.0}{\pi} \frac{1.0}{a_x a_y} \frac{1.0}{((\mathbf{h} \cdot \mathbf{x})^2 / a_{x}^2 + (\mathbf{h} \cdot \mathbf{y})^2 / a_{y}^2 + (\mathbf{h} \cdot \mathbf{n})^2)^2} =
\frac{1.0}{\pi a_x a_y ((\mathbf{h} \cdot \mathbf{x})^2 / a_{x}^2 + (\mathbf{h} \cdot \mathbf{y})^2 / a_{y}^2 + (\mathbf{h} \cdot \mathbf{n})^2)^2}
$$

下面是 $\gamma = 1$ 和 $\gamma = 2$ 时 GTR 分布的 Shader 实现代码：

```c
// Generalized-Trowbridge-Reitz distribution
float GTR1(float NdotH, float a)
{
    if (a >= 1.0) return 1/PI;
    float a2 = a * a;
    float t = 1.0 + (a2 - 1.0) * NdotH * NdotH;
    return (a2 - 1.0) / (PI * log(a2) * t);
}

float GTR2(float NdotH, float a)
{
    float a2 = a * a;
    float t = 1.0 + (a2 - 1.0) * NdotH * NdotH;
    return a2 / (PI * t * t);
}
```

以及各向异性的版本：

```c
float sqr(float x) { return x * x; }

float GTR2_aniso(float NdotH, float HdotX, float HdotY, float ax, float ay)
{
    return 1.0 / (PI * ax * ay * sqr(sqr(HdotX / ax) + sqr(HdotY / ay) + NdotH * NdotH));
}
```

需要注意的是，`ax` 和 `ay` 分别表示沿切线方向 $\mathbf{t}$ 和副切线方向 $\mathbf{b}$ 的粗糙度。一般 Shader 的写法，会将切线方向 $\mathbf{t}$ 写作 `X`，副切线 $\mathbf{b}$ 写作 Y。

#### **菲涅尔项（Specular F）：Schlick Fresnel**

菲涅尔项（Specular F）方面，Disney 表示 Schlick Fresnel 近似已经足够精确，且比完整的菲涅尔方程简单的多。这种近似引入的误差远小于其它因素引起的误差。Schlick Fresnel 近似公式如下：

$$
F_{Schlick} = F_0 + (1.0 - F_0)(1.0 - \cos{\theta_d})^5
$$

其中：

- 常数 $F_0$ 表示垂直入射时的镜面反射率。
- $\theta_d$ 为是入射光线 $\mathbf{l}$ 与半程向量（Half Vector， $\mathbf{h}$ ）的夹角。

下面是 Schlick Fresnel 的 Shader 实现代码：

```c
float SchlickFresnel(float u)
{
    float m = clamp(1.0 - u, 0.0, 1.0);
    float m2 = m * m;
    return m2 * m2 * m; // pow(m, 5)
}
```

可以看到上面的代码避免了 pow(xx, 5.0) 的调用，以省去 pow 函数带来的稍昂贵的性能开销。

#### **几何项（Specular G）：Smith-GGX**

几何项方面，对于主镜面波瓣（Primary Specular Lobe），Disney 参考了 Walter 为 GGX 推导的 G 项，并将粗糙度参数重新映射以减少光亮表面的极端增益，即将 $\text{a}$ 从 `[0, 1]` 重映射到 `[0.5, 1]`，从而使几何项的粗糙度变化更加平滑，更便于艺术家们的使用。需要注意的是，对粗糙度重映射是在前面提到的对其平方操作之前进行的，因此最终的 $\text{a}$ 值为： $(0.5 + \text{roughness}/2)^2$ 。

以下为 Smith GGX 的几何项表达式：

$$
G(\mathbf{l},\mathbf{v},\mathbf{h}) = G_{GGX}(\mathbf{l})G_{GGX}(\mathbf{v})
$$

$$
G_{GGX}(\mathbf{v}) = \frac{2(\mathbf{n} \cdot \mathbf{v})}{(\mathbf{n} \cdot \mathbf{v}) + \sqrt{\text{a}^2 + (1 - \text{a}^2)(\mathbf{n} \cdot \mathbf{v})^2}}
$$

$$
\text{a} = (0.5 + \text{roughness} / 2.0)^2
$$

另外，对清漆层进行处理的次级波瓣（Secondary Lobe），Disney 将粗糙度固定为 `0.25`，这种选择就可以得到合理且很好的视觉效果。

几何项的 Shader 代码实现如下：

```c
// Smith GGX G项，各向同性版本
float smithG_GGX(float NdotV, float alphaG)
{
    float a = alphaG * alphaG;
    float b = NdotV * NdotV;
    return 1.0 / (NdotV + sqrt(a + b - a * b));
}

// Smith GGX G项，各向异性版本
float smithG_GGX_aniso(float NdotV, float VdotX, float VdotY, float ax, float ay)
{
    return 1.0 / (NdotV + sqrt(sqr(VdotX * ax) + sqr(VdotY * ay) + sqr(NdotV)));
}
```

Disney在 SIGGRAPH 2015 上对 G 项进行了修订，他们基于 Heitz 的分析[Heitz 2014]，淘汰了对于主镜面波瓣的 Smith G 粗糙度的特殊重映射，采用了 Heitz 各向异性的 G。

最后还有一点要注意，在上面的公式中，函数 $G_{GGX}$ 的分子是 $2(\mathbf{n} \cdot \mathbf{v})$ ，而为什么 Shader 代码中却是 `1.0` 呢？

这是因为在渲染方程中：

$$
f(l, v) = \text{diffuse} + \frac{D(\theta_h)F(\theta_d)G(\theta_l, \theta_v)}{4\cos{\theta_l}\cos{\theta_v}}
$$

反射部分最终要除以 $4\cos{\theta_l}\cos{\theta_v}$ ，也就是 $4(\mathbf{n} \cdot \mathbf{l})(\mathbf{n} \cdot \mathbf{v})$ ，而几何 G 项中， $G_{GGX}(\mathbf{l})G_{GGX}(\mathbf{v})$ 的结果中，分子正好就是：

$$
2(\mathbf{n} \cdot \mathbf{l}) \times 2(\mathbf{n} \cdot \mathbf{v}) = 4(\mathbf{n} \cdot \mathbf{l})(\mathbf{n} \cdot \mathbf{v})
$$

所以在 Shader 的代码中，直接提前约掉了 G 项中分子的部分，这样在最终计算渲染结果的方程中， $D(\theta_h)F(\theta_d)G(\theta_l, \theta_v)$ 的结果也不需要再除以 $4(\mathbf{n} \cdot \mathbf{l})(\mathbf{n} \cdot \mathbf{v})$ 了。

## 基于物理的环境光照（Physically-based Environment Lighting）

渲染方程和反射方程描述的是直接光部分，而在实际的渲染中，还需要环境光。基于物理的环境光照，一般直接默认说的是 **基于图像的光照（Image-based Lighting，IBL）**，这也是真正让基于物理的渲染画质提升的主要贡献者。

基于图像的光照（Image-based Lighting），简称 IBL，是一种用于在三维场景中模拟现实世界中的光照效果的技术，它通过将周围环境视为一个大的光源来为物体照明。常见的环境贴图包括立方体贴图（Cube Map）和球形贴图（Spherical Map）。环境贴图可以是从真实世界中获取的图像，也可以是从 3D 场景生成出的图像。在物体的照明计算中，将环境贴图的每个纹素视为一个光源，来计算光照，通过这种方式，可以有效的捕捉环境的全局光照，使物体更好地融入环境中。下图展示了球形贴图(左)和立方体贴图（右）的对比：

![30_sphericalmap_cubemap](/assets/images/2024/2024-08-12-PhysicallyBasedRendering/30_sphericalmap_cubemap.jpg)

再来回顾一下渲染方程：

$$
L_o(\mathbf{p}, \omega_o) = L_e(\mathbf{p}, \omega_o) + \int_{\Omega}^{}L_i(\mathbf{p}, \omega_i)f_r(\mathbf{p}, \omega_i, \omega_o)(n \cdot \omega_i)d\omega_i
$$

对于直接光照的计算，入射光线 $\omega_i$ 和出射方向（也就是观察方向） $\omega_o$ 是已知的，场景中的直接光照光源的数量也是已知的，所以直接光照的计算只需要计算渲染方程中被积函数部分，也就是 $L_i(\mathbf{p}, \omega_i)f_r(\mathbf{p}, \omega_i, \omega_o)(n \cdot \omega_i)$ ，而不需要做积分。而对于环境光照的计算，它需要求解半球 $\Omega$ 上所有入射方向（ $\omega_i$ ）的积分（对于某一个着色点 $\mathbf{p}$ 来说，来自周围环境的每个入射方向 $\omega_i$ 都有可能具有一定的辐射度）。因此，环境光照的计算涉及对整个半球 $\Omega$ 的积分，以考虑所有可能的间接光照。

而在实时渲染中使用环境光照，又对求解这个积分提了两个主要要求：

- 需要一种方法来获取给定任何方向向量 $\omega_i$ 的环境辐射度。
- 求解积分需要快速且实时（Real-time）。

第一个要求很好满足，在渲染中使用立方体贴图来表示环境光照，可以将立方体贴图的每一个纹素视为一个单独发光的光源，通过使用任何方向向量 $\omega_i$ 来采样这个立方体贴图，就可以获取该方向的环境辐射度。而对于第二个要求，就需要先了解一些求解积分相关的数学知识了。

### 求解积分相关的数学知识

#### **一些概率论的概念**

- **随机变量**：用于表示一组潜在值得分布，随机变量得分布描述了如何将概率分配给随机变量可能取的值。随机变量一般用大写字母表示（如 $X$ ），具体的取值用小写字母表示（如 $x$ ）。随机变量分为离散随机变量和连续随机变量：
  - **离散随机变量**：取值可能为有限个或可数无限个。例如掷一枚骰子，可能点数为 1 到 6，这些都是离散的取值。
  - **连续随机变量**：取值为一个区间内的无限多个实数。
- **概率密度函数（Probability Density Function，PDF）**：用以描述连续随机变量在各个取值上的概率分布情况。对于连续随机变量 $X$ ，其概率密度函数 $p(x)$ 满足以下条件：
  - 非负性：对于所有的 $x$ ，有 $p(x) \geq 0$ 。
  - 归一性：概率密度函数在整个定义域上的积分为 1，即所有的概率加起来要是 1： $\int_{}^{} p(x)dx = 1$ 。
  - 概率解释：对于任意的区间 `[a, b]`，随机变量 $X$ 落在该区间内的概率可以表示为： $P(a \leq X \leq b) = \int_{a}^{b} f(x)dx$
- **期望值（Expected Value）**：用来描述随机变量的平均值或中心趋势（Central Tendency）。可以理解为随机变量在大量重复实验中的平均结果。对于一个连续随机变量 $X$ ，其期望值 $E[X]$ 为所有可能的取值和其概率密度的乘积在定义域上积分起来的值，即： $E[X] = \int{}^{} x \ p(x)dx$ 。
  - 而对于一个随机变量的函数 $Y = f(X)$ ，其中： $X$ ~ $p(x)$ ，那么函数 $Y$ 的期望可以表示为： $E[Y] = E[f(X)] = \int{}^{}f(x)p(x)dx$ 。
- **累积分布函数（Cumulative Distribution Function，CDF）**：用以描述随机变量的分布情况。对于一个给定的随机变量 $X$ ，其累积分布函数 $F_X(x) = P(X \leq x)$ ，其中 $P(X \leq x)$ 表示随机变量 $X$ 取值小于或等于 $x$ 的概率。

#### **蒙特卡罗积分（Monte Carlo Integration）**

蒙特卡罗积分是一种利用随机采样来估算定积分的方法，它基于概率和统计学原理，通过在积分区域内生成大量随机点来近似积分的值。假设要对一个一维函数 $f(x)$ 从 `a` 到 `b` 进行积分，函数如下：

$$
\int_{a}^{b}f(x)dx
$$

那么其对应的 **蒙特卡罗估算器（Monte Carlo Estimator）** 为：

$$
F_N = \frac{1}{N}\sum_{i=0}^{N-1}\frac{f(X_i)}{\text{pdf}(X_i)}
$$

其中， $pdf(x)$ 是随机变量 $X$ 的概率密度函数，而 $X_i$ 是根据 $\text{pdf}(x)$ 生成的随机样本， $N$ 是随机样本数。

下面举一个特殊的例子来更好的说明蒙特卡罗积分的主要思想。先看下图：

![31_monte_carlo_integration](/assets/images/2024/2024-08-12-PhysicallyBasedRendering/31_monte_carlo_integration.jpeg)

首先要明确的是，对一个函数进行求积分，可以理解为是计算函数曲线下方的面积，如上图所示要对函数 $f(x)$ 在 `[a, b]` 范围内求积分，也就是计算上面最后一张图中函数曲线下黑色部分的面积。

另外，假设随机变量 $X$ 在 `[a, b]` 之间**均匀分布（Uniform Distribution）**，也就是说随机变量 $X$ 在 `[a, b]` 范围内每个取值的概率都是一样的，也就是 $\text{pdf}(x) = \frac{1}{b - a}$ 。

要求上图中黑色区域的面积，可以在 `[a, b]` 内随机选取一个值 $x1$ ，在 $x1$ 处计算函数 $f(x1)$ ，然后将函数结果乘以 $(b - a)$ ，最终得到的面积可以将其视为函数曲线下方面积的一个非常粗略的近似。一次随机取值计算的结果可能和最终正确的积分结果相差甚大（上图中在 $x1$ 处计算出的红色区域面积可以明显的看出来要比黑色区域面积小很多），但随着随机样本数量的不断增加，累加这些矩形的面积并求平均值会最终收敛到要求的黑色区域面积（比如在 $x2$ 处的计算，会高估这个面积，但是和 $x1$ 的结果求和在平均后，会变得更接近正确的结果）。上图展示了 4 次随机选择的 $x$ 的值，并计算 $f(x) \times (b - a)$ ，然后求和并平均（除以 4）。这种在随机变量 $X$ 的概率密度函数 $\text{pdf}(x)$ 是均匀分布的情况下的蒙特卡罗估算器的表达式为：

$$
F_N = \frac{1}{N}\sum_{i=0}^{N-1}\frac{f(X_i)}{\frac{1}{b - a}} = (b - a)\frac{1}{N}\sum_{i=0}^{N-1}f(X_i)
$$

另外，在蒙特卡罗积分估算器中，函数 $f(x)$ 为什么要除以 $\text{pdf}(x)$ 呢？首先， $\text{pdf}(x)$ 给出了随机变量 $X$ 取某个值的概率，从任意 PDF 中抽取样本时，样本并不是均匀分布的：而是在 PDF 高的地方生成更多的样本，而在 PDF 低的地方生成更少的样本，如下图所展示的 PDF 分布：

![32_samplespdf](/assets/images/2024/2024-08-12-PhysicallyBasedRendering/32_samplespdf.png)

在蒙特卡罗积分中，如果在函数的某些 PDF 高的区域生成大量的样本，会导致最终的结果并不能体现函数完整的曲线，会有较大的误差。而通过 $f(x)$ 除以 $\text{pdf}(x)$ 会抵消这种效应：当 PDF 高时（更多的样本），除以 $\text{pdf}(x)$ 后，会减少这些样本在和中的权重；当 PDF 低时（样本更少），除以一个很小的 $\text{pdf}(x)$（比如 0.1），这些样本的贡献会被放大。总结来说，就是高 PDF 的样本降低它的贡献，低 PDF 的样本增加它的贡献。通过这种方式使估算器的结果更接近正确的积分结果。

通过上面的了解可以知道，蒙特卡罗积分是通过生成随机样本，并对每个样本的贡献进行加权平均求和以得到最终的结果。

#### **低差异序列（low-discrepancy sequence）：Hammersley序列（Hammersley Sequence）**

默认情况下，蒙特卡罗积分的每个样本是完全（伪）随机的，但这样会造成一个问题，叫做 **样本聚集（Sample Clumping）**，当随机生成样本时，其中一些可能非常接近，它们形成了聚集，由于样本的数量有限，所以需要通过随机样本更多地收集被积函数的信息，如果两个（或多个）样本彼此非常接近，它们提供的信息（多多少少）是相同的，因此其中一个是有用的，而其它的则浪费了。这种样本聚集的问题会造成最终结果的偏差。下图展示了样本聚集的一个例子：

![33_clumping](/assets/images/2024/2024-08-12-PhysicallyBasedRendering/33_clumping.png)

而通过使用 **低差异序列（low-discrepancy sequence）** 来生成蒙特卡罗积分的样本可以避免样本聚集的问题，这种方式的蒙特卡罗积分也叫做 **准蒙特卡罗积分（Quasi-Monte Carlo integration）**。准蒙特卡罗积分具有更快的收敛速度，通过这种方法可以显著提高蒙特卡罗积分的效率和精度。

低差异序列通过特定的生成规则，使得生成出的样本点在样本空间内均匀分布，但不完全均匀（有差异，差异尽可能的小，但不是零差异），这些序列在每个区域内的样本密度接近相同。下图展示了伪随机序列和低差异序列的对比：

![34_low_discrepancy_sequence](/assets/images/2024/2024-08-12-PhysicallyBasedRendering/34_low_discrepancy_sequence.jpeg)

**Hammersley 序列（Hammersley Sequence）** 基于 Van Der Corput 序列，该序列通过小数点镜像二进制表示。Hammersley 序列在低维情况下表现尤为出色，特别是在二维情况下，其构造方法简单且具有良好的分布特性。

生成 Hammersley 序列的 Shader 代码：

```c
vec2 Hammersley2d(uint i, uint N) 
{
    // Radical inverse based on http://holger.dammertz.org/stuff/notes_HammersleyOnHemisphere.html
    uint bits = (i << 16u) | (i >> 16u);
    bits = ((bits & 0x55555555u) << 1u) | ((bits & 0xAAAAAAAAu) >> 1u);
    bits = ((bits & 0x33333333u) << 2u) | ((bits & 0xCCCCCCCCu) >> 2u);
    bits = ((bits & 0x0F0F0F0Fu) << 4u) | ((bits & 0xF0F0F0F0u) >> 4u);
    bits = ((bits & 0x00FF00FFu) << 8u) | ((bits & 0xFF00FF00u) >> 8u);
    float rdi = float(bits) * 2.3283064365386963e-10;
    return vec2(float(i) /float(N), rdi);
}
```

#### **重要性采样（Importance Sampling）**

对于蒙特卡罗积分和准蒙特卡罗积分，还有一个性质可以用来实现更快的收敛速度，那就是 **重要性采样（Importance Sampling）**。重要性采样，即通过现有的一些已经条件（分布函数），想办法集中于被积函数分布可能性较高的区域进行采样，进而可高效的计算准确的估算结果的一种策略。

当涉及到光的镜面反射时，反射光向量被限制在一个由表面粗糙度决定的镜面波瓣（Specular Lobe）中。鉴于任何在镜面波瓣之外生成的随机样本对镜面积分没有意义，因此将样本生成集中在镜面叶片内是合理的，尽管这样做会使蒙特卡罗估算器有偏。

**GGX 重要性采样**，有别于在积分的半球 $\Omega$ 上均匀或随机的生成样本，而是基于 GGX（Trowbridge-Reitz）微表面模型法线分布，，生成偏向于微表面法线向量 $\mathbf{h}$ （也就是 入射方向 $\mathbf{l}$ 和 出射方向 $\mathbf{v}$ 之间的半程向量）的样本向量。然后利用反射关系通过出射方向 $\mathbf{v}$ 和微表面法线向量 $\mathbf{h}$ 计算出入射方向 $\mathbf{l}$。

为了在渲染中实现高效采样，重要性采样的目标是生成分布与 GGX（Trowbridge-Reitz）分布相似分布的样本，从而减少噪声和提高效率。

GGX 重要性采样生成偏向于微表面法线向量的 Shader 实现代码如下：

```c
vec3 ImportanceSampleGGX(vec2 Xi, vec3 N, float roughness)
{
    // 参数 Xi 是生成的低差异 Hammersley 序列

    // GGX（Trowbridge-Reitz）分布中通常使用粗糙度的平方来计算
    float a = roughness * roughness;

    // 通过 Xi.x 生成的方位角，范围是 [0, 2π]
    float phi = 2.0 * M_PI * Xi.x;
    // 通过 Xi.y 生成的极角的余弦值，使用了 GGX（Trowbridge-Reitz）分布的逆变换采样方法
    // theta = atan(a * sqrt(Xi.y) / sqrt(1.0 - Xi.y));
    float cosTheta = sqrt((1.0 - Xi.y) / (1.0 + (a * a - 1.0) * Xi.y));
    // 极角的正弦值，通过三角恒等式计算
    float sinTheta = sqrt(1.0 - cosTheta * cosTheta);

    // 从球面坐标到笛卡尔坐标，得到半程向量 H
    vec3 H;
    H.x = cos(phi) * sinTheta;
    H.y = sin(phi) * sinTheta;
    H.z = cosTheta;

    // 转换到世界坐标
    // 选择一个参考向量。如果 N 的 z 分量不接近 1，选择 (0, 0, 1) 作为参考向量，否则选择 (1, 0, 0)
    vec3 up = abs(N.z) < 0.999 ? vec3(0.0, 0.0, 1.0) : vec3(1.0, 0.0, 0.0);
    // 通过叉乘计算法线 N 的正交切线向量
    vec3 T = normalize(cross(up, N));
    // 通过叉乘计算法线 N 和切线向量的正交副切线向量
    vec3 B = cross(N, T);

    // 变换采样向量到世界坐标系
    return normalize(vec3(dot(T, H), dot(B, H), dot(N, H)));
}
```

**TODO：在这里只是列出了实现的代码，但是具体 GGX 采样点生成的推导过程还需要详细的学习分析。下图展示了主要思路和一些文章参考** ：

![39_GGX_ImportanceSampling](/assets/images/2024/2024-08-12-PhysicallyBasedRendering/39_GGX_ImportanceSampling.png)

- https://blog.tobias-franke.eu/2014/03/30/notes_on_importance_sampling.html
- https://blog.selfshadow.com/publications/s2013-shading-course/#course_content
- https://placeholderart.wordpress.com/2015/07/28/implementation-notes-runtime-environment-map-filtering-for-image-based-lighting/

<!-- #### **球谐函数（Spherical Harmonics）** -->

### 实时渲染中的环境光照

渲染方程是一切光照计算的基础，所以再次回顾一下渲染方程

$$
L_o(\mathbf{p}, \omega_o) = L_e(\mathbf{p}, \omega_o) + \int_{\Omega}^{}L_i(\mathbf{p}, \omega_i)f_r(\mathbf{p}, \omega_i, \omega_o)(n \cdot \omega_i)d\omega_i
$$

对于反射积分部分中的 $f_r(\mathbf{p}, \omega_i, \omega_o)$ 就是 BRDF 反射方程：

$$
f_r(\mathbf{p}, \omega_i, \omega_o) = f_d + \frac{DFG}{4(\mathbf{n} \cdot \omega_i)(\mathbf{n} \cdot \omega_o)}
$$

那么反射积分可以写为如下公式，并拆分漫反射部分和镜面反射部分：

$$
\int_{\Omega}^{}L_i(\mathbf{p}, \omega_i)(f_d + \frac{DFG}{4(\mathbf{n} \cdot \omega_i)(\mathbf{n} \cdot \omega_o)})(\mathbf{n} \cdot \omega_i)d\omega_i =
\int_{\Omega}^{}L_i(\mathbf{p}, \omega_i)f_d(\mathbf{n} \cdot \omega_i)d\omega_i + \int_{\Omega}^{}L_i(\mathbf{p}, \omega_i)\frac{DFG}{4(\mathbf{n} \cdot \omega_i)(\mathbf{n} \cdot \omega_o)}(\mathbf{n} \cdot \omega_i)d\omega_i
$$

通过这样的变换，把环境光照拆分为两部分的积分，漫反射部分和镜面反射部分。

#### **漫反射环境光**

对于漫反射部分的环境光，先看看上面总结的积分公式：

$$
\int_{\Omega}^{}L_i(\mathbf{p}, \omega_i)f_d(\mathbf{n} \cdot \omega_i)d\omega_i
$$

$f_d$ 表示的是物体表面漫反射的一些属性，比如基础色，折射率等，与积分无关，所以可以把 $f_d$ 从积分中提出来：

$$
f_d\int_{\Omega}^{}L_i(\mathbf{p}, \omega_i)(\mathbf{n} \cdot \omega_i)d\omega_i
$$

至此，需要求解的积分问题就是来自半球 $\Omega$ 上的所有方向的辐射度的积分。如下图所示：

![29_environment_intergration](/assets/images/2024/2024-08-12-PhysicallyBasedRendering/29_environment_intergration.png)

求解这个积分需要从半球 $\Omega$ 上的所有可能方向进行采样，并对所有的采样结果加权平均，这对于实时渲染来说消耗太大，为了以更高效的方式求解积分，可以通过对环境贴图预计算的方式对半球上所有方向的环境光照进行加权平均，这样在实时渲染时，只需要一次采样就能得到近似积分的结果。下图展示了一个镜面反射的例子，对反射方向上镜面波瓣内所有反射方向的加权平均约等于是对环境光照上这一块区域光照信息的加权平均：

![35_prefiltering](/assets/images/2024/2024-08-12-PhysicallyBasedRendering/35_prefiltering.jpeg)

所以，可以通过预计算的方式提前计算好预过滤（Pre-filtered）的环境贴图，每个方向上的辐照度存储的都是这个方向对应的半球 $\Omega$ 上所有方向的环境辐照度的平均。在实时渲染中，通过一次采样，就得到了这个方向上对整个半球 $\Omega$ 上所有入射光的积分结果也就是上面公式中的 $\int_{\Omega}^{}L_i(\mathbf{p}, \omega_i)(\mathbf{n} \cdot \omega_i)d\omega_i$ 部分，然后再拿采样的结果与 $f_d$ 相乘，就得到了这个点的环境漫反射光照结果。

如下图所示，需要对整个半球 $\Omega$ 上的立体角 $\omega_i$ 进行积分：

![36_spherical_integrate](/assets/images/2024/2024-08-12-PhysicallyBasedRendering/36_spherical_integrate.jpeg)

半球 $\Omega$ 上的立体角 $\omega_i$ 用球面坐标可以表示为 $(\theta, \phi)$ 的形式，而 $(\mathbf{n} \cdot \omega_i)$ 可以表示为 $\cos{\theta}$ ，新的积分公式也就是：

$$
f_d\int_{\phi=0}^{2\pi}\int_{\theta=0}^{\frac{\pi}{2}}L_i(\mathbf{p}, \phi_i, \theta_i)\cos{\theta}\sin{\theta}d\phi d\theta
$$

上式中 $\theta$ 的范围是 `[0, π/2]` 是因为半球积分是围绕表面法线进行的，只计算朝向表面的光线的贡献。

最后，通过黎曼和方法（Riemann Sum）来预计算这个积分。而且累加完成后，还需要进行归一化处理，以获得正确的辐照度：因为半球的表面积为 $2\pi r^2$ ，其中 $r$ 是半径，在单位球的情况下， $r = 1$ ，因此半球的表面积为 $2\pi$ ，而在离散积分时，只考虑了半球的一部分（从 $\theta = 0$ 到 $\theta = \frac{\pi}{2}$ ），因此累加的结果还要乘以 $2\pi$ 的一半来归一化积分结果（在离散积分中，乘以 $\pi$ 可以使离散求和的结果更接近实际的连续积分结果）。最终的公式是：

$$
f_d \frac{\pi}{n1n2} \sum_{\phi=0}^{n1}\sum_{\theta=0}^{n2}L_i(\mathbf{p}, \phi_i, \theta_i)\cos{\theta}\sin{\theta}d\phi d\theta
$$

下面是预计算环境贴图的 Shader 代码：

```c
void main()
{
    // 片段法线方向
    vec3 N = normalize(UVW);
    // 定义一个向上的参考向量 
    vec3 up = vec3(0.0, 1.0, 0.0);
    vec3 right = cross(up, N);
    up = cross(N, right);

    vec3 irradiance = vec3(0.0);
    float sampleDelta = 0.025;
    float nrSamples = 0.0;

    // 根据步长 sampleDelta，遍历整个半球
    // 遍历方位角 phi，从 0 到 2π
    for (float phi = 0.0; phi < 2.0 * PI; phi += sampleDelta)
    {
        // 遍历极角 theta，从 0 到 π/2，因为是半球，是到 π/2
        for (float theta = 0.0; theta < 0.5 * PI; theta += sampleDelta)
        {
            // 将球面坐标 (theta, phi) 转换到笛卡尔坐标系 (x, y, z)
            vec3 tangentSample = vec3(sin(theta) * cos(phi), sin(theta) * sin(phi), cos(theta));

            // 转换到世界空间
            vec3 sampleVec = tangentSample.x * right + tangentSample.y * up + tangentSample.z * N;

            // 对环境贴图进行采样，并根据角度 ( \theta ) 进行加权累加到 irradiance。这里的加权因子 ( \cos(\theta) \cdot \sin(\theta) ) 用于补偿样本区域的差异。
            irradiance += texture(uEnvironmentCubemap, sampleVec).rgb * cos(theta) * sin(theta);
            nrSamples++;
        }
    }

    // 将累加的辐照度乘以 π 并归一化，得到最终的辐照度值
    irradiance = PI * irradiance * (1.0 / nrSamples);

    FragColor = vec4(irradiance, 1.0);
}
```

#### **镜面反射环境光**

而对于镜面反射部分的环境光。先将蒙特卡罗积分公式带入方程可得：

$$
\int_{\Omega}^{}L_i(\mathbf{p}, \omega_i)\frac{DFG}{4(\mathbf{n} \cdot \omega_i)(\mathbf{n} \cdot \omega_o)}(\mathbf{n} \cdot \omega_i)d\omega_i \approx
\frac{1}{N}\sum_{k=1}^{N}\frac{L_i(l_k)f(l_k, v)\cos{\theta_{l_{k}}}}{p(l_k, v)}
$$

其中 $f(l_k, v)$ 是 $\frac{DFG}{4(\mathbf{n} \cdot \omega_i)}$ ， $\cos{\theta_{l_{k}}}$ 是 $(\mathbf{n} \cdot \omega_o)$ 。

上式的直接求解较为复杂，进行完全的实时渲染不太现实。目前游戏业界的主流做法是，基于 **分解求和近似（Split Sum Approximation）** 的方法，将上式拆分为 **光亮度的均值** 和 **环境BRDF** 两项：

光亮度的均值：

$$
\frac{1}{N}\sum_{k=1}^{N}L_i(l_k)
$$

环境 BRDF：

$$
\frac{1}{N}\sum_{N}^{k=1}\frac{f(l_k, v)\cos{\theta_{l_{k}}}}{p(l_k, v)}
$$

也就是：

$$
\frac{1}{N}\sum_{k=1}^{N}\frac{L_i(l_k)f(l_k, v)\cos{\theta_{l_{k}}}}{p(l_k, v)} \approx
(\frac{1}{N}\sum_{k=1}^{N}L_i(l_k))(\frac{1}{N}\sum_{N}^{k=1}\frac{f(l_k, v)\cos{\theta_{l_{k}}}}{p(l_k, v)})
$$

完成拆分后，分别对两项进行离线预计算。在实时渲染中，分别计算分解求和近似（Split Sum Approximation）方案中几乎已经预计算好的两项，再进行组合，作为实时的IBL物理环境光照部分的渲染结果。

**第一项，预过滤环境贴图（Pre-filtered Environment Map）**：

第一项可以理解为对光亮度 $L_i(l_k)$ 求均值。经过 $\mathbf{n} = \mathbf{v} = \mathbf{r}$ 的假设，仅取决于表面粗糙度（surface roughness）和反射矢量（reflection vector）。

简单描述一下实现的原理：根据不同的粗糙度，将环境贴图预过滤到不同的多级 mipmap ，在预过滤的过程中，通过生成低差异的 Hammersley 序列，然后运用 GGX 法线分布生成随机的微表面模型下的半程向量（可以理解为对蒙特卡罗积分的重要性采样，只生成镜面波瓣内的随机向量），通过生成的向量采样环境贴图并将这些结果加权平均，最终得到的就是镜面波瓣区域环境光照辐射度的积分。在实时渲染的过程中，根据表面粗糙度采样不同 mipmap 层级的预过滤环境贴图（越粗糙的表面反射的环境光照越模糊）：

$$
mip = calculate \ from \ roughness
$$

$$
\frac{1}{N}\sum_{k=1}^{N}L_i(l_k) \approx Cubemap.sample(r, mip)
$$

**第二项，预计算环境 BRDF（Environment BRDF）**：

第二项为镜面反射项的半球方向反射率（Hemispherical-directional Reflectance），可以理解为环境 BRDF，其取决于仰角 $\theta$ ，粗糙度 `a` 和菲涅尔项 `F`。通常使用 Schlick 近似来计算 `F`，其仅在单个值 `F0` 上参数化，从而使其成为三个参数（仰角 $\theta$（NdotV），粗糙度 `a`、`F0`）的函数。

这一项的主要流派有两个，UE4 的 2D LUT，以及 COD：OP2 的解析拟合。

- **流派1：2D LUT**

  UE4在[[Real Shading in Unreal Engine 4, 2013]]中提出，第二个求和项 ，使用Schlick近似后， F0可以从积分中分出来：

  ![37_F0](/assets/images/2024/2024-08-12-PhysicallyBasedRendering/37_F0.png)

  上式留下了两个输入（粗糙度 `roughness` 和 $\cos{\theta_v}$ ）和两个输出（`F0` 的缩放和偏差（scale and bias）），即把上式看成是 `F0 * scale + bias` 的形式，预先计算此函数的结果并将其存储在2D查找纹理（LUT，look-up texture）中。

  ![38_BRDF_LUT](/assets/images/2024/2024-08-12-PhysicallyBasedRendering/38_BRDF_LUT.png)

  这张红绿色的贴图，输入roughness、cosθ，输出环境BRDF镜面反射的强度。实时渲染时的使用方式：

$$
\frac{1}{N}\sum_{N}^{k=1}\frac{f(l_k, v)\cos{\theta_{l_{k}}}}{p(l_k, v)} = LUT.r * \text{F0} + LUT.g
$$

- **流派2：解析拟合**

  COD：Black Ops 2的做法，是通过数学工具Mathematica（http://www.wolfram.com/mathematica/） 中的数值积分拟合出曲线，即将UE4离线计算的这张 2D LUT 用如下函数进行了拟合：

  ```c
  float3 EnvironmentBRDF( float g, float NoV, float3 rf0 )
  {
      float4 t = float4( 1/0.96, 0.475, (0.0275 - 0.25 * 0.04)/0.96, 0.25 );
      t *= float4( g, g, g, g );
      t += float4( 0, 0, (0.015 - 0.75 * 0.04)/0.96, 0.75 );
      float a0 = t.x * min( t.y, exp2( -9.28 * NoV ) ) + t.z; float a1 = t.w;
      return saturate( a0 + rf0 * ( a1 - a0 ) );
  }
  ```

  需要注意的是，上面的方程是基于Blinn-Phong分布的结果，https://knarkowicz.wordpress.com/2014/12/27/analytical-dfg-term-for-ibl/ 一文中提出了基于GGX分布的EnvironmentBRDF解析版本：

  ```c
  float3 EnvDFGLazarov( float3 specularColor, float gloss, float ndotv )
  {
      float4 p0 = float4( 0.5745, 1.548, -0.02397, 1.301 );
      float4 p1 = float4( 0.5753, -0.2511, -0.02066, 0.4755 );
      float4 t = gloss * p0 + p1;
      float bias = saturate( t.x * min( t.y, exp2( -7.672 * ndotv ) ) + t.z );
      float delta = saturate( t.w );
      float scale = delta - bias;
      bias *= saturate( 50.0 * specularColor.y );
      return specularColor * scale + bias;
  }
  ```

  上式中的 `specularColor` 即 `F0`。Environment BRDF 函数的输入参数分别为光泽度 `gloss`，`NdotV`，`F0`。和 UE4 的做法有异曲同工之妙，但 COD：Black Ops 2 的做法不需要额外的贴图采样，这在进行移动端优化时，是不错的选择。

## 参考

- https://github.com/QianMo/PBR-White-Paper/blob/master/content/part%201/README.md
- https://learnopengl.com/PBR/Theory
- https://github.com/KhronosGroup/glTF-IBL-Sampler
- https://www.scratchapixel.com/lessons/mathematics-physics-for-computer-graphics/monte-carlo-methods-in-practice/monte-carlo-integration.html
- https://sites.cs.ucsb.edu/~lingqi/teaching/games202.html
