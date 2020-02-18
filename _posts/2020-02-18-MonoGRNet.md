---
layout: post
title:  "MonoGRNet: A Geometric Reasoning Network for Monocular 3D Object Localization"
date:   2019-03-23 21:03:36 +0530
categories: PaperReading ObjectDetection Vision
---

分享一篇多任务的3D目标检测模型。文章主要提出了一种端到端的多任务网络，在 KITTI 数据集上完成目标检测任务，同时对各个目标预测其 3D BoundingBox、深度等。模型的三维检测只需输入一张单目图像即可完成。

**MonoGRNet: A Geometric Reasoning Network for Monocular 3D Object Localization**: [PDF](https://arxiv.org/abs/1811.10247)

源码地址：[Github](https://github.com/Zengyi-Qin/MonoGRNet)

Demo视频：[Demo](https://cloud.tsinghua.edu.cn/f/194ddabfd05d4dc78b9f/)


## 主要贡献

* 创新的基于 instance-level 获取直接获取目标对应的实例深度，而无需稠密的深度图。该方案对目标的遮挡和截断情况较为鲁棒；
* 基于2D图像来推断目标中心点的三维坐标；
* 将目标检测的多项任务（传统2D检测、三维尺寸、三维坐标）融合到一个网络当中进行端到端的训练和推断。


## 模型结构

![](https://note.youdao.com/yws/public/resource/e9583a079e676f597aa824ca1db3b39a/xmlnote/WEBRESOURCEdb3f3eba0551146192ec429ca06dd6dd/198)

MonoGRNet的的模型包含四个主要的组成部分
* 传统的2D检测 (棕色)
* 实例深度预测 (绿色)
* 目标中心点坐标预测 (蓝色)
* 3D BoundingBox 角点预测 (黄色).
  
#### 模型流程

根据流程图，模型进行预测的流程为：
1. 首先完成基本的2D目标检测；
2. 然后预测：1)目标深度，2)目标中心点三维坐标在图像上的投影，并结合两项预测得到中心点的世界坐标;
3. 针对目标回归其3D BBox的角点的相对偏移；
4. 最终，结合多尺度的特征，对上述的深度和坐标进行优化，得到目标的全局三维信息。


## 实现细节

对于3D的目标检测任务，一个目标的3D信息检测可以定义为对以下参数的预测：

$$B_{3d}=(B_{2d}, Z_c, \bm{c}, O)$$

* $B_{2d}$: 目标的 2D BBox
* $Z_c$: 目标的深度
* $\bm{c}$: 目标三维中心点坐标在二维图像上的投影
* $O$: 基于局部特征进行预测的角点偏移

因此，对于文章所提出模型将 3D 目标检测任务分解为4项对应各参数的子任务，并整合在同一网络中进行运算。

### ROI Align

模型在对目标深度、中心点坐标、3D BBox角点的回归过程中，需要用到目标的局部特征。局部特征在 feature map 上根据检测到的 2D BBox 来获取。由于不同目标对应的 BBox 大小不一，模型在BBox 对应范围的 feature map 上，进行了 ROI align的操作，保证了相同大小的 feature 输出到后续模块。

ROI Align 可参考：[RoIPooling、RoIAlign笔记](https://www.cnblogs.com/wangyong/p/8523814.html)

### 三维中心点推断流程



### 损失函数定义

#### 2D 目标检测 Loss

$$L_{conf} = CE_{g\in G}(softmax(P_{obj}^{g}), \widetilde{P}_{obj}^{g})
$$
$$L_{bbox} = \sum_{g}1_{g}^{obj} * d(B_{2d}^{g}, d(\widetilde{B}_{2d}^{g})$$

$$L_{2d} = L_{conf} + \omega L_{bbox}$$

其中，与YOLO类似，$g$ 为最终 featuremap 网格上的各个 cell，负责检测该各 cell 所在位置的目标。 $\omega$ 用于平衡两项 loss 的尺度。

因此可以看到，2D的目标检测的损失包含两部分：
* 各个 cell 是否有目标的置信度，利用 softmax cross_entropy 来计算与 GT 误差；
* 2D BBox 的损失，利用 L1 Loss 计算与 GT 的误差，此处利用 mask 只计算感兴趣cell对应的 BBox损失。

#### 实例测距 Loss

$$L_{zc} = \sum_{g} 1_{g}^{obj} * d(Z_{cc}^{g}, \widetilde{Z}_{c}^{g}) $$
$$L_{\delta c} = \sum_{g} 1_{g}^{obj} * d(Z_{cc}^{g}+\delta _{Z_{c}}^{g}, \widetilde{Z}_{c}^{g}) $$

$$L_{depth} = \alpha L_{zc} + L_{z\delta}$$

文章引入了一种 coarse-to-fine[1] 的距离回归方法。同样对于感兴趣的cell，基于全局特征回归测距，并利用局部特征对测距结果进行优化。因此，测距的损失包含这两部分损失的和。

#### 目标三维坐标 Loss

$$L_{c}^{2d} = \sum_{g} 1_{g}^{obj} * d(g + \delta _{c}^{g}, \widetilde{c}^{g})$$
$$L_{c}^{3d} = \sum_{g} 1_{g}^{obj} * d(C_{s}^{g} + \delta _{C}^{g}, \widetilde{C}^{g})$$

$$L_{location} = \beta L_{c}^{2d} + L_{c}^{3d}$$

由于文章中三维中心点的预测，是基于中心点在图像上的投影进行计算的，此处定义的损失也为两者的损失之和。

#### 3D BBox 角点预测 Loss

$$L_corners= \sum_{g} \sum_{k} 1_{g}^{obj} * d(O_{k}, \widetilde{O}_{k})$$

即所有顶点误差的 L1 Loss。


最终，3D 检测的任务所用的 Loss 可以由整合上述相关 Loss 来得到。通过将角点的局部坐标转到相机坐标系下，得到最终的全局 3D Loss：

$$L_{joint} = \sum_{g} \sum_{k} 1_{g}^{obj} * d(O_{k}^{cam}, \widetilde{O}_{k}^{cam})$$


## 实验和结果

模型在 KITTI 3D Object Detection 上进行进行测试。与 [Mon3D]()， [MF3D]()，[3DOP]() 等方法进行了比较，得到运行时间和 $AP_{3D}$ 的精度如下：

![](https://note.youdao.com/yws/public/resource/e1f9e977c6f457d1baa77d5b7155e232/xmlnote/WEBRESOURCE6d75b03da7e78d2698b77fd31839d275/764)

在速度和精度上都有较好的表现。

#### 示例

![](https://note.youdao.com/yws/public/resource/e1f9e977c6f457d1baa77d5b7155e232/xmlnote/WEBRESOURCEc39e37b81f900156304ba0b738daf1da/766)

上图为部分结果实例，

* (d), (e), (f)：对于截断的目标也有较好的结果
* (g), (h), (i)：对于一些遮挡较为严重的目标，精度较低