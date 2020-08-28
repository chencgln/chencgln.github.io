---
layout: post
title:  "Object-Aware Centroid Voting for Monocular 3D Object Detection"
date:   2020-07-28 18:00
categories: PaperReading 
tags: [3D Object Detection, Vision]
comments: true
toc: true
---

介绍一篇单目的 3D 目标检测方法。对于 3D 目标检测，核心任务是目标的定位和角度预测。本文主要的核心在于如何更准确的预测目标的中心点的三维坐标。

对目标的定位主要从两个方面进行：

1. 根据目标高度的先验信息，利用针孔相机的成像模型，计算目标的深度；
2. 利用图像上 2D -> 3D 中心点的统计分布和注意力机制，得到目标三维中心点在图像上投影的像素坐标，并求解得到世界坐标系下的 $(X, Y, Z)$ 坐标。

整体的三维位置预测的原理如下：

![](https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200728160718.png)

文章关于目标深度预测的本质是利用高度先验和针孔投影模型，建立像素和真实高度的比例关系，求解目标的深度。并且文章的意思是高度的先验是取某一类别的平均物理高度，最终是否能达论文所描述的精度需要通过实验来验证一下。

## 主要贡献

* 提出了一种端到端的单目3D目标检测框架；
* 基于深度学习的图像特征，实现了图像上目标三维中心点投影坐标的预测。

## 算法流程

### 模型结构

![](https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200728170655.png)

算法整体结构基于 Faster-RCNN 实现，流程可以概括为：

* RPN 模块预测 2D region proposals； 
* 将 ROI 划分为网格，网格中的某一点可能为目标三维中心点的投影。因此，可以提出多个 3D centroid proposals，作为可能的中心点投影 
* 结合上述中心点的 proposals，利用图像上 2D -> 3D 中心点的统计分布和注意力机制，预测中心点的投影坐标，用于求解三维中心的 $(X, Y)$ 值；
* 最后，预测目标的三维尺寸和朝向，得到完整的三维信息。

### 实现细节

#### 基本流程

3D 感知的具体参数有：模型预测目标的三维尺寸(dimensions), 目标朝向(orientation), 目标深度(depth/Z); 并通过预测的深度信息，通过针孔相机的透视模型计算得到目标的$(X,Y,Z)$。

其中三维尺寸和朝向角的预测与主流方法类似，对三维尺寸进行直接回归，朝向角利用了如 3DDeepBox 中 multi-bin 的方法。为了得到完整的 3D 感知结果，还需要目标三维中心坐标的预测。

如何获取目标的中心坐标也是此论文的重点。


#### 目标三维位置预测

目标的深度预测主要利用了目标尺寸中高度的先验值。考虑自动驾驶的场景中，作者假设统一类别障碍物的高度分布较为集中，即方差较小。假定某一类别障碍物的平均高度为 $\bar{H}$，预测的包围框高度 $h$，则目标的距离可以计算为：$Z \approx \frac{f*\bar{H}}{h}$。

随后，通过将目标对应的 ROI 区域划分成 $s*s$ 的网格，在网格中可以选取可能的目标三维中心点的投影。对于中心点投影的 proposal，可以利用其像素坐标和深度预测结果，计算世界坐标下的 $X、Y$：

$$X_i = (u_i- p_x)Z/f,  Y_i = (v_i - p_y)Z/f$$

由此可以得到网格中各点的三维世界坐标 $(X, Y, Z)$

考虑遮挡、截断等情况，对目标 3D 中心点投影的选择需要进一步的优化。文章提出了两种方法对中心点投影的预测进行了优化：

* Appearance Attention Map

由于 2D BBox 存在的误差及目标遮挡等问题的存在，文章引入了 appearance attention 的机制。通过 $1*1$ 的卷积在 ROI 的 feature map 上得到注意力信息 $M_{app}$ 

* Geometric Projection Distribution

GPD 利用 2D BBox 的中心点坐标，预测目标三维中心投影的可能存在区域。假定三维中心的投影相对于 2D BBox 中心点的偏移为一高斯分布。利用 ROI 范围内的特征，可以学习高斯分布的参数 $N(\mu, \sigma^2)$。则可得到 GPD 预测的 confidence map: $M_(geo) = \bar{N}(\bar{\mu}, \bar{\sigma}^2)$

最后，目标中心点的预测表示为：

$$\tilde{P_{3d}} = P_{3d} * G(M_{app}, M_{geo})$$ 

其中，$P_{3d}$ 为网格得到的初步中心点 proposal，$G(·)$ 即两个 confidence map 的组合。


## 实验结果


1. 首先是 KITTI 验证集上 3D 检测的 $AP_{3d}$ 和 $AP_{bev}$。比较了主流的单目、无深度监督方法的 3D 检测精度 (iou 0.5/0.7):

![](https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200730004328.png)


2. KITTI 测试集上的结果比较：

![](https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200730004957.png)

3. 3D 目标检测任务中，最重要的任务之一是测距，目标测距也是此论文提出方法的一重点。下图比较了一些主要的 3D 检测算法的测距精度：
![](https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200730003432.png)


4. 最后，关于三维中心坐标的回归，可以直接对中心坐标的 proposals 进行平均，也可以用全连接网络进行回归。两者对距离的结果比较如下：

![](https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200730015054.png)



