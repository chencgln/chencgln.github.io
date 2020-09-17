---
layout: post
title:  "3DVP & SubCNN for 3D Detection"
date:   2020-08-01 18:00
categories: PaperReading 
tags: [3D Object Detection, Vision]
comments: true
toc: true
---

## 3DVP

3DVP: [Data-Driven 3D Voxel Patterns for Object Category Recognition]() （CVPR 2015）针对如 KITTI 的这样的自动驾驶环境，存在较多的遮挡、截断、目标位置集中的情况下，实现 3D 目标检测，并能够有效的预测遮挡的边界。

文章提出了 3DVP，将目标编码成 3D voxel exemplar，包含了目标的信息包括：

* 目标表面的 RGB 信息
* 目标的体素 voxel
* 遮挡和截断的分割 mask
* 可从上述信息推断观察角

具体的 3D voxel exemplar 的生成方法如下，利用预定义的 CAD 模型，匹配 KITTI 数据中最匹配的目标。将 CAD 模型投影至图像并体素化，来去定目标各体素单元的遮挡等信息：

![](https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200901165321.png)

图中可见，用不同颜色标记了目标遮挡状态的分割结果。

文章目标提出一个算法，能够对输入图像生成分割和3D Voxel，最终得到包含appearance、shape和occlusion的 3D Voxel exemplar样本。


### 3DVPs

定义了 3DVP (3D Voxel Pattern) 为一组可见的特征较为类似的 3D voxel exemplar。

根据各个 3D Voxel exemplar，使用聚类的方法，可以把 3D Voxel 所编码的可见性信息类似的目标进行分组，分组的示例如下：

![](https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200901184925.png)

然后，对某一组 3D Voxel exemplar 对应的 3DVP，单独训练检测器预测对此类特征的目标进行检测。



总的训练和测试流程如下：

![](https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200901190556.png)

训练过程：

* 将 3D CAD 模型和图像的目标进行匹配
* 生成编码了目标特征、遮挡状态、形状等特征的 3D voxel exemplar
* 聚类包含相似可见性特征的 3D voxel exemplar 为 3DVP，对不同特征的组，即得到多个 3DVPs
* 对各 3DVP 训练独立的 2D 目标检测器

测试阶段，如上图 $b$

* 利用各个 3DVP 对应的检测器，在输入图像上检测相应的目标，得到 2D 检测结果
* 将检测结果变换至 3DVP，并解析遮挡信息
* 将 3DVP 投影至 3D 空间，获取位置、朝向的 3D 信息



## SubCNN

SubCNN：[Subcategory-aware Convolutional Neural Networks for Object Proposals and Detection]() 是作者 2017 年发表的文章。

同样提出传统 2D 检测在自动驾驶数据方面表现不足的问题，针对目标的 scale 多变、遮挡、截断的数据，需要设计更好的 region proposal 方案。

对此，引入了 subcategory 的概念来进行目标检测的 region proposal。Subcategory 指的是一些拥有相似表面特征、3D 姿态和形状属性的目标。根据这些属性对目标进行分类，在预测阶段能够作为特征来更好的检测目标。

这篇文章的方法就是将 3DVPs 作为 subcategory，与目标检测进行联合训练，得到 3D 检测的效果。


文章的主要工作：

1. 在 RPN 网络中，引入了 subcategory 的卷积模块，卷积的各个核用于预测某一特定的类别；即对应输出的 channel 用于生成该类别对应的 heat map
2. 将 3DVPs 作为子任务，并联合训练检测和 subcategory 任务
3. 使用了 RPN 网络，并添加了 feature extrapolating layer，有效地提取多尺度特征


### 算法流程

模型结构和流程如下图所示：

#### RPN 网络
![](https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200904164450.png)

* 构建不同尺度的图像金字塔，每个尺度得到相应的 feature map
* 引入 feature extrapolating layer 来得到生成图像的 scale 没有覆盖的尺寸的 feature map
* 将得到的特征输入用于 subcategory 的卷积层，卷积层的每个 filter 对应 subcategory 中的一类，这一 filter 能够给出特定 scale 的目标在 feature map 上分布的 confidence。**在heat map 的生成和RPN的分类使用的是共享权重的卷积层**，都用于生成对应 subcategory 的 confidence；
* 根据 confidence，过滤得到目标的 ROI，并将 extrapolated conv 的特征图对应的 ROI 进行 ROI pooling
* ROI pooling 的结果最后用作 subcategory 的分类和 BBox 的回归

note：
 * subcategory conv layer 的权重只在 RPN 的分类分支 softmax 返回的梯度进行更新；

#### 检测网络

![](https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200904164546.png)


RPN 后是检测阶段，检测阶段即使用了上述提取的特征，经过 ROI pooling 得到 ROI 特征，接上了三个分支：类别和子类别的分类，以及 bbox 的回归。


得到检测的目标以及目标的对应的 3DVP 类别（subcategory），即可使用 3DVP 当中的 Occlusion Reasoning 后处理，得到 3D 相关的参数。


## Performance

![](https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200905163432.png)




















