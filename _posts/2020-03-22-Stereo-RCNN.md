---
layout: post
title:  "Stereo R-CNN based 3D Object Detection for Autonomous Driving
"
date:   2020-04-06 18:00
categories: PaperReading 
tags: [Object Detection, Vision]
comments: true
toc: true
---

Stereo R-CNN 是一种基于 Faster R-CNN 进行双目 3D 目标检测的方法。通过扩展了 Faster R-CNN，其通过接收双目的左图右图，同时完成左右图中的目标检测。

文章认为，目标的深度信息与二维图像的特征并没有很直接的联系，而将木白哦的三维位置预测转换为相关参数的学习并通过立体几何进行计算或许更为合理。因此整体的 3D 目标检测任务概括为：

模型通过在 RPN 网络后添加两个额外的分支，实现对目标关键点、视角、和目标三维尺寸的预测，以上输出可以结合左右图像上检测到的目标 2D BBox 还原 3D BBox。最后，根据目标ROI的特征，利用左右图和图像的成像透视原理，可以对 3D BBox 进行优化，得到精确的 3D BBox。

模型在未使用真实的目标深度和位置标签、但是在表现上超越了现有的任何基于图像的 3D 目标检测方法。比先前最佳的双目方案在 $3D_{AP}$ 和  3D 位置预测提升了约 30% 的精度。


## 主要贡献

* 提出了双目的目标检测模型，能够同时完成左右图上目标的检测
* 根据目标双目检测结果，和预测的关键点，实现基于透视几何算法得到的 3D BBox 和准确的三维位置信息
* KITTI上的指标评价表明，文章提出的方法在基于图像的 3D 目标检测任务上达到了 SOTA，同时达到了类似基于激光方法的性能。



## 算法流程

### 模型结构

![](https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200323171022.png)

模型扩展了 Faster R-CNN 的网络结构，同时接收左图、右图作为输入进行左右图上的目标检测。其中，左右图的输入经过权重共享的 Backbone （ResNet-101 或 FPN）进行特征提取。

整体流程包括：

* 双目的 RPN 网络提供左右视图中的感兴趣区域 ROI；
* stereo regression 分支用于回归左右视图上的 2D BBox, 对目标的观察角, 目标的三维尺寸和目标类别；
* 在左视图上预测的目标语义关键点，用于构建几何约束来求解目标的3D BBox

最终的 3D 检测结果即可通过上述预测的各个参数计算得到。

#### Stereo RPN

2D BBox 的标签定义为目标左图、右图2D BBox的并集，如下图。这样能够保证检测测目标能同时包含目标在左图和右图上的位置。
![](https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200323180604.png)

如 Faster R-CNN 中的 anchor 机制一样，模型通过判断各 anchor 将通过与 GT BBox 的 IOU确定其是否对应正样本。

由于每个 anchor 同时对应左图右图上的 BBox，则 BBox 回归的分支共需要回归 6 个关于目标 BBox 的参数，用于表示预测的 BBox 相对与 anchor 的偏移量。

$$[\Delta u, \Delta w, \Delta u', \Delta w', \Delta v, \Delta h]$$

其中，$u, v$ 表示左图 BBox 的中心点像素坐标， $w, h$ 表示左图 BBox的宽和高。相应的， 符号$(·)‘$表示右图上相应的值。表示考虑左右视图已经完成矫正，所以在两图上 BBox 的高度和纵向像素坐标应是相同的。

至此，结合分类分支，即可得到左右视图上目标的检测候选结果。在训练和测试阶段，分别会保留最多2000和300个候选框，


根据以上 ROI，分别在左右视图的 feature map 上进行 ROI Align 操作，传入后续用于 3D BBox 检测的网络。

首先，以下是具体的待预测的参数。

#### Stereo Regression


如结构图中 Stereo Regression 分支表示，该分支接收左右视图上 ROI Align 得到的特征，concatenate后经过两个全联接层，最后并传入四个子分支，分别用于回归以: 目标的类别、三维BBox、三维尺寸和视角。
其中，三维BBox回归的参数如上节介绍，为：$[\Delta u, \Delta w, \Delta u', \Delta w', \Delta v, \Delta h]$；而视角的定义如同 [3D-DeepBox](3D-DeepBox) 中关于目标朝向的论证：利用局部图像特征不能有效的表现目标的朝向，本文结合方位角（azimuth）和与 $r\_y$ 相关的 $\theta$($\frac{\pi}{2}-r\_y$) 最为待预测的变量，如图：

![](https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200324004948.png)


#### Keypoint Prediction 关键点预测

文章通过预测目标 3D BBox 上的关键点来拟合 3D BBox，将目标3D BBox底边四个顶点作为关键点，其中，通常只有一个点是可见的并落在底边线段中间，这个点可以定义为观察点（perspective keypoint），观察点往往能够为3D BBox的预测提供更多的约束。此外，关键点还包括两个边界点（boundary keypoint），提供了目标在图像上所占区域的边界信息，也可用来作为约束帮助预测3D BBox。

![](https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200324102237.png)

如模型结构图所示，关键点的预测在左图 ROI Align 后得到的 feature map 上进行。通过若干卷积后，得到大小 $28*28$，通道数为 $6$ 的 feature map（$6*28*28$）。由于关键点均位于目标 2D BBox 的底边，因此更关心水平坐标值的预测，此处通过将 feature map 的各列进行相加，将 feature map 压缩至 $6*28*1$ 的大小。最终 $6*28*1$ 特征图中的$28$ 列用于关键点的预测。$6$ 个 channel 的预测值分别表示：

* 1～4：用于当前$u$位置，对应四个关键点各自的概率
* 5，6：边界点预测：当前点分别对应左右可见边界的概率 

#### 3D BBox

根据以上得到的参数：


* Stereo BBox
* 3D Dimensions
* Perspective Angle
* Perspective Point
  
即可在上述参数构建的几何约束下求解 3D BBox检测结果，求解的目标为：目标的中心点三维坐标，以及目标的全局旋转角（$yaw$角）。

**不同于 [Stereo vision-based semantic 3d object and ego-motion tracking for autonomous driving](Stereo) 只基于单目图像和尺度先验，求解3D BBox的位置和方向，文章基于的双目方法更为鲁棒和准确。**

假设在如下视角对应的双目图像上进行 3D BBox 的预测：

![](https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200402020247.png)

可在预测参数中，提取 $z=\{u_{l}, v_{t}, u_{r}, v_{b}, u_{l}', u_{r}', u_{p}\}$ 七个参数作为约束，结合透视几何，建立方程。假定参数都对相机焦距 $f$ 进行归一化，则方程如下：


![](https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200402020323.png)

共七个方程，求解坐标的三个参数。类似3D DeepBox中的方法，进行拟合得到最终的目标中心坐标和方向信息。

另外，当目标可见的表面少于两个或目标截断、遮挡情况下观察点不可见时，可以使用观察角来对预测结果进行补充：

$$\alpha = \theta + arctan(-\frac{x}{z})$$
 
#### 3D BBox 的优化

由于上述 3D BBox 基于 $7*7$ 大小的 ROI 特征图得到，考虑最终的 feature map 在模型卷积的过程中，损失了图像大量像素级的特征信息，模型同时在原图上获取了sub-pixel级别的高分辨率特征。在优化3D BBox的阶段，文章的方法利用左右视图中匹配到的像素RGB值偏差作为误差，并尝试最小化误差来对目标的深度坐标进行优化。

具体的，若假定目标为规则的长方体，则对目标表面的各个像素，能够得到其相对于目标三维中心点的距离，通过选取目标表面的像素点，可利用像素点的RGB值和像素点的距离构建损失函数。对于像素点的选取，为防止选取了非目标本身对应的像素，算法利用目标的边界做了限制，选取了下图中目标下半部分对应的像素：

![](https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200407192756.png)

对选取的像素点，通过左右视图匹配，计算对应像素点 RGB 的差值作为待优化的误差，如，对一$(u_i, v_i)$的像素，可定义误差：

$$\boldsymbol{e} = \parallel I_l(u_i, v_i) - I_r(u_i - \frac{b}{z+\Delta z_i}, v_i) \parallel$$

其中，$u_i - \frac{b}{z+\Delta z_i}$ 就是双目测距的基本方程，$b$为基线长度，$\Delta z_i = z_i-z $ 表示像素点 $i$ 对应的深度与目标中心点的距离，可以通过已有的目标三维尺寸计算得到。因此，最终上式中待优化的仅是 $z$ 值。

将所有选定的像素对应损失累加，作为待优化的总体损失：
$$\boldsymbol{E} = \sum_{i=0}^N\boldsymbol{e_i}$$

通过在初始的 $z$ 值相邻的范围内进行枚举，来最小化总体损失，即可较高效的得到目标优化后的深度。最后的最后，将固定目标的深度，并再对3D BBox进行校正，得到最终的检测结果。


## 分析结果

在 KITTI 上评测的 $AP_{BEV}$ 和 $AP_3D$：

![](https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200408002747.png)


引入 keypoint 与否的精度对比

![](https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200408003218.png)

引入 alignment 和 rectify 步骤的效果：

![](https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200408003710.png)


可视化结果：

![](https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200408004551.png)




