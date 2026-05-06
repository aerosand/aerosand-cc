---
uid: 20251112144059
title: 14_time
date: 2025-11-12
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
weight: 15
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

This section briefly introduces the Time class in OpenFOAM, and based on this, discusses the fundamental header file `createTime.H`.

This section primarily discusses:

- [ ] Testing new scripts
- [ ] Understanding the `createTime.H` header file
- [ ] Understanding the Time class
- [ ] Understanding time-related member methods
- [ ] Compiling and running a time project

## 1. Project Preparation

Run the following commands in the terminal to create the project:

```terminal {fileName="terminal"}
ofsp
foamNewApp ofsp_14_time
cd ofsp_14_time
cp -r $FOAM_TUTORIALS/incompressible/icoFoam/cavity/cavity debug_case
code .
```

Run the following command in the terminal to test the initial solver:

```terminal {fileName="terminal"}
wmake
ofsp_14_time -case debug_case
```

The initial solver runs without issues and can be further developed. This step will not be repeated in the following sections.

### 1.1. Scripts and Documentation

Readers may have noticed that the scripts used in previous projects are not generic and require modifications each time, which is quite tedious.

We will gradually introduce more convenient script writing methods.

Run the following command in the terminal to create new scripts:

```terminal {fileName="terminal"}
code caserun caseclean
```

The `caserun` script is responsible for running the test case after the application is successfully compiled. Its content is as follows:

```bash {fileName="caserun",linenos=table,linenostart=1}
#!/bin/sh
cd "${0%/*}" || exit 1                              # Run from this directory
#------------------------------------------------------------------------------

appPath=$(cd `dirname $0`; pwd)                     # Get application path
appName="${appPath##*/}"                            # Get application name

blockMesh -case debug_case | tee debug_case/log.mesh
echo "Meshing done."

$appName -case debug_case | tee debug_case/log.run

```

Here, `$0` returns the absolute path of the current script, `%xxx` removes the `xxx` characters, and `/*` represents `/` and all characters after it. `|| exit 1` means that if the `cd` command fails (returns a non-zero value), the script exits with error code 1.

>[!tip]
>Do not worry about the script language for now; it is sufficient to know that the above script can be used. Deeper script language knowledge will be introduced gradually as the discussion progresses.

The `caseclean` script is responsible for cleaning up the test case and restoring it to its initial state. Its content is as follows:

```bash {fileName="caseclean",linenos=table,linenostart=1}
#!/bin/sh
cd "${0%/*}" || exit 1                              # Run from this directory
#------------------------------------------------------------------------------

appPath=$(cd `dirname $0`; pwd)
appName="${casePath##*/}"

cd debug_case

rm -rf log.*
foamCleanTutorials
echo "Cleaning Done."

```

These two scripts automatically obtain the application path and application name, making them suitable for use in other applications discussed later. Simply copy and use them.

Run the following command in the terminal to make the scripts executable:

```terminal {fileName="terminal"}
chmod +x caserun caseclean
```


## 2. createTime.H

API page: [https://api.openfoam.com/2506/createTime_8H.html](https://api.openfoam.com/2506/createTime_8H.html)

We can locate this file via the terminal:

```terminal {fileName="terminal"}
find $FOAM_SRC -iname createTime.H
```

Using VS Code, we can directly click on the file path output in the terminal to open the file.

In VS Code, we can also use the OFextension to directly jump to and open the file; this will not be repeated in the following sections.

The code is as follows:

```cpp {fileName="createTime.H",linenos=table,linenostart=1}
Foam::Info<< "Create time\n" << Foam::endl;
// A simple output statement

Foam::Time runTime(Foam::Time::controlDictName, args);
// Constructs a runTime object of the Foam::Time class
// The construction is based on controlDictName and args
// args needs to be created in advance, so setRootCase.H must come before this

```

## 3. Time Class

The Time class is a fundamental class in OpenFOAM, with many member data and member methods. Let us briefly explore the Time class by looking at a few parts of the code.

API page: [https://api.openfoam.com/2506/Time_8H.html](https://api.openfoam.com/2506/Time_8H.html)

Run the following command in the terminal to view the declaration of the Time class:

```terminal {fileName="terminal"}
find $FOAM_SRC -iname Time.H
```

Open `Time.H`, which contains the following:

```cpp {fileName="Time.H",linenos=table,linenostart=1}
...
class Time
:
    public clock,
    public cpuTime,
    public TimePaths,
    public objectRegistry,
    public TimeState
    // The Time class inherits from several base classes
{
public:
	...
private:
	...
protected:
	...
private:
	...
public:
	...
	// Constructor based on parameters
	inline Time
	(
		const word& ctrlDictName, // Reference to a word type
		const argList& args, // Reference to an argList type
		const bool enableFunctionObjects = true,
		const bool enableLibs = true
	);
	// Find this inline function; it is the method prototype for line 2 of createTime.H
	// The third and fourth formal parameters default to true and can be omitted
	// This constructor corresponds to the construction in createTime.H
	
	...
	
	// Searching

	//- Return time instance (location) of \c directory containing
	//- the file \c name (eg, used in reading mesh data).
	//- When \c name is empty, searches for \c directory only.
	//- Does not search beyond \c stopInstance (if set) or \c constant.
	//
	//  If the instance cannot be found:
	//  - FatalError when readOpt is MUST_READ or READ_MODIFIED
	//  - return \c stopInstance (if set and reached)
	//  - return \c constant if constant_fallback is true
	//  - return an empty word if constant_fallback is false
	//  .
	word findInstance
	(
		//! The subdirectory (local) for the search
		const fileName& directory,
		//! The filename for the search. If empty, only search for directory
		const word& name = word::null,
		//! The search type : generally MUST_READ or READ_IF_PRESENT
		IOobjectOption::readOption rOpt = IOobjectOption::MUST_READ,
		//! The search stop instance
		const word& stopInstance = word::null,
		//! Return \c "constant" instead of \c "" if the search failed
		const bool constant_fallback = true
	) const;
	// It can be seen that the Time class provides a search mechanism
	// Starting from the current time, it searches through time directories from the most recent to the earliest,
	// until it finds a time instance containing the directory, or reaches the set stopInstance.
	// If still not found, and constant_fallback is true, it checks the constant directory.
	// If still not found, it reports an error based on the read option, otherwise returns an empty string.
	// This mechanism is generally used to find mesh data.
	// Code details will not be delved into at this stage.
	
	...
	
	// Member Functions
		// Access
		// Member functions of the Time class
		//- Return start time
		virtual dimensionedScalar startTime() const;

		//- Return end time
		virtual dimensionedScalar endTime() const;

		//- Return the stop control information
		virtual stopAtControls stopAt() const;
	...
	
	
...
```


>[!tip]
>To reiterate, at this stage, it is not advisable for readers to delve too deeply into the code. It is sufficient to have a general understanding without feeling unfamiliar or resistant.

## 4. Time Information

We can use some methods of the Time class to output and utilize time information.

Modify the main function as follows:

```cpp {fileName="ofsp_14_time/ofsp_14_time.C",linenos=table,linenostart=1}
#include "fvCFD.H"

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    #include "setRootCase.H"
    // #include "createTime.H"
    // Expand the content of createTime.H
    Foam::Info<< "Create time\n" << Foam::endl;
    Foam::Time runTime(Foam::Time::controlDictName, args);
    // The runTime object is created
    // This object has the member methods and properties of the Time class


    Info<< "Case name   :   " << runTime.caseName() << endl;
    Info<< "Root path   :   " << runTime.rootPath() << endl;
    Info<< "Path        :   " << runTime.path() << endl;
    Info<< "Time path   :   " << runTime.timePath() << endl;
    Info<< "Constant path :   " << runTime.constantPath() << endl;
    Info<< "Constant    :   " << runTime.constant() << endl;
    Info<< "System path :   " << runTime.systemPath() << endl;
    Info<< "Controdict  :   " << runTime.controlDict() << endl;
    Info<< "Format      :   " << runTime.writeFormat() << endl;
    Info<< "Version     :   " << runTime.writeVersion() << endl;
    Info<< "Start time  :   " << runTime.startTime() << endl;
    Info<< "End time    :   " << runTime.endTime() << endl;
    Info<< "Time step   :   " << runTime.deltaT() << endl;

    Info<< "# set end time = 10" << endl;
    runTime.setEndTime(10);

    Info<< "# set deltaT = 1" << endl;
    runTime.setDeltaT(1);

    Info<< "Start time  :   " << runTime.startTime() << endl;
    Info<< "End time    :   " << runTime.endTime() << endl;
    Info<< "Time step   :   " << runTime.deltaT() << endl;


    while (runTime.loop())
    {
        Info<< "Time: " << runTime.timeName() << endl;
        runTime.write();
    }

    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< nl;
    // runTime.printExecutionTime(Info);
    // Expand the main content of the member function printExecutionTime()
    Info<< "Execution time: " << runTime.elapsedCpuTime() << " s" << endl;
    Info<< "Clock time: " << runTime.elapsedClockTime() << " s" << endl;

    Info<< "End\n" << endl;

    return 0;
}

```

Run the following commands in the terminal to compile and run this project:

```terminal {fileName="terminal"}
wclean
wmake
./caseclean
./caserun
```

The terminal output is as follows:

```terminal {fileName="terminal"}
Create time

Case name   :   "debug_case"
Root path   :   "/home/aerosand/01_project/ofsp/ofsp_14_time"
Path        :   "/home/aerosand/01_project/ofsp/ofsp_14_time/debug_case"
Time path   :   "/home/aerosand/01_project/ofsp/ofsp_14_time/debug_case/0"
Constant path :   "/home/aerosand/01_project/ofsp/ofsp_14_time/debug_case/constant"
Constant    :   constant
System path :   "/home/aerosand/01_project/ofsp/ofsp_14_time/debug_case/system"
Controdict  :   
{
    application     icoFoam;
    startFrom       startTime;
    startTime       0;
    stopAt          endTime;
    endTime         0.5;
    deltaT          0.005;
    writeControl    timeStep;
    writeInterval   20;
    purgeWrite      0;
    writeFormat     ascii;
    writePrecision  6;
    writeCompression off;
    timeFormat      general;
    timePrecision   6;
    runTimeModifiable true;
}

Format      :   ascii
Version     :   2.0
Start time  :   startTime [0 0 1 0 0 0 0] 0
End time    :   endTime [0 0 1 0 0 0 0] 0.5
Time step   :   deltaT [0 0 1 0 0 0 0] 0.005
# set end time = 10
# set deltaT = 1
Start time  :   startTime [0 0 1 0 0 0 0] 0
End time    :   endTime [0 0 1 0 0 0 0] 10
Time step   :   deltaT [0 0 1 0 0 0 0] 1
Time: 1
Time: 2
Time: 3
Time: 4
Time: 5
Time: 6
Time: 7
Time: 8
Time: 9
Time: 10

Execution time: 0 s
Clock time: 0 s
End
```

Through this project, it can be seen that the `runTime` object can not only access time directories but also access folders such as `system/`, `constant/`, etc.

## 5. Summary

This section briefly discussed the Time class and explained the roles of the member methods of the Time class.

So far, we have generally understood the common header files `setRootCase.H` and `createTime.H` in OpenFOAM projects. The next section will discuss another essential header file.

This section has completed the following discussions:

- [x] Testing new scripts
- [x] Understanding the `createTime.H` header file
- [x] Understanding the Time class
- [x] Understanding time-related member methods
- [x] Compiling and running a time project



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


