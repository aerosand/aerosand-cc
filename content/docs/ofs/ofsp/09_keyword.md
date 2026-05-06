---
uid: 20250916165944
title: 09_keyword
date: 2025-09-16
update: 2026-04-08
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
> Visit [https://aerosand.cc](https://aerosand.cc/) for the latest updates.



## 0. Preface

Let us pause and carefully consider the project from the previous section. It is difficult to claim that it reads parameters by keyword because the code merely implements mechanical reading. If we change the order of keywords in the input file, it will lead to inconsistent parameter assignment (readers can try changing the order of keywords in the `ofspProperties` file, re-run the code, and observe the results).

Let us consider how to truly implement reading by keyword.

This section primarily discusses:

- [ ] Understanding how OpenFOAM reads parameters
- [ ] Implementing keyword-based parameter reading using C++
- [ ] Compiling and running a keyword project

## 1. OpenFOAM Parameter Reading


>[!tip]
>OpenFOAM's parameter reading approach has undergone significant changes with architecture updates. We will use code from an earlier version as an example to facilitate reader understanding.


API page: [https://github.com/OpenFOAM/OpenFOAM-2.0.x/blob/master/applications/solvers/basic/laplacianFoam/createFields.H](https://github.com/OpenFOAM/OpenFOAM-2.0.x/blob/master/applications/solvers/basic/laplacianFoam/createFields.H)

Let us examine what the statements for reading simple types of parameters look like in OpenFOAM.

In the `laplacianFoam/createFields.H` file, we can see the following statements:

```cpp {fileName="laplacianFoam/createFields.H",linenos=table,linenostart=1}
    Info<< "Reading transportProperties\n" << endl;

    IOdictionary transportProperties
    (
        IOobject
        (
            "transportProperties", // File name (based on the path of the following parameters)
            runTime.constant(), // Path: constant
            mesh, // Associate with the mesh
            IOobject::MUST_READ_IF_MODIFIED, // Must read if modified
            IOobject::NO_WRITE // Do not write
        )
    );
    // Constructs a transportProperties object of the IOdictionary class based on the IOobject
    // The IOobject is constructed with the file name "transportProperties" (and path) and other parameters


    Info<< "Reading diffusivity DT\n" << endl;

    dimensionedScalar DT
    (
        transportProperties.lookup("DT")
    );
    // The transportProperties object has a lookup function that reads the value after the keyword "DT"

```

Although the syntax of `lookup()` is somewhat outdated in OpenFOAM, it is relatively intuitive.

To summarize briefly:

- `IOdictionary` is a class related to file reading    
- `IOdictionary` constructs the object `transportProperties` based on the external file `transportProperties`
- The object `transportProperties` has methods of the `IOdictionary` class, such as `lookup()`
- That is, the object `transportProperties` can use the `lookup()` method

We will attempt to mimic this ourselves and implement OpenFOAM's reading functionality.

## 2. Implementing Keyword-Based Parameter Reading

### 2.1. Project Preparation

Run the following commands in the terminal to create this project:

```terminal {fileName="terminal"}
ofsp
foamNewApp ofsp_09_keyword
code ofsp_09_keyword
```

Create new files for this project. The final project structure is as follows:

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

### 2.2. Self-Developed IOdictionary

We envision that our self-developed `IOdictionary` class can also have a syntax similar to OpenFOAM. A simplified form is as follows:

```cpp
// Desired syntax
IOdictionary ofspProperties // Construct an instance of the class
(
	"ofspProperties" // Name of the external file, root directory path
);

double viscosity = ofspProperties.lookup("nu"); // The class instance can use methods
```

Here, the object `ofspProperties` is an instantiation of the class `IOdictionary` based on the file name (path) `ofspProperties`. The object `ofspProperties` can use the method `lookup()` defined in the class `IOdictionary`.

To implement the functionality for the self-developed class `IOdictionary` to read external files, we will still use the C++ native `fstream` class.

The class declaration in `IOdictionary.H` is as follows:

```cpp {fileName="/IOdictionary/IOdictionary.H",linenos=table,linenostart=1}
#pragma once

#include <iostream>
#include <fstream>
#include <string>

namespace Aerosand // Custom namespace to avoid naming conflicts
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
> The code style is intentionally aligned with that of OpenFOAM.

The class definition in `IOdictionary.C` is as follows:

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
	    // Extract the string after the keyword
        size_t keywordPos = line.find(keyword);
        if (keywordPos != std::string::npos)
        {
            returnValue = line.substr(keywordPos + keyword.length());
        }

		// Remove all spaces
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

We also need to define the library Make.

The content of `IOdictionary/Make/files` is as follows:

```wmake {fileName="/IOdictionary/Make/files"}
IOdictionary.C

LIB = $(FOAM_USER_LIBBIN)/libIOdictionary

```

Since no other libraries are used, the file `IOdictionary/Make/options` can be left empty.

Run the following commands in the terminal to compile and generate the dynamic library:

```terminal {fileName="terminal"}
wclean IOdictionary
wmake IOdictionary
```

### 2.3. Main Project

The main source code in `ofsp_09_keyword.C` is as follows:

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

We also need to define the project Make.

The content of `ofsp_09_keyword/Make/files` is as follows:

```wmake {fileName="/Make/files"}
ofsp_01_IO_keyword.C

EXE = $(FOAM_USER_APPBIN)/ofsp_01_IO_keyword

```

The content of `ofsp_09_keyword/Make/options` is as follows:

```wmake {fileName="/Make/options"}
EXE_INC = \
    -IIOdictionary/lnInclude

EXE_LIBS = \
    -L$(FOAM_USER_LIBBIN) \
    -lIOdictionary

```

### 2.5. Compilation and Execution

Provide a file similar to an OpenFOAM dictionary in the root directory of the project. The content of `ofspProperties` is as follows (located at `/ofsp/ofsp_09_keyword/ofspProperties`):

```cpp {fileName="/ofspProperties"}
nu                  0.01
application         laplacianFoam
deltaT              0.005
writeInterval       20
purgeWrite          0
```

Run the following commands in the terminal to compile and run the project:

```terminal {fileName="terminal"}
wclean
wmake

ofsp_09_keyword
```

The terminal output is as follows:

```terminal {fileName="terminal"}
nu              0.01
application     laplacianFoam
deltaT          0.005
writeInterval   20
purgeWrite      0
```

Even if we adjust the order of parameters in the external dictionary file, for example:

```cpp {fileName="/ofspProperties"}
nu                  0.01
application         laplacianFoam

writeInterval       20
purgeWrite          0

deltaT              0.005
```

Without recompiling the project, we can run it directly:

Run the following command in the terminal:

```terminal {fileName="terminal"}
ofsp_09_keyword
```

The result remains the same.

## 3. Summary

From the above discussion, it can be seen that the approach to reconstructing the project is essentially correct. However, the parameters obtained are uniformly of type `string`, which still does not meet our final requirements; we need to further develop functional features.

This section has completed the following discussions:

- [x] Understanding how OpenFOAM reads parameters
- [x] Implementing keyword-based parameter reading using C++
- [x] Compiling and running a keyword project


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

