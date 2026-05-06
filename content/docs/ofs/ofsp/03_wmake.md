---
uid: 20250826161556
title: 03_wmake
date: 2025-08-26
update: 2026-04-08
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
weight: 4
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

Having understood the Make-based implementation of C++ projects, we can now more easily grasp the `wmake` implementation approach provided by OpenFOAM.

This section primarily covers:

- [ ] A brief understanding of `wmake`
- [ ] Direct linking using `wmake`
- [ ] Dynamic library linking using `wmake`
- [ ] Understanding Make
- [ ] Compiling and running a `wmake`-based project


## 1. Understanding wmake

`wmake` in OpenFOAM is a build tool based on `make`. Essentially, it is a set of scripts and configuration files specifically designed for OpenFOAM projects. It simplifies the compilation process of OpenFOAM and automatically handles certain dependencies and path settings.

 1. `wmake` is a wrapper script around `make` that sets up the environment variables and rules required for OpenFOAM compilation.
2. `wmake` automatically locates OpenFOAM’s specific directory structure and sets corresponding compilation options (such as include paths, library paths, etc.).
3. `wmake` provides an interface similar to `make` but is optimized for OpenFOAM.

OpenFOAM uses `Make/files` and `Make/options` to manage compilation with `wmake`.

> [!tip]
> Simply put, the two files `files` and `options` under the `Make` folder together serve the function of a Makefile.

OpenFOAM convention uses `.C` as the suffix for source files and `.H` for header files.

To avoid confusion regarding file structure, a loose distinction is made here. OpenFOAM’s **project-level** `.H` files are often used merely to split the main source code by functionality, facilitating code reading and maintenance; many of these are not class header files. This differs from class header files at the C++ development level and from **source-level** header files in OpenFOAM (e.g., `$FOAM_SRC/OpenFOAM/dimensionSet/dimensionSet.H`).

> [!tip]
> In OpenFOAM, **applications** include:
> - Solvers
> - Utilities
> - Pre-processing Applications
> - Post-processing Applications
>
> In this series, a **project** refers to all components that solve a given problem, including:
> - Solver
> - Test case
> - Linked libraries
> - Utilities
> - etc.

The following discussion focuses on simple **source-level** header files.

## 2. Project

Run the following commands in the terminal to create the project for this section:

```terminal {fileName="terminal"}
ofsp
mkdir ofsp_03_wmake
code ofsp_03_wmake
```

Continue using terminal commands or the VS Code interface to create additional files. The final file structure is as follows:

```terminal {fileName="terminal"}
tree
.
├── Aerosand
│   ├── Aerosand.C
│   └── Aerosand.H
├── Make
│   ├── files
│   └── options
└── ofsp_03_wmake.C
```

The declaration of the `Aerosand` library in `Aerosand.H` is as follows:

```cpp {fileName="/Aerosand/Aerosand.H",linenos=table}
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

> [!warning]
> Ensure that each code file ends with a blank line; otherwise, OpenFOAM will issue a `parse error` during compilation.

The definition of the `Aerosand` library in `Aerosand.C` is as follows:

```cpp {fileName="/Aerosand/Aerosand.C",linenos=table}
#include "Aerosand.H"

void Aerosand::SetLocalTime(double t) {
    localTime_ = t;
}

double Aerosand::GetLocalTime() const {
    return localTime_;
}

```

The main application source file `ofsp_03_wmake.C` is as follows:

```cpp {fileName="/ofsp_03_wmake.C",linenos=table}
#include <iostream>

#include "Aerosand.H"

using namespace std;

int main()
{
    int a = 1;
    double pi = 3.1415926;

    cout << "Hi, OpenFOAM!" << " Here we are." << endl;
    cout << a << " + " << pi << " = " << a + pi << endl;
    cout << a << " * " << pi << " = " << a * pi << endl;


    Aerosand mySolver;
    mySolver.SetLocalTime(0.2);
    cout << "\nCurrent time step is : " << mySolver.GetLocalTime() << endl;

    return 0;
}

```


OpenFOAM provides Make to assist development, where:

- The `/Make/files` file specifies the source files to compile and the name and location of the target to generate.
- The `/Make/options` file specifies compilation and linking options, including header file paths and libraries to link.

Note that `#include "Aerosand.H"` does not require a path specification because the path will be handled in the Make.

The following sections discuss different linking approaches and the details of the Make.

## 3. Direct Linking

Similar to the direct linking discussed in `02_helloWorld/3.4. Linking`, this can also be achieved in OpenFOAM.

### 3.1. Configuring Make

The contents of `ofsp_03_wmake/Make/files` are as follows:

```wmake {fileName="/Make/files" linenos=table}
Aerosand/Aerosand.C
ofsp_03_wmake.C

EXE = $(FOAM_USER_APPBIN)/ofsp_00_helloWorld_wmake
```

- The source files are `Aerosand.C` and `ofsp_03_wmake.C`.
- The location of the generated target is `$(FOAM_USER_APPBIN)`.
- The name of the generated target is `ofsp_03_wmake`.

The contents of `ofsp_03_wmake/Make/options` are as follows:

```wmake {fileName="/Make/options" linenos=table}
EXE_INC = \
	-IAerosand

EXE_LIBS = \
```

- The prefix `EXE` indicates that this configuration is for an executable (the prefix `LIB` indicates configuration for a library).
- The suffix `INC` denotes include paths.
    - The `-I` flag specifies header file search paths.
    - Typically points to the `lnInclude` directory of an OpenFOAM module (linked include directory).
    - Can include system header paths or third-party library header paths.
- The suffix `LIBS` denotes library linking (for direct linking, no libraries need to be linked; this can be left blank).
    - The `-l` flag specifies libraries to link.
    - Library names omit the `lib` prefix and `.so` suffix (e.g., `-lfiniteVolume` corresponds to `libfiniteVolume.so`).
    - The `-L` flag specifies library search paths.

Run the following command in the terminal to compile:

```terminal {fileName="terminal"}
wclean
wmake
```

### 3.2. Compilation

The terminal output consists of three sections, corresponding to the three stages of application compilation.

The first section is the compilation of the custom `Aerosand` class, producing the object file `Aerosand.o` (see the end of the output):

```terminal {fileName="terminal"}
g++ -std=c++14 -m64 -pthread -DOPENFOAM=2406 -DWM_DP -DWM_LABEL_SIZE=32 -Wall 
-Wextra -Wold-style-cast -Wnon-virtual-dtor -Wno-unused-parameter -Wno-invalid
-offsetof -Wno-attributes -Wno-unknown-pragmas -O3  -DNoRepository -ftemplate
-depth-100  -IAerosand -iquote. -IlnInclude 
-I/usr/lib/openfoam/openfoam2406/src/OpenFOAM/lnInclude 
-I/usr/lib/openfoam/openfoam2406/src/OSspecific/POSIX/lnInclude   -fPIC 
-c Aerosand/Aerosand.C -o Make/linux64GccDPInt32Opt/Aerosand/Aerosand.o
```

The second section is the compilation of the main source file, producing the object file `ofsp_03_wmake.o` (see the end of the output):

```terminal {fileName="terminal"}
g++ -std=c++14 -m64 -pthread -DOPENFOAM=2406 -DWM_DP -DWM_LABEL_SIZE=32 -Wall 
-Wextra -Wold-style-cast -Wnon-virtual-dtor -Wno-unused-parameter -Wno-invalid
-offsetof -Wno-attributes -Wno-unknown-pragmas -O3  -DNoRepository -ftemplate
-depth-100  -IAerosand -iquote. -IlnInclude 
-I/usr/lib/openfoam/openfoam2406/src/OpenFOAM/lnInclude 
-I/usr/lib/openfoam/openfoam2406/src/OSspecific/POSIX/lnInclude   -fPIC 
-c ofsp_03_wmake.C -o Make/linux64GccDPInt32Opt/ofsp_03_wmake.o
```

The third section is the linking stage, where the two object files are directly linked to produce the final executable (see the end of the output):

```terminal {fileName="terminal"}
g++ -std=c++14 -m64 -pthread -DOPENFOAM=2406 -DWM_DP -DWM_LABEL_SIZE=32 -Wall 
-Wextra -Wold-style-cast -Wnon-virtual-dtor -Wno-unused-parameter -Wno-invalid
-offsetof -Wno-attributes -Wno-unknown-pragmas -O3  -DNoRepository -ftemplate
-depth-100  -IAerosand -iquote. -IlnInclude 
-I/usr/lib/openfoam/openfoam2406/src/OpenFOAM/lnInclude 
-I/usr/lib/openfoam/openfoam2406/src/OSspecific/POSIX/lnInclude   
-fPIC -Xlinker --add-needed -Xlinker --no-as-needed
Make/linux64GccDPInt32Opt/Aerosand/Aerosand.o 
Make/linux64GccDPInt32Opt/ofsp_03_wmake.o 
-L/usr/lib/openfoam/openfoam2406/platforms/linux64GccDPInt32Opt/lib \
     -lOpenFOAM -ldl  \
     -lm -o /home/aerosand/OpenFOAM/aerosand
 -v2406/platforms/linux64GccDPInt32Opt/bin/ofsp_03_wmake
```

The intermediate build files are located in `ofsp_03_wmake/Make/linux64GccDPInt32Opt/` (the exact path may vary depending on the platform). The generated executable is placed in the `$FOAM_USER_APPBIN` directory (as specified in `Make/files`).

Run the following command in the terminal to locate all successfully generated executables:

```terminal {fileName="terminal"}
tree $FOAM_USER_APPBIN
```

### 3.3. Running

This executable is a standalone program that does not require any external files for input parameters. Thanks to OpenFOAM’s environment variables, the compiled program can be run directly from the terminal in any directory.

Run the following command in the terminal:

```terminal {fileName="terminal"}
ofsp_03_wmake
```

The output is as follows:

```terminal {fileName="terminal"}
Hi, OpenFOAM! Here we are.
1 + 3.14159 = 4.14159
1 * 3.14159 = 3.14159

Current time step is : 0.2
```

## 4. Dynamic Library Linking

In practical development, we still use dynamic libraries to ensure memory efficiency and performance.

Adjust the file structure as follows:

```terminal {fileName="terminal"}
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
└── ofsp_03_wmake.C
```

### 4.1. Library Make

Create a Make for the library.

The contents of `/Aerosand/Make/files` are as follows:

```wmake {fileName="/Aerosand/Make/files"}
Aerosand.C

LIB = $(FOAM_USER_LIBBIN)/libAerosand
```

- Note that `LIB` is used instead of `EXE` for a library.
- The target path ends with `LIBBIN` instead of `APPBIN`.
- The target file must be prefixed with `lib`.

Since this `Aerosand` library does not require linking with any other libraries, `/Aerosand/Make/options` can be left empty.

Run the following command in the terminal to compile the library from the root directory:

```terminal {fileName="terminal"}
wmake Aerosand
```

The terminal output consists of two sections, corresponding to the compilation stages.

The first section compiles the custom `Aerosand` library, producing the object file `Aerosand.o` (see the end of the output):

```terminal {fileName="terminal"}
g++ -std=c++14 -m64 -pthread -DOPENFOAM=2406 -DWM_DP -DWM_LABEL_SIZE=32 -Wall 
-Wextra -Wold-style-cast -Wnon-virtual-dtor -Wno-unused-parameter -Wno-invalid
-offsetof -Wno-attributes -Wno-unknown-pragmas -O3  -DNoRepository -ftemplate
-depth-100   -iquote. -IlnInclude 
-I/usr/lib/openfoam/openfoam2406/src/OpenFOAM/lnInclude 
-I/usr/lib/openfoam/openfoam2406/src/OSspecific/POSIX/lnInclude   -fPIC 
-c Aerosand.C -o Make/linux64GccDPInt32Opt/Aerosand.o
```

The second section links the object file `Aerosand.o` into a dynamic library `Aerosand.so` (see the end of the output):

```terminal {fileName="terminal"}
g++ -std=c++14 -m64 -pthread -DOPENFOAM=2406 -DWM_DP -DWM_LABEL_SIZE=32 -Wall 
-Wextra -Wold-style-cast -Wnon-virtual-dtor -Wno-unused-parameter -Wno-invalid
-offsetof -Wno-attributes -Wno-unknown-pragmas -O3  -DNoRepository -ftemplate
-depth-100   -iquote. -IlnInclude 
-I/usr/lib/openfoam/openfoam2406/src/OpenFOAM/lnInclude 
-I/usr/lib/openfoam/openfoam2406/src/OSspecific/POSIX/lnInclude   -fPIC -shared 
-Xlinker --add-needed -Xlinker 
--no-as-needed  Make/linux64GccDPInt32Opt/Aerosand.o 
-L/usr/lib/openfoam/openfoam2406/platforms/linux64GccDPInt32Opt/lib \
      -o /home/aerosand/OpenFOAM/aerosand
-v2406/platforms/linux64GccDPInt32Opt/lib/libAerosand.so
```

The intermediate build files are located in `ofsp_03_wmake/Aerosand/Make/linux64GccDPInt32Opt/` (the exact path may vary depending on the platform). The generated dynamic library is placed in the `$FOAM_USER_LIBBIN` directory (as specified in `Make/files`).

Run the following command in the terminal to locate all successfully generated dynamic libraries:

```terminal {fileName="terminal"}
tree $FOAM_USER_LIBBIN
```

A library may contain multiple classes or even other sub-libraries. After compilation, an `lnInclude` folder is generated in the library’s path. This folder contains symbolic links to the declarations (`.H` files) and implementations (`.C` files) of all classes (or sub-libraries) within the library, providing a unified and simplified path for subsequent linking. For reference, examine the `$FOAM_SRC/OpenFOAM` library in OpenFOAM, where the `lnInclude` folder contains shortcuts to all classes (or sub-libraries) within that library.

Run the following command to view the file structure of the compiled `Aerosand` library:

```terminal {fileName="terminal"}
Aerosand
├── Aerosand.C
├── Aerosand.H
├── lnInclude
│   ├── Aerosand.C -> ../Aerosand.C
│   └── Aerosand.H -> ../Aerosand.H
└── Make
    ├── files
    ├── linux64GccDPInt32Opt
    │   ├── Aerosand.C.dep
    │   ├── Aerosand.o
    │   ├── options
    │   ├── sourceFiles
    │   └── variables
    └── options
```

### 4.2. Project Make

Because the `Aerosand` library has been compiled into a dynamic library, we need to specify it in the project’s Make.

Modify `/Make/files` as follows:

```wmake {fileName="/Make/files"}
ofsp_03_wmake.C

EXE = $(FOAM_USER_APPBIN)/ofsp_03_wmake
```

Modify `/Make/options` as follows:

```wmake {fileName="/Make/options"}
EXE_INC = \
    -IAerosand/lnInclude

EXE_LIBS = \
    -L$(FOAM_USER_LIBBIN) \
    -lAerosand

```

- The prefix `EXE` indicates that this configuration is for an executable (the prefix `LIB` indicates configuration for a library).
- The suffix `INC` denotes include paths.
    - The `-I` flag specifies header file search paths.
    - Typically points to the `lnInclude` directory of an OpenFOAM module (linked include directory).
    - Can include system header paths or third-party library header paths.
- The suffix `LIBS` denotes library linking.
    - The `-l` flag specifies libraries to link.
    - Library names omit the `lib` prefix and `.so` suffix (e.g., `-lAerosand` corresponds to `libAerosand.so`).
    - The `-L` flag specifies library search paths.
- Ensure the file ends with a blank line to avoid OpenFOAM compilation warnings.


> [!tip]
> In common OpenFOAM solver `Make/options` files, you will often see only `-l` flags without `-L` flags. This is because the solvers use built-in libraries whose paths are already configured, so only the library names need to be specified with `-l`. For user-defined libraries compiled in other locations, `-L` path specification is necessary.
> 
> Pay close attention to paths, as they are crucial for successful compilation. Typically, paths are specified relative to the project root directory (absolute paths can also be used, as demonstrated below).

### 4.3. Compilation

Run the following command in the terminal to perform the compilation:

```terminal {fileName="terminal"}
wclean
wmake
```

The terminal output consists of two sections, corresponding to the compilation stages.

The first section compiles the main source file, producing the object file `ofsp_03_wmake.o` (see the end of the output):

```terminal {fileName="terminal"}
g++ -std=c++14 -m64 -pthread -DOPENFOAM=2406 -DWM_DP -DWM_LABEL_SIZE=32 -Wall 
-Wextra -Wold-style-cast -Wnon-virtual-dtor -Wno-unused-parameter -Wno-invalid
-offsetof -Wno-attributes -Wno-unknown-pragmas -O3  -DNoRepository -ftemplate
-depth-100  -IAerosand/lnInclude -iquote. -IlnInclude 
-I/usr/lib/openfoam/openfoam2406/src/OpenFOAM/lnInclude 
-I/usr/lib/openfoam/openfoam2406/src/OSspecific/POSIX/lnInclude   -fPIC 
-c ofsp_03_wmake.C -o Make/linux64GccDPInt32Opt/ofsp_03_wmake.o
```

The second section links the dynamic library to the application and generates the executable `ofsp_03_wmake` (see the end of the output):

```terminal {fileName="terminal"}
g++ -std=c++14 -m64 -pthread -DOPENFOAM=2406 -DWM_DP -DWM_LABEL_SIZE=32 -Wall 
-Wextra -Wold-style-cast -Wnon-virtual-dtor -Wno-unused-parameter -Wno-invalid
-offsetof -Wno-attributes -Wno-unknown-pragmas -O3  -DNoRepository -ftemplate
-depth-100  -IAerosand/lnInclude -iquote. -IlnInclude 
-I/usr/lib/openfoam/openfoam2406/src/OpenFOAM/lnInclude 
-I/usr/lib/openfoam/openfoam2406/src/OSspecific/POSIX/lnInclude   -fPIC -Xlinker
--add-needed -Xlinker --no-as-needed  Make/linux64GccDPInt32Opt/ofsp_03_wmake.o 
-L/usr/lib/openfoam/openfoam2406/platforms/linux64GccDPInt32Opt/lib \
    -L/home/aerosand/OpenFOAM/aerosand-v2406/platforms/linux64GccDPInt32Opt/lib 
    -lAerosand -lOpenFOAM -ldl  \
     -lm -o /home/aerosand/OpenFOAM/aerosand
-v2406/platforms/linux64GccDPInt32Opt/bin/ofsp_03_wmake
```

Similarly, the generated executable is placed in the `$FOAM_USER_APPBIN` directory (as specified in `Make/files`).

### 4.4. Running

Run the following command in the terminal:

```terminal {fileName="terminal"}
ofsp_03_wmake
```

The output is as follows:

```terminal {fileName="terminal"}
Hi, OpenFOAM! Here we are.
1 + 3.14159 = 4.14159
1 * 3.14159 = 3.14159

Current time step is : 0.2
```

## 5. Summary

This section has completed the following discussions:

- [x] A brief understanding of `wmake`
- [x] Direct linking using `wmake`
- [x] Dynamic library linking using `wmake`
- [x] Understanding Make
- [x] Compiling and running a `wmake`-based project



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


