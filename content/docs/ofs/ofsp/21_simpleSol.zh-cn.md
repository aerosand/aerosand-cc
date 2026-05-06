---
uid: 20260128101852
title: 21_simpleSol
date: 2026-01-28
update: 2026-03-26
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
weight: 22
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

在上一篇讨论中，我们详细讨论了 SIMPLE 算法，也粗略看了 OpenFOAM 中 SIMPLE 算法的一部分代码片段。

这里，我们将基于对算法的讨论，进行代码实现，理解一些代码的使用。

本文主要讨论

- [ ] SIMPLE算法实现
- [ ] 计算 cavity 算例

## 1. 控制方程

控制方程如下

连续方程（质量方程）

$$
\nabla\cdot U = 0
$$

动量方程为

$$\nabla \cdot (UU) = \nabla\cdot(\nu\nabla U)-\nabla p$$

依然有如下处理

- 稳态
- 粘性项已经简化
- 不考虑重力
- 密度已经处理

## 2. 项目准备

终端输入命令，建立项目

```terminal {fileName="terminal"}
ofsp
foamNewApp ofsp_21_simpleSol
cd ofsp_21_simpleSol
cp -r $FOAM_TUTORIALS/incompressible/icoFoam/cavity/cavity debug_case
code .
```

### 2.1. 说明文件

为项目提供说明文件

```markdown {fileName="README.md"}
## About

简单复现 SIMPLE 算法的求解器。

## Bio

aerosand @ aerosand

## Caution

注意 OpenFOAM 版本。

## Deploy

保证求解器文件完整，并包括 debug_case 文件夹

终端输入命令

./Allclean
./Allrun


## Event

@20260312

- 增加脚本 #done

```

### 2.2. 脚本文件

我们精简之前的脚本。

终端输入命令，新建脚本

```terminal {fileName="terminal"}
code {Allclean,Allrun}
```

清理脚本 `Allclean` 如下

```bash {fileName="Allclean",linenos=table,linenostart=1}
#!/bin/sh
cd "${0%/*}" || exit                                # Run from this directory
. ${WM_PROJECT_DIR:?}/bin/tools/RunFunctions        # Tutorial run functions
#------------------------------------------------------------------------------

cd debug_case

# 函数：初始化 0 文件夹
init_0() {
    if [ -d "0.orig" ] && [ -d "0" ]; then
        echo "Both 0 and 0.orig exist. Removing 0 and restoring from 0.orig..."
        rm -rf 0
        cp -a 0.orig 0
    elif [ -d "0" ] && [ ! -d "0.orig" ]; then
        echo "Backing up 0 to 0.orig..."
        cp -a 0 0.orig
    elif [ -d "0.orig" ] && [ ! -d "0" ]; then
        echo "Restoring 0 from 0.orig...".
        cp -a 0.orig 0
    else
        echo "Warning: Neither 0 nor 0.orig exists. Skipping."
    fi
    echo ">>>>>>>>>>>>> Initial condition done."
}

# 确保初始条件
init_0

# 清理算例
. ${WM_PROJECT_DIR:?}/bin/tools/CleanFunctions      # Tutorial clean functions
cleanCase0

```

运行脚本 `Allrun` 如下

```cpp {fileName="Allrun",linenos=table,linenostart=1}
#!/bin/sh
cd "${0%/*}" || exit                                # Run from this directory
. ${WM_PROJECT_DIR:?}/bin/tools/RunFunctions        # Tutorial run functions
#------------------------------------------------------------------------------

cd debug_case

restore0Dir

runApplication blockMesh

runApplication $(getApplication)

```

不同担心脚本，随着讨论深入，我们会不断扩充脚本的内容，尝试和熟悉不同的写法。

> [!note]
> 使用 `chmod +x {Allclean,Allrun}` 提升脚本权限。
> 
> 以后继续使用此脚本即可，除非强调，不再赘述。

## 3. 项目实现

我们用最简单的代码去实现 SIMPLE 算法。

### 3.1. 主源码

考虑 `20_simple` 中的讨论，在主源码中实现 SIMPLE 算法主要框架

```cpp {fileName="ofsp_21_simpleSol.C",linenos=table,linenostart=1}
#include "fvCFD.H" // 参考 07_fisrtApp，以后不再赘述

#include "simpleControl.H" // 包含simple算法控制的头文件
// 该头文件属于finiteVolume库，无需为 Make 指定额外链接


// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    #include "setRootCase.H" // 参考 13_commandLine，以后不再赘述
    #include "createTime.H" // 参考 14_time，以后不再赘述

    #include "createMesh.H" // 参考 15_mesh，以后不再赘述
    #include "createControl.H" // 创建算法控制，可以根据使用情况自动创建
    #include "createFields.H" // 用户提供场的接入


    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< "\nStarting time loop\n" << endl;

    while (simple.loop()) // SIMPLE算法循环
    {
        Info<< "Time = " << runTime.timeName() << nl << endl;

        // --- Pressure-velocity SIMPLE corrector
        {
            #include "UEqn.H" // 动量预测
            #include "pEqn.H" // 压力动量修正
        }

        runTime.write();

        runTime.printExecutionTime(Info);
    }

    Info<< "End\n" << endl;

    return 0;
}

```

可以看到算法主框架和上一篇的讨论一样。

### 3.2. 场的接入

场的接入 `createFields.H` 如下

```cpp {fileName="createFields.H",linenos=table,linenostart=1}
/*
    # Basic fileds
    # transportProperties
    # Pressure reference
*/

// # Basic fields // 基础场量

Info<< "Reading field p\n" << endl; // 压力场量
volScalarField p
(
    IOobject
    (
        "p",
        runTime.timeName(),
        mesh,
        IOobject::MUST_READ,
        IOobject::AUTO_WRITE
    ),
    mesh
);

Info<< "Reading field U\n" << endl; // 速度场量
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

#include "createPhi.H" // 质量通量场量

// # transportProperties // 输运性质

Info<< "Reading transportProperties\n" << endl;

IOdictionary transportProperties
(
    IOobject
    (
        "transportProperties",
        runTime.constant(),
        mesh,
        IOobject::MUST_READ_IF_MODIFIED,
        IOobject::NO_WRITE
    )
);

dimensionedScalar nu
(
    "nu",
    dimViscosity,
    transportProperties
);


// # Pressure reference // 压力参考

label pRefCell = 0;
scalar pRefValue = 0.0;
setRefCell(p, simple.dict(), pRefCell, pRefValue);

```

### 3.3. 动量预测

动量预测在 `UEq.H` 中，代码如下

```cpp {fileName="UEqn.H",base_url="https://aerosand.cc",linenos=table,linenostart=1}
// Momentum predictor

  fvVectorMatrix UEqn
  (
      fvm::div(phi, U)
    - fvm::laplacian(nu, U)
  );

  UEqn.relax(); // 方程松弛，后续会讨论

  solve(UEqn == -fvc::grad(p));

```

关于动量方程的松弛，后续会进行讨论。读者后续可以将该松弛代码注释，比较计算差别。

### 3.4. 压力动量修正

压力动量修正在 `pEqn.H` 中，代码如下

```cpp {fileName="pEqn.H",base_url="https://aerosand.cc",linenos=table,linenostart=1}
{
    volScalarField rAU(1.0/UEqn.A()); // A^-1 
    volVectorField HbyA(rAU*UEqn.H()); // HbyA
    surfaceScalarField phiHbyA("phiHbyA", fvc::flux(HbyA)); // phiHbyA

    fvScalarMatrix pEqn
    (
        fvm::laplacian(rAU, p) == fvc::div(phiHbyA)
    );
	// 压力修正方程
	// 同样的，fvm离散返回一个矩阵，离散操作对象是参数p，p也就是待求未知量
	// fvc离散返回一个场，作为方程的已知量


    pEqn.setReference(pRefCell, pRefValue); // 设置压力参考

    pEqn.solve(); // 求解压力修正方程

    // Explicitly relax pressure for momentum corrector
    p.relax(); // 场量松弛，后续会讨论

    // Momentum corrector // 动量修正
    U = HbyA - rAU*fvc::grad(p); // 求解得到修正速度
    
    // 修正后的压力和速度，重新参与下一次的SIMPLE循环
}

```

压力同样进行了松弛，后续会讨论，读者可以自行尝试注释，查看计算差别。

> [!tip]
> 注意，为了方便理解，这里没有进行非正交修正。因为后面使用的 cavity 算例网格简单，所以也没有什么影响。

### 3.5. 项目Make

如上面讨论的那样，该求解器并没有使用更多的库，所以项目 Make 文件无需额外增加链接的指定。

其中的 `Make/files` 内容如下

```cpp {fileName="Make/files",base_url="https://aerosand.cc",linenos=table,linenostart=1}
ofsp_21_simpleSol.C

EXE = $(FOAM_USER_APPBIN)/ofsp_21_simpleSol

```

其中的 `Make/options` 内容如下

```cpp {fileName="Make/options",base_url="https://aerosand.cc",linenos=table,linenostart=1}
EXE_INC = \
    -I$(LIB_SRC)/finiteVolume/lnInclude \
    -I$(LIB_SRC)/meshTools/lnInclude

EXE_LIBS = \
    -lfiniteVolume \
    -lmeshTools

```

### 3.6. 编译

终端输入命令，编译整个项目

```terminal {fileName="terminal"}
wclean
wmake
```

编译成功，没有问题。

### 3.7. 测试算例

我们调整拷贝的测试算例，用来测试上述求解器。

控制字典 `controlDict` 中修改如下

```cpp {fileName="controlDict",base_url="https://aerosand.cc",linenos=table,linenostart=1}
...
application     ofsp_21_simpleSol;
...
endTime         1.0; // 延长时间，比较是否开启松弛的不同计算效果
// 如果不进行松弛，计算可能会在一段时间后发散终止
...
```

求解字典 `fvSolution` 中修改如下

```cpp {fileName="fvSolution",base_url="https://aerosand.cc",linenos=table,linenostart=1}
solvers
{
    p
    {
        solver          PCG;
        preconditioner  DIC;
        tolerance       1e-06;
        relTol          0.05;
    }

    pFinal
    {
        $p;
        relTol          0;
    }

    U
    {
        solver          smoothSolver;
        smoother        symGaussSeidel;
        tolerance       1e-05;
        relTol          0;
    }
}


SIMPLE // 指定 SIPMLE 算法的字典参数
{
    nNonOrthogonalCorrectors 0;
    consistent      yes;
    pRefCell        0;
    pRefValue       0;

    residualControl
    {
        p               1e-2;
        U               1e-3;
    }
}

relaxationFactors // 指定松弛处理的字典参数
{
    fields // 场量松弛
    {
        p               0.3;   // 压力场松弛因子 0.3
        ".*"       0.7;   // 使用正则表达式匹配所有 p_rgh 变体
    }
    
    equations // 方程松弛
    {
        U               0.9; // 0.9 is more stable but 0.95 more convergent
        ".*"            0.9; // 0.9 is more stable but 0.95 more convergent
    }
}

```

其他文件保持不变。

### 3.7. 运行

求解器项目编译没有问题，直接通过脚本运行即可。

运行项目

```terminal {fileName="terminal"}
./Allclean
./Allrun
```

后处理可视化

```terminal {fileName="terminal"}
paraFoam -case debug_case
```

可以在 paraview 中查看计算结果。

读者可以修改部分代码，探索不同语句的作用。

## 4. 小结

通过求解器项目的实现和讨论，相信我们现在应该对 PISO 算法和求解器实现有了较为全面的理解。

下一篇，我们将讨论 PISO 算法。

本文完成讨论

- [x] SIMPLE算法实现
- [x] 计算 cavity 算例


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


