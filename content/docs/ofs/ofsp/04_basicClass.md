---
uid: 20250826191442
title: 04_basicClass
date: 2025-08-26
update: 2026-03-27
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
> Visit [https://aerosand.cc](https://aerosand.cc/) for the latest updates.


## 0. Preface

We have already explored native C++ implementations, Make-based implementations, and OpenFOAM’s `wmake` implementation. Before officially delving into OpenFOAM development, it is necessary to gain a basic understanding of common classes in OpenFOAM to facilitate their use.

We can refer to the official API at https://api.openfoam.com/2506/ for detailed information.

> [!tip]
> Changing the version number at the end of the URL allows access to the API for the corresponding version.

This section primarily discusses:

- [ ] Navigating the OpenFOAM API
- [ ] Understanding some common basic classes


## 1. Switch

You can search for the keyword `Switch` directly through the API website to access the Switch Class Reference page, which provides relevant information about the Switch class, including the corresponding Source files: `Switch.H` and `Switch.C`.

API page: [https://api.openfoam.com/2506/Switch_8H.html](https://api.openfoam.com/2506/Switch_8H.html)

You can also locate and read the source code via the terminal.

Run the following command to find the source location:

```terminal {fileName="terminal"}
find $FOAM_SRC -iname switch.H
```

The terminal output for the corresponding version is as follows:

```terminal {fileName="terminal"}
/usr/lib/openfoam/openfoam2406/src/OpenFOAM/primitives/bools/Switch/Switch.H
/usr/lib/openfoam/openfoam2406/src/OpenFOAM/lnInclude/Switch.H
```

Based on the discussion in the previous section, we can copy the correct path and read the source code locally using VS Code:

```terminal {fileName="terminal"}
code /usr/lib/openfoam/openfoam2406/src/OpenFOAM/primitives/bools/Switch/Switch.H
```

> [!tip]
> Use `Ctrl + Shift + C` and `Ctrl + Shift + V` for copy-paste operations in the terminal.

In any case, part of the `Switch.H` source code is as follows (click the code block title to jump to the source code):

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

According to the Description, `Switch` is a wrapper around the `bool` type, serving similar judgment functionality as `bool`.

We will not delve deeper into the detailed code definitions at this point.

## 2. label

Similarly, the source code for the `label` class can be found online or locally.

API page: [https://api.openfoam.com/2506/label_8H.html](https://api.openfoam.com/2506/label_8H.html)

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


Based on the source code description, the `label` type is either `int32_t` or `int64_t`, depending on the definition of the preprocessor macro `WM_LABEL_SIZE`.

Simply put, `label` can be understood as a wrapper around the `int` type.

## 3. scalar

API page: [https://api.openfoam.com/2506/scalar_8H.html](https://api.openfoam.com/2506/scalar_8H.html)

```cpp {filename="scalar.H"}
/*
...
 Description
     A floating-point number identical to float or double depending on
     whether WM_SP, WM_SPDP or WM_DP is defined.
...
*/
```

According to the source code description, the `scalar` type can be understood as a wrapper around floating-point types, equivalent to `float` or `double` depending on the usage context.

## 4. vector

API page: [https://api.openfoam.com/2506/vector_8H.html](https://api.openfoam.com/2506/vector_8H.html)

```cpp {fileName="vector.H"}
/*
...
 Description
     A Vector of values with scalar precision,
     where scalar is float/double depending on the compilation flags.
...
*/
```

Thus, the `vector` type is a vector composed of `scalar` values. This class also defines mathematical operations for vectors.

>[!question]
>What about vectors composed of other data types?

## 5. tensor

API page: [https://api.openfoam.com/2506/tensor_8H.html](https://api.openfoam.com/2506/tensor_8H.html)

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

Thus, the `tensor` type is a tensor composed of `scalar` values. This class also defines mathematical operations for tensors.

>[!question]
>What about tensors composed of other data types?

## 6. dimensionedScalar/Vector/Tensor

Taking `dimensionedScalar` as an example, the API page is: [https://api.openfoam.com/2506/dimensionedScalar_8H_source.html](https://api.openfoam.com/2506/dimensionedScalar_8H_source.html)

```cpp {fileName="dimensionedScalar.H"}
/*
...
 Description
     Dimensioned scalar obtained from generic dimensioned type.
...
*/
```

These are scalars, vectors, and tensors that incorporate OpenFOAM’s unit system and corresponding computational methods.

## 7. scalar/vector/tensorField

Taking `scalarField` as an example, the API page is: [https://api.openfoam.com/2506/scalarField_8H.html](https://api.openfoam.com/2506/scalarField_8H.html)

These are essentially list values (i.e., field data storage) of `scalar`/`vector`/`tensor` types, with corresponding computational methods.

## 8. dimensionSet

API page: [https://api.openfoam.com/2506/dimensionSet_8H.html](https://api.openfoam.com/2506/dimensionSet_8H.html)

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

This class defines the unit system for basic types, enabling rigorous dimensional checking during algebraic computations.

Through the API page, you can see the files directly or indirectly included by this class. The source code provides many practical unit combinations:

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

These predefined unit combinations will frequently appear in future programming practice.

## 9. tmp

API page: [https://api.openfoam.com/2506/tmp_8H.html](https://api.openfoam.com/2506/tmp_8H.html)

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

This class is a smart pointer template class implemented by OpenFOAM, commonly used for automatically managing the lifecycle of temporary objects. Its primary purpose is to optimize performance by avoiding unnecessary copying of large data (e.g., physical fields with massive data volumes).

## 10. IOobject

API page: [https://api.openfoam.com/2506/IOobject_8H.html](https://api.openfoam.com/2506/IOobject_8H.html)

In simple terms, this class provides a complete description and interface for objects that need to be read from or written to disk (as files) and memory (as objects).

In CFD simulations, thousands of variables (fields, boundary conditions, model parameters, etc.) need to be read from dictionary files or written to disk as results. This class addresses input/output issues by encapsulating all these miscellaneous tasks into a unified class.

We will continue to use this class in practice in the following sections.

## 11. Summary

> [!warning]
> For now, there is no need to delve deeply into the programming implementation details of these class definitions, as such details are not the focus of this series. Overly deep exploration can cause significant confusion for beginners and disrupt the learning pace.

This section has completed the following discussions:

- [x] Navigating the OpenFOAM API
- [x] Understanding some common basic classes



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

