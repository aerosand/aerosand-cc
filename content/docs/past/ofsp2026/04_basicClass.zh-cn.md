---
uid: 20250826191442
title: 04_basicClass
date: 2025-08-26
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
weight: 5
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

前面我们已经了解了 C++ 原生实现、make 实现以及 OpenFOAM 的 wmake 实现。在正式进入到 OpenFOAM 实现之前，我们有必要稍微了解一点 OpenFOAM 中常见的类，以方便使用。

我们可以通过官方 API  https://api.openfoam.com/2506/ 进行查询。

> [!tip]
> 改变上面网址的末尾版本号，可以打开对应版本的 API。

本文主要讨论

- [ ] 查阅 OpenFOAM API 
- [ ] 认识一些常见的基础类


## 1. Switch

可以通过 API 网站直接搜索 `Switch` 关键词找到 Switch Class Reference 页面，该页面提供了 Switch 类的相关内容，包括对应的 Source files 即 `Switch.H` 和 `Switch.C`。

API 页面 https://api.openfoam.com/2506/Switch_8H.html

也可以通过终端查找打开阅读源码。


终端输入命令，查找源码位置

```terminal {fileName="terminal"}
find $FOAM_SRC -iname switch.H
```

终端输出对应版本下的查找结果

```terminal {fileName="terminal"}
/usr/lib/openfoam/openfoam2406/src/OpenFOAM/primitives/bools/Switch/Switch.H
/usr/lib/openfoam/openfoam2406/src/OpenFOAM/lnInclude/Switch.H
```

基于上一篇的讨论，我们复制正确的路径，可以通过 vscode 本地阅读源码

```terminal {fileName="terminal"}
code /usr/lib/openfoam/openfoam2406/src/OpenFOAM/primitives/bools/Switch/Switch.H
```

> [!tip]
> 终端里的复制粘贴请使用 `ctrl + shift + c/v`

无论如何，`Switch.H` 部分代码如下（点击代码块名称可以跳转源码）

```cpp {fileName="Switch.H",base_url="https://api.openfoam.com/2506/Switch_8H_source.html",linenos=table,linenostart=1}
/*---------------------------------------------------------------------------*\
  =========                 |
  \\      /  F ield         | OpenFOAM: The Open Source CFD Toolbox
   \\    /   O peration     |
    \\  /    A nd           | www.openfoam.com
     \\/     M anipulation  |
-------------------------------------------------------------------------------
    Copyright (C) 2011-2016 OpenFOAM Foundation
    Copyright (C) 2017-2023 OpenCFD Ltd.
-------------------------------------------------------------------------------
License
	...

Class
    Foam::Switch

Description
    A simple wrapper around bool so that it can be read as a word:
    true/false, on/off, yes/no, any/none.
    Also accepts 0/1 as a string and shortcuts t/f, y/n.

SourceFiles
    Switch.C

\*---------------------------------------------------------------------------*/

#ifndef Foam_Switch_H
#define Foam_Switch_H

#include "bool.H"
#include "stdFoam.H"

...
```

根据描述 Description 可以知道 Switch 是对 bool 类型的封装，起到和 bool 类型类似的判断功能。

更深入的代码定义，我们暂不深究。

## 2. label

同样可以在网上或是本地找到 label 类的源码。

API 页面 https://api.openfoam.com/2506/label_8H.html

```cpp {fileName="label.H"}
/*
...
 Description
     A label is an int32_t or int64_t as specified by the pre-processor macro
     WM_LABEL_SIZE.
 
     A readLabel function is defined so that label can be constructed from
     Istream.
...
*/
```


根据源码描述，label 类型为 int32_t 或 int64_t，具体取决于预处理器宏 WM_LABEL_SIZE 的定义。

简单可以理解 label 实际上是对 int 类型的封装。

## 3. scalar

API 页面 https://api.openfoam.com/2506/scalar_8H.html

```cpp {filename="scalar.H"}
/*
...
 Description
     A floating-point number identical to float or double depending on
     whether WM_SP, WM_SPDP or WM_DP is defined.
...
*/
```

根据源码描述，scalar 类型可以理解为浮点类型的封装，根据不同使用方式等同于 float 或者 double。

## 4. vector

API 页面 https://api.openfoam.com/2506/vector_8H.html

```cpp {fileName="vector.H"}
/*
...
 Description
     A Vector of values with scalar precision,
     where scalar is float/double depending on the compilation flags.
...
*/
```

可见，vector 类型是由 scalar 构成的向量。该类也定义了向量的数学运算方法。

>[!note]
>由其他数据类型构成的向量是什么呢？

## 5. tensor

API 页面 https://api.openfoam.com/2506/tensor_8H.html

```cpp {fileName="tensor.H"}
/*
...
 Description
     Tensor of scalars, i.e. Tensor<scalar>.
 
     Analytical functions for the computation of complex eigenvalues and
     complex eigenvectors from a given tensor.
...
*/
```

可见，tensor 类型是由 scalar 构成的张量。该类也定义了张量的数学运算方法。

>[!note]
>由其他数据类型构成的张量是什么呢？

## 6. dimensionedScalar/Vector/Tensor

以 dimensionedScalar 为例，API 页面 https://api.openfoam.com/2506/dimensionedScalar_8H_source.html

```cpp {fileName="dimensionedScalar.H"}
/*
...
 Description
     Dimensioned scalar obtained from generic dimensioned type.
...
*/
```

可见，这些是具有 OpenFOAM 单位系统的包含对应计算方法的标量、向量、张量。

## 7. scalar/vector/tensorField

以 scalarField 为例，API 页面 https://api.openfoam.com/2506/scalarField_8H.html

它们本质上就是 scalar/vector/tensor 类型的包含了对应计算方法的列表值，也就是“场”的数值存储。

## 8. dimensionSet

API 页面 https://api.openfoam.com/2506/dimensionSet_8H.html

```cpp {fileName="dimensionSet.H"}
/*
...
 Description
     Dimension set for the base types, which can be used to implement
     rigorous dimension checking for algebraic manipulation.
 
     The dimensions are specified in the following order
     (SI units for reference only):
     \table
         Property    | SI Description        | SI unit
         MASS        | kilogram              | \c kg
         LENGTH      | metre                 | \c m
         TIME        | second                | \c s
         TEMPERATURE | Kelvin                | \c K
         MOLES       | mole                  | \c mol
         CURRENT     | Ampere                | \c A
         LUMINOUS_INTENSITY | Candela        | \c cd
     \endtable
...
*/
```

该类规定了基础类型的单位系统，可以在代数计算中进行严格的单位检查。

通过 API 页面可以看到该类直接或者间接包含的文件。可以在源码定义中提供了很多实用的单位组合。

```cpp {fileName="dimensionSets.C",linenos=table,linenostart=1}
...
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
...
```

这个预设单位组合会经常出现在以后的编程实践中。

## 9. tmp

API 页面 https://api.openfoam.com/2506/tmp_8H.html

```cpp {fileName="dimensionedScalar.H"}
/*
...
 Description
     A class for managing temporary objects.
 
     This is a combination of std::shared_ptr (with intrusive ref-counting)
     and a shared_ptr without ref-counting and null deleter.
     This allows the tmp to double as a pointer management and an indirect
     pointer to externally allocated objects.
     In contrast to std::shared_ptr, only a limited number of tmp items
     will ever share a pointer.
...
*/
```

该类是 OpenFOAM 实现的、常见用于自动管理临时对象生命周期的智能指针模板类。主要目的是优化性能，避免不必要的大型数据（比如巨大数据量的物理场）拷贝。

## 10. IOobject

API 页面 https://api.openfoam.com/2506/IOobject_8H.html

简单来说，该类提供了所有需要在磁盘（作为文件）和内存（作为对象）之间进行读写操作的对象的完整描述和接口。

在 CFD 模拟中，有成千上万的变量（场、边界条件、模型参数等）需要从字典文件中读取，或者将计算结果写入磁盘。该类通过将所有这些杂事封装到一个统一的类中来解决输入输出问题。

以后我们将在实践中不断使用该类。

## 11. 小结

> [!warning]
> 暂时不用深究这些类定义的编程实现细节，这些细节并不是本系列的重点。过度深挖细节会给初学者带来巨大的困扰，打乱学习进度。

本文完成讨论

- [x] 查阅 OpenFOAM API 
- [x] 认识一些常见的基础类

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
