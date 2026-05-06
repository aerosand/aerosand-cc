---
uid: 20251125114545
title: 18_fieldCal
date: 2025-11-25
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
weight: 19
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

基于前面的讨论，我们练习一个简单场计算的案例。

本文主要讨论

- [ ] 为计算增加简单的物理场
- [ ] 以 OpenFOAM 的方式管理物理场
- [ ] 练习开发库
- [ ] 编译运行 fieldCal 项目

## 1. 项目准备

终端输入命令，建立项目

```terminal {fileName="terminal"}
ofsp
foamNewApp ofsp_18_fieldCal
cd ofsp_18_fieldCal
cp -r $FOAM_TUTORIALS/incompressible/icoFoam/cavity/cavity debug_case
code .
```

测试初始求解器，提供脚本（参考前文脚本）和说明。

## 2. 增加物理场

拷贝过来的测试算例并没有温度场，我们给算例添加初始温度场。

### 2.1. 温度场

温度和压力一样是标量，可以通过拷贝压力场文件，修改后得到温度场文件。

终端输入命令，准备温度场文件

```terminal {fileName="terminal"}
cp -r debug_case/0/p debug_case/0/T
code debug_case/0/T
```

修改后得到的温度场文件为

```cpp {fileName="debug_case/0/T",linenos=table,linenostart=1}
FoamFile
{
    version     2.0;
    format      ascii;
    class       volScalarField;
    object      T;
}
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

// dimensions      [0 2 -2 0 0 0 0];
dimensions      [0 0 0 1 0 0 0];

// internalField   uniform 0;
internalField   uniform 2026;

boundaryField
{
    movingWall
    {
        // type            zeroGradient;
        type            fixedValue;
        value           uniform 2050;
    }

    fixedWalls
    {
        type            zeroGradient;
    }

    frontAndBack
    {
        type            empty;
    }
}

```

### 2.2. 主源码

主源码修改为

```cpp {fileName="ofsp_18_fieldCal.C",linenos=table,linenostart=1}
#include "fvCFD.H"

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //
scalar calculatePressure(scalar t, vector x, vector x0, scalar scale);
// 自定义函数原型声明


int main(int argc, char *argv[])
{
    #include "setRootCase.H"
    #include "createTime.H"

    #include "createMesh.H"

    Info<< "Reading transportProperties\n" << endl;
    IOdictionary transportProperties // 构造字典对象，从字典文件中读入参数
    (
        IOobject
        (
            "transportProperties", // 字典文件名称
            runTime.constant(), // 字典文件路径
            mesh, // 和 mesh 建立关系
            IOobject::MUST_READ_IF_MODIFIED,
            IOobject::NO_WRITE
        )
    );
    dimensionedScalar nu("nu",dimViscosity,transportProperties); // 读取字典文件中的参数

    Info<< "Reading field p\n" << endl;
    volScalarField p // 构造 p 场
    (
        IOobject // 基于 IOobject
        (
            "p", // 场文件的名称
            runTime.timeName(), // 场文件的路径
            mesh, // IOobject 和 mesh 建立关系
            IOobject::MUST_READ, // 必须要给初始条件
            IOobject::AUTO_WRITE
        ),
        mesh // 和 mesh 建立关系
    );

    Info<< "Reading field T\n" << endl;
    volScalarField T // 构造 T 场
    (
        IOobject
        (
            "T",
            runTime.timeName(),
            mesh,
            IOobject::MUST_READ,
            IOobject::AUTO_WRITE
        ),
        mesh
    );

    Info<< "Reading field U\n" << endl;
    volVectorField U // 构造 U 场
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

	// 举例创建空值场
	// 参考上文讨论的 GeometircField 构造函数
    volScalarField zeroScalarField
    (
        IOobject
        (
            "zeroScalarField",
            runTime.timeName(),
            mesh,
            IOobject::NO_READ, // 无需给初始条件的文件
            IOobject::AUTO_WRITE
        ),
        mesh,
        dimensionedScalar("zeroScalarField",dimless,Zero) // 初始化为零
    );

    volVectorField zeroVectorField
    (
        IOobject
        (
            "zeroVectorField",
            runTime.timeName(),
            mesh,
            IOobject::NO_READ,
            IOobject::AUTO_WRITE
        ),
        mesh,
        dimensionedVector("zeroVectorField",dimless,Zero) // 初始化为零
    );

    volVectorField gradT // 构造 gradT 场
    (
        IOobject
        (
            "gradT",
            runTime.timeName(),
            mesh,
            IOobject::NO_READ,
            IOobject::AUTO_WRITE
        ),
        fvc::grad(T) // 因为是从 T 场计算而来，所以不需要再次和 mesh 建立关系
    );

    Info<< "Reading/calculating face flux field phi\n" << endl;
    surfaceScalarField phi // 构造 phi 场
    (
        IOobject
        (
            "phi",
            runTime.timeName(),
            mesh,
            IOobject::READ_IF_PRESENT,
            IOobject::AUTO_WRITE
        ),
        fvc::flux(U)
        // fvc::interpolate(U)*mesh.Sf(); // 等同于此行计算
    );


    const vector originVector(0.05, 0.05, 0.005); // 构造带初值vector类的对象originVector

    const scalar rFarCell  = // 构造scalar类的对象rFarCell
    max // 求最大值
    (
        mag // 求数值大小
        (
            dimensionedVector("x0", dimLength, originVector)
            // 以originVector创建dimensionedVector
            - mesh.C() // 减去网格单元中心点坐标，实际上也是向量
        )
    ).value(); // 返回值的大小

    Info<< "\nStarting time loop\n" << endl;

    while (runTime.loop()) // 时间推进
    {
        Info<< "Time = " << runTime.timeName() << nl << endl;

        for (label cellI=0; cellI<mesh.C().size(); ++cellI) // 遍历所有网格单元
        {
            p[cellI] = calculatePressure // 通过自定义函数求 p 场
            (
                runTime.time().value(),
                mesh.C()[cellI],
                originVector,
                rFarCell
            );
        }

		// 假装计算速度场
        U = fvc::grad(p) * dimensionedScalar("tmp",dimTime,1.0); // 求 U 场
        // 额外乘以一个含单位临时量 tmp ，保证两边结果单位相同

        runTime.write(); // 写出计算的场
    }

    Info<< "Calculation done." << endl;

    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< nl;
    runTime.printExecutionTime(Info);

    Info<< "End\n" << endl;

    return 0;
}


// ************************************************************************* //

// 自定义函数的实现
scalar calculatePressure(scalar t, vector x, vector x0, scalar scale)
{
    scalar r(mag(x - x0) / scale); // 构造scalar类的对象r，并初始化

    scalar rR(1.0 / (r + 1e-12)); // 同上

    scalar f(1.0); // 同上

    return Foam::sin(2.0 * Foam::constant::mathematical::pi * f * t) * rR;
    // 返回一个计算值，并没有什么物理意义
}

```

### 2.3. 编译运行

编译运行此项目（注意不要清除温度场文件）。

终端输出如下

```terminal {fileName="terminal"}
Create time

Create mesh for time = 0

Reading transportProperties

Reading field p

Reading field T

Reading field U

Reading/calculating face flux field phi


Starting time loop

Time = 0.005

Time = 0.01

Time = 0.015

Time = 0.02

...

Time = 0.49

Time = 0.495

Time = 0.5

Calculation done.

ExecutionTime = 0.02 s  ClockTime = 0 s

End

```

终端输入命令，对计算结果可视化

```terminal {fileName="terminal"}
paraFoam -case debug_case
```

注意到 debug_case/ 的时间步文件夹下出现了新时间步的场文件，以 debug_case/0.3/ 为例，其包含的计算结果文件如下

```terminal {fileName="terminal"}
debug_case/0.3
├── gradT
├── p
├── phi
├── T
├── U
├── uniform
│   ├── functionObjects
│   │   └── functionObjectProperties
│   └── time
├── zeroScalarField
└── zeroVectorField
```

可以查看各个场文件中的计算结果数值。

>[!warning]
>打开 0.3/phi 可以看到并没有计算更新值，其他非 p,U 文件也没有计算更新值。这是因为场在创建时候给定的计算公式只是初始化，并不会随着时间推进而计算更新。所以，需要在主源码的时间循环中去计算需要的场。

## 3. 项目整理

在 OpenFOAM 实际应用中，一般的
- 场的接入应放入单独的文件 createFields.H
- OpenFOAM 本身提供 createPhi.H
- 自定义方法也放入单独文件，如 calculatePressure.H

所以该应用的文件结构调整后为

```terminal {fileName="terminal"}
tree -L 1
.
├── calculatePressure.H
├── caseclean
├── caserun
├── createFields.H
├── debug_case
├── Make
└── ofsp_18_fieldCal.C
```

### 3.1. createFields.H

场接入文件 createFields.H 为

```cpp {fielName="createFields.H",linenos=table,linenostart=1}
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

    dimensionedScalar nu("nu",dimViscosity,transportProperties);

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

    Info<< "Reading field T\n" << endl;
    volScalarField T
    (
        IOobject
        (
            "T",
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

    volScalarField zeroScalarField
    (
        IOobject
        (
            "zeroScalarField",
            runTime.timeName(),
            mesh,
            IOobject::NO_READ,
            IOobject::AUTO_WRITE
        ),
        mesh,
        dimensionedScalar("zeroScalarField",dimless,Zero)
    );

    volVectorField zeroVectorField
    (
        IOobject
        (
            "zeroVectorField",
            runTime.timeName(),
            mesh,
            IOobject::NO_READ,
            IOobject::AUTO_WRITE
        ),
        mesh,
        dimensionedVector("zeroVectorField",dimless,Zero)
    );

    volVectorField gradT
    (
        IOobject
        (
            "gradT",
            runTime.timeName(),
            mesh,
            IOobject::NO_READ,
            IOobject::AUTO_WRITE
        ),
        fvc::grad(T)
    );

    Info<< "Reading/calculating face flux field phi\n" << endl;
    surfaceScalarField phi
    (
        IOobject
        (
            "phi",
            runTime.timeName(),
            mesh,
            IOobject::READ_IF_PRESENT,
            IOobject::AUTO_WRITE
        ),
        fvc::flux(U)
        // fvc::interpolate(U)*mesh.Sf();
    );

```

### 3.2. calculatePressure.H

自定义方法 calculatePressure.H 为

```cpp {fileName="calculatePressure.H",linenos=table,linenostart=1}
scalar calculatePressure(scalar t, vector x, vector x0, scalar scale)
{
    scalar r(mag(x - x0) / scale);

    scalar rR(1.0 / (r + 1e-12));

    scalar f(1.0);

    return Foam::sin(2.0 * Foam::constant::mathematical::pi * f * t) * rR;
}

```

### 3.3. 主源码

主源码整理为

```cpp {fileName="ofsp_18_fieldCal.C",linenos=table,linenostart=1}
#include "fvCFD.H"

#include  "calculatePressure.H"

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    #include "setRootCase.H"
    #include "createTime.H"

    #include "createMesh.H"

    #include "createFields.H"

    const vector originVector(0.05, 0.05, 0.005);

    const scalar rFarCell  =
    max
    (
        mag
        (
            dimensionedVector("x0", dimLength, originVector)
            - mesh.C()
        )
    ).value();

    Info<< "\nStarting time loop\n" << endl;

    while (runTime.loop())
    {
        Info<< "Time = " << runTime.timeName() << nl << endl;

        for (label cellI=0; cellI<mesh.C().size(); ++cellI)
        {
            p[cellI] = calculatePressure
            (
                runTime.time().value(),
                mesh.C()[cellI],
                originVector,
                rFarCell
            );
        }

        U = fvc::grad(p) * dimensionedScalar("tmp",dimTime,1.0);

		phi = fvc::flux(U); // 增加 phi 的计算

        runTime.write();
    }

    Info<< "Calculation done." << endl;

    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< nl;
    runTime.printExecutionTime(Info);

    Info<< "End\n" << endl;

    return 0;
}

```

### 3.4. 编译运行

编译运行，可以看到结果是一样的。

项目整理后，

- 主源码功能划分非常清晰
- 每个功能都更加易于维护

## 4. 开发库

当某一类方法的实现可以保留为特定功能以便后续使用，所以可以将它们固定为开发库，以供各种项目随时调用。

回忆参考之前的讨论 https://aerosand.cc/docs/ofs/ofsp/03_wmake/

### 4.1. 开发库代码

我们建立开发库 computeVelocityPressure 并提供相应文件。

终端输入命令，建立开发库

```terminal {fileName="terminal"}
foamNewApp calculateVelocityPressure
```

建立开发库后，项目的文件结构为

```terminal {fileName="terminal"}
.
├── calculateVelocityPressure
│   ├── calculateVelocityPressure.C
│   ├── calculateVelocityPressure.H
│   └── Make
│       ├── files
│       └── options
├── caseclean
├── caserun
├── createFields.H
├── debug_case
│   ├── 0.org
│   │   ├── p
│   │   ├── T
│   │   └── U
│   ├── constant/
│   └── system/
├── Make
│   ├── files
│   └── options
└── ofsp_18_fieldCal.C
```

开发库声明 calculateVelocityPressure.H 为

```cpp {fileName="calculateVelocityPressure.H",linenos=table,linenostart=1}
#include "fvCFD.H"

scalar computeR(const fvMesh& mesh, volScalarField& r, dimensionedVector x0);

volScalarField computePressure(const fvMesh& mesh, scalar t, dimensionedVector x0, scalar scale,scalar f);

void computeVelocity(const fvMesh& mesh, volVectorField& U, word pName = "p");
// 第三个参数给了默认值

```

开发库定义 calculateVelocityPressure.C 为

```cpp {fileName="calculateVelocityPressure.C",linenos=table,linenostart=1}
#include "calculateVelocityPressure.H"

scalar calculateR(const fvMesh& mesh, volScalarField& r, dimensionedVector x0)
{
    r = mag(mesh.C() - x0); // 场 r 保存各个单元网格中心点向量到参考向量的距离

    return max(r).value(); // 返回各个距离的最大值
}

volScalarField calculatePressure // 计算压力场，注意返回类型
(
    const fvMesh& mesh, // 传入 mesh 的引用，引用传递效率高
    scalar t,
    dimensionedVector x0, // 不仅是 vector，而且是有单位制的 vector
    scalar scale, // 纯 scalar
    scalar f // 纯 scalar
)
{
    volScalarField r(mag(mesh.C()-x0)/scale);
    // 上一篇讨论中的 r 只是 scalar 类型，这里是 volScalarField 类型
    // 注意计算中变量类型的统一

    volScalarField rR(1.0/(r+dimensionedScalar("tmp",dimLength,1e-12)));
    // 赋予 1e-12 单位，以通过OpenFOAM对计算的单位检查
    // 注意计算中结果单位的统一

    return Foam::sin(2.0*Foam::constant::mathematical::pi*f*t)*rR*dimensionedScalar("tmp",pow(dimLength,3)/pow(dimTime,2),1.0);
    // 单位可以按数学方式计算
    // 同样为了保证结果单位一致，额外乘以含单位临时量 tmp
    // 该函数返回的计算结果的单位，等于，主函数要接收该值的变量的单位
}

void calculateVelocity(const fvMesh& mesh, volVectorField& U, word pName)
// 注意函数的实现不能再写上形参的默认值
{
    const volScalarField& pField = mesh.lookupObject<volScalarField>(pName);
    // 按 pName 字段查找场量，并赋值给 pField

    U = fvc::grad(pField) * dimensionedScalar("tmp",dimTime,1.0);
    // 计算 U，保证单位统一
}

```

### 4.2. 开发库 Make

开发库 Make/files 内容为

```wmake {fileName="calculateVelocityPressure/Make/files",linenos=table,linenostart=1}
calculateVelocityPressure.C

LIB = $(FOAM_USER_LIBBIN)/libcalculateVelocityPressure

```

开发库 Make/options 内容为

```wmake {fileName="calculateVelocityPressure/Make/options",linenos=table,linenostart=1}
EXE_INC = \
    -I$(LIB_SRC)/finiteVolume/lnInclude \
    -I$(LIB_SRC)/meshTools/lnInclude

EXE_LIBS = \
    -lfiniteVolume \
    -lmeshTools

```

该库，强调是这个库，在编译的时候，不会使用到其他更多的库，所以 options 中有这些基础库就够了，不需要另外增加。

### 4.3. 开发库编译

终端输入命令，编译开发库

```terminal {fileName="terminal"}
wmake calculateVelocityPressure
```

终端提示开发库编译成功，可以供项目使用。

### 4.4. 场的接入

保持 createFields.H 代码不变。

### 4.5. 主源码

主源码 ofsp_18_fieldCal.C 修改为

```cpp {fileName="ofsp_18_fieldCal.C",linenos=table,linenostart=1}
#include "fvCFD.H"

#include "calculateVelocityPressure.H"

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    #include "setRootCase.H"
    #include "createTime.H"

    #include "createMesh.H"

    #include "createFields.H"

    dimensionedVector originVector("x0",dimLength,vector(0.05,0.05,0.005));
    // 构造 dimensionedVector 类的对象 originVector，并赋单位和初始值

    scalar f(1.0);

    volScalarField r // 只是计算的过程量，所以没有放进 createFields.H 中
    (
        IOobject
        (
            "r",
            runTime.timeName(),
            mesh,
            IOobject::NO_READ,
            IOobject::NO_WRITE
        ),
        mesh,
        dimensionedScalar("r",dimLength,0.0)
    );

    Info<< "This is a test" << endl;

    const scalar rFarCell = calculateR(mesh,r,originVector);
    // 计算得到距离的最大值，并传给 rFarCell 变量保存

    Info<< "\nStarting time loop\n" << endl;

    while (runTime.loop()) // 时间推进
    {
        Info<< "Time = " << runTime.timeName() << nl << endl;

        p = calculatePressure(mesh,runTime.time().value(),originVector,rFarCell,f);
        // 计算 p 场

        calculateVelocity(mesh,U);
        // 计算 U 场

        phi = fvc::flux(U);
        // 计算 phi 场

        runTime.write(); // 写出场
    }

    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< nl;
    runTime.printExecutionTime(Info);

    Info<< "End\n" << endl;

    return 0;
}

```

### 4.6. 项目 Make

项目 Make/files 内容为

```wmake {fileName="Make/files",linenos=table,linenostart=1}
ofsp_18_fieldCal.C

EXE = $(FOAM_USER_APPBIN)/ofsp_18_fieldCal

```

项目 Make/options 内容为

```wmake {fileName="Make/options",linenos=table,linenostart=1}
EXE_INC = \
    -I$(LIB_SRC)/finiteVolume/lnInclude \
    -I$(LIB_SRC)/meshTools/lnInclude \
    -IcalculateVelocityPressure/lnInclude

EXE_LIBS = \
    -lfiniteVolume \
    -lmeshTools \
    -L$(FOAM_USER_LIBBIN) -lcalculateVelocityPressure

```

注意开发库的处理，既要“包含”，也要“链接”。既然指明”路径“，也要指明”名称“。

### 4.7. 编译运行

编译运行项目。

终端输出如下

```terminal {fileName="terminal"}
Create time

Create mesh for time = 0

Reading transportProperties

Reading field p

Reading field T

Reading field U

Reading/calculating face flux field phi

This is a test

Starting time loop

Time = 0.005

Time = 0.01

Time = 0.015

Time = 0.02

...

Time = 0.49

Time = 0.495

Time = 0.5


ExecutionTime = 0.02 s  ClockTime = 0 s

End
```

### 4.8. 后处理

我们同样可以通过 paraview 可视化计算结果。

终端输入命令，可视化计算结果。

```terminal {fileName="terminal"}
paraFoam -case debug_case
```

## 5. 小结

回顾之前的讨论，我们大概可以感受到数值计算的核心要素所在——时间、网格、物理场。有了这些工具之后，我们便可以将数学物理模型构建为数学方程，从而组建线性代数方程，进行数值求解。不要着急，我们距离求解器编程越来越近了。

本文完成讨论

- [x] 为计算增加简单的物理场
- [x] 以 OpenFOAM 的方式管理物理场
- [x] 练习开发库
- [x] 编译运行 fieldCal 项目

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
