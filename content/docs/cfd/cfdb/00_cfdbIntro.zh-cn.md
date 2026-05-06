---
uid: 20250918162728
title: 00_cfdbIntro
date: 2025-09-18
update: 2026-03-27
authors:
  - name: Aerosand
    link: https://github.com/aerosand
    image: https://github.com/aerosand.png
tags:
  - cfd
  - cfdb
excludeSearch: false
toc: true
weight: 1
math: true
next:
prev:
comments: true
sidebar:
  exclude: false
draft: false
---

> [!important]
> 访问 [https://aerosand.cc](https://aerosand.cc/) 以获取最近更新。

## 0. 前言

本系列旨在让读者流畅的理解计算流体力学的基础理论。

## 1. 路线

{{% steps %}}

### 流体动力学

理论方程

### 有限差分法

有限差分法基础思路和方法。

### 有限体积法

有限体积法基础思路和方法

{{% /steps %}}

## 2. 数学基础

### 2.1. 偏导计算

和差法则

$$
\frac{\partial (f+g)}{\partial x} = \frac{\partial f}{\partial x} + \frac{\partial g}{\partial x}
$$

乘积法则

$$
\frac{\partial fg}{\partial x} = g\frac{\partial f}{\partial x} + f\frac{\partial g}{\partial x}
$$

常数

$$
\frac{\partial Cf}{\partial x} = C \frac{\partial f}{\partial x}
$$

### 2.2. 张量约定

张量约定也被称为爱因斯坦求和约定，这种数学表达形式十分简洁准确。

重复的下标要求和。

比如

$$
\frac{\partial \phi_{i}}{\partial x_{i}} = \sum\limits_{i} \frac{\partial \phi_{i}}{\partial x_{i}} = \frac{\partial \phi_{1}}{\partial x_{1}} + \frac{\partial \phi_{2}}{\partial x_{2}} + \frac{\partial \phi_{3}}{\partial x_{3}},i=1,2,3
$$

不同的下标要独立。

比如

$$
\begin{align*}
\frac{\partial u_{i}u_{j}}{\partial x_{i}} &= \sum\limits_{i} \frac{\partial u_{i}u_{j}}{\partial x_{i}} \\
&= \begin{pmatrix}
\frac{\partial u_{x}u_{x}}{\partial x} + \frac{\partial u_{y}u_{x}}{\partial y} + \frac{\partial u_{z}u_{x}}{\partial z}\\
\frac{\partial u_{x}u_{y}}{\partial x} + \frac{\partial u_{y}u_{y}}{\partial y} + \frac{\partial u_{z}u_{y}}{\partial z}\\
\frac{\partial u_{x}u_{z}}{\partial x} + \frac{\partial u_{y}u_{z}}{\partial y} + \frac{\partial u_{z}u_{z}}{\partial z}
\end{pmatrix}
\end{align*}
$$

### 2.3. 张量计算

向量内积（inner product）

$$
\mathbf{a}\cdot \mathbf{b} =a_{i}b_{i} = \mathbf{a}^{T}\mathbf{b} = a_{1}b_{1} + a_{2}b_{2} + a_{3}b_{3}
$$

向量和张量内积

$$
\mathbf{a}\cdot \mathbf{T} = a_{i}T_{ij} = \begin{pmatrix}a_{1}T_{11}+a_{2}T_{21}+a_{3}T_{31} \\ a_{1}T_{12}+a_{2}T_{22}+a_{3}T_{32} \\ a_{1}T_{13}+a_{2}T_{23}+a_{3}T_{33}\end{pmatrix}
$$

张量和向量内积

$$
\mathbf{T}\cdot\mathbf{a} = T_{ij}a_{j} = \begin{pmatrix}T_{11}a_{1}+T_{12}a_{2}+T_{13}a_{3} \\ T_{21}a_{1}+T_{22}a_{2}+T_{23}a_{3} \\ T_{31}a_{1}+T_{32}a_{2}+T_{33}a_{3}\end{pmatrix}
$$

张量双内积（double inner product / dyadic product）

$$
\mathbf{T}:\mathbf{S} = T_{ij}S_{ij} = T_{11}S_{11}+T_{12}S_{12}+T_{13}S_{13}+T_{21}S_{21}+T_{22}S_{22}+T_{23}S_{23}+T_{31}S_{31}+T_{32}S_{32}+T_{33}S_{33}
$$

向量外积（outer product）

$$
\mathbf{a} \otimes \mathbf{b} = a_{i}b_{j} = \begin{pmatrix}a_{1}b_{1}&a_{1}b_{2}&a_{1}b_{3} \\ a_{2}b_{1}&a_{2}b_{2}&a_{2}b_{3} \\ a_{3}b_{1}&a_{3}b_{2}&a_{3}b_{3}\end{pmatrix}
$$

### 2.4. 梯度计算

梯度计算是一种升维。

标量的梯度计算

$$
\mathsf{grad}\phi = \nabla\phi = \frac{\partial \phi}{\partial x_{i}} = \begin{pmatrix} \frac{\partial \phi}{\partial x} \\ \frac{\partial \phi}{\partial y}  \\ \frac{\partial \phi}{\partial z} \end{pmatrix}
$$

向量的梯度计算

$$
\mathsf{grad} \mathbf{U} = \nabla \mathbf{U} = \nabla\otimes \mathbf{U} = \frac{\partial b_{i}}{\partial x_{j}} = \begin{pmatrix} \frac{\partial u_{1}}{\partial x_{1}} & \frac{\partial u_{2}}{\partial x_{1}} & \frac{\partial u_{3}}{\partial x_{1}}  \\ \frac{\partial u_{1}}{\partial x_{2}} & \frac{\partial u_{2}}{\partial x_{2}} & \frac{\partial u_{3}}{\partial x_{2}}  \\ \frac{\partial u_{1}}{\partial x_{3}} & \frac{\partial u_{2}}{\partial x_{3}} & \frac{\partial u_{3}}{\partial x_{3}} \end{pmatrix}
$$

### 2.5. 散度计算

散度计算是一种降维。

向量的散度计算

$$
div \mathbf{U} = \nabla\cdot \mathbf{U} = \frac{\partial u_{i}}{\partial x_{i}} = \frac{\partial u_{1}}{\partial x_{1}} + \frac{\partial u_{2}}{\partial x_{2}} + \frac{\partial u_{3}}{\partial x_{3}}
$$

张量的散度计算

$$
div \mathbf{T} = \nabla\cdot \mathbf{T} = \frac{\partial T_{ij}}{\partial x_{i}} = \begin{pmatrix} \frac{\partial T_{11}}{\partial x_{1}}+\frac{\partial T_{21}}{\partial x_{2}}+\frac{\partial T_{31}}{\partial x_{3}} \\ \frac{\partial T_{12}}{\partial x_{1}}+\frac{\partial T_{22}}{\partial x_{2}}+\frac{\partial T_{32}}{\partial x_{3}} \\ \frac{\partial T_{13}}{\partial x_{1}}+\frac{\partial T_{23}}{\partial x_{2}}+\frac{\partial T_{33}}{\partial x_{3}} \end{pmatrix}
$$

### 2.6. 梯度的迹

参考梯度计算

$$
\mathsf{grad} \mathbf{U} = \nabla \mathbf{U} = \nabla\otimes \mathbf{U} = \frac{\partial b_{i}}{\partial x_{j}} = \begin{pmatrix} \frac{\partial u_{1}}{\partial x_{1}} & \frac{\partial u_{2}}{\partial x_{1}} & \frac{\partial u_{3}}{\partial x_{1}}  \\ \frac{\partial u_{1}}{\partial x_{2}} & \frac{\partial u_{2}}{\partial x_{2}} & \frac{\partial u_{3}}{\partial x_{2}}  \\ \frac{\partial u_{1}}{\partial x_{3}} & \frac{\partial u_{2}}{\partial x_{3}} & \frac{\partial u_{3}}{\partial x_{3}} \end{pmatrix}
$$

该梯度计算的迹为

$$
\begin{align*}
\mathrm{tr}(\nabla\mathbf{U}) &=  \sum\limits_{i=1}^{3}(\nabla\mathbf{U})_{ii} \\
&=  \frac{\partial u_{1}}{\partial x_{1}} + \frac{\partial u_{2}}{\partial x_{2}} + \frac{\partial u_{3}}{\partial x_{3}}
\end{align*}
$$

可以看到

$$
\nabla\cdot\mathbf{U} = \mathrm{tr}(\nabla\mathbf{U}) = \mathrm{tr}((\nabla\mathbf{U})^{T})
$$

### 2.6. 混合计算

$$
\nabla\cdot(\mathbf{U}\rho) = \mathbf{U}\cdot\nabla\rho + \rho\nabla\cdot \mathbf{U}
$$

$$
\nabla\cdot(\mathbf{U}\otimes \mathbf{U}) = \mathbf{U}\cdot\nabla\otimes \mathbf{U} + \mathbf{U}\nabla\cdot \mathbf{U}
$$

$$
\nabla\cdot(\mathbf{T}\cdot \mathbf{U}) = \mathbf{T}:\nabla\otimes \mathbf{U} + \mathbf{U}\cdot\nabla\cdot \mathbf{T}
$$

### 2.7. 矩阵分解

任何一个矩阵都可以分解成体部分（平均性）（hydrostatic）和偏部分（非平均性）（deviatoric）。

$$
\mathbf{A} = \mathbf{A}^{hyd} + \mathbf{A}^{dev}
$$

体部分的大小为对角线元素的总和，也是迹的计算

$$
|\mathbf{A}^{hyd}| = \frac{1}{3}tr(\mathbf{A}) = \frac{1}{3}a_{ii}
$$

体部分矩阵为

$$
\mathbf{A}^{hyd} = \frac{1}{3}tr(\mathbf{A})\mathbf{I}
$$

也可以得到

$$
\mathbf{A}^{dev} = \mathbf{A} - \mathbf{A}^{hyd} = \mathbf{A} - \frac{1}{3}tr(\mathbf{A})\mathbf{I}
$$

为了帮助理解，我们以一个三维矩阵为例

$$
\mathbf{A} = \begin{pmatrix} 1&2&3 \\ 3&2&0 \\ 2&0&3 \end{pmatrix}
$$

该矩阵的迹为

$$
tr(\mathbf{A}) = 1 + 2 + 3 = 6
$$

该矩阵的体部分为

$$\mathbf{A}^{hyd} = \frac{1}{3}tr(\mathbf{A})\mathbf{I} = 2\mathbf{I} = \begin{pmatrix}2&0&0 \\ 0&2&0 \\ 0&0&2\end{pmatrix}$$

该矩阵的偏部分为

$$
\mathbf{A}^{dev} = \begin{pmatrix}-1&2&3 \\ 3&0&0 \\ 2&0&1\end{pmatrix}
$$

### 2.8. 高斯定理

高斯定理也被称为散度定理

体积边界上的通量积分等于体积内的散度积分，即

$$
\int_{\partial{V}} \mathbf{U}\cdot d\mathbf{S} = \int_{V}\nabla\cdot \mathbf{U} dV
$$

## 3. 建议

强烈建议读者亲自动手推导理论公式。

另外，要意识到理论学习和理解是一个反复迭代的过程。一个知识点查阅很多书，一本书看很多次，都是非常正常的现象。

建议读者遇到学习和理解的困难的时候，不要退缩，应广泛查阅，积极讨论，最后形成自己的输出。

## References

[1] The Finite Volume Method in Computational Fluid Dynamics, https://link.springer.com/book/10.1007/978-3-319-16874-6

[2] Computational fluid dynamics : the basics with applications, https://searchworks.stanford.edu/view/2989631

[3] Mathematics, Numerics, Derivations and OpenFOAM®, https://holzmann-cfd.com/community/publications/mathematics-numerics-derivations-and-openfoam-free

[4] Notes on Computational Fluid Dynamics: General Principles, https://doc.cfd.direct/notes/cfd-general-principles/

## 支持我们

>[!tip]
>希望这里的分享可以对坚持、热爱又勇敢的您有所帮助。
>
>如果这里的分享对您有帮助，您的评论或赞助将对本系列以及后续其他系列的更新、勘误、迭代和完善都有很大的意义，这些行动也会为后来的新同学的学习有很大的助益。
>
>赞助打赏时的信息和留言将用于展示和感谢。

{{< cards >}}
  {{< card link="/" title="支持" image="https://www.notion.so/image/attachment%3A3be6af9a-4829-4dfd-997e-641dfd055ba9%3Aalipay.jpg?table=block&id=22cd34b0-7c4c-8086-bdda-d558df1d9a11&t=22cd34b0-7c4c-8086-bdda-d558df1d9a11" subtitle="支付宝" >}}
{{< /cards >}}

> Copyright @ 2026 Aerosand
>
> - 课程（文本、图片等）：[CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/)
> - OpenFOAM 开发代码：[GPL v3](https://www.gnu.org/licenses/gpl-3.0.html)
> - 其他代码：[MIT License](https://opensource.org/licenses/MIT)