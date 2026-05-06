---
uid: 20250827192135
title: 06_tensor
date: 2025-08-27
update: 2026-03-26
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
> Visit [https://aerosand.cc](https://aerosand.cc/) for the latest updates.



## 0. Preface

The previous section discussed some code details based on the `Vector` class. This section discusses the `Tensor` class.

This section primarily discusses:

- [ ] Discussing parts of the implementation of the `Tensor` class
- [ ] Understanding the compilation and linking of multi-class complex libraries
- [ ] Compiling and running a `tensor` project

## 1. Tensor Class

API page: [https://api.openfoam.com/2506/classFoam_1_1Tensor.html](https://api.openfoam.com/2506/classFoam_1_1Tensor.html)

Run the following command in the terminal to search locally:

```terminal {fileName="terminal"}
find $FOAM_SRC -iname tensor
```

Run the following command to open the class folder:

```terminal {fileName="terminal"}
code $FOAM_SRC/OpenFOAM/primitives/Tensor
```

The file structure of this class is as follows:

```terminal {fileName="terminal"}
tree -L 1
.
├── floats
├── ints
├── lists
├── Tensor.H
└── TensorI.H
```

Examine `Tensor/Tensor.H` to see the implementation details of this class. We will not go through it line by line here.

We can also use the API or the terminal to find and read related classes:

- `dimensionedTensor`
- `tensorField`

> [!warning]
> Do not delve into code details at this stage; a general understanding of the usage of member functions is sufficient.


## 2. OFextension Plugin

It is highly recommended to install the community plugin `OFextension` in VS Code.

### 2.1. Configuring the Plugin

1. Click the gear icon in the lower-left corner of VS Code to open `Settings`.    
2. Search for `ofextension` in the search bar.
3. Set the correct OpenFOAM path in `Ofextension: OFpath`.
4. Open the user's development application with VS Code, press `F1`, enter `ofInit` to initialize the configuration.

### 2.2. Using the Plugin

During project development, for example, in the main source code of this section, when entering relevant objects, VS Code will automatically pop up available methods (member functions).

Moreover, in the main source code, you can select header files, classes, etc., right-click, and use `Go to Definition` or `Go to Declaration` to directly jump to the source code.

This plugin is highly recommended for its convenience. Be careful not to initialize it in the OpenFOAM source folder.

> [!warning]
> Sometimes the jumped source code may not point to the correct location; pay attention to distinguish.


## 3. Project Setup

Run the following commands in the terminal to create the project for this section:

```terminal {fileName="terminal"}
ofsp
mkdir ofsp_06_tensor
code ofsp_06_tensor
```

Continue using terminal commands or the VS Code interface to create additional files. The final file structure is as follows:

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

Note that the file structure of the development library is slightly different from the previous section. We have already observed that under an OpenFOAM library, there are generally multiple sub-libraries/classes. A user's development library may also consist of several classes, and the development library has its own Make file to manage multiple classes. For example, here the `Aerosand` library has three classes: `class1`, `class2`, and `class3`.

## 4. Development Library

### 4.1. class1

For the first class, we still use the previous code.

The code in `class1.H` is:

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

The code in `class1.C` is:

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

For the second class, we attempt to create a new class through inheritance.

The code in `class2.H` is:

```cpp {fileName="/Aerosand/class2/class2.H",linenos=table,linenostart=1}
#pragma once

#include "vector.H"

namespace Foam
{

class class2 : public vector
{
public:
    // Inherit constructors
    using Foam::vector::vector;

    // Compute the sum of components
    Foam::scalar sum() const;
};

} // namespace Foam

```

The code in `class2.C` is:

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
> Note that `scalar` and `vector` used in the declaration and definition belong to the `Foam` namespace, so the namespace must be used.

### 4.3. class3

For the third class, we write some simple content.

The code in `class3.H` is:

```cpp {fileName="/Aerosand/class3/class3.H",linenos=table,linenostart=1}
#pragma once

class class3
{
public:
    void class3Info() const;
};

```

The code in `class3.C` is:

```cpp {fileName="/Aerosand/class3/class3.C",linenos=table,linenostart=1}
#include "class3.H"

#include <iostream>

void class3::class3Info() const 
{
    std::cout << "This is class3\n" << std::endl;
}

```

### 4.4. Library Make

The library `Make/files` is:

```wmake {fileName="/Aerosand/Make/files",linenos=table,linenostart=1}
class1/class1.C
class2/class2.C
class3/class3.C

LIB = $(FOAM_USER_LIBBIN)/libAerosand

```

This development library has no other dependencies; the library `Make/options` can be left empty.

### 4.5. Library Compilation

Run the following commands in the terminal to compile the library:

```terminal {fileName="terminal"}
wclean Aerosand
wmake Aerosand
```

## 5. Main Project

### 5.1. Main Source Code

The code in `ofsp_06_tensor.C` is:

```cpp {fileName="/ofsp_06_tesnsor.C",linenos=table,linenostart=1}
#include "tensor.H"
#include "dimensionedTensor.H"
#include "tensorField.H"
// Including OpenFOAM classes

#include "class1.H"
#include "class2.H"
#include "class3.H"
// Including classes from the development library

using namespace Foam;

int main()
{
    scalar s(3.14); // scalar class is indirectly included via tensor.H
    vector v1(1, 2, 3); // vector class is indirectly included via tensor.H
    vector v2(0.5, 1, 1.5);
    
    // Using member functions of the tensor class below
    
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

	// Using member functions of the dimensionedTensor class below
	
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

	// Using member functions of the tensorField class below
	
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

	// class1 construction and output
    class1 mySolver;
    mySolver.SetLocalTime(0.2);
    Info<< "\nCurrent time step is : " << mySolver.GetLocalTime() << endl;
	
	// class2 construction and output
	class2 v3(2,4,6); // class2 inherits the constructor from vector
    Info<< "Sum of vector components: " << v3.sum() << endl;

	// class3 construction and output
    class3 myMessage;
    myMessage.class3Info();

    return 0;
}

```

### 5.2. Project Make

The project `Make/files` is:

```wmake {fileName="/Make/files",linenos=table,linenostart=1}
ofsp_06_tensor.C

EXE = $(FOAM_USER_APPBIN)/ofsp_06_tensor

```


The project `Make/options` is:

```wmake {fileName="/Make/options",linenos=table,linenostart=1}
EXE_INC = \
    -IAerosand/lnInclude

EXE_LIBS = \
    -L$(FOAM_USER_LIBBIN) \
    -lAerosand

```


Similarly, the `$FOAM_SRC/OpenFOAM` library is automatically included as a dependency; any classes used from it do not require the user to link again.

## 6. Compilation and Execution

Run the following commands in the terminal to compile and run the project:

```terminal {fileName="terminal"}
wclean
wmake
ofsp_06_tensor
```

The output is as follows:

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


## 7. Summary

This section has completed the following discussions:

- [x] Discussing parts of the implementation of the `Tensor` class
- [x] Understanding the compilation and linking of multi-class complex libraries
- [x] Compiling and running a `tensor` project



## Support us

>[!tip]
>Hopefully, the sharing here can be helpful to you.
>
>If you find this content helpful, your comments or donations would be greatly appreciated. Your support helps ensure the ongoing updates, corrections, refinements, and improvements to this and future series, ultimately benefiting new readers as well.
>
>The information and message provided during donation will be displayed as an acknowledgment of your support.

{{< cards >}}
  {{< card link="/" title="Support" image="https://www.notion.so/image/attachment%3A3be6af9a-4829-4dfd-997e-641dfd055ba9%3Aalipay.jpg?table=block&id=22cd34b0-7c4c-8086-bdda-d558df1d9a11&t=22cd34b0-7c4c-8086-bdda-d558df1d9a11" subtitle="AliPay" >}}
{{< /cards >}}


> Copyright @ 2026 Aerosand
> 
> - Course (text, images, etc.)：[CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/)
> - Code derived from OpenFOAM：[GPL v3](https://www.gnu.org/licenses/gpl-3.0.html)
> - Other code：[MIT License](https://opensource.org/licenses/MIT)

