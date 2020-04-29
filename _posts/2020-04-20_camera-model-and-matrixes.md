---
layout: post
title:  相机成像模型、内外参矩阵和坐标变换
date:   2020-04-20 
categories: 计算机视觉 
tags: [图像基础, 相机坐标]
comments: true
toc: true
---

## 针孔相机模型

针孔相机的成像模型可以表示为：


<div style="text-align: center">

<img src="https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200412004000.png" width=40% />

</div>

相机原点为光心，利用小孔成像原理，将在焦平面上得到场景的倒影。为了减少坐标变换的的麻烦，可在相机前方距离 $f$ 处定义一个像平面，表示相机获取的场景图像。因此，相机光心与实际世界坐标系下某一点 $(x, y, z)$ 的连线，与像平面的交点，即最终该点对应图像上的位置。


<div style="text-align: center">
<img src="https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200412004517.png")/></div>

于是对于世界坐标系下任一点，其在图像上的坐标变换过程如下图：

![](https://glimg.oss-cn-shanghai.aliyuncs.com/test/camera_coords.png)

可以看到，完整的变换过程共涉及四个坐标系：
* **世界坐标系**，单位 m
* **相机坐标系**，以相机为原点，相机的正前方为Z轴+，向右为x轴+，向下为y轴+，单位 m
* **图像坐标系**，以光轴与像平面的交点为原点的二维平面，右下分别为x、y轴的正方向，单位 mm
* **像素坐标**，即以像素 pixel 为单位的坐标系，以最终图像的左上角为原点，右下分别为u、v轴的正方向

利用相机的内外参，可以实现上述四个坐标间的变换，整体过程如下：

![](https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200412233032.png)

下面依次推导空间某一点在各个坐标系下的转换关系。

## 坐标变换

### 世界坐标系 到 相机坐标系

指定世界坐标系下某一顶点 $(X, Y, Z)$ ，通过相机外参指定的相机旋转和平移信息，可以将顶点转换到以相机为原点的，坐标轴的方向如前文所述。

空间中的变换可以通过旋转和平移进行表示，因此以上变换可以表示为：

$$
\left[
 \begin{matrix}
   X_c \\
   Y_c \\
   Z_c 
  \end{matrix}
  \right] = \left[ 
 \begin{matrix}
   R_{11} & R_{12} & R_{13}\\
   R_{21} & R_{22} & R_{23} \\
   R_{31} & R_{32} & R_{33} 
  \end{matrix} 
  \right] \left[
 \begin{matrix}
   X_w \\
   Y_w \\
   Z_w
  \end{matrix}  
  \right] + \left[
 \begin{matrix}
  t_x \\
  t_y \\
  t_z
  \end{matrix} 
  \right]
  $$

其中，旋转矩阵：

$$
 R = \left[
 \begin{matrix}
   r_{11} & r_{12} & r_{13} \\
   r_{21} & r_{22} & r_{23} \\
   r_{31} & r_{32} & r_{33}
  \end{matrix}
  \right] \in R_{3*3}
$$

平移矩阵：

$$
 t = \left[
 \begin{matrix}
   t_x \\
   t_y \\
   t_z 
  \end{matrix}
  \right] \in R_{3*1}
$$

通常，为了简化在三维空间中的点线面表达方式和旋转平移等操作，会将上述变换转换成齐次坐标，齐次即将一个原本是n维的向量用一个n+1维向量来表示：

$$
 \left[
 \begin{matrix}
   X_c \\
   Y_c \\
   Z_c \\
   1 
  \end{matrix}
  \right] = \left[ 
 \begin{matrix}
   R_{11} & R_{12} & R_{13} & t_x\\
   R_{21} & R_{22} & R_{23} & t_y \\
   R_{31} & R_{32} & R_{33} & t_z \\
   0 & 0 & 0 & 1
  \end{matrix} 
  \right] \left[
 \begin{matrix}
   X_w \\
   Y_w \\
   Z_w \\
   1 
  \end{matrix}  
  \right] = \left[
 \begin{matrix}
   R & t \\
   0 & 1 
  \end{matrix}
  \right] \left[
 \begin{matrix}
   X_w \\
   Y_w \\
   Z_w \\
   1 
  \end{matrix}  
  \right] 
$$


### 相机坐标 到 图像坐标

相机坐标系下的某一点 $(X_c， Y_c, Z_c)$ 投影到到图像坐标主要根据相机成像原理，利用相机内参实现转换。相机的内参包括相机的焦距 $(fx，fy)$ 和光心。如文章开头的成像模型所示，根据相似三角形原理，有：

$$\frac{Z_c}{f} = \frac{X_c}{x} = \frac{Y_c}{y}$$

整理得到：

$$\begin{cases}
x = f\frac{X_C}{Z_C} \\
y = f\frac{Y_C}{Z_C} 
\end{cases}$$

对应上一小结的齐次矩阵，上式的矩阵形式：

$$Z_{c}
\left[
    \begin{matrix}
      x \\
      y \\
      1 
    \end{matrix}
\right] = 
\left[
    \begin{matrix}
        f & 0 & 0 & 0 \\
        0 & f & 0 & 0 \\
        0 & 0 & 1 & 0 
    \end{matrix}
\right]
\left[
    \begin{matrix}
        X_c \\ 
        Y_c \\ 
        Z_c \\ 
        1
    \end{matrix}
\right]
$$



### 图像坐标 到 像素坐标

![](https://glimg.oss-cn-shanghai.aliyuncs.com/test/20200428112324.png)

先看一下图像坐标和像素坐标的对应关系，**像素坐标** $(u,v)$ 的原点$o(uv)$位于图像的左上角，$(u, v)$表示像素的列数与行数。**图像坐标**$(x,y)$的原点为相机光轴与图像平面的交点，且x轴、y轴分别与像素坐标的$u$轴$v$轴平行，单位为毫米。

定义像素宽 $dx$、像素高 $dy$ 为毫米/像素，即表示每个像素代表的物理宽、高距离。则对一图像坐标 $x$，计算其对应的像素大小： 

$$ \Delta u = \frac{\Delta x}{dx}$$

结合上图可得图像上的像素坐标：

$$\begin{cases}
u = x\frac{1}{dx} \\
v = y\frac{1}{dy} 
\end{cases}$$

转至矩阵形式为：


$$
\left[
    \begin{matrix}
      u \\
      v \\
      1 
    \end{matrix}
\right] = 
\left[
    \begin{matrix}
        1/dx & 0 & u_0 \\
        0 & 1/dy & v_0 \\
        0 & 0 & 1 
    \end{matrix}
\right]
\left[
    \begin{matrix}
        x \\ 
        y \\ 
        1
    \end{matrix}
\right]
$$

### 完整流程

总结上面的转换过程，则一个完整的从世界坐标一顶点 $(X, Y, Z)$ 转换至图像像素坐标 $(u, v)$ 的方程概括为：

$$Z_c
\left[
    \begin{matrix}
      u \\
      v \\
      1 
    \end{matrix}
\right] = 
\left[
    \begin{matrix}
        1/dx & 0 & u_0 \\
        0 & 1/dy & v_0 \\
        0 & 0 & 1 
    \end{matrix}
\right]
\left[
    \begin{matrix}
        f & 0 & 0 & 0 \\
        0 & f & 0 & 0 \\
        0 & 0 & 1 & 0 
    \end{matrix}
\right]
\left[
 \begin{matrix}
   R & t \\
   0 & 1 
  \end{matrix}
  \right] \left[
 \begin{matrix}
   X_w \\
   Y_w \\
   Z_w \\
   1 
  \end{matrix}  
  \right] 
$$

$$Z_c
\left[
    \begin{matrix}
      u \\
      v \\
      1 
    \end{matrix}
\right] = 
\left[
    \begin{matrix}
        f/dx & 0 & u_0 & 0 \\
        0 & f/dy & v_0 & 0 \\
        0 & 0 & 1 & 0
    \end{matrix}
\right]
\left[
 \begin{matrix}
   R & t \\
   0 & 1 
  \end{matrix}
  \right] \left[
 \begin{matrix}
   X_w \\
   Y_w \\
   Z_w \\
   1 
  \end{matrix}  
  \right] 
$$

其中，相机内参：

$$
\left[
    \begin{matrix}
        f/dx & 0 & u_0 & 0 \\
        0 & f/dy & v_0 & 0 \\
        0 & 0 & 1 & 0
    \end{matrix}
\right]
$$

相机外参：
$$
\left[
 \begin{matrix}
   R & t \\
   0 & 1 
  \end{matrix}
  \right] 
$$