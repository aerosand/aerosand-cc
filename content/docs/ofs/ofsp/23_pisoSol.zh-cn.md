---
uid: 20260320181145
title: 23_pisoSol
date: 2026-03-20
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
weight: 24
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

在上一篇讨论中，我们详细讨论了 PISO 算法，也粗略看了 OpenFOAM 中 PISO 算法主框架的代码片段。

这里，我们将基于对算法的讨论，进行代码实现，理解一些代码的使用。

本文主要讨论

- [ ] PISO算法实现
- [ ] 计算 cavity 算例

## 1. 控制方程

控制方程如下

连续方程（质量方程）

$$
\nabla\cdot U = 0
$$

动量方程为

$$\frac{\partial U}{\partial t} + \nabla \cdot (UU) = \nabla\cdot(\nu\nabla U)-\nabla p$$

依然有如下处理

- 粘性项已经简化
- 不考虑重力
- 密度已经处理

注意，动量方程存在瞬态项。

## 2. 项目准备

终端输入命令，建立项目

```terminal {fileName="terminal"}
ofsp
foamNewApp ofsp_23_pisoSol
cd ofsp_21_pisoSol
cp -r $FOAM_TUTORIALS/incompressible/icoFoam/cavity/cavity debug_case
code .
```

### 2.1. 说明文件

为项目提供说明文件

```markdown {fileName="README.md"}
## About

简单复现 PISO 算法的求解器。

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

@20260313

- 增加脚本 #done

```

### 2.2. 脚本文件

我们依然使用之前的脚本。以后除非改动，不再赘述。

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
    echo "Aerosand >>> Initial condition done."
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

## 3. 项目实现

我们用最简单的代码去实现 PISO 算法。

### 3.1. 主源码

考虑 `22_piso` 中的讨论，在主源码中实现 PISO 算法主要框架

```cpp {fileName="ofsp_23_pisoSol.C",linenos=table,linenostart=1}
#include "fvCFD.H"

#include "pisoControl.H" // 包含piso算法控制的头文件
// 该头文件属于finiteVolume库，无需为 Make 指定额外链接


// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    argList::addNote
    (
        "Aerosand: a test solver for PISO algorithm."
    );

    #include "postProcess.H" // 参考 13_commandLine，不再赘述

    #include "addCheckCaseOptions.H" // 参考 13_commandLine，不再赘述
    #include "setRootCaseLists.H" // 参考 13_commandLine，不再赘述
    #include "createTime.H"
    #include "createMesh.H"
    #include "createControl.H"
    #include "createFields.H" // 用户指定场的接入


    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< "\nStarting time loop\n" << endl;

    while (runTime.loop()) // 时间推进
    {
        Info<< "Time = " << runTime.timeName() << nl << endl;

        // Pressure-velocity PISO corrector
        {
            #include "UEqn.H" // 求解动量预测方程

            // --- PISO loop
            while (piso.correct()) // PISO循环，内循环
            {
                #include "pEqn.H" // 多次进行压力动量修正
            }
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

场的接入 `createFields.H` 和 `0fsp_21_simpleSol` 一样，具体如下

```cpp {fileName="createFields.H",linenos=table,linenostart=1}
/*
    # Basic fileds
    # transportProperties
    # Pressure reference
*/


// # Basic fields

Info<< "Reading field p\n" << endl;
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


// # Pressure reference

label pRefCell = 0;
scalar pRefValue = 0.0;
setRefCell(p, piso.dict(), pRefCell, pRefValue);

```

### 3.3. 动量预测

动量预测在 `UEq.H` 中，代码如下

```cpp {fileName="UEqn.H",base_url="https://aerosand.cc",linenos=table,linenostart=1}
// Momentum predictor

  fvVectorMatrix UEqn
  (
      fvm::ddt(U) // 瞬态项
    + fvm::div(phi, U) // 对流项
    - fvm::laplacian(nu, U) // 扩散项
  );
  // 不包括压力梯度

  UEqn.relax(); // 方程松弛

  solve(UEqn == -fvc::grad(p));

```

关于动量方程的松弛，后续会进行讨论。读者后续可以将该松弛代码注释，比较计算差别。


### 3.4. 压力动量修正

压力动量修正在 `pEqn.H` 中，代码如下

```cpp {fileName="pEqn.H",base_url="https://aerosand.cc",linenos=table,linenostart=1}
{
    volScalarField rAU(1.0/UEqn.A()); // A^-1 
    volVectorField HbyA(rAU*UEqn.H()); // HbyA
    surfaceScalarField phiHbyA // phiHbyA
	(
	    "phiHbyA",
	    fvc::flux(HbyA)
	);

    fvScalarMatrix pEqn
    (
        fvm::laplacian(rAU, p) == fvc::div(phiHbyA)
    );
	// 压力修正方程
	// 同样的，fvm离散返回一个矩阵，离散操作对象是参数p，p也就是待求未知量
	// fvc离散返回一个场，作为方程的已知量


    pEqn.setReference(pRefCell, pRefValue); // 设置压力参考

    pEqn.solve(); // 求解压力修正方程

	// 无需压力松弛

    // Momentum corrector // 动量修正
    U = HbyA - rAU*fvc::grad(p); // 求解得到修正速度
    
    // 修正后的压力和速度，重新参与下一次的PISO循环，直到满足次数后，进入时间循环
}

```


> [!tip]
> 注意，为了方便理解，这里没有进行非正交修正。因为后面使用的 cavity 算例网格简单，所以也没有什么影响。

### 3.5. 项目Make

如上面讨论的那样，该求解器并没有使用更多的库，所以项目 Make 文件无需额外增加链接的指定。

其中的 `Make/files` 内容如下

```cpp {fileName="Make/files",base_url="https://aerosand.cc",linenos=table,linenostart=1}
ofsp_23_pisoSol.C

EXE = $(FOAM_USER_APPBIN)/ofsp_23_pisoSol

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
application     ofsp_23_pisoSol;
...
endTime         1.0; // 延长时间，比较是否松弛的计算效果
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
    // 因为PISO算法有内循环，最后一次的压力修正需要验证压力是否收敛

    U
    {
        solver          smoothSolver;
        smoother        symGaussSeidel;
        tolerance       1e-05;
        relTol          0;
    }
}

PISO // 指定 PISO 算法的字典参数
{
    nCorrectors     2;
    nNonOrthogonalCorrectors 0; // 非正交修正相关，其实我们没有涉及这部分
    pRefCell        0;
    pRefValue       0;
}
```

其他文件保持不变。

### 3.6. 编译运行

编译项目

```terminal {fileName="terminal"}
wclean
wmake
```

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


## 4. 小结

通过求解器项目的实现和讨论，相信我们现在应该对 PISO 算法和求解器实现有了较为全面的理解。

下一篇，我们将讨论 PIMPLE 算法。

本文完成讨论

- [x] PISO算法实现
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

