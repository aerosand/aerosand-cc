---
uid: 20260128101852
title: 21_simpleSol
date: 2026-01-28
update: 2026-04-08
authors:
  - name: Aerosand
    link: https://github.com/aerosand
    image: https://github.com/aerosand.png
tags:
  - OpenFOAM
  - ofsp
  - ofsp2026
excludeSearch: false
toc: true
weight: 22
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

In the previous discussion, we examined the SIMPLE algorithm in detail and briefly looked at some code snippets of the SIMPLE algorithm in OpenFOAM.

Here, we will implement the code based on our discussion of the algorithm and understand the usage of some code.

This section primarily discusses:

- [ ] Implementation of the SIMPLE algorithm
- [ ] Computing the cavity test case

## 1. Governing Equations

The governing equations are as follows:

Continuity equation (mass equation):

$$
\nabla\cdot U = 0
$$

Momentum equation:

$$\nabla \cdot (UU) = \nabla\cdot(\nu\nabla U)-\nabla p$$

The following assumptions are still applied:

- Steady state
- Viscous term is simplified
- Gravity is neglected
- Density has been accounted for

## 2. Project Preparation

Run the following commands in the terminal to create the project:

```terminal {fileName="terminal"}
ofsp
foamNewApp ofsp_21_simpleSol
cd ofsp_21_simpleSol
cp -r $FOAM_TUTORIALS/incompressible/icoFoam/cavity/cavity debug_case
code .
```

### 2.1. Documentation File

Provide a documentation file for the project:

```markdown {fileName="README.md"}
## About

A solver that simply reproduces the SIMPLE algorithm.

## Bio

aerosand @ aerosand

## Caution

Pay attention to the OpenFOAM version.

## Deploy

Ensure the solver files are complete and include the debug_case folder

Enter the following commands in the terminal:

./Allclean
./Allrun


## Event

@20260312

- Added scripts #done

```

### 2.2. Script Files

We will streamline the previous scripts.

Run the following command in the terminal to create the scripts:

```terminal {fileName="terminal"}
code {Allclean,Allrun}
```

The cleaning script `Allclean` is as follows:

```bash {fileName="Allclean",linenos=table,linenostart=1}
#!/bin/sh
cd "${0%/*}" || exit                                # Run from this directory
. ${WM_PROJECT_DIR:?}/bin/tools/RunFunctions        # Tutorial run functions
#------------------------------------------------------------------------------

cd debug_case

# Function: initialize the 0 folder
init_0() {
    if [ -d "0.orig" ] && [ -d "0" ]; then
        echo "Both 0 and 0.orig exist. Removing 0 and restoring from 0.orig..."
        rm -rf 0
        cp -a 0.orig 0
    elif [ -d "0" ] && [ ! -d "0.orig" ]; then
        echo "Backing up 0 to 0.orig..."
        cp -a 0 0.orig
    elif [ -d "0.orig" ] && [ ! -d "0" ]; then
        echo "Restoring 0 from 0.orig...".
        cp -a 0.orig 0
    else
        echo "Warning: Neither 0 nor 0.orig exists. Skipping."
    fi
    echo ">>>>>>>>>>>>> Initial condition done."
}

# Ensure initial conditions
init_0

# Clean the case
. ${WM_PROJECT_DIR:?}/bin/tools/CleanFunctions      # Tutorial clean functions
cleanCase0

```

The run script `Allrun` is as follows:

```cpp {fileName="Allrun",linenos=table,linenostart=1}
#!/bin/sh
cd "${0%/*}" || exit                                # Run from this directory
. ${WM_PROJECT_DIR:?}/bin/tools/RunFunctions        # Tutorial run functions
#------------------------------------------------------------------------------

cd debug_case

restore0Dir

runApplication blockMesh

runApplication $(getApplication)

```

Do not worry about the scripts; as the discussion deepens, we will continuously expand the script content and try different writing styles.

> [!note]
> Use `chmod +x {Allclean,Allrun}` to grant execution permissions to the scripts.
> 
> These scripts can continue to be used in the future unless otherwise noted.

## 3. Project Implementation

We will implement the SIMPLE algorithm with the simplest code possible.

### 3.1. Main Source Code

Considering the discussion in `20_simple`, implement the main framework of the SIMPLE algorithm in the main source code:

```cpp {fileName="ofsp_21_simpleSol.C",linenos=table,linenostart=1}
#include "fvCFD.H" // Refer to 07_firstApp, will not be repeated

#include "simpleControl.H" // Header file containing the SIMPLE algorithm control
// This header file belongs to the finiteVolume library; no additional linking is needed in Make


// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    #include "setRootCase.H" // Refer to 13_commandLine, will not be repeated
    #include "createTime.H" // Refer to 14_time, will not be repeated

    #include "createMesh.H" // Refer to 15_mesh, will not be repeated
    #include "createControl.H" // Create algorithm control; can be automatically created based on usage
    #include "createFields.H" // User-provided field inclusion


    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< "\nStarting time loop\n" << endl;

    while (simple.loop()) // SIMPLE algorithm loop
    {
        Info<< "Time = " << runTime.timeName() << nl << endl;

        // --- Pressure-velocity SIMPLE corrector
        {
            #include "UEqn.H" // Momentum predictor
            #include "pEqn.H" // Pressure and momentum correction
        }

        runTime.write();

        runTime.printExecutionTime(Info);
    }

    Info<< "End\n" << endl;

    return 0;
}

```

It can be seen that the main algorithm framework is the same as discussed in the previous section.

### 3.2. Field Inclusion

The field inclusion `createFields.H` is as follows:

```cpp {fileName="createFields.H",linenos=table,linenostart=1}
/*
    # Basic fields
    # transportProperties
    # Pressure reference
*/

// # Basic fields

Info<< "Reading field p\n" << endl; // Pressure field
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

#include "createPhi.H" // Mass flux field

// # transportProperties

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

dimensionedScalar nu
(
    "nu",
    dimViscosity,
    transportProperties
);


// # Pressure reference

label pRefCell = 0;
scalar pRefValue = 0.0;
setRefCell(p, simple.dict(), pRefCell, pRefValue);

```

### 3.3. Momentum Predictor

The momentum predictor is in `UEqn.H`, with the following code:

```cpp {fileName="UEqn.H",base_url="https://aerosand.cc",linenos=table,linenostart=1}
// Momentum predictor

  fvVectorMatrix UEqn
  (
      fvm::div(phi, U)
    - fvm::laplacian(nu, U)
  );

  UEqn.relax(); // Equation relaxation, to be discussed later

  solve(UEqn == -fvc::grad(p));

```

Regarding the relaxation of the momentum equation, this will be discussed later. Readers can comment out this relaxation code and compare the differences in calculation results.

### 3.4. Pressure and Momentum Correction

The pressure and momentum correction is in `pEqn.H`, with the following code:

```cpp {fileName="pEqn.H",base_url="https://aerosand.cc",linenos=table,linenostart=1}
{
    volScalarField rAU(1.0/UEqn.A()); // A^-1 
    volVectorField HbyA(rAU*UEqn.H()); // HbyA
    surfaceScalarField phiHbyA("phiHbyA", fvc::flux(HbyA)); // phiHbyA

    fvScalarMatrix pEqn
    (
        fvm::laplacian(rAU, p) == fvc::div(phiHbyA)
    );
	// Pressure correction equation
	// Similarly, fvm discretization returns a matrix, with the discretization operation targeting the parameter p, which is the unknown quantity to be solved
	// fvc discretization returns a field, which serves as a known quantity in the equation


    pEqn.setReference(pRefCell, pRefValue); // Set the pressure reference

    pEqn.solve(); // Solve the pressure correction equation

    // Explicitly relax pressure for momentum corrector
    p.relax(); // Field relaxation, to be discussed later

    // Momentum corrector
    U = HbyA - rAU*fvc::grad(p); // Solve for the corrected velocity
    
    // The corrected pressure and velocity participate in the next SIMPLE loop iteration
}

```

Pressure also undergoes relaxation, which will be discussed later. Readers can try commenting it out to observe the differences in calculation results.

> [!tip]
> Note that for ease of understanding, non-orthogonal correction is not performed here. Since the cavity test case used later has a simple mesh, this has no impact.

### 3.5. Project Make

As discussed above, this solver does not use additional libraries, so no extra linking specifications are needed in the project Make.

The `Make/files` content is as follows:

```cpp {fileName="Make/files",base_url="https://aerosand.cc",linenos=table,linenostart=1}
ofsp_21_simpleSol.C

EXE = $(FOAM_USER_APPBIN)/ofsp_21_simpleSol

```

The `Make/options` content is as follows:

```cpp {fileName="Make/options",base_url="https://aerosand.cc",linenos=table,linenostart=1}
EXE_INC = \
    -I$(LIB_SRC)/finiteVolume/lnInclude \
    -I$(LIB_SRC)/meshTools/lnInclude

EXE_LIBS = \
    -lfiniteVolume \
    -lmeshTools

```

### 3.6. Compilation

Run the following command in the terminal to compile the entire project:

```terminal {fileName="terminal"}
wclean
wmake
```

Compilation is successful with no issues.

### 3.7. Test Case

We adjust the copied test case to test the above solver.

Modify the control dictionary `controlDict` as follows:

```cpp {fileName="controlDict",base_url="https://aerosand.cc",linenos=table,linenostart=1}
...
application     ofsp_21_simpleSol;
...
endTime         1.0; // Extend the time to compare calculation effects with and without relaxation
// If relaxation is not performed, the calculation may diverge and terminate after some time
...
```

Modify the solution dictionary `fvSolution` as follows:

```cpp {fileName="fvSolution",base_url="https://aerosand.cc",linenos=table,linenostart=1}
solvers
{
    p
    {
        solver          PCG;
        preconditioner  DIC;
        tolerance       1e-06;
        relTol          0.05;
    }

    pFinal
    {
        $p;
        relTol          0;
    }

    U
    {
        solver          smoothSolver;
        smoother        symGaussSeidel;
        tolerance       1e-05;
        relTol          0;
    }
}


SIMPLE // Specify the dictionary parameters for the SIMPLE algorithm
{
    nNonOrthogonalCorrectors 0;
    consistent      yes;
    pRefCell        0;
    pRefValue       0;

    residualControl
    {
        p               1e-2;
        U               1e-3;
    }
}

relaxationFactors // Specify the dictionary parameters for relaxation
{
    fields // Field relaxation
    {
        p               0.3;   // Pressure field relaxation factor 0.3
        ".*"       0.7;   // Use regular expression to match all p_rgh variants
    }
    
    equations // Equation relaxation
    {
        U               0.9; // 0.9 is more stable but 0.95 more convergent
        ".*"            0.9; // 0.9 is more stable but 0.95 more convergent
    }
}

```

Other files remain unchanged.

### 3.7. Execution

Since the solver project compiles without issues, it can be run directly using the scripts.

Run the project:

```terminal {fileName="terminal"}
./Allclean
./Allrun
```

Post-process visualization:

```terminal {fileName="terminal"}
paraFoam -case debug_case
```

The calculation results can be viewed in ParaView.

Readers can modify parts of the code to explore the effects of different statements.

## 4. Summary

Through the implementation and discussion of the solver project, I believe we now have a relatively comprehensive understanding of the SIMPLE algorithm and solver implementation.

In the next section, we will discuss the PISO algorithm.

This section has completed the following discussions:

- [x] Implementation of the SIMPLE algorithm
- [x] Computing the cavity test case


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


