---
layout: post
title:  "Rethinking Pseudo-LiDAR Representation"
date:   2020-8-16 18:00
categories: PaperReading 
tags: [3D Object Detection, Vision]
comments: true
toc: true
---

## 要点

本文目的是探索 Pseudo—Lidar 文章能够获得较好的 3D 检测结果的根源。通过设计对比实验，分析了不同形式的 Pseudo-Lidar 点云得到的 3D 检测结果。得出结论：深度图到 3D 点云的坐标转换，是获取深度信息辅助 3D 检测的关键。并非如 Pseudo-Lidar 论文中描述的必须用点云的方法去训练，2D CNN 对点云进行卷积，同样可以达到一样的效果。

## 背景

#### Pseudo-Lidar Pipeline

基于深度信息的3D目标检测方法，大致有如下几个关键点：

![](https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200816202432.png)

a). 根据输入的rgb图像，生成2D目标检测和深度估计的结果
b). 利用深度图，将深度图的像素坐标转换至点云世界坐标
c). 基于 pseudo-lidar 论文的方法：对点云按照 PointNet 的方式进行学习，得到有效的特征和检测结果
d). 本文的PatchNet方法，直接对转换坐标的深度图使用图像的2D卷积方法，预测 3D 的相关参数

回顾一下 Pseudo-Lidar（PL）论文，论文认为，将深度的信息直接与图像结合，如作为 RGB 以外的另一通道，无法提升 3D 检测的性能。其原因为深度信息的表达方式使用不正确。实验如下：

1. 对一深度图像，进行一次 2D CNN，并将原始深度图和卷积的结果同时投影到三维空间（bottom-left, bottom-right），可以观察到 2D CNN 无法保留深度方面的特征：
![](https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200821104908.png)
2. 而若将深度图投影成点云，直接使用点云的 3D 检测方法，则更有益于保留空间的特征，并大幅提升3D 检测的性能：
![](https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200821105503.png)

由此推断，对深度数据的表达方式，是影响 3D 检测性能的关键。

#### F-PointNet

F-PointNet 是一种结合图像和点云的 3D 检测方法，模型同时将 RGB 图像和点云作为输入。利用图像进行 2D 获取目标范围内的点云，最后对点云进行分类、并完成 3D 参数的预测：

![](https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200824143807.png)

\* amodal (非模态) BBox 指 BBox 包含完整的目标，即使目标部分不可见。

1. 由 RGB 图像得到 2D BBox 的检测结果（FPN）；
2. 2D BBox 对应三维空间的一个视椎体，可在视椎体内获取输入的点云数据，对应当前目标相关的点云 $(n*c), c=3$;
3. 对 $(n*c)$ 的各点云进行分类，得到类似目标的点云实例分割结果 $(m*c)$;
4. 经过实验，直接对实例分割的点云结果计算目标的中心点，会有一定的偏差。为此，设计了一个轻量级的**空间变换网络（spatial transformer network）** T-Net 用于预测目标的 3D 中心点；
5. 最后，用 3D Box Estimation 模型预测 3D 相关参数 $(x,y,z), (w,h,l), orien$


## 本文主要工作


* 验证了 Pseudo-Lidar 的能够有效的进行 3D 检测，并非得益于其数据的表达方式。像素坐标 -> 点云坐标的坐标变换，是提升 3D 检测的关键。
* 引入了更 powerful 的 backbone，以及基于图像的一些优化，表明了 2D CNN 相比于离散的处理点云数据，更能获取语义信息，且有更好的优化前景。

## 算法流程

为了论证 Pseudo-Lidar 有利于 3D 检测是否得益于转换了数据的表达形式，作者设计了两个模型进行了实验：
* PatchNet-vanilla：保留了与 Pseudo-Lidar 基本一致的流程和模型结构，用于论证数据的表征不是3D检测任务的关键
* PatchNet：用了更 powerful 的 backbone，用于论证基于图像的2D 卷积有更好的性能完成深度学习任务。

#### PatchNet-vanilla

PatchNet-vanilla 主要用于验证 Pseudo-Lidar 能够得到更好的3D检测效果，是否得益于数据的表现形式。

Pseudo-Lidar 论文的整理流程可以概括为：

1. 深度图预测，如PSMNet
2. 2D检测获取目标的ROI
3. 对图像 ROI 区域，根据深度预测结果，将像素投影成世界坐标系下的点云坐标
4. 引用 F-PointNet 的方法，进行 point-wise 卷积训练 3D 检测器。

对此，设计了 PatchNet-vanilla，首先使用上述的步骤 1～3 进行目标检测和虚拟点云的生成。其中，在点云数据的使用上，PatchNet-vanilla 将其组织成了尺寸为 $N*N*3$ 的张量，其中最后一个维度分别表示一个空间点的 $(x, y, x)$ 坐标。如下图所示，左图为 Pseudo-Lidar 使用的无序点云，用以为卷积进行操作；右图为 PatchNet-vanilla 得到的点云 ($M=N*N$)，其表征与图像类似:

![](https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200816201111.png)


实验比较

![](https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200819210459.png)

数值上看，论文提出的 PatchNet-vanilla 基本与 Pseudo-Lidar 的性能相匹配。也证明了数据的表达方法，并不是影响检测性能的关键。


### PatchNet

![](https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200816201016.png)

PatchNet 首先利用 Pseudo-Lidar 相同的步骤进行目标检测和深度图预测，获取目标 ROI 并对 ROI 范围内按深度预测投影成世界坐标点云。

随后将点云组织成上述的 $(x, y, z)$ 的通道，并作为输入给到上图所示的模型。最终，按难度分配至相应的 head 预测目标3D检测对应的参数：$(x, y, z, h, w, l, θ)$

* BackBone: 使用了 SE-ResNet-18 (Squeeze-and-Excitation + Resnet-18)。 其中，去除了 pooling 层，保持了模型特征图与输入图像的尺寸一致。
* Mask Global Pooling：Backbone 得到的特征送入 head 前，会进行 global pooling 将其转换至向量。文中，为使更好的聚焦目标对应的特征，使用目标分割的 mask 对特征进行了过滤，去除了无效的特征。可根据深度图设定阈值来区分前景和北京，得到 mask。
* Head: KITTI的目标按检测难度划分了“Hard, Moderate, Easy”三个类别。模型据此使用了三个对应的head。因此，将特征输入到对应Head前，还经过了一个判断目标难度的模块。这样能够让不同的分支按照难度学习不同的模型参数。（这段不是很清楚）

#### 损失函数


![](https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200826152958.png)

$$L = L_{center} + L_{size} + L_{head\_angle} + \lambda L_{corners}$$ 

\* $L_{corners}$ 参照了 F-PointNet：尽管前三项 3D 参数足够表征 3D BBox，加入 8 各 3D 角点的误差，能对各项误差进行融合。

#### 实验结果


在 KITTI 上 3D 检测的表现：
![](https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200819220156.png)


下图比较了，输入不同坐标系的数据得到的检测精度。可以看到：
1. 只提供深度信息的效果不佳，需要左右的横向位移；（line1/line3）
2. (x) 和 (u,v) 能同时提供目标的横向位移的信息；但是真实的三维空间坐标比像素坐标更有效；（line2/line3/line4）

![](https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200819220630.png)


最后，为了比较 2D CNN 利用的语义特征，是否能得到更好的检测效果，做了如下的 不同 BackBone的对比实验：

![](https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200819222012.png)

1. 基于 2D CNN 的 PatchNet 能够得到更好的检测效果，对 Hard 样本的优势更为明显。推测因其获取的图像语义信息能够更好地处理遮挡、截断的情况。
2. 更深的 Backbone 不能持续提升检测性能，可能发生了过拟合。


各类 pooling 方法的性能比较

![](https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200820014201.png)

mask 的可视化效果。左：mask， 右：without mask

![](https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200816213750.png)

分类网络的作用。分别采用目标检测的难度和目标距离划分样本对应的 head 分支，其表现如下：

![](https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200820014643.png)



### Appendix

![](https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200824213446.png)

F-PointNet 模块选择