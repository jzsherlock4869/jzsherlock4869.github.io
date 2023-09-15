---
title: 双边滤波原理与改进策略（Bilateral Filtering）
categories:
  - - Image Processing
    - Denoise
date: 2022-12-29 13:18:36
tags: 
- filter
- denoise
description:  双边滤波（bilateral filtering）的基本思路是同时考虑将要被滤波的像素点的空域信息（domain）和值域信息（range）。
---


## 双边滤波原理与改进策略（Bilateral Filtering）

![双边滤波原理（Bilateral Filtering）](v2-20b82bc22476606533ce250d68e32f56_720w.png)

### 基本思路

双边滤波（bilateral filtering）的基本思路是同时考虑将要被滤波的像素点的空域信息（domain）和值域信息（range）。因此是一种 combined 滤波方式，因此叫做 bilateral ，即同时考虑两方面的信息。首先，对于图像滤波来说，一个通常的intuition是：（自然）图像在空间中变化缓慢，因此相邻的像素点会更相近。但是这个假设在图像的边缘处变得不成立。如果在边缘处也用这种思路来进行滤波的话，即认为相邻相近，则得到的结果必然会模糊掉边缘，这是不吼的，因此考虑再利用像素点的值的大小进行补充，因为边缘两侧的点的像素值差别很大，因此会使得其加权的时候权重具有很大的差别，从而使得只考虑自己所属的一边的邻域。可以理解成先根据像素值对要用来进行滤波的邻域做一个分割或分类，再给该点所属的类别相对较高的权重，然后进行邻域加权求和，得到最终结果。

### 实现原理

在 bilateral filtering 中，两个要素即：closeness 和 similarity ，或者说 domain 和 range ，或者 geometric 和 photometric ，其数学表达方式相近，如下：

![v2-c9650f647e5dbaf520e8530b69f218da_720w](v2-c9650f647e5dbaf520e8530b69f218da_720w.png)

![v2-f8abe1c837073b64500290db281fb894_720w](v2-f8abe1c837073b64500290db281fb894_720w.png)

其中积分号前面为归一化因子，这里考虑对所有的像素点进行加权，c 和 s 是 closeness 和 similarity 函数，x 代表要求的点，f (x) 代表该点的像素值。f(x) –> h(x) 为滤波前后的image。我们最后的滤波函数为

![v2-061e03c53523b7665159a5b1c38d4ae1_720w](v2-061e03c53523b7665159a5b1c38d4ae1_720w.png)


由于domain component，使得滤波特性较好，由于range component，使得crisp edge 可以保持。

下图示意了有边缘的时候的权重和最后的滤波结果，可以看出权重在边界有很明显的分界，从而几乎只对自己所属的边缘一侧的像素点进行加权。

![v2-ed438e46a5837a5369f996e0e3b7f84f_720w](v2-ed438e46a5837a5369f996e0e3b7f84f_720w.webp)


实现 c 和 s 两个函数的一种方法即 Gaussian 核，决定其性质的为各自的sigma参数，即 σd 和 σr 。![v2-ebb11d34423238e7596062d5b7b25062_720w](v2-ebb11d34423238e7596062d5b7b25062_720w.png)

![v2-a110c49ab57d4cd6d94069eb69067e9b_720w](v2-a110c49ab57d4cd6d94069eb69067e9b_720w.png)



### 参数讨论

对于空域的 Gaussian 滤波不需要继续解释，对于 Range Filtering ，即不考虑空间只考虑像素点的相似性进行加权的结果，作者进行分析，得到结论，值域滤波只是对待滤波的图像的直方图的一个变换，而对于单峰的（unimodal）的直方图，range filtering 将值域范围向着峰的中间即均值方向压缩。

对于参数的选取，进行如下讨论：

首先，两个 sigma 值为 kernel 的方差，值的方差越大，说明值临近的点影响更大，空间位置的方差越大，说明空间相邻的像素点影响更大。因此：

两个方面的某个的 sigma 相对变大 表示这一方面相对较重要，得到强调。如 sigma_d 变大，表示更多采用近邻的值作平滑，说明图像的空间信息更重要，即相近相似。如 sigma_r 变大，表示和自己同一类的条件变得重要，从而强调值域的相似性。sigma_d 表示的是空域的平滑，因此对于没有边缘的，变化慢的部分更适合；sigma_r 表示值域的差别，因此强调这一差别，即增大 sigma_r 会使得更多类似值被拉齐，从而对比度降低。

### 其它

对于彩色图片，由于两种颜色中可能有其他完全不同的颜色，因此不像灰度图那样，仅仅是 blurred ，而是会产生 auras like 的奇怪的晕圈，所以在双边滤波的过程中，将RGB转换到 CIE-Lab 色彩空间，这个空间与人的主观色彩辨识能力相关，因此可以改善这一缺陷。

### 双边滤波思路的相关改进

#### 双边网格的快速滤波方法

[Real-time Edge-Aware Image Processing with the Bilateral Grid](https://people.csail.mit.edu/sparis/publi/2007/siggraph/Chen_07_Bilateral_Grid.pdf)

由于双边滤波需要计算空间和值域的邻域加权，但是值域的操作与空间不同，因此一个自然的想法就是将值域看做图像的第三维度，得到一个三维张量（类似于二元函数的函数图像，一元函数的函数图为二维，二元函数的函数图就是三维张量）。然后，根据sigma_d和sigma_r的取值不同，对三维数据进行各个维度的切分，划分成网格。对网格滤波后，再通过切片操作（slice，实际上就是用高分辨率图像位置与像素值去低维网格进行插值，获取高分辨率处理结果）得到输出。

#### 双边引导上采样

[Bilateral Guided Upsampling](https://dl.acm.org/doi/pdf/10.1145/2980179.2982423)

TODO...



THE END

version1: 2017/12/18 Mon 22:27

version2: 2023-09-15 Fri 14:29



> reference：
> Tomasi C, Manduchi R. Bilateral filtering for gray and color images” ICCV[J]. Iccv, 1998:839 - 846.
