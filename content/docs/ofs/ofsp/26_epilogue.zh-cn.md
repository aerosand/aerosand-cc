---
uid: 20260321205618
title: 26_epilogue
date: 2026-03-21
update: 2026-03-27
authors:
  - name: Aerosand
    link: https://github.com/aerosand
    image: https://github.com/aerosand.png
tags:
  - OpenFOAM
  - ofsp
  - ofsp2026
excludeSearch: false
toc: true
weight: 27
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

## 1. 本系列

通过不断的试讲、审核、扩充、修订、更新，本系列已经从最初的 10 讲演变成了现在的 26 讲。

回顾系列设计初衷，理想的状态是：虽然读者还有一些困惑（大部分暂时不影响主线推进，均有提及会在以后的分享中继续讨论），但是已经扫除了大障碍，对 OpenFOAM 和求解器应用有了整体和全面的概念，做好了去学习和探索 OpenFOAM 标准求解器（我们也准备了 `ofss` 系列）的准备。之后读者可以阅读和理解其他求解器，并在标准求解器基础上修改得到能解决自己专业问题的求解器。

## 2. 一些QA

> [!question]
>我们还是想问读者是否确定要投入时间学习 OpenFOAM？

如果没有修改求解器的需求，其实不一定要学 OpenFOAM。商业软件提供的求解器成熟且稳定，可以计算解决很多问题。学习 OpenFOAM 的时间久、难度大、前期收益低，需要综合考虑做个取舍。另外，学习前也需要确定 OpenFOAM 是否真的适合解决自己的问题。

> [!question]
> OpenFOAM的源代码需要深挖吗？

OpenFOAM 的学习和使用，不严谨的大概可以分成以下几种：

第一层就是，直接使用 OpenFOAM 自带的标准求解器，用来求解自己专业的较为复杂的问题，可能是更真实的几何，可能是更复杂的网格。也可能是使用 OpenFOAM 更专业第三方求解器等等。

然而有时候会发现自己的专业问题耦合的物理场过于复杂，或者涉及到的运动形式很独特，亦或是边界条件的特殊，所以不得不考虑对 OpenFOAM 标准求解器做一些修改以达成我们的目的。这算是第二层的使用。

还有一些人的专业偏向算法开发、数值格式研究，以及求解器开发，可能就需要更加深入 C++ 代码技巧和算法原理，而不是在具体的问题上。这算是第三层的使用

对于大部分奔着既想解决复杂问题、想尽可能掌控这些问题细节、又想尽可能避免从零开始编程构建系统的人来说，OpenFOAM 是比较好的解决方案之一。对于这部分的使用者来说，更多的情况是第二层的使用，偶尔涉及第三层使用，过度深挖代码很容易迷失方向。

OpenFOAM 的代码更新这么多年，专家学者们投入时间编写、实现和开发算法，已经相当的抽象、深入和复杂。使用 C++ 声明中的成员数据和成员方法，忽略 C++ 定义中繁琐的细节，这些都是十分必要的，不要追求完美，不要追求一挖到底，否则很容易迷失在代码中，从而耽误自己的主要任务和宝贵时间。

允许自己不理解某些内容，不要追求完美主义。

> [!question]
> 为什么越深入越难保持线性化推进学习？

说实话，一步一步线性推进学习是相关系列的初衷，希望能指一条线性路径从而尽可能节约时间使新手过渡流畅的尽快上手。但是必须要说，越后期的学习越难以线性。太多的知识点互相关联，互相验证理解。对于普通人来说，一本书看两三遍是很正常的。在同一个大问题上找了好几本书交叉来看也是很正常的事情。

允许困惑存在，在前进的同时不断回顾，一点一点解决自己的困惑，夯实自己对前前后后知识点的理解。

## 3. 更新勘误和致谢

本系列讨论还有不少表达需要更加兼顾理解和严谨，有一些讨论的处理还不够流畅，可能还有一些未纠正的错误需要修正。在日后不停的更新和勘误的过程中，还有会有更多的细节可以在后续补充。

一如本系列开篇所言，本系列参考、受鼓励或受启发于大量的开源代码、文献和书籍。在收篇时，本系列必须对这些作者们表示由衷的感谢和敬意。以下为不完整列表。

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

由衷希望系列讨论能对读者有一些帮助。

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


