---
uid: 20251104124420
title: 13_commandLine
date: 2025-11-04
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
weight: 14
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

Newcomers may find it hard to ignore that solvers always include a few fixed header files, such as `setRootCase.H`, `createTime.H`, and `createMesh.H`. Code explanations found online often provide only a brief comment about their functionality, which may not fully resolve the confusion. This series will discuss them separately.

The previous section discussed command-line arguments in C++. This section will transition to discussing command-line arguments in OpenFOAM. Before that, let us first examine the `setRootCase.H` file.

This section primarily discusses:

- [ ] Understanding the `setRootCase.H` header file
- [ ] Understanding the `argList` class
- [ ] Developing user-defined command-line arguments
- [ ] Compiling and running a commandLine project

## 1. Project Preparation

Run the following commands in the terminal to create the project:

```terminal {fileName="terminal"}
ofsp
foamNewApp ofsp_13_commandLine
cd ofsp_13_commandLine
cp -r $FOAM_TUTORIALS/incompressible/icoFoam/cavity/cavity debug_case
code .
```

Run the following command in the terminal to test the initial solver:

```terminal {fileName="terminal"}
wmake
ofsp_13_commandLine -case debug_case
```

The initial solver runs without issues and can be further developed. This step will not be repeated in the following sections.

Run the following commands in the terminal to create scripts and documentation, and grant permissions to the scripts:

```terminal {fileName="terminal"}
code caserun caseclean READEME.md
chmod +x caserun caseclean
```

Refer to previous discussions for the scripts and documentation. Unless there are special circumstances, details of the scripts and documentation will not be repeated.

Run the following command to view the file structure:

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
├── ofsp_13_commandLine.C
└── README.md

6 directories, 15 files
```

Unless there are special circumstances, simple file structures will not be elaborated further.

## 2. setRootCase.H

API page: [https://api.openfoam.com/2506/setRootCase_8H.html](https://api.openfoam.com/2506/setRootCase_8H.html)

We can locate this file via the terminal:

```terminal {fileName="terminal"}
find $FOAM_SRC -iname setRootCase.H
```

Using VS Code, we can directly click on the file path output in the terminal to open the file.

In VS Code, we can also use the OFextension to directly jump to and open the file.

For ease of understanding, let us examine the code from the OpenFOAM 2.0x version.

The GitHub repository file link is: [https://github.com/OpenFOAM/OpenFOAM-2.0.x/blob/master/src/OpenFOAM/include/setRootCase.H](https://github.com/OpenFOAM/OpenFOAM-2.0.x/blob/master/src/OpenFOAM/include/setRootCase.H)

The code is as follows:

```cpp {fileName="setRootCase.H",linenos=table,linenostart=1}
//
// setRootCase.H
// ~~~~~~~~~~~~~

    Foam::argList args(argc, argv);
    if (!args.checkRootCase())
    {
        Foam::FatalError.exit();
    }
```

The code in the modern version is as follows:

```cpp {fileName="setRootCase.H",linenos=table,linenostart=1}
// Construct from (int argc, char* argv[]),
// - use argList::argsMandatory() to decide on checking command arguments.
// - check validity of the options

Foam::argList args(argc, argv); // Instantiate the args object of the argList class
if (!args.checkRootCase()) 
{
    Foam::FatalError.exit();
}
// Check the root directory and case directory based on the case root directory
// 1. Whether the root directory exists
// 2. Whether the case directory exists
// Otherwise, report an error and exit

// The args object is constructed based on the command-line arguments argc, argv
// Therefore, this .H file must be placed after all code related to command-line arguments

// User can also perform checks on otherwise optional arguments.
// Eg,
//
//  if (!args.check(true, false))
//  {
//      Foam::FatalError.exit();
//  }
// Users can add additional parameter validation following this template; we will not delve into this for now

// Force dlOpen of FOAM_DLOPEN_LIBS (principally for Windows applications)
#include "foamDlOpenLibs.H"
// Compatibility-related; we will not delve into this for now
```

Many solvers also use the enhanced `setRootCaseLists.H`, whose code is as follows:

```cpp {fileName="setRootCaseLists.H"}
// This is setRootCase, but with additional solver-related listing

#include "setRootCaseListOptions.H"
// Add a series of command-line options for querying and listing to the solver

#include "setRootCase.H" // Original version

#include "setRootCaseListOutput.H"
// When the program starts, check whether the user has used these options and perform the corresponding output operations (e.g., list information and exit the program)

```

This header file not only maintains the simplicity of the basic functionality but also provides flexible extensibility for solvers (such as post-processing tools) that need to use these advanced features.

For example, a solver with added command-line options can be used in the terminal as follows:

```terminal {fileName="terminal"}
simpleFoam -listSwitches
```

The terminal will display all debugging/optimization switches.

Readers are encouraged to explore the command-line options provided in the code on their own.

## 3. argList Class

The `Foam::argList` class is a fundamental class in OpenFOAM, with many member data and member methods. Let us briefly explore the `argList` class by looking at a few parts of the code.

API page: [https://api.openfoam.com/2506/argList_8H.html](https://api.openfoam.com/2506/argList_8H.html)

We can locate this file via the terminal:

```terminal {fileName="terminal"}
find $FOAM_SRC -iname argList.H
```

Open the declaration of this class in `argList.H`, which contains the following:

```cpp {fileName="argList.H",linenos=table,linenostart=1}
...
...
class argList
{
	...
public:
	...
	// Constructor based on command-line arguments
	argList
	(
		int& argc, // Number of main function arguments
		char**& argv, // Pointer to main function arguments
		bool checkArgs = argList::argsMandatory(), // Whether argument checking is mandatory
		bool checkOpts = true,
		bool initialise = true
	);
	...
	//- Check root path and case path
        bool checkRootCase() const;
    ...
...
```

To see the implementation of the member function `checkRootCase()`, we need to look at the class definition in the same directory, i.e., the `argList.C` file. A portion of its content is as follows:

```cpp {fileName="argList.C",linenos=table,linenostart=1}
...
bool Foam::argList::checkRootCase() const // The function returns a boolean type
{
    if (!fileHandler().isDir(rootPath())) // Check whether the case root directory exists
    {
        FatalError
            << executable_
            << ": cannot open root directory " << rootPath()
            << endl;

        return false;
    }
    // If the case root directory (the parent directory of the case) does not exist, trigger this error
    // Example: simpleFoam -case wrongRoot/cavity

	// Check whether the case directory is incorrect
    const fileName pathDir(fileHandler().filePath(path(), false));

    if (checkProcessorDirectories_ && pathDir.empty() && Pstream::master())
    {
        // Allow non-existent processor directories on sub-processes,
        // to be created later (e.g. redistributePar)
        FatalError
            << executable_
            << ": cannot open case directory " << path()
            << endl;

        return false;
    }
    // If the case directory does not exist, trigger this error
    // Example: simpleFoam -case rootPath/wrongCase

    return true;
}
...
```

At this point, readers should understand what directories `setRootCase.H` actually checks.

>[!tip]
>At this stage, it is not advisable for readers to delve too deeply into the code. It is sufficient to have a general understanding without feeling unfamiliar or resistant.
>
>More in-depth code discussions can be found in the `ofsc` (OpenFOAM Sharing Coding) series.

## 4. Help Information

We can use command-line arguments to implement certain functionalities.

Modify the main function as follows:

```cpp {fileName="ofsp_13_commandLine/ofsp_13_commandLine.C",linenos=table,linenostart=1}
#include "fvCFD.H"

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    argList::addNote
    (
        "Program for command line @ Aerosand\n"
        "Solver setup:\n"
        "   Accuracy     - o1, o2, o3\n"
        "   Acceleration - 1, 2, 3\n"
    );

    #include "setRootCase.H"
    #include "createTime.H"

    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< nl;
    runTime.printExecutionTime(Info);

    Info<< "End\n" << endl;

    return 0;
}

```

Run the following commands in the terminal to compile and view the help information:

```terminal {fileName="terminal"}
wmake
ofsp_13_commandLine -help
```

The terminal output is as follows:

```terminal {fileName="terminal"}
Usage: ofsp_13_commandLine [OPTIONS]
Options:
  -case <dir>       Case directory (instead of current directory)
  -decomposeParDict <file>
                    Alternative decomposePar dictionary file
  -parallel         Run in parallel
  -doc              Display documentation in browser
  -help             Display short help and exit
  -help-full        Display full help and exit

Program for command line @ Aerosand
Solver options:
   Calculation Accuracy     - o1, o2, o3
   Calculation Acceleration - 1, 2, 3

Using: OpenFOAM-2406 (2406) - visit www.openfoam.com
Build: _9bfe8264-20241212 (patch=241212)
Arch:  LSB;label=32;scalar=64
```

Custom help information can be seen here.

## 5. Mandatory Arguments

Mandatory arguments must be provided when running the project; otherwise, an error will be triggered.

Modify the main function as follows:

```cpp {fileName="ofsp_13_commandLine/ofsp_13_commandLine.C",linenos=table,linenostart=1}
#include "fvCFD.H"

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    argList::addNote
    (
        "Program for command line @ Aerosand\n"
        "Solver setup:\n"
        "   Accuracy     - o1, o2, o3\n"
        "   Acceleration - 1, 2, 3\n"
    );

    argList::validArgs.append("Accuracy");
    argList::validArgs.append("Acceleration");
    // Append application arguments to the main function argument list
    // These two arguments must be provided when running the program

    #include "setRootCase.H"
    #include "createTime.H"

	// The 0th argument of args is the program name itself
    const word args1 = args[1]; // The 1st argument of args is the 1st appended argument
    const scalar args2 = args.get<scalar>(2); // The 2nd appended argument


	// Display the arguments
    Info<< "Solver setup: " << nl
        << "    Accuracy     : " << args1 << nl
        << "    Acceleration : " << args2 << nl
        << nl << endl;

    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< nl;
    runTime.printExecutionTime(Info);

    Info<< "End\n" << endl;

    return 0;
}

```

Recompile this project.

Run the following command in the terminal to run the project:

```terminal {fileName="terminal"}
ofsp_13_commandLine -case debug_case/ o2 2
```

The terminal output is as follows:

```terminal {fileName="terminal"}
Create time

Solver setup: 
    Accuracy     : o2
    Acceleration : 2



ExecutionTime = 0 s  ClockTime = 0 s

End
```

As can be imagined, these command-line arguments can be used to participate in the project's computation.

## 6. Application Options

Application options can be provided or omitted when running the project.

Modify the main function as follows:

```cpp {fileName="ofsp_13_commandLine/ofsp_13_commandLine.C",linenos=table,linenostart=1}
#include "fvCFD.H"

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    argList::addNote
    (
        "Program for command line @ Aerosand\n"
        "Solver setup:\n"
        "   Accuracy     - o1, o2, o3\n"
        "   Acceleration - 1, 2, 3\n"
    );

    argList::validArgs.append("Accuracy");
    argList::validArgs.append("Acceleration");
    // Append application arguments to the main function argument list
    // These two arguments must be provided when running the program

    argList::addOption
    (
        "dict",
        "word",
        "Use additional dictionary (just for example)"
    );

    argList::addOption
    (
        "nPrecision",
        "label",
        "Set the precision (just for example)"
    );

    argList::addBoolOption
    (
        "debug",
        "Enable debug mode (just for example)"
    );

    #include "setRootCase.H"
    #include "createTime.H"

    #include "createMesh.H"

	// The 0th argument of args is the program name itself
    const word args1 = args[1]; // The 1st argument of args is the 1st appended argument
    const scalar args2 = args.get<scalar>(2); // The 2nd appended argument


	// Display the arguments
    Info<< "Solver setup: " << nl
        << "    Accuracy     : " << args1 << nl
        << "    Acceleration : " << args2 << nl
        << nl << endl;


    // Determine the dictionary file path
    fileName dictPath("./system/myDict");
    if (args.found("dict"))
    {
        args.readIfPresent("dict", dictPath);
        Info<< "Using custom dictionary: " << dictPath << endl;
    }
    else
    {
        Info<< "Using default dictionary: " << dictPath << endl;
    }

	// Custom dictionary object
    IOdictionary myDict
    (
        IOobject
        (
            dictPath,
            runTime,
            IOobject::MUST_READ,
            IOobject::NO_WRITE
        )
    );
    
    const word mathLib = myDict.get<word>("mathLib");
    const scalar tolerance = myDict.get<scalar>("tolerance");

    fileName dict_("./system/myDict"); // Default dictionary path
    if (args.optionFound("dict"))
    {
        args.readIfPresent("dict", dict_);
        Info<< "Reading myDict " << endl;
    }
    Info<< "Dictionary from " << dict_ << nl
        << "    mathLib   : " << mathLib << nl
        << "    tolerance : " << tolerance << nl
        << nl << endl;

    if (args.optionFound("nPrecision"))
    {
        label nPrecision_(6);
        args.readIfPresent("nPrecision", nPrecision_);
        Info<< "Setting nPrecision to " << nPrecision_ << nl << endl;
    }

    if (args.optionFound("debug"))
    {
        Info<< "Debug mode enabled" << nl << endl;
    }


    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< nl;
    runTime.printExecutionTime(Info);

    Info<< "End\n" << endl;

    return 0;
}


// ************************************************************************* //

```

Recompile this project.

Provide a custom dictionary for this project. Instead of using the default dictionary path set in the code, create a dictionary at `/debug_case/constant/myDict`.

The custom dictionary is as follows:

```cpp {fileName="/debug_case/constant/myDict",linenos=table,linenostart=1}
FoamFile
{
    version     2.0;
    format      ascii;
    class       dictionary;
    location    "system";
    object      myDict;
}


mathLib    Eigen;
tolerance  1e-6;
```

Run the following command in the terminal to run the project:

```terminal {fileName="terminal"}
ofsp_13_commandLine -case debug_case/ o2 2 -dict ./debug_case/constant/myDict -nPrecision 16 -debug
```

The terminal output is as follows:

```terminal {fileName="terminal"}
Create time

Create mesh for time = 0

Solver setup: 
    Accuracy     : o2
    Acceleration : 2


Using custom dictionary: "./debug_case/constant/myDict"
Reading myDict 
Dictionary from "./debug_case/constant/myDict"
    mathLib   : Eigen
    tolerance : 1e-06


Setting nPrecision to 16

Debug mode enabled


ExecutionTime = 0 s  ClockTime = 0 s

End
```


## 7. Other Command-Line Features

The `postProcess.H` header file is also commonly found in solver header files. This header file is primarily used to integrate **post-processing functionality** into solvers or utility programs. Its core purpose is to allow users to compute and output additional derived fields (such as vorticity, stream function, gradients, etc.) during or after simulation execution, without the need to run separate post-processing tools.

For example, after the computation is complete, the following command can be entered in the terminal:

```terminal {fileName="terminal"}
postProcess -func vorticity
```

This calculates the vorticity field based on the saved time steps without modifying the code.

Alternatively, the `functions` module can be added in `controlDict`, such as:

```cpp {fileName="controlDict"}
...
functions
{
    vorticity
    {
        type    vorticity;
        libs    (fieldFunctionObjects);
    }
}
```

This allows the solver to compute and output the vorticity field at every write time step.

Additionally, the `addCheckCaseOptions.H` header file is also commonly found in solver header files, with the following code:

```cpp {fileName="addCheckCaseOptions.H"}
Foam::argList::addDryRunOption
(
    "Check case set-up only using a single time step"
);
Foam::argList::addBoolOption
(
    "dry-run-write",
    "Check case set-up and write only using a single time step"
);
```

This header file adds additional standard command-line options to the solver, primarily used to check the completeness of case setup without running the full simulation.

For example, the following command can be entered in the terminal:

```terminal {fileName="terminal"}
simpleFoam -case debug_case -dry-run
// Check case setup only, do not run the solver

simpleFoam -case debug_case -dry-run-write
// Check case setup and write the initial fields
```

These commands allow users to quickly identify setup errors, avoiding the situation where problems are only discovered halfway through a full run.

## 8. Summary

This section has step by step discussed the possible functionalities of command-line arguments in OpenFOAM. Although the project presented here is only a simple demonstration, I believe readers can, based on this discussion, understand command-line arguments in OpenFOAM and gain ideas for their own future development.

This section primarily discussed:

- [x] Understanding the `setRootCase.H` header file
- [x] Understanding the `argList` class
- [x] Developing user-defined command-line arguments
- [x] Compiling and running a commandLine project



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

