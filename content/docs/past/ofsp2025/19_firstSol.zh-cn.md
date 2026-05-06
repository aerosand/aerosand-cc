---
uid: 20251125190436
title: 19_firstSol
date: 2025-11-25
update: 2026-02-04
authors:
  - name: Aerosand
    link: https://github.com/aerosand
    image: https://github.com/aerosand.png
tags:
  - ofsp
  - ofsp2025
  - OpenFOAM
excludeSearch: false
toc: true
weight: 20
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

通过前面两个阶段的讨论，我们已经大概熟悉了 OpenFOAM 中数值应用的架构，熟悉了数值计算的关键要素（时间、网格、场），也熟悉了一些编程的语法、一些工作流程。下面我们使用所掌握的工具，尝试写一个最简单的 OpenFOAM 求解器（不涉及数值算法）。

计算流体力学的通用基本方程为

$$\frac{\partial}{\partial t}(\rho \phi) + \nabla \cdot (\rho U\phi) = \nabla\cdot(\Gamma\nabla\phi) + S_{\phi}$$

我们使用 $\phi$ 表示待求物理场，注意不要和 OpenFOAM 中常见的通量 phi 误会。

为了简单起见，我们考虑如下控制方程

$$\frac{\partial}{\partial t}\phi + \nabla\cdot(U\phi) - \nabla\cdot(\gamma\nabla \phi)=0$$

可见，这是一个瞬态不可压无源输运问题。

我们尝试对此控制方程进行编程求解。

本文主要讨论

- [ ] 完善脚本写法
- [ ] 简单理解方程构建和代码的对应关系
- [ ] 理解数值计算的简单流程
- [ ] 编译运行 firstSol 项目

## 1. 项目准备

终端输入命令，建立项目

```terminal {fileName="terminal"}
ofsp
foamNewApp ofsp_19_firstSol
cd ofsp_19_firstSol
cp -r $FOAM_TUTORIALS/incompressible/icoFoam/cavity/cavity debug_case
code .
```

测试初始求解器，提供脚本和说明。

为了进一步方便测试算例，我们将脚本分成四个部分

- 前处理 casepre
	- 备份和恢复初始条件
	- 划分网格（可选）
	- 设置区域（可选）
	- 等等
- 计算处理 caserun
	- 使用求解器对算例计算
- 后处理  casepost
	- 计算结果后处理（可选）
	- 计算结果可视化
- 计算清理 caseclean
	- 清理测试算例
	- 还原算例到初始状态

> [!note]
> 后续脚本非特别说明，暂时直接拷贝此版本即可。

前处理 casepre 如下

```bash {fileName="casepre",linenos=table,linenostart=1}
#!/bin/sh
cd "${0%/*}" || exit 1                              # Run from this directory
#------------------------------------------------------------------------------

cd debug_case # if run in debug_case/ folder

FOLDER=$(pwd) 
FOLDER0=$FOLDER/0
FOLDER0org=$FOLDER/0.org

if test -d "$FOLDER0" && test -d "$FOLDER0org"; then
  rm -rf 0/
  cp -r 0.org/ 0/
elif test -d "$FOLDER0"; then
  cp -r 0/ 0.org/
elif test -d "$FOLDER0org"; then
  cp -r 0.org/ 0/
fi
echo "\n>>>>>>>>>>>>> Initial condition done.\n"

#blockMesh > log.mesh # meshing in the background & logging
blockMesh | tee log.mesh # meshing & logging
echo "\n>>>>>>>>>>>>> Mesh done.\n"

# if needs setFields
# setFields
# echo "\n>>>>>>>>>>>>> Setting fileds done.\n"

```

计算处理 caserun 如下

```bash {fileName="caserun",linenos=table,linenostart=1}
#!/bin/sh
cd "${0%/*}" || exit 1                              # Run from this directory
#------------------------------------------------------------------------------

appPath=$(cd `dirname $0`; pwd)                     # Get application path
appName="${appPath##*/}"                            # Get application name

cd debug_case

# FOLDER=$(pwd) 
# FOLDER0=$FOLDER/0

# if ! test -d "$FOLDER0"; then
#     cp -r 0.org/ 0/
# fi

# $appName > log.run # if run in the background to save time
$appName | tee log.run

# pyFoamPlotWatcher.py log.run # if installed the PyFoam

```

后处理 casepost 如下

```bash {fileName="casepost",linenos=table,linenostart=1}
#!/bin/sh
cd "${0%/*}" || exit 1                              # Run from this directory
#------------------------------------------------------------------------------

cd debug_case

paraFoam

```

计算清理 caseclean 如下

```bash {fileName="caseclean",linenos=table,linenostart=1}
#!/bin/sh
cd "${0%/*}" || exit 1                              # Run from this directory
#------------------------------------------------------------------------------

appPath=$(cd `dirname $0`; pwd)
appName="${casePath##*/}"

cd debug_case

rm -rf log.*
# foamCleanTutorials                              # if clean mesh and time folders
foamListTimes -rm                                # only clean time folders
rm -rf 0/                                       # clean initial 0
echo "\n>>>>>>>>>>>>> Cleaning done.\n"

```

## 2. 场的接入

场的接入 createFields.H 如下

```cpp {fileName="createFields.H",linenos=table,linenostart=1}
Info<< "Reading transportProperties\n" << endl;
IOdictionary transportProperties
(
    IOobject
    (
        "transportProperties",
        runTime.constant(),
        mesh,
        IOobject::MUST_READ,
        IOobject::NO_WRITE
    )
);

Info<< "Reading diffusivity\n" << endl;
dimensionedScalar gamma("gamma",dimViscosity,transportProperties);


Info<< "Reading field A\n" << endl; // 引入待求的物理场量，比如 A
volScalarField A
(
    IOobject
    (
        "A",
        runTime.timeName(),
        mesh,
        IOobject::MUST_READ,
        IOobject::AUTO_WRITE
    ),
    mesh
);

Info<< "Reading field U\n" << endl;
volVectorField U
(
    IOobject
    (
        "U",
        runTime.timeName(),
        mesh,
        IOobject::MUST_READ,
        IOobject::AUTO_WRITE
    ),
    mesh
);

#include "createPhi.H"

```

## 3. 方程构建

我们修改主源码，来实现一次数学方程的计算。

主源码 ofsp_19_firstSol.C 如下

```cpp {fileName="ofsp_19_firstSol.C",linenos=table,linenostart=1}
#include "fvCFD.H"

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    #include "setRootCase.H"
    #include "createTime.H"

    #include "createMesh.H"

    #include "createFields.H"

    ++runTime;
    // 推进一个时间步，避免计算结果覆盖初始时间步

    solve // 求解控制方程
    (
	      fvm::ddt(A)
        + fvm::div(phi,A)
        - fvm::laplacian(gamma,A)
    );

    A.write(); // 写出计算后的物理场量A


    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< nl;
    runTime.printExecutionTime(Info);

    Info<< "End\n" << endl;

    return 0;
}


```

可以看到数学方程和编程语言的对应，即

$$\frac{\partial}{\partial t}A + \nabla\cdot(UA) - \nabla\cdot(\gamma\nabla A)=0$$

对应于

```cpp
	  fvm::ddt(A)
	+ fvm::div(phi,A)
	- fvm::laplacian(gamma,A)
```

### 3.1. 时间项

时间项的构造显然是使用了函数 ddt()

```cpp
fvm::ddt(A)
```

我们查找这个函数（或者通过 OFextension 插件直接访问代码）

```terminal {fileName="terminal"}
find $FOAM_SRC -iname fvmddt.c
```

为了帮助理解，从代码声明 fvmDdt.C 中摘取几处代码简单讨论如下（不建议继续深究代码，代码挖的太深无益于主要学习的推进）

```cpp {fileName="$FOAM_SRC//finiteVolume/finiteVolume/fvm/fvmDdt.C",linenos=table,linenostart=1}
namespace Foam // 属于命名空间 Foam
{

namespace fvm // 属于命名空间 fvm
{

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

template<class Type>
tmp<fvMatrix<Type>> // 返回类型
ddt // 函数名称
(
    const GeometricField<Type, fvPatchField, volMesh>& vf // 形参 
    // 需求 GeometricField 类型，回忆 GeometricField 有不同的类型定义
)
{
    return fv::ddtScheme<Type>::New
    (
        vf.mesh(),
        vf.mesh().ddtScheme("ddt(" + vf.name() + ')')
    ).ref().fvmDdt(vf);
}
...
```

我们了解了此函数的形参类型，也明白了返回的是 fvMatrix 类型的变量，数学上对应的是和待求量A相关的矩阵。

### 3.2. 对流项

对流项的构造使用了函数 div()

```cpp
fvm::div(phi,A)
```

我们查找这个函数（或者通过 OFextension 插件直接访问代码）

```terminal {fileName="terminal"}
find $FOAM_SRC -iname fvmdiv.C
```

为了帮助理解，从代码声明 fvmDiv.C 中摘取几处代码简单讨论如下（不建议继续深究代码，代码挖的太深无益于主要学习的推进）

```cpp {fileName="$FOAM_SRC/finiteVolume/finiteVolume/fvm/fvmDiv.C",linenos=table,linenostart=1}
...

// 对应 fvm::div(phi,A)

template<class Type>
tmp<fvMatrix<Type>> // 返回类型
div // 函数名
(
    const surfaceScalarField& flux, // 形参1 surfaceScalarField 类型的引用
    const GeometricField<Type, fvPatchField, volMesh>& vf 
    // 形参2 GeometricField 类型
)
{
    return fvm::div(flux, vf, "div("+flux.name()+','+vf.name()+')');
}
...

```

我们了解了此函数的形参类型，也明白了返回的是 fvMatrix 类型的变量，数学上对应的是和待求量A相关的矩阵。

### 3.3. 扩散项

扩散项的构造使用了函数 laplacian() 

我们查找这个函数（或者通过 OFextension 插件直接访问代码，不再赘述）

```terminal {fileName="terminal"}
find $FOAM_SRC -iname fvlaplacian.C
```

同样的，摘取部分代码如下

```cpp {fileName="$FOAM_SRC/finiteVolume/finiteVolume/fvm/fvmLaplacian.C",linenos=table,linenostart=1}
...

// 对应 fvm::laplacian(gamma,A)

template<class Type, class GType>
tmp<fvMatrix<Type>> // 返回类型
laplacian // 函数名
(
    const dimensioned<GType>& gamma, // 形参1 dimensioned 类型的引用
    // dimensioned 有不同类型的定义
    const GeometricField<Type, fvPatchField, volMesh>& vf
    // 形参2 同样是 GeometricField 类型的引用
)
{
    const GeometricField<GType, fvsPatchField, surfaceMesh> Gamma
    (
        IOobject
        (
            gamma.name(),
            vf.instance(),
            vf.mesh(),
            IOobject::NO_READ
        ),
        vf.mesh(),
        gamma
    );

    return fvm::laplacian(Gamma, vf);
}
...
```

同样的，我们了解了此函数的形参类型，也明白了返回的是 fvMatrix 类型的变量，数学上对应的是和待求量A相关的矩阵。

### 3.4. 方程求解

离散方程最后形式为

$$
Ax = b
$$

对流项和扩散项的矩阵相加，构成了待求左侧项，因为没有源项，所以右侧项置空为零，可以缺省不写。

对流项和扩散项组建在一起，并通过 solve() 函数进行求解

```cpp
    solve // 求解控制方程
    (
          fvm::ddt(A)
        + fvm::div(phi,A)
        - fvm::laplacian(gamma,A)
    );
```

求解函数的代码实现，暂不深究。

## 4. 调整算例

因为项目中加入了未知场 A 的计算，所以需要调整算例。

### 4.1. 初始场A

为项目提供初始场A，如下

```cpp {fileName="debug_case/0.org/A",linenos=table,linenostart=1}
FoamFile
{
    version     2.0;
    format      ascii;
    class       volScalarField;
    object      A;
}
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

dimensions      [0 0 0 0 0 0 0];

internalField   uniform 500;

boundaryField
{
    movingWall
    {
        type            fixedValue;
        value           uniform 100;
    }

    fixedWalls
    {
        type            fixedValue;
        value           uniform 0;
    }

    frontAndBack
    {
        type            empty;
    }
}

```

### 4.2. transportProperties

为 transportProperties 字典添加新的扩散系数

```cpp {fileName="debug_case/constant/transportProperties"}
...
gamma           1.0;
...
```

### 4.3. fvSchemes

为数值计算项指定离散格式，检查保证有以下指定

```cpp {fileName="debug_case/system/fvSchemes",linenos=table,linenostart=1}
...

ddtSchemes
{
    default         Euler;
    ddt(A)          Euler; // 如果没有default指定
}

...

divSchemes
{
    default         none;
    div(phi,A)      Gauss linear; // 如果没有default指定
}

laplacianSchemes
{
    default         Gauss linear orthogonal;
    laplacian(gamma,A)  Gauss linear orthogonal; // 如果没有default指定
}

...
```

具体指定背后的数学物理是什么，暂不深究。

### 4.4. fvSolution

为线性代数系统指定代数求解器，检查保证有以下指定

```cpp {fileName="debug_case/system/fvSolution",linenos=table,linenostart=1}
...
solvers
{
	...    
    A
    {
        solver          GAMG;
        smoother        symGaussSeidel;
        tolerance       1e-06;
        relTol          0.05;
    }
}
...
```

具体指定背后的数学物理是什么，暂不深究。

## 5. 编译运行

编译并运行此项目

```terminal {fileName="terminal"}
wclean
wmake
./casepre & ./caserun & ./casepost
```

通过计算，我们得到了一个时间步的计算结果。通过 paraview 将结果可视化，可以选择查看该时间步的 A 场分布情况。

## 6. 完整问题

在简单了解了 OpenFOAM 如何构建方程之后，我们继续将完整考虑上述问题

### 6.1. 问题描述

![[image.png]]

在一个与 cavity 算例相同的方形容器中，对于速度场来说，上边界有速度 $U_{x}=1m/s$ ，其他边界为固定边界（无法向速度交换）。对于某种无单位的A场来说，内计算域为500，上边界为100，其他边界为0。考虑以下数学物理方程描述的瞬态不可压无源输运问题。

$$\frac{\partial}{\partial t}A + \nabla\cdot(UA) - \nabla\cdot(\gamma\nabla A)=0$$

### 6.2. 求解器

求解器主源码如下

```cpp {fileName="ofsp_19_firstSol.C",linenos=table,linenostart=1}
#include "fvCFD.H"

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    #include "setRootCase.H"
    #include "createTime.H"

    #include "createMesh.H"

    #include "createFields.H"

    while(runTime.loop())
    {
        Info<< "Time = " << runTime.timeName() << nl << endl;

        // Solve the transport equation
        fvScalarMatrix AEqn
        (
            fvm::ddt(A)
          + fvm::div(phi, A)
          - fvm::laplacian(gamma, A)
        );

        AEqn.solve();

        runTime.write();

        runTime.printExecutionTime(Info);
    }

    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< nl;

    Info<< "End\n" << endl;

    return 0;
}

```

### 6.3. 测试算例

我们继续使用前面的测试算例的文件（包括初始条件、边界条件、网格字典、输运属性）。

不过为了能够在可视化的计算结果中看到明显变化，需要调整控制计算的字典

```cpp {fileName="debug_case/system/controlDict",linenos=table,linenostart=1}
...

endTime         0.01;

deltaT          0.001;

writeControl    timeStep;

writeInterval   1;

...
```

### 6.4. 编译运行

重新编译并运行此项目

```terminal {fileName="terminal"}
wclean
wmake
./caseclean & ./casepre & ./caserun & ./casepost
```

通过计算，我们得到了随时间变化的计算结果。通过 paraview 将结果可视化，可以选择查看A场随着时间变化的分布情况。

## 7. 小结

一些读者可能注意到，我们留下了很多的问题。比如，

- 为什么在散度计算中，使用的是通量 phi 而不是公式中的 U 呢？

简单来说，如果我们学习过有限体积法，可以理解在 FVM 中，真正守恒的是通量，这也是 FVM 的核心和优势。

- 为什么使用 fvm 呢？和常见到的 fvc 有什么区别呢？

简单理解的话， 求解器中的 fvm 离散所有对系数矩阵有贡献的项（隐式离散），相对比的，fvc 离散和系数矩阵无关的项（显式离散）。

但是上面轻飘飘的一句话解释，其实很难让初学者理解。为了讨论上面两个问题，我们需要回到控制方程的偏微分方程数值求解上，从有限体积法开始，一步一步推导，直到矩阵求解，才可以真正明白为什么要使用通量进行计算，为什么会有隐式离散和显式离散的区分。这里暂不深究，参见CFD 基础系列讨论（cfdb），以及 OpenFOAM 求解器系列讨论（ofss）。

好在求解器的代码语法已经十分贴近数学公式，即使带着这些困惑也不要紧，我们仍然可以编写简单的求解器。

本文完成讨论

- [x] 完善脚本写法
- [x] 简单理解方程构建和代码的对应关系
- [x] 理解数值计算的简单流程
- [x] 编译运行 firstSol 项目

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
