---
title: 导向滤波（Guided Filter）原理、应用与工程实现
mathjax: true
tags:
  - filter
  - edge-preserving
categories:
  - - Image Processing
    - Denoise
description: 利用导向图对图像进行保边滤波，取得双边滤波类似的效果的同时，提升效率并避免梯度翻转等artefact。
date: 2023-01-04 18:30:21
---

# 简介

导向滤波（Guided Fliter）显式地利用 guidance image 计算输出图像，其中 guidance image 可以是输入图像本身或者其他图像。导向滤波比起双边滤波来说在边界附近效果较好；另外，它还具有 O(N) 的线性时间的速度优势。

![ae96c131-4f9b-4935-ba60-0d01c73db384](ae96c131-4f9b-4935-ba60-0d01c73db384.png)



# 相关工作

- **Explicit Weighted-Average Filters（显式加权平均滤波器）**

双边滤波可以在平滑的过程中保持边缘，但是会出现不希望的梯度翻转（gradient reversal） 的 artifact。原因在于如果一个像素周围有较少的相似的像素，那么Gaussian  weighed average 就会不稳定。 另外就是双边滤波的效率问题，brute-force 实现双边滤波需要 O(Nr^2)  的时间，后来又有文章提出了O(Nlogr) 和 O(N) 的实现方法。但是这些加速方法需要coarse sampling，从而在Nyquist  condition被严重破坏的情况下，牺牲了质量。其他此类滤波器还有 Edge-Avoiding Wavelets，Domain  Transform Filter。

- **Implicit Weighted-Average Filters （隐式加权平均滤波器）**

优化损失函数，求解线性系统的方法等价与隐式图像滤波。尽管 optimization-based 方法一般可以得到高质量的结果，但是求解线性系统较为费时间。已经有人证明这些隐式的滤波器和显式的是相关联的，显式的滤波器也可以写成求解矩阵问题的形式。

- **Non-average Filters （非均值滤波器）**

保边滤波可以不用平均的方法，如中值滤波，可以看成是一个局部的直方图滤波器。 其余还有 Total-Variation filters。非均值滤波器通常都是计算量较大的。



# 导向滤波（Guided Filter）

除了速度优势以外，导向滤波的一个很好的性能就是可以保持梯度，这是bilateral做不到的，因为会有梯度翻转现象。（Preserves edges, but not gradients）。

![img](v2-4fefe1babc940218364d62939dac34c5_720w.webp)

而导向滤波可以避免这一缺点。基本原理如下：

$$ O_i = a_k G_i + b_k, \forall i \in \omega_k $$

$$ O_i = I_i - n_i $$

其中，$I$ 为输入图像，$G$ 为导向图，$O$ 为输出图像。在这里我们认为输出图像可以看成导向图 $G$ 的一个局部线性变换，其中k是局部化的窗口的中点，因此属于窗口 $\omega_k$  的像素点，都可以用导向图对应的像素点通过（$a_k$, $b_k$）的系数进行变换计算出来。同时，我们认为输入图像 $I$ 是由 $O$  加上我们不希望的噪声或纹理得到的，因此有 $I = O + n$ 。

接下来就是解出这样的系数，使得 $O$ 和 $I$ 的差别尽量小，而且还可以保持局部线性模型。这里利用了带有正则项的 linear ridge regression（岭回归）

$$ E(a_k, b_k) = \sum_{i \ in \omega_k}((a_k G_i + b_k - I_i)^2 + \epsilon a_k^2) $$

求解以上方程得到 $a_k$ 和 $b_k$ 的值，对于一个像素，都会被包含在多个window中，因此最终的结果是对这些window中映射后的值进行平均：

$$ a_k = \frac{Cov(G_k, I_k)}{Var(G_k) + \epsilon}, \quad b_k = \bar{I_k} - a_k\bar{G_k} $$

最后得到的算法为：

![img](v2-8b44dad903656926a0748acaa5530d56_720w.webp)

其中的 $f_{mean}$ 是指具有窗口半径r的均值滤波。下面说明为何其可以实现平滑和保边的效果。

![guided_filter_param_influence](guided_filter_param_influence.png)

【上图为原论文的slide，其中p为input，I为引导图像，q为输出图像。为英文名称对应方便，后文还是用I、G、O表示input，guidance和output】。

可以看出，如果 $I$ 和 $G$ 相同，那么每个窗口内的 $a_k$ 就是输入图像的方差比上方差加 $\epsilon$，$b_k$ 等于 $(1 - a_k)$ 乘以该窗口内的原图像的平均值。



- **如果方差较小，即是一个平滑区域的话，那么方差较小约等于0，则$a_k$ 约等于 0 。那么 $b_k$ 约等于1, 于是 $O_k =  mean(I_k)，这样相当于说这个像素在该窗口内的输出值相当于在这个窗口进行了均值平滑，而考虑到pixel属于多个窗口，如果这是一个平滑区域，那么就相当与多个均值平滑滤波器的级联。**



- **如果方差大，即是边缘的话，$a_k$ 就约等于 cov 除以 var（$\epsilon$ 很小），而由于是线性变换，梯度与 $b_k$ 无关，这样可以保证梯度的仅仅做了一个放缩变换，不会引起梯度翻转。**



为了展示GF和BF的效果的相似性，将同一图像经过GF和BF滤波后，计算两个输出的“PSNR”，**（It is often considered as visually insensitive when the PSNR >= 40 dB） 。**



![53f6dff9-5487-4004-8fb7-30c1c0820353](53f6dff9-5487-4004-8fb7-30c1c0820353.png)

双边滤波（BF）引起梯度翻转问题的原因在于它无法保证滤波前后在边缘处的梯度还能保持一致。对于一个如上图所示的斜坡slope形状的边缘来说，在两个交界点附近的slope上的像素，空间的权重各个方向类似，但是值域上来说，较小的值会更有优势（将左边的平面设想为等值的，那么这些点都距离该slope上的值一样近，因此权重较大，而右边的slope上的点越来越大，导致权重越来越小）。这样一来，左边界附近的slope就被往下拉，类似地，右边界附近会往上提。这种边缘上的梯度变化导致了detail上多了artifact，在进行enhance的时候，就会带来梯度翻转的效应。而导向滤波（GF）由于显式控制了梯度的比例关系，因此保证斜边基本一致，从而解决了这个问题。

除了平滑保边滤波以外，GF还可以作为结构转移滤波。（Interestingly, the  guided filter is not simply a smoothing filter. Due to the local linear  model of q = aI+b, the output q is locally a scaling (plus an offset) of the guidance I. This makes it possible to transfer structure from the  guidance I to the output q, even if the filtering input p is  smooth），因此可以用于feathering/matting 以及 dehazing 等一系列的应用。



# 算法实现与加速

文中还提到一个计算fmean时候的加速算法，由于计算每个像素点都要对周围2r+1个点进行求和（1D情况），但是相邻内两个需要计算的像素点实际上只有一个求和的加数有差别，即右边的pixel只是将左边的pixel已经求好的和减去求和时候的最左边一个数，再加上最右边的右边的数值。例如，如果 r=2 求 fmean(5) = f(3) + f(4) + f(5) + f(6) + f(7) ，而fmean(6) = f(4) +  f(5) + f(6) + f(7) + f(8) ，因此 fmean(6) = fmean(5) - f(3) + f(8)  。这样计算当r比较大的时候可以显著减少运算量，避免了重复求和。算法如下：

![v2-69114b7134b01dfb5e99495fd394b49a_720w](v2-69114b7134b01dfb5e99495fd394b49a_720w.png)

GF 还有更高的提速的算法，通过降采样求出a和b，在升采样回原尺寸，并用来对 I 线性变换。这样时间可以由 O(N) 降低到 O(N/s^2) ，其中s是采样率。

算法如下：

![v2-199c95da139938b1f178c73c0099f553_720w](v2-199c95da139938b1f178c73c0099f553_720w.webp)



# 导向滤波参数控制

导向滤波有两个参数，一个是窗口大小 radius，一个是正则系数 $epsilon$ 。导向滤波实际上主要是利用正则项系数 $epsilon$ 来控制滤波强度。直观理解就是对图像的局部区域向guidance（可能是input图像本身）进行线性重构，正则系数控制的就是重构的准确性与平滑性之间的均衡。如果在guidance就是原图的情况下，该系数实际上起着一个对于局部方差的soft threshold的作用，从而区分边缘和平滑区的噪声。



# 导向图为输入图自身的导向滤波

导向滤波的guidance可以是另外的图，比如一个分割mask或者一个经过颜色变换后的低分辨率等尺寸图像。但是也可以将输入图自身作为guidance的。这种情况下，导向滤波的 $a_k$ 和 $b_k$ 可以被化简为仅与输入图窗口内部的方差有关。

为了便于说明，可以更进一步，将目标图像的表达式写为

$$ O_k = \bar{I_k} + a_k (G_k - \bar{G_k}) $$

其中：

$$ a_k = \frac{cov(I_k, G_k)}{var(I_k) + \epsilon} $$

可以直观理解：输出图包括两部分：输入图的均值；导向图guidance的detail成分（按照一定比例叠加）。

当导向图为输入图自身时，上述目标图像的表达式可以写成：

$$O_k = \bar{I_k} + a_k (I_k - \bar{I_k}) = \bar{I_k} + a_k I_{detail} $$

同时：

$$ a_k = \frac{var(I_k)}{var(I_k) + \epsilon} $$

也就是说，$a_k$ 控制的是多大程度地保留原图的细节。这也正是保边滤波的核心问题。当区域方差大时，$a_k$ 约等于1，也就是保留原图的全部信息（高频+低频）；反之，$a_k$约等于0，此时等价于一个均值滤波，用来平滑噪声。



# reference

[1] K. He, J. Sun, and X. Tang. Guided image filtering. In ECCV, pages 1–14. 2010.

[2] K. He, J. Sun, and X. Tang. Guided image filtering. TPAMI, 35(6):1397–1409, 2013

[3] He K, Sun J. Fast Guided Filter[J]. Computer Science, 2015.

[4] https://kaiminghe.github.io/eccv10/eccv10ppt.pdf



> 2017/12/22 Fri 14:34 version 1 初版
>
> 2023/01/04 Wed 21:02 version 2 修改之前的一些问题，增补直观理解与输入做guide的分析







