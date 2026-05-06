---
uid: 20251125190436
title: 19_firstSol
date: 2025-11-25
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
weight: 20
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

Through the discussions in the previous two stages, we have become familiar with the architecture of numerical applications in OpenFOAM, the key elements of numerical computation (time, mesh, fields), as well as some programming syntax and workflows. Now, using the tools we have acquired, we will attempt to write the simplest possible OpenFOAM solver (without involving numerical algorithms).

The general fundamental equation in computational fluid dynamics is:

$$\frac{\partial}{\partial t}(\rho \phi) + \nabla \cdot (\rho U\phi) = \nabla\cdot(\Gamma\nabla\phi) + S_{\phi}$$

We use $\phi$ to denote the physical field to be solved, taking care not to confuse it with the common flux `phi` in OpenFOAM.

For simplicity, we consider the following governing equation:

$$\frac{\partial}{\partial t}\phi + \nabla\cdot(U\phi) - \nabla\cdot(\gamma\nabla \phi)=0$$

This is a transient, incompressible, source-free transport problem.

We will attempt to programmatically solve this governing equation.

This section primarily discusses:

- [ ] Refining script writing
- [ ] Briefly understanding the correspondence between equation construction and code
- [ ] Understanding the simple workflow of numerical computation
- [ ] Compiling and running the firstSol project


## 1. Project Preparation

Run the following commands in the terminal to create the project:

```terminal {fileName="terminal"}
ofsp
foamNewApp ofsp_19_firstSol
cd ofsp_19_firstSol
cp -r $FOAM_TUTORIALS/incompressible/icoFoam/cavity/cavity debug_case
code .
```

Test the initial solver, and provide scripts and documentation.

To further facilitate test case management, we will split the scripts into four parts:

- Pre-processing `casepre`
    - Backup and restore initial conditions
    - Mesh generation (optional)
    - Domain setup (optional)
    - etc.
- Computation processing `caserun`
    - Run the solver on the test case
- Post-processing `casepost`
    - Post-processing of results (optional)
    - Result visualization
- Case cleaning `caseclean`
    - Clean up the test case
    - Restore the case to its initial state

> [!note]
> Unless otherwise specified, these scripts can be directly copied for subsequent use.

Pre-processing `casepre` is as follows:

```bash {fileName="casepre",linenos=table,linenostart=1}
#!/bin/sh
cd "${0%/*}" || exit 1                              # Run from this directory
#------------------------------------------------------------------------------

cd debug_case # if run in debug_case/ folder

FOLDER=$(pwd) 
FOLDER0=$FOLDER/0
FOLDER0org=$FOLDER/0.org

if test -d "$FOLDER0" && test -d "$FOLDER0org"; then
  rm -rf 0/
  cp -r 0.org/ 0/
elif test -d "$FOLDER0"; then
  cp -r 0/ 0.org/
elif test -d "$FOLDER0org"; then
  cp -r 0.org/ 0/
fi
echo "\n>>>>>>>>>>>>> Initial condition done.\n"

#blockMesh > log.mesh # meshing in the background & logging
blockMesh | tee log.mesh # meshing & logging
echo "\n>>>>>>>>>>>>> Mesh done.\n"

# if needs setFields
# setFields
# echo "\n>>>>>>>>>>>>> Setting fileds done.\n"

```

Computation processing `caserun` is as follows:

```bash {fileName="caserun",linenos=table,linenostart=1}
#!/bin/sh
cd "${0%/*}" || exit 1                              # Run from this directory
#------------------------------------------------------------------------------

appPath=$(cd `dirname $0`; pwd)                     # Get application path
appName="${appPath##*/}"                            # Get application name

cd debug_case

# FOLDER=$(pwd) 
# FOLDER0=$FOLDER/0

# if ! test -d "$FOLDER0"; then
#     cp -r 0.org/ 0/
# fi

# $appName > log.run # if run in the background to save time
$appName | tee log.run

# pyFoamPlotWatcher.py log.run # if installed the PyFoam

```

Post-processing `casepost` is as follows:

```bash {fileName="casepost",linenos=table,linenostart=1}
#!/bin/sh
cd "${0%/*}" || exit 1                              # Run from this directory
#------------------------------------------------------------------------------

cd debug_case

paraFoam

```

Case cleaning `caseclean` is as follows:

```bash {fileName="caseclean",linenos=table,linenostart=1}
#!/bin/sh
cd "${0%/*}" || exit 1                              # Run from this directory
#------------------------------------------------------------------------------

appPath=$(cd `dirname $0`; pwd)
appName="${casePath##*/}"

cd debug_case

rm -rf log.*
# foamCleanTutorials                              # if clean mesh and time folders
foamListTimes -rm                                # only clean time folders
rm -rf 0/                                       # clean initial 0
echo "\n>>>>>>>>>>>>> Cleaning done.\n"

```

## 2. Field Inclusion

The field inclusion `createFields.H` is as follows:

```cpp {fileName="createFields.H",linenos=table,linenostart=1}
Info<< "Reading transportProperties\n" << endl;
IOdictionary transportProperties
(
    IOobject
    (
        "transportProperties",
        runTime.constant(),
        mesh,
        IOobject::MUST_READ,
        IOobject::NO_WRITE
    )
);

Info<< "Reading diffusivity\n" << endl;
dimensionedScalar gamma("gamma",dimViscosity,transportProperties);


Info<< "Reading field A\n" << endl; // Introduce the physical field to be solved, e.g., A
volScalarField A
(
    IOobject
    (
        "A",
        runTime.timeName(),
        mesh,
        IOobject::MUST_READ,
        IOobject::AUTO_WRITE
    ),
    mesh
);

Info<< "Reading field U\n" << endl; // Velocity field
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

#include "createPhi.H"

```

## 3. Equation Construction

We modify the main source code to implement one computation of the mathematical equation.

The main source code `ofsp_19_firstSol.C` is as follows:

```cpp {fileName="ofsp_19_firstSol.C",linenos=table,linenostart=1}
#include "fvCFD.H"

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    #include "setRootCase.H"
    #include "createTime.H"

    #include "createMesh.H"

    #include "createFields.H"

    ++runTime;
    // Advance one time step to avoid overwriting the initial time step

    solve // Solve the governing equation
    (
	      fvm::ddt(A)
        + fvm::div(phi,A)
        - fvm::laplacian(gamma,A)
    );

    A.write(); // Write the computed physical field A


    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< nl;
    runTime.printExecutionTime(Info);

    Info<< "End\n" << endl;

    return 0;
}

```

The correspondence between the mathematical equation and the programming language can be seen:

$$\frac{\partial}{\partial t}A + \nabla\cdot(UA) - \nabla\cdot(\gamma\nabla A)=0$$

Corresponds to:

```cpp
	  fvm::ddt(A)
	+ fvm::div(phi,A)
	- fvm::laplacian(gamma,A)
```

### 3.1. Time Term

The time term is constructed using the function `ddt()`:

```cpp
fvm::ddt(A)
```

We can search for this function (or access the code directly via the `OFextension` plugin):

```terminal {fileName="terminal"}
find $FOAM_SRC -iname fvmddt.c
```

To aid understanding, a few excerpts from the code declaration in `fvmDdt.C` are discussed briefly below (it is not recommended to delve too deeply into the code at this stage, as it may hinder progress):

```cpp {fileName="$FOAM_SRC//finiteVolume/finiteVolume/fvm/fvmDdt.C",linenos=table,linenostart=1}
namespace Foam // Belongs to the Foam namespace
{

namespace fvm // Belongs to the fvm namespace
{

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

template<class Type>
tmp<fvMatrix<Type>> // Return type
ddt // Function name
(
    const GeometricField<Type, fvPatchField, volMesh>& vf // Formal parameter
    // Requires a GeometricField type; recall that GeometricField has different type definitions
)
{
    return fv::ddtScheme<Type>::New
    (
        vf.mesh(),
        vf.mesh().ddtScheme("ddt(" + vf.name() + ')')
    ).ref().fvmDdt(vf);
}
...
```

We now understand the formal parameter type of this function and that it returns a variable of type `fvMatrix`, which mathematically corresponds to the matrix associated with the unknown quantity A

### 3.2. Convection Term

The convection term is constructed using the function `div()`:

```cpp
fvm::div(phi,A)
```

We can search for this function (or access the code directly via the `OFextension` plugin):

```terminal {fileName="terminal"}
find $FOAM_SRC -iname fvmdiv.C
```

To aid understanding, a few excerpts from the code declaration in `fvmDiv.C` are discussed briefly below:

```cpp {fileName="$FOAM_SRC/finiteVolume/finiteVolume/fvm/fvmDiv.C",linenos=table,linenostart=1}
...

// Corresponds to fvm::div(phi,A)

template<class Type>
tmp<fvMatrix<Type>> // Return type
div // Function name
(
    const surfaceScalarField& flux, // Formal parameter 1: reference to surfaceScalarField type
    const GeometricField<Type, fvPatchField, volMesh>& vf
    // Formal parameter 2: GeometricField type
)
{
    return fvm::div(flux, vf, "div("+flux.name()+','+vf.name()+')');
}
...

```

We now understand the formal parameter types of this function and that it returns a variable of type `fvMatrix`, which mathematically corresponds to the matrix associated with the unknown quantity A.

### 3.3. Diffusion Term

The diffusion term is constructed using the function `laplacian()`.

We can search for this function (or access the code directly via the `OFextension` plugin; this will not be repeated):

```terminal {fileName="terminal"}
find $FOAM_SRC -iname fvlaplacian.C
```

Similarly, a portion of the code is excerpted as follows:

```cpp {fileName="$FOAM_SRC/finiteVolume/finiteVolume/fvm/fvmLaplacian.C",linenos=table,linenostart=1}
...

// Corresponds to fvm::laplacian(gamma,A)

template<class Type, class GType>
tmp<fvMatrix<Type>> // Return type
laplacian // Function name
(
    const dimensioned<GType>& gamma, // Formal parameter 1: reference to dimensioned type
    // dimensioned has different type definitions
    const GeometricField<Type, fvPatchField, volMesh>& vf
    // Formal parameter 2: reference to GeometricField type
)
{
    const GeometricField<GType, fvsPatchField, surfaceMesh> Gamma
    (
        IOobject
        (
            gamma.name(),
            vf.instance(),
            vf.mesh(),
            IOobject::NO_READ
        ),
        vf.mesh(),
        gamma
    );

    return fvm::laplacian(Gamma, vf);
}
...
```

Again, we understand the formal parameter types of this function and that it returns a variable of type `fvMatrix`, which mathematically corresponds to the matrix associated with the unknown quantity A.

### 3.4. Equation Solution

The final discretized form of the equation is:

$$
Ax = b
$$

The matrices from the convection and diffusion terms are added together to form the left-hand side. Since there is no source term, the right-hand side is empty and can be omitted.

The convection and diffusion terms are combined and solved using the `solve()` function:

```cpp
    solve // Solve the governing equation
    (
          fvm::ddt(A)
        + fvm::div(phi,A)
        - fvm::laplacian(gamma,A)
    );
```

The implementation details of the `solve()` function will not be explored here.

## 4. Adjusting the Test Case

Since the project now includes computation of the unknown field A, the test case needs to be adjusted accordingly.

### 4.1. Initial Field A

Provide the initial field A for the project as follows:

```cpp {fileName="debug_case/0.org/A",linenos=table,linenostart=1}
FoamFile
{
    version     2.0;
    format      ascii;
    class       volScalarField;
    object      A;
}
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

dimensions      [0 0 0 0 0 0 0];

internalField   uniform 500;

boundaryField
{
    movingWall
    {
        type            fixedValue;
        value           uniform 100;
    }

    fixedWalls
    {
        type            fixedValue;
        value           uniform 0;
    }

    frontAndBack
    {
        type            empty;
    }
}

```

### 4.2. transportProperties

Add the new diffusion coefficient to the `transportProperties` dictionary:

```cpp {fileName="debug_case/constant/transportProperties"}
...
gamma           1.0;
...
```

### 4.3. fvSchemes

Specify discretization schemes for the numerical terms, ensuring the following specifications are present:

```cpp {fileName="debug_case/system/fvSchemes",linenos=table,linenostart=1}
...

ddtSchemes
{
    default         Euler;
    ddt(A)          Euler; // If no default is specified
}

...

divSchemes
{
    default         none;
    div(phi,A)      Gauss linear; // If no default is specified
}

laplacianSchemes
{
    default         Gauss linear orthogonal;
    laplacian(gamma,A)  Gauss linear orthogonal; // If no default is specified
}

...
```

The underlying mathematical physics of these specifications will not be explored here.

### 4.4. fvSolution

Specify the algebraic solver for the linear system, ensuring the following specifications are present:

```cpp {fileName="debug_case/system/fvSolution",linenos=table,linenostart=1}
...
solvers
{
	...    
    A
    {
        solver          GAMG;
        smoother        symGaussSeidel;
        tolerance       1e-06;
        relTol          0.05;
    }
}
...
```

The underlying mathematical physics of these specifications will not be explored here.

## 5. Compilation and Execution

Compile and run this project:

```terminal {fileName="terminal"}
wclean
wmake
./casepre & ./caserun & ./casepost
```

Through the computation, we obtain the result for a single time step. Visualize the results using ParaView, and examine the distribution of the A field at that time step.

## 6. Complete Problem

After gaining a basic understanding of how OpenFOAM constructs equations, we will now consider the complete problem described above.

### 6.1. Problem Description

![[image.png]]

In a square container identical to the cavity test case, the velocity field has a velocity of $U_{x}=1m/s$ at the top boundary, while the other boundaries are fixed (no normal velocity exchange). For the dimensionless A field, the internal domain has a value of 500, the top boundary has a value of 100, and the other boundaries have a value of 0. Consider the transient, incompressible, source-free transport problem described by the following mathematical-physical equation:

$$\frac{\partial}{\partial t}A + \nabla\cdot(UA) - \nabla\cdot(\gamma\nabla A)=0$$

### 6.2. Solver

The main source code of the solver is as follows:

```cpp {fileName="ofsp_19_firstSol.C",linenos=table,linenostart=1}
#include "fvCFD.H"

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    #include "setRootCase.H"
    #include "createTime.H"

    #include "createMesh.H"

    #include "createFields.H"

    while(runTime.loop())
    {
        Info<< "Time = " << runTime.timeName() << nl << endl;

        // Solve the transport equation
        fvScalarMatrix AEqn
        (
            fvm::ddt(A)
          + fvm::div(phi, A)
          - fvm::laplacian(gamma, A)
        );

        AEqn.solve();

        runTime.write();

        runTime.printExecutionTime(Info);
    }

    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< nl;

    Info<< "End\n" << endl;

    return 0;
}

```

### 6.3. Test Case

We continue to use the test case files from the previous section (including initial conditions, boundary conditions, mesh dictionary, and transport properties).

However, to observe significant changes in the visualized results, adjust the control dictionary:

```cpp {fileName="debug_case/system/controlDict",linenos=table,linenostart=1}
...

endTime         0.01;

deltaT          0.001;

writeControl    timeStep;

writeInterval   1;

...
```

### 6.4. Compilation and Execution

Recompile and run this project:

```terminal {fileName="terminal"}
wclean
wmake
./caseclean & ./casepre & ./caserun & ./casepost
```

Through the computation, we obtain results that evolve over time. Visualize the results using ParaView, and examine how the A field distribution changes over time.

## 7. Summary

Some readers may notice that many questions remain unanswered. For example:

- Why is the flux `phi` used in the divergence calculation instead of `U` as in the equation?

In brief, if one has studied the finite volume method, one understands that in FVM, the conserved quantity is the flux, which is the core and advantage of FVM.

- Why is `fvm` used? What is the difference from the commonly seen `fvc`?

In simple terms, `fvm` in solvers discretizes all terms that contribute to the coefficient matrix (implicit discretization). In contrast, `fvc` discretizes terms that are not associated with the coefficient matrix (explicit discretization).

However, such a brief explanation is often difficult for beginners to fully grasp. To address these two questions, we would need to return to the numerical solution of the partial differential equation governing the problem, starting from the finite volume method and deriving step by step until matrix solution, only then can we truly understand why flux is used in computation and why there is a distinction between implicit and explicit discretization. These topics will not be explored here; refer to the CFD Fundamentals series (cfdb) and the OpenFOAM Solver Series (ofss) for further discussion.

Fortunately, the syntax of the solver code is already very close to the mathematical formulation, so even with these lingering questions, we can still write simple solvers.

This section has completed the following discussions:

- [ ] Refining script writing
- [ ] Briefly understanding the correspondence between equation construction and code
- [ ] Understanding the simple workflow of numerical computation
- [ ] Compiling and running the firstSol project


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


