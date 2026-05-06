---
uid: 20251117115757
title: 16_meshRatio
date: 2025-11-17
update: 2026-03-26
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
weight: 17
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

Based on our previous discussion of meshes, we will attempt to develop an application that computes mesh parameters, allowing readers to practice and deepen their understanding of meshes and the OpenFOAM project workflow.

The code for this project is adapted from Tom Smith’s lecture notes, *Programming with OpenFOAM* (Tutorial at The 3rd UCL OpenFOAM Workshop).

This section primarily discusses:

- [ ] Practicing the use of mesh class methods
- [ ] Considering code execution efficiency
- [ ] Reviewing the use of custom dictionary files
- [ ] Compiling and running a meshRatio project

## 1. Project Preparation

Run the following commands in the terminal to create the project:

```terminal {fileName="terminal"}
ofsp
foamNewApp ofsp_16_meshRatio
cd ofsp_16_meshRatio
cp -r $FOAM_TUTORIALS/incompressible/simpleFoam/pitzDaily debug_case
code .
```

Test the initial solver, and provide scripts and documentation.

## 2. Mesh Volume Ratio

Modify the main source code as follows:

```cpp {fileName="ofsp_16_meshRatio/ofsp_16_meshRatio.C",linenos=table,linenostart=1}
#include "fvCFD.H"

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    #include "setRootCase.H"
    #include "createTime.H"

    #include "createMesh.H"

    const scalar c = mesh.C().size();
    Info<< "Number of cells = " << c << endl;
    // Number of mesh cells

    const labelListList& neighbour = mesh.cellCells();
    // Returns a new list of neighboring cells

    List<scalar> ratios(0); // Declare a scalar list initialized to 0
    scalar volumeRatio = 0;

    forAll (neighbour, cellI) // Iterate over neighboring cells (i.e., all cells)
    {
        List<label> n = neighbour[cellI];
        // For each cell, obtain its neighboring cells into list n

        const scalar cellVolume = mesh.V()[cellI];
        // Volume of the current cell

        forAll (n, i) // For the current cell, iterate over its neighboring cells
        {
            label neighbourIndex = n[i];
            scalar neighbourVolume = mesh.V()[neighbourIndex];
            // Volume of the neighboring cell

            if (neighbourVolume >= cellVolume) // Cases where volume ratio > 1
            {
                volumeRatio = neighbourVolume / cellVolume;
                ratios.append(volumeRatio);
                // Store the volume ratio in the ratios list
                // Convenient but inefficient
            }
        }
    }

    Info<< "Maximum volume ratio = " << max(ratios) << endl;
    // Maximum volume ratio


    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< nl;
    runTime.printExecutionTime(Info);

    Info<< "End\n" << endl;

    return 0;
}

```

Execute the `caserun` script, or directly generate the mesh for the test case and run the solver:

```terminal {fileName="terminal"}
wmake
blockMesh -case debug_case
ofsp_16_meshRatio -case debug_case
```

The terminal output is as follows:

```terminal {fileName="terminal"}
Create time

Create mesh for time = 0

Number of cells = 12225
Maximum volume ratio = 1.49908

ExecutionTime = 0.11 s  ClockTime = 0 s

End
```

## 3. Improving Volume Ratio Calculation

Although the `append` method of the `List` class is convenient, it is very inefficient because each time data is added to the `List`, a new `List` of size `n+1` is created, the old data is copied over, and the new data is appended. When dealing with large amounts of data, this process of creating and copying each time new data is added is highly inefficient.

We can consider that for a given mesh, the total number of volume ratios to be computed is fixed. We can declare a `List` of fixed size, allocate memory for it, and then simply add data to it.

The improved main source code is as follows:

```cpp {fileName="ofsp_16_meshRatio/ofsp_16_meshRatio.C",linenos=table,linenostart=1}
#include "fvCFD.H"

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    #include "setRootCase.H"
    #include "createTime.H"

    #include "createMesh.H"

    const scalar c = mesh.C().size();
    Info<< "Number of cells = " << c << endl;

    const labelListList& neighbour = mesh.cellCells();

	/*
	* Code improvement as follows
	*/
    label len = mesh.Cf().size(); // Size of the required ratios List
    scalar initial = 0;
    List<scalar> ratios(len, initial); // Create a fixed-size List
    label counter = 0; // Index for ratios
    scalar volumeRatio = 0;

    forAll (neighbour, cellI) // Iterate over neighboring cells
    {
        List<label> n = neighbour[cellI];

        const scalar cellVolume = mesh.V()[cellI];

        forAll (n, i)
        {
            label neighbourIndex = n[i];
            scalar neighbourVolume = mesh.V()[neighbourIndex];

            if (neighbourVolume >= cellVolume)
            {
                volumeRatio = neighbourVolume / cellVolume;
                ratios[counter] = volumeRatio; // Store the volume ratio
                counter += 1;
            }
        }
    }

    Info<< "Maximum volume ratio = " << max(ratios) << endl;

    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< nl;
    runTime.printExecutionTime(Info);

    Info<< "End\n" << endl;

    return 0;
}

```

Run the following commands in the terminal to compile and run:

The terminal output is as follows:

```terminal {fileName="terminal"}
Create time

Create mesh for time = 0

Number of cells = 12225
Maximum volume ratio = 1.49908

ExecutionTime = 0.05 s  ClockTime = 0 s

End
```

It can be clearly seen that the execution time has decreased from 0.13 s to 0.06 s, significantly improving efficiency.

## 4. Maximum Volume Ratio

If we only need to obtain the maximum volume ratio, we do not actually need to store all volume ratios; storing only the maximum value is sufficient.

The main source code is as follows:

```cpp {fileName="ofsp_16_meshRatio/ofsp_16_meshRatio.C",linenos=table,linenostart=1}
#include "fvCFD.H"

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    #include "setRootCase.H"
    #include "createTime.H"

    #include "createMesh.H"

    const scalar c = mesh.C().size();
    Info<< "Number of cells = " << c << endl;

    const labelListList& neighbour = mesh.cellCells();

	/*
	* Code improvement as follows
	*/
    scalar volumeRatio = 0.0;
    scalar currentRatio = 0.0;

    forAll (neighbour, cellI)
    {
        List<label> n = neighbour[cellI];

        const scalar cellVolume = mesh.V()[cellI];

        forAll (n, i)
        {
            label neighbourIndex = n[i];
            scalar neighbourVolume = mesh.V()[neighbourIndex];

            if (neighbourVolume >= cellVolume)
            {
                volumeRatio = neighbourVolume / cellVolume;

                if (volumeRatio > currentRatio) // Store only the maximum volume ratio
                {
                    currentRatio = volumeRatio;
                }
            }
        }
    }

    Info<< "Maximum volume ratio = " << currentRatio << endl;

    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< nl;
    runTime.printExecutionTime(Info);

    Info<< "End\\n" << endl;

    return 0;
}

```

Compile and run the project.

The terminal output is as follows:

```terminal {fileName="terminal"}
Create time

Create mesh for time = 0

Number of cells = 12225
Maximum volume ratio = 1.49908

ExecutionTime = 0.05 s  ClockTime = 0 s

End
```


## 5. Volume Ratio Criterion

Simply obtaining the volume ratio is not sufficient; we may also want to know how many cells in the test case exceed a specified volume ratio criterion.

We will introduce the volume ratio criterion parameter via an OpenFOAM dictionary.

Provide a dictionary for the test case at `/ofsp_16_meshRatio/debug_case/system/volumeRatioDict` with the following content:


```cpp {fileName="ofsp_16_meshRatio/debug_case/system/volumeRatioDict",linenos=table,linenostart=1}
FoamFile
{
    version     2.0;
    format      ascii;
    class       dictionary;
    object      blockMeshDict;
}
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

maxRatio 1.2;

```

Users can modify the parameters via the dictionary file without recompiling the project.


> [!tip]
> We adopt the following conventions:
> - `Properties` files are placed in the `/userApp/constant` folder
> - `Dict` files are placed in the `/userApp/system` folder

The main source code is as follows:

```cpp {fileName="ofsp_16_meshRatio/ofsp_16_meshRatio.C",linenos=table,linenostart=1}
#include "fvCFD.H"

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    #include "setRootCase.H"
    #include "createTime.H"

    #include "createMesh.H"

    IOdictionary volumeRatioDict // Create a dictionary object
    (
        IOobject
        (
            "volumeRatioDict", // File name
            runTime.system(), // File path
            mesh, // Construct based on the mesh
            IOobject::MUST_READ, // Must read
            IOobject::NO_WRITE // Read-only, no writing
        )
    );

    const scalar c = mesh.C().size();
    Info<< "Number of cells = " << c << endl;

    const labelListList& neighbour = mesh.cellCells();

    scalar volumeRatio = 0.0;
    scalar currentRatio = 0.0;

    label nFail = 0; // Counter for cells exceeding the criterion

    scalar maxRatio(readScalar(volumeRatioDict.lookup("maxRatio")));
	// Read from the dictionary

    forAll (neighbour, cellI)
    {
        List<label> n = neighbour[cellI];

        const scalar cellVolume = mesh.V()[cellI];

        forAll (n, i)
        {
            label neighbourIndex = n[i];
            scalar neighbourVolume = mesh.V()[neighbourIndex];

            if (neighbourVolume >= cellVolume)
            {
                volumeRatio = neighbourVolume / cellVolume;

                if (volumeRatio > currentRatio)
                {
                    currentRatio = volumeRatio;
                }

                if (volumeRatio > maxRatio) // Count cells exceeding the criterion
                {
                    nFail += 1;
                }
            }
        }
    }

    Info<< "Maximum volume ratio = " << currentRatio << nl
        << "Number of cell volume ratios exceeding " << maxRatio
        << " = " << nFail << nl
        << endl;

    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< nl;
    runTime.printExecutionTime(Info);

    Info<< "End\n" << endl;

    return 0;
}

```

Compile and run the project.

The terminal output is as follows:

```terminal {fileName="terminal"}
Create time

Create mesh for time = 0

Number of cells = 12225
Maximum volume ratio = 1.49908
Number of cell volume ratios exceeding 1.2 = 532


ExecutionTime = 0.04 s  ClockTime = 0 s

End
```

## 6. Summary

This section continued the discussion of mesh class methods and provided practical examples. Through repeated practice, readers can review previous knowledge points, overcome any unfamiliarity with OpenFOAM programming, and facilitate subsequent learning.

This section has completed the following discussions:

- [x] Practicing the use of mesh class methods
- [x] Considering code execution efficiency
- [x] Reviewing the use of custom dictionary files
- [x] Compiling and running a meshRatio project


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


