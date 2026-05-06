---
uid: 20251006170601
title: 05_generalConservation
date: 2025-10-06
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
weight: 6
math: true
next:
prev:
comments: true
sidebar:
  exclude: false
draft: false
---


## 0. 前言

前文已经完成了连续性方程（质量方程），动量方程和能量方程的推导和讨论。

本文主要讨论

- [ ] 通用方程的推导
- [ ] 不同角度的推导
- [ ] 理解数学表达的物理意义

## 1. 通用方程

通过前文的讨论，可以看到无论是什么物理量总是有雷诺输运定理。

回忆雷诺输运定理有

$$
\bigg(\frac{dB}{dt}\bigg)_{MV} = \int_V\bigg[\frac{\partial}{\partial t}(\rho b) + \nabla \cdot (\rho U b)\bigg]dV = \int_V\bigg[\frac{D}{D t}(\rho b) + \rho b \nabla \cdot U\bigg]dV
$$

对于一个系统，物理量 $\phi$ 总是有

**物质体的物理量变化 A = 经过控制体表面的通量变化 B + 控制体内部的产生/消失 C**


> change of $\phi$ over time $\Delta t$ within the material volume (MV) - A
> 
> $=$
> 
> surface flux of $\phi$ over time $\Delta t$ across the control volume - B
> 
> $+$
> 
> source/sink of $\phi$ over time $\Delta t$ within control volume - C


对于第一项 A 

结合雷诺输运定理有

$$
A = \frac{d}{dt}\bigg(\int_{MV}(\rho\phi )dV\bigg) = \int_{V}\bigg[\frac{\partial}{\partial t}(\rho\phi) + \nabla\cdot(\rho U\phi)\bigg]dV
$$

基于前文速度散度和雷诺定理的讨论，我们可以理解 $\rho U \phi$ 本质上表示流动中物理量 $\phi$ 的输运，称为**对流通量 (convective flux)**。

我们定义**对流通量 (convective flux)**

$$F^{C} = \rho U \phi$$

对于第二项 B

第二项是物理量 $\phi$ 由于控制体表面上的物理现象而发生的变化。

我们另外定义**扩散通量 (diffusion flux)**

$$F^{D} = -\Gamma^{\phi}\nabla \phi$$

有（整理用到散度定理）

$$B = -\int_{\partial V} F^{D}\cdot \vec ndS = -\int_{V} \nabla\cdot F^{D}dV = \int_V\nabla\cdot(\Gamma^{\phi}\nabla\phi)dV$$

> [!tip]
> 通量向内是正号，通量向外是负号。负号也因为扩散的本质是系统的“损耗”。

对于第三项 C

第三项是物理量 $\phi$ 由于控制体体积上的物理现象而发生的变化。

$$C = \int_V S_{\phi}dV$$

综上有

$$\int_{V} \bigg[\frac{\partial}{\partial t}(\rho\phi) + \nabla\cdot(\rho\phi U)\bigg]dV = \int_V\nabla\cdot(\Gamma^{\phi}\nabla\phi)dV  + \int_V S_{\phi}dV$$

我们停下脚步考虑一下输运过程中的物理本质。

对于我们所研究的流体来说，因为对流、扩散的物理现象，总是应该有以下的守恒关系

> 控制体的物理量的增加（正号）
> $=$
> 因对流机制通过控制体边界的物理量流入（与控制体表面方向相反，负号）
> $+$
> 因扩散机制通过控制体边界的物理量流出（与控制体表面方向相同，正号）
> $+$
> 因产生源的物理量变化（正号）

通用守恒方程总结为

$$\frac{\partial}{\partial t}(\rho\phi) + \nabla\cdot(\rho\phi U) - \nabla\cdot(\Gamma^{\phi}\nabla\phi) = S_{\phi}$$

这个表达式的四大项就对应了物理本质的四大项，即时间项（当地时间变化）、对流项（对流变化）、扩散项（基于表面的扩散变化）、源项（基于体积的变化）。

有的读者可能会对繁杂的扩散项、粘性项等讨论感到迷惑，这里需要做一点补充讨论。

上文提到的扩散项是广义扩散项，本质上包含着质量方程的质量扩散、动量方程的动量扩散和能量方程的能量扩散。

前文在讨论质量方程的时候，并没有讨论物质扩散的情况。物质扩散或组分扩散一般偏向化工化学材料专业，以后在遇到的时候会特别讨论。

对比之前分别推导出的 N-S 方程

1. 质量方程

$$\frac{\partial}{\partial t}\rho + \nabla\cdot(\rho U) = 0$$

2. 动量方程

$$\frac{\partial}{\partial t}(\rho U) + \nabla \cdot (\rho UU) = \nabla\cdot(\mu\nabla U)-\nabla p + Q$$

3. 能量方程

$$\frac{\partial}{\partial t}(\rho c_pT) + \nabla\cdot(\rho c_p U T) = \nabla\cdot(k\nabla T) + Q^T$$

总结有通用形式基本方程

$$\frac{\partial}{\partial t}(\rho \phi) + \nabla \cdot (\rho U\phi) = \nabla\cdot(\Gamma\nabla\phi) + S_{\phi}$$

从左至右，我们依次称为：时间项（瞬态项）、对流项、扩散项、源项。

## 2. 小结

到此为止，流体动力学的基本方程已经大体讨论结束。

希望读者能快速的跨过计算流体动力学的这第一道坎。

本文完成讨论

- [x] 通用方程的推导
- [x] 不同角度的推导
- [x] 理解数学表达的物理意义


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