---
uid: 20250901123439
title: 07_firstApp
date: 2025-09-01
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
weight: 8
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

Having discussed compilation principles, dynamic library linking, and common OpenFOAM classes, we now use OpenFOAM to complete a relatively comprehensive development workflow.

This section primarily discusses:

- [ ] Understanding the file structure of standard OpenFOAM applications
- [ ] Learning to use scripts
- [ ] Understanding the application development and testing process
- [ ] Calling development libraries located elsewhere
- [ ] Compiling and running the firstApp project

## 1. fvCFD.H

In practical development, in addition to the previously mentioned classes like `vector` and `tensor`, we also need more classes related to the Finite Volume Method (FVM) to discretely solve partial differential equations. OpenFOAM provides `fvCFD.H`, which includes most of the header files related to FVM, including the `tensor` class, among others. Using `fvCFD.H` can significantly reduce the number of header files that need to be included in the main source code.

Run the following command in the terminal to find the `fvCFD.H` file:

```terminal {fileName="terminal"}
find $FOAM_SRC -iname fvcfd.h
```

The terminal output is as follows:

```terminal {fileName="terminal"}
/usr/lib/openfoam/openfoam2406/src/finiteVolume/cfdTools/general/include/fvCFD.H
/usr/lib/openfoam/openfoam2406/src/finiteVolume/lnInclude/fvCFD.H
```

Clearly, `fvCFD.H` is located in the `finiteVolume` library, so it needs to be included and linked in `Make/options`.

> [!tip]
> OpenFOAM low-level libraries are configured as automatic dependencies. The `finiteVolume` library, however, is a high-level library that depends on many low-level libraries and requires manual configuration.

We can replace the native header files from the previous section with `fvCFD.H` and configure the corresponding Make files.

## 2. Project

We will use the method provided by OpenFOAM to create a basic solver application template and develop on top of it.

Run the following commands in the terminal to create the project for this section:

```terminal {fileName="terminal"}
ofsp
foamNewApp ofsp_07_firstApp
code ofsp_07_firstApp
```

### 2.1. Source Code

The source code will be discussed below.

### 2.2. Test Case

For different solvers, a corresponding native test case can be copied for debugging solver development. For this project, the test case is only used to pass the solver syntax check.

Run the following command in the terminal to copy the test case:

```terminal {fileName="terminal"}
cp -r $FOAM_TUTORIALS/incompressible/icoFoam/cavity/cavity debug_case
```

### 2.3. Script Files

Using scripts facilitates user development and testing. Taking this project as an example, we create scripts as follows:

Run the following command in the terminal to create the scripts:

```terminal {fileName="terminal"}
code caserun caseclean
```

The `caserun` script is primarily responsible for running the test case after the application is successfully compiled. It can include the following content:

```bash {fileName="/caserun",linenos=table,linenostart=1}
#!/bin/bash
# The first line tells the system to use the bash interpreter to execute the script

# Generate the mesh for the test case and output logs to the test case path
blockMesh -case debug_case | tee debug_case/log.mesh
# Output the message "Meshing done."
echo "Meshing done."

# Run the solver for the test case and output logs to the test case path
ofsp_07_firstApp -case debug_case | tee debug_case/log.run
```

The `caseclean` script is primarily responsible for cleaning up the application to its pre-compilation state. If the application is modified, the test case should also be restored to its pre-run state. It can include the following content:

```bash {fileName="/caseclean",linenos=table,linenostart=1}
#!/bin/bash

# Delete all log files in the test case directory
rm -rf debug_case/log.*

# Clean the test case
foamCleanTutorials debug_case
# Output the message "Cleaning done."
echo "Cleaning done."
```

### 2.4. Documentation File

For the convenience of future reading, development, and use, we should also prepare a documentation file.

Run the following command in the terminal to create the documentation file:

```terminal {fileName="terminal"}
code README.md
```

The content of the documentation file is as follows:

```markdown {fileName="/README.md"}
## About

This is the first standard OpenFOAM application we have created in the ofsp series.

## Bio

- Aerosand @ Aerosand

## Caution

OpenFOAM v2406 or newer versions are required.

## Deploy

Prepare the environment and all files.

Execute the following commands in the terminal in the root directory:

Clean and recompile the application:

1. wclean
2. wmake
   
Clean and recompute the test case:

1. ./caseclean
2. ./caserun

## Event

@ 20250903
- Added cleaning script #done

@ 20250901
- Created application #done

```

It is recommended to follow the A-B-C-D-E principle (About-Bio-Caution-Deploy-Event) when writing documentation files, clearly expressing the necessary information.

## 3. File Structure

The file structure of this project is as follows:

Run the following command in the terminal:

```terminal {fileName="terminal"}
tree
.
├── caseclean
├── caserun
├── debug_case
│   ├── 0
│   │   ├── p
│   │   └── U
│   ├── constant
│   │   ├── polyMesh
│   │   │   ├── boundary
│   │   │   ├── faces
│   │   │   ├── neighbour
│   │   │   ├── owner
│   │   │   └── points
│   │   └── transportProperties
│   ├── log.mesh
│   ├── log.run
│   └── system
│       ├── blockMeshDict
│       ├── controlDict
│       ├── decomposeParDict
│       ├── fvSchemes
│       ├── fvSolution
│       └── PDRblockMeshDict
├── Make
│   ├── files
│   └── options
└── ofsp_07_firstApp.C
```

## 4. Testing

Run the following commands in the terminal to execute the scripts:

```terminal {fileName="terminal"}
wclean
wmake
./caseclean
./caserun
```

If the execution prompts `Permission denied`, you need to grant permission to the scripts:

```terminal {fileName="terminal"}
chmod +x caserun caseclean
```

Although we have not yet written any development code for the solver, the basic solver template provided by OpenFOAM can still run when a test case exists.

The terminal output consists of two sections:

The first section is the output log for mesh generation:

```terminal {fileName="terminal"}
/*---------------------------------------------------------------------------*\
| =========                 |                                                 |
| \\      /  F ield         | OpenFOAM: The Open Source CFD Toolbox           |
|  \\    /   O peration     | Version:  2406                                  |
|   \\  /    A nd           | Website:  www.openfoam.com                      |
|    \\/     M anipulation  |                                                 |
\*---------------------------------------------------------------------------*/
Build  : _9bfe8264-20241212 OPENFOAM=2406 patch=241212 version=2406
Arch   : "LSB;label=32;scalar=64"
Exec   : blockMesh -case debug_case
Date   : Sep 01 2025
Time   : 18:09:17
Host   : aerosand
PID    : 200553
I/O    : uncollated
Case   : /home/aerosand/github/data_project/openfoam_sharing/ofsp/ofsp_07_firstApp/debug_case
nProcs : 1
trapFpe: Floating point exception trapping enabled (FOAM_SIGFPE).
fileModificationChecking : Monitoring run-time modified files using timeStampMaster (fileModificationSkew 5, maxFileModificationPolls 20)
allowSystemOperations : Allowing user-supplied system call operations

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //
Create time

Creating block mesh from "system/blockMeshDict"
Creating block edges
No non-planar block faces defined
Creating topology blocks

Creating topology patches - from boundary section

Creating block mesh topology - scaling/transform applied later

Check topology

        Basic statistics
                Number of internal faces : 0
                Number of boundary faces : 6
                Number of defined boundary faces : 6
                Number of undefined boundary faces : 0
        Checking patch -> block consistency

Creating block offsets
Creating merge list (topological search)...

Creating polyMesh from blockMesh
Creating patches
Creating cells
Creating points with scale (0.1 0.1 0.1)
    Block 0 cell size :
        i : 0.005 .. 0.005
        j : 0.005 .. 0.005
        k : 0.01 .. 0.01

No patch pairs to merge

Writing polyMesh with 0 cellZones
----------------
Mesh Information
----------------
  boundingBox: (0 0 0) (0.1 0.1 0.01)
  nPoints: 882
  nCells: 400
  nFaces: 1640
  nInternalFaces: 760
----------------
Patches
----------------
  patch 0 (start: 760 size: 20) name: movingWall
  patch 1 (start: 780 size: 60) name: fixedWalls
  patch 2 (start: 840 size: 800) name: frontAndBack

End

Meshing done.
```

The second section is the output log for solver execution:

```terminal {fileName="terminal"}
/*---------------------------------------------------------------------------*\
| =========                 |                                                 |
| \\      /  F ield         | OpenFOAM: The Open Source CFD Toolbox           |
|  \\    /   O peration     | Version:  2406                                  |
|   \\  /    A nd           | Website:  www.openfoam.com                      |
|    \\/     M anipulation  |                                                 |
\*---------------------------------------------------------------------------*/
Build  : _9bfe8264-20241212 OPENFOAM=2406 patch=241212 version=2406
Arch   : "LSB;label=32;scalar=64"
Exec   : ofsp_07_firstApp -case debug_case
Date   : Sep 01 2025
Time   : 18:09:17
Host   : aerosand
PID    : 200555
I/O    : uncollated
Case   : /home/aerosand/github/data_project/openfoam_sharing/ofsp/ofsp_07_firstApp/debug_case
nProcs : 1
trapFpe: Floating point exception trapping enabled (FOAM_SIGFPE).
fileModificationChecking : Monitoring run-time modified files using timeStampMaster (fileModificationSkew 5, maxFileModificationPolls 20)
allowSystemOperations : Allowing user-supplied system call operations

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //
Create time


ExecutionTime = 0 s  ClockTime = 0 s

End
```


## 5. Native Main Source Code

The code in `ofsp_07_firstApp.C` is:

```cpp {fileName="/ofsp_07_firstApp.C",linenos=table,linenostart=1}
/*---------------------------------------------------------------------------*\
  =========                 |
  \\      /  F ield         | OpenFOAM: The Open Source CFD Toolbox
   \\    /   O peration     |
    \\  /    A nd           | www.openfoam.com
     \\/     M anipulation  |
-------------------------------------------------------------------------------
    Copyright (C) 2025 AUTHOR,AFFILIATION
-------------------------------------------------------------------------------
License
    This file is part of OpenFOAM.

    OpenFOAM is free software: you can redistribute it and/or modify it
    under the terms of the GNU General Public License as published by
    the Free Software Foundation, either version 3 of the License, or
    (at your option) any later version.

    OpenFOAM is distributed in the hope that it will be useful, but WITHOUT
    ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or
    FITNESS FOR A PARTICULAR PURPOSE.  See the GNU General Public License
    for more details.

    You should have received a copy of the GNU General Public License
    along with OpenFOAM.  If not, see <http://www.gnu.org/licenses/>.

Application
    ofsp_07_firstApp

Description
	This section contains OpenFOAM comment content, providing the necessary introduction for this application.
\*---------------------------------------------------------------------------*/

#include "fvCFD.H"
// Includes classes, functions, etc., related to FVM

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    #include "setRootCase.H"
    // Sets up command-line arguments
    // Checks and determines the case directory
    // This will be discussed in detail later
    
    #include "createTime.H"
    // Creates and initializes the Time control object
    // This will be discussed in detail later

    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< nl;
    // Info is a global object provided by OpenFOAM for printing information and formatted output
    // nl is a newline character provided by OpenFOAM
    
    runTime.printExecutionTime(Info);
    // Function call used to print program execution time

    Info<< "End\n" << endl;

    return 0;
}


// ************************************************************************* //
// ************************************************************************* //

```

## 6. Using a Development Library

In this project, we selectively use a class from the `Aerosand` library developed in the previous section.

### 6.1. Main Source Code

The code in `ofsp_07_firstApp.C` is modified as follows:

```cpp {fileName="/ofsp_07_firstApp.C",linenos=table,linenostart=1}
#include "fvCFD.H"

#include "class1.H" // Include the header file of the development library we want to use

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    #include "setRootCase.H"
    #include "createTime.H"


    label a = 1;
    scalar pi = 3.1415926;
    Info<< "Hi, OpenFOAM!" << " Here we are." << nl
	    << a << " + " << pi << " = " << a + pi << nl
	    << a << " * " << pi << " = " << a * pi << nl
	    << endl;

    class1 mySolver;
    mySolver.SetLocalTime(0.2);
    Info<< "\nCurrent time step is : " << mySolver.GetLocalTime() << endl;

    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< nl;
    runTime.printExecutionTime(Info);

    Info<< "End\n" << endl;

    return 0;
}

```

### 6.2. Project Make

The project `Make/files` is:

```wmake {fileName="/Make/files"}
ofsp_07_firstApp.C

EXE = $(FOAM_USER_APPBIN)/ofsp_07_firstApp

```

> [!caution]
> Note that the development library created in the previous section needs to be linked here.

The project `Make/options` is:

```wmake {fileName="/Make/options"}
EXE_INC = \
    -I$(LIB_SRC)/finiteVolume/lnInclude \
    -I$(LIB_SRC)/meshTools/lnInclude \
    -I../ofsp_07_tensor/Aerosand/lnInclude

EXE_LIBS = \
    -lfiniteVolume \
    -lmeshTools \
    -L$(FOAM_USER_LIBBIN) \
    -lAerosand

```

We explain its syntax again:

- The prefix `EXE` indicates that this configuration is for an executable (the prefix `LIB` indicates configuration for a library).
- The suffix `INC` denotes include paths.
    - The `-I` flag specifies header file search paths.
    - Typically points to the `lnInclude` directory of an OpenFOAM module (linked include directory).
    - Can include system header paths or third-party library header paths.
- The suffix `LIBS` denotes library linking.
    - The `-l` flag specifies libraries to link.
    - Library names omit the `lib` prefix and `.so` suffix (e.g., `-lfiniteVolume` corresponds to `libfiniteVolume.so`).
    - The `-L` flag specifies library search paths.
- Paths can be written in various ways:
    - `$(LIB_SRC)` is usually a combination of **OpenFOAM environment variables** and **relative paths**.
    - It is also acceptable to write `-I$(FOAM_SRC)/finiteVolume/lnInclude`.
    - Absolute path: `-I/usr/lib/openfoam/openfoam2306/src/finiteVolume/lnInclude`.
    - Relative path: `-I../ofsp_07_tensor/Aerosand/lnInclude`.

## 7. Compilation and Execution

Since the `Aerosand` library discussed in the previous section has already been successfully compiled, it does not need to be recompiled here; it can be called directly.

```terminal {fileName="terminal"}
wclean
wmake
./caseclean
./caserun
```

> [!tip]
> Use the Tab key frequently for quick completion when typing.

The terminal output is as follows (only excerpts with significant changes are shown):

```terminal {fileName="terminal"}
...

Create time

Hi, OpenFOAM! Here we are.
1 + 3.14159 = 4.14159
1 * 3.14159 = 3.14159


Current time step is : 0.2

ExecutionTime = 0.01 s  ClockTime = 0 s

End
```

## 8. Summary

This section has completed the following discussions:

- [x] Understanding the file structure of standard OpenFOAM applications
- [x] Learning to use scripts
- [x] Understanding the application development and testing process
- [x] Calling development libraries located elsewhere
- [x] Compiling and running the firstApp project



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


