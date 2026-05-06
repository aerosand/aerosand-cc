---
uid: 20251117115757
title: 16_meshRatio
date: 2025-11-17
update: 2026-02-04
authors:
  - name: Aerosand
    link: https://github.com/aerosand
    image: https://github.com/aerosand.png
tags:
  - ofsp
  - ofsp2026
  - OpenFOAM
excludeSearch: false
toc: true
weight: 17
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

我们基于前文对网格的讨论，来尝试一个计算网格参数的应用，便于读者练习和理解网格以及 OpenFOAM 项目流程。

本项目代码来自于 Tom Smith 讲义《programming with OpenFOAM》（Tutorial at The 3rd UCL OpenFOAM Workshop）。

本文主要讨论

- [ ] 练习网格类方法的使用
- [ ] 考虑代码的运行效率
- [ ] 复习自定义字典文件的使用
- [ ] 编译运行 meshRatio 项目

## 1. 项目准备

终端输入命令，建立项目

```terminal {fileName="terminal"}
ofsp
foamNewApp ofsp_16_meshRatio
cd ofsp_16_meshRatio
cp -r $FOAM_TUTORIALS/incompressible/simpleFoam/pitzDaily debug_case
code .
```

测试初始求解器，提供脚本和说明。

## 2. 网格体积比

主源码修改为

```cpp {fileName="ofsp_16_meshRatio/ofsp_16_meshRatio.C",linenos=table,linenostart=1}
#include "fvCFD.H"

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    #include "setRootCase.H"
    #include "createTime.H"

    #include "createMesh.H"

    const scalar c = mesh.C().size();
    Info<< "Number of cells = " << c << endl;
    // 网格单元数量

    const labelListList& neighbour = mesh.cellCells();
    // 返回邻单元为新列表

    List<scalar> ratios(0); // 声明一个scalar列表且全初始化为0
    scalar volumeRatio = 0;

    forAll (neighbour, cellI) // 遍历邻单元，其实就是所有单元
    {
        List<label> n = neighbour[cellI];
        // 对每个单元，获得它的邻单元到列表 n

        const scalar cellVolume = mesh.V()[cellI];
        // 本单元体积

        forAll (n, i) // 对于某个本单元，遍历它的邻单元
        {
            label neighbourIndex = n[i];
            scalar neighbourVolume = mesh.V()[neighbourIndex];
            // 邻单元体积

            if (neighbourVolume >= cellVolume) // 体积比大于 1 的情况
            {
                volumeRatio = neighbourVolume / cellVolume;
                ratios.append(volumeRatio);
                // 体积比存入 ratios 列表
                // 方便但是效率低
            }
        }
    }

    Info<< "Maximum volume ratio = " << max(ratios) << endl;
    // 最大体积比


    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< nl;
    runTime.printExecutionTime(Info);

    Info<< "End\n" << endl;

    return 0;
}

```

执行 caserun 脚本，或者直接对测试算例划分网格，运行求解器

```terminal {fileName="terminal"}
wmake
blockMesh -case debug_case
ofsp_16_meshRatio -case debug_case
```

终端输出如下

```terminal {fileName="terminal"}
Create time

Create mesh for time = 0

Number of cells = 12225
Maximum volume ratio = 1.49908

ExecutionTime = 0.11 s  ClockTime = 0 s

End
```

## 3. 改进体积比

List 类的 append 方法虽然方便，但是效率很低，因为每一次向 List 添加数据的时候，都要创建一个 n+1 大小的新 List ，将老数据拷贝过去，再加入新数据。在数据量很大的时候，每次添加新数据都有新建再复制的过程，效率非常低。

我们考虑到对于要处理的网格，因为网格确定，所以要计算的体积比总量也是确定的。可以提前声明一个固定大小的 List，分配好内存，后续只需要向其添加数据就行。

改进后主源码如下

```cpp {fileName="ofsp_16_meshRatio/ofsp_16_meshRatio.C",linenos=table,linenostart=1}
#include "fvCFD.H"

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    #include "setRootCase.H"
    #include "createTime.H"

    #include "createMesh.H"

    const scalar c = mesh.C().size();
    Info<< "Number of cells = " << c << endl;

    const labelListList& neighbour = mesh.cellCells();

	/*
	* 代码改进如下
	*/
    label len = mesh.Cf().size(); // 需要的 ratio 的 List 大小
    scalar initial = 0;
    List<scalar> ratios(len, initial); // 创建固定大小的 List
    label counter = 0; // ratios 的序号
    scalar volumeRatio = 0;

    forAll (neighbour, cellI) // 遍历邻单元
    {
        List<label> n = neighbour[cellI];

        const scalar cellVolume = mesh.V()[cellI];

        forAll (n, i)
        {
            label neighbourIndex = n[i];
            scalar neighbourVolume = mesh.V()[neighbourIndex];

            if (neighbourVolume >= cellVolume)
            {
                volumeRatio = neighbourVolume / cellVolume;
                ratios[counter] = volumeRatio; // 存储体积比
                counter += 1;
            }
        }
    }

    Info<< "Maximum volume ratio = " << max(ratios) << endl;

    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< nl;
    runTime.printExecutionTime(Info);

    Info<< "End\n" << endl;

    return 0;
}

```

终端输入命令，编译运行

终端输出如下

```terminal {fileName="terminal"}
Create time

Create mesh for time = 0

Number of cells = 12225
Maximum volume ratio = 1.49908

ExecutionTime = 0.05 s  ClockTime = 0 s

End
```

可以很明显看到运行时间从 0.13s 减少到了 0.06s，效率大大提高。

## 4. 最大体积比

如果我们只是要获得最大体积比，那么其实我们不需要保存所有的体积比，保存体积比的最大值就够了。

主源码如下

```cpp {fileName="ofsp_16_meshRatio/ofsp_16_meshRatio.C",linenos=table,linenostart=1}
#include "fvCFD.H"

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    #include "setRootCase.H"
    #include "createTime.H"

    #include "createMesh.H"

    const scalar c = mesh.C().size();
    Info<< "Number of cells = " << c << endl;

    const labelListList& neighbour = mesh.cellCells();

	/*
	* 代码改进如下
	*/
    scalar volumeRatio = 0.0;
    scalar currentRatio = 0.0;

    forAll (neighbour, cellI)
    {
        List<label> n = neighbour[cellI];

        const scalar cellVolume = mesh.V()[cellI];

        forAll (n, i)
        {
            label neighbourIndex = n[i];
            scalar neighbourVolume = mesh.V()[neighbourIndex];

            if (neighbourVolume >= cellVolume)
            {
                volumeRatio = neighbourVolume / cellVolume;

                if (volumeRatio > currentRatio) // 只存储最大体积比
                {
                    currentRatio = volumeRatio;
                }
            }
        }
    }

    Info<< "Maximum volume ratio = " << currentRatio << endl;

    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< nl;
    runTime.printExecutionTime(Info);

    Info<< "End\\n" << endl;

    return 0;
}

```

编译运行项目

终端输出如下

```terminal {fileName="terminal"}
Create time

Create mesh for time = 0

Number of cells = 12225
Maximum volume ratio = 1.49908

ExecutionTime = 0.05 s  ClockTime = 0 s

End
```


## 5. 体积比准则

仅获得体积比还不够，我们可能希望知道测试算例的网格有多少超过了指定的体积比准则。

我们通过 OpenFOAM 的字典来接入体积比准测这一参数。

为测试算例提供字典 /ofsp_16_meshRatio/debug_case/system/volumeRatioDict ，内容如下

```cpp {fileName="ofsp_16_meshRatio/debug_case/system/volumeRatioDict",linenos=table,linenostart=1}
FoamFile
{
    version     2.0;
    format      ascii;
    class       dictionary;
    object      blockMeshDict;
}
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

maxRatio 1.2;

```

用户可以通过字典文件修改参数而不用再次编译项目。

> [!tip]
> 我们约定
> - `Properties` 文件放在 `/userApp/constant` 文件夹下
> - `Dict` 文件放在 `/userApp/system` 文件夹下

主源码如下

```cpp {fileName="ofsp_16_meshRatio/ofsp_16_meshRatio.C",linenos=table,linenostart=1}
#include "fvCFD.H"

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    #include "setRootCase.H"
    #include "createTime.H"

    #include "createMesh.H"

    IOdictionary volumeRatioDict // 建立字典对象
    (
        IOobject
        (
            "volumeRatioDict", // 文件名称
            runTime.system(), // 文件路径
            mesh, // 基于 mesh 构造
            IOobject::MUST_READ, // 必读
            IOobject::NO_WRITE // 只读不写
        )
    );

    const scalar c = mesh.C().size();
    Info<< "Number of cells = " << c << endl;

    const labelListList& neighbour = mesh.cellCells();

    scalar volumeRatio = 0.0;
    scalar currentRatio = 0.0;

    label nFail = 0; // 统计超出准测的数量

    scalar maxRatio(readScalar(volumeRatioDict.lookup("maxRatio")));
	// 从字典读入

    forAll (neighbour, cellI)
    {
        List<label> n = neighbour[cellI];

        const scalar cellVolume = mesh.V()[cellI];

        forAll (n, i)
        {
            label neighbourIndex = n[i];
            scalar neighbourVolume = mesh.V()[neighbourIndex];

            if (neighbourVolume >= cellVolume)
            {
                volumeRatio = neighbourVolume / cellVolume;

                if (volumeRatio > currentRatio)
                {
                    currentRatio = volumeRatio;
                }

                if (volumeRatio > maxRatio) // 统计超出准测的数量
                {
                    nFail += 1;
                }
            }
        }
    }

    Info<< "Maximum volume ratio = " << currentRatio << nl
        << "Number of cell volume ratios exceeding " << maxRatio
        << " = " << nFail << nl
        << endl;

    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< nl;
    runTime.printExecutionTime(Info);

    Info<< "End\n" << endl;

    return 0;
}

```

编译运行项目

终端输出如下

```terminal {fileName="terminal"}
Create time

Create mesh for time = 0

Number of cells = 12225
Maximum volume ratio = 1.49908
Number of cell volume ratios exceeding 1.2 = 532


ExecutionTime = 0.04 s  ClockTime = 0 s

End
```

## 6. 小结

本文继续讨论了网格类的方法，提供了实践案例。读者通过反复练习，可以回顾之前的知识点，克服 OpenFOAM 编程陌生感，方便后续学习。

本文完成讨论

- [x] 练习网格类方法的使用
- [x] 考虑代码的运行效率
- [x] 复习自定义字典文件的使用
- [x] 编译运行 meshRatio 项目

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
