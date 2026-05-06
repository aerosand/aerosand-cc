---
uid: 20250916165944
title: 09_keyword
date: 2025-09-16
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
weight: 10
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

让我们停下脚步，仔细想一下前一篇文章的项目。我们很难称它是按关键词读取参数，因为代码只是实现了机械读取。如果我们更改输入文件的关键词顺序，就会导致传入参数的不一致（读者可以尝试更换 `ofspProperties` 文件中关键词顺序，重新运行代码，查看结果）。

我们来考虑一下如何真正的按关键词读取。

本文主要讨论

- [ ] 了解 OpenFOAM 读取参数的方式
- [ ] 使用 C++ 方式实现按关键词读参
- [ ] 编译运行 keyword 项目

## 1. OpenFOAM 读参

>[!tip]
>OpenFOAM 的读参方式随着架构更新已经发生了较大的变化。我们以较早版本的代码为例，方便读者理解。

参考：https://github.com/OpenFOAM/OpenFOAM-2.0.x/blob/master/applications/solvers/basic/laplacianFoam/createFields.H

我们看一下 OpenFOAM 中一些简单类型的参数的读取语句是什么样子的。

在 `laplacianFoam/createFields.H` 文件中可以看到以下语句

```cpp {fileName="laplacianFoam/createFields.H",linenos=table,linenostart=1}
....

    Info<< "Reading transportProperties\n" << endl;

    IOdictionary transportProperties
    (
        IOobject
        (
            "transportProperties", // 文件名称（基于下面参数的路径）
            runTime.constant(), // 路径即 constant
            mesh, // 关联网格
            IOobject::MUST_READ_IF_MODIFIED, // 如果修改必须读取
            IOobject::NO_WRITE // 不写出
        )
    );
    // 基于IOobject构造IOdictionary类的transportProperties对象
    // IOobject是基于"transportProperties"文件名称（路径）和其他参数构造的


    Info<< "Reading diffusivity DT\n" << endl;

    dimensionedScalar DT
    (
        transportProperties.lookup("DT")
    );
    // transportProperties对象有lookup函数，按照关键词"DT"来读取关键词后的值
    
```

虽然 `lookup()` 的语法在 OpenFOAM 中有点过时，但是相对来说非常直观。

简单概括如下，

- `IOdictionary` 是和文件读取相关的类
- `IOdictionary` 基于外部文件 `transportProperties` 构造对象 `transportProperties`
- 对象 `transportProperties` 具有 `IOdictionary` 类的方法，比如 `lookup()`
- 也就是说，对象 `transportProperties` 可以使用 `lookup()` 方法

我们尝试自己动手模仿，实现 OpenFOAM 的这个读取功能。

## 2. 实现按关键词读参

### 2.1. 项目准备

终端输入命令，建立本项目

```terminal {fileName="terminal"}
ofsp
foamNewApp ofsp_09_keyword
code ofsp_09_keyword
```

为该项目新建文件，最终项目架构如下

```terminal {fileName="terminal"}
.
├── IOdictionary
│   ├── IOdictionary.C
│   ├── IOdictionary.H
│   └── Make
│       ├── files
│       └── options
├── Make
│   ├── files
│   └── options
├── ofsp_09_keyword.C
└── ofspProperties
```

### 2.2. 自开发 IOdictionary

我们设想，自开发的 `IOdictionary` 类也可以有类似 OpenFOAM 的语法表达。简化的形式如下

```cpp
// 想要的语法效果
IOdictionary ofspProperties // 构造类的实体
(
	"ofspProperties" // 外部文件的名称，根目录路径
);

double viscosity = ofspProperties.lookup("nu"); // 类的实体可以使用方法
```

其中，对象 `ofspProperties` 是类 `IOdictionary` 基于文件名（路径） `ofspProperties` 的实例化。对象 `ofspProperties` 可以使用类 `IOdictionary` 中定义的方法 `lookup()`。

为了实现自开发类 `IOdictionary` 可以读取外部文件的功能，我们考虑仍然使用 C++ 原生 `fstream` 类。

类的声明 `IOdictionary.H` 内容如下

```cpp {fileName="/IOdictionary/IOdictionary.H",linenos=table,linenostart=1}
#pragma once

#include <iostream>
#include <fstream>
#include <string>

namespace Aerosand // 自定义命名空间，避免命名重复
{

class IOdictionary
{
    
private:
    std::ifstream dictionary_;
    std::string filePath_;

public:

    // Constructor

        IOdictionary(const std::string& filePath);

    // Destructor

        ~IOdictionary();

    // Member Function

        std::string lookup(const std::string& keyword);

};

}

```


> [!tip]
> 尽量贴合 OpenFOAM 的代码风格。

类的定义 `IOdictionary.C` 内容如下

```cpp {fileName="/IOdictionary/IOdictionary.C",linenos=table,linenostart=1}
#include "IOdictionary.H"

// Constructor

Aerosand::IOdictionary::IOdictionary(const std::string& filePath)
{
    filePath_ = filePath;
}

// Destructor

Aerosand::IOdictionary::~IOdictionary()
{
    if (dictionary_.is_open())
    {
        dictionary_.close();
    }
}

// Member Function

std::string Aerosand::IOdictionary::lookup(const std::string& keyword)
{
    dictionary_.open(filePath_);

    std::string returnValue;

    std::string line;

    while (std::getline(dictionary_, line))
    {
	    // 截取关键词后的字符串
        size_t keywordPos = line.find(keyword);
        if (keywordPos != std::string::npos)
        {
            returnValue = line.substr(keywordPos + keyword.length());
        }

		// 剔除所有的空格
        size_t spacePos = returnValue.find(" ");
        while (spacePos != std::string::npos)
        {
            returnValue.erase(spacePos, 1);
            spacePos = returnValue.find(" ");
        }
    }

    dictionary_.close(); 

    return returnValue;
}

```

我们还需要定义库 Make

文件 `IOdictionary/Make/files` 内容如下

```wmake {fileName="/IOdictionary/Make/files"}
IOdictionary.C

LIB = $(FOAM_USER_LIBBIN)/libIOdictionary

```

因为没有使用到其他库，文件 `IOdictionary/Make/options` 留空即可。

终端输入命令，编译生成动态库

```terminal {fileName="terminal"}
wclean IOdictionary
wmake IOdictionary
```

### 2.3. 主项目

主源码 `ofsp_09_keyword.C` 内容如下

```cpp {fileName="/ofsp_09_keyword.C",linenos=table,linenostart=1}
#include <iostream>
#include <fstream>

#include "IOdictionary.H"

int main(int argc, char const *argv[])
{
    Aerosand::IOdictionary ofspProperties("ofspProperties");

    std::string viscosity = ofspProperties.lookup("nu");
    std::string application = ofspProperties.lookup("application");
    std::string deltaT = ofspProperties.lookup("deltaT");
    std::string writeInterval = ofspProperties.lookup("writeInterval");
    std::string purgeWrite = ofspProperties.lookup("purgeWrite");

    std::cout << "nu\t\t" << viscosity << std::endl;
    std::cout << "application\t" << application << std::endl;
    std::cout << "deltaT\t\t" << deltaT << std::endl;
    std::cout << "writeInterval\t" << writeInterval << std::endl;
    std::cout << "purgeWrite\t" << purgeWrite << std::endl;

    return 0;
}

```

我们还需要定义项目 Make

文件 `ofsp_09_keyword/Make/files` 内容如下

```wmake {fileName="/Make/files"}
ofsp_01_IO_keyword.C

EXE = $(FOAM_USER_APPBIN)/ofsp_01_IO_keyword

```

文件 `ofsp_09_keyword/Make/options` 内容如下

```wmake {fileName="/Make/options"}
EXE_INC = \
    -IIOdictionary/lnInclude

EXE_LIBS = \
    -L$(FOAM_USER_LIBBIN) \
    -lIOdictionary

```

### 2.5. 编译运行

我们在该项目的根目录下提供类似 OpenFOAM 字典的文件。`ofspProperties` 文件内容如下（路径为 `/ofsp/ofsp_09_io/ofspProperties`）

```cpp {fileName="/ofspProperties"}
nu                  0.01
application         laplacianFoam
deltaT              0.005
writeInterval       20
purgeWrite          0
```

终端输入命令，编译运行该项目

```terminal {fileName="terminal"}
wclean
wmake

ofsp_09_keyword
```

终端输出内容如下

```terminal {fileName="terminal"}
nu              0.01
application     laplacianFoam
deltaT          0.005
writeInterval   20
purgeWrite      0
```

即使我们调整外部文件字典中的参数顺序，例如

```cpp {fileName="/ofspProperties"}
nu                  0.01
application         laplacianFoam

writeInterval       20
purgeWrite          0

deltaT              0.005
```

无需重新编译项目，直接运行该项目

终端输入命令

```terminal {fileName="terminal"}
ofsp_09_keyword
```

最后的结果是一样的。

## 3. 小结

从上面的讨论可见，重构项目的思路基本正确。但是读取得到的参数类型统一是 `string` 类型，还是不符合我们最后的要求，我们需要进一步开发功能特性。

本文完成讨论

- [x] 了解 OpenFOAM 读取参数的方式
- [x] 使用 C++ 方式实现按关键词读参
- [x] 编译运行 keyword 项目

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
