---
uid: 20251117135613
title: 17_field
date: 2025-11-17
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
weight: 18
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

Recalling the previous discussions, we have covered input/output, command-line arguments, time, and meshes. Most computations in OpenFOAM are based on physical fields. This project begins the discussion of "fields" in OpenFOAM.

This section primarily discusses:

- [ ] Understanding the code structure of fields
- [ ] Understanding how fields participate in computations
- [ ] Compiling and running a field project

## 1. Project Preparation

Run the following commands in the terminal to create the project:

```terminal {fileName="terminal"}
ofsp
foamNewApp ofsp_17_field
cd ofsp_17_field
cp -r $FOAM_TUTORIALS/incompressible/icoFoam/cavity/cavity debug_case
code .
```

Test the initial solver, and provide scripts and documentation.

## 2. Field Creation

### 2.1. dimensionSet

We previously encountered OpenFOAM's predefined units, which are defined in `$FOAM_SRC/OpenFOAM/dimensionSet/dimensionSets.C`. This file also defines many predefined unit combinations.

Excerpts are shown again below:

```cpp {fileName="dimensionSets.C",linenos=table,linenostart=1}
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
```

Again, we will not delve into the implementation details for now.

We can set custom physical units for variables, for example:

```cpp
dimensionSet dimAerosand(1,2,3,4,5,6,7); // just for example
```

### 2.2. dimensionedType

Find the class definition:

API page: [https://api.openfoam.com/2506/dimensionedType_8H.html](https://api.openfoam.com/2506/dimensionedType_8H.html)

Run the following command in the terminal to find the source code:

```terminal {fileName="terminal"}
find $FOAM_SRC -iname dimensionedType.H
```

Locate and excerpt the constructors as follows:

```cpp {fileName="dimensionedType.H",linenos=table,linenostart=1}
template<class Type>
class dimensioned
{
...
public:
...
	// Constructors
		...
		//- Construct from components (name, dimensions, value).
        dimensioned
        (
            const word& name,
            const dimensionSet& dims,
            const Type& val
        );
	    ...
	...
...

```

Correspondingly, similar constructions can be seen in OpenFOAM code, such as:

```cpp {linenos=table,linenostart=1}
dimensionedScalar airPressure
(
	"airPressure",
	dimensionSet(1, -1, -2, 0, 0, 0, 0),
	1.0
);

```

In `dimensionedType.H`, associated member functions are also defined, for example:

```cpp {fileName="dimensionedType.H",linenos=table,linenostart=1}
...
	//- Return const reference to value.
	const Type& value() const noexcept { return value_; }

	//- Return non-const reference to value.
	Type& value() noexcept { return value_; }
...
```

Correspondingly, we can conveniently use the `value()` member function, for example, to obtain the value of `airPressure` above and then compute the maximum:

```cpp
maxP = max(airPressure.value());
```

Of course, OpenFOAM also provides other constructors and member functions, allowing us to construct our own physical quantities with units in different scenarios. At the current stage, we do not need to delve deeper into the source code. It is sufficient to understand that this is indeed how it works and that we will use it in this way in the future.

Note that OpenFOAM checks whether the final units on both sides of a mathematical equation are the same; if they differ, it will force an error and terminate the program. This issue will be encountered and addressed in subsequent code.

### 2.3. volFields

We will notice typical statements in OpenFOAM, such as:

```cpp {linenos=table,linenostart=1}
...
    Info<< "Reading fieldp\\n" << endl;
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
...

```

So, what is `volScalarField`? How is a field constructed?

API page: [https://api.openfoam.com/2512/volFieldsFwd_8H.html](https://api.openfoam.com/2512/volFieldsFwd_8H.html)

Run the following command in the terminal to view the source file:

```terminal {fileName="terminal"}
find $FOAM_SRC -iname volFieldsFwd.H
```

> [!tip]
> Some students may wonder, "How do I know which file to look at?" Referring to the discussion in Section 06_tensor, after installing the OFextension plugin, you can simply right-click on the `volScalarField` keyword and jump to it. Alternatively, you can search for the keyword on the API page and read the relevant page.

The `volFieldsFwd.H` file contains type definitions as follows:

```cpp {fileName="volFieldsFwd.H"}
...
typedef GeometricField<scalar, fvPatchField, volMesh> volScalarField;
typedef GeometricField<vector, fvPatchField, volMesh> volVectorField;
...
```

It can be seen that `volFields` include the common `volScalarField` and `volVectorField`, which are volume field quantities. Both `volScalarField` and `volVectorField` are actually different specializations of the `GeometricField` template. Therefore, `volFields` is just a collective term for a group of template specializations, not an actual class.

### 2.4. surfaceFields

Similarly, surface field quantities `surfaceFields` include the common `surfaceScalarField` and `surfaceVectorField`.

API page: [https://api.openfoam.com/2506/surfaceFieldsFwd_8H.html](https://api.openfoam.com/2506/surfaceFieldsFwd_8H.html)

Run the following command in the terminal to view the source file:

```terminal {fileName="terminal"}
find $FOAM_SRC -iname surfaceFieldsFwd.H
```

The `surfaceFieldsFwd.H` file contains type definitions as follows:

```cpp {fileName="surfaceFieldsFwd.H"}
...
typedef GeometricField<scalar, fvsPatchField, surfaceMesh> surfaceScalarField;
typedef GeometricField<vector, fvsPatchField, surfaceMesh> surfaceVectorField;
...
```

Thus, `surfaceFields` are also different specializations of the `GeometricField` template.

>[!tip]
>Why is "volume" abbreviated to "vol" in `volFields` but "surface" is written in full in `surfaceFields`? Perhaps it is due to historical reasons in the code...

### 2.5. GeometricField

Based on the discussion above, we examine the `GeometricField.H` code.

API page: [https://api.openfoam.com/2506/GeometricField_8H.html](https://api.openfoam.com/2506/GeometricField_8H.html)

Run the following command in the terminal to view the source file:

```terminal {fileName="terminal"}
find $FOAM_SRC -iname GeometricField.H
```

Excerpt the main structure and several common constructors as follows:

```cpp {fileName="GeometricField.H",linenos=table,linenostart=1}
...
template<class Type, template<class> class PatchField, class GeoMesh>
class GeometricField
:
    public DimensionedField<Type, GeoMesh>
{
public:
	...
private:
	...
public:
	...
	// Constructors

	//- Construct given IOobject, mesh, dimensioned<Type> and patch type.
	//  This assigns both dimensions and values.
	//  The name of the dimensioned\\<Type\\> has no influence.
	GeometricField
	(
		const IOobject& io,
		const Mesh& mesh,
		const dimensioned<Type>& dt,
		const word& patchFieldType = PatchField<Type>::calculatedType()
	);
	...
	//- Construct and read given IOobject
	GeometricField
	(
		const IOobject& io,
		const Mesh& mesh,
		const bool readOldTime = true
	);
	...
	//- Construct as copy resetting IO parameters
	GeometricField
	(
		const IOobject& io,
		const GeometricField<Type, PatchField, GeoMesh>& gf
	);
	...
...

```

The above constructors correspond to actual field constructions, with examples as follows:

```cpp {linenos=table,linenostart=1}
...
	Info<< "Creating field light pressure\n" << endl;
	volScalarField lightP
	(
		IOobject
		(
			"lightP",
			runTime.timeName(),
			mesh,
			IOobject::NO_READ,
			IOobject::AUTO_WRITE
		),
		mesh,
		dimensionedScalar("lightP",dimPressure,0.0)
		// Initialized with a constructed dimensioned<Type>& type variable
		// Based on name, units, initial value
		// find $FOAM_SRC -iname dimensionedType.H
	);
...
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
...
	Info<< "Calculating field mixture density\n" << endl;
	volScalarField rho
	(
		IOobject
		(
			"rho",
			runTime.timeName(),
			mesh,
			IOobject::NO_READ,
			IOobject::AUTO_WRITE
		),
		alpha1*rho1 + alpha2*rho2
		// Passes the computed field
	);
...
```

These three examples correspond to the most common field constructors. Other constructors and more in-depth code-level analysis will be introduced gradually in the future, so there is no need to worry. Readers may also try to read a few more on their own.

## 3. Time Advancement of Fields

Taking the pressure field as an example, combined with field creation, we discuss the time advancement computation of fields.

### 3.1. Single Time Step Computation

The main source code `ofsp_17_field.C` is as follows:

```cpp {fileName="ofsp_17_field/ofsp_17_field.C",linenos=table,linenostart=1}
#include "fvCFD.H"

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    #include "setRootCase.H"
    #include "createTime.H"

    #include "createMesh.H"
    // The three common header files, which will not be repeated in the future

    volScalarField p // Create the pressure field p based on IOobject and mesh
    (
        IOobject // Create IOobject based on the following parameters
        (
            "p",
            runTime.timeName(),
            mesh,
            IOobject::MUST_READ,
            IOobject::AUTO_WRITE // Automatically write according to settings in the source code
        ),
        mesh
    );
    // Read options:
	//     MUST_READ: must read; if the file does not exist, an error is reported
	//     READ_IF_PRESENT: read only if present; if the file format is incorrect, an error is reported
	//     NO_READ: do not read
	// Write options:
	//     AUTO_WRITE: automatically write as required
	//     NO_WRITE: do not write, but can be explicitly written via code

    runTime++; // Advance by one time step
    // If this line is omitted, the time step is not advanced, and the computed results will be written to the initial time step, overwriting the initial field

    forAll(p, i) // Iterate over the pressure field
    {
        if (mesh.C()[i][1] < 0.5)
        // Remember what mesh.C() returns?
        // Returns the y-coordinate of the mesh cell center (C++ indexing starts from 0)
        {
            p[i] = i*runTime.value(); // Assign a possible computed value
        }
    }
    // The entire pressure field computation is completed

    Info<< "Max p = " << max(p).value() << endl; // Directly return the maximum value of the pressure field
    Info<< "Min p = " << min(p).value() << endl; // Return the minimum value of the pressure field
    Info<< "Average p = " << average(p).value() << endl; // Return the average value of the pressure field
    Info<< "Sum p = " << sum(p).value() << endl; // Return the sum of the pressure field

    p.write(); // Explicit code to write only the pressure field at this time step

    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< nl;
    runTime.printExecutionTime(Info);

    Info<< "End\n" << endl;

    return 0;
}

```


>[!tip]
>Note that the overwriting of the field in the main source code does not distinguish between the initial time step and subsequent time steps. Moreover, the `foamCleanTutorials` command cannot restore the content of the field files in the initial time step folder. Therefore, to avoid overwriting the initial conditions, it is highly recommended to back up the initial conditions. Once the initial field is overwritten, it can be restored at any time.


#### 3.1.1. Scripts

Modify the `caserun` script as follows:

```bash {fileName="caserun",linenos=table,linenostart=1}
#!/bin/sh
cd "${0%/*}" || exit 1                              # Run from this directory
#------------------------------------------------------------------------------

appPath=$(cd `dirname $0`; pwd)                     # Get application path
appName="${appPath##*/}"                            # Get application name

cd debug_case

FOLDER=$(pwd) 
FOLDER0=$FOLDER/0
FOLDER0_ORG=$FOLDER/0.org

echo "Setting up case directories..."

# 确保 0.org 存在
if ! test -d "$FOLDER0_ORG"; then
    if test -d "$FOLDER0"; then
        echo "Creating 0.org from existing 0 folder"
        cp -r "$FOLDER0" "$FOLDER0_ORG"
    else
        echo "Error: Neither 0.org nor 0 folder exists!"
        echo "Please ensure at least one of these directories exists."
        exit 1
    fi
else
    echo "0.org folder exists"
fi

# 确保 0 存在且是 0.org 的精确副本
if test -d "$FOLDER0"; then
    echo "Removing existing 0 folder"
    rm -rf "$FOLDER0"
fi

echo "Creating 0 folder from 0.org"
cp -r "$FOLDER0_ORG" "$FOLDER0"

echo "Directory setup complete. Running application..."

blockMesh

# $appName > log.run # if run in the background to save time
$appName | tee log.run

```

Modify the `caseclean` script as follows:

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

The modified scripts are more generic. Unless otherwise specified, they can be directly copied and used in the future.

#### 3.1.2. Compilation and Execution

Run the following commands in the terminal to compile and run the project:

```terminal {fileName="terminal"}
wclean
wmake
./caseclean
./caserun
```

The terminal output is as follows:

```terminal {fileName="terminal"}
Create time

Create mesh for time = 0

Max p = 1.995
Min p = 0
Average p = 0.9975
Sum p = 399

ExecutionTime = 0.01 s  ClockTime = 0 s

End
```

Check the `debug_case/` folder; an additional folder `0.005/` appears, and the computed field is written to `0.005/p`. The final output corresponds to the field value at the latest time step.

### 3.2. Multi-Time Step Computation

How to advance computations over multiple time steps?

Modify the main source code as follows:

```cpp {fileName="ofsp_17_filed/ofsp_17_field.C",linenos=table,linenostart=1}
#include "fvCFD.H"

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    #include "setRootCase.H"
    #include "createTime.H"

    #include "createMesh.H"

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

    // runTime++;

    for (int i=0; i < 3; ++i) // Compute over 3 time steps
    {
        ++runTime; // Advance time

        forAll(p, i)
        {
            if (mesh.C()[i][1] < 0.5)
            {
                p[i] = i*runTime.value();
            }
        }
        p.write(); // Write at this time step
    }

    Info<< "Max p = " << max(p).value() << endl;
    Info<< "Min p = " << min(p).value() << endl;
    Info<< "Average p = " << average(p).value() << endl;
    Info<< "Sum p = " << sum(p).value() << endl;

    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< nl;
    runTime.printExecutionTime(Info);

    Info<< "End\n" << endl;

    return 0;
}

```

>[!tip]
>On writing `++runTime` instead of `runTime++`:
>- In principle, pre-increment `++a` is more efficient than post-increment `a++`
>	- Pre-increment `++a` increments directly and returns itself, avoiding potential copying
>	- Post-increment creates a temporary copy, increments, and returns the temporary copy
>- For built-in types (int, float, pointers, etc.), modern compilers generally optimize away the difference
>- For complex types (classes, iterators), pre-increment is more efficient

Compile and run the project.

The terminal output is as follows:

```terminal {fileName="terminal"}
Create time

Create mesh for time = 0

Max p = 5.985
Min p = 0
Average p = 2.9925
Sum p = 1197

ExecutionTime = 0.01 s  ClockTime = 0 s

End
```

Observe that three additional time step folders appear in the `debug_case/` folder. The file structure of the test case is now as follows:

```terminal {fileName="terminal"}
tree debug_case/
debug_case/
├── 0
│   ├── p
│   └── U
├── 0.005
│   └── p
├── 0.01
│   └── p
├── 0.015
│   └── p
├── 0.org
│   ├── p
│   └── U
├── constant
│   ├── polyMesh/...
│   └── transportProperties
├── log.run
└── system/...
```

Checking the pressure field values at each time step shows that the final output from the terminal corresponds to the computed value at the latest time step.

### 3.3. Dictionary-Controlled Computation

Even for multi-time step computations, we directly modify the number of time steps in the main source code, requiring recompilation of the entire application each time. For large applications, recompilation can be quite cumbersome.

In practice, we hope to control the computation through external files after the application is compiled, i.e., using OpenFOAM's dictionary file `controlDict` to control the computation.

Modify the main source code as follows:

```cpp {fileName="ofsp_17_field/ofsp_17_field.C",linenos=table,linenostart=1}
#include "fvCFD.H"

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    #include "setRootCase.H"
    #include "createTime.H"

    #include "createMesh.H"

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

    while(runTime.loop()) // Time advancement
    {
        forAll(p, i)
        {
            if (mesh.C()[i][1] < 0.5)
            {
                p[i] = i*runTime.value();
            }
        }
		runTime.write(); // Write as required by the dictionary
    }

    Info<< "Max p = " << max(p).value() << endl;
    Info<< "Min p = " << min(p).value() << endl;
    Info<< "Average p = " << average(p).value() << endl;
    Info<< "Sum p = " << sum(p).value() << endl;

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

Max p = 199.5
Min p = 0
Average p = 99.75
Sum p = 39900

ExecutionTime = 0 s  ClockTime = 0 s

End
```

Several additional time step folders appear in the `debug_case/` folder: `0.1`, `0.2`, `0.3`, `0.4`, `0.5`. The time steps at which data are written are controlled by `system/controlDict`.

Partial parameters of the `controlDict` dictionary are as follows (writing occurs every `0.005 * 20 = 0.1`):

```cpp {fileName="debug_case/system/controlDict",linenos=table,linenostart=1}
...
endTime         0.5;

deltaT          0.005;

writeControl    timeStep;

writeInterval   20;
...
```

These parameters can be modified in the dictionary at any time for testing without recompiling the project.

## 4. Summary

This section introduced "fields" in OpenFOAM. Through the code and project exercises, readers can understand how fields participate in computational projects.

This section has completed the following discussions:

- [x] Understanding the code structure of fields
- [x] Understanding how fields participate in computations
- [x] Compiling and running a field project


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


