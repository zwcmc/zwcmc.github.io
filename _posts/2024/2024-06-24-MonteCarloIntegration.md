---
layout: post
title:  "蒙特卡罗积分"
date:   2024-06-24 16:16:00 +0800
categories:
---

**蒙特卡罗积分（Monte Carlo Integration）**，是指采用 **蒙特卡罗法（Monte Carlo Methods）** 来估计积分。蒙特卡罗法是一类算法的统称，估计积分只是其中的一个应用，而积分计算在图形渲染中起到非常重要的作用。

为了方便理解，本篇笔记会统一符号表示，被积函数采用 $f(x)$ 小写形式，被积函数的积分采用 $F(x)$ 大写形式，概率密度函数采用 $pdf(x)$ ，累积概率分布函数采用 $cdf(x)$ ，大写的 $X$ ， $Y$ 等表示随机变量，小写的 $x$ ， $y$ 等表示具体的实数， $n$ 表示样本数， $N$ 表示维度。

## 蒙特卡罗法用到的数学理论知识

自然界中，有一类现象在一定条件下必然会发生，例如向上抛一石子、同性电荷互相排斥等，这类现象称为**确定性现象**。还有一类现象在大量重复试验下，呈现出某种规律，例如多次重复抛一枚硬币得到正面朝上大致有一半，等等，这种大量重复试验中所呈现出的固有规律性，就是我们所说的**统计规律性**。这种在个别试验中结果呈现不确定性，在大量重复试验中结果有统计规律性的现象，就称之为**随机现象**。

在随机试验中，尽管在每次试验之前不能预知结果，但试验的所有可能结果组成的集合是已知的，我们将随机试验的所有可能结果组成的集合称为**样本空间（Sample Space）**，样本空间通常用 $S$ 来表示，例如在投骰子的情况下，样本空间被定义为 $S = \{1,2,3,4,5,6\}$ 。样本空间的元素称为**样本点**。

一般，我们称试验的样本空间的任何子集都是一个**事件（Event）**，例如规定某种灯泡的寿命小于500小时的为次品，那么满足这一条件的样本点组成的子集，就称为事件。**当且仅当**这一子集中的一个样本点出现时，称为**事件发生**。

在相同的条件下，进行了 $n$ 次试验，在这 $n$ 次试验中，事件A发生的次数 $n_A$ 称为事件A发生的**频数**，比值 $\frac{n_A}{n}$ 称为事件A发生的**频率**。

频数和频率是通过试验观察得到的数据，当重复试验的次数n逐渐增大时，频率呈现出稳定性，逐渐稳定于某个常数，这种“频率稳定性”即通常所说的统计规律性，用来表征事件A发生可能性的大小，这就是所谓的**概率**，通常用 $P$ 来表示。概率的准确定义是：设E是随机试验， $S$ 是它的样本空间，对于E的每一事件A赋予一个实数，记为 $P(A)$ ，称为事件A的概率，它是表征事件发生可能性大小。

以抛掷硬币为例，它的样本空间是 {正面，反面}，这样就比较难记录和研究。所以引入一个法则，**将随机试验的每一个结果与一个实数对应起来，这就有了随机变量的概念**。设随机试验的样本空间为 $S={e}$ ， $X=X(e)$ 是定义在样本空间 $S$ 上的实数单值函数，就称 $X$ 为**随机变量**。就如抛硬币试验为例，它的随机变量可以定义为， $X(正面)=0$ ， $X(反面)=1$ ，那么样本空间就可以表示为 $S = \{0, 1\}$ 。这里的的0和1是随机变量 $X$ 映射出来的值，**也就是说随机变量 $X$ 本质是函数**。

随机变量有两种类型：**离散型（Discrete）**和**连续型（Continuous）**。当随机变量是离散型时，随机过程的结果只能取一列确切的值。掷骰子、抛硬币都是只能用离散随机变量来衡量的试验的例子，因为它们产生**有限**或**可数无限的多个值**（有限和可数无限之间的区别很重要，因为尽管例如计算样本空间中的元素数量可能永远不会结束，但该样本空间中的每个元素却是可数的并且可以与一个实数关联）。温度测量可以被视为连续随机变量的一个例子，因为测量温度的试验可以取得的可能值落在一个连续区间范围内，无法列举所有的可能性。

设 $X$ 是一个随机变量， $x$ 是任意实数，函数 $cdf(x) = P{(X \leq x)}, -\infty < x < +\infty$ 称为 $X$ 的**累积分布函数（Cumulative Distribution Function, CDF）**。

如果对于随机变量 $X$ 的累积分布函数 $cdf(x)$ ，存在非负数 $pdf(x)$ ，使对于任意实数 $x$ ，有

$$ cdf(x) = \int_{-\infty}^{x} pdf(t) dt $$

则称 $X$ 为**连续型随机变量**，其中函数 $pdf(x)$ 称为 $X$ 的**概率密度函数（Probability Distribution Function，PDF）**，概率密度函数 $pdf(x)$ 具有以下几个性质：

- $pdf(x) \geq 0$ ；
- $\int_{-\infty}^{+\infty} pdf(t) dt = 1$ ；
- 对于任意实数 $x_1$ ， $x_2$ ， $(x_1 \leq x_2)$ ，有 $P\{x_1 < X \leq x_2\} = \int_{x_1}^{x_2} pdf(t) dt$ ；
- 若 $pdf(x)$ 在点 $x$ 处连续，则有 $cdf'(x) = pdf(x)$ ；

有三种重要的连续型随机变量：

- 均匀分布；
- 指数分布；
- 正态分布；

暂时先介绍均匀分布。若连续型随机变量 $X$ 具有概率密度：

$$
pdf(x) =
\begin{cases}
    \frac{1}{b-a} & a < x < b \\[2ex]
    0 & \text{others}
\end{cases}
$$

则称 $X$ 在区间 $(a,b)$ 上服从**均匀分布**。

对于连续型随机变量 $(X,Y)$ ，设它的概率密度函数为 $pdf(x,y)$ ， $X$ 是一个连续型随机变脸，其概率密度函数表示为：

$$ pdf_X(x) = \int_{-\infty}^{+\infty} pdf(x,y) dy $$

类似的， $Y$ 也是一个连续型随机变量，其概率密度函数表示为：

$$ pdf_Y(y) = \int_{-\infty}^{+\infty} pdf(x,y) dx $$

分别称 $pdf_X(x)$ ， $pdf_Y(y)$ 为 $(X,Y)$ 关于 $X$ 与关于 $Y$ 的**边缘概率密度函数**。

设二维随机变量 $(X,Y)$ 的概率密度函数为 $pdf(x)$ ， $(X,Y)$ 关于 $Y$ 的边缘概率密度函数为 $pdf_Y(y)$ ， 若对于固定的 $y$ ， $pdf_Y(y) > 0$ ，则称 $\frac{pdf(x,y)}{pdf_Y(y)}$ 是在 $Y = y$ 的条件下的 $X$ 的**条件概率密度函数**，记为：

$$ pdf_{X|Y}(x|y) = \frac{pdf(x,y)}{pdf_Y(y)} $$

设连续型随机变量 $X$ 的概率密度函数为 $pdf(x)$ ，积分 $\int_{-\infty}^{+\infty} x \cdot pdf(x) dx$ 的值为随机变量 $X$ 的**期望值（Expected Value）**，记为 $E(X)$ 。期望值是随机试验下所有那些可能结果的平均值，它有几个重要的性质：

- 设 $C$ 是常数，则有 $E(C) = C$ ；
- 设 $X$ 是一个随机变量， $C$ 是常数，则有 $E(CX) = CE(X)$ ；
- 设 $X$ ， $Y$ 是两个随机变量，则有 $E(X + Y) = E(X) + E(Y)$ ；
- 设 $X$ ， $Y$ 是互相独立的随机变量，则有 $E(XY) = E(X)E(Y)$ ；

设 $X$ 是一个随机变量，若 $E\{[X - E(X)]^2\}$ 存在，则称它的值为 $X$ 的**方差**，记为 $D(X)$ ，即：

$$ D(X) = E\{[X - E(X)]^2\} $$

方差的平方根叫做**标准差**，记为 $\sigma(X)$ ， 即：

$$ \sigma(X) = \sqrt{D(X)} $$

方差表示的是随机变量与其均值的偏移程度，随机变量 $X$ 的方差可按下列公式计算：

$$ D(X) = E(X^2) - [E(X)]^2 $$

方差也有几个重要的性质：

- 设 $C$ 是常数，则有 $D(C) = 0$ ；
- 设 $X$ 是随机变量， $C$ 是常数，则有 $D(CX) = C^2 D(X), D(X+C) = D(X)$ ；
- 设 $X$ ， $Y$ 是两个随机变量，则有 $D(X+Y) = D(X) + D(Y) + 2E\{(X - E(X))(Y-E(Y))\}$ ；

当 $X$ ， $Y$ 互相独立，则 $D(X+Y) = D(X) + D(Y)$ 。

设随机变量 $X$ 具有数学期望 $E(X) = \mu$ ，方差 $D(X) = \sigma^2$ ，则对于任意正数 $\epsilon$ ，不等式

$$ P\{|X - \mu| \geq \epsilon \} \leq \frac{\sigma^2}{\epsilon^2} $$

成立。这一不等式称为**切比雪夫不等式（Chebyshev's inequality）**。

**辛钦大数定理**：设 $X_1$ ， $X_2$ ，... 是相互独立，服从统一分布随机变量序列，且具有期望值 $E(X_k) = \mu (k = 1,2,...)$ ，则对于任意 $\epsilon > 0$ ， 有：

$$ \lim_{n \to \infty} P\{|\frac{1}{n} \sum_{k=1}^{n} X_k - \mu| < \epsilon \} = 1 $$

**伯努利大数定理**：设 $f_A$ 是 $n$ 次独立重复试验中事件A发生的次数， $p$ 是事件A在每次试验中发生的概率，则对于任意正数 $\epsilon > 0$ ，有：

$$ \lim_{n \to \infty} P\{|\frac{f_A}{n} - p| < \epsilon \} = 1 $$

辛钦大数定理解释了：在大量重复试验下，样本的平均值约等于总体的平均值；伯努利大数定理解释了：在大量重复试验下，样本的频率收敛于其概率。

## 蒙特卡罗法

引用俄罗斯的数学家Sobol的一句话来描述蒙特卡罗法：

```txt
The Monte Carlo method is a numerical method of solving mathematical problems by random sampling (or by the simulation of random variables).
```

蒙特卡罗法是一类通过随机采样来求解问题的算法的统称，要求解的问题是某随机事件的概率或某随机变量的期望。通过随机抽样的方法，以随机事件出现的频率估计其概率，并将其作为问题的解。

![montecarlo2](/assets/images/2024/2024-09-24-MonteCarloIntegration/montecarlo2.png)

蒙特卡洛的基本做法是通过大量重复试验，通过统计频率，来估计概率，从而得到问题的求解。如上图所示，一个矩形内有一个不规则图案，要求解不规则图形的面积。显然，矩形的面积可以简单计算为 $A = ab \times ac$ ，点位于不规则形状内的概率为 $p = \frac{A_{shape}}{A}$ ，现在重复往矩形范围内随机的投射点，样本点有一定概率会落在不规则图形内，若复杂 $n$ 次试验，落到不规则图形内的次数为 $k$ ，频率为 $\frac{k}{n}$ ，若样本数量较大，则有：

$$ p = \frac{A_{shape}}{A} \approx \frac{k}{n} $$

根据伯努利大数定理得到的结论就是：随着样本数增加，频率 $\frac{k}{n}$ 会收敛于概率 $p$ ，使得该等式成立。

因此，我们可以估计出不规则图形的面积为 $A_{shape} = \frac{k}{n} A$ 。假设矩形面积为1，投射了1000次，有200个点位于不规则形状内，则可以推算出不规则图形的面积为0.2，投射的次数越大，计算出来的面积越精确。

这样的一个例子说明蒙特卡洛方法的基本思路，它并不是“缘分”求解法，而是有严格的数学基础作为依托，前面介绍的大数定理是它重要的理论基础。但是，蒙特卡洛方法的求解的结果是有误差的，重复的试验越多误差就会越低。

## 蒙特卡罗估计积分

举一个简单的例子，设一个函数 $f(x) = 3x^2$ ，计算其在区间 $[a,b]$ 上的积分值，如下图所示，容易得到：

$$ F(x) = \int_{a}^{b} f(x) = x^3 \Big|_{a}^{b} $$

![function_int](/assets/images/2024/2024-09-24-MonteCarloIntegration/function_int.jpeg)

假定要求的积分区间是 $[1,3]$ ， 那么积分结果为 $3^3 - 1^3 = 26$ 。

若采用蒙特卡罗方法来估计积分。先给出蒙特卡罗积分的一般等式，设 $X_1$ ， $X_2$ ，...， $X_n$ 是相互独立的样本且服从同一分布，概率密度函数为 $pdf(x)$ ，则使用蒙特卡罗法积分可以表示为：

$$ F_n(X) = \frac{1}{n}\sum_{k=1}^{n} \frac{f(X_k)}{pdf(X_k)}$$

回到上面那个例子，函数 $f(x)$ 在区间 $[a,b]$ 上均匀分布，则任意一个样本点的概率密度函数 $pdf(x) = \frac{1}{b - a}$ ，带入上面的等式可知：

$$ F_n(X) = \frac{b-a}{n} \sum_{k=1}^{n}f(X_k) $$

为了演示的目的，我们设计一个用不停止代码来对上面的函数进行蒙特卡罗积分，此代码每秒随机采样1000个均匀的采样点，并计算最终积分的结果。代码如下：

```c++
#include <random>
#include <time.h>
#include <iostream>
#include <iomanip>

#include <chrono>
#include <thread>

double f(float x)
{
    return 3.0 * x * x;
}

int main()
{
    std::cout << std::fixed << std::setprecision(12);

    int n = 0;
    float a = 1.0f, b = 3.0f;
    double sum = 0.0;
    std::default_random_engine seed(time(NULL));
    std::uniform_real_distribution<float> random(a, b);

    std::chrono::milliseconds interval(1);

    while (true)
    {
        auto start = std::chrono::steady_clock::now();

        n++;
        float sample = random(seed);
        sum += f(sample);

        if (n % 1000 == 0)
        {
            std::cout << "n = " << n  << ", and F_n(X) = " << (sum * (b - a) / n) << '\n';
        }

        auto end = std::chrono::steady_clock::now();
        std::chrono::duration<double> elapsed = end - start;

        std::chrono::milliseconds sleep_time = interval - std::chrono::duration_cast<std::chrono::milliseconds>(elapsed);

        if (sleep_time.count() > 0)
        {
            std::this_thread::sleep_for(sleep_time);
        }
    }

    return 0;
}
```

输出如下：

```bash
n = 1000, and F_n(X) = 26.710483721084
n = 2000, and F_n(X) = 26.685367324192
n = 3000, and F_n(X) = 26.461647337535
n = 4000, and F_n(X) = 26.266476991844
n = 5000, and F_n(X) = 26.099455179171
n = 6000, and F_n(X) = 26.056674834070
```

可以看到，随着采样点数量的增多，最终的结果越来越逼近积分的结果 26。

最后，我们来证明蒙特卡罗法的积分估计量的正确性：

$$ E[F_n(X)] $$

$$ = E\Big[\frac{1}{n} \sum_{k=1}^{n} \frac{f(X_k)}{pdf(X_k)}\Big] $$

$$ = \frac{1}{n} \sum_{k=1}^{n} \int \frac{f(x)}{pdf(x)} \cdot pdf(x) dx$$

$$ = \frac{1}{n} \sum_{k=1}^{n} \int f(x) dx $$

$$ = \int f(x) dx $$

蒙特卡洛法的积分估计量的数学期望等于被积函数的积分真值，证明 $F_n(X)$ 是**无偏估计量**。

估计量的方差，代表了它与被估真值之间的误差，设蒙特卡罗估计量的标准差为 $\sigma$ ，由于 $X_1$ ， $X_2$ ，... $X_n$ 是相互独立的样本且服从同一分布，则：

$$ \sigma^2[F_n(X)] $$

$$ = \sigma^2 \Big[ \frac{1}{n} \sum_{k=1}^{n} \frac{f(X_k)}{p(X_k)} \Big] $$

$$ = \frac{1}{n^2} \sum_{k=1}^{n} \int \Big(\frac{f(x)}{pdf(x)} - E(F_n(X))\Big)^2 \cdot pdf(x) dx $$

$$ = \frac{1}{n} \Big[ \int \Big( \frac{f(x)}{pdf(x)} \Big)^2 \cdot pdf(x) dx - E(F_n(X))^2 \Big]$$

$$ = \frac{1}{n} \Big[ \int \frac{f(x)^2}{pdf(x)} dx - E(F_n(X))^2 \Big] $$

从式子中可以知道，标准差与 $\frac{1}{\sqrt{n}}$ 正相关，也就是说，蒙特卡罗积分估计量的收敛与被积函数的维度等都无关，只跟样本数有关。

蒙特卡洛法进行积分估计，是非常简单的，无偏的。但是它的收敛速度比较慢，如何想要将误差降低1/2，需要提高4倍的采样数，不容易精确的计算方差，这就代表无法精确计算误差值。

前面介绍的切比雪夫不等、中心大数定理也保证了在大样本条件下，蒙特卡洛法估计出来的积分值是正确的，但是我们更希望在较小的采样数的条件也可以控制误差范围。方差表示随机变量与其均值的偏移程度，蒙特卡洛的估计值与积分真值存在差异，这个差异就是用方差表示，方差的大小也就代表了误差的大小，常提到的**方差缩减**，目的就是降低误差。方差缩减的研究课题是如何能做到不提高采样数，达到缩减误差的目的。后续要介绍的重要性采样和拟蒙特卡洛就是方差缩减中的两种策略。

## 生成符合指定分布的随机数

随机变量 $X$ 通常表示某种概率分布，随机数通常指生成某种概率分布的生成器，也就是随机变量 $X$ 的生成器。如何生成符合指定概率分布特点的随机数就是下面要介绍的内容。

设 $X$ 是一个随机变量，它的概率密度函数为 $pdf(x)$ ，它的累积分布函数可以表示为：

$$ cdf(x) = \int_{-\infty}^{+\infty} pdf(t) dt $$

那么，计算符合该概率分布的随机数方法如下：

1. 对概率密度函数 $pdf(x)$ ，计算它的分布函数 $cdf(x)$ ；
2. 计算 $cdf(x)$ 的反函数 $cdf^{-1}(x)$ ；
3. 对于一个均匀分布的随机数 $\xi$ ， 则 $X = cdf^{-1}(\xi)$ 就是符合该概率分布的随机数；

举个例子，已知区间 $[0,1]$ 之间的概率密度函数：

$$ pdf(x) = cx^n , x \in [0, 1] $$

其中 $c$ 和 $n$ 是一个常数，由于概率密度函数要求 $\int_{0}^{1} pdf(x) = 1$ ，则：

$$ \int_{0}^{1} cx^n = c\frac{x^{n+1}}{n+1} \Big|_{0}^{1} = \frac{c}{n+1} = 1 $$

得到 $c = n+1$ ，首先计算累积分布函数：

$$ cdf(x) = \int_{0}^{x} (n+1)t^n dt = x^{n+1} $$

其反函数为：

$$ cdf^{-1}(x) = \sqrt[n+1]{x} $$

则符合该概率分布的随机数为：

$$ X = \sqrt[n+1]{\xi} $$

### 分布变换

现在考虑一个随机变量的概率分布是从另外一个概率分布变换而来，这种情况怎么生成随机数呢？

设 $X$ 是随机变量，它的概率密度函数为 $pdf_X(x)$ 。 $Y$ 是另外一个随机变量，它的概率密度函数是 $pdf_Y(pdf_X(x))$ ，如何计算随机变量 $Y$ 的概率密度函数呢，设 $y = pdf_X(x)$ ，则 $Y$ 的密度函数为 $pdf_Y(x)$ 。

由概率密度函数的性质，容易得出结论 $x$ 与 $pdf_Y(y)$ 必须是一一对应的关系，易知：

$$ cdf_Y(y) = cdf_X(x) $$

两边分别求导，得：

$$ pdf_Y(y) \cdot \frac{dy}{dx} = pdf_X(x) $$

即有：

$$ pdf_Y(y) = pdf_X(x) \cdot \Big( \frac{dy}{dx} \Big)^{-1} $$

一般 $y$ 的导数都是正或者负的，则：

$$ pdf_Y(y) = pdf_X(x) \cdot \Big| \frac{dy}{dx} \Big|^{-1} $$

举个例子，已知区间 $[0,1]$ 之间的的概率密度函数为 $pdf_x(x) = 2x , Y = \sin{X}$ ，如何计算 $Y$ 的概率密度函数呢？

由于 $\frac{dy}{dx} = \cos{x}$ ，则：

$$ pdf_Y(y) = \frac{2x}{|\cos{x}|} = \frac{2\arcsin{y}}{\sqrt{1-y^2}} $$

可以扩展到多维的情况，设 $X_1, X_2, \cdots , X_n$ 是随机变量， $Y_k = T_k(X_k)$ 是变换函数，对应的随机变量是 $Y_1, Y_2, \cdots , Y_n$ ，可以构造一个**雅可比矩阵**：

$$ J_T = \begin{pmatrix}
    \partial T_1 / \partial x_1 & \cdots & \partial T_1 / \partial x_n \\
    \vdots & \ddots & \vdots \\
    \partial T_n / \partial x_1 & \cdots & \partial T_n / \partial x_n
\end{pmatrix} $$

那么可以得到：

$$
pdf_Y(y_1,y_2,\cdots,y_n) = \frac{pdf_X(x_1,x_2,\cdots,x_n)}{|J_T|} \tag{1}
$$

其中， $\|J_T\|$ 是矩阵 $J_T$ 的**行列式**。

以三维空间下的球坐标为例，如下图所示：

$$
\begin{cases}
    x = r \sin{\theta}\cos{\phi} \\
    y = r \sin{\theta}\sin{\phi} \\
    z = r \cos{\theta}
\end{cases}
$$

其中， $\theta \in [0, \pi], \phi \in [0,2\pi]$ 。

对应的雅可比矩阵为：

$$
J_T = \begin{pmatrix}
    \sin{\theta}\cos{\phi} & r\cos{\theta}\cos{\phi} & -r\sin{\theta}\sin{\phi} \\
    \sin{\theta}\sin{\phi} & r\cos{\theta}\sin{\phi} & r\sin{\theta}\cos{\phi} \\
    \cos{\theta} & -r\sin{\theta} & 0
\end{pmatrix}
$$

$J_T$ 矩阵的行列式 $\|J_T\| = r^2 \sin{\theta}$ 。代入等式(1)，得：

$$ pdf(r,\theta,\phi) = r^2 \sin{\theta} \cdot pdf(x,y,z) $$

现在考虑半球积分的采样，设半球的概率密度函数为 $pdf(\omega) = c$ ，其中 $\omega$ 是立体角，球坐标与立体角对应的关系有：

$$ \int_{\Omega^2}p(\omega)d\omega = \int_{\Omega^2}\sin{\theta}d\phi d\theta \tag{2} $$

则有

$$ \int_{\Omega^2}p(\omega)d\omega = c \int_{0}^{\frac{\pi}{2}} \sin{\theta} \int_{0}^{2\pi} d\phi d\theta = 2\pi c = 1 $$

则：

$$ c = \frac{1}{2\pi} $$

那么：

$$ pdf(\theta, \phi) = \frac{\sin{\theta}}{2\pi} $$

可以计算出：

$$ pdf(\theta) = \int_{0}^{2\pi} p(\theta,\phi)d\phi = \sin{\theta} $$

再根据条件概率的公式，可知：

$$ pdf(\phi | \theta) = \frac{pdf(\theta,\phi)}{pdf(\theta)} = \frac{1}{2\pi} $$

分别计算累积分布函数：

$$ cdf(\theta) = \int_{0}^{\theta} \sin{t} dt = 1 - \cos{\theta} $$

$$ cdf(\phi | \theta) = \int_{0}^{\phi} \frac{1}{2\pi} dt = \frac{\phi}{2\pi} $$

根据前面介绍的随机数生成法，设 $\xi_1, \xi_2$ 是 $[0, 1]$ 之间均匀分布的随机数，可以用 $1-\xi_1$ 替换 $\xi_1$ ，可以求出：

$$ \theta = \cos^{-1}{\xi_1} $$

$$ \phi = 2\pi \xi_2 $$

将球坐标角度转换为笛卡尔坐标，可以得到：

$$ x = \sin{\theta}\cos{\phi} = \cos{(2\pi \xi_2)} \sqrt{1 - \xi_1^2} $$

$$ y = \sin{\theta}\sin{\phi} = \sin{(2\pi \xi_2)} \sqrt{1 - \xi_1^2} $$

$$ z = \cos{\theta} = \xi_1 $$

UE4中的算法实现为：

```c++
float4 UniformSampleHemisphere( float2 E )
{
    float Phi = 2 * PI * E.x;
    float CosTheta = E.y;
    float SinTheta = sqrt( 1 - CosTheta * CosTheta );

    float3 H;
    H.x = SinTheta * cos( Phi );
    H.y = SinTheta * sin( Phi );
    H.z = CosTheta;

    float PDF = 1.0 / (2 * PI);

    return float4( H, PDF );
}
```

还有一种是 $\cos$ 加权的半球采样，有：

$$ pdf(\omega) \propto \cos{\theta} $$

由于概率密度函数在区间上的积分值为 1，可以得到 $c = \frac{1}{\pi}$ ，所以：

$$ pdf(\theta, \phi) = \frac{1}{\pi} \cos{\theta} \sin{\theta} $$

相似的方法，可以计算出：

$$ \theta = \cos^{-1}{\sqrt{\xi_1}} $$

$$ \phi = 2\pi \xi_2 $$

将球坐标角度转换为笛卡尔坐标，可以得到：

$$ x = \sin{\theta}\cos{\phi} = \cos{(2\pi \xi_2)} \sqrt{1 - \xi_1} $$

$$ y = \sin{\theta}\sin{\phi} = \sin{(2\pi \xi_2)} \sqrt{1 - \xi_1} $$

$$ z = \cos{\theta} = \sqrt{\xi_1} $$

UE4中的算法实现为：

```c++
float4 CosineSampleHemisphere( float2 E )
{
    float Phi = 2 * PI * E.x;
    float CosTheta = sqrt( E.y );
    float SinTheta = sqrt( 1 - CosTheta * CosTheta );

    float3 H;
    H.x = SinTheta * cos( Phi );
    H.y = SinTheta * sin( Phi );
    H.z = CosTheta;

    float PDF = CosTheta * (1.0 /  PI);

    return float4( H, PDF );
}
```

## 重要性采样

**重要性采样（Importance Sampling）** 是已知被积函数的一些分布信息而采用的一种缩减方差的策略，还有别的策略像 **俄罗斯轮盘切割（Russian Roulette and Splitting）** ， **分层采样（Stratified Sampling）** ， **拉丁超立方体采样（Latin Hypercube Sampling）** 等，都是通过控制采样的策略达到缩减方差的目的。

考虑一个简单的例子，设一个函数为：

$$
f(x)=\begin{cases}
    99.01 & x \in [0, 0.01) \\
    0.01 & x \in [0.01, 1]
\end{cases}
$$

它的几何示意图如下图所示：

![00_mc](/assets/images/2024/2024-09-24-MonteCarloIntegration/00_mc.jpeg)

我们采用蒙特卡洛法估计它的积分，选择区间 $[0, 1]$ 之间的均匀分布作为它的随机数，那么存在绝大部分的采样点在区间 $[0.01, 1]$ 之间，但是它对积分估计的贡献只有0.01，小部分采样点在区间 $[0, 0.01)$ 之间，它对积分估计的贡献却非常的大，这种现象就会导致误差非常的大，简单提高采样数对估计量收敛的影响较小。

蒙特卡洛法的核心，是根据某一概率分布来随机采样，那么，**根据被积函数曲线规律来设计类似的概率密度来进行采样，这就符合蒙特卡洛积分的要求，也就是重要性采样的核心**。

考虑另外一个例子，需要计算函数 $f(x) = \sin{x}, x \in [0, \frac{\pi}{2}]$ 的积分。首先，很容易计算出它的积分真值为：

$$ F(x) = \int_{0}^{\frac{\pi}{2}} f(x) dx = -\cos{x} \Big|_{0}^{\frac{\pi}{2}} = 1 $$

我们设计两种概率密度函数，一种是均匀分布，随机变量 $X_1$ 的概率密度函数为：

$$ pdf_1(x) = \frac{1}{\frac{\pi}{2}}, x \in [0, \frac{\pi}{2}] $$

另外一种，随机变量 $X_2$ 概率密度函数为：

$$ pdf_2(x) = \frac{8x}{\pi^2}, x \in [0, \frac{\pi}{2}] $$

两种概率分布与函数的几何示意图如下图所示：

![01_mc](/assets/images/2024/2024-09-24-MonteCarloIntegration/01_mc.jpeg)

显然 $pdf_2(x)$ 更符合被积函数的曲线特点。

接下来我们来验证这两种概率分布估计出的积分值与真值间的差异，设区间 $[0,1]$ 间均匀分布的随机数 $\xi$ ，根据前面的随机数计算方法，则容易算出两种概率分布的随机数， $\xi_1$ 对应 $pdf_1(x)$ ， $\xi_2$ 对应 $pdf_2(x)$ ：

$$ \xi_1 = \frac{\pi}{2} \xi $$

$$ \xi_2 = \frac{\pi}{2} \sqrt{\xi} $$

C++的实现代码如下：

```c++
#include <random>
#include <time.h>
#include <cmath>
#include <iostream>

#define PI 3.14159265358979323846

double random_generator1(double x){
    return x * PI / 2.0;
}

double random_generator2(double x) {
    return sqrt(x) * PI / 2.0;
}

double pdf1(double x) {
    return 2.0 / PI;
}

double pdf2(double x) {
    return 8.0 * x / (PI * PI);
}

double f(double x) {
    return sin(x);
}

int main() {
    const int total_times = 16;
    int run_times = 1;
    double real = 1.0;
    std::default_random_engine seed(time(NULL));
    printf("No     Uniform     Importance   Err.Uniform(%%) Err.Importance(%%)\n");
    while ((run_times++) < total_times) {
        int num_samples = 16;
        double sum1 = 0.0, sum2 = 0.0;
        std::uniform_real_distribution<double> random(0, 1);

        for (int i = 0; i < num_samples; i++) {
            double sample = random(seed);
            double random1 = random_generator1(sample);
            double random2 = random_generator2(sample);
            double p1 = pdf1(random1);
            double p2 = pdf2(random2);

            sum1 += f(random1) / p1;
            sum2 += f(random2) / p2;
        }
        sum1 /= num_samples;
        sum2 /= num_samples;
        printf("%-6d %-12.4f %-14.4f %-14.1f %-12.1f\n", run_times, sum1, sum2, 100.0 * (sum1 - real) / real, 100.0 * (sum2 - real) / real);
    }
    return 0;
}
```

输出结果：

```bash
No     Uniform     Importance   Err.Uniform(%) Err.Importance(%)
2      0.7643       1.0667         -23.6          6.7
3      0.9643       1.0050         -3.6           0.5
4      1.0448       0.9916         4.5            -0.8
5      0.9986       1.0063         -0.1           0.6
6      0.9768       1.0010         -2.3           0.1
7      1.1330       0.9482         13.3           -5.2
8      1.0037       0.9933         0.4            -0.7
9      0.8938       1.0231         -10.6          2.3
10     1.0168       0.9977         1.7            -0.2
11     1.0376       0.9949         3.8            -0.5
12     1.2211       0.9471         22.1           -5.3
13     1.0696       0.9796         7.0            -2.0
14     0.8819       1.0307         -11.8          3.1
15     0.9998       1.0024         -0.0           0.2
16     1.0309       0.9931         3.1            -0.7
```

可以看出，**在相同采样数下，重要性采样估计值比均匀采样的误差小**。

从理论的角度来说，任意一个被积函数 $f(x)$ ，它的最优概率密度函数是：

$$ pdf(x) = \frac{|f(x)|}{\int f(x) dx} $$

从这个结论来说，上述的例子中，最优的概率密度函数应该是：

$$ pdf(x) = \sin{x} \tag{3} $$

但是实际上我们无法得知被积函数的积分真值，好的概率密度函数应该是与函数本身的几何曲线形状相似。

最后，介绍下重要性采样实时渲染中的应用，这里不会过多的解释物理光照模型，只介绍其中的**GGX重要性采样**，GGX的法线分布函数，有很长的拖尾效果，得到广泛应用。

在实时渲染中，采用的物理光照模型是基于微表面理论，需要统计微面元的法线分布，如下图所示：

![02_mc](/assets/images/2024/2024-09-24-MonteCarloIntegration/02_mc.jpeg)

设 $D(m)$ 是微表面的法线分布函数，有下列等式成立：

$$ \int_{\Omega} D(m) (m \cdot n) d\omega = 1 \tag{4} $$

其中， $m$ 是微表面的法线， $n$ 是几何平面的发现。

法线分布函数GGX，如下所示：

$$ D(m) = \frac{a^2}{\pi (1 + (m \cdot n)^2 (a^2 - 1))^2} \tag{5} $$

其中， $a = \text{roughness}^2$ 。

将等式（2）和等式（5）带入等式等式（4），得：

$$ \int_{\Omega} D(m) (m \cdot n) d\omega = \int_{\Omega} \frac{a^2}{\pi (1 + (m \cdot n)^2 (a^2 - 1))^2} \cos{\theta} \sin{\theta} d\theta d\phi = 1 $$

其中， $\theta \in [0, \pi], \phi \in [0,2\pi]$ 。

由等式（3）可知，最优的概率密度函数是：

$$ pdf(\theta,\phi) = \frac{a^2}{\pi (1 + (m \cdot n)^2 (a^2 - 1))^2} \cos{\theta} \sin{\theta} $$

首先计算关于 $\theta$ 边缘概率密度函数 $pdf(\theta)$ ：

$$ pdf(\theta) = \int_{0}^{2\pi} pdf(\theta,\phi) d\phi = \frac{2a^2}{(1 + \cos^2{\theta}(a^2 - 1))^2} \cos{\theta} \sin{\theta} $$

再计算条件概率密度函数 $pdf(\phi \| \theta)$ ：

$$ pdf(\phi | \theta) = \frac{pdf(\theta, \phi)}{pdf(\theta)} = \frac{1}{2\pi} $$

计算等式的积分：

$$ cdf(x) = \int_{0}^{x} pdf(\theta) d\theta $$

$$ = -2a^2 \int_{1}^{\cos{x}} \frac{t}{(1 + t^2(a^2 - 1))^2} dt, t = \cos{\theta} $$

$$ = -a^2 \int_{1}^{\cos^2{x}} \frac{1}{(1 + y(a^2 - 1))^2} dy, y = t^2 $$

$$ = -\frac{a^2}{a^2 - 1} \int_{a^2}^{(a^2 - 1)\cos^2{x} + 1} \frac{1}{z^2} dz, z = (a^2 - 1)y + 1 $$

$$ = \frac{a^2}{a^2 - 1} \frac{1}{z} \Big|_{a^2}^{(a^2 - 1)\cos^2{x} + 1} $$

$$ = \frac{1 - \cos^2{x}}{(a^2 - 1) \cos^2{x} + 1} $$

则 $cdf^{-1}(x)$ 为：

$$ cdf^{-1}(x) = \cos^{-1} \sqrt{\frac{1-x}{(a^2 - 1)x + 1}} $$

设 $\xi_1, \xi_2$ 是区间 $[0,1]$ 中均匀分布的随机数，则：

$$ \phi = 2\pi \xi_1 $$

$$ \theta = \cos^{-1} \sqrt{\frac{1-\xi_2}{(a^2 - 1)\xi_2 + 1}} $$

UE4中GGX的重要性采样的代码如下所示，有没有豁然开朗的感觉：

```c++
float4 ImportanceSampleGGX( float2 E, float a2 )
{
    float Phi = 2 * PI * E.x;
    float CosTheta = sqrt( (1 - E.y) / ( 1 + (a2 - 1) * E.y ) );
    float SinTheta = sqrt( 1 - CosTheta * CosTheta );

    float3 H;
    H.x = SinTheta * cos( Phi );
    H.y = SinTheta * sin( Phi );
    H.z = CosTheta;

    float d = ( CosTheta * a2 - CosTheta ) * CosTheta + 1;
    float D = a2 / ( PI*d*d );
    float PDF = D * CosTheta;

    return float4( H, PDF );
}
```

## 拟蒙特卡洛

**拟蒙特卡洛（Quasi Monte Carlo）积分** 估计技术的核心点是积分估计时采用 **低差异序列（Low Discrepancy Sequence）** 来替换纯随机数，它的优点是积分估计的收敛速度更快，理论上拟蒙特卡洛能达到 $O(n^{-1})$ 的收敛速度，而普通蒙特卡洛的收敛速度是 $O(n^{-0.5})$ ，更快的收敛速度意味着相同的采样数下，低差异序列估计的结果误差更小。

我们先举一个简单的例子解释，随机采样很可能会导致采样点的 **聚集（Clump）** 现象，如下图所示，聚集会导致积分估计的误差扩大：

![03_mc](/assets/images/2024/2024-09-24-MonteCarloIntegration/03_mc.jpeg)

为了消除这种聚集现象，就有了 **分层采样（Stratified Sampling）** 的提出，它是将采样空间均分为 $n$ 个格子，每个格子里面再随机采样一个点，如下图所示：

![04_mc](/assets/images/2024/2024-09-24-MonteCarloIntegration/04_mc.jpeg)

那么，有没有一种采样方式，它不需要对采样空间进行手动分割，就同时具备空间平均性和随机性这两个特点呢？数学家引入了 **差异（Discrepancy）** 这个概念，完全随机的样本具有很高的差异，但是完全平均的样本的差异为0，我们的目标就是找到 **一组有较低差异且不失随机性的序列，这就是低差异序列** ，以期望达到消除聚集的目的。

下面给出差异的标准数学定义：设 $P = \{ x_1, x_2, \cdots ,x_n \}$ 是一组位于空间 $[0,1]^s$ 下的点集，差异可以定义为：

![05_mc](/assets/images/2024/2024-09-24-MonteCarloIntegration/05_mc.jpeg)

其中， $s$ 表示维度， $\lambda_s$ 是 $s$ 维空间的体积度量（非标准命名）， $A(B)$ 表示点集 $P$ 中落入区域 $B$ 的点的个数， $J$ 表示 $s$ 维空间下的任意盒状包围范围。举个例子解释下差异的定义，如下图所示，在二维空间下区间 $[0,1] x [0, 1]$ 内有一组采样点集，点集个数为 $n$ ，整个点集的面积（在二维空间下的体积度量就称为面积）就为 1， 任意划定一块区域 $B$ ，区域 $B$ 囊括了 $k$ 个采样点，此区域的面积是 $\lambda(B)$ ，那么差异指的就是点集个数比值 $k/n$ 与面积比值 $\lambda(B) / 1$ 的差的绝对值，注意B是指空间范围 $[0,1] x [0, 1]$ 内的任意一块区域：

![06_mc](/assets/images/2024/2024-09-24-MonteCarloIntegration/06_mc.jpeg)

常用到的几种低差异序列有：

- Van der Corput序列；
- Halton序列；
- Hammersley序列；
- Sobol序列；

下面介绍下 Van der Corput序列、Halton序列和Hammersley序列。

Van der Corput序列是数学家Van der Corput在1935年设计的一种一维低差异序列，也是最简单的一种序列。现实生活中的数字通常都是十进制，例如1314可以拆解为：

$$ 1 \times 10^3 + 3 \times 10^2 + 1 \times 10^1 + 1 \times 10^0 $$

由十进制扩展到 $b$ 进制，它的数学表示就是：

$$ x = \sum_{k=0}^{m} d_k \cdot b^k $$

其中， $b$ 是底数， $d_k \in [0, b-1]$ 。

它对应的Van der Corput序列就可以表示为：

$$ g_b(x) = \sum_{k=1}^{m} d_k \cdot b^{-k-1} $$

生成步骤可以描述为：

1. **表示为基 \( b \) 的数字**：将 \( n \) 表示为基 \( b \) 的数字。例如，基 \( b = 2 \) 时，数字 2 表示为 10。
2. **反转数字顺序**：将这些数字的顺序反转。例如，10 反转后得到 01。
3. **转换为小数**：将反转后的数字作为小数的个位、十位、百位等。例如，01 变为 0.01（二进制），即 $2^{-2} = 0.25$ （十进制）。

以十进制数来说，1到13的Van der Corput序列就是：

$$ \{ \frac{1}{10},\frac{2}{10},\frac{3}{10},\frac{4}{10},\frac{5}{10},\frac{6}{10},\frac{7}{10},\frac{8}{10},\frac{9}{10},\frac{1}{100},\frac{11}{100},\frac{21}{100},\frac{31}{100} \} $$

以十进制的 10 来举个例子：

- 基 \( b = 10 \) 时，数字 10 表示为 10 ；
- 反转数字顺序，得到 01 ；
- 转换为小数，0.01 ；
- 所以当基数为10时，十进制的10转换为Van der Corput序列就是 0.01 ；

C++算法的实现代码为：

```c++
double van_der_corput(int x, int base)
{
    double q = 0, bk = 1.0 / base;
    while (x > 0)
    {
        q += (x % base) * bk;
        x /= base;
        bk /= base;
    }
    return q;
}
```

Halton序列就是将Van der Corput序列扩展为N维的情况，可以表示为：

$$ halton(x) = (g_{b_1}(x),g_{b_2}(x), \cdots , g_{b_N}(x)) $$

其中,底数 $b_1,b_2, \cdots, b_N$ 互为质数。

Hammersley序列是在已知采样点集个数的情况下才可计算出的序列，设采样点集个数为n，维度为N，Hammersley序列表示为：

$$ Hammersley(x) = (\frac{x}{n},g_{b_1}(x),g_{b_2}(x), \cdots , g_{b_{N-1}}(x)) $$

Halton序列的收敛速度是 $O((\log{n})^N / n)$ ，而Hammersley序列的收敛速度是 $O((\log{n})^{N-1} / n)$ ，所以在已知样本个数的情况下，Hammersley序列会有更优的表现。

最后，实现Halton和Hammersley两种低差异序列，并生成采样分布的纹理，如下图所示：

![07_mc](/assets/images/2024/2024-09-24-MonteCarloIntegration/07_mc.jpeg)
![08_mc](/assets/images/2024/2024-09-24-MonteCarloIntegration/08_mc.jpeg)

```c++
#include <cstdlib> 
#include <cstdio> 
#include <cmath> 
#include <iostream> 
#include <fstream> 
#include <random>

#define Float float

const int Primes[] = {
    2, 3, 5, 7, 11,
    13, 17, 19, 23,
    29, 31, 37, 41,
    43, 47, 53, 59,
    61, 67, 71, 73 };

const int Width = 512;
const int Height = 512;
const int NumSamples = 200;
const int Strand = 3;

const int Offset[9][2] = {
    {0, 0},
    {0, 1},
    {0, -1},
    {1, 0},
    {-1, 0},
    {1, 1},
    {-1, -1},
    {1, -1},
    {-1, 1},
};

static Float RadicalInverse(int a, int base) {
    const Float invBase = (Float)1 / (Float)base;
    int reversedDigits = 0;
    Float invBaseN = 1;
    while (a) {
        int next = a / base;
        int digit = a - next * base;
        reversedDigits = reversedDigits * base + digit;
        invBaseN *= invBase;
        a = next;
    }
    return reversedDigits * invBaseN;
}

int GetNthPrime(int dimension){
    return Primes[dimension];
}

Float Halton(int dimension, int index){
    return RadicalInverse(index, GetNthPrime(dimension));
}

Float Hammersley(int dimension, int index, int numSamples){
    if (dimension == 0)
        return index / (Float)numSamples;
    else
        return RadicalInverse(index, GetNthPrime(dimension - 1));
}

void WritePPM(const char* filename, const unsigned char* data, int width, int height) {
    std::ofstream ofs;
    ofs.open(filename);
    ofs << "P5\n" << width << " " << height << "\n255\n";
    ofs.write((char*)data, width * height);
    ofs.close();
}

void SaveSample(unsigned char* pixels, int width, int height, Float x, Float y){
    int px = (int)(x * Width + 0.5);
    int py = (int)(y * Height + 0.5);
    for (int i = 0; i < 9; i++) {
        int col = std::max(std::min(px + Offset[i][0], Width - 1), 0);
        int row = std::max(std::min(py + Offset[i][1], Height - 1), 0);
        pixels[row * width + col] = 0;
    }
}

int main(){
    unsigned char pixels[Height][Width];
    // generate halton sequence.
    memset(pixels, 0xff, sizeof(pixels));
    for (int i = 0; i < NumSamples; i++)
    {
        Float x = Halton(0, i);
        Float y = Halton(1, i);
        SaveSample(&pixels[0][0], Width, Height, x, y);
    }
    WritePPM("halton.ppm", &pixels[0][0], Width, Height);

    // generate hammersley sequence.
    memset(pixels, 0xff, sizeof(pixels));
    for (int i = 0; i < NumSamples; i++)
    {
        Float x = Hammersley(0, i, NumSamples);
        Float y = Hammersley(1, i, NumSamples);
        SaveSample(&pixels[0][0], Width, Height, x, y);
    }
    WritePPM("hammersley.ppm", &pixels[0][0], Width, Height);
    return 0;
}
```
