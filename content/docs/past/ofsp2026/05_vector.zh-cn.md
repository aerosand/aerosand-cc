---
uid: 20250827141152
title: 05_vector
date: 2025-08-27
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
weight: 6
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

上一篇浏览了几个常见的基础类，本文尝试讨论 vector 类的部分细节，帮助读者熟悉开发流程、工具使用、编译原理。

本文主要讨论

- [ ] 练习源码的查阅
- [ ] 讨论 vector 类的部分代码实现
- [ ] 理解原生库的使用
- [ ] 编译运行 vector 项目

## 1. Vector 类

API 页面 https://api.openfoam.com/2506/classFoam_1_1Vector.html

终端输入命令，本地查找

```terminal {fileName="terminal"}
find $FOAM_SRC -iname vector
```

终端输入命令，打开该类的文件夹

```terminal {fileName="terminal"}
code $FOAM_SRC/OpenFOAM/primitives/Vector
```

该类的文件结构如下

```terminal {fileName="terminal"}
tree -L 1
.
├── bools/
├── complex/
├── floats/
├── ints/
├── lists/
├── Vector.H
└── VectorI.H
```

阅读 `Vector.H` 的代码描述可知，该类是 3D Vector 的模板类，继承自 VectorSpace 类并添加封装了 3 分量的构造、分量元素的接口函数、点积和叉积的等方法。

具体到 `Vector.H` 的代码，简单方法在代码中直接实现，复杂方法则大量使用内联函数（inline function）以提高代码性能。OpenFOAM 特别提供 `VectorI.H` 文件来写内联函数的实现。

进入 `Vector/int` 文件夹，查看 `Vector/ints/labelVector.H` 代码

```cpp {filName="Vector/ints/labelVector.H"}
...
typedef Vector<label> labelVector;
...
```

进入 `Vector/floats` 文件夹，查看 `Vector/floats/vector.H` 代码

```cpp {fileName="Vector/floats/vector.H",linenos=table,linenostart=1}
...
//! \class Foam::floatVector
//! \brief A Vector of values with float precision
typedef Vector<float> floatVector;

//! \class Foam::doubleVector
//! \brief A Vector of values with double precision
typedef Vector<double> doubleVector;

// With float or double precision (depending on compilation)
typedef Vector<scalar> vector;

// With float or double precision (depending on compilation)
typedef Vector<solveScalar> solveVector;
...
```

可以看到，代表不同数据类型的文件夹 `/bools` ， `/complex` ， `/floats` ，`/ints` ，`/lists` 都是 Vector 模板类对不同基本数据类型的 `typedef` ，也就是类型别名。

> [!warning]
> 暂不深究代码的所有细节

## 2. 源码讨论

### 2.1. 声明

查阅 `Vector/Vector.H` 代码，我们分段讨论

API 页面 https://api.openfoam.com/2506/Vector_8H_source.html

>[!tip]
>API 的代码页可以点击跳转到不同的头文件、类、函数等。

```cpp {fileName="Vector/Vector.H",linenos=table,linenostart=1}
#ifndef Foam_Vector_H
#define Foam_Vector_H
// 预处理指令，防止头文件被重复包含

#include "contiguous.H" // 提供类型连续性检查的功能
#include "VectorSpace.H" // 基类

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

namespace Foam // 命名空间
{

// Forward Declarations
template<class T> class List; // 前向声明
// 声明了 List 模板类，后续可能会用到，但暂时不用提供完整头文件

/*---------------------------------------------------------------------------*\
                           Class Vector Declaration
\*---------------------------------------------------------------------------*/

template<class Cmpt>
class Vector // 定义 Vector 模板类
:
    public VectorSpace<Vector<Cmpt>, Cmpt, 3> // 继承自基类，3维向量空间
{
public:

    // Typedefs // 定义类型别名

        //- Equivalent type of labels used for valid component indexing
        typedef Vector<label> labelType;
        // 使用 label 类型作为分量


    // Member Constants

        //- Rank of Vector is 1
        static constexpr direction rank = 1; // 定义秩为一阶


    //- Component labeling enumeration
    enum components { X, Y, Z }; // 枚举对应向量的分量


    // Generated Methods

        //- Default construct
        Vector() = default; // 默认构造

        //- Copy construct
        Vector(const Vector&) = default; // 拷贝构造

        //- Copy assignment
        Vector& operator=(const Vector&) = default; // 拷贝赋值


    // Constructors

        //- Construct initialized to zero
        inline Vector(const Foam::zero); // 置零构造

        //- Copy construct from VectorSpace of the same rank
        template<class Cmpt2>
        inline Vector(const VectorSpace<Vector<Cmpt2>, Cmpt2, 3>& vs);
        // 从另一种分量类型的向量空间构造向量

        //- Construct from three components
        inline Vector(const Cmpt& vx, const Cmpt& vy, const Cmpt& vz);
        // 从三个分量值直接构造向量

        //- Construct from Istream
        inline explicit Vector(Istream& is);
        // 从输入流读取并构造向量
        // explicit 关键字防止隐式转换


    // Member Functions

    // Component Access

        //- Access to the vector x component
        const Cmpt& x() const noexcept { return this->v_[X]; }

        //- Access to the vector y component
        const Cmpt& y() const noexcept { return this->v_[Y]; }

        //- Access to the vector z component
        const Cmpt& z() const noexcept { return this->v_[Z]; }
        
        // 使用枚举值 X, Y, Z 作为索引访问基类的 v_ 数组
        // noexcept 表示这些函数不会抛出异常

        //- Access to the vector x component
        Cmpt& x() noexcept { return this->v_[X]; }

        //- Access to the vector y component
        Cmpt& y() noexcept { return this->v_[Y]; }

        //- Access to the vector z component
        Cmpt& z() noexcept { return this->v_[Z]; }
        
        // 声明非常量成员函数，用于访问和修改向量的分量


    // Vector Operations

        //- Return \c this (for point which is a typedef to Vector\<scalar\>)
        inline const Vector<Cmpt>& centre
        (
            const Foam::UList<Vector<Cmpt>>&  /* (unused) */
        ) const noexcept;
        // 定义在内联函数中
        // 返回向量本身（对于点类型很有用）
        // 参数未使用，但为了接口一致性而保留

        //- The length (L2-norm) of the vector
        inline scalar mag() const;
        // 定义在内联函数中
        // 返回向量的模（长度）

        //- The length (L2-norm) squared of the vector.
        inline scalar magSqr() const;
        // 定义在内联函数中
        // 返回向量模的平方（避免开方运算，提高性能）

        //- The L2-norm distance from another vector.
        //- The mag() of the difference.
        inline scalar dist(const Vector<Cmpt>& v2) const;
        // 定义在内联函数中
        // 返回与另一个向量的距离

        //- The L2-norm distance squared from another vector.
        //- The magSqr() of the difference.
        inline scalar distSqr(const Vector<Cmpt>& v2) const;
        // 定义在内联函数中
        // 返回与另一个向量距离的平方

        //- Inplace normalise the vector by its magnitude
        //  For small magnitudes (less than ROOTVSMALL) set to zero.
        //  Will not be particularly useful for a vector of labels
        inline Vector<Cmpt>& normalise(const scalar tol = ROOTVSMALL);
        // 定义在内联函数中
        // 将向量归一化（转换为单位向量）
        // tol 参数是一个容差值，用于处理非常小的向量

        //- Inplace removal of components that are collinear to the given
        //- unit vector.
        inline Vector<Cmpt>& removeCollinear(const Vector<Cmpt>& unitVec);
        // 定义在内联函数中
        // 移除与给定单位向量共线的分量

        //- Scalar-product of \c this with another Vector.
        inline Cmpt inner(const Vector<Cmpt>& v2) const;
        // 定义在内联函数中
        // 返回与另一个向量的点积（内积）

        //- Cross-product of \c this with another Vector.
        inline Vector<Cmpt> cross(const Vector<Cmpt>& v2) const;
        // 定义在内联函数中
        // 返回与另一个向量的叉积（外积）


    // Comparison Operations

        //- Lexicographically compare \em a and \em b with order (x:y:z)
        static inline bool less_xyz
        (
            const Vector<Cmpt>& a,
            const Vector<Cmpt>& b
        );
        // 定义在内联函数中
        // 按 X→Y→Z 的字典序比较两个向量

        //- Lexicographically compare \em a and \em b with order (y:z:x)
        static inline bool less_yzx
        (
            const Vector<Cmpt>& a,
            const Vector<Cmpt>& b
        );
        // 定义在内联函数中
        // 按 Y→Z→X 的字典序比较两个向量

        //- Lexicographically compare \em a and \em b with order (z:x:y)
        static inline bool less_zxy
        (
            const Vector<Cmpt>& a,
            const Vector<Cmpt>& b
        );
        // 定义在内联函数中
        // 按 Z→X→Y 的字典序比较两个向量
};


// * * * * * * * * * * * * * * * * * Traits  * * * * * * * * * * * * * * * * //

// 类型特性,暂不深究

//- Data are contiguous if component type is contiguous
template<class Cmpt>
struct is_contiguous<Vector<Cmpt>> : is_contiguous<Cmpt> {};
// 如果分量类型 Cmpt 是连续的，则 Vector<Cmpt> 也是连续的

//- Data are contiguous label if component type is label
template<class Cmpt>
struct is_contiguous_label<Vector<Cmpt>> : is_contiguous_label<Cmpt> {};
// 如果分量类型 Cmpt 是 label 类型且连续，则 Vector<Cmpt> 也是

//- Data are contiguous scalar if component type is scalar
template<class Cmpt>
struct is_contiguous_scalar<Vector<Cmpt>> : is_contiguous_scalar<Cmpt> {};
// 如果分量类型 Cmpt 是 scalar 类型且连续，则 Vector<Cmpt> 也是


template<class Cmpt>
class typeOfRank<Cmpt, 1>
{
public:

    typedef Vector<Cmpt> type;
};


template<class Cmpt>
class symmTypeOfRank<Cmpt, 1>
{
public:

    typedef Vector<Cmpt> type;
};


template<class Cmpt>
class typeOfSolve<Vector<Cmpt>>
{
public:

    typedef Vector<solveScalar> type;
};

// 暂不深究

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

} // End namespace Foam

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

#include "VectorI.H"

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

#endif
```

### 2.2. 内联

很多方法的定义在内联文件中

API 页面 https://api.openfoam.com/2506/VectorI_8H_source.html

查阅 `Vector/VectorI.H` 代码

```cpp {fileName="Vector/VectorI.H",linenos=table,linenostart=1}
// * * * * * * * * * * * * * * * * Constructors  * * * * * * * * * * * * * * //
 
 template<class Cmpt>
 inline Foam::Vector<Cmpt>::Vector(const Foam::zero)
 :
     Vector::vsType(Zero)
 {}
 // 置零构造
 // - 传入 Zero 常量
 
 
 template<class Cmpt>
 template<class Cmpt2>
 inline Foam::Vector<Cmpt>::Vector
 (
     const VectorSpace<Vector<Cmpt2>, Cmpt2, 3>& vs
 )
 :
     Vector::vsType(vs)
 {}
 // 模板构造函数
 // - 从不同分量类型的向量构造
 
 
 template<class Cmpt>
 inline Foam::Vector<Cmpt>::Vector
 (
     const Cmpt& vx,
     const Cmpt& vy,
     const Cmpt& vz
 )
 {
     this->v_[X] = vx;
     this->v_[Y] = vy;
     this->v_[Z] = vz;
 }
 // 从三个分量构造
 // - 直接访问继承自 VectorSpace 的 v_ 数组成员 
 
 template<class Cmpt>
 inline Foam::Vector<Cmpt>::Vector(Istream& is)
 :
     Vector::vsType(is)
 {}
 // 从输入流构造向量
 
 
 // * * * * * * * * * * * * * * * Member Functions  * * * * * * * * * * * * * //
 
 template<class Cmpt>
 inline const Foam::Vector<Cmpt>& Foam::Vector<Cmpt>::centre
 (
     const Foam::UList<Vector<Cmpt>>&
 ) const noexcept
 {
     return *this;
 }
 // 直接返回当前对象的引用（*this）
 
 
 template<class Cmpt>
 inline Foam::scalar Foam::Vector<Cmpt>::magSqr() const
 {
     return
     (
         Foam::magSqr(this->x())
       + Foam::magSqr(this->y())
       + Foam::magSqr(this->z())
     );
 }
 // 计算向量模的平方
 // - 使用 OpenFOAM 的 magSqr() 函数计算各分量的平方
 
 
 template<class Cmpt>
 inline Foam::scalar Foam::Vector<Cmpt>::mag() const
 {
     return ::sqrt(this->magSqr());
 }
 // 计算向量模
 // - 先计算模的平方，然后调用标准库的 sqrt 函数开方
 
 
 template<class Cmpt>
 inline Foam::scalar Foam::Vector<Cmpt>::distSqr(const Vector<Cmpt>& v2) const
 {
     return
     (
         Foam::magSqr(v2.x() - this->x())
       + Foam::magSqr(v2.y() - this->y())
       + Foam::magSqr(v2.z() - this->z())
     );
 }
 // 计算与另一向量距离的平方
 // - 计算各分量差值的平方和
 
 
 template<class Cmpt>
 inline Foam::scalar Foam::Vector<Cmpt>::dist(const Vector<Cmpt>& v2) const
 {
     return ::sqrt(this->distSqr(v2));
 }
 // 计算与另一向量距离
 // - 先计算距离平方，然后开方
 
 
 template<class Cmpt>
 inline Foam::Vector<Cmpt>& Foam::Vector<Cmpt>::normalise(const scalar tol)
 {
     #ifdef __clang__
     volatile  // Use volatile to avoid aggressive branch optimization
     #endif
     const scalar s = this->mag();
 
     if (s < tol)
     {
         *this = Zero;
     }
     else
     {
         *this /= s;
     }
 
     return *this;
 }
 // 向量归一化
 // - 使用 #ifdef __clang__ 和 volatile 关键字防止 Clang 编译器进行过于激进的优化
 // - 如果模长小于容差值 tol，则将向量设为零向量
 // - 否则，将向量除以其模长，使其成为单位向量
 // - 返回当前对象的引用
 
 
 template<class Cmpt>
 inline Foam::Vector<Cmpt>&
 Foam::Vector<Cmpt>::removeCollinear(const Vector<Cmpt>& unitVec)
 {
     *this -= (*this & unitVec) * unitVec;
     return *this;
 }
 // 移除与给定单位向量共线分量
 // - 计算当前向量在给定单位向量方向上的投影
 // - 从当前向量中减去这个投影分量
 // - 返回当前对象的引用
 
 
 // * * * * * * * * * * * * * * * Member Operations * * * * * * * * * * * * * //
 
 template<class Cmpt>
 inline Cmpt Foam::Vector<Cmpt>::inner(const Vector<Cmpt>& v2) const
 {
     const Vector<Cmpt>& v1 = *this;
 
     return (v1.x()*v2.x() + v1.y()*v2.y() + v1.z()*v2.z());
 }
 // 计算点积（内积）
 // - 计算两个向量各分量乘积的和
 
 
 template<class Cmpt>
 inline Foam::Vector<Cmpt>
 Foam::Vector<Cmpt>::cross(const Vector<Cmpt>& v2) const
 {
     const Vector<Cmpt>& v1 = *this;
 
     return Vector<Cmpt>
     (
         (v1.y()*v2.z() - v1.z()*v2.y()),
         (v1.z()*v2.x() - v1.x()*v2.z()),
         (v1.x()*v2.y() - v1.y()*v2.x())
     );
 }
 // 计算叉积（外积）
 // - 按照叉积公式计算新向量的各分量
 // - 返回一个新的 Vector 对象
 
 
 // * * * * * * * * * * * * * Comparison Operations * * * * * * * * * * * * * //
 
 template<class Cmpt>
 inline bool
 Foam::Vector<Cmpt>::less_xyz(const Vector<Cmpt>& a, const Vector<Cmpt>& b)
 {
     return
     (
         (a.x() < b.x())  // Component is less
      ||
         (
             !(b.x() < a.x())  // Equal? Check next component
          &&
             (
                 (a.y() < b.y())  // Component is less
              ||
                 (
                     !(b.y() < a.y())  // Equal? Check next component
                  && (a.z() < b.z())
                 )
             )
         )
     );
 }
 // 按 X→Y→Z 字典序比较两个向量
 // - 先比较 X 分量，如果不等则返回结果
 // - 如果 X 分量相等，则比较 Y 分量
 // - 如果 Y 分量也相等，则比较 Z 分量
 
 
 template<class Cmpt>
 inline bool
 Foam::Vector<Cmpt>::less_yzx(const Vector<Cmpt>& a, const Vector<Cmpt>& b)
 {
     return
     (
         (a.y() < b.y())  // Component is less
      ||
         (
             !(b.y() < a.y())  // Equal? Check next component
          &&
             (
                 (a.z() < b.z())  // Component is less
              ||
                 (
                     !(b.z() < a.z())  // Equal? Check next component
                  && (a.x() < b.x())
                 )
             )
         )
     );
 }
 // 按 Y→Z→X 字典序比较两个向量
 
 
 template<class Cmpt>
 inline bool
 Foam::Vector<Cmpt>::less_zxy(const Vector<Cmpt>& a, const Vector<Cmpt>& b)
 {
     return
     (
         (a.z() < b.z())  // Component is less
      ||
         (
             !(b.z() < a.z())  // Equal? Check next component
          &&
             (
                 (a.x() < b.x())  // Component is less
              ||
                 (
                     !(b.x() < a.x())  // Equal? Check next component
                  && (a.y() < b.y())
                 )
             )
         )
     );
 }
 // 按 Z→X→Y 字典序比较两个向量
 
 
 // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //
 
 namespace Foam
 {
 
 // * * * * * * * * * * * * * * * Global Functions  * * * * * * * * * * * * * //
 
 //- Linear interpolation of vectors a and b by factor t
 template<class Cmpt>
 inline Vector<Cmpt> lerp
 (
     const Vector<Cmpt>& a,
     const Vector<Cmpt>& b,
     const scalar t
 )
 {
     const scalar onet = (1-t);
 
     return Vector<Cmpt>
     (
         onet*a.x() + t*b.x(),
         onet*a.y() + t*b.y(),
         onet*a.z() + t*b.z()
     );
 }
 // 线性插值函数
 // - 计算两个向量的线性插值结果
 // - t 是插值参数，取值范围 [0, 1]
 // - 当 t=0 时返回 a，t=1 时返回 b，中间值则返回两者的线性组合
 
 
 // * * * * * * * * * * * * * * * Global Operators  * * * * * * * * * * * * * //
 
 //- Dummy innerProduct for scalar
 //  Allows the construction of vtables for virtual member functions
 //  involving the inner-products of fields
 //  for which a "NotImplemented" specialization for scalar is provided.
 template<class Cmpt>
 class innerProduct<Vector<Cmpt>, scalar>
 {
 public:
 
     typedef scalar type;
 };
 // 为 Vector<Cmpt> 和 scalar 的特化内积类型特征
 // - 定义内积结果为 scalar 类型
 
 
 template<class Cmpt>
 inline Cmpt operator&(const Vector<Cmpt>& v1, const Vector<Cmpt>& v2)
 {
     return v1.inner(v2);
 }
 // 点积运算符的重载
 // - 调用 inner 成员函数计算点积
 
 
 template<class Cmpt>
 inline Vector<Cmpt> operator^(const Vector<Cmpt>& v1, const Vector<Cmpt>& v2)
 {
     return v1.cross(v2);
 }
 // 叉积运算符的重载
 // - 调用 cross 成员函数计算叉积
 
 
 template<class Cmpt>
 inline Vector<Cmpt> operator*(const Cmpt& s, const Vector<Cmpt>& v)
 {
     return Vector<Cmpt>(s*v.x(), s*v.y(), s*v.z());
 }
 // 标量与向量乘法运算符的重载（标量在前）
 // - 将向量的每个分量乘以标量
 // - 返回新的向量
 
 template<class Cmpt>
 inline Vector<Cmpt> operator*(const Vector<Cmpt>& v, const Cmpt& s)
 {
     return s*v;
 }
 // 向量与标量乘法运算符的重载（标量在后）
 // - 通过调用前面的实现，确保乘法可交换
 
 
 
 // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //
 
 } // End namespace Foam
```

> [!warning]
> 上面这些代码讨论主要是为了帮助读者熟悉 C++ 语言在 OpenFOAM 中的使用，克服对 C++ 语言的陌生和恐惧，便于读者理解后续实践的代码。暂时不需要花费更多时间去阅读更多 OpenFOAM 的源代码，也不需要深究代码细节，后续会在 `ofsc` 系列讨论代码。

## 3. 项目构建

终端输入命令，建立本文项目

```terminal {fileName="terminal"}
ofsp
mkdir ofsp_05_vector
code ofsp_05_vector
```

继续使用终端命令或者使用 vscode 界面创建其他文件，最终文件结构如下

```terminal {fileName="terminal"}
tree
.
├── Aerosand
│   ├── Aerosand.C
│   ├── Aerosand.H
│   └── Make
│       ├── files
│       └── options
├── Make
│   ├── files
│   └── options
└── ofsp_05_vector.C
```

## 4. 开发库

### 4.1. 库源码

代码 `Aerosand.H` 为

```cpp {fileName="/Aerosand/Aerosand.H",linenos=table,linenostart=1}
#pragma once

class Aerosand
{
public:
    void SetLocalTime(double t);
    double GetLocalTime() const;


private:
    double localTime_;
};

```

代码 `Aerosand.C` 为

```cpp {fileName="/Aerosand/Aerosand.C",linenos=table,linenostart=1}
#include "Aerosand.H"

void Aerosand::SetLocalTime(double t) {
    localTime_ = t;
}

double Aerosand::GetLocalTime() const {
    return localTime_;
}

```

### 4.2. 库 Make

库 `Make/files` 为

```wmake {fileName="/Aerosand/Make/files",linenos=table,linenostart=1}
Aerosand.C

LIB = $(FOAM_USER_LIBBIN)/libAerosand

```

开发库没有其他依赖，库 `Make/options` 为空

### 4.3. 库编译

终端输入命令，进行库的编译

```terminal {fileName="terminal"}
wclean Aerosand
wmake Aerosand
```

## 5. 主项目

### 5.1. 主源码

代码 `ofsp_05_vector.C` 为

```cpp {fileName="/ofsp_05_vector.C",linenos=table,linenostart=1}
#include <iostream>

#include "Aerosand.H"

#include "vector.H"

using namespace Foam;

int main()
{
    scalar s(3.14);
    vector v1(1, 2, 3);
    vector v2(2, 3, 4);
    Info<< s << " * " << v1 << " = "  << s * v1 << nl 
        << "distance:  " << v1.dist(v2) << nl
        << "normalise: " << v1.normalise(1e-6) << nl
        << "inner product: " << v1.inner(v2) << nl
        << "inner product: " << (v1 & v2) << nl // 使用括号提高计算优先级
        << "cross product: " << v1.cross(v2) << nl
        << "cross product: " << (v1^v2) << nl // 使用括号提高计算优先级
        << endl;

	label a = 1;
    scalar pi = 3.1415926;
    Info<< "Hi, OpenFOAM!" << " Here we are." << nl
	    << a << " + " << pi << " = " << a + pi << nl
	    << a << " * " << pi << " = " << a * pi << nl
	    << endl;


    Aerosand mySolver;
    mySolver.SetLocalTime(0.2);
    Info<< "\nCurrent time step is : " << mySolver.GetLocalTime() << endl;

    return 0;
}

```

### 5.2. 项目 Make

项目 `Make/files` 为

```wmake {fileName="/Make/files",linenos=table,linenostart=1}
ofsp_05_vector.C

EXE = $(FOAM_USER_APPBIN)/ofsp_05_vector

```

因为我们包含了 `vector.H` ，路径为 `$FOAM_SRC/OpenFOAM/primitives/Vector/floats/vector.H`。由 Make 文件的位置能判断，该文件属于 OpenFOAM 库，路径为 `$FOAM_SRC/OpenFOAM`。归属关系如下。

```terminal {fileName="terminal"}
$FOAM_SRC/OpenFOAM
├── lnInclude/
├── ...
├── Make
│   ├── files
│   └── options
└── primitives
	├── ...
	└── Vector
		├── ...
		├── floats
		│   ├── ...
		│   └── vector.H
		├── Vector.H
		└── VectorI.H
```

原则上，我们应该在项目 `Make/options` 中包含 OpenFOAM 库。实际上，OpenFOAM 的构建系统（wmake）已经自动处理原生必备库的依赖关系。我们只需要添加第三方库、自己开发的库、某些可选模块的库即可。

> [!important]
> `$FOAM_SRC/OpenFOAM` 库已经自动依赖，其中类的使用均无需用户再次链接。

项目 `Make/options` 为

```wmake {fileName="/Make/options",linenos=table,linenostart=1}
EXE_INC = \
    -IAerosand/lnInclude

EXE_LIBS = \
    -L$(FOAM_USER_LIBBIN) \
    -lAerosand

```

## 6. 编译运行

终端输入命令，编译运行该项目

```terminal {fileName="terminal"}
wclean
wmake
ofsp_05_vector
```

运行结果如下

```terminal {fileName="terminal"}
3.14 * (1 2 3) = (3.14 6.28 9.42)
distance:  1.73205
normalise: (0.267261 0.534522 0.801784)
inner product: 5.34522
inner product: 5.34522
cross product: (-0.267261 0.534522 -0.267261)
cross product: (-0.267261 0.534522 -0.267261)

Hi, OpenFOAM! Here we are.
1 + 3.14159 = 4.14159
1 * 3.14159 = 3.14159


Current time step is : 0.2
```

## 7. 小结

本文完成讨论

- [x] 练习源码的查阅
- [x] 讨论 vector 类的部分代码实现
- [x] 理解原生库的使用
- [x] 编译运行 vector 项目

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
