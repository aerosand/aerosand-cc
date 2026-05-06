---
uid: 20250827192135
title: 06_tensor
date: 2025-08-27
update: 2026-03-20
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
weight: 7
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

上一篇基于 Vector 类讨论了一些代码细节，本文讨论 Tensor 类。

本文主要讨论

- [ ] 讨论 Tensor 类的部分代码实现
- [ ] 理解多类复杂库的编译和链接
- [ ] 编译运行 tensor 项目

## 1. Tensor 类

API 页面 https://api.openfoam.com/2506/classFoam_1_1Tensor.html

终端输入命令，本地查找

```terminal {fileName="terminal"}
find $FOAM_SRC -iname tensor
```

终端输入命令，打开该类的文件夹

```terminal {fileName="terminal"}
code $FOAM_SRC/OpenFOAM/primitives/Tensor
```

该类的文件结构如下

```terminal {fileName="terminal"}
tree -L 1
.
├── floats
├── ints
├── lists
├── Tensor.H
└── TensorI.H
```

查看 `Tensor/Tensor.H`，可以看到该类的实现细节。这里不再逐条阅读。

我们还可以通过 API 或者终端查找阅读相关的类

- `dimensionedTensor`
- `tensorField`

> [!warning]
> 暂不深究代码细节，大概了解成员函数的用法即可。


## 2. OFextension 插件

十分推荐在 vscode 中安装社区插件 `OFextension`。

### 2.1. 配置插件

1. 点击 vscode 左下角小齿轮，打开 `settings`
2. 搜索栏搜索 `ofextension`
3. 在 `Ofextension: OFpath` 中设置正确的 OpenFOAM 路径
4. 使用 vscode 打开用户的开发应用，使用 `F1` 输入 `ofInit` 初始化配置

### 2.2. 插件使用

在项目开发中，例如本文应用，在主源码中输入相关对象，vscode 会自动弹出可选的方法（成员函数）。

而且可以在主源码中选中头文件、类等，右键使用 `Go to Definition` ，`Go to Declaration` 等，直接跳转查看源代码。

非常推荐此插件，十分方便。注意避免在 OpenFOAM 源文件夹下初始化。

> [!warning]
> 注意有时候跳转的源代码并不是正确指向，要注意分辨。


## 3. 项目构建

终端输入命令，建立本文项目

```terminal {fileName="terminal"}
ofsp
mkdir ofsp_06_tensor
code ofsp_06_tensor
```

继续使用终端命令或者使用 vscode 界面创建其他文件，最终文件结构如下

```terminal {fileName="terminal"}
tree
.
├── Aerosand
│   ├── class1
│   │   ├── class1.C
│   │   └── class1.H
│   ├── class2
│   │   ├── class2.C
│   │   └── class2.H
│   └── Make
│       ├── files
│       └── options
├── Make
│   ├── files
│   └── options
└── ofsp_06_tensor.C
```

注意，开发库的文件结构与前文稍有不同。我们在前文已经可以注意到 OpenFOAM 库下一般有多个子库/类。用户的开发库里同样可能也会由好几个类构成，开发库拥有自己的 Make 文件，用于管理多个类，比如这里 Aerosand 库有 `class1` , `class2` 和 `class3` 三个类。

## 4. 开发库

### 4.1. class1

对于第一个类，我们依然使用之前的代码。

代码 `class1.H` 为

```cpp {fileName="/Aerosand/class1/class1.H",linenos=table,linenostart=1}
#pragma once

class class1
{
private:
    double localTime_;

public:
    void SetLocalTime(double t);
    double GetLocalTime() const;
};

```

代码 `class1.C` 为

```cpp {fileName="/Aerosand/class1/class1.C",linenos=table,linenostart=1}
#include "class1.H"

void class1::SetLocalTime(double t) {
    localTime_ = t;
}

double class1::GetLocalTime() const {
    return localTime_;
}

```

### 4.2. class2

对于第二个类，我们尝试通过继承来创建一个新类。

代码 `class2.H` 为

```cpp {fileName="/Aerosand/class2/class2.H",linenos=table,linenostart=1}
#pragma once

#include "vector.H"

namespace Foam
{

class class2 : public vector
{
public:
    // 继承构造函数
    using Foam::vector::vector;

    // 计算分量和
    Foam::scalar sum() const;
};

} // namespace Foam

```

代码 `class2.C` 为

```cpp {fileName="/Aerosand/class2/class2.C",linenos=table,linenostart=1}
#include "class2.H"

namespace Foam
{

Foam::scalar class2::sum() const
{
    return this->x() + this->y() + this->z();
}

} // namespace Foam

```

> [!tip]
> 注意声明和定义中使用的 scalar 和 vector 都属于 Foam 命名空间，所以需要使用该命名空间。

### 4.3. class3

对于第三个类，我们写一些简单的内容。

代码 `class3.H` 为

```cpp {fileName="/Aerosand/class3/class3.H",linenos=table,linenostart=1}
#pragma once

class class3
{
public:
    void class3Info() const;
};

```

代码 `class3.C` 为

```cpp {fileName="/Aerosand/class3/class3.C",linenos=table,linenostart=1}
#include "class3.H"

#include <iostream>

void class3::class3Info() const 
{
    std::cout << "This is class3\n" << std::endl;
}

```

### 4.4. 库 Make

库 `Make/files` 为

```wmake {fileName="/Aerosand/Make/files",linenos=table,linenostart=1}
class1/class1.C
class2/class2.C
class3/class3.C

LIB = $(FOAM_USER_LIBBIN)/libAerosand

```

本开发库没有其他依赖，库 `Make/options` 置空即可。

### 4.5. 库编译

终端输入命令，进行库的编译

```terminal {fileName="terminal"}
wclean Aerosand
wmake Aerosand
```

## 5. 主项目

### 5.1. 主源码

代码 `ofsp_06_tensor.C` 为

```cpp {fileName="/ofsp_06_tesnsor.C",linenos=table,linenostart=1}
#include "tensor.H"
#include "dimensionedTensor.H"
#include "tensorField.H"
// 调用 OpenFOAM 的类

#include "class1.H"
#include "class2.H"
#include "class3.H"
// 调用开发库中的类

using namespace Foam;

int main()
{
    scalar s(3.14); // scalar类已经被间接包含在 tensor.H 中
    vector v1(1, 2, 3); // vector类已经被间接包含在 tensor.H 中
    vector v2(0.5, 1, 1.5);
    
    // 下面使用tensor类的成员函数
    
    Info<< s << " * " << v1 << " = "  << s * v1 << nl 
        << "pos(s): " << pos(s) << nl
        << "asinh(s): " << Foam::asinh(s) << nl
        << endl;

    tensor T(11, 12, 13, 21, 22, 23, 31, 32, 33);
    Info<< "T: " << T << nl 
        << "Txy: " << T.xy() << nl
        << endl;

    tensor T1(1, 2, 3, 4, 5, 6, 7, 8, 9);
    tensor T2(1, 2, 3, 1, 2, 3, 1, 2, 3);
    tensor T3 = T1 + T2;
    Info<< "T3: " << T3 << nl << endl;

    tensor T4(3, -2, 1, -2, 2, 0, 1, 0, 4);
    Info<< "T4': " << inv(T4) << nl
        << "T4' * T4: " << (inv(T4) & T4) << nl
        << "T4.x(): " << T4.x() << nl
        << "T4.y(): " << T4.y() << nl
        << "T4.z(): " << T4.z() << nl
        << "T4^T: " << T4.T() << nl
        << "det(T4): " << det(T4) << nl
        << endl;

	// 下面使用dimensionedTensor类的成员函数
	
    dimensionedTensor sigma
    (
        "sigma",
        dimensionSet(1, -2, -2, 0, 0, 0, 0),
        tensor(1e6, 0, 0, 0, 1e6, 0, 0, 0, 1e6)
    );
    Info<< "sigma: " << sigma << nl 
        << "sigma name: " << sigma.name() << nl
        << "sigma dimension: " << sigma.dimensions() << nl
        << "sigma value: " << sigma.value() << nl
        << "sigma yy value: " << sigma.value().yy() << nl
        << endl;

	// 下面使用tensorField类的成员函数
	
    tensorField tf(2, tensor::one);
    Info<< "tf: " << tf << endl;
    tf[0] = tensor(1, 2, 3, 4, 5, 6, 7, 8, 9);
    tf[1] = T2;
    Info << "tf: " << tf << nl
        << "2.0 * tf" << 2.0 * tf << nl
        << endl;

	label a = 1;
    scalar pi = 3.1415926;
    Info<< "Hi, OpenFOAM!" << " Here we are." << nl
	    << a << " + " << pi << " = " << a + pi << nl
	    << a << " * " << pi << " = " << a * pi << nl
	    << endl;

	// class1 构造和输出
    class1 mySolver;
    mySolver.SetLocalTime(0.2);
    Info<< "\nCurrent time step is : " << mySolver.GetLocalTime() << endl;
	
	// class2 构造和输出
	class2 v3(2,4,6); // class2继承了vector的构造函数
    Info<< "Sum of vector components: " << v3.sum() << endl;

	// class3 构造和输出
    class3 myMessage;
    myMessage.class3Info();

    return 0;
}

```

### 5.2. 项目 Make

项目 `Make/files` 为

```wmake {fileName="/Make/files",linenos=table,linenostart=1}
ofsp_06_tensor.C

EXE = $(FOAM_USER_APPBIN)/ofsp_06_tensor

```


项目 `Make/options` 为

```wmake {fileName="/Make/options",linenos=table,linenostart=1}
EXE_INC = \
    -IAerosand/lnInclude

EXE_LIBS = \
    -L$(FOAM_USER_LIBBIN) \
    -lAerosand

```


同样的，`$FOAM_SRC/OpenFOAM` 库已经自动依赖，其中类的使用均无需用户再次链接。

## 6. 编译运行

终端输入命令，编译运行该项目

```terminal {fileName="terminal"}
wclean
wmake
ofsp_06_tensor
```

运行结果如下

```terminal {fileName="terminal"}
3.14 * (1 2 3) = (3.14 6.28 9.42)
pos(s): 1
asinh(s): 1.86181

T: (11 12 13 21 22 23 31 32 33)
Txy: 12

T3: (2 4 6 5 7 9 8 10 12)

T4': (1.33333 1.33333 -0.333333 1.33333 1.83333 -0.333333 -0.333333 -0.333333 0.333333)
T4' * T4: (1 0 0 1.66533e-16 1 0 -5.55112e-17 0 1)
T4.x(): (3 -2 1)
T4.y(): (-2 2 0)
T4.z(): (1 0 4)
T4^T: (3 -2 1 -2 2 0 1 0 4)
det(T4): 6

sigma: sigma [1 -2 -2 0 0 0 0] (1e+06 0 0 0 1e+06 0 0 0 1e+06)
sigma name: sigma
sigma dimension: [1 -2 -2 0 0 0 0]
sigma value: (1e+06 0 0 0 1e+06 0 0 0 1e+06)
sigma yy value: 1e+06

tf: 2{(1 1 1 1 1 1 1 1 1)}
tf: 2((1 2 3 4 5 6 7 8 9) (1 2 3 1 2 3 1 2 3))
2.0 * tf2((2 4 6 8 10 12 14 16 18) (2 4 6 2 4 6 2 4 6))

Hi, OpenFOAM! Here we are.
1 + 3.14159 = 4.14159
1 * 3.14159 = 3.14159


Current time step is : 0.2
Sum of vector components: 12
This is class3
```


## 7. 小结

本文完成讨论

- [x] 讨论 Tensor 类的部分代码实现
- [x] 理解多类复杂库的编译和链接
- [x] 编译运行 tensor 项目

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
