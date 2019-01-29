---
title: Early Stopping
date: 2018-04-06 15:08:09
tags: deeplearning
---
# 目的
为了获得性能良好的神经网络，网络定型过程中需要进行许多关于所用设置（超参数）的决策。超参数之一是定型周期（epoch）的数量：亦即应当完整遍历数据集多少次（一次为一个epoch）？如果epoch数量太少，网络有可能发生欠拟合（即对于定型数据的学习不够充分）；如果epoch数量太多，则有可能发生过拟合（即网络对定型数据中的“噪声”而非信号拟合）。

早停法旨在解决epoch数量需要手动设置的问题。它也可以被视为一种能够避免网络发生过拟合的正则化方法（与L1/L2权重衰减和丢弃法类似）。

根本原因就是因为继续训练会导致测试集上的准确率下降。
那继续训练导致测试准确率下降的原因猜测可能是1. 过拟合 2. 学习率过大导致不收敛
# 原理
* 将数据分为训练集和验证集
* 每个epoch结束后（或每N个epoch后）： 在验证集上获取测试结果，随着epoch的增加，如果在验证集上发现测试误差上升，则停止训练； 
* 将停止之后的权重作为网络的最终参数。

这种做法很符合直观感受，因为精度都不再提高了，在继续训练也是无益的，只会提高训练的时间。那么该做法的一个重点便是怎样才认为验证集精度不再提高了呢？并不是说验证集精度一降下来便认为不再提高了，因为可能经过这个Epoch后，精度降低了，但是随后的Epoch又让精度又上去了，所以不能根据一两次的连续降低就判断不再提高。一般的做法是，在训练的过程中，记录到目前为止最好的验证集精度，当连续10次Epoch（或者更多次）没达到最佳精度时，则可以认为精度不再提高了。
## 直观理解
![Early Stopping](https://upload-images.jianshu.io/upload_images/10461798-7451d24900546ce7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

最优模型是在垂直虚线的时间点保存下来的模型，即处理测试集时准确率最高的模型。
# 为什么能减小过拟合
![](https://upload-images.jianshu.io/upload_images/10461798-22c4f22cc9d2c95c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当还未在神经网络运行太多迭代过程的时候，w参数接近于0，因为随机初始化w值的时候，它的值是较小的随机值。当你开始迭代过程，w的值会变得越来越大。到后面时，w的值已经变得十分大了。所以early stopping要做的就是在中间点停止迭代过程。我们将会得到一个中等大小的w参数，会得到与L2正则化相似的结果，选择了w参数较小的神经网络。
# Early Stopping的缺点
![](https://upload-images.jianshu.io/upload_images/10461798-ce9a75357b5bb137.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

没有采取不同的方式来解决优化损失函数和降低方差这两个问题，而是用一种方法同时解决两个问题 ，结果就是要考虑的东西变得更复杂。之所以不能独立地处理，因为如果你停止了优化代价函数，你可能会发现代价函数的值不够小，同时你又不希望过拟合。

# 扩充
如果不用early stopping降低过拟合，另一种方法就是L2正则化，但需尝试L2正则化超级参数λ的很多值，个人更倾向于使用L2正则化，尝试许多不同的λ值。

# 参考资料
* [https://deeplearning4j.org/cn/earlystopping](https://deeplearning4j.org/cn/earlystopping)
* [https://blog.csdn.net/heyongluoyao8/article/details/49429629](https://blog.csdn.net/heyongluoyao8/article/details/49429629)
* [deeplearning.ai](https://www.deeplearning.ai/)