---
uid: 20250722130605
title: 00_ofspIntro
date: 2025-07-22
update: 2026-03-27
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
> Visit [https://aerosand.cc](https://aerosand.cc/) for the latest updates.


## 0. Preface

This series is designed to help readers connect the two sections of "CFD Fundamentals" and "OpenFOAM Introductory Practice".

> [!warning]
> It is recommended to learn the basics of computational fluid mechanics and the finite volume method before starting this series.

## 1. Introduction 

What is OpenFOAM? According to the explanation from Wikipedia:

> OpenFOAM (for "Open-source Field Operation And Manipulation") is a C++ toolbox for the development of customized numerical solvers, and pre-/post-processing utilities for the solution of continuum mechanics problems, most prominently including computational fluid dynamics (CFD).

Therefore, OpenFOAM can be used to build solver applications—implemented in C++—for theories such as computational fluid dynamics (CFD).

This series draws inspiration from, is encouraged by, or is informed by a substantial body of open-source code, literature, and books. Before proceeding, it is essential to express sincere gratitude and respect to these authors. The following is an incomplete list.

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


## 2. Roadmap

We begin with the implementation of a simple C++ program to gain a basic understanding of compilation principles. Through the use of `make`, we gradually take control of our project and transition to exploring the `wmake` approach used in OpenFOAM. From there, we become familiar with the fundamental structure of OpenFOAM programs and progressively delve into the details of solver implementations within OpenFOAM.

{{% steps %}}

### Compilation Principles

1. Compilation of C++ Programs
2. Managing Program Compilation with `make`
3. Managing Program Compilation with `wmake`
4. Building OpenFOAM Applications

### Data Interaction

1. Input and Output
2. Command-Line Arguments

### Basic Classes

1. Time
2. Mesh
3. Field

### Solvers

1. Development Libraries
2. The First Solver

### A First Look at Algorithms

1. SIMPLE, PISO & PIMPLE Algorithms
2. SIMPLE, PISO & PIMPLE Solvers

{{% /steps %}}

> [!note]
> Each section provides detailed code and operation explanations.


## 3. Environment and Tools

Given the operating environment of OpenFOAM, we choose to conduct development and discussions within a Linux system environment, based on OpenFOAM version 2406 (or later). For convenience, we will use Visual Studio Code (VS Code) as our development tool.

> [!caution]
> - The version evolution from openfoam.com involves relatively minor changes, and newer versions remain suitable for the discussions in this series.
> - The version architecture from openfoam.org has undergone significant modifications and is therefore not recommended for beginners.


## 4. Recommendations

> [!tip]
> - It is recommended that readers actively follow along with the programming operations as they are discussed.


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

