---
uid: 20250916203141
title: 11_dictionary
date: 2025-09-16
update: 2026-03-27
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
weight: 12
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

The previous discussions have helped us understand the essence of input/output and dictionaries in OpenFOAM. Now, let us examine the input/output methods provided by OpenFOAM.

OpenFOAM applications typically need to read dictionaries from the case and output computational results to the case, among other operations.

How does OpenFOAM implement reading from and writing to folders? OpenFOAM's input and output are more advanced; the method of indexing and searching by keyword is directly encapsulated in the relevant classes, and can be used directly by calling the methods. We do not need to delve into the implementation code at this stage.

This section primarily discusses:

- [ ] Understanding input/output for different data formats in OpenFOAM
- [ ] Understanding the methods provided by OpenFOAM dictionary classes
- [ ] Compiling and running a dictionary project

## 1. Project Preparation

Run the following commands in the terminal to create the project for this section:

```terminal {fileName="terminal"}
ofsp
foamNewApp ofsp_11_dictionary
code ofsp_11_dictionary
```

Run the following command in the terminal to prepare a test case for the project:

```terminal {fileName="terminal"}
cp -r $FOAM_TUTORIALS/incompressible/icoFoam/cavity/cavity debug_case
```

Run the following commands in the terminal to test the initial solver:

```terminal {fileName="terminal"}
wmake
ofsp_11_dictionary -case debug_case
```

The terminal output is as follows:

```terminal {fileName="terminal"}
Create time


ExecutionTime = 0 s  ClockTime = 0 s

End
```

The above output indicates that the initial solver is functioning correctly and can be used as a basis for development.

## 2. Documentation File

As a relatively complete OpenFOAM project, we provide a documentation file:

```markdown {fileName="/README.md"}
## About

This is a project for understanding OpenFOAM dictionaries.

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

@ 202509010
- Added cleaning script

```

## 3. Script Files

The scripts are similar to those discussed in previous projects; only the solver name in the scripts needs to be modified.

The `caserun` script is primarily responsible for running the test case after the application is successfully compiled. It currently contains the following content:

```bash {fileName="/caserun",linenos=table,linenostart=1}
#!/bin/bash

blockMesh -case debug_case | tee debug_case/log.mesh
echo "Meshing done."

ofsp_11_dictionary -case debug_case | tee debug_case/log.run
```

The `caseclean` script is primarily responsible for cleaning up the application to its pre-compilation state. If the application is modified, the test case should also be restored to its pre-run state. It currently contains the following content:

```bash {fileName="/caseclean",linenos=table,linenostart=1}
#!/bin/bash

rm -rf debug_case/log.*
foamCleanTutorials debug_case
echo "Cleaning done."
```

Run the following command in the terminal to grant permissions to the scripts:

```terminal {fileName="terminal"}
chmod +x caserun caseclean
```


## 4. File Structure

The file structure is as follows:

```terminal {fileName="terminal"}
.
├── caseclean
├── caserun
├── debug_case
│   ├── 0
│   │   ├── p
│   │   └── U
│   ├── constant
│   │   └── transportProperties
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
├── ofsp_11_dictionary.C
└── README.md
```

## 5. Main Source Code

The main source code in `ofsp_11_dictionary.C` is as follows:

```cpp {fileName="/ofsp_11_dictionary.C",linenos=table,linenostart=1}
#include "fvCFD.H"

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    #include "setRootCase.H"
    #include "createTime.H"

    #include "createMesh.H"


    /* 
    * About dictionary customProperties
    */
    const word dictName("customProperties"); // Create a word type object to store the dictionary name
    IOobject dictIO // Construct an IOobject type object
    (
        dictName, // Dictionary name
        runTime.constant(), // Dictionary location
        mesh, // Associated with the mesh
        IOobject::MUST_READ, // Must read from the dictionary
        IOobject::NO_WRITE // Do not write to the dictionary
    );

    if (!dictIO.typeHeaderOk<dictionary>(true))
    {
        FatalErrorIn(args.executable()) << "Cannot open specified dictionary"
            << dictName << exit(FatalError);
    }
    // If the dictionary file header does not specify a dictionary type, report an error

    /* 
    * About dictionary myProperties
    */

    dictionary myDictionary;
    myDictionary = IOdictionary(dictIO);
    // Create a dictionary object based on the IOobject entity

    // The following compact form is commonly used
    Info<< "Reading myProperties\n" << endl; 
    IOdictionary myProperties // Dictionary variable name and dictionary file name are the same
    (
        IOobject
        (
            "myProperties",
            runTime.constant(),
            mesh,
            IOobject::MUST_READ,
            IOobject::NO_WRITE
        )
    );

    word solver; // Create a word type variable
    myProperties.lookup("application") >> solver;
    // Find the keyword from the myProperties file and assign the value to solver

    word format(myProperties.lookup("writeFormat"));
    // Or written in a more compact form

    scalar timeStep(myProperties.lookupOrDefault("deltaT", scalar(0.01)));
    // Can also be written as
    // scalar timeStep(myProperties.lookupOrDefault<scalar>("deltaT", 0.01));
    // If the keyword is not provided in the dictionary, use the default value provided here

    dimensionedScalar alpha("alpha",dimless,myProperties);
    // This syntax is commonly used to read parameters from dictionaries
    
    dimensionedScalar beta(myProperties.lookup("beta"));
    // This form is common in older code, but in newer versions, although compilation succeeds, it issues a deprecation warning


    bool ifPurgeWrite(myProperties.lookupOrDefault<Switch>("purgeWrite",0));
    // bool types can also be read by keyword

    List<scalar> pointList(myProperties.lookup("point"));
    // Lists can also be read

    HashTable<vector,word> sourceField(myProperties.lookup("source"));
    // Hash tables can also be read

	vector myVec = vector(myProperties.subDict("subDict").lookup("myVec"));
	// Sub-dictionaries can also be used within dictionary files
	// Note the syntax for sub-dictionaries in the dictionary file below

	// Output the read content
    Info<< nl
        << "application: " << solver << nl << nl
        << "writeFormat: " << format << nl << nl
        << "deltaT: " << timeStep << nl << nl
        << "alpha: " << alpha << nl << nl
        << "beta: " << beta << nl << nl
        << "purgeWrite: " << ifPurgeWrite << nl << nl
        << "point: " << pointList << nl << nl
        << "source: " << sourceField << nl << nl
	    << "myVec: " << myVec << nl << nl
        << endl;

    /* 
    * Write to files
    */
    fileName outputDir = runTime.path()/"processing"; // Create an outputDir variable and assign a path
    mkDir(outputDir); // Create the folder for the above path

    autoPtr<OFstream> outputFilePtr; // Pointer to the output file stream
    outputFilePtr.reset(new OFstream(outputDir/"myOutPut.dat")); // Assign a target to the pointer

	// Write information to the output file via the pointer
    outputFilePtr() << "processing/myOutPut.dat" << endl;
    outputFilePtr() << "0 1 2 3 ..." << endl;
    sourceField.insert("U3", vector(1, 0.0, 0.0));
    outputFilePtr() << sourceField << endl;

    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< nl;
    runTime.printExecutionTime(Info);

    Info<< "End\n" << endl;

    return 0;
}

```

## 6. Providing Dictionaries

Provide the dictionary file `debug_case/constant/customProperties`. Since this dictionary has no read or write operations, it only needs a correct file header, with its content left blank.

```cpp {fileName="/debug_case/constant/customProperties",linenos=table,linenostart=1}
FoamFile
{
    version     2.0;
    format      ascii;
    class       dictionary;
    location    "constant";
    object      customProperties;
}

// Leave blank

```

The dictionary file `debug_case/constant/myProperties` has the following content:

```cpp {fileName="/debug_case/constant/myProperties",linenos=table,linenostart=1}
FoamFile
{
    version     2.0;
    format      ascii;
    class       dictionary;
    location    "constant";
    object      myProperties;
}

application     icoFoam;

writeFormat     ascii;

purgeWrite      1;

alpha           0.2;
beta            beta [0 0 0 0 0 0 0]  0.5;

point
(
    0
    1
    2
);

source
(
    U1 (0 0 0)
    U2 (1 0 0)
);

subDict
{
    myVec (0.0 0.0 1.0);
}

```

## 7. Compilation and Execution

Run the following commands in the terminal to compile and run:

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

Reading myProperties


application: icoFoam

writeFormat: ascii

deltaT: 0.01

alpha: alpha [0 0 0 0 0 0 0] 0.2

beta: beta [0 0 0 0 0 0 0] 0.5

purgeWrite: 1

point: 3(0 1 2)

source: 
2
(
U1 (0 0 0)
U2 (1 0 0)
)

myVec: (0 0 1)



ExecutionTime = 0 s  ClockTime = 0 s

End

```

Additionally, a new folder `debug_case/processing/` has been created under the test case folder, and the content of `myOutPut.dat` under this path is as follows:

```cpp {fileName="/debug_case/processing/myOutPut.dat",linenos=table,linenostart=1}
processing/myOutPut.dat
0 1 2 3 ...

3
(
U1 (0 0 0)
U3 (1 0 0)
U2 (1 0 0)
)

```

## 8. Summary

This project discussed the file stream input/output methods designed in OpenFOAM. In future practice, dictionaries and dictionary-related methods will be continuously used.

This section has completed the following discussions:

- [x] Understanding input/output for different data formats in OpenFOAM
- [x] Understanding the methods provided by OpenFOAM dictionary classes
- [x] Compiling and running a dictionary project


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


