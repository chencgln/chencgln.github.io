---
layout: post
title:  "Joint Monocular 3D Vehicle Detection and Tracking"
date:   2020-03-02 18:00
categories: PaperReading 
tags: [Object Detection, Vision]
comments: true
toc: true
---

此篇论文提出的一种基于图像的3D目标检测方法，能够在时序上匹配检测的车辆实例，完成目标的检测和跟踪，并同时还原目标的完整3D BBox。

模型根据检测到的目标2D BBox，利用世界坐标和相机坐标对目标进行帧间关联。对于跟踪，算法使用了 occlusion-aware 和 深度信息得到较好的跟踪效果。最后，获取目标的移动信息并利用 LSTM 完成目标的运动轨迹预测。



**Joint Monocular 3D Vehicle Detection and Tracking**: [PDF](https://arxiv.org/abs/1811.10742)

项目源码：[Github](https://github.com/ucbdrive/3d-vehicle-tracking)

## 主要贡献

* 提出了一种单目进行 3D 目标检测的方法，通过预测目标三维尺寸和朝向，并基于投影几何（projective geometry）求解目标姿态和 3D BBox；
* 使用了 discrete-continuous 的策略：**MultiBin regression**，实现目标朝向的预测；
* 引入了三种新的 3D BBox 评价指标；
* 算法同时在 Pascal 3D+ 数据集上验证了对**视点**预测的有效性.

## 整体流程

![](https://note.youdao.com/yws/public/resource/b448f97098a0c7699cad971aeb63da30/xmlnote/WEBRESOURCE317285be5348312f96b15fcf6b64448a/1009
)

* (a) 首先将单帧图像送入网络，得到用于目标检测及跟踪的感兴趣区域 ROI；

* (b) 对各 ROI 区域，预测目标的三维信息，包括目标朝向、三维尺寸、目标深度（距离），以及目标中心三维坐标在对应的二维图像坐标；

* (c) 根据上述三维信息，利用 LSTM 模型进行帧间匹配，实现目标的跟踪；

* (d) 最终，在 3D 跟踪的基础上，利用目标的历史信息，来校正目标当前帧的预测参数，得到更稳定平滑的结果。

 