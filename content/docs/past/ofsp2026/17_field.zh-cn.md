---
uid: 20251117135613
title: 17_field
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
weight: 18
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

回忆前文，我们已经讨论了输入输出、命令行参数、时间和网格。OpenFOAM 的计算绝大多数是基于物理场的，本项目开始讨论 OpenFOAM 中的“场”。

本文主要讨论

- [ ] 理解场的代码结构
- [ ] 理解场的计算参与
- [ ] 编译运行 field 项目

## 1. 项目准备

终端输入命令，建立项目

```terminal {fileName="terminal"}
ofsp
foamNewApp ofsp_17_field
cd ofsp_17_field
cp -r $FOAM_TUTORIALS/incompressible/icoFoam/cavity/cavity debug_case
code .
```

测试初始求解器，提供脚本和说明。

## 2. 场的创建

### 2.1. dimensionSet

前文用到了 OpenFOAM 的单位预设，其定义在 `$FOAM_SRC/OpenFOAM/dimensionSet/dimensionSets.C` ，文件中还定义了很多预设单位

再次摘阅如下

```cpp {fileName="dimensionSets.C",linenos=table,linenostart=1}
const dimensionSet dimless;

const dimensionSet dimMass(1, 0, 0, 0, 0, 0, 0);
const dimensionSet dimLength(0, 1, 0, 0, 0, 0, 0);
const dimensionSet dimTime(0, 0, 1, 0, 0, 0, 0);
const dimensionSet dimTemperature(0, 0, 0, 1, 0, 0, 0);
const dimensionSet dimMoles(0, 0, 0, 0, 1, 0, 0);
const dimensionSet dimCurrent(0, 0, 0, 0, 0, 1, 0);
const dimensionSet dimLuminousIntensity(0, 0, 0, 0, 0, 0, 1);

const dimensionSet dimArea(sqr(dimLength));
const dimensionSet dimVolume(pow3(dimLength));
const dimensionSet dimVol(dimVolume);

const dimensionSet dimVelocity(dimLength/dimTime);
const dimensionSet dimAcceleration(dimVelocity/dimTime);

const dimensionSet dimDensity(dimMass/dimVolume);
const dimensionSet dimForce(dimMass*dimAcceleration);
const dimensionSet dimEnergy(dimForce*dimLength);
const dimensionSet dimPower(dimEnergy/dimTime);

const dimensionSet dimPressure(dimForce/dimArea);
const dimensionSet dimCompressibility(dimDensity/dimPressure);
const dimensionSet dimGasConstant(dimEnergy/dimMass/dimTemperature);
const dimensionSet dimSpecificHeatCapacity(dimGasConstant);
const dimensionSet dimViscosity(dimArea/dimTime);
const dimensionSet dimDynamicViscosity(dimDensity*dimViscosity);
```

同样暂不深究代码实现细节。

我们可以设置自定义变量的物理单位，如

```cpp
dimensionSet dimAerosand(1,2,3,4,5,6,7); // just for example
```

### 2.2. dimensionedType

查找类的定义

API 页面 https://api.openfoam.com/2506/dimensionedType_8H.html

终端输入命令，查找源代码

```terminal {fileName="terminal"}
find $FOAM_SRC -iname dimensionedType.H
```

找到并摘取其中的构造函数如下

```cpp {fileName="dimensionedType.H",linenos=table,linenostart=1}
template<class Type>
class dimensioned
{
...
public:
...
	// Constructors
		...
		//- Construct from components (name, dimensions, value).
        dimensioned
        (
            const word& name,
            const dimensionSet& dims,
            const Type& val
        );
	    ...
	...
...

```

相应的，在 OpenFOAM 代码中可以看到如下类似的构造

```cpp {linenos=table,linenostart=1}
dimensionedScalar airPressure
(
	"airPressure",
	dimensionSet(1, -1, -2, 0, 0, 0, 0),
	1.0
);

```

在 dimensionedType.H 中也定义了配套的成员函数，比如

```cpp {fileName="dimensionedType.H",linenos=table,linenostart=1}
...
	//- Return const reference to value.
	const Type& value() const noexcept { return value_; }

	//- Return non-const reference to value.
	Type& value() noexcept { return value_; }
...
```

相应的，我们可以便捷的使用 value() 成员函数，例如对上面的 airPressure 取值，再计算最大值。

```cpp
maxP = max(airPressure.value());
```

当然 OpenFOAM 中还提供了其他的构造函数和成员函数，我们可以在不同情况下构造自己的有单位物理量。目前阶段来说，我们不需要继续深挖源代码。点到为止，明白确实可以这样用，并且以后按照这种方式去用就可以了。

注意，OpenFOAM 会检查数学计算式等号两遍的最终单位是否相同，如果不同则会强制报错并中断程序，后续代码会面临并解决这个问题。

### 2.3. volFields

我们会注意到 OpenFOAM 中有典型的语句如下

```cpp {linenos=table,linenostart=1}
...
    Info<< "Reading fieldp\\n" << endl;
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
...

```

那么 volScalarField 是什么呢？又是如何构造场的呢？

API 页面 

终端输入命令，查阅源文件

```terminal {fileName="terminal"}
find $FOAM_SRC -iname volFieldsFwd.H
```

> [!tip]
> 也许有同学还有疑惑“我怎么知道要查看这个文件”。参考 06_tensor 的讨论，安装 OFextension 插件后，直接在 volScalarField 关键字上右键跳转即可。也可以在 API 页面搜索关键词，以阅读相关页面。

代码 volFieldsFwd.H 中有类型定义如下

```cpp {fileName="volFieldsFwd.H"}
...
typedef GeometricField<scalar, fvPatchField, volMesh> volScalarField;
typedef GeometricField<vector, fvPatchField, volMesh> volVectorField;
...
```

可以看到，volFields 包括常见的 volScalarField 和 volVectorField，是体积场量。无论是 volScalarField 还是 volVectorField 其实都是 GeometricField 的模板的不同情况。所以， volFields 只是一种模板类特化分组的称呼，而不是真正的类。

### 2.4. surfaceFields

同样的，面场量 surfaceFields 包括常见的 surfaceScalarField 和 surfaceVectorField 。

API 页面 https://api.openfoam.com/2506/surfaceFieldsFwd_8H.html

终端输入命令，查阅源文件

```terminal {fileName="terminal"}
find $FOAM_SRC -iname surfaceFieldsFwd.H
```

代码 surfaceFieldsFwd.H 中有类型定义如下

```cpp {fileName="surfaceFieldsFwd.H"}
...
typedef GeometricField<scalar, fvsPatchField, surfaceMesh> surfaceScalarField;
typedef GeometricField<vector, fvsPatchField, surfaceMesh> surfaceVectorField;
...
```

可见，surfaceFields 也是 GeometricField 的模板的不同情况。

>[!tip]
>surfaceFields 的 surface 都可以写完整，为什么 volFields 的 volume 要简写成 vol 呢？也许是历史代码的原因吧。。。

### 2.5. GeometricField

基于上文的讨论，我们查阅 GeometricField.H 代码

API 页面 https://api.openfoam.com/2506/GeometricField_8H.html

终端输入命令，查阅源文件

```terminal {fileName="terminal"}
find $FOAM_SRC -iname GeometricField.H
```

摘取类的主要结构和几个常用构造函数如下

```cpp {fileName="GeometricField.H",linenos=table,linenostart=1}
...
template<class Type, template<class> class PatchField, class GeoMesh>
class GeometricField
:
    public DimensionedField<Type, GeoMesh>
{
public:
	...
private:
	...
public:
	...
	// Constructors

	//- Construct given IOobject, mesh, dimensioned<Type> and patch type.
	//  This assigns both dimensions and values.
	//  The name of the dimensioned\\<Type\\> has no influence.
	GeometricField
	(
		const IOobject& io,
		const Mesh& mesh,
		const dimensioned<Type>& dt,
		const word& patchFieldType = PatchField<Type>::calculatedType()
	);
	...
	//- Construct and read given IOobject
	GeometricField
	(
		const IOobject& io,
		const Mesh& mesh,
		const bool readOldTime = true
	);
	...
	//- Construct as copy resetting IO parameters
	GeometricField
	(
		const IOobject& io,
		const GeometricField<Type, PatchField, GeoMesh>& gf
	);
	...
...

```

上述构造函数，对应实际的场的构造，分别举例如下

```cpp {linenos=table,linenostart=1}
...
	Info<< "Creating field light pressure\\n" << endl;
	volScalarField lightP
	(
		IOobject
		(
			"lightP",
			runTime.timeName(),
			mesh,
			IOobject::NO_READ,
			IOobaject::AUTO_WRITE
		),
		mesh,
		dimensionedScalar("lightP",dimPressure,0.0)
		// 以构造的 dimensioned<Type>& 类型的变量初始化
		// 基于名称，单位，初始值
		// find $FOAM_SRC -iname dimensionedType.H
	);
...
	Info<< "Reading field p\\n" << endl;
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
...
	Info<< "Calculating field mixture density\\n" << endl;
	volScalarField rho
	(
		IOobject
		(
			"rho",
			runTime.timeName(),
			mesh,
			IOobject::NO_READ,
			IOobject::AUTO_WRITE
		),
		alpha1*rho1 + alpha2*rho2
		// 传入计算场
	);
...
```

这三个例子对应了最常见的场的构造函数，其他构造函数以及更深入的代码层面的解析在以后会陆续介绍，无需担心。读者也可以尝试自行多阅读几个。

## 3. 场的时间推进

我们以压力场为例，结合场的创建，讨论一下场的时间推进计算。

### 3.1. 单时间步计算

主源码 ofsp_17_field.C 内容如下

```cpp {fileName="ofsp_17_field/ofsp_17_field.C",linenos=table,linenostart=1}
#include "fvCFD.H"

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    #include "setRootCase.H"
    #include "createTime.H"

    #include "createMesh.H"
    // 常见三个头文件，以后不再赘述

    volScalarField p // 基于 IOobject 和 mesh 创建压力场p
    (
        IOobject // 基于以下参数创建 IOobject
        (
            "p",
            runTime.timeName(),
            mesh,
            IOobject::MUST_READ,
            IOobject::AUTO_WRITE // 根据源代码中的设置自动写入
        ),
        mesh
    );
    // 读取可选：
	//     MUST_READ，必须读取，如果文件不存在则报错
	//     READ_IF_PRESENT，如果存在才读取，如果文件格式错误则报错
	//     NO_READ，不读取
	// 写入可选：
	//     AUTO_WRITE，根据要求自动写入
	//     NO_WRITE，不写入，但是可以通过代码显式写入

    runTime++; // 推进一个时间步
    // 如果省略此行，则不推进时间步，计算结果会写入到初始时间步，覆盖掉初始场

    forAll(p, i) // 遍历压力场
    {
        if (mesh.C()[i][1] < 0.5)
        // 还记得 mesh.C() 返回什么吗？
        // 返回网格中心点坐标的y坐标值（C++从0开始编号）
        {
            p[i] = i*runTime.value(); // 随便给一个可能的计算
        }
    }
    // 完成了整个压力场的计算

    Info<< "Max p = " << max(p).value() << endl; // 直接返回压力场最大值
    Info<< "Min p = " << min(p).value() << endl; // 返回压力场最小值
    Info<< "Average p = " << average(p).value() << endl; // 返回压力场平均值
    Info<< "Sum p = " << sum(p).value() << endl; // 返回压力场总和值

    p.write(); // 显式的代码，要求只写入该时间步的压力场值

    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< nl;
    runTime.printExecutionTime(Info);

    Info<< "End\n" << endl;

    return 0;
}

```

>[!tip]
>注意到主源码中的场的覆写是不区分初始时间步和后续时间步的，而且 foamCleanTutorials 命令不能恢复初始时间步文件夹内的场文件的内容。所以，为了避免初始条件被覆盖，强烈推荐备份初始条件。一旦初始场被覆盖，可以随时复原。

#### 3.1.1. 脚本

脚本 caserun 修改为

```bash {fileName="caserun",linenos=table,linenostart=1}
#!/bin/sh
cd "${0%/*}" || exit 1                              # Run from this directory
#------------------------------------------------------------------------------

appPath=$(cd `dirname $0`; pwd)                     # Get application path
appName="${appPath##*/}"                            # Get application name

cd debug_case

FOLDER=$(pwd) 
FOLDER0=$FOLDER/0
FOLDER0_ORG=$FOLDER/0.org

echo "Setting up case directories..."

# 确保 0.org 存在
if ! test -d "$FOLDER0_ORG"; then
    if test -d "$FOLDER0"; then
        echo "Creating 0.org from existing 0 folder"
        cp -r "$FOLDER0" "$FOLDER0_ORG"
    else
        echo "Error: Neither 0.org nor 0 folder exists!"
        echo "Please ensure at least one of these directories exists."
        exit 1
    fi
else
    echo "0.org folder exists"
fi

# 确保 0 存在且是 0.org 的精确副本
if test -d "$FOLDER0"; then
    echo "Removing existing 0 folder"
    rm -rf "$FOLDER0"
fi

echo "Creating 0 folder from 0.org"
cp -r "$FOLDER0_ORG" "$FOLDER0"

echo "Directory setup complete. Running application..."

blockMesh

# $appName > log.run # if run in the background to save time
$appName | tee log.run

```

脚本 caseclean 修改为

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

可以看到修改后的脚本更加通用。后续如非特别说明，可以直接拷贝使用此脚本。

#### 3.1.2. 编译运行

终端输入命令，编译运行项目

```terminal {fileName="terminal"}
wclean
wmake
./caseclean
./caserun
```

终端输出如下

```terminal {fileName="terminal"}
Create time

Create mesh for time = 0

Max p = 1.995
Min p = 0
Average p = 0.9975
Sum p = 399

ExecutionTime = 0.01 s  ClockTime = 0 s

End
```

查看 debug_case/ 文件夹下，多了 0.005/ 文件夹，其中的 0.005/p 文件中写入了计算的场。最后的输出就是最新时间步的场的值。

### 3.2. 多时间步计算

如何在多个时间步上推进计算呢？

主源码修改如下

```cpp {fileName="ofsp_17_filed/ofsp_17_field.C",linenos=table,linenostart=1}
#include "fvCFD.H"

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    #include "setRootCase.H"
    #include "createTime.H"

    #include "createMesh.H"

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

    // runTime++;

    for (int i=0; i < 3; ++i) // 计算 3 个时间步
    {
        ++runTime; // 时间推进

        forAll(p, i)
        {
            if (mesh.C()[i][1] < 0.5)
            {
                p[i] = i*runTime.value();
            }
        }
        p.write(); // 写入该时间步
    }

    Info<< "Max p = " << max(p).value() << endl;
    Info<< "Min p = " << min(p).value() << endl;
    Info<< "Average p = " << average(p).value() << endl;
    Info<< "Sum p = " << sum(p).value() << endl;

    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< nl;
    runTime.printExecutionTime(Info);

    Info<< "End\n" << endl;

    return 0;
}

```

>[!tip]
>关于写成 ++runTime 而不是 runTime++ 的讨论
>- 原则上来说，前置递增 ++a 比 后置递增 a++ 更高效
>	- 前置递增 ++a 直接递增并返回自身，避免潜在拷贝
>	- 后置递增则需要创建临时副本再增加并返回临时副本
>- 对于内置类型（int, float, 指针等）现代编译器一般会优化掉差异
>- 对于复杂类型（类、迭代器）前置更高效

编译运行项目

终端输出如下

```terminal {fileName="terminal"}
Create time

Create mesh for time = 0

Max p = 5.985
Min p = 0
Average p = 2.9925
Sum p = 1197

ExecutionTime = 0.01 s  ClockTime = 0 s

End
```

观察 debug_case/ 文件夹下多了三个时间步的文件夹，现在的测试算例文件结构如下

```terminal {fileName="terminal"}
tree debug_case/
debug_case/
├── 0
│   ├── p
│   └── U
├── 0.005
│   └── p
├── 0.01
│   └── p
├── 0.015
│   └── p
├── 0.org
│   ├── p
│   └── U
├── constant
│   ├── polyMesh/...
│   └── transportProperties
├── log.run
└── system/...
```

查看各个时间步的压力场值，可以知道终端最后输出的值为最新时间步的计算值。

### 3.3. 字典控制计算

即使是多时间步计算，我们也是直接修改主源码的计算时间步个数，每次修改都要重新编译整个应用。如果是大型应用，重新编译会相当麻烦。

在实际工作中，我们希望在应用编译完成后，也可以通过外部文件去控制计算，也就是使用 OpenFOAM 的字典文件 controlDict 控制计算。

主源码修改为

```cpp {fileName="ofsp_17_field/ofsp_17_field.C",linenos=table,linenostart=1}
#include "fvCFD.H"

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    #include "setRootCase.H"
    #include "createTime.H"

    #include "createMesh.H"

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

    while(runTime.loop()) // 时间推进
    {
        forAll(p, i)
        {
            if (mesh.C()[i][1] < 0.5)
            {
                p[i] = i*runTime.value();
            }
        }
		runTime.write(); // 按字典要求写入
    }

    Info<< "Max p = " << max(p).value() << endl;
    Info<< "Min p = " << min(p).value() << endl;
    Info<< "Average p = " << average(p).value() << endl;
    Info<< "Sum p = " << sum(p).value() << endl;

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

Max p = 199.5
Min p = 0
Average p = 99.75
Sum p = 39900

ExecutionTime = 0 s  ClockTime = 0 s

End
```

测试算例 debug_case/ 文件多了几个时间步文件夹，分别是 0.1, 0.2, 0.3, 0.4, 0.5，写入的时间步由 system/controlDict 控制。

字典 controlDict 的部分参数如下（每 `0.005*20 = 0.1` 写入一次）

```cpp {fileName="debug_case/system/controlDict",linenos=table,linenostart=1}
...
endTime         0.5;

deltaT          0.005;

writeControl    timeStep;

writeInterval   20;
...
```

这些参数都可以随时在字典中修改测试，无需重新编译项目。

## 4. 小结

本文介绍了 OpenFOAM 中的“场”。从代码和项目练习中，读者可以理解场是如何参与到计算项目中的。

本文完成讨论

- [x] 理解场的代码结构
- [x] 理解场的计算参与
- [x] 编译运行 field 项目


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


