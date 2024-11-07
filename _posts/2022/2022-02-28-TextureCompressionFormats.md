---
layout: post
title:  "ETC1 与 ETC2 纹理压缩"
date:   2022-02-28 02:03:00 PM +800
category: Others
---

- [1. 纹理压缩格式](#1-纹理压缩格式)
- [2. ETC1 Compressed Texture Image Formats](#2-etc1-compressed-texture-image-formats)
  - [2.1. ETC1S](#21-etc1s)
  - [2.2. ETC1总结](#22-etc1总结)
- [3. ETC2 Compressed Texture Image Formats](#3-etc2-compressed-texture-image-formats)
  - [3.1. Format RGB ETC2](#31-format-rgb-etc2)
  - [3.2. Format RGBA ETC2](#32-format-rgba-etc2)

## 1. 纹理压缩格式

[参考](https://www.khronos.org/registry/DataFormat/specs/1.3/dataformat.1.3.html#_introduction)

## 2. ETC1 Compressed Texture Image Formats

纹理压缩成一个个 4x4 的纹素块。如果一个纹理(或者mipmap的特定mip-level)的任意方向上的尺寸小于4(比如纹理为 2x2 或者 8x1 )，则纹理的数据被存储在 4x4 纹素块的左/上部分，纹素块的其他部分不使用没有数据。举个例子，一个 4x2 的纹理即被存储在 4x4 的纹素块的上半部分，纹素块的下半部分没有数据。

<span id="jumpFigure38">Pixel layout for an 8×8 texture using four ETC1 compressed blocks</span>

![Texture Compression Formats1](/assets/images/2022/2022-02-28-TextureCompressionFormats/Texture_Compression_Formats_1.png)

看上图，内存中第一个 4x4 纹素块的纹素$a_1$表示的是纹理坐标$(u=0, v=0)$。内存中第二个 4x4 纹素块中的$a_2$毗邻第一个纹素块中的$m_1$，循环往复直到纹理的宽度；第三个纹素块的$a_3$毗邻第一个纹素块的$d_1$，和$u$方向一样，循环往复知道纹理的高度。使用第一个、第二个、第三个和第四个 4x4 纹素块来对这个 8×8 纹理的数据存储（如果按该顺序存储在内存中），通过简单线性格式相同的顺序对纹素进行编码，按以下内存顺序排列：$a_1\ e_1\ i_1\ m_1\ a_2\ e_2\ i_2\ m_2\ b_1\ f_1\ i_1\ n_1\ b_2\ f_2\ i_2\ n_2\ c_1\ g_1\ k_1\ o_1\ c_2\ g_2\ k_2\ o_2\ d_1\ h_1\ l_1\ p_1\ d_2\ h_2\ l_2\ p_2\ a_3\ e_3\ i_3\ m_3\ a_4\ e_4\ i_4\ m_4\ b_3\ f_3\ i_3\ n_3\ b_4\ f_4\ i_4\ n_4\ c_3\ g_3\ k_3\ o_3\ c_4\ g_4\ k_4\ o_4\ d_3\ h_3\ l_3\ p_3\ d_4\ h_4\ l_4\ p_4$。

字节 Byte 和比特 bit 的换算关系是 1 Byte = 8 bit

一个 4x4 的纹素块通过64比特位(bit)来表示，这个纹素块的数据有8个字节(Byte)：$q_0\ q_1\ q_2\ q_3\ q_4\ q_5\ q_6\ q_7$，其中$q_0$是最低内存地址，$q_7$是最高位内存地址。纹素块的64比特位由以下64位整数表示：

![Texture Compression Formats2](/assets/images/2022/2022-02-28-TextureCompressionFormats/Texture_Compression_Formats_2.png)

每个 64 位字都包含有关 4×4 纹素块的信息，如图所示：

<span id="jumpFigure39">Figure 39. Pixel layout for an ETC1 compressed block</span>

![Texture Compression Formats3](/assets/images/2022/2022-02-28-TextureCompressionFormats/Texture_Compression_Formats_3.png)

ETC1中有2中模式：”individual模式“和”differential模式“，模式由**第33比特位的diff bit**来控制，diff bit=0时表示的是individual模式，diff bit=1时表示的是differential模式。这2种模式的比特位布局有一些不一样，individual模式下的位布局是下图种的a和c，differential模式下的位布局是下图种的b和c：

<span id="jumpTable125">Table 125.Texel Data format for ETC1 compressed textures</span>

![Texture Compression Formats4](/assets/images/2022/2022-02-28-TextureCompressionFormats/Texture_Compression_Formats_4.png)

不管在哪个模式下，一个 4x4 的纹素块都被分为2个子纹素块，子纹素块的为 2x4 或者 4x2。通过第32比特位的flip bit来决定是哪种尺寸，flip bit=0时则2个子纹素块为 2x4，如下图所示：

![Texture Compression Formats5](/assets/images/2022/2022-02-28-TextureCompressionFormats/Texture_Compression_Formats_5.png)

flip bit=1时则2个子纹素块为 4x2，如下图所示：

![Texture Compression Formats6](/assets/images/2022/2022-02-28-TextureCompressionFormats/Texture_Compression_Formats_6.png)

<span id="jumpETC1Basecolor">Decode base color for each subblock</span>

2种模式下，对于每一个子纹素块都会存储基色(base color)，但是不同模式下的存储方式不同：

在individual模式(diff bit = 0)下，第一个子纹素块1的基色R,G,B分别存储在4个比特位中，R(比特位 63-60)，G(比特位 55-52)，B(比特位 47-44)，如上图[Table 125](#jumpTable125)中(a)部分所示。通过把4高位复制到4低位把这4比特位的扩展到8位RGB888。举个例子，如果R = 14 = 1110b，G = 3 = 0011b，B = 8 = 1000b，则子纹素块1的基色R通道值为11101110b = 238，G通道值为00110011b = 51，B通道值为10001000b = 136。第二个子纹素块2的基色同理，使用的是$R_2$(比特位 59-56)，$G_2$(比特位 51-48)，$B_2$(比特位 43-40)。总之，individual模式下子纹素块的基色为：

$$
base color_{subblock1}=extend\_4to8bits(R,G,B)\\
base color_{subblock2}=extend\_4to8bits(R_2,G_2,B_2)
$$

在differential模式(diff bit = 1)下，第一个子纹素块1的基色R,G,B分别存储在5个比特位中，通过把最高3位复制到最低3位来扩展到8位。举个例子，如果R = 28 = 11100b，则子纹素块1的基色R通道值为11100111b = 231，如果G = 4 = 00100b，则子纹素块1的基色G通道值为00100001b = 33，如果B = 3 = 00011b，则子纹素块1的基色G通道值为00011000b = 24，所以子纹素块1的基色为(231,33,24)。第二个子纹素块2的颜色由RGB和颜色增量$R_{d}G_{d}B_{d}$来决定，RGB每个占5比特位，$R_{d}G_{d}B_{d}$每个占3比特位，其中$R_{d}G_{d}B_{d}$是通过 three-bit two’s complement number[two’s complement资料](#jumptwo-s-complement)来表示-4到+3的值。举个例子，如果R = 28，$R_{d}$ = 100b = -4，则R的5比特位表示的为28 + (-4) = 24 = 11000b，扩展到8比特位即是11000110b = 198，同样，如果G = 4, $G_{d}$ = 2, B = 3, $B_{d}$ = 0，则子纹素块2的基色为(198,49,24)。总之，differential模式下子纹素块的基色为：

$$
base color_{subblock1}=extend\_5to8bits(R,G,B)\\
base color_{subblock2}=extend\_5to8bits(R+R_{d},G+G_{d},B+B_{d})
$$

请注意，这些添加不允许溢出（低于0或高于31）。压缩方案可以很容易地确保它们不会。对于 4×4 块中的所有纹素，溢出或下溢的值是未定义的。另请注意，扩展到八位是在添加后执行的。

获得基色之后，对于2种不同的模式，下面的操作是一样的：

首先对于子纹素块1，取表码字1(table codeword)(比特位 39-37)，对于子纹素块2，取表码字2(比特位 36-34)。通过表码字来选择8个编辑表(Modifier table)中的一个，8个编辑表如下图。举个例子，比如表码字是010b = 2，则取的编辑表就是[-29, -9, 9, 29]。需要注意的是下图中的值适用于所有纹理，因此可以硬编码到解压缩单元中。

![Texture Compression Formats7](/assets/images/2022/2022-02-28-TextureCompressionFormats/Texture_Compression_Formats_7.png)

然后，通过2个纹素索引(pixel index)位来确定使用编辑表中的哪个值。每个纹素都有自己独立的纹素索引位。举个例子，[Figure 39](#jumpFigure39)中，对于纹素d的纹素索引可以通过比特位19(most significant bit, MSB)和比特位3(least significant bit, LSB)来确定，[Table 125](#jumpTable125)中(c)部分。注意，对于一个特定的纹素，它的纹素索引永远都是存储在相同的比特位，和diff bit，flip bit无关。纹素索引通过下表来解析。假设纹素索引是01b = 1，编辑表是[-29, -9, 9, 29]，则通过下表就能确定编辑值(modifier value)就是29。现在把这个编辑值和基色相加，举例，如果有基色(231, 8, 16)，则把RGB三个值都加上编辑值，即是：(231+29, 8+29, 16+29)，结果为(260,37,45)，这些值需要被限制在[0, 255]，所以最终的颜色为(255, 37, 45)。这样就完成了纹素的解码。

![Texture Compression Formats8](/assets/images/2022/2022-02-28-TextureCompressionFormats/Texture_Compression_Formats_8.png)

<span id="jumptwo-s-complement">two’s complement资料</span>

[two’s complement资料 1](https://www.cs.cornell.edu/~tomf/notes/cps104/twoscomp.html#operations)

[two’s complement资料 2](https://www.cnblogs.com/effulgent/archive/2011/10/30/two_s_complement.html)

### 2.1. ETC1S

ETC1S格式是ETC1的子集，简化了一些内容方便图像压缩。纹素块使用差分编码(diff bit = 1)，颜色增量$R_{d} = G_{d} = B_{d}$ = 0，所以2个子纹素块使用同一个基色，2个子纹素块的表码字也是相同的，最后flip bit = 0。

### 2.2. ETC1总结

- 纹理通过一个个的 4x4 纹素块存储，到边缘比4小的时候，会使用纹素块的左部分或者上部分，根据flip bit来控制是 2x4(flip bit = 0) 还是 4x2(flip bit = 1)
- 每个 4x4 的纹素块有64比特位(bit)，通过8字节(Byte)$q_0, q_1, q_2, q_3, q_4, q_5, q_6, q_7$存储，1 Byte = 8 bit，由以下 64 位整数表示：

    ![Texture Compression Formats2](/assets/images/2022/2022-02-28-TextureCompressionFormats/Texture_Compression_Formats_2.png)

- individual模式和differential模式分别有自己的位布局
- 根据flip bit的不同，把 4x4 的纹素块分为2个子纹素块，左右或者上下
- 每一个子纹素块都有一个基色
- 每一个子纹素块有对应的表码字，根据表码字得到编辑表
- 对于每一个子纹素块内的每一个纹素，通过MSB和LSB得到它的编辑值
- 通过基色和编辑值得到每个纹素的RGB值

## 3. ETC2 Compressed Texture Image Formats

- **RGB ETC2**用于压缩RGB数据的格式。是ETC1格式的超集(superset)，这意味着ETC2的解码器可以解码ETC1格式的纹理。ETC2和ETC1主要的不同就是ETC2新增了3中模式，T模式(T-mode)，H模式(H-mode)和Planar模式，其中T模式和H模式适用于锐利的色度块，Planar模式适合平滑的色度块。

- **RGB ETC2 with sRGB encoding**与线性RGB ETC2基本相同，不同之处在于这些值应被解释为使用sRGB传递函数而不是线性RGB值进行编码。

- **RGBA ETC2**编码为RGBA 8比特位数据。RGB部分的编码和RGB ETC2相同，alpha通道单独编码。

- **RGBA ETC2 with sRGB encoding**与RGBA ETC2基本相同，不同之处在于这些值应被解释为使用sRGB传递函数。

- **Unsigned R11 EAC**是单通道无符号格式，和RGBA ETC2的alpha通道相似但不完全相同，它提供更高的精度。可以让硬件以最小的开销来解码。

- **Unsigned RG11 EAC**是双通道无符号格式，每一个通道的编码和Unsigned R11 EAC一样。

- **Signed R11 EAC**是单通道无符号格式，这种可以精确地保存0值并且仍然可以使用正值和负值。它和Unsigned R11 EAC有点相似可以让硬件以最小的开销来解码，但它们也不完全相同。比如，有符号的版本(signed version)不需要在基本码字的基础上加上0.5，和11比特位的扩展不同。

- **Signed RG11 EAC**是双通道无符号格式，每一个通道的编码和Signed R11 EAC一样。

- **RGB ETC2 with punchthrough alpha**和RGB ETC2相似，但是它可以表示穿透(punchthrough)的alpha(完全不透明或者透明)。每一个纹素块可以通过1比特位来决定是不是完全不透明，所以在RGB ETC2 with punchthrough alpha中没有individual模式。对于完全不透明的纹素块，编码成RGB ETC2格式，对于完全透明的纹素块，保留一个索引来表示透明度，RGB通道的解码也受到影响。

- **RGB ETC2 with punchthrough alpha and sRGB encoding**和线性RGB ETC2 with punchthrough alpha基本相同，不同之处在于这些值应被解释为使用sRGB传递函数。

任何ETC格式的纹理压缩都是把纹理压缩成一个个 4x4 的纹素块。ETC2的纹素布局和[Pixel layout for an 8×8 texture using four ETC1 compressed blocks](#jumpFigure38)一样。

如果纹理(或者mipmap的特定mip-level)的宽或者高不是4的倍数，然后添加填充以确保纹理在每个维度中包含整数个 4×4 块，填充不影响纹素坐标。对于[纹素布局](#jumpFigure38)中的纹素$a_1$的纹理坐标总是(u = 0, j = 0)，填充的纹素是无关紧要的，例如对于一个 3x3 的纹理，纹素$m_1,n_1,o_1,d_1,h_1,l_1$和$p_1$对最终的纹理图片不会有影响。

对于RGB ETC2, RGB ETC2 with sRGB encoding, RGBA ETC2 with punchthrough alpha, RGB ETC2 with punchthrough alpha and sRGB encoding这些格式，一个 4x4 的纹素块有64比特位，这个纹素块的数据有8个字节(Byte)：$\{q_0, q_1, q_2, q_3, q_4, q_5, q_6, q_7\}$，其中$q_0$是最低内存地址，$q_7$是最高位内存地址。纹素块的64比特位由以下64位整数表示：

![Texture Compression Formats2](/assets/images/2022/2022-02-28-TextureCompressionFormats/Texture_Compression_Formats_2.png)

对于具有线性或 sRGB 传递函数的 RGBA ETC2，一个 4x4 的纹素块有128比特位，这个纹素块的数据有16个字节(Byte)：$\{q_0, q_1, q_2, q_3, q_4, q_5, q_6, q_7, q_8, q_9, q_{10}, q_{11}, q_{12}, q_{13}, q_{14}, q_{15}\}$，其中$q_{0}$是最低内存地址，$q_{15}$是最高位内存地址。纹素块的128比特位由以下2个64位整数表示：

![Texture Compression Formats9](/assets/images/2022/2022-02-28-TextureCompressionFormats/Texture_Compression_Formats_9.png)

### 3.1. Format RGB ETC2

对于RGB ETC2，每个64位字包含有三通道 4×4 纹素块的信息如下图：

![Texture Compression Formats10](/assets/images/2022/2022-02-28-TextureCompressionFormats/Texture_Compression_Formats_10.png)

下图是ETC2纹理压缩格式的每一个纹素的数据结构，每一个 4x4 的纹素块通过五种模式中的一种来压缩，使用的是哪种模式由[Table 128](#jumpTable128)中的a部分来判断，其中的D是differential bit，如果D = 0，则是individual模式；否则，通过5比特位的R,G和B，以及3比特位的$R_{d},G_{d}$和$B_{d}$来判断是哪种模式，其中R,G和B是0-31的整数，$R_{d},G_{d}$和$B_{d}$是-4到+3的二进制补码整数，首先，R和$R_{d}$相加，如果结果不在[0, 31]区间内，则是T模式；G和$G_{d}$相加，如果结果不在[0, 31]区间内，则是H模式；B和$B_{d}$相加，如果结果不在[0, 31]区间内，则是Planar模式，最后，如果D是1并且所有的R,G,B分别和$R_{d},G_{d}$和$B_{d}$相加的结果都在区间[0, 31]中时，就是differential模式。

<span id="jumpTable128">Table 128. Texel Data format for ETC2 compressed texture formats</span>

![Texture Compression Formats11](/assets/images/2022/2022-02-28-TextureCompressionFormats/Texture_Compression_Formats_11.png)

individual模式和differential模式的比特位布局分别时[Table 128](#jumpTable128)中的(b)部分和(c)部分。它们有一些共同的特点，4x4 的纹素块都被分成2个子纹素块，尺寸是 2x4 或者 4x2，尺寸是由第32比特位来决定，叫作flip bit([Table 128](#jumpTable128)中(b)部分和(c)部分的$F_{B}$)，flip bit = 0时，4x4 的纹素块被分为左右2个 2x4 的子纹素块，flip bit = 1时，4x4 的纹素块被分为上下2个 4x2 的子纹素块。2个子纹素块基色的解析[和ETC1相同](#jumpETC1Basecolor)。

T模式和H模式也有一些共同的特点：和individual模式一样，基色的每个通道都是用4比特位来存储，不一样的是这些位不按顺序存储，如[Table 128](#jumpTable128)中(d)部分和(e)部分。T模式下2个基色的计算如下：

![Texture Compression Formats12](/assets/images/2022/2022-02-28-TextureCompressionFormats/Texture_Compression_Formats_12.png)

其中，<< 表示的是按位左移(bit-wise left shift), | 表示的是按位或(bit-wise OR)。H模式下2个基色的计算如下：

![Texture Compression Formats13](/assets/images/2022/2022-02-28-TextureCompressionFormats/Texture_Compression_Formats_13.png)

T模式和H模式最终有4个描述颜色(paint color)用于解码 4x4 的纹素块。T模式下，描述颜色0 就是第一个基色，描述颜色2 是第二个基色，另外2个描述颜色通过一个距离(distance)来调整一个基色来决定。用[Table 128](#jumpTable128)中$d_{a}$和$d_{b}$，通过计算 $(d_{a}<< 1) \| d_{b}$，得到的值去下表中得到距离d。举个例子，$d_{a}$ = 10b，$d_{b}$ = 1b，则distance index = 101b = 5，通过下表可得到d = 32，则描述颜色1即为第二个基色的RGB通道分别加上d。描述颜色3就为第二个基色的RGB通道都减去这个d。

![Texture Compression Formats14](/assets/images/2022/2022-02-28-TextureCompressionFormats/Texture_Compression_Formats_14.png)

总结一下，T模式下的 4x4 纹素块的4个描述颜色求法为：

![Texture Compression Formats15](/assets/images/2022/2022-02-28-TextureCompressionFormats/Texture_Compression_Formats_15.png)

每一个颜色通道的值都应该限制在[0, 255]。

最后T模式和H模式的纹素最终颜色就是这4个描述颜色中的一个，使用哪个描述颜色由求得的index来决定。例如对于一个纹素d，它的index由第19比特位(most significant bit)和第3比特位(least significant bit)，如果index = 2，则纹素d的颜色就是描述颜色2。

最后一种模式是Planar模式。提供三种基色并用于形成一个颜色平面，用于确定块中各个纹素的颜色。三种基色被存储在RGB676格式中，图[Table 128](#jumpTable128)中的(g)部分。两种辅助颜色的后缀为“h”和“v”，举个例子，三种基色的R通道分别为R,$R_{h}$和$R_{v}$。一些颜色通道被分成不连续的比特位范围，举个例子，最高位的$B^{5}$和后面二位的$B^{4..3}$以及最低三位的$B^{2..0}$构成了6比特位的B通道。

提取出基色的比特位数据后，需要对每个通道的数据扩展到8比特位。举个例子，6比特位的B通道和R通道通过把最高二位复制到最低二位扩展到8位。使用 RGB888 格式的三种基色，每个纹素的颜色可以确定为：

![Texture Compression Formats16](/assets/images/2022/2022-02-28-TextureCompressionFormats/Texture_Compression_Formats_16.png)

其中x, y的范围是0到3，x就是u方向的坐标，y就是v方向的坐标，这2个坐标都是相对于一个 4x4 的纹素块内的。比如[Figure 39](#jumpFigure39)中的纹素g，它的坐标就是x = 1，y = 2。

然后将这些值四舍五入为最接近的整数（如果存在平局，则为较大的整数），然后限制为0到255之间的值。

![Texture Compression Formats17](/assets/images/2022/2022-02-28-TextureCompressionFormats/Texture_Compression_Formats_17.png)

其中 ≫ 表示按位右移。

### 3.2. Format RGBA ETC2

每个 4×4 纹素块的 RGBA8888 信息块被压缩为 128 比特位。分为2个64位整数，$int64bit_{Alpha}$和$int64bit_{Color}$，其中RGB部分解码和RGB ETC2一样。Alpha部分比特位结构如下图：

<span id="jumptTable132">Table 132. Texel Data format for alpha part of RGBA ETC2 compressed textures</span>

![Texture Compression Formats18](/assets/images/2022/2022-02-28-TextureCompressionFormats/Texture_Compression_Formats_18.png)

分为2部分，前16比特位包括一个基码字(base codeword)，表码字(table codeword)和一个乘数(multiplier)，它们一起用于计算要在块中使用的 8 个像素值，其余 48 位分为 16 个 3 位索引，用于为块中的每个像素选择这 8 个可能值之一。

颜色像素索引存储在 a-p 在大端字表示中以递增位顺序排列，低位与高位分开存储。但是，alpha指数存储在 p-a 大端字表示中位递增顺序的顺序，每个 alpha 索引的每个位都连续存储。

一个像素的解码值是一个介于 0 和 255 之间的值，计算方法如下：

![Texture Compression Formats19](/assets/images/2022/2022-02-28-TextureCompressionFormats/Texture_Compression_Formats_19.png)

基码字存储在前 8 位（位 63-56）中，如[Table 132](#jumptTable132)部分(a)所示。修饰符(modifier)处于 位 51-48 ，它们形成一个 4 位索引，用于选择 16 个预先确定的“修饰符表”之一，表如下:

![Texture Compression Formats20](/assets/images/2022/2022-02-28-TextureCompressionFormats/Texture_Compression_Formats_20.png)

例如，表索引为 13（1101 二进制）意味着我们应该使用表 [-1， -2 -3， -10， 0， 1， 2， 9]。要选择应使用哪些值，请查阅要解码的像素的像素索引。如[Table 132](#jumptTable132)部分(b)所示，位 47-0 用于存储块中每个像素的 3 位索引，选择 8 个可能值之一。假设我们对像素 b 感兴趣。它的像素索引存储在位 44-42 中，最高有效位存储在 44 中，最低有效位存储在 42 中。如果像素索引为 011 binary = 3，这意味着我们应该从表中左侧取值 3，即 -10。这现在是我们的修饰符。

在下一步中，我们获得乘数值位(multiplier) 55-52 在 0 和 15 之间形成四位乘法器。此值应与修饰符相乘。不允许编码器产生零的乘法器，但解码器仍应能够处理这种情况（在这种情况下，× 修饰符 = 0）时产生0）。

计算总和并将值限制在区间 [0, 255] 内。结果值为 8 位输出值。

例如，假设基码字为 103，表索引为 13，像素索引为 3，乘数为 2。然后，我们将从基本码字 103（01100111二进制）开始。接下来，表索引为 13 将选择表 [-1、-2、-3、-10、0、1、2、9]，并使用像素索引 3 将生成修饰符 -10。乘数为 2，形成 -10 × 2 = -20。现在，我们将此值添加到基值中，得到 103 - 20 = 83。限制后，我们仍然得到83 = 01010011二进制。这是我们的 8 位输出值。

该规范以 0 到 255 之间的 8 位整数值给出每个通道的输出，这些值都需要除以 255 以获得最终的浮点表示。

请注意，硬件可以在此格式的 alpha 解码部分和 R11 EAC 纹理的部分之间有效共享。
