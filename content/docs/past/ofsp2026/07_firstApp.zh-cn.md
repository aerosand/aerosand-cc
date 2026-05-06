---
uid: 20250901123439
title: 07_firstApp
date: 2025-09-01
update: 2026-02-04
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
weight: 8
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

之前讨论了编译原理、动态库的链接和 OpenFOAM 常见的类，现在我们使用 OpenFOAM ，完成一个较为完整的开发流程。

本文主要讨论

- [ ] 理解 OpenFOAM 标准应用的文件架构
- [ ] 认识使用脚本
- [ ] 认识应用开发测试流程
- [ ] 调用其他位置的开发库
- [ ] 编译运行 firstApp 项目

## 1. fvCFD.H

在实际开发中，除了之前提到了 vector 和 tensor 等，我们还需要用到更多的和 FVM 相关的类来离散求解偏微分方程。OpenFOAM 提供 `fvCFD.H` ，其中包含了大部分和 FVM 相关的头文件，包括 tensor 类等。使用 `fvCFD.H` 可以大大减少主源码要写的头文件数量。

终端输入命令，查找 `fvCFD.H` 文件

```terminal {fileName="terminal"}
find $FOAM_SRC -iname fvcfd.h
```

终端输出如下

```terminal {fileName="terminal"}
/usr/lib/openfoam/openfoam2406/src/finiteVolume/cfdTools/general/include/fvCFD.H
/usr/lib/openfoam/openfoam2406/src/finiteVolume/lnInclude/fvCFD.H
```

显然 `fvCFD.H` 在 finiteVolume 库里，所以需要另外在 `Make/options` 中包含、链接。

> [!tip]
> OpenFOAM 库是低层库，已配置为自动依赖。而 finiteVolume 库属于高层库，依赖于众多低层库，需要手动配置。

我们可以将上篇代码中的原生头文件替换成 `fvCFD.H` ，并配置相关的 Make 文件。



## 2. 项目

我们使用 OpenFOAM 提供的方式创建求解器应用基础模板，并在此基础上进行开发。

终端输入命令，建立本文项目

```terminal {fileName="terminal"}
ofsp
foamNewApp ofsp_07_firstApp
code ofsp_07_firstApp
```

### 2.1. 源代码

下面将讨论源代码。

### 2.2. 测试算例

对于不同的求解器，可以拷贝一个对应的原生算例用于测试求解器的开发。对于本项目来说，测试算例仅仅用于通过求解器语法检查。

终端输入命令，拷贝测试算例

```terminal {fileName="terminal"}
cp -r $FOAM_TUTORIALS/incompressible/icoFoam/cavity/cavity debug_case
```

### 2.3. 脚本文件

使用脚本可以方便用户开发测试。以本项目为例，我们新建脚本如下

终端输入命令，新建脚本

```terminal {fileName="terminal"}
code caserun caseclean
```

脚本 caserun 主要是负责应用编译成功后，测试算例的运行，可以写入如下内容

```bash {fileName="/caserun",linenos=table,linenostart=1}
#!/bin/bash
# 首行告诉系统使用bash解释器执行脚本

# 对测试算例画网格，并输出日志到测试算例路径下
blockMesh -case debug_case | tee debug_case/log.mesh
# 输出信息“画网格完成”
echo "Meshing done."

# 对测试算例运行求解器，并输出日志到测试算例路径下
ofsp_07_firstApp -case debug_case | tee debug_case/log.run
```

脚本 caseclean 主要是负责清理应用到到编译前状态，如果应用要修改，那么测试算例也要还原到运行前的状态，可以写入如下内容

```bash {fileName="/caseclean",linenos=table,linenostart=1}
#!/bin/bash

# 删除测试算例中所有的日志文件
rm -rf debug_case/log.*

# 清理测试算例
foamCleanTutorials debug_case
# 输出信息“清理完成”
echo "Cleaning done."
```

### 2.4. 说明文件

为了方便后续阅读、开发和使用，我们还应该准备说明文件。

终端输入命令，新建说明文件

```terminal {fileName="terminal"}
code README.md
```

说明文件内容如下

```markdown {fileName="/README.md"}
## About

这是ofsp系列中，我们创建的第一个OpenFOAM标准应用。

## Bio

- Aerosand @ Aerosand

## Caution

需要使用 OpenFOAM v2406 及更新版本。

## Deploy

准备好环境和所有的文件。

在根目录下执行终端命令

清理并重新编译应用

1. wclean
2. wmake
   
清理并重新计算测试算例

1. ./caseclean
2. ./caserun

## Event

@ 20250903
- 增加清理脚本 #done

@ 20250901
- 新建应用 #done

```

建议遵循 **A**bout-**B**io-**C**aution-**D**eploy-**E**vent 的 A-B-C-D-E 原则书写说明文件，尽量表述清楚必要信息。

## 3. 文件结构

可以看到该项目的文件结构如下

终端输入命令

```terminal {fileName="terminal"}
tree
.
├── caseclean
├── caserun
├── debug_case
│   ├── 0
│   │   ├── p
│   │   └── U
│   ├── constant
│   │   ├── polyMesh
│   │   │   ├── boundary
│   │   │   ├── faces
│   │   │   ├── neighbour
│   │   │   ├── owner
│   │   │   └── points
│   │   └── transportProperties
│   ├── log.mesh
│   ├── log.run
│   └── system
│       ├── blockMeshDict
│       ├── controlDict
│       ├── decomposeParDict
│       ├── fvSchemes
│       ├── fvSolution
│       └── PDRblockMeshDict
├── Make
│   ├── files
│   └── options
└── ofsp_07_firstApp.C
```

## 4. 测试

终端输入命令，运行脚本

```terminal {fileName="terminal"}
wclean
wmake
./caseclean
./caserun
```

如果运行提醒 `Permission denied`，那就需要给脚本权限

```terminal {fileName="terminal"}
chmod +x caserun caseclean
```

虽然我们还没有给求解器写入任何开发代码，但是 OpenFOAM 提供的求解器基础模板在测试算例存在的情况下，仍然可以运行。

终端输出信息有两段

第一段是划分网格的输出日志

```terminal {fileName="terminal"}
/*---------------------------------------------------------------------------*\
| =========                 |                                                 |
| \\      /  F ield         | OpenFOAM: The Open Source CFD Toolbox           |
|  \\    /   O peration     | Version:  2406                                  |
|   \\  /    A nd           | Website:  www.openfoam.com                      |
|    \\/     M anipulation  |                                                 |
\*---------------------------------------------------------------------------*/
Build  : _9bfe8264-20241212 OPENFOAM=2406 patch=241212 version=2406
Arch   : "LSB;label=32;scalar=64"
Exec   : blockMesh -case debug_case
Date   : Sep 01 2025
Time   : 18:09:17
Host   : aerosand
PID    : 200553
I/O    : uncollated
Case   : /home/aerosand/github/data_project/openfoam_sharing/ofsp/ofsp_07_firstApp/debug_case
nProcs : 1
trapFpe: Floating point exception trapping enabled (FOAM_SIGFPE).
fileModificationChecking : Monitoring run-time modified files using timeStampMaster (fileModificationSkew 5, maxFileModificationPolls 20)
allowSystemOperations : Allowing user-supplied system call operations

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //
Create time

Creating block mesh from "system/blockMeshDict"
Creating block edges
No non-planar block faces defined
Creating topology blocks

Creating topology patches - from boundary section

Creating block mesh topology - scaling/transform applied later

Check topology

        Basic statistics
                Number of internal faces : 0
                Number of boundary faces : 6
                Number of defined boundary faces : 6
                Number of undefined boundary faces : 0
        Checking patch -> block consistency

Creating block offsets
Creating merge list (topological search)...

Creating polyMesh from blockMesh
Creating patches
Creating cells
Creating points with scale (0.1 0.1 0.1)
    Block 0 cell size :
        i : 0.005 .. 0.005
        j : 0.005 .. 0.005
        k : 0.01 .. 0.01

No patch pairs to merge

Writing polyMesh with 0 cellZones
----------------
Mesh Information
----------------
  boundingBox: (0 0 0) (0.1 0.1 0.01)
  nPoints: 882
  nCells: 400
  nFaces: 1640
  nInternalFaces: 760
----------------
Patches
----------------
  patch 0 (start: 760 size: 20) name: movingWall
  patch 1 (start: 780 size: 60) name: fixedWalls
  patch 2 (start: 840 size: 800) name: frontAndBack

End

Meshing done.
```

第二段是求解器运行的输出日志

```terminal {fileName="terminal"}
/*---------------------------------------------------------------------------*\
| =========                 |                                                 |
| \\      /  F ield         | OpenFOAM: The Open Source CFD Toolbox           |
|  \\    /   O peration     | Version:  2406                                  |
|   \\  /    A nd           | Website:  www.openfoam.com                      |
|    \\/     M anipulation  |                                                 |
\*---------------------------------------------------------------------------*/
Build  : _9bfe8264-20241212 OPENFOAM=2406 patch=241212 version=2406
Arch   : "LSB;label=32;scalar=64"
Exec   : ofsp_07_firstApp -case debug_case
Date   : Sep 01 2025
Time   : 18:09:17
Host   : aerosand
PID    : 200555
I/O    : uncollated
Case   : /home/aerosand/github/data_project/openfoam_sharing/ofsp/ofsp_07_firstApp/debug_case
nProcs : 1
trapFpe: Floating point exception trapping enabled (FOAM_SIGFPE).
fileModificationChecking : Monitoring run-time modified files using timeStampMaster (fileModificationSkew 5, maxFileModificationPolls 20)
allowSystemOperations : Allowing user-supplied system call operations

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //
Create time


ExecutionTime = 0 s  ClockTime = 0 s

End
```


## 5. 原生主源码

代码 `ofsp_07_firstApp.C` 为

```cpp {fileName="/ofsp_07_firstApp.C",linenos=table,linenostart=1}
/*---------------------------------------------------------------------------*\
  =========                 |
  \\      /  F ield         | OpenFOAM: The Open Source CFD Toolbox
   \\    /   O peration     |
    \\  /    A nd           | www.openfoam.com
     \\/     M anipulation  |
-------------------------------------------------------------------------------
    Copyright (C) 2025 AUTHOR,AFFILIATION
-------------------------------------------------------------------------------
License
    This file is part of OpenFOAM.

    OpenFOAM is free software: you can redistribute it and/or modify it
    under the terms of the GNU General Public License as published by
    the Free Software Foundation, either version 3 of the License, or
    (at your option) any later version.

    OpenFOAM is distributed in the hope that it will be useful, but WITHOUT
    ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or
    FITNESS FOR A PARTICULAR PURPOSE.  See the GNU General Public License
    for more details.

    You should have received a copy of the GNU General Public License
    along with OpenFOAM.  If not, see <http://www.gnu.org/licenses/>.

Application
    ofsp_07_firstApp

Description
	这部分是 OpenFOAM 的注释内容，提供该应用的必要介绍。
\*---------------------------------------------------------------------------*/

#include "fvCFD.H"
// 包含了和FVM有关的类、函数等

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    #include "setRootCase.H"
    // 建立命令行参数
    // 检查确定算例目录
    // 后续会详细讨论
    
    #include "createTime.H"
    // 创建并初始化时间控制对象Time
    // 后续会详细讨论

    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< nl;
    // Info是OpenFOAM提供打印信息、格式化输出的全局对象
    // nl是OpenFOAM提供的换行符(newline)
    
    runTime.printExecutionTime(Info);
    // 用来打印程序运行时间的函数调用

    Info<< "End\n" << endl;

    return 0;
}


// ************************************************************************* //
```

## 6. 使用开发库

在该项目中，我们有选择的使用上一篇讨论所开发的 Aerosand 库中的某个类。

### 6.1. 主源码

代码 `ofsp_07_firstApp.C` 修改为

```cpp {fileName="/ofsp_07_firstApp.C",linenos=table,linenostart=1}
#include "fvCFD.H"

#include "class1.H" // 调用我们想要的开发库头文件名称

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    #include "setRootCase.H"
    #include "createTime.H"


    label a = 1;
    scalar pi = 3.1415926;
    Info<< "Hi, OpenFOAM!" << " Here we are." << nl
	    << a << " + " << pi << " = " << a + pi << nl
	    << a << " * " << pi << " = " << a * pi << nl
	    << endl;

    class1 mySolver;
    mySolver.SetLocalTime(0.2);
    Info<< "\nCurrent time step is : " << mySolver.GetLocalTime() << endl;

    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< nl;
    runTime.printExecutionTime(Info);

    Info<< "End\n" << endl;

    return 0;
}
```

### 6.2. 项目 Make


项目 `Make/files` 为

```wmake {fileName="/Make/files"}
ofsp_07_firstApp.C

EXE = $(FOAM_USER_APPBIN)/ofsp_07_firstApp

```

> [!caution]
> 注意，这里需要链接的是上一篇讨论建立的开发库。

项目 `Make/options` 为

```wmake {fileName="/Make/options"}
EXE_INC = \
    -I$(LIB_SRC)/finiteVolume/lnInclude \
    -I$(LIB_SRC)/meshTools/lnInclude \
    -I../ofsp_07_tensor/Aerosand/lnInclude

EXE_LIBS = \
    -lfiniteVolume \
    -lmeshTools \
    -L$(FOAM_USER_LIBBIN) \
    -lAerosand

```

我们再次解释其写法

- 前缀 EXE 表示这是可执行文件的配置（前缀 LIB 表示是库文件的配置）
- 后缀 INC 表示包含路径
	- 使用 `-I` 标志指定头文件搜索路径
	- 通常指向 OpenFOAM 模块的 `lnInclude` 目录（链接包含目录）
	- 可以包含系统头文件路径或第三方库头文件路径
- 后缀 LIBS 表示库链接（直接链接无需链接任何库，可以留空不写）
	- 使用 `-l` 标志指定需要链接的库
	- 库名称不需要前缀 `lib` 和后缀 `.so`（如 `-lfiniteVolume` 对应 `libfiniteVolume.so`）
	- 使用 `-L` 标志指定库搜索路径
- 路径可以采用多种写法
	- `$(LIB_SRC)` 通常由 **OpenFOAM 环境变量**和 **相对路径**组合而成
	- 常见写为 `-I$(FOAM_SRC)/finiteVolume/lnInclude` 也可以
	- 绝对地址 `-I/usr/lib/openfoam/openfoam2306/src/finiteVolume/lnInclude`
	- 相对地址 `-I../ofsp_07_tensor/Aerosand/lnInclude`

## 7. 编译运行

因为上一篇讨论的 Aerosand 库已经编译成功，这里不用再次编译，直接调用即可。

```terminal {fileName="terminal"}
wclean
wmake
./caseclean
./caserun
```

> [!tip]
> 输入时可以常用 Tab 键进行快速补全。

终端输出如下（仅摘取有主要变化的片段）

```terminal {fileName="terminal"}
...

Create time

Hi, OpenFOAM! Here we are.
1 + 3.14159 = 4.14159
1 * 3.14159 = 3.14159


Current time step is : 0.2

ExecutionTime = 0.01 s  ClockTime = 0 s

End
```

## 8. 小结

本文完成讨论

- [x] 理解 OpenFOAM 标准应用的文件架构
- [x] 认识使用脚本
- [x] 认识应用开发测试流程
- [x] 调用其他位置的开发库
- [x] 编译运行 firstApp 项目



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
