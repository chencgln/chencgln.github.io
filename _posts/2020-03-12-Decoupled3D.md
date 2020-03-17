---
layout: post
title:  "Monocular 3D Object Detection with Decoupled Structured Polygon Estimation and Height-Guided Depth Estimation
"
date:   2020-03-12 18:00
categories: PaperReading 
tags: [Object Detection, Vision]
comments: true
toc: true
---

分享一片最近发表关于 3D 目标检测的文章，文章提出了一种 Decoupled-3D 的目标检测框架进行三维的目标检测，将完整的 3D 目标检测的任务分解成了：多边形预测（structured polygon prediction）和距离预测两个子任务。利用检测得到的多边形，即3D BBox在图像上的二维投影，加上各个顶点深度的预测结果，即可求解各个顶点的三维坐标，得到目标的3D BBox。


## 背景

众所周知，由于二维图像损失了深度信息，距离信息未知时，二维图像目标检测得到的 2D BBox 可以对应世界坐标系下无数个 3D BBox，如图：
![](https://note.youdao.com/yws/public/resource/b7398f4a0e3c8a7a1d1e02f8732308f8/xmlnote/WEBRESOURCE45176c15a50b121b74759b61fe74cd66/1129)

目标的三维信息通常包括：三维世界坐标$(x, y, z)$和三维尺寸$(h, w, l)$，相比于这些参数的预测，利用图像特征预测图像上 3D BBox 的投影（8个顶点）或许更加切实可行的。

而利用 3D BBox 的投影和其多边形特征，再结合目标的深度信息和投影矩阵，能够首先还原一个目标的粗略3D BBox。再利用单目depth map（[DORN](Deep Ordinal Regression Network for Monocular Depth Estimation.)），生成鸟瞰图，对3D BBox进行优化，得到最终精确的3D BBox。

算法通过图像->深度图->鸟瞰图的流程，类似Pseudo Lidar的思路，通过2D图像得到的提取的三维信息来辅助预测，更好的保留了深度的特征，保证了3D BBox的准确性。

基于上述分析，一个3D目标检测的任务可以拆分成3D BBox投影的多边形预测和目标的距离预测两项子任务。


<!--
1. 3d object proposals using stereo imagery for accurate object class detection
2. Monocular 3d object de- tection for autonomous driving.
3. Multi-view 3d object detection network for au- tonomous driving.
4. 3d bounding box esti- mation using deep learning and geometry.
5. Multi-level fusion based 3d object detection from monocular images.
6. Frustum pointnets for 3d object detection from rgb-d data. -->


## 主要贡献

* 对 3D 目标检测任务进行了拆分，分别利用图像特征，求解目标的3D BBox的投影多边形和目标的距离
* 利用目标的高度先验信息，并结合以上步骤得到的目标多边形和距离，得到粗略的3D BBox
* 基于单目深度图生成鸟瞰图，利用图像的深度特征来对3D BBox进行优化，得到精确的3D BBox结果



## 算法流程

### 基本概念

#### 3D BBox

空间中一个3D BBox包含8个顶点，其在图像上的投影可以得到一个二维的多边形：

$$\{P_{i}=[X_{i}, Y_{i}, Z_{i}]^T \in \mathbb{R}^3\} \Rightarrow \{p_i=[u_i,v_i]^T\}; i=1,...,8$$

假设相机内参$K$，则有：

$$ 
Z_i\begin{bmatrix} u_i \\ v_i \\ 1 \end{bmatrix} 
= K\begin{bmatrix} X_i \\ Y_i \\ Z_i \end{bmatrix}
$$

其中Z_i为 $P$ 点的距离，即深度信息，则根据图像坐标 $(u_i, v_i)$ ，可以计算三维世界坐标：

$$
\begin{bmatrix} X_i \\ Y_i \\ Z_i \end{bmatrix} = 
K^{-1}\begin{bmatrix} u_i \\ v_i \\ 1 \end{bmatrix} Z_i
$$

因此，通过预测目标的3D BBox各顶点的图像投影及其深度信息，即可求解目标的三维坐标系下的3D BBox。

### 模型结构

根据上述对3D目标检测任务的拆分，可以将算法概括成如下流程：

![](https://note.youdao.com/yws/public/resource/b7398f4a0e3c8a7a1d1e02f8732308f8/xmlnote/WEBRESOURCE33380ca247f196285bb5d402223776b3/1133)

模型共包含三个部分，

* 基于 hourglass network 进行的目标对应的多边形检测
* 基于目标高度的先验信息，计算的到目标的深度信息
* 利用单目深度图得到的鸟瞰图，用深度特征对3D BBox的结果进行调优

### 目标对应多边形预测

首先，利用模型使用了 Faster-RCNN 完成常规的2D目标检测任务。然后，利用各目标在图像上的对应特征，对其3D BBox顶点的投影进行预测。考虑到3D BBox的顶点在图像上往往对应的并非目标本身的特征，以下图最右图像为例，顶点的投影会落在地面或者墙上：

![](https://note.youdao.com/yws/public/resource/b7398f4a0e3c8a7a1d1e02f8732308f8/xmlnote/WEBRESOURCEee3c4c0724b28be91ce14b92846f43e7/1135)

因此，模型同时引入了全局特征对角点进行预测。全局的特征由一个基于 Hourglass的分支得到，该分支对 8 个顶点分别输出一个相应的 heatmap。HeatMap表示了 feature map 上各pixcel对应3D BBox顶点的概率，最终选取概率最大pixcel作为角点的像素坐标。

### 基于高度先验的深度预测

算法对目标深度的预测，通过目标高度的先验信息，结合透视算法计算得到。

![](https://note.youdao.com/yws/public/resource/b7398f4a0e3c8a7a1d1e02f8732308f8/xmlnote/WEBRESOURCEd49c7ce37a86c4c32053e1063b4fd9b4/1137)

结合上图，根据相机的透视算法，我们知道，对一距离为 $Z$ 的目标，相机焦距 $f$，根据角点得到的目标像素高度 $h$，其实际高度$H$，有：$Z=\frac{fH}{h}$

此处，目标的实际高度是基于目标对应的ROI feature接上全连接层回归得到的，模型的输出$t_H$与实际高度标签$G_H$的关系如下：

$$t_H = log(G_H/A_H)$$

其中$A_H$为数据集上所有目标高度的均值。

得到各个顶点的深度后，即可计算三维世界3D BBox各个顶点的位置，得到一个粗略的3D BBox，同时，目标的朝向也可以根据3D BBox的各边计算得到。


### 3D BBox的优化

由于角点和高度等的预测的误差，以上得到的3D BBox往往存在一定的偏差，如下图左：

![](https://note.youdao.com/yws/public/resource/b7398f4a0e3c8a7a1d1e02f8732308f8/xmlnote/WEBRESOURCE5d637b44475e53f216f803dfdb53c804/1140)

考虑基于前视图像的特征，往往容易损失深度方面的信息。为更好的利用深度信息对3D BBox的结果进行优化，算法通过前视图像生成 depth map，并转化为鸟瞰图，最后利用鸟瞰图的特征来校准3D BBox。其中，单目的深度图通过 [DORN](http://openaccess.thecvf.com/content_cvpr_2018/papers/Fu_Deep_Ordinal_Regression_CVPR_2018_paper.pdf)（Deep Ordinal Regression Network ）的方法得到。

首先将预测得到得到的鸟瞰图输入一个CNN网络得到 feature map，根据已有的的3D BBox，在 feature map 上获取相应的固定长度的 fearure vector：3D-ROI。最终，feature vector通过卷积和全连接网络，回归得到关于目标位置、尺寸、朝向的偏差： $(\delta x, \delta y, \delta z, \delta l, \delta w, \delta h, \delta \theta)$。利用偏差对3D BBox进行修正，既得优化的3D BBox。

### 结果分析

3D检测的结果与近年较为典型的方法进行了比较，以鸟瞰图和3D BBox的AP评价来看，模型在 KITTI 的各种难度指标下都由比较理想的结果。也普遍比原理想通的 Pseudo-Lidar的方法。

![](https://note.youdao.com/yws/public/resource/b7398f4a0e3c8a7a1d1e02f8732308f8/xmlnote/WEBRESOURCE647256a0f406e24f4c65bd5a9f0dbd19/1142)

目标在前视和鸟瞰图上的可视化结果，证明了目标的3D检测结果在前视图像和鸟瞰图上都很好的贴合目标，即保证了目标的测距、朝向角的预测精度。

![](https://note.youdao.com/yws/public/resource/b7398f4a0e3c8a7a1d1e02f8732308f8/xmlnote/WEBRESOURCEde5e5e08062c968b498334409058c96c/1144)
 
