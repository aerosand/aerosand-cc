---
uid: 20251125114545
title: 18_fieldCal
date: 2025-11-25
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
weight: 19
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

Based on the previous discussions, we will practice a simple case of field computation.

This section primarily discusses:

- [ ] Adding simple physical fields for computation
- [ ] Managing physical fields in the OpenFOAM way
- [ ] Practicing development library usage
- [ ] Compiling and running a fieldCal project

## 1. Project Preparation

Run the following commands in the terminal to create the project:

```terminal {fileName="terminal"}
ofsp
foamNewApp ofsp_18_fieldCal
cd ofsp_18_fieldCal
cp -r $FOAM_TUTORIALS/incompressible/icoFoam/cavity/cavity debug_case
code .
```

Test the initial solver, and provide scripts (refer to previous scripts) and documentation.

## 2. Adding Physical Fields

The copied test case does not have a temperature field. We will add an initial temperature field to the case.

### 2.1. Temperature Field

Temperature, like pressure, is a scalar. We can obtain the temperature field file by copying the pressure field file and modifying it.

Run the following command in the terminal to prepare the temperature field file:

```terminal {fileName="terminal"}
cp -r debug_case/0/p debug_case/0/T
code debug_case/0/T
```

The modified temperature field file is as follows:

```cpp {fileName="debug_case/0/T",linenos=table,linenostart=1}
FoamFile
{
    version     2.0;
    format      ascii;
    class       volScalarField;
    object      T;
}
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

// dimensions      [0 2 -2 0 0 0 0];
dimensions      [0 0 0 1 0 0 0];

// internalField   uniform 0;
internalField   uniform 2026;

boundaryField
{
    movingWall
    {
        // type            zeroGradient;
        type            fixedValue;
        value           uniform 2050;
    }

    fixedWalls
    {
        type            zeroGradient;
    }

    frontAndBack
    {
        type            empty;
    }
}

```

### 2.2. Main Source Code

Modify the main source code as follows:

```cpp {fileName="ofsp_18_fieldCal.C",linenos=table,linenostart=1}
#include "fvCFD.H"

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //
scalar calculatePressure(scalar t, vector x, vector x0, scalar scale);
// Custom function prototype declaration


int main(int argc, char *argv[])
{
    #include "setRootCase.H"
    #include "createTime.H"

    #include "createMesh.H"

    Info<< "Reading transportProperties\n" << endl;
    IOdictionary transportProperties // Construct a dictionary object to read parameters from the dictionary file
    (
        IOobject
        (
            "transportProperties", // Dictionary file name
            runTime.constant(), // Dictionary file path
            mesh, // Associate with the mesh
            IOobject::MUST_READ_IF_MODIFIED,
            IOobject::NO_WRITE
        )
    );
    dimensionedScalar nu("nu",dimViscosity,transportProperties); // Read the parameter from the dictionary file

    Info<< "Reading field p\n" << endl;
    volScalarField p // Construct the p field
    (
        IOobject // Based on IOobject
        (
            "p", // Field file name
            runTime.timeName(), // Field file path
            mesh, // Associate IOobject with the mesh
            IOobject::MUST_READ, // Initial conditions must be provided
            IOobject::AUTO_WRITE
        ),
        mesh // Associate with the mesh
    );

    Info<< "Reading field T\n" << endl;
    volScalarField T // Construct the T field
    (
        IOobject
        (
            "T",
            runTime.timeName(),
            mesh,
            IOobject::MUST_READ,
            IOobject::AUTO_WRITE
        ),
        mesh
    );

    Info<< "Reading field U\n" << endl;
    volVectorField U // Construct the U field
    (
        IOobject
        (
            "U",
            runTime.timeName(),
            mesh,
            IOobject::MUST_READ,
            IOobject::AUTO_WRITE
        ),
        mesh
    );

	// Example of creating a zero field
	// Refer to the GeometricField constructor discussed earlier
    volScalarField zeroScalarField
    (
        IOobject
        (
            "zeroScalarField",
            runTime.timeName(),
            mesh,
            IOobject::NO_READ, // No initial condition file needed
            IOobject::AUTO_WRITE
        ),
        mesh,
        dimensionedScalar("zeroScalarField",dimless,Zero) // Initialize to zero
    );

    volVectorField zeroVectorField
    (
        IOobject
        (
            "zeroVectorField",
            runTime.timeName(),
            mesh,
            IOobject::NO_READ,
            IOobject::AUTO_WRITE
        ),
        mesh,
        dimensionedVector("zeroVectorField",dimless,Zero) // Initialize to zero
    );

    volVectorField gradT // Construct the gradT field
    (
        IOobject
        (
            "gradT",
            runTime.timeName(),
            mesh,
            IOobject::NO_READ,
            IOobject::AUTO_WRITE
        ),
        fvc::grad(T) // Since it is computed from the T field, no need to associate with the mesh again
    );

    Info<< "Reading/calculating face flux field phi\n" << endl;
    surfaceScalarField phi // Construct the phi field
    (
        IOobject
        (
            "phi",
            runTime.timeName(),
            mesh,
            IOobject::READ_IF_PRESENT,
            IOobject::AUTO_WRITE
        ),
        fvc::flux(U)
        // fvc::interpolate(U)*mesh.Sf(); // Equivalent to this line
    );


    const vector originVector(0.05, 0.05, 0.005); // Construct an originVector object of the vector class with initial values

    const scalar rFarCell = // Construct a scalar object rFarCell
    max // Find the maximum value
    (
        mag // Compute the magnitude
        (
            dimensionedVector("x0", dimLength, originVector)
            // Create a dimensionedVector from originVector
            - mesh.C() // Subtract the mesh cell center coordinates, which are also vectors
        )
    ).value(); // Return the magnitude value

    Info<< "\nStarting time loop\n" << endl;

    while (runTime.loop()) // Time advancement
    {
        Info<< "Time = " << runTime.timeName() << nl << endl;

        for (label cellI=0; cellI<mesh.C().size(); ++cellI) // Iterate over all mesh cells
        {
            p[cellI] = calculatePressure // Compute the p field using the custom function
            (
                runTime.time().value(),
                mesh.C()[cellI],
                originVector,
                rFarCell
            );
        }

		// Simulate the computation of the velocity field
        U = fvc::grad(p) * dimensionedScalar("tmp",dimTime,1.0); // Compute the U field
        // Multiply by a temporary quantity with units to ensure unit consistency on both sides

        runTime.write(); // Write the computed fields
    }

    Info<< "Calculation done." << endl;

    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< nl;
    runTime.printExecutionTime(Info);

    Info<< "End\n" << endl;

    return 0;
}


// ************************************************************************* //

// Implementation of the custom function
scalar calculatePressure(scalar t, vector x, vector x0, scalar scale)
{
    scalar r(mag(x - x0) / scale); // Construct a scalar object r and initialize

    scalar rR(1.0 / (r + 1e-12)); // Same as above

    scalar f(1.0); // Same as above

    return Foam::sin(2.0 * Foam::constant::mathematical::pi * f * t) * rR;
    // Returns a computed value with no physical significance
}

```

### 2.3. Compilation and Execution

Compile and run this project (ensure that the temperature field file is not cleared).

The terminal output is as follows:

```terminal {fileName="terminal"}
Create time

Create mesh for time = 0

Reading transportProperties

Reading field p

Reading field T

Reading field U

Reading/calculating face flux field phi


Starting time loop

Time = 0.005

Time = 0.01

Time = 0.015

Time = 0.02

...

Time = 0.49

Time = 0.495

Time = 0.5

Calculation done.

ExecutionTime = 0.02 s  ClockTime = 0 s

End

```

Run the following command in the terminal to visualize the results:

```terminal {fileName="terminal"}
paraFoam -case debug_case
```

Note that new time step field files appear in the `debug_case/` folder. Taking `debug_case/0.3/` as an example, the computed result files contained are as follows:

```terminal {fileName="terminal"}
debug_case/0.3
├── gradT
├── p
├── phi
├── T
├── U
├── uniform
│   ├── functionObjects
│   │   └── functionObjectProperties
│   └── time
├── zeroScalarField
└── zeroVectorField
```

You can view the computed numerical values in each field file.

>[!warning]
>When opening `0.3/phi`, you can see that the values have not been updated, and other files besides p and U have not been updated either. This is because the computation formulas given during field creation only initialize the fields; they do not automatically update as time advances. Therefore, the required fields need to be computed in the time loop of the main source code.

## 3. Project Organization

In practical OpenFOAM applications, generally:

- Field inclusion should be placed in a separate file `createFields.H`
- OpenFOAM itself provides `createPhi.H`
- Custom methods should also be placed in separate files, such as `calculatePressure.H`

Therefore, the file structure of this application after reorganization is:

```terminal {fileName="terminal"}
tree -L 1
.
├── calculatePressure.H
├── caseclean
├── caserun
├── createFields.H
├── debug_case
├── Make
└── ofsp_18_fieldCal.C
```

### 3.1. createFields.H

The field inclusion file `createFields.H` is as follows:

```cpp {fielName="createFields.H",linenos=table,linenostart=1}
Info<< "Reading transportProperties\n" << endl;
    IOdictionary transportProperties
    (
        IOobject
        (
            "transportProperties",
            runTime.constant(),
            mesh,
            IOobject::MUST_READ_IF_MODIFIED,
            IOobject::NO_WRITE
        )
    );

    dimensionedScalar nu("nu",dimViscosity,transportProperties);

    Info<< "Reading field p\n" << endl;
    volScalarField p
    (
        IOobject
        (
            "p",
            runTime.timeName(),
            mesh,
            IOobject::MUST_READ,
            IOobject::AUTO_WRITE
        ),
        mesh
    );

    Info<< "Reading field T\n" << endl;
    volScalarField T
    (
        IOobject
        (
            "T",
            runTime.timeName(),
            mesh,
            IOobject::MUST_READ,
            IOobject::AUTO_WRITE
        ),
        mesh
    );

    Info<< "Reading field U\n" << endl;
    volVectorField U
    (
        IOobject
        (
            "U",
            runTime.timeName(),
            mesh,
            IOobject::MUST_READ,
            IOobject::AUTO_WRITE
        ),
        mesh
    );

    volScalarField zeroScalarField
    (
        IOobject
        (
            "zeroScalarField",
            runTime.timeName(),
            mesh,
            IOobject::NO_READ,
            IOobject::AUTO_WRITE
        ),
        mesh,
        dimensionedScalar("zeroScalarField",dimless,Zero)
    );

    volVectorField zeroVectorField
    (
        IOobject
        (
            "zeroVectorField",
            runTime.timeName(),
            mesh,
            IOobject::NO_READ,
            IOobject::AUTO_WRITE
        ),
        mesh,
        dimensionedVector("zeroVectorField",dimless,Zero)
    );

    volVectorField gradT
    (
        IOobject
        (
            "gradT",
            runTime.timeName(),
            mesh,
            IOobject::NO_READ,
            IOobject::AUTO_WRITE
        ),
        fvc::grad(T)
    );

    Info<< "Reading/calculating face flux field phi\n" << endl;
    surfaceScalarField phi
    (
        IOobject
        (
            "phi",
            runTime.timeName(),
            mesh,
            IOobject::READ_IF_PRESENT,
            IOobject::AUTO_WRITE
        ),
        fvc::flux(U)
        // fvc::interpolate(U)*mesh.Sf();
    );

```

### 3.2. calculatePressure.H

The custom method `calculatePressure.H` is as follows:

```cpp {fileName="calculatePressure.H",linenos=table,linenostart=1}
scalar calculatePressure(scalar t, vector x, vector x0, scalar scale)
{
    scalar r(mag(x - x0) / scale);

    scalar rR(1.0 / (r + 1e-12));

    scalar f(1.0);

    return Foam::sin(2.0 * Foam::constant::mathematical::pi * f * t) * rR;
}

```

### 3.3. Main Source Code

The main source code is reorganized as follows:

```cpp {fileName="ofsp_18_fieldCal.C",linenos=table,linenostart=1}
#include "fvCFD.H"

#include  "calculatePressure.H"

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    #include "setRootCase.H"
    #include "createTime.H"

    #include "createMesh.H"

    #include "createFields.H"

    const vector originVector(0.05, 0.05, 0.005);

    const scalar rFarCell  =
    max
    (
        mag
        (
            dimensionedVector("x0", dimLength, originVector)
            - mesh.C()
        )
    ).value();

    Info<< "\nStarting time loop\n" << endl;

    while (runTime.loop())
    {
        Info<< "Time = " << runTime.timeName() << nl << endl;

        for (label cellI=0; cellI<mesh.C().size(); ++cellI)
        {
            p[cellI] = calculatePressure
            (
                runTime.time().value(),
                mesh.C()[cellI],
                originVector,
                rFarCell
            );
        }

        U = fvc::grad(p) * dimensionedScalar("tmp",dimTime,1.0);

		phi = fvc::flux(U); // Add phi computation

        runTime.write();
    }

    Info<< "Calculation done." << endl;

    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< nl;
    runTime.printExecutionTime(Info);

    Info<< "End\n" << endl;

    return 0;
}

```

### 3.4. Compilation and Execution

Compile and run. The results will be the same.

After reorganization:

- The main source code has a clear functional division
- Each function is easier to maintain

## 4. Development Library

When the implementation of a certain type of method can be retained as a specific functionality for future use, it can be fixed as a development library for use in various projects.

Refer to the previous discussion: [https://aerosand.cc/docs/ofs/ofsp/03_wmake/](https://aerosand.cc/docs/ofs/ofsp/03_wmake/)

### 4.1. Development Library Code

We will create a development library `calculateVelocityPressure` and provide the corresponding files.

Run the following command in the terminal to create the development library:

```terminal {fileName="terminal"}
foamNewApp calculateVelocityPressure
```

After creating the development library, the project file structure is as follows:

```terminal {fileName="terminal"}
.
├── calculateVelocityPressure
│   ├── calculateVelocityPressure.C
│   ├── calculateVelocityPressure.H
│   └── Make
│       ├── files
│       └── options
├── caseclean
├── caserun
├── createFields.H
├── debug_case
│   ├── 0.org
│   │   ├── p
│   │   ├── T
│   │   └── U
│   ├── constant/
│   └── system/
├── Make
│   ├── files
│   └── options
└── ofsp_18_fieldCal.C
```

The development library declaration `calculateVelocityPressure.H` is as follows:

```cpp {fileName="calculateVelocityPressure.H",linenos=table,linenostart=1}
#include "fvCFD.H"

scalar calculateR(const fvMesh& mesh, volScalarField& r, dimensionedVector x0);

volScalarField calculatePressure(const fvMesh& mesh, scalar t, dimensionedVector x0, scalar scale, scalar f);

void calculateVelocity(const fvMesh& mesh, volVectorField& U, word pName = "p");
// The third parameter is given a default value

```

The development library definition `calculateVelocityPressure.C` is as follows:

```cpp {fileName="calculateVelocityPressure.C",linenos=table,linenostart=1}
#include "calculateVelocityPressure.H"

scalar calculateR(const fvMesh& mesh, volScalarField& r, dimensionedVector x0)
{
    r = mag(mesh.C() - x0); // Field r stores the distance from each cell center to the reference vector

    return max(r).value(); // Return the maximum distance
}

volScalarField calculatePressure // Compute the pressure field, note the return type
(
    const fvMesh& mesh, // Pass a reference to the mesh, reference passing is efficient
    scalar t,
    dimensionedVector x0, // Not just a vector, but a vector with dimensions
    scalar scale, // Pure scalar
    scalar f // Pure scalar
)
{
    volScalarField r(mag(mesh.C()-x0)/scale);
    // In the previous discussion, r was of scalar type; here it is of volScalarField type
    // Pay attention to the consistency of variable types in computations

    volScalarField rR(1.0/(r+dimensionedScalar("tmp",dimLength,1e-12)));
    // Assign dimensions to 1e-12 to pass OpenFOAM's unit checking
    // Pay attention to the consistency of result units in computations

    return Foam::sin(2.0*Foam::constant::mathematical::pi*f*t)*rR*dimensionedScalar("tmp",pow(dimLength,3)/pow(dimTime,2),1.0);
    // Units can be computed mathematically
    // Similarly, multiply by a temporary quantity with units to ensure result unit consistency
    // The unit of the result returned by this function must match the unit of the variable receiving this value in the main function
}

void calculateVelocity(const fvMesh& mesh, volVectorField& U, word pName)
// Note that the default value for the formal parameter cannot be specified in the function implementation
{
    const volScalarField& pField = mesh.lookupObject<volScalarField>(pName);
    // Look up the field by the pName and assign it to pField

    U = fvc::grad(pField) * dimensionedScalar("tmp",dimTime,1.0);
    // Compute U, ensuring unit consistency
}

```

### 4.2. Development Library Make

The development library `Make/files` is as follows:

```wmake {fileName="calculateVelocityPressure/Make/files",linenos=table,linenostart=1}
calculateVelocityPressure.C

LIB = $(FOAM_USER_LIBBIN)/libcalculateVelocityPressure

```

The development library `Make/options` is as follows:

```wmake {fileName="calculateVelocityPressure/Make/options",linenos=table,linenostart=1}
EXE_INC = \
    -I$(LIB_SRC)/finiteVolume/lnInclude \
    -I$(LIB_SRC)/meshTools/lnInclude

EXE_LIBS = \
    -lfiniteVolume \
    -lmeshTools

```

When compiling this library, it does not require additional libraries beyond these basic ones, so the above options are sufficient.

### 4.3. Development Library Compilation

Run the following command in the terminal to compile the development library:

```terminal {fileName="terminal"}
wmake calculateVelocityPressure
```

The terminal indicates that the development library has been successfully compiled and is ready for use.

### 4.4. Field Inclusion

Keep the `createFields.H` code unchanged.

### 4.5. Main Source Code

Modify the main source code `ofsp_18_fieldCal.C` as follows:

```cpp {fileName="ofsp_18_fieldCal.C",linenos=table,linenostart=1}
#include "fvCFD.H"

#include "calculateVelocityPressure.H"

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    #include "setRootCase.H"
    #include "createTime.H"

    #include "createMesh.H"

    #include "createFields.H"

    dimensionedVector originVector("x0",dimLength,vector(0.05,0.05,0.005));
    // Construct an originVector object of the dimensionedVector class, assigning units and initial values

    scalar f(1.0);

    volScalarField r // This is an intermediate quantity for computation, so it is not placed in createFields.H
    (
        IOobject
        (
            "r",
            runTime.timeName(),
            mesh,
            IOobject::NO_READ,
            IOobject::NO_WRITE
        ),
        mesh,
        dimensionedScalar("r",dimLength,0.0)
    );

    Info<< "This is a test" << endl;

    const scalar rFarCell = calculateR(mesh,r,originVector);
    // Compute the maximum distance and store it in the rFarCell variable

    Info<< "\nStarting time loop\n" << endl;

    while (runTime.loop()) // Time advancement
    {
        Info<< "Time = " << runTime.timeName() << nl << endl;

        p = calculatePressure(mesh,runTime.time().value(),originVector,rFarCell,f);
        // Compute the p field

        calculateVelocity(mesh,U);
        // Compute the U field

        phi = fvc::flux(U);
        // Compute the phi field

        runTime.write(); // Write the fields
    }

    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< nl;
    runTime.printExecutionTime(Info);

    Info<< "End\n" << endl;

    return 0;
}

```

### 4.6. Project Make

The project `Make/files` is as follows:

```wmake {fileName="Make/files",linenos=table,linenostart=1}
ofsp_18_fieldCal.C

EXE = $(FOAM_USER_APPBIN)/ofsp_18_fieldCal

```

The project `Make/options` is as follows:

```wmake {fileName="Make/options",linenos=table,linenostart=1}
EXE_INC = \
    -I$(LIB_SRC)/finiteVolume/lnInclude \
    -I$(LIB_SRC)/meshTools/lnInclude \
    -IcalculateVelocityPressure/lnInclude

EXE_LIBS = \
    -lfiniteVolume \
    -lmeshTools \
    -L$(FOAM_USER_LIBBIN) -lcalculateVelocityPressure

```

Note that when using a development library, both "inclusion" and "linking" are required. Both the "path" and the "name" must be specified.

### 4.7. Compilation and Execution

Compile and run the project.

The terminal output is as follows:

```terminal {fileName="terminal"}
Create time

Create mesh for time = 0

Reading transportProperties

Reading field p

Reading field T

Reading field U

Reading/calculating face flux field phi

This is a test

Starting time loop

Time = 0.005

Time = 0.01

Time = 0.015

Time = 0.02

...

Time = 0.49

Time = 0.495

Time = 0.5


ExecutionTime = 0.02 s  ClockTime = 0 s

End
```

### 4.8. Post-Processing

We can also visualize the computed results using ParaView.

Run the following command in the terminal to visualize the results:

```terminal {fileName="terminal"}
paraFoam -case debug_case
```

## 5. Summary

Reflecting on the previous discussions, we can now appreciate the core elements of numerical computation: time, mesh, and physical fields. With these tools, we can construct mathematical physical models into mathematical equations, thereby forming systems of linear algebraic equations for numerical solution. Do not rush; we are getting closer to solver programming.

This section has completed the following discussions:

- [ ] Adding simple physical fields for computation
- [ ] Managing physical fields in the OpenFOAM way
- [ ] Practicing development library usage
- [ ] Compiling and running a fieldCal project



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


