---
uid: 20250722130605
title: 00_ofspIntro
date: 2025-07-22
update: 2026-03-28
authors:
  - name: Aerosand
    link: https://github.com/aerosand
    image: https://github.com/aerosand.png
tags:
  - ofsp2026
  - OpenFOAM
  - ofsp
excludeSearch: false
toc: true
weight: 1
math: true
next:
prev:
comments: true
draft: false
category:
  - ofsp2026
---

> [!important]
> 访问 [https://aerosand.cc](https://aerosand.cc/) 以获取最近更新。  



## 0. 前言

本系列旨在帮助读者衔接“CFD 基础理论”和“OpenFOAM 入门实践”两个部分。

> [!warning]
> 建议先学习计算流体力学基础以及有限体积法，之后再开始本系列的学习。

## 1. 介绍

OpenFOAM 是什么呢？引用 wiki 解释如下

> OpenFOAM (for "Open-source Field Operation And Manipulation") is a C++ toolbox for the development of customized numerical solvers, and pre-/post-processing utilities for the solution of continuum mechanics problems, most prominently including computational fluid dynamics (CFD).

所以我们可以使用 OpenFOAM 来构建基于 C++ 的实现 CFD 等理论的求解器应用。

本系列参考、受鼓励或受启发于大量的开源代码、文献和书籍。在开始前，本系列必须对这些作者们表示由衷的感谢和敬意。以下为不完整列表。

- The Finite Volume Method in Computational Fluid Dynamics: An Advanced Introduction with OpenFOAM® and Matlab
- https://github.com/UnnamedMoose/BasicOpenFOAMProgrammingTutorials
- https://www.topcfd.cn/simulation/solve/openfoam/openfoam-program/
- https://www.tfd.chalmers.se/~hani/kurser/OS_CFD/
- https://github.com/ParticulateFlow/OSCCAR-doc/blob/master/openFoamUserManual_PFM.pdf
- https://www.youtube.com/watch?v=KB9HhggUi_E&ab_channel=UCLOpenFOAMWorkshop
- http://dyfluid.com/#
- https://journal.openfoam.com/index.php/ofj/issue/archive
- https://www.wolfdynamics.com/our-services/training/openfoam-intro-training.html?id=50
- https://www.fluiddynamics.at/downloads/basic-openfoam-programming


## 2. 路线

我们从简单的 C++ 程序实现开始，简单了解编译原理，通过 make 逐渐掌控我们的项目，过渡到了解 OpenFOAM 的 wmake 实现方式，然后认识 OpenFOAM 的基本程序，然后逐渐深入了解 OpenFOAM 的求解器应用细节。

{{% steps %}}

### 编译原理

1. C++ 程序的编译
2. make 管理程序编译
3. wmake 管理程序编译
4. OpenFOAM 应用构建

### 数据交互

1. 输入输出
2. 命令行参数

### 基础类

1. 时间
2. 网格
3. 场

### 求解器

1. 开发库
2. 第一个求解器

### 算法初见

1. SIMPLE & PISO & PIMPLE 算法
2. SIMPLE & PISO & PIMPLE 求解器

{{% /steps %}}

> [!note]
> 每个部分都会有详细的代码和操作解释。


## 3. 环境和工具

鉴于 OpenFOAM 的使用环境，我们选择在 <mark style="background: #FFB86CA6;">ubuntu 24.04</mark> 系统环境中，基于 <mark style="background: #FFB86CA6;">OpenFOAM 2406</mark> 版本进行开发讨论，方便起见使用 vscode 工具。

> [!caution]
> - openfoam.com 的版本变化较小，较新的版本均适合本系列讨论使用
> - openfoam.org 的版本架构大改，暂不推荐入门


## 4. 建议

> [!tip]
> - 建议读者动手跟随讨论编程操作



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
> - OpenFOAM 开发代码 ：[GPL v3](https://www.gnu.org/licenses/gpl-3.0.html)
> - 其他代码：[MIT License](https://opensource.org/licenses/MIT)

