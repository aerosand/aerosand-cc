---
uid: 20260321180844
title: 25_pimpleSol
date: 2026-03-21
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
weight: 26
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

In the previous discussion, we examined the PIMPLE algorithm in detail and briefly looked at some code snippets of the PIMPLE algorithm framework in OpenFOAM.

Here, we will implement the code based on our discussion of the algorithm and understand the usage of some code.

This section primarily discusses:

- [ ] Implementation of the PIMPLE algorithm
- [ ] Computing the cavity test case


## 1. Governing Equations

The governing equations are as follows:

Continuity equation (mass equation):

$$
\nabla\cdot U = 0
$$

Momentum equation:

$$\frac{\partial U}{\partial t} + \nabla \cdot (UU) = \nabla\cdot(\nu\nabla U)-\nabla p$$

The following assumptions are still applied:

- Viscous term is simplified
- Gravity is neglected
- Density has been accounted for

Note that the momentum equation includes a transient term.


## 2. Project Preparation

Run the following commands in the terminal to create the project:

```terminal {fileName="terminal"}
ofsp
foamNewApp ofsp_25_pimpleSol
cd ofsp_25_pimpleSol
cp -r $FOAM_TUTORIALS/incompressible/icoFoam/cavity/cavity debug_case
code .
```

### 2.1. Documentation File

Provide a documentation file for the project:

```markdown {fileName="README.md"}
## About

A solver that simply reproduces the PIMPLE algorithm.

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

@20260313

- Added scripts #done

```

### 2.2. Script Files

We continue to use the previous scripts. Unless modified in the future, they will not be elaborated further.

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
    echo "Aerosand >>> Initial condition done."
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

## 3. Project Implementation

We will implement the PIMPLE algorithm with the simplest code possible.

### 3.1. Main Source Code

Considering the discussion in `24_pimple`, implement the main framework of the PIMPLE algorithm in the main source code:

```cpp {fileName="ofsp_25_pimpleSol.C",linenos=table,linenostart=1}
#include "fvCFD.H"

#include "pimpleControl.H" // Header file containing the PIMPLE algorithm control
// This header file belongs to the finiteVolume library; no additional linking is needed in Make

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    argList::addNote
    (
        "Aerosand: a test solver for PIMPLE algorithm."
    );

    #include "postProcess.H"

    #include "addCheckCaseOptions.H"
    #include "setRootCaseLists.H"
    #include "createTime.H"

    #include "createMesh.H"

    #include "createControl.H"

    #include "createFields.H"

    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< "\nStarting time loop\n" << endl;

    while (runTime.loop()) // Time advancement
    {

        Info<< "Time = " << runTime.timeName() << nl << endl;

        // --- Pressure-velocity PIMPLE corrector loop
        while (pimple.loop()) // PIMPLE outer loop
        {

            #include "UEqn.H"

            // --- Pressure corrector loop
            while (pimple.correct()) // PIMPLE inner loop
            {
                #include "pEqn.H"
            }

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

The field inclusion `createFields.H` is the same as in the previous solvers, as follows:

```cpp {fileName="createFields.H",linenos=table,linenostart=1}
/*
    # Basic fields
    # transportProperties
    # Pressure reference
*/


// # Basic fields

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

#include "createPhi.H"


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
setRefCell(p, piso.dict(), pRefCell, pRefValue);

```

### 3.3. Momentum Predictor

The momentum predictor is in `UEqn.H`, with the following code:

```cpp {fileName="UEqn.H",base_url="https://aerosand.cc",linenos=table,linenostart=1}
// Momentum predictor

tmp<fvVectorMatrix> tUEqn // Use OpenFOAM's temporary object class tmp to construct the equation matrix for convenient memory management
(
    fvm::ddt(U) 
  + fvm::div(phi, U)
  - fvm::laplacian(nu, U) // Diffusion term
);
fvVectorMatrix& UEqn = tUEqn.ref();

UEqn.relax();

solve(UEqn == -fvc::grad(p)); // Fully solve the momentum equation

```


### 3.4. Pressure-Momentum Correction

The pressure-momentum correction is in `pEqn.H`, with the following code:

```cpp {fileName="pEqn.H",base_url="https://aerosand.cc",linenos=table,linenostart=1}
volScalarField rAU(1.0/UEqn.A()); // A^-1
volVectorField HbyA(rAU*UEqn.H()); // HbyA
surfaceScalarField phiHbyA("phiHbyA", fvc::flux(HbyA)); // phiHbyA

tmp<volScalarField> rAtU(rAU); // Temporary object for convenient memory management

fvScalarMatrix pEqn // Pressure correction equation
(
    fvm::laplacian(rAtU(), p) == fvc::div(phiHbyA)
);

pEqn.setReference(pRefCell, pRefValue); // Set the pressure reference

pEqn.solve(); // Solve the pressure correction equation

p.relax(); // Field relaxation, to be discussed later

// Momentum corrector

U = HbyA - rAtU*fvc::grad(p); // Momentum correction

```


> [!tip]
> Note that for ease of understanding, non-orthogonal correction is not performed here. Since the cavity test case used later has a simple mesh, this has no impact.

### 3.5. Project Make

As discussed above, this solver does not use additional libraries, so no extra linking specifications are needed in the project Make.

The `Make/files` content is as follows:

```cpp {fileName="Make/files",base_url="https://aerosand.cc",linenos=table,linenostart=1}
ofsp_23_pisoSol.C

EXE = $(FOAM_USER_APPBIN)/ofsp_23_pisoSol

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
application     ofsp_25_pimpleSol;
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
    
	"(U|k|epsilon)Final"
    {
        $U;
        relTol          0;
    }
    // Because PIMPLE has an outer loop, the final momentum predictor needs to verify velocity convergence
}

PIMPLE // Control dictionary for the PIMPLE algorithm
{
    // Basic loop control
    nOuterCorrectors         2;          // Number of outer loop iterations (PIMPLE-specific)
    nCorrectors              2;          // Number of inner loop (PISO corrector) iterations
    nNonOrthogonalCorrectors 0;          // Number of non-orthogonal correction iterations
    
    // Residual control (for early exit from outer loop)
    residualControl
    {
        p
        {
            tolerance       1e-4;        // Absolute residual threshold
            relTol          0;           // Relative residual threshold (0 means disabled)
        }
        U
        {
            tolerance       1e-4;
            relTol          0;
        }
    }
    
    // Pressure reference
    pRefCell                0;           // Mesh cell index where pressure reference is set
    pRefValue               0;           // Pressure reference value (usually 0)
}

// Relaxation factors (usually outside the PIMPLE dictionary)
relaxationFactors
{
    fields
    {
        p                   0.3;         // Relaxation factor for the pressure field
    }
    equations
    {
        U                   0.7;         // Relaxation factor for the momentum equation
        ".*"                1;           // No relaxation for other equations (matched by regular expression)
    }
}

```

Other files remain unchanged.

### 3.6. Compilation and Execution

Compile the project:

```terminal {fileName="terminal"}
wclean
wmake
```

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

## 4. Summary

Through the implementation and discussion of the solver project, I believe we now have a relatively comprehensive understanding of the PIMPLE algorithm and solver implementation.

Looking back at these algorithm discussions, we have roughly clarified the main ideas of various algorithms and their simplified code implementations. The idea of pressure-velocity coupling runs through almost all types of computational fluid dynamics solvers in OpenFOAM. Many cases are extensions based on these core algorithms.

As readers adjust the code themselves and compare it with native solvers, they will find many code statements that have been streamlined in our discussion.

On one hand, to facilitate understanding of the main algorithm ideas, we avoided involving other details. Do not worry; we will discuss and explain them all in the future. On the other hand, we used the cavity test case as a debugging case, which is relatively simple. Even without "boundary condition constraints," "consistency checks," etc., it can still produce usable results.

This section has completed the following discussions:

- [x] Implementation of the PIMPLE algorithm
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


