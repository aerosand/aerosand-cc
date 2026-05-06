---
uid: 
title: ofsp2024-simple-piso-pimple
date: 2025-11-04
update: 2025-11-12
authors:
  - name: Aerosand
    link: https://github.com/aerosand
    image: https://github.com/aerosand.png
tags:
  - ofsp
  - ofsp2024
  - OpenFOAM
excludeSearch: false
toc: true
weight: 10
math: true
next:
prev:
comments: true
sidebar:
  exclude: false
draft: false
---

> 参考：
> - https://github.com/UnnamedMoose/BasicOpenFOAMProgrammingTutorials
> - https://www.topcfd.cn/simulation/solve/openfoam/openfoam-program/
> - https://www.tfd.chalmers.se/~hani/kurser/OS_CFD/
> - https://github.com/ParticulateFlow/OSCCAR-doc/blob/master/openFoamUserManual_PFM.pdf
> - https://www.youtube.com/watch?v=KB9HhggUi_E&ab_channel=UCLOpenFOAMWorkshop
> - http://dyfluid.com/#
> - https://ss1.xrea.com/penguinitis.g1.xrea.com/study/OpenFOAM/index.html
> - https://www.youtube.com/watch?v=OOILoJ1zuiw&t=33s
> - https://www.youtube.com/watch?v=ahdW5TKacok&t=2054s
> 
> 感谢原作者们的无私引路和宝贵工作。


我们首先回忆一下基本方程 （[CFD理论基础00 基本方程 | 𝓐𝓮𝓻𝓸𝓼𝓪𝓷𝓭 (aerosand.cn)](https://aerosand.cn/posts/20230529141915/)）

基本方程为

1. 质量方程

$$\frac{\partial}{\partial t}\rho + \nabla\cdot(\rho U) = 0$$

2. 动量方程

$$\frac{\partial}{\partial t}(\rho U) + \nabla \cdot (\rho UU) = \nabla\cdot(\mu\nabla U)-\nabla p + \vec Q_V$$

3. 能量方程

$$\frac{\partial}{\partial t}(\rho c_pT) + \nabla\cdot(\rho c_p U T) = \nabla\cdot(k\nabla T) + Q^T$$

通用形式

$$\frac{\partial}{\partial t}(\rho \phi) + \nabla \cdot (\rho U\phi) = \nabla\cdot(\Gamma\nabla\phi) + S_{\phi}$$

实际问题可能要求解多个方程，而方程组的求解，即使是数值求解也很困难。

## 压力速度修正算法

对于不可压流动，密度恒定，压力速度耦合控制方程为（压力是除以密度的相对压力）

$$
\nabla\cdot U = 0 \tag{1} \\
$$

$$
\frac{\partial U}{\partial t} + \nabla\cdot UU - \nabla\cdot(\mu\nabla U) = -\nabla p \tag{2}
$$

关于方程组

- 对于三维问题来说，四个分量方程对应四个未知数（$p, U_x, U_y, U_z$），看似可以求解
- 但是压力没有自己的约束方程（不可压流动没有关于压力的状态方程）
- 质量方程实际上是对动量方程求解后的再约束，也就是说动量方程求解后的速度场仍然要回代以满足质量方程

在阅读过 FVM 理论之后，可以知道，本地离散方程总是

`系数 * 本单元量 + 系数 * 邻单元量 = 本地源项` 

遍历全局后，组建全局离散方程，形成待求解的线性代数系统，形如

$$Ax = b$$

与待求场量相关的统统的纳入系数矩阵，放在左边，也就是 LHS（Left-hand-side），涉及源项线性化之后与场不直接相关的项等等，统统放在右手边，也就是 RHS(Right-hand-side)。最终的最终，动量方程形式可以为

$$MU = -\nabla p \tag{3}$$

> 这里没有考虑因为离散或者因为边界条件等原因出现的 RHS

可以想见场的系数矩阵 M 是一个对角占优的稀疏方阵（或者说我们希望它能够对角占优），$M$ 矩阵的对角线都是离散方程中的本元素（有的书用字母 C 指代，有的用字母 P 指代），同一行上，对角线前后（在该行也许不紧挨）的元素都是和该单元相邻的单元。

最基本的思路是，我们最开始假设一个初始压力场，根据动量方程求出速度场。直接从动量方程求出来的速度场反过来再去满足质量方程的约束。

那，怎么从动量方程求出速度场呢？

为了更好的求解方程（3），先处理系数矩阵 $M$ 。

从 $M$ 矩阵中分解出对角矩阵 $A$ ，对角矩阵 $A$ 可以很容易求出逆矩阵。OpenFOAM 中的对角矩阵以及求解方法已经高度抽象化，也更易读。

```cpp
// 对角矩阵的倒数
volScalarField rAU(1.0/UEqn.A());
```

抽离得到对角矩阵后，方程（3）的左侧可以写成

$$MU = AU - H \tag{4}$$

作为比较，注意以下处理是错误的。

$$
\cancel{MU = AU - HU}
$$

所以，分解后的动量方程为

$$AU - H = -\nabla p \tag{5}$$

两边同时乘以 $A$ 的逆矩阵

$$A^{-1}AU = A^{-1}H - A^{-1}\nabla p \tag{6}$$

可以求出速度场

$$U = A^{-1}H - A^{-1}\nabla p \tag{7}$$

其中

$$H = AU - MU \tag{8}$$

$H$ 的求解在 OpenFOAM 直接使用成员函数调用即可 `UEqn.H()` 。

到这里速度场就求解出来了，OpenFOAM 把方程求解高度抽象化，但也更加容易阅读。

动量方程求解出速度的相关代码在文件 `UEqn.H` 中，这一步也被称为 `momentum predictor`。

```cpp
// OpenFOAM
solve(UEqn == -fvc::grad(p));
...
```

求出的速度场并不是最终结果，速度场需要满足质量方程，所以还要求解质量方程。
质量方程为 

$$\nabla\cdot U = 0 \tag{1}$$

代入速度场的解，也就是方程（7），有

$$\nabla\cdot (A^{-1}H - A^{-1}\nabla p) = 0$$

整理后得到实际用来求解的速度约束方程，

$$\nabla\cdot (A^{-1}\nabla p) = \nabla\cdot(A^{-1}H) \tag{9}$$

方程（9）也被称为压力方程，OpenFOAM 中的相关代码在 `pEqn.H` 中

```cpp
// OpenFOAM
volScalarField rAU(1.0/UEqn.A());
volScalarField HbyA(constrainHbyA(rAU*UEqn.H(), U, p));
solve(fvm::laplacian(rAU,p) == fvc::div(HbyA));
```

求解压力方程（9），可以得到新的压力场。

压力速度修正计算的大体思路就是这样子，下面具体看不同的求解算法。

## SIMPLE

### 理论

1. 基于上一步的压力场（启动步使用初始压力场），计算 `momentum predictor` ，求得速度场

$$\frac{\partial U}{\partial t} + \nabla\cdot UU - \nabla\cdot(\mu\nabla U) = -\nabla p \tag{2}$$

2. 求出非对角线矩阵 $H$

$$H = AU - MU \tag{8}$$

3.  求解质量方程，求出新的压力场

$$\nabla\cdot (A^{-1}\nabla p) = \nabla\cdot(A^{-1}H) \tag{9}$$

4. 以新的压力场，更新速度场

$$U = A^{-1}H - A^{-1}\nabla p \tag{7}$$

5. 回到步骤 1，基于新的压力场，求解速度场，继续迭代

> 这一过程被称为 outer loops

### 实现

我们对比 OpenFOAM 原生 `simpleFoam` 求解器，看看单纯的 `SIMPLE` 算法的实现应该是什么样子的。

```
sol
cd incompressible/simpleFoam
code .
```

1. 检查是否继续循环——`simple.loop()`
2. 使用 `momentum predictor` 求解速度——`UEqn.H`（也就是上面的 step1）
3. 修正压力和速度——`pEqn.H` （也就是上面的 step3，step4）
4. 为湍流模型求解输运方程——`turbulence->correct()`
5. 返回步骤 1

`Momentum predictor` 也就是 `UEqn.H` 中的主要代码（因版本有变化只摘取主要部分）

```cpp
// Main code
// Momentum predictor
...
tmp<fvVectorMatrix> UEqn
(
	fvm::div(phi, U) // 为什么使用 phi，见相关系列讨论
	+ tubulence->divDevReff(U) // 这里其实就是扩散项，使用了湍流模型
	== 
	fvOptions(U) // 源项处理框架，暂时不用深究
);

UEqn().relax(); // 亚松驰（欠松弛）

...

solve(UEqn() == -fvc::grad(p)); // 方程（2）

```

之后进行压力修正，基于上一步的速度求解（`predicted velocity`）。修正后的压力被拿来求解质量方程，也就修正了速度。`pEqn.H` 代主要码如下（因版本有变化只摘取主要部分）

```cpp
...

volScalarField rAU(1.0/UEqn().A()); // 系数矩阵对角部分的倒数
volVectorField HbyA = UEqn().H() * rA;
// OpenFOAM 为什么使用 phiHbyA 呢？需要学习 ofss 系列讨论
...

// Non-orthogonal pressure corrector loop
while (simple.correctNonOrthogonal())
{
	fvScalarMatrix pEqn // 构建压力方程
	(
		fvm::laplacian(rAU, p) == fvc::div(HbyA) //  方程（9）
	);
	...

	pEqn.solve(); // 求解压力方程

	...
}

...

// Explicitly relax pressure for momentum corrector
p.relax();

// Momentum corrector
U = HbyA - rAU()*fvc::grad(p); // 方程（7）

...

```

主要算法框架主要结构如下

```cpp
while (simple.loop())
{
	{
		#include "UEqn.H"
		#include "pEqn.H"
	}
	laminarTransport.correct();
	turbulence->correct();
	
	runTime.write();
}
```

### 讨论

OpenFOAM 经过多年的更新，求解器增添了很多保证数值计算结果的方法。我们暂时不用深究，只需要把握主要算法即可。
## PISO

### 理论

1. 基于上一步的压力场（启动步使用初始压力场），计算 `momentum predictor` ，求得速度场

$$\frac{\partial U}{\partial t} + \nabla\cdot UU - \nabla\cdot(\mu\nabla U) = -\nabla p \tag{2}$$

2. 求出非对角线参与矩阵 $H$

$$H = AU - MU \tag{8}$$

3.  求解质量方程，求出新的压力场

$$\nabla\cdot (A^{-1}\nabla p) = \nabla\cdot(A^{-1}H) \tag{9}$$

4. 以新的压力场，更新速度场

$$U = A^{-1}H - A^{-1}\nabla p \tag{7}$$

5. 回到步骤 2（不再重新求解原始动量方程），继续迭代
6. 求解满足要求后，时间推进

> 这一过程被称为 inner loops

### 实现

在 OpenFOAM 里，PISO 用来求解瞬态问题。PISO 和 SIMPLE 的两个方程的文件差不多，但是主体算法结构不太相同。

PISO 的算法结构主要如下

```cpp
while (runTime.loop())
{
	{
		#include "UEqn.H"
		while (piso.corret())
		{
			#include "pEqn.H"
		}
	}
	laminarTransport.correct();
	turbulence->correct();
	
	runTime.write();
}
```

## SIMPLE & PISO 对比

SIMPLE 最一开始用来计算稳态流动，也就是没有时间瞬态项。

如果有时间瞬态项，当时间步长很小的时候，时间瞬态项将比其他项大的多得多，此时也会主导系数矩阵的对角元素。

越小的时间步长，系数矩阵越占优。

而对于稳态问题，没有这种瞬态的对角占优的优势，所以为了达到对角占优，求解器需要使用欠松弛来增加对角占优性质，从而提高计算稳定性。

对于 SIMPLE 来说，未知量 $A$ 场计算的处理方法如下：

> 如果对下面这个方程很陌生，需要马上去补有限体积法

我们知道离散方程总成写成以下形式

$$a_PA_P + \sum a_NA_N = R_P \tag{10}$$

两边添加松弛

$$\frac{1-\alpha}{\alpha}a_pA_p^{old-iteration}+a_pA_p+ \sum a_NA_N = R_p + \frac{1-\alpha}{\alpha}a_pA_p^{older-iteration}$$

收敛的时候，旧时间步和新时间步的场量相等，即

$$\frac{1-\alpha}{\alpha}a_pA_p+a_pA_p+ \sum a_NA_N = R_p + \frac{1-\alpha}{\alpha}a_pA_p^{older-iteration}$$


整理上式有

$$\frac{1}{\alpha}a_pA_p + \sum a_NA_N = R_p + \frac{1-\alpha}{\alpha}a_pA_p^{older-iteration} \tag{11}$$

因为 $\alpha$ 是大于 0 小于 1 的，所以对角元素变相增大了，也就是增加了对角占优性质，也就增加了计算稳定性。

CFD 中对场量 $\phi$ 的欠松弛操作如下

$$A = \alpha A_{new} + (1-\alpha)A_{old}$$

该欠松弛等式代入方程（11）的 $A_P$，化简即可回到方程（10），即松弛后也满足原始方程。

注意到 SIMPLE 的大循环十分消耗资源，对于非稳态计算，每个时间步都要完整循环是难以承受的。

如果时间步长足够小，我们可以作 1 次 momentum predictor（外循环）和 2 次 pressure corrector（内循环），此时也不再必然需要欠松弛处理。

## PIMPLE

### 理论

基于 SIMPLE 和 PISO 的讨论，结合二者的优点有了 PIMPLE 算法。是大时间步长瞬态不可压问题求解器。

我们着重讨论一下 PIMPLE 算法，从控制方程开始。

质量方程

$$\frac{\partial \rho}{\partial t} + \nabla\cdot(\rho U) = 0 \tag{10}$$

因为不可压缩，所以密度恒定 $\rho = const$，所以

$$\nabla\cdot U = 0 \tag{11}$$

动量方程

$$\frac{\partial \rho U}{\partial t} + \nabla\cdot \rho UU + \nabla\cdot\tau = -\nabla p + g \tag{12}$$

因为密度恒定

$$\frac{\partial U}{\partial t} + \nabla\cdot UU + \frac{1}{\rho}\nabla\cdot\tau = -\frac{\nabla p}{\rho} + \frac{g}{\rho} \tag{13}$$

方程（13）右边的最后一项概括成广义源项，切应力项和压力梯度项对密度处理

$$\frac{\partial U}{\partial t} + \nabla\cdot UU + \nabla\cdot R^{eff} = -\nabla p + Q \tag{14}$$

由 Boussinesq 假设，我们在切应力项中包含雷诺应力

$$R^{eff} = -\nu^{eff}(\nabla U + (\nabla U)^T)$$

可以分成 `deviatoric part` 和 `hydrostatic part` 两部分 #why 

$$R^{eff} = dev(R^{eff}) + \frac{1}{3}tr(R^{eff})$$

第二部分实际上为零。

动量方程最终写成

$$\frac{\partial U}{\partial t} + \nabla\cdot UU + \nabla\cdot dev(-\nu^{eff}(\nabla U + (\nabla U)^T)) = -\nabla p + Q \tag{15}$$

### 实现

动量方程 `UEqn.H` 和压力方程 `pUqn.H` 的主要的代码和 `SIMPLE` 以及 `PISO` 并没有什么区别。

我们来看一下主源码中的算法结构，主要如下

```cpp
while (runTime.run())
{
	#include "readTimeCOntrols.H"
	#include "CourantNo.H"
	#include "setDeltaT.H"
	++runTime;

	while (pimple.loop())
	{
		#include "UEqn.H"
		while (pimple.correct())
		{
			#include "pEqn.H"
		}
		if (pimple.turbCorr())
		{
			turbulence->correct();
		}
	}
	runTime.write();
}
```

看起来和 PISO 很像，实际上 `pimple.correct()` 进行了不一样的循环实现。

1. 检查是否收敛 `pimple.loop`
2. 使用 `momentum predictor` 求解速度——`UEqn.H`
3. 检查内循环是否结束 `pimple.correct()`
	1. 如果内循环继续，修正压力和速度——`pEqn.H` 
	2. 使用更新后速度，更新 **HbyA**
	3. 再次修正压力和速度，继续循环——`pEqn.H`
4. 如果内循环结束，判断是否湍流修正
	1. 如果需要湍流修正，为湍流模型求解输运方程——`turbulence->correct()`
5. 返回步骤 1

新版本 `readTimeControls.H` 中的内容已经放进了用户文件 `setRDeltaT.H` 中，跟随求解器用户自定义。

PIMPLE 算法的使用要求主代码头文件要包含 `#include "pimpleContro.H"`。这个 control 文件到底到底有些什么呢？

可以查找打开看一看 `/usr/lib/openfoam/openfoam2212/src/finiteVolume/cfdTools/general/solutionControl/pimpleControl/pimpleControl.H`

比如 `pimpleControl.H` 其中的 protected data 有

```cpp
            //- Maximum number of PIMPLE correctors
            label nCorrPIMPLE_;

            //- Maximum number of PISO correctors
            label nCorrPISO_;

            //- Current PISO corrector
            label corrPISO_;

            //- Flag to indicate whether to only solve turbulence on final iter
            bool turbOnFinalIterOnly_;

            //- Converged flag
            bool converged_;
```

而 `pimpleControl.C` 中有

```cpp
bool Foam::pimpleControl::read()
{
    solutionControl::read(false);
    ...
    const dictionary pimpleDict(dict());
    
    nCorrPIMPLE_ = pimpleDict.getOrDefault<label>("nOuterCorrectors", 1);
    nCorrPISO_ = pimpleDict.getOrDefault<label>("nCorrectors", 1);
    turbOnFinalIterOnly_ =
        pimpleDict.getOrDefault("turbOnFinalIterOnly", true);
    ...
```

上面这些实现描述了使用的关键字和默认值，这些关键字和取值放在 `fvSolution` 文件中。

`pimple.loop()` 是怎么代码实现的呢？

`pimpleControl.C` 中大概有如下语句，返回值决定循环是否结束。

```cpp
bool Foam :: pimpleControl :: loop ()
{
	read () ;
	corr_ ++; // 内置计算器
	...
	if ( corr_ == nCorrPIMPLE_ + 1)  // 超出指定循环次数
	{
		if ((! residualControl_ . empty () ) && ( nCorrPIMPLE_ != 1) ) // 没收敛
		{
			Info << algorithmName_ << ": not converged within "
			<< nCorrPIMPLE_ << " iterations " << endl ; // 没收敛报告
		}
		corr_ = 0;
		mesh_ . data :: remove (" finalIteration ");
		return false; // 退出 pimple.loop()
	}
}
```

`pimpleControl` 是从 `solutionControl` 继承而来的，比如 `corr_` 其实是声明和初始化在 `solutionControl.C` 中的。

```cpp
Foam::solutionControl::solutionControl(fvMesh& mesh, const word& algorithmName)
:
    regIOobject
    (
        IOobject
        (
            typeName,
            mesh.time().timeName(),
            mesh,
            IOobject::NO_READ,
            IOobject::NO_WRITE
        )
    ),
    mesh_(mesh),
    residualControl_(),
    algorithmName_(algorithmName),
    nNonOrthCorr_(0),
    momentumPredictor_(true),
    transonic_(false),
    consistent_(false),
    frozenFlow_(false),
    corr_(0),
    corrNonOrtho_(0)
{}
```

`nCorrPISO_` 用来控制 PISO loop，也称为 correct loop，也称为 inner loop，在 fvSolution 中使用关键字 `nCorrectors` 。

`pimple.loop()` 大概理解后，我们再看一下 `pimple.correct()` ，定义在 `pimpleControlI.H`。返回值决定是否进入循环。

```cpp
inline bool Foam::pimpleControl::correct()
{
    setFirstIterFlag();

    ++corrPISO_;

    if (debug)
    {
        Info<< algorithmName_ << " correct: corrPISO = " << corrPISO_ << endl;
    }

    if (corrPISO_ <= nCorrPISO_) // 如果小于指定的循环数
    {
        return true; // 需要进入循环
    }

    corrPISO_ = 0;

    setFirstIterFlag();

    return false;
}
```

### PIMPLE & PISO

在 `pimpleControl.C` 的构造函数有

```cpp
Foam::pimpleControl::pimpleControl
(
    fvMesh& mesh,
    const word& dictName,
    const bool verbose
)
:
    solutionControl(mesh, dictName),
    solveFlow_(true),
    nCorrPIMPLE_(0),
    nCorrPISO_(0),
    corrPISO_(0),
    SIMPLErho_(false),
    turbOnFinalIterOnly_(true),
    finalOnLastPimpleIterOnly_(false),
    ddtCorr_(true),
    converged_(false)
{
    read();

    if (verbose)
    {
        Info<< nl << algorithmName_;

        if (nCorrPIMPLE_ > 1)
        {
			...
        }
        else
        {
            Info<< ": Operating solver in PISO mode" << nl;
        }

        Info<< endl;
    }
}

```

可以看到，如果不指定 PIMPLE 循环次数，缺省值为 1，不大于 1，则执行 PISO 循环。

## 小结

我们大概了解了 OpenFOAM 基本算法 `SIMPLE, PISO, PIMPLE` 。因为涉及到了更多的代码语法，所以理解起来多少有些困惑。建议初学者优先理解算法思想，不要担心，细节会在后续讨论中逐渐充实。