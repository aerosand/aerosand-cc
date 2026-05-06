---
uid: 20260126161556
title: "21_PISO&PIMPLE"
date: 2026-01-26
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
weight:
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

前文讨论了求解稳态问题的 SIMPLE 算法，本文讨论求解瞬态问题的 PISO 和 PIMPLE 算法。

本文主要讨论

- [ ] PISO 理论和实现
- [ ] 松弛理论和实现
- [ ] PIMPLE 理论和实现

## 1. 控制方程

考虑无重力瞬态不可压缩流动的 NS 方程

连续方程（质量方程）

$$
\nabla\cdot U = 0
$$

描述了粘性力的动量方程

$$
\frac{\partial}{\partial t}(\cancel{\rho} U) + \nabla \cdot (\cancel{\rho} UU) = - \frac{1}{\rho} \nabla p + \frac{1}{\rho} \nabla\cdot\vec{\tau} + \cancel{\rho\vec{g}}
$$

参考前文关于压力项和粘度项的密度处理，最终有

控制方程为

连续方程（质量方程）

$$
\nabla\cdot U = 0
$$

描述了粘性力的动量方程

$$
\frac{\partial U}{\partial t} + \nabla \cdot ( UU) = -  \nabla p + (\nabla\cdot\vec{\tau})
$$


## 2. PISO

在 OpenFOAM 中，PISO（**P**ressure-**I**mplicit with **S**plitting of **O**perators）算法用于求解瞬态问题。

### 2.1. 理论

第一步

基于上一步的压力场和速度场（非线性的一部分）（启动步使用初始压力场和初始速度场），进行 momentum predictor 

$$
\frac{\partial U}{\partial t} + \nabla \cdot ( UU) = -  \nabla p + (\nabla\cdot\vec{\tau})
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

第四步（内循环）

基于新的压力和新的速度作为已知量，回到第二步，再次求解压力修正方程，通过修正压力再次修正速度，形成迭代，直到修正速度和修正压力满足要求。

第五步（时间推进）

由内循环迭代得到的速度场和压力场作为旧时间步的结果，进入下一个时间步的计算。


>[!tip] 
>这个过程也被称为 inner loop

### 2.2. 实现

我们对比 OpenFOAM 原生 pisoFoam 求解器，看看 PISO 算法的代码实现应该是什么样子的。

终端输入命令，参看当前版本的 pisoFoam 求解器

```terminal {fileName="terminal"}
sol
code incompressible/pisoFoam
```


避免被复杂代码迷惑，我们参考历史版本的 OpenFOAM 代码，链接如下

https://github.com/OpenFOAM/OpenFOAM-3.0.x/tree/master/applications/solvers/incompressible/pisoFoam

代码对应分析如下

第一步

动量预测，即求解动量方程，得到预测速度，也就是代码中的 momentum predictor 部分。

$$
\frac{\partial U}{\partial t} + \nabla \cdot ( UU) = -  \nabla p + (\nabla\cdot\vec{\tau})
$$

为了方便理解，我们摘抄主要代码如下

```cpp {fileName="UEqn.H",linenos=table,linenostart=1}
// Solve the Momentum equation

...

fvVectorMatrix UEqn // 构建动量方程的未知量矩阵
(
    fvm::ddt(U) // 瞬态项
  + fvm::div(phi, U) // 对流项
  // ...
  + turbulence->divDevReff(U) // 湍流粘性扩散项
 ==
    fvOptions(U)
);

UEqn.relax(); // 矩阵松弛，后面将讨论

...

if (piso.momentumPredictor()) // 可选开启动量预测，后续会讨论
{
    solve(UEqn == -fvc::grad(p)); // 求解动量方程

    ...
}
```

> [!warning]
> 如果不开启动量预测，岂不是丢失了求解动量方程这一步？后续将讨论。


第二步

压力修正，即求解连续方程，得到修正后的压力，也就是 `pEqn.H` 中的 pressure corrector loop 部分。

第三步

速度修正，即基于修正后的压力修正速度，也就是 `pEqn.H` 中速度更新部分。

为了方便理解，我们摘抄主要代码如下

```cpp {fileName="pEqn.H",linenos=table,linenostart=1}
volScalarField rAU(1.0/UEqn.A());
volVectorField HbyA("HbyA", U);
HbyA = rAU*UEqn.H(); // 更新HbyA
surfaceScalarField phiHbyA // HbyA的通量
(
    "phiHbyA",
    (fvc::interpolate(HbyA) & mesh.Sf())
  + fvc::interpolate(rAU)*fvc::ddtCorr(U, phi)
);

...

// Non-orthogonal pressure corrector loop
while (piso.correctNonOrthogonal())
{
    // Pressure corrector

    fvScalarMatrix pEqn // 构建压力方程
    (
        fvm::laplacian(rAU, p) == fvc::div(phiHbyA)
    );

	...

	// 求解压力修正方程
    pEqn.solve(mesh.solver(p.select(piso.finalInnerIter())));

	...
}

#include "continuityErrs.H"

U = HbyA - rAU*fvc::grad(p); // 速度修正
...
```

第四步

求解压力，速度修正，内部循环满足要求，之后才时间推进，重新求解动量方程，进入下一个大循环。

为了方便理解，我们摘抄主要代码如下

```cpp {fileName="pisoFoam.C",linenos=table,linenostart=1}
while (runTime.loop()) // 内循环结束后，时间推进，进入下一个外循环
{
	Info<< "Time = " << runTime.timeName() << nl << endl;

	...

	// Pressure-velocity PISO corrector
	{
		#include "UEqn.H"

		// --- PISO loop // PISO inner loop 内循环
		while (piso.correct())
		{
			#include "pEqn.H"
		}
	}

	...

	runTime.write();

	Info<< "ExecutionTime = " << runTime.elapsedCpuTime() << " s"
		<< "  ClockTime = " << runTime.elapsedClockTime() << " s"
		<< nl << endl;
}
```


## 3. 亚松弛

### 3.1. 理论

SIMPLE 算法最一开始用来计算稳态流动，也就是没有时间瞬态项。

如果有时间瞬态项，当时间步长很小的时候，离散后的时间瞬态项将比其他项大的多得多，此时也会主导系数矩阵的对角元素。

越小的时间步长，系数矩阵越占优。对角占优在物理上意味着有限体积单元的本单元影响大于邻单元的影响。对角占优在数学上保证了系数矩阵的可逆性，保证了迭代法的可用，也保证数值误差不会被放大。

而对于稳态问题，没有瞬态问题中对角占优的优势，所以为了达到对角占优，求解器需要使用亚松弛来增加对角占优性质，从而提高计算稳定性。

> [!note]
> 对离散方程有困惑的，需要复习有限体积法的基础知识。

参考前文 07_fvm-basics ，对于一般的物理场 $\phi$ 来说，离散后的控制方程如下

$$
a_{P}\phi_{P}+\sum\limits_{N}a_{N}\phi_{N} = R_{P}
$$

增加相同的人工项，离散方程依然为

$$
\frac{1-\alpha}{\alpha}a_{P}\phi_{P} + a_{P}\phi_{P} + \sum\limits a_{N}\phi_{N} - \frac{1-\alpha}{\alpha}a_{P}\phi_{P} = R_{P}
$$

离散方程此时为

$$
\frac{1}{\alpha}a_{P}\phi_{P} + \sum\limits a_{N}\phi_{N} - \frac{1-\alpha}{\alpha}a_{P}\phi_{P} = R_{P}
$$

对于稳态计算，当稳态计算收敛的时候，我们认为右侧人工项的新迭代步和旧迭代步的场量几乎相等，即

$$
\phi_{P}^{old-iteration} = \phi_{P}
$$

整理后的离散方程也就是

$$
\frac{1}{\alpha}a_{P}\phi_{P} + \sum\limits a_{N}\phi_{N} = R_{P} + \frac{1-\alpha}{\alpha}a_{P}\phi_{P}^{old-iteration}
$$

一般情况下，松弛因子 $0 < \alpha < 1$ ，所以对于求解代数系统的系数矩阵，其对角元素增大了，也就是增加了对角占优性质，也就是增加了计算稳定性。

注意到，对于上述松弛后的方程，在数值求解上，仍然约等于求解原方程。

### 3.2. 实现

我们有动量方程如下

$$
MU = AU - H  = \underbrace{-\nabla p}_{souce}
$$

> [!tip]
> 注意 $A$ 本就是对角矩阵 `UEqn.A()`，而 $H$ 就是非对角矩阵 `UEqn.H()` 。

增加人工项之后为

$$
\frac{1-\alpha }{\alpha} AU + AU - H  = \underbrace{-\nabla p}_{old-source} + \frac{1-\alpha }{\alpha} AU
$$

整理为

$$
\frac{1}{\alpha} AU - H  = \underbrace{-\nabla p + \frac{1-\alpha }{\alpha} AU}_{new-source}
$$

松弛后的方程为

$$
A^{relax} U - H = \underbrace{- \nabla p + (A^{relax} - A)U}_{new-source}
$$

其中

$$
A^{relax} = \frac{1}{\alpha}A
$$

在求解器中，动量方程的待求量矩阵会有如下松弛处理

```cpp
tmp<fvVectorMatrix> tUEqn
(
	...
);
fvVectorMatrix& UEqn = tUEqn.ref();

UEqn.relax();
```


通过官方 API 网站查阅 `fvMatrix.C` 的代码

API https://api.openfoam.com/2512/fvMatrix_8C_source.html

为了方便理解，我们摘抄主要代码如下

```cpp {fileName="fvMatrx.C",linenos=table,linenostart=1}
 template<class Type>
 void Foam::fvMatrix<Type>::relax(const scalar alpha)
 {
     if (alpha <= 0) // 如果松弛因子小于零，则直接返回
     {
         return;
     }
 
     DebugInFunction // psi是构建矩阵中的未知量
         << "Relaxing " << psi_.name() << " by " << alpha << endl;
 
     Field<Type>& S = source(); // 源项
     scalarField& D = diag(); // 对角项
 
     // Store the current unrelaxed diagonal for use in updating the source
     scalarField D0(D); // 未松弛处理的对角项
 
	 ...
 
     // ... then relax
     D /= alpha; // 松弛对角矩阵
 
	...
 
     // Finally add the relaxation contribution to the source.
     S += (D - D0)*psi_.primitiveField(); // 松弛源项
 }
```


> [!tip]
> 注意到，此处暂且仅讨论了离散方程的矩阵亚松弛。关于物理场量的松弛处理，后续会讨论。


## 4. PIMPLE

对于 PISO 算法来说，时间步较大的时候，计算容易发散。此时可以在每个时间步内添加松弛，进行迭代收敛。

依然有控制方程如下

连续方程（质量方程）

$$
\nabla\cdot U = 0
$$

描述了粘性力的动量方程

$$
\frac{\partial U}{\partial t} + \nabla \cdot ( UU) = -  \nabla p + (\nabla\cdot\vec{\tau})
$$

### 4.1. 理论

第一步

基于上一步的压力场和速度场（非线性的一部分）（启动步使用初始压力场和初始速度场），进行 momentum predictor 

$$
\frac{\partial U}{\partial t} + \nabla \cdot ( UU) = -  \nabla p + (\nabla\cdot\vec{\tau})
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

第四步（内循环）

基于新的压力和新的速度作为已知量，回到第二步，再次求解压力，得到修正速度，形成迭代，直到满足要求后。此循环类似于 PISO 算法的循环。

第五步（外循环）

基于迭代修正后的速度场和压力场，返回第一步，重新求解动量方程，直到满足要求。此大循环类似于 SIMPLE 算法的循环。

第六步（时间推进）

经过内循环和外循得到的速度场和压力场作为旧时间步的结果，进入下一个时间步的计算。

### 4.2. 实现

我们对比 OpenFOAM 原生 pimpleFoam 求解器，看看 PIMPLE 算法的代码实现应该是什么样子的。

终端输入命令，参看当前版本的 pimpleFoam 求解器

```terminal {fileName="terminal"}
sol
code incompressible/pimpleFoam
```


避免被复杂代码迷惑，我们依然参考历史版本的 OpenFOAM 代码，链接如下

https://github.com/OpenFOAM/OpenFOAM-4.x/tree/master/applications/solvers/incompressible/pimpleFoam

代码对应分析如下

第一步

动量预测，即求解动量方程，得到预测速度，也就是代码中的 momentum predictor 部分。

$$
\frac{\partial U}{\partial t} + \nabla \cdot ( UU) = -  \nabla p + (\nabla\cdot\vec{\tau})
$$

为了方便理解，我们摘抄主要代码如下

```cpp {fileName="UEqn.H",linenos=table,linenostart=1}
// Solve the Momentum equation

...

tmp<fvVectorMatrix> tUEqn // 构建动量方程的未知量矩阵
(
    fvm::ddt(U) // 瞬态项
  + fvm::div(phi, U) // 对流项
  // + MRF.DDt(U)
  + turbulence->divDevReff(U) // 湍流粘性扩散项
 ==
    fvOptions(U)
);
fvVectorMatrix& UEqn = tUEqn.ref(); // 动量方程矩阵的引用

UEqn.relax(); // 矩阵松弛

...

if (pimple.momentumPredictor()) // 可选开启动量预测，后续会讨论
{
    solve(UEqn == -fvc::grad(p)); // 求解动量方程

    fvOptions.correct(U); // 以后会讨论
}
```

第二步

压力修正，即求解连续方程，得到修正后的压力，也就是 `pEqn.H` 中的 pressure corrector loop 部分。

第三步

速度修正，即基于修正后的压力修正速度，也就是 `pEqn.H` 中速度更新部分。

为了方便理解，我们摘抄主要代码如下

```cpp {fileName="pEqn.H",linenos=table,linenostart=1}
volScalarField rAU(1.0/UEqn.A());
volVectorField HbyA(constrainHbyA(rAU*UEqn.H(), U, p)); // HbyA
surfaceScalarField phiHbyA // HbyA的通量
(
    "phiHbyA",
    fvc::flux(HbyA) //主要贡献
  + fvc::interpolate(rAU)*fvc::ddtCorr(U, phi) //数值考虑
);

...

tmp<volScalarField> rAtU(rAU); // rAU的临时副本

...

// Non-orthogonal pressure corrector loop
while (pimple.correctNonOrthogonal())
{
    // Pressure corrector
    fvScalarMatrix pEqn // 压力修正方程
    (
        fvm::laplacian(rAtU(), p) == fvc::div(phiHbyA)
    );

	...

	// 压力方程求解
    pEqn.solve(mesh.solver(p.select(pimple.finalInnerIter())));

    if (pimple.finalNonOrthogonalIter())
    {
        phi = phiHbyA - pEqn.flux(); // HbyA的通量
    }
}

#include "continuityErrs.H"

// Explicitly relax pressure for momentum corrector
p.relax(); // 压力松弛

U = HbyA - rAtU()*fvc::grad(p); // 修正速度
...
```

第四步

求解压力，速度修正，内部循环满足要求，之后才时间推进，重新求解动量方程，进入下一个大循环。

```cpp {fileName="pisoFoam.C",linenos=table,linenostart=1}
while (runTime.run()) // 时间推进
{
	...

	runTime++;

	Info<< "Time = " << runTime.timeName() << nl << endl;

	// --- Pressure-velocity PIMPLE corrector loop
	while (pimple.loop()) // 外循环
	{
		#include "UEqn.H"

		// --- Pressure corrector loop
		while (pimple.correct()) // 内循环
		{
			#include "pEqn.H"
		}

		...
	}

	runTime.write();

	Info<< "ExecutionTime = " << runTime.elapsedCpuTime() << " s"
		<< "  ClockTime = " << runTime.elapsedClockTime() << " s"
		<< nl << endl;
}
```


## 5. 小结

我们一起讨论了 PISO 算法，以及 PISO 算法在 OpenFOAM 中的代码实现。在 PISO 算法和 SIMPLE 算法的比较中，讨论了矩阵的亚松弛。最后讨论了 PIMPLE 算法，以及 PIMPLE 算法在 OpenFOAM 中的代码实现。

通过讨论可以看到，如果 PIMPLE 算法的外循环次数为 1 ，则 PIMPLE 算法和 PISO 算法完全一样。

本文完成讨论

- [x] PISO 理论和实现
- [x] 松弛理论和实现
- [x] PIMPLE 理论和实现



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
