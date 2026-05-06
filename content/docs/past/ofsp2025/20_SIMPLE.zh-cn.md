---
uid: 20251202115950
title: 20_SIMPLE
date: 2025-12-02
update: 2026-02-04
authors:
  - name: Aerosand
    link: https://github.com/aerosand
    image: https://github.com/aerosand.png
tags:
  - OpenFOAM
  - ofsp
  - ofsp2025
excludeSearch: false
toc: true
weight: 21
math: true
next:
prev:
comments: true
sidebar:
  exclude: false
draft: false
---

> [!important]
> 访问 https://aerosand.cn 以获取最近更新。

## 0. 前言

我们回忆计算流体力学的基本控制方程 https://aerosand.cc/docs/cfd/cfdb/05_general-conservation/


1. 质量方程

$$\frac{\partial}{\partial t}\rho + \nabla\cdot(\rho U) = 0$$

2. 动量方程

$$\frac{\partial}{\partial t}(\rho U) + \nabla \cdot (\rho UU) = \nabla\cdot(\mu\nabla U)-\nabla p + Q$$

3. 能量方程

$$\frac{\partial}{\partial t}(\rho c_pT) + \nabla\cdot(\rho c_p U T) = \nabla\cdot(k\nabla T) + Q^T$$

总结有通用形式基本方程

$$\frac{\partial}{\partial t}(\rho \phi) + \nabla \cdot (\rho U\phi) = \nabla\cdot(\Gamma\nabla\phi) + S_{\phi}$$


本文主要讨论

- [ ] 压力速度耦合方程
- [ ] SIMPLE 算法
- [ ] SIMPLE 代码框架



## 1. NS 方程

### 1.1. 无重力稳态不可压

考虑无重力稳态不可压缩流动的 NS 方程

连续方程（质量方程）

$$
\nabla\cdot U = 0
$$

描述了粘性力的动量方程

$$
\cancel{\frac{\partial}{\partial t}(\rho U)} + \nabla \cdot (\cancel{\rho} UU) = - \frac{1}{\rho} \nabla p + \frac{1}{\rho} \nabla\cdot\vec{\tau} + \cancel{\rho\vec{g}}
$$

在 OpenFOAM 的不可压缩求解器中，压力 $p$ 实际上是动压头，隐含的已经除以了密度，即实际上

$$
p[m^{2}.s^{-2}] = \frac{P[Pa]}{\rho[kg.m^{-3}]}
$$

其中，$P$ 是物理压力，此时的 $p$ 的单位为 $[m^{2}.s^{-2}]$ 

类似的，在 OpenFOAM 不可压缩求解器中，粘性力项也默认除以了密度，用运动粘度 `nu` 代替了动力粘度 `mu` ，即

$$
\nu[m^{2}.s^{-1}] = \frac{\mu[Pa.s]}{\rho[kg.m^{-3}]}
$$

此时，我们知道在 OpenFOAM 中使用的控制方程为

连续方程（质量方程）

$$
\nabla\cdot U = 0
$$

动量方程为

$$
\nabla \cdot (UU) = - \nabla p + \nabla\cdot\vec{\tau}
$$

对于此控制方程，在求解中面临两个问题：

- $\nabla\cdot(UU)$ 的非线性
- 压力和速度的耦合

我们需要一定的算法来求解此方程组。

### 1.2. 非线性

回忆之前的对流项，物理量 $\phi$ 由质量通量 $F$ 的影响而产生对流。而这里相当于是自己输运自己（速度“输运”速度）。

现在的对流项，作体积积分后离散

$$
\begin{aligned}
\int_{V_{P}}\nabla\cdot(UU)dV &= \int_{\partial V_{P}}(UU)\cdot d S \\
&= \sum\limits_{f}\int_{f}(UU) \cdot d S\\
&\approx \sum\limits_{f}\underbrace{(UU)_{f}}_{nonlinear}\cdot S_{f} \\
&=  \sum\limits_{f}F^{C}_{f}\cdot S_{f} \\
&= a_{P}U_{P} + \sum\limits_{f}a_{N}U_{N}
\end{aligned}
$$

对于其中的非线性部分 $(UU)$，实践中更倾向于线性处理，也就是令其中一个速度使用上一时间步的值作为已知值。当然，这样会导致计算过程中，非线性项存在一个延迟。不过这对于稳态计算来说并不重要。

对于瞬态计算来说，仍然可以忽略非线性延迟而作一个线性化处理，也可以单独对非线性项作迭代进行处理。但是，单独作迭代会增加计算消耗。

当时间步很小的时候，连续解之间的差别就会很小，此时非线性延迟也就不再显著。

实践中，一般使用 PISO 压力-速度耦合算法来计算瞬态流动计算，使用 SIMPLE 压力-速度耦合算法来计算稳态流动计算。

### 1.3. 压力速度耦合

动量方程的形式可以简化为

$$
MU = -\nabla p
$$

可以想见场的系数矩阵 $M$ 是一个对角占优的稀疏方阵（或者说我们希望它能够对角占优），$M$ 矩阵的对角线都是离散方程中的本元素，同一行上，对角线前后（在该行也许不紧挨）的元素都是和该单元相邻的单元。

最基本的思路是，我们最开始假设一个初始压力场，根据动量方程求出速度场。直接从动量方程求出来的速度场，再去考虑满足质量方程的约束。

那，怎么从动量方程求出速度场呢？

我们先处理系数矩阵 $M$ 。

从 $M$ 矩阵中分解出对角矩阵 $A$ （不是求解线性代数系统 $Ax=b$ 中的 $A$），对角矩阵 $A$ 可以很容易求出逆矩阵。

OpenFOAM 中的对角矩阵以及求解方法已经高度抽象化，也更易读，如下所示

```cpp
// 对角矩阵的倒数
volScalarField rAU(1.0/UEqn.A());
```

动量方程的左侧抽离对角矩阵，可以写成

$$
MU = AU - H
$$

相应的非对角矩阵为

$$
H = AU - MU
$$

OpenFOAM 中也提供可以直接调用的成员方法 `UEqn.H()` 。

作为比较，注意以下处理是错误的。

$$
\cancel{MU = AU - HU}
$$

分解后的动量方程为

$$
AU - H = -\nabla p
$$

两边同时乘以 $A$ 的逆矩阵

$$
A^{-1}AU = A^{-1}H -A^{-1}\nabla p
$$

可以得到【预测速度方程】

$$
U = A^{-1}H -A^{-1}\nabla p
$$

上面求出的速度场并不是最终结果，我们称之为预测速度。预测速度场还需要满足连续方程。

预测速度带入连续方程

$$
\nabla\cdot U = 0
$$

即有

$$
\nabla\cdot(A^{-1}H -A^{-1}\nabla p) = 0
$$

进一步整理得到【压力修正方程】

$$
\nabla\cdot(A^{-1}\nabla p) = \nabla\cdot(A^{-1}H)
$$

OpenFOAM 中对应的代码在 `pEqn.H` 中

>[!note]
>摘取自早期版本 OpenFOAM-2.2，方便理解和阅读。

```cpp
// OpenFOAM
volScalarField rAU(1.0/UEqn().A());
volVectorField HbyA("HbyA", U);
HbyA = rAU*UEqn().H();
surfaceScalarField phiHbyA("phiHbyA", fvc::interpolate(HbyA) & mesh.Sf());

fvScalarMatrix pEqn
(
	fvm::laplacian(rAU, p) == fvc::div(phiHbyA)
);

pEqn.solve();
```

求解压力修正方程后，可以得到新的压力场。

>[!warning]
>在代码中，实际参与方程计算的为什么是是 $phiHbyA$ ，而不是 $HbyA$ ？
>
>暂且记下，后续会继续讨论。 

## 2. SIMPLE

在 OpenFOAM 中，SIMPLE（**S**emi-**I**mplicit **M**ethod for **P**ressure **L**inked **E**quations）算法用于求解稳态问题。

### 2.1. 理论

第一步

基于上一步的压力场和速度场（非线性的一部分）（启动步使用初始压力场和初始速度场），进行 momentum predictor 

$$
\nabla \cdot (UU) = - \nabla p + \nabla\cdot\vec{\tau}
$$

求解方程，可得预测速度。

第二步

求解连续方程，也即求解压力修正方程

先更新 H 矩阵

$$
H = AU - MU
$$

再求解压力

$$
\nabla\cdot(A^{-1}\nabla p) = \nabla\cdot(A^{-1}H)
$$

第三步

以新的压力场，修正速度场

$$
U = A^{-1}H -A^{-1}\nabla p
$$

第四步（外循环）

基于新的压力和新的速度作为已知量，回到第一步，再次求解动量方程，再次求得速度场，形成循环迭代。

>[!tip] 
>这个过程也被称为 outer loop

### 2.2. 实现

我们对比 OpenFOAM 原生 simpleFoam 求解器，看看 SIMPLE 算法的代码实现应该是什么样子的。

终端输入命令，参看当前版本的 simpleFoam 求解器

```terminal {fileName="terminal"}
sol
code incompressible/simpleFoam
```


避免被复杂代码迷惑，我们参考历史版本的 OpenFOAM 代码，链接如下

https://github.com/OpenFOAM/OpenFOAM-2.2.x/tree/master/applications/solvers/incompressible/simpleFoam

代码对应分析如下

第一步

动量预测，即求解动量方程，得到预测速度，也就是 `UEqn.H` 中的 momentum predictor 部分。

$$
\nabla\cdot UU - \nabla\cdot(\mu\nabla U) = -\nabla p
$$

为了方便理解，我们摘抄主要代码如下

```cpp {fileName="UEqn.H",linenos=table,linenostart=1}
    // Momentum predictor 

    tmp<fvVectorMatrix> UEqn // 构建动量方程的未知量矩阵
    (
        fvm::div(phi, U) // 对流项
      + turbulence->divDevReff(U) // 对应湍流模型下的粘性力扩散项
      ==
        fvOptions(U)
    );
    
	UEqn().relax(); // 松弛处理，后续会讨论
	...
    solve(UEqn() == -fvc::grad(p)); // 求解动量方程
	...
```

> [!warning]
> 注意湍流粘性力项在左边，而且是正号，为什么不是负号呢？
> 暂且记下，以后会讨论。

第二步

压力修正，即求解连续方程，得到修正后的压力，也就是 `pEqn.H` 中的 pressure corrector loop 部分。

第三步

速度修正，即基于修正后的压力修正速度，也就是 `pEqn.H` 中的 momentum corrector 部分。

为了方便理解，我们摘抄主要代码如下

```cpp {fileName="pEqn.H",linenos=table,linenostart=1}
{   // 第二步
    volScalarField rAU(1.0/UEqn().A());
    volVectorField HbyA("HbyA", U);
    HbyA = rAU*UEqn().H(); // 更新HbyA
	...

    surfaceScalarField phiHbyA("phiHbyA", fvc::interpolate(HbyA) & mesh.Sf()); // HbyA的通量
	...

    // Non-orthogonal pressure corrector loop
    while (simple.correctNonOrthogonal()) // 非正交修正
    {
        fvScalarMatrix pEqn // 构建压力方程
        (
            fvm::laplacian(rAU, p) == fvc::div(phiHbyA)
            // 使用phiHbyA而非直接使用HbyA
        );
		...
        pEqn.solve(); // 求解压力修正方程
		...
    }

	...
	p.relax(); // 松弛处理，后续会讨论

    // Momentum corrector
    U = HbyA - rAU*fvc::grad(p); // 第三步，速度修正
	...
}
```


第四步

循环迭代，即使用修正后的压力和速度，重新回到第一步，再次求得速度场，形成迭代，也就是 
simple.loop() 部分。

为了方便理解，我们摘抄主要代码如下

```cpp {fileName="simpleFoam.C",linenos=table,linenostart=1}
    while (simple.loop()) // 算法主框架，进行 outer loop
    {
        Info<< "Time = " << runTime.timeName() << nl << endl;

        // --- Pressure-velocity SIMPLE corrector
        {
            #include "UEqn.H"
            #include "pEqn.H"
        }

        ...

        runTime.write();

        Info<< "ExecutionTime = " << runTime.elapsedCpuTime() << " s"
            << "  ClockTime = " << runTime.elapsedClockTime() << " s"
            << nl << endl;
    }
```

## 3. 小结

本文介绍了压力速度耦合方程的求解问题，并讨论了 SIMPLE 算法的理论和实现。

通过讨论，我们明白了 SIMPLE 算法的主要思路，不过我们还是有一些问题未解答

- 为什么使用 $phiHbyA$ 而不是 $HbyA$ ？
- 为什么湍流粘性扩散项在代码中的符号是正号？

这些问题会在以后再次讨论，无需担心。读者也可先行探索思考。

本文完成讨论

- [x] 压力速度耦合方程
- [x] SIMPLE 算法
- [x] SIMPLE 代码框架


## 支持我们

>[!tip]
>希望这里的分享可以对坚持、热爱又勇敢的您有所帮助。 
>
>如果这里的分享对您有帮助，您的评论、转发和赞助将对本系列以及后续其他系列的更新、勘误、迭代和完善都有很大的意义，这些行动也会为后来的新同学的学习有很大的助益。 
>
>赞助打赏时的信息和留言将用于展示和感谢。

{{< cards >}}
  {{< card link="/" title="支持我们" image="https://www.notion.so/image/attachment%3A3be6af9a-4829-4dfd-997e-641dfd055ba9%3Aalipay.jpg?table=block&id=22cd34b0-7c4c-8086-bdda-d558df1d9a11&t=22cd34b0-7c4c-8086-bdda-d558df1d9a11" subtitle="支付宝AliPay" >}}
{{< /cards >}}
