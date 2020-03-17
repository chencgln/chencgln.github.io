---
layout: post
title:  "Joint Monocular 3D Vehicle Detection and Tracking"
date:   2020-03-02 18:00
categories: PaperReading 
tags: [Object Detection, Vision]
comments: true
toc: true
---

这篇论文提出的一种基于图像的3D目标检测方法，能够在时序上匹配检测的车辆实例，完成目标的检测和跟踪。

模型根据检测到的目标，利用世界坐标和相机坐标对目标进行帧间关联。对于跟踪，算法使用了 occlusion-aware 和 深度信息得到较好的跟踪效果。最后，获取目标的移动信息并利用 LSTM 完成目标的运动轨迹预测。

为得到更多更丰富的训练数据，模型使用了 synthetic 的数据集（GTA5）对模型进行训练。

因此，整体算法流程可以归纳为两部分，

* 对单帧图像进行的单目 3D 目标检测，同时包括目标跟踪的匹配算法；
* LSTM 实现的目标运动状态预测。



**Joint Monocular 3D Vehicle Detection and Tracking**: [PDF](https://arxiv.org/abs/1811.10742)

项目源码：[Github](https://github.com/ucbdrive/3d-vehicle-tracking)


## 整体流程

![](https://note.youdao.com/yws/public/resource/b448f97098a0c7699cad971aeb63da30/xmlnote/WEBRESOURCE317285be5348312f96b15fcf6b64448a/1009
)

* (a) 首先将单帧图像送入网络，得到用于目标检测及跟踪的感兴趣区域 ROI；

* (b) 对各 ROI 区域，预测目标的三维信息，包括目标朝向、三维尺寸、目标深度（距离），以及目标中心三维坐标在对应的二维图像坐标；

* (c) 根据上述三维信息，利用 LSTM 模型进行帧间匹配，实现目标的跟踪；

* (d) 最终，在 3D 跟踪的基础上，利用目标的历史信息，来校正目标当前帧的预测参数，得到更稳定平滑的结果。

 ### 2D BBox

对于目标的检测及 2D BBox 的回归，主要使用了 Faster R-CNN 完成。除了常规的 2D BBox 的检测以外，模型同时预测目标中心店三维坐标在图像上的投影。中心点的在 ROIPooling/ROIAlign 得到的特征上使用 L1 Loss 进行训练。

 ### 3D BBox

3D BBox 的检测任务包含了目标中心点、目标方向、目标三维尺寸和目标距离（深度）的预测。


* 目标中心三维坐标：根据预测的目标深度和中心点的二维投影坐标，结合相机的内外参矩阵，可求解目标的三维坐标；
* 目标方向：目标方向的计算逻辑同 [3D-DeeoBox](https://chencgln.github.io/chencgln.github.io/3DDeepBox/) 中的 MultiBin 的方法；

 ### 目标跟踪

 #### Data Association and Tracking

 目标跟踪的匹配准则主要两点：

 * 已跟踪的目标和当前检测目标之间 BBox 的相交情况。其中当前的跟踪目标会根据预测的速度信息更新其当前的位置。
 * 已跟踪的目标和当前检测目标的表面深度特征的匹配情况。

在跟踪目标并更新状态的策略上，算法同时会过滤一定距离范围外的目标，同时将目标遮挡情况作为状态变化的一项参考因素。


当前帧的检测目标 $s_{a}$ 与已有跟踪目标 $\tau_{a}$ 的关联性由矩阵 $A(\tau_{a}, s_{a})$ 表示，用于衡量目标特征与位置的相关程度：

<!-- $$A_{deep}(\tau_{a}, s_{a})=exp(-\left\|F_{\tau_{a}}, F_{s_{a}}\right\|)$$

$$A_{2D}(\tau_{a}, s_{a})=\frac{d_{\tau_{a}}\cap d_{s_{a}}}{d_{\tau_{a}}\cap d_{s_{a}}}$$

$$A_{3D}(\tau_{a}, s_{a})=\frac{M(X_{\tau_{a}})\cap M(X_{s_{a}})}{M(X_{\tau_{a}})\cap M(X_{s_{a}})}$$ -->



$$A(\tau_{a}, s_{a})=\omega_{deep}A_{deep}+\omega_{2D}A_{2D}+\omega_{3D}A_{3d}$$


其中，$A_{deep}(\tau_{a}, s_{a})$是关于表面特征的相关程度，$A_{2D}$ 和 $A_{3D}$ 分别是 2D、3D BBox 在图像上的IOU。

### 运动状态预测

模型使用了两个 LSTM 模型来保证目标运动状态预测的稳定和连续。

* Prediction LSTM (P-LSTM)，根据先前预测的目标速度，对目标的三维坐标进行更新；
* Updating LSTM (U-LSTM)，根据 P-LSTM 预测的目标当前帧位置，进一步更新目标速度和当前位置。


## 实验和结果

![](https://note.youdao.com/yws/public/resource/b448f97098a0c7699cad971aeb63da30/xmlnote/WEBRESOURCEc255caba77edc305564ad9c3eac97627/1021)

![](https://note.youdao.com/yws/public/resource/b448f97098a0c7699cad971aeb63da30/xmlnote/WEBRESOURCE2a24587175f87618e3947ce49f02c913/1023)

### 示例

![](https://note.youdao.com/yws/public/resource/b448f97098a0c7699cad971aeb63da30/xmlnote/WEBRESOURCEddd1cbc8b4425339c9e10e4023168cc8/1018)

在 KITTI 测试集上的结果如上。