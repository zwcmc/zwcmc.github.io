---
layout: post
title:  "半球上采样方案"
date:   2024-10-18 16:16:00 +800
category: Rendering
---

- [1. 球面坐标与笛卡尔坐标](#1-球面坐标与笛卡尔坐标)
- [2. 半球上采样的原理](#2-半球上采样的原理)
  - [2.1. 立体角转换到 $\\theta$ 与 $\\phi$](#21-立体角转换到-theta-与-phi)
  - [2.2. 从立体角上的 PDF 转换到相对于球面坐标 $\\theta$ 和 $\\phi$ 的联合概率密度函数](#22-从立体角上的-pdf-转换到相对于球面坐标-theta-和-phi-的联合概率密度函数)
  - [2.3. 联合概率密度函数的分解](#23-联合概率密度函数的分解)
  - [2.4. 从联合概率密度函数中进行二维采样的基本思路](#24-从联合概率密度函数中进行二维采样的基本思路)
- [3. 一些半球上采样方案](#3-一些半球上采样方案)
  - [3.1. 均匀采样半球（Uniformly Sampling a Hemisphere）](#31-均匀采样半球uniformly-sampling-a-hemisphere)
  - [3.2. 余弦加权半球采样（Cosine-Weighted Hemisphere Sampling）](#32-余弦加权半球采样cosine-weighted-hemisphere-sampling)
  - [3.3. 微表面 BRDF 半球采样（Microfacet BRDF Hemisphere Sampling）](#33-微表面-brdf-半球采样microfacet-brdf-hemisphere-sampling)
    - [3.3.1. 基于 GGX 法线分布的半球采样](#331-基于-ggx-法线分布的半球采样)
- [4. 参考](#4-参考)

## 1. 球面坐标与笛卡尔坐标

在半球上生成采样样本时，通常通过计算 **极角 $\theta$** 和 **方位角 $\phi$** 来使用单位球面（单位球的半径为 1 ）的坐标。这两个角度可以通过以下公式转换到笛卡尔坐标：

$$
\left\{
\begin{array}{ll}
x = \sin\theta \cos\phi \\
y = \sin\theta \sin\phi \\
z = \cos\theta
\end{array}
\right.
$$

## 2. 半球上采样的原理

在半球上采样的原理可以分为以下几个步骤：

1. 计算归一化的立体角上的 **概率密度函数 PDF** ，并将其转换为使用球面坐标 $\theta$ 和 $\phi$ 表示的联合概率密度函数
2. 将联合概率密度函数分解（一个是 $\theta$ 的边缘密度函数，另一个是在给定 $\theta$ 的情况下 $\phi$ 的条件密度函数）
3. 通过对分解出的两个密度函数积分计算 **累积分布函数 CDF**
4. 将 CDF 设为 0 到 1 之间的随机数，以获得定义采样样本的 $\theta$ 和 $\phi$ 角度

### 2.1. 立体角转换到 $\theta$ 与 $\phi$

首先我们知道， **对球面上立体角的微分 $\mathrm{d}\omega$ 可以通过球面坐标中极小差分变化的 $\theta$ 和 $\phi$ 来表示** ，也就是：

$$ \mathrm{d}\omega = \sin\theta \mathrm{d}\theta \mathrm{d}\phi $$

在半球上采样常见的一个操作就是 **从使用立体角的半球积分（ $\mathrm{d}\omega$ ）转换为使用球面坐标的半球积分（ $\mathrm{d}\theta \mathrm{d}\phi$ ）** ，转换过程如下：

$$
\int_{H^2} f(\theta)\mathrm{d}\omega = \int_{0}^{2\pi} \int_{0}^{\pi / 2} f(\theta, \phi) \sin\theta \mathrm{d}\theta \mathrm{d}\phi
$$

### 2.2. 从立体角上的 PDF 转换到相对于球面坐标 $\theta$ 和 $\phi$ 的联合概率密度函数

假设有一个定义在立体角上的概率密度函数 $p(\omega)$ ，此密度函数相对于球面坐标角度 $\theta$ 和 $\phi$ 的联合概率密度函数可以推导出来为：

$$
\begin{align*}
p(\theta,\phi) \mathrm{d}\theta \mathrm{d}\phi &= p(\omega) \mathrm{d}\omega \\
p(\theta,\phi) \mathrm{d}\theta \mathrm{d}\phi &= p(\omega) \sin\theta \mathrm{d}\theta \mathrm{d}\phi \\
p(\theta,\phi) &= \sin\theta p(\omega)
\end{align*}
$$

### 2.3. 联合概率密度函数的分解

**联合概率密度函数是可分解的，可以表示为分解出的边缘密度函数和条件密度函数的乘积** 。

假设给定一个联合概率密度函数： $p(x,y)$ ，通过将 $y$ 这个维度的整个取值范围积分起来可以得到 **边缘密度函数（Marginal Density Function）** $p(x)$ ：

$$ p(x) = \int p(x,y) \mathrm{d}y $$

$p(x)$ 可以看作是单独的 $x$ 的 PDF ，更准确的说，它是特定 $x$ 在所有可能的 $y$ 值上的平均 PDF 。

那么，对于给定某个特定 $x$ 的情况下， $y$ 的 PDF 可以表示为：

$$ p(y|x) = \frac{p(x,y)}{p(x)} $$

上式也被称为是， **在给定 $x$ 的情况下， $y$ 的条件密度函数（Conditional Density Function）** 。

### 2.4. 从联合概率密度函数中进行二维采样的基本思路

首先计算边缘密度函数以隔离一个特定的变量，然后从该边缘密度函数中抽取样本，一旦抽取了该样本，那就可以计算给定该值的情况下的另一个变量的条件密度函数，并再次从该条件密度函数中抽取另外一个样本。

## 3. 一些半球上采样方案

### 3.1. 均匀采样半球（Uniformly Sampling a Hemisphere）

对于**均匀采样， 立体角上的 PDF 应该是一个常数，并且在定义的积分域积分的结果为 1** 。也就是说：

$$ \int_{H^2} p(\omega) \mathrm{d}\omega = 1 $$

因为 PDF 是一个常数，所以可以从积分中提出来，也就是：

$$ p(\omega) \int_{H^2} \mathrm{d}\omega = 1 $$

对于整个半球的立体角的积分，结果是 $2\pi$ ，所以可以得到：

$$ p(\omega) 2\pi = 1 $$

也就是说，对于均匀采样半球，立体角上 PDF 为：

$$ p(\omega) = \frac{1}{2\pi} $$

那么相对于 $\theta$ 和 $\phi$ 的联合密度函数可以表示为：

$$ p(\theta,\phi) = \sin\theta p(\omega) = \frac{\sin\theta}{2\pi} $$

下面要做的，就是将这个联合 PDF 分解。首先，通过对 $\phi$ 的整个取值范围进行积分来获得边缘密度函数 $p(\theta)$ ：

$$
p(\theta) = \int_{0}^{2\pi} p(\theta, \phi) \mathrm{d}\phi = \int_{0}^{2\pi} \frac{\sin\theta}{2\pi} \mathrm{d}\phi = \sin\theta
$$

那么，在给定 $\theta$ 的情况下， $\phi$ 的条件密度函数可以表示为：

$$
p(\phi | \theta) = \frac{p(\theta,\phi)}{p(\theta)} = \frac{\frac{\sin\theta}{2\pi}}{\sin\theta} = \frac{1}{2\pi}
$$

首先对 $p(\theta)$ 进行积分来得到 $cdf(\theta)$ ：

$$
cdf(\theta) = \int_{0}^{\theta} p(\theta) \mathrm{d}\theta = \int_{0}^{\theta} \sin\theta \mathrm{d}\theta
$$

因为，积分 $\int \sin\theta \mathrm{d}\theta$ 的不定积分是：

$$
\int \sin\theta \mathrm{d}\theta = -\cos\theta + C
$$

其中 $C$ 是积分常数。

所以，积分 $cdf(\theta)$ 可以计算为：

$$
\begin{align*}
cdf(\theta) &= \int_{0}^{\theta} \sin\theta \mathrm{d}\theta \\
&= \left[ -\cos\theta \right]_{0}^{\theta} \\
&= 1 - \cos\theta
\end{align*}
$$

然后再对 $p(\phi \| \theta)$ 积分得到 $cdf(\phi \| \theta)$ ：

$$
cdf(\phi | \theta) = \int_{0}^{\phi} p(\phi | \theta) \mathrm{d}\phi = \int_{0}^{\phi} \frac{1}{2\pi} \mathrm{d}\phi = \frac{\phi}{2\pi}
$$

最后，通过将这些 CDF 设为等于一个在 $[0,1]$ 之间的随机数 $\epsilon$ ，我们就可以找到用于单位半球上采样样本的角度 $\theta$ 和 $\phi$ ：

$$
\begin{align*}
cdf(\theta) &= 1 - \cos\theta = \epsilon_{0} \\
\implies \theta &= \cos^{-1}(1 - \epsilon_{0}) \\
cdf(\phi | \theta) &= \frac{\phi}{2\pi} = \epsilon_{1} \\
\implies \phi &= 2\pi \epsilon_{1}
\end{align*}
$$

因为 $\epsilon$ 是一个介于 0 到 1 之间的随机数字，这些方程可以简化为：

$$
\left\{
\begin{array}{ll}
\theta = \cos^{-1}(\epsilon_{0}) \\
\phi = 2\pi \epsilon_{1}
\end{array}
\right.
$$

### 3.2. 余弦加权半球采样（Cosine-Weighted Hemisphere Sampling）

所谓的余弦加权半球采样，描述的是立体角上的 PDF 应与极角 $\theta$ 的余弦成正比关系，也就是立体角的极角 $\theta$ 的余弦越大，随机生成此立体角用于采样的概率就越大。

首先还是来计算立体角上的 PDF ：

$$
\begin{align*}
\int_{H^2} p(\omega) \mathrm{d}\omega &= 1 \\
\int_{0}^{2\pi} \int_{0}^{\pi / 2} c \cos\theta \sin\theta \mathrm{d}\theta \mathrm{d}\phi &= 1 \\
c 2\pi \int_{0}^{\pi / 2} \cos\theta \sin\theta \mathrm{d}\theta &= 1 \\
c &= \frac{1}{\pi} \\
p(\omega) &= \frac{\cos\theta}{\pi}
\end{align*}
$$

那么相对于 $\theta$ 和 $\phi$ 的联合 PDF 可以表示为：

$$
p(\theta, \phi) = \sin\theta p(\omega) = \frac{1}{\pi} \sin\theta \cos\theta
$$

接下来，将联合 PDF 分解。首先，通过对 $\phi$ 的整个取值范围进行积分来获得边缘密度函数 $p(\theta)$ ：

$$
p(\theta) = \int_{0}^{2\pi} p(\theta, \phi) \mathrm{d}\phi = \int_{0}^{2\pi} \frac{1}{\pi} \cos\theta \sin\theta \mathrm{d}\phi = \frac{2\pi}{\pi} \cos\theta \sin\theta = 2 \sin\theta \cos\theta
$$

那么，在给定 $\theta$ 的情况下， $\phi$ 的条件概率密度函数可以表示为：

$$
p(\phi | \theta) = \frac{p(\theta,\phi)}{p(\theta)} = \frac{\frac{1}{\pi} \sin\theta \cos\theta}{2 \sin\theta \cos\theta} = \frac{1}{2\pi}
$$

首先对 $p(\theta)$ 进行积分来得到 $cdf(\theta)$ ：

$$
cdf(\theta) = \int_{0}^{\theta} p(\theta) \mathrm{d}\theta = \int_{0}^{\theta} 2 \sin\theta \cos\theta \mathrm{d}\theta
$$

因为三角恒等式 $\sin(2\theta) = 2 \sin\theta \cos\theta$ ，所以积分可以简化为：

$$
cdf(\theta) = \int_{0}^{\theta} \sin(2\theta) \mathrm{d}\theta
$$

因为，积分 $\int \sin(2\theta) \mathrm{d}\theta$ 的不定积分是：

$$
-\frac{1}{2} \cos(2\theta) + C
$$

其中 $C$ 是积分常数。

所以，积分 $cdf(\theta)$ 可以计算为：

$$
\begin{align*}
cdf(\theta) &= \int_{0}^{\theta} \sin(2\theta) \mathrm{d}\theta \\
&= \left[-\frac{1}{2} \cos(2\theta)\right]_{0}^{\theta} \\
&= -\frac{1}{2} \cos(2\theta) - \left(-\frac{1}{2} \cos(0)\right) \\
&= \frac{1}{2} -\frac{1}{2} \cos(2\theta)
\end{align*}
$$

又因为三角恒等式 $\cos(2\theta) = 1 - 2\sin^2(\theta)$ ，所以：

$$
cdf(\theta) = \frac{1}{2} -\frac{1}{2} (1 - 2\sin^2 \theta) = \sin^2 \theta
$$

然后再对 $p(\phi \| \theta)$ 积分得到 $cdf(\phi \| \theta)$ ：

$$
cdf(\phi | \theta) = \int_{0}^{\phi} p(\phi | \theta) \mathrm{d}\phi  = \int_{0}^{\phi}\frac{1}{2\pi} \mathrm{d}\phi = \frac{\phi}{2\pi}
$$

最后，通过将这些 CDF 设为等于一个在 $[0,1]$ 之间的随机数 $\epsilon$ ，我们就可以找到用于单位半球上采样样本的角度 $\theta$ 和 $\phi$ ：

$$
\begin{align*}
cdf(\theta) &= \sin^2 \theta = \epsilon_{0} \\
\implies \theta &= \sin^{-1}(\sqrt{\epsilon_{0}}) = \cos^{-1}(\sqrt{1-\varepsilon_{0}}) \\
cdf(\phi | \theta) &= \frac{\phi}{2\pi} = \epsilon_{1} \\
\implies \phi &= 2\pi \epsilon_{1}
\end{align*}
$$

同样的，因为 $\epsilon$ 是一个介于 0 到 1 之间的随机数字，简化为：

$$
\left\{
\begin{array}{ll}
\theta = \cos^{-1}(\sqrt{\epsilon_{0}}) \\
\phi = 2\pi \epsilon_{1}
\end{array}
\right.
$$

### 3.3. 微表面 BRDF 半球采样（Microfacet BRDF Hemisphere Sampling）

微表面 BRDF 是 PBR 的基础，它的基本表达式是：

$$
f_r(\omega_i, \omega_o, p) = \frac{F(\omega_i, h)G(\omega_i, \omega_o, h)D(h)}{4\cos(\theta_i)\cos(\theta_o)}
$$

其中 $F(\omega_i, h)$ 是菲涅尔项（Fresnel Term），它描述了光线在物体表面反射的强度变化，反射的强度取决于入射方向 $\omega_i$ 与物体材质的折射率； $G(\omega_i, \omega_o, h)$ 是几何项（G Term），它描述了微平面自阴影的属性，具体来说就是具有半程向量法线 $h$ 的微平面中，能同时被入射方向 $\omega_i$ 和反射方向 $\omega_o$ 可见的比例；最后一项 $D(h)$ 是法线分布函数（Normal Distribution Function, NDF），它描述了在所有微平面中，具有半程向量法线 $h$ 的微表面法线的分布情况。法线分布函数 NDF 通常是占主导地位的，它决定了反射波瓣的形状和强度。

为了对微表面 BRDF 模型进行采样，通常的步骤是：

- 首先对 NDF 进行采样，获取一个符合 NDF 的随机微表面法线方向
- 然后根据生成出的微表面法线方向与反射方向（也就是观察方向）计算出入射方向
- 最后将微表面法线方向、反射方向和入射方向代入微表面 BRDF 中计算反射

NDF 的类型有很多种，有各向同性的也有各向异性的，常见的各向同性的 NDF 有： GGX 、 Beckmann 、 Blinn 等。

需要注意的是， NDF 必须满足以下方程来计算在立体角上的 PDF ，其中 $h$ 表示的是半程向量（ `normalize(L + V)` ）， $\theta$ 表示的是宏观表面法线 $n$ 与半程向量 $h$ 之间的夹角：

$$
\int_{H^2} D(h) \cos\theta \mathrm{d}\omega = 1
$$

#### 3.3.1. 基于 GGX 法线分布的半球采样

首先， GGX 的法线分布函数是：

$$
D(h) = \frac{a^2}{\pi((a^2 - 1)\cos^{2}\theta + 1)^2}
$$

其中， $a$ 表示粗糙度， $\theta$ 是半程向量 $h$ 与宏观表面法线 $n$ 之间的夹角。

那么立体角上的 PDF 可以表示为：

$$
\begin{align*}
\int_{H^2} \frac{a^2}{\pi((a^2 - 1)\cos^{2}\theta + 1)^2} \cos\theta \mathrm{d}\omega = 1 \\
p_h(\omega) = \frac{a^2 \cos\theta}{\pi((a^2 - 1)\cos^{2}\theta + 1)^2}
\end{align*}
$$

相对于 $\theta$ 和 $\phi$ 的联合 PDF 可以表示为：

$$
p_h(\theta, \phi) = \frac{a^2 \cos\theta \sin\theta}{\pi((a^2 - 1)\cos^{2}\theta + 1)^2}
$$

接下来，将联合 PDF 分解。首先，通过对 $\phi$ 的整个取值范围进行积分来获得边缘密度函数 $p(\theta)$ ：

$$
p_h(\theta) = \int_{0}^{2\pi} p_h(\theta, \phi) \mathrm{d}\phi = \int_{0}^{2\pi} \frac{a^2 \cos\theta \sin\theta}{\pi((a^2 - 1)\cos^{2}\theta + 1)^2} \mathrm{d}\phi = \frac{2 a^2 \cos\theta \sin\theta}{((a^2 - 1)\cos^{2}\theta + 1)^2}
$$

那么，在给定 $\theta$ 的情况下， $\phi$ 的条件概率密度函数可以表示为：

$$
p_h(\phi | \theta) = \frac{p(\theta,\phi)}{p(\theta)} = \frac{\frac{a^2 \cos\theta \sin\theta}{\pi((a^2 - 1)\cos^{2}\theta + 1)^2}}{\frac{2 a^2 \cos\theta \sin\theta}{((a^2 - 1)\cos^{2}\theta + 1)^2}} = \frac{1}{2 \pi}
$$

首先对 $p_h(\theta)$ 进行积分来得到 $cdf_h(\theta)$ ：

$$
\begin{align*}
cdf_h(\theta) &= \int_{0}^{\theta} \frac{2 a^2 \cos(t) \sin(t)}{(\cos^{2}t (a^2 - 1) + 1)^2} \mathrm{d}t \\
&= \int_{\theta}^{0} \frac{a^2}{(\cos^{2}t (a^2 - 1) + 1)^2} \mathrm{d}(\cos^{2}t) \\
&= \frac{a^2}{a^2 - 1} \int_{0}^{\theta} \mathrm{d}(\frac{1}{\cos^{2}t(a^2 - 1) + 1}) \\
&= \frac{a^2}{a^2 - 1} \big(\frac{1}{\cos^{2}\theta(a^2 - 1) + 1} - \frac{1}{a^2} \big) \\
&= \frac{a^2}{\cos^{2}\theta (a^2 - 1)^2 + (a^2 - 1)} - \frac{1}{a^2 - 1}
\end{align*}
$$

然后再对 $p_h(\phi \| \theta)$ 积分得到 $cdf_h(\phi \| \theta)$ ：

$$
cdf_h(\phi | \theta) = \int_{0}^{\phi} p_h(\phi | \theta) \mathrm{d}t  = \int_{0}^{\phi} \frac{1}{2 \pi} \mathrm{d}t = \frac{\phi}{2\pi}
$$

最后，通过将这些 CDF 设为等于一个在 $[0,1]$ 之间的随机数 $\epsilon$ ，我们就可以找到用于单位半球上采样样本的角度 $\theta$ 和 $\phi$ ：

$$
\begin{align*}
cdf_h(\theta) &= \frac{a^2}{\cos^{2}\theta (a^2 - 1)^2 + (a^2 - 1)} - \frac{1}{a^2 - 1} = \epsilon_{0} \\
\implies \theta &= \arctan(a \sqrt{\frac{\epsilon_{0}}{1 - \epsilon_{0}}}) \\
cdf_h(\phi | \theta) &= \frac{\phi}{2\pi} = \epsilon_{1} \\
\implies \phi &= 2\pi \epsilon_{1}
\end{align*}
$$

最终表达式为：

$$
\left\{
\begin{array}{ll}
\theta = \arctan(a \sqrt{\frac{\epsilon_{0}}{1 - \epsilon_{0}}}) \\
\phi = 2\pi \epsilon_{1}
\end{array}
\right.
$$

## 4. 参考

- [1] [2D Sampling with Multidimensional Transformations](https://www.pbr-book.org/3ed-2018/Monte_Carlo_Integration/2D_Sampling_with_Multidimensional_Transformations)
- [2] [Sampling the hemisphere](https://ameye.dev/notes/sampling-the-hemisphere/#)
- [3] [Sampling Microfacet BRDF](https://agraphicsguynotes.com/posts/sample_microfacet_brdf/)
- [4] [Notes on importance sampling](https://www.blog.tobias-franke.eu/2014/03/30/notes_on_importance_sampling.html)
