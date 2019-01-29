---
title: z-score归一化
date: 2018-04-10 17:34:52
tags: deeplearning
---
# 用途
<br/>
对输入数据进行归一化处理
<br/>
# 公式
<br/>
![](https://upload-images.jianshu.io/upload_images/10461798-67ac2db52752b77e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其中σ为数据标准差（划重点，不是方差），μ为样本平均值。对数据进行归一化后，数据的平均值变为0，方差变为1。
# 直观过程
* 第一步零均值化
* 第二步归一化方差

![](https://upload-images.jianshu.io/upload_images/10461798-4282ff05ca34f274.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

原始数据一开始是这样的：

![](https://upload-images.jianshu.io/upload_images/10461798-b94ed73759496aff.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

前两步减去均值，数据分布为：

![](https://upload-images.jianshu.io/upload_images/10461798-f2e557a28d22d5e8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

注意：此时特征x2的方差比x1要大很多

之后除以数据的标准差，数据分布为：

![](https://upload-images.jianshu.io/upload_images/10461798-ccfe7d02f619d9fd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

注意：需要使用相同的μ和σ来归一化测试集和训练集，而不是在训练集和测试集上分别预估μ和σ。因为我们希望不论是训练数据还是测试数据，都是使用相同μ和σ定义的相同数据转换。其中μ和σ是由训练集数据计算得来的。

# 这有什么用
<br/>
如果不使用归一化,将会得到一个细长狭窄的代价函数(图中箭头标示为最小值点)
<br/>
![](https://upload-images.jianshu.io/upload_images/10461798-8748326185d9eab1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

梯度下降过程为：

![](https://upload-images.jianshu.io/upload_images/10461798-7ed27c8aba6995f0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

下面是归一化后的代价函数

![](https://upload-images.jianshu.io/upload_images/10461798-4179aa1793f67e8a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

梯度下降过程为

![](https://upload-images.jianshu.io/upload_images/10461798-07836f2a8b674e23.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当数据没有归一化的时候，x1的范围较大（这里假设为0 ~ 1000），x2范围较小（这里假设为0 ~ 10），可以观察到这里x2取值范围远大于x2，这样就造成画损失函数的时候，损失函数可以表示为：

![](https://upload-images.jianshu.io/upload_images/10461798-cd9b05d4d20527c9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这样画出来的函数图想的等高线为椭圆状，寻找最优解的过程也较为曲折

![](https://upload-images.jianshu.io/upload_images/10461798-cce1072713bb87bf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

而如果进行归一化处理后，函数的损失函数可以表示为：

![](https://upload-images.jianshu.io/upload_images/10461798-26c532b83f26e012.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

由于两个系数几乎一样，这样画出来的函数图像的等高线则会类似于圆形形状，这样寻找最优解就会较为顺畅：

![](https://upload-images.jianshu.io/upload_images/10461798-79e4582afa3d629b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从上可以看出，数据归一化后，最优解的寻优过程明显会变得平缓，更容易正确的收敛到最优解，加快了梯度下降求最优解的速度。

也可以换一种说法，不同特征的量纲的单位可能会有所不同，变化区间处于不同的数量级。如果不进行归一化，可能导致某些特征被忽略，比如x1的特征范围为1 ~ 1000，x2特征的范围为0 ~ 1，此时如果我们要做一个分类的话，那么x2很可能就会被忽略掉。即使不被忽略掉，做梯度下降时也会变得很慢，甚至不收敛。

# 什么时候使用
当特征的区间相差非常大时使用。比如X1区间是[0,2000]，X2区间是[1,5]，其所形成的等高线非常尖。当使用梯度下降法寻求最优解时，很有可能走“之字型”路线（垂直等高线走），从而导致需要迭代很多次才能收敛。
# 其他类型的归一化
## min-max归一化
也称为离差标准化，是对原始数据的线性变换，使结果值映射到[0 – 1]之间。转换函数如下：

![](https://upload-images.jianshu.io/upload_images/10461798-7c1c0a71119e3f9a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这种比较适和于数值比较集中的情况。如果max和min不稳定，很容易使得归一化结果不稳定，使得后续使用效果也不稳定，实际使用中可以用经验常量值来替代max和min。而且当有新数据加入时，可能导致max和min的变化，需要重新定义。(例子:在处理自然图像时，我们获得的像素值在 [0,255] 区间中，常用的处理是将这些像素值除以 255，使它们缩放到 [0,1] 中)

# 参考资料
* [https://uqer.io/community/share/56c3e9c6228e5b0fe6b17d95](https://uqer.io/community/share/56c3e9c6228e5b0fe6b17d95)
* [deeplearning.ai](https://www.deeplearning.ai/)

注：本文涉及的图片及资料均整理翻译自Andrew Ng的Deep Learning系列课程，版权归其所有。