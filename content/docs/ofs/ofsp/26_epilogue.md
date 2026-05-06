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
> Visit [https://aerosand.cc](https://aerosand.cc/) for the latest updates.

## 1. This Series

Through continuous trial teaching, review, expansion, revision, and updates, this series has evolved from its initial 10 lectures to the current 26 lectures.

Looking back at the original intention of the series design, the ideal scenario is: although readers may still have some questions (most of which do not hinder the main progression and have been noted for future discussions), the major obstacles have been cleared. Readers now have a holistic and comprehensive understanding of OpenFOAM and solver applications, and are well-prepared to learn and explore OpenFOAM's standard solvers (for which we have also prepared the `ofss` series). Subsequently, readers can read and understand other solvers and modify standard solvers to create solvers that address their own specialized problems.

## 2. Some Q&A

> [!question]
> We still want to ask: are you sure you want to invest time in learning OpenFOAM?


If you do not have a need to modify solvers, learning OpenFOAM may not be necessary. Commercial software provides mature and stable solvers that can solve many problems. Learning OpenFOAM requires a significant time investment, is challenging, and offers low initial returns—a trade-off that needs careful consideration. Additionally, before starting, it is important to determine whether OpenFOAM is truly suitable for solving your specific problems.

> [!question]
> Do I need to dig deep into OpenFOAM's source code?


Learning and using OpenFOAM can be roughly categorized into several levels (though this is not a strict classification):

The first level is directly using OpenFOAM's built-in standard solvers to solve relatively complex problems in one's own field, which may involve more realistic geometries or more complex meshes. It may also involve using more specialized third-party solvers built on OpenFOAM.

Sometimes, however, one finds that the coupled physical fields in their problem are too complex, the motion involved is very unique, or the boundary conditions are special, making it necessary to modify OpenFOAM's standard solvers to achieve the desired objectives. This can be considered the second level of usage.

For some, their expertise lies in algorithm development, numerical scheme research, or solver development, which may require a deeper dive into C++ programming techniques and algorithmic principles, rather than focusing on specific application problems. This represents the third level of usage.

For most who aim to solve complex problems, maintain as much control over problem details as possible, and avoid building a system from scratch, OpenFOAM is one of the better solutions. For such users, the typical usage falls under the second level, with occasional forays into the third level. Overly deep dives into the code can easily lead to getting lost.

After years of code updates, with experts and scholars investing time in writing, implementing, and developing algorithms, OpenFOAM has become quite abstract, intricate, and complex. Using the member data and member methods from C++ declarations while ignoring the tedious details of C++ definitions is essential. Do not strive for perfection; do not aim to dig down to the very bottom, or you may easily get lost in the code, wasting valuable time and detracting from your main objectives.

It is perfectly acceptable to not understand certain parts. Avoid perfectionism.

> [!question]
> Why does it become increasingly difficult to maintain a linear learning progression as one goes deeper?


Honestly, the intention of this series was to provide a step-by-step linear learning path, hoping to chart a linear course that would help beginners transition smoothly and get up to speed as efficiently as possible. However, it must be said that the further you progress, the harder it is to maintain linearity. Too many knowledge points are interconnected and rely on each other for validation. For most people, reading a book two or three times is perfectly normal. It is also common to consult multiple books to cross-reference and understand the same major topic.

Allow confusion to exist. While moving forward, continuously revisit previous topics, gradually resolve your questions, and solidify your understanding of the interconnected knowledge.

## 3. Updates, Corrections, and Acknowledgments

Throughout this series, there are still expressions that need to better balance understanding and rigor. Some discussions are not yet perfectly fluid, and there may still be uncorrected errors that need revision. In the ongoing process of updates and corrections, more details can be added in the future.

As stated at the beginning of this series, this work draws inspiration from, is encouraged by, or is informed by a substantial body of open-source code, literature, and books. As we conclude this series, it is essential to express sincere gratitude and respect to these authors. The following is an incomplete list.

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

It is our sincere hope that this series can be of some help to readers.

## Support us

>[!tip]
>Hopefully, the sharing here can be helpful to you.
>
>If you find this content helpful, your comments or donations would be greatly appreciated. Your support helps ensure the ongoing updates, corrections, refinements, and improvements to this and future series, ultimately benefiting new readers as well.
>
>The information and message provided during donation will be displayed as an acknowledgment of your support.

{{< cards >}}
  {{< card link="/" title="Support" image="https://www.notion.so/image/attachment%3A3be6af9a-4829-4dfd-997e-641dfd055ba9%3Aalipay.jpg?table=block&id=22cd34b0-7c4c-8086-bdda-d558df1d9a11&t=22cd34b0-7c4c-8086-bdda-d558df1d9a11" subtitle="AliPay" >}}
{{< /cards >}}


> Copyright @ 2026 Aerosand
> 
> - Course (text, images, etc.)：[CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/)
> - Code derived from OpenFOAM：[GPL v3](https://www.gnu.org/licenses/gpl-3.0.html)
> - Other code：[MIT License](https://opensource.org/licenses/MIT)


