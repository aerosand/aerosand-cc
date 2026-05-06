---
uid: 20251020195957
title: 12_arg
date: 2025-10-20
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
weight: 13
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

Many OpenFOAM applications can accept runtime parameters via the command line. Although users may sometimes find this unfamiliar, mastering command-line usage enables efficient and flexible handling of various problems.

We will start with C++ by briefly introducing command-line arguments in C++, and then further discuss command-line arguments in OpenFOAM.

This section primarily discusses:

- [ ] Understanding the OpenFOAM command line
- [ ] Understanding command-line arguments in C++
- [ ] Understanding the functionality and options of command-line arguments
- [ ] Compiling and running an arg project

## 1. OpenFOAM Command Line

Reference: [https://doc.openfoam.com/2312/fundamentals/command-line/](https://doc.openfoam.com/2312/fundamentals/command-line/)

The basic usage format of the OpenFOAM command line is as follows:

```terminal {fileName="terminal"}
<application> <options> <arguments>
```

For example, a typical command-line usage in practice is:

```terminal {fileName="terminal"}
blockMesh -case debug_case
```

Using the `-help` option allows viewing more command-line options. For instance, entering `blockMesh -help` in the terminal produces the following output:

```terminal {fileName="terminal"}
Usage: blockMesh [OPTIONS]
Options:
  -case <dir>       Case directory (instead of current directory)
  -dict <file>      Alternative blockMeshDict
  -merge-points     Geometric point merging instead of topological merging
                    [default for 1912 and earlier].
  -no-clean         Do not remove polyMesh/ directory or files
  -region <name>    Specify mesh region (default: region0)
  -sets             Write cellZones as cellSets too (for processing purposes)
  -time <time>      Specify a time to write mesh to (default: constant)
  -verbose          Force verbose output. (Can be used multiple times)
  -write-vtk        Write topology as VTU file and exit
  -doc              Display documentation in browser
  -help             Display short help and exit
  -help-compat      Display compatibility options and exit
  -help-full        Display full help and exit

Block mesh generator.

  The ordering of vertex and face labels within a block as shown below.
  For the local vertex numbering in the sequence 0 to 7:
    Faces 0, 1 (x-direction) are left, right.
    Faces 2, 3 (y-direction) are front, back.
    Faces 4, 5 (z-direction) are bottom, top.

                        7 ---- 6
                 f5     |\     :\     f3
                 |      | 4 ---- 5     \
                 |      3.|....2 |      \
                 |       \|     \|      f2
                 f4       0 ---- 1
    Y  Z
     \ |                f0 ------ f1
      \|
       o--- X

Using: OpenFOAM-2406 (2406) - visit www.openfoam.com
Build: _9bfe8264-20241212 (patch=241212)
Arch:  LSB;label=32;scalar=64
```


Generally, the command line is also heavily used during data processing and post-processing.

For example, using:

```terminal {fileName="terminal"}
foamPostProcess -help
```

Specific usage techniques will not be elaborated here and will be discussed separately in the future.

Let us first briefly review the use of command-line arguments in C++.

## 2. Project Preparation

Run the following commands in the terminal to create the project:

```terminal {fileName="terminal"}
ofsp
mkdir ofsp_12_arg
code ofsp_12_arg
```

Using VS Code, press `F1` and enter `Create C++ project`, enter `ofsp_12_arg` as the project name, select `/ofsp` as the parent directory, and initialize the project.

Run the following command in the terminal to test the initial project:

```terminal {fileName="terminal"}
make run
```

The terminal output `Hello world!` indicates that the initial project runs successfully.

## 3. Command-Line Arguments

When we first learn C++, the parameters of the main function are typically left empty, as follows:

```cpp
...
int main() // Omitted parameter list
{
	...
}
```

When delving deeper into C++ development, we learn that the more general form of the main function is:

```cpp
...
int main(int argc, char *argv[]) {}
// Or
int main(int argc, char **argv) {}
```

Where:

- `argc` stands for "argument count," storing the number of arguments passed to the main function when the program is executed.
- `argv` stands for "argument vector," storing pointers to the specific arguments passed to the main function when the program is executed. Each pointer points to a specific argument.
    - `argv[0]` points to the full path name of the program at runtime.
    - `argv[1]` points to the first string after the program name in the command line at runtime.
    - `argv[2]` points to the second string after the program name in the command line at runtime.
    - And so on for the rest.

## 4. Argument Order

The main source code in `src/main.cpp` is as follows:

```cpp {fileName="/src/main.cpp",linenos=table,linenostart=1}
#include <iostream>

int main(int argc, char *argv[])
{
    std::cout << "Number of arguments = " << argc << std::endl;

    for (int i=0; i<argc; ++i)
    {
        std::cout << "Argument " << i << ": "
            << argv[i] << std::endl;
    }

    return 0;
}
```

Run the following command in the terminal to compile and run:

```terminal {fileName="terminal"}
make run
```

The terminal output is as follows:

```terminal {fileName="terminal"}
Number of arguments = 1
Argument 0: ./output/main
```

If additional arguments are provided at runtime, for example:

```terminal {fileName="terminal"}
./output/main hi hey hello
```

The terminal output is as follows:

```terminal {fileName="terminal"}
Number of arguments = 4
Argument 0: ./output/main
Argument 1: hi
Argument 2: hey
Argument 3: hello
```


From the results, we can see that `argc` is, of course, the total number of arguments, and `argv[0]` is the application name itself, with other arguments following in order.

For example, for the following command line:

```terminal {fileName="terminal"}
blockMesh -case debug_case
```

Here, `argc` equals `3`, with `argv[0]` being `blockMesh`, `argv[1]` being `-case`, and `argv[2]` being `debug_case`.

In summary, each parameter in the command line can be accessed in the program through the main function parameters, allowing them to participate in the program's execution and computation.

## 5. Argument Functionality

Based on this project, we modify the main source code to add functionality to the command-line arguments.

The main source code in `src/main.cpp` is as follows:

```cpp {fileName="/src/main.cpp",linenos=table,linenostart=1}
#include <iostream>
#include <fstream>
#include <string>
using namespace std;

int main(int argc, char *argv[])
{
    std::cout << "Number of arguments = " << argc << std::endl;

    for (int i = 0; i < argc; ++i)
    {
        std::cout << "Argument " << i << ": "
                  << argv[i] << std::endl;
    }

    // Execute this branch if there are no additional command-line arguments
    if (argc < 2) {
        cout << "Usage: " << argv[0] << " <filename> [mode]" << endl;
        cout << "Modes: read (default), write" << endl;
        return 1;
    }
    
    string filename = argv[1];
    string mode = (argc > 2) ? argv[2] : "read"; // Default mode set to read
    
    if (mode == "read") { // Implement read mode
        ifstream file(filename);
        if (!file) { // Error handling
            cout << "Error: Cannot open file " << filename << endl;
            return 1; // Terminate the program
        }
        
        string line;
        cout << "File content:" << endl;
        // Output all lines in the file
        while (getline(file, line)) {
            cout << line << endl;
        }
        file.close();
        
    } else if (mode == "write") { // Implement write mode
        ofstream file(filename, ios::app); // Append mode
        if (!file) { // Error handling
            cout << "Error: Cannot create/write to file " << filename << endl;
            return 1;
        }
        
        cout << "Enter content to write (empty line to finish):" << endl;
        string line;
        // Continue input line by line as long as line is not empty
        while (getline(cin, line) && !line.empty()) {
            file << line << endl;
        }
        file.close();
        cout << "Content written to file" << endl;
        
    } else { // Invalid mode error handling
        cout << "Error: Unsupported mode '" << mode << "'" << endl;
        return 1;
    }
    
    return 0;
}

```

Run the following command in the terminal to compile the program:

```terminal {fileName="terminal"}
make
```

Run the following command to view the default output:

```terminal {fileName="terminal"}
./output/main
```

The terminal output is as follows:

```terminal {fileName="terminal"}
Number of arguments = 1
Argument 0: ./output/main
Usage: ./output/main <filename> [mode]
Modes: read (default), write
```

Run the following command to write to a file:

```terminal {fileName="terminal"}
./output/main data.txt write
```

The terminal output prompt is as follows:

```terminal {fileName="terminal"}
Number of arguments = 3
Argument 0: ./output/main
Argument 1: data.txt
Argument 2: write
Enter content to write (empty line to finish):

```

Enter test information in the terminal, and press `Enter` with a blank line to finish input:

```terminal {fileName="terminal"}
ofsp
This is a ofsp test.

```

Run the following command to read from the file:

```terminal {fileName="terminal"}
./output/main data.txt read
```

The terminal output is as follows:

```terminal {fileName="terminal"}
Number of arguments = 3
Argument 0: ./output/main
Argument 1: data.txt
Argument 2: read
File content:
ofsp
This is a ofsp test.
```

It can be seen that by entering different command-line arguments, different functionalities of the program can be achieved.

## 6. Argument Options

We continue developing this project to implement command-line argument options.

The main source code in `src/main.cpp` is as follows:

```cpp {fileName="/src/main.cpp",linenos=table,linenostart=1}
#include <iostream>
#include <fstream>
#include <string>
#include <vector>
using namespace std;

// Function to display help information
void displayHelp(const string& programName) {
    cout << "Usage: " << programName << " [OPTIONS] <filename> [mode]" << endl;
    cout << "A simple file read/write utility @ Aerosand" << endl << endl;
    cout << "Arguments:" << endl;
    cout << "  filename          Name of the file to read/write" << endl;
    cout << "  mode              Operation mode: read or write (default: read)" << endl << endl;
    cout << "Options:" << endl;
    cout << "  -h, --help        Display this help message" << endl;
    cout << "  -l                Show line numbers when reading" << endl;
    cout << "  -v, --version     Display version information" << endl;
    cout << "  -a, --append      Use append mode when writing (default)" << endl;
    cout << "  -o, --overwrite   Use overwrite mode when writing" << endl << endl;
    cout << "Examples:" << endl;
    cout << "  " << programName << " data.txt read" << endl;
    cout << "  " << programName << " -l notes.txt" << endl;
    cout << "  " << programName << " -o log.txt write" << endl;
    cout << "  " << programName << " -a data.txt write" << endl;
}

// Function to display version information
void displayVersion() {
    cout << "Version v1.0 @ Aerosand" << endl;
    cout << "This is a ofsp program from Aerosand" << endl;
}

// Structure to store parsed command-line arguments
struct Config {
    string filename;
    string mode = "read";
    bool showLineNumbers = false;
    bool showHelp = false;
    bool showVersion = false;
    bool appendMode = true;  // Default is append mode
    bool overwriteMode = false;
};

// Function to parse command-line arguments
Config parseArguments(int argc, char* argv[]) {
    Config config;
    vector<string> positionalArgs;
    
    // Skip the program name, iterate over the remaining command-line arguments
    for (int i = 1; i < argc; i++) {
        string arg = argv[i];
        
        if (arg == "-h" || arg == "--help") {
            config.showHelp = true;
        } else if (arg == "-l") {
            config.showLineNumbers = true;
        } else if (arg == "-v" || arg == "--version") {
            config.showVersion = true;
        } else if (arg == "-a" || arg == "--append") {
            config.appendMode = true;
            config.overwriteMode = false;
        } else if (arg == "-o" || arg == "--overwrite") {
            config.overwriteMode = true;
            config.appendMode = false;
        } else if (arg[0] == '-') {
            // Unknown option error handling
            cerr << "Warning: Unknown option '" << arg << "'" << endl;
            cerr << "Use '" << argv[0] << " --help' for usage information" << endl;
        } else {
            // Otherwise, filename and mode
            positionalArgs.push_back(arg);
        }
    }
    
    // Process filename and mode
    if (positionalArgs.size() > 0) {
        config.filename = positionalArgs[0];
    }
    if (positionalArgs.size() > 1) {
        config.mode = positionalArgs[1];
    }
    
    return config;
}

int main(int argc, char* argv[])
{
    // Parse command-line arguments
    Config config = parseArguments(argc, argv);
    
    // If help information is to be displayed
    if (config.showHelp) {
        displayHelp(argv[0]);
        return 0;
    }
    
    // If version information is to be displayed
    if (config.showVersion) {
        displayVersion();
        return 0;
    }
    
    // Empty filename error handling
    if (config.filename.empty()) {
        cerr << "Error: No filename specified" << endl;
        cerr << "Use '" << argv[0] << " --help' for usage information" << endl;
        return 1;
    }
    
    // Invalid mode error handling
    if (config.mode != "read" && config.mode != "write") {
        cerr << "Error: Invalid mode '" << config.mode << "'. Use 'read' or 'write'" << endl;
        return 1;
    }
    
    // Implementation of the two modes
    if (config.mode == "read") {
        ifstream file(config.filename);
        if (!file) {
            cerr << "Error: Cannot open file " << config.filename << endl;
            return 1;
        }
        
        string line;
        int lineNumber = 1;
        cout << "File content:" << endl;
        
        while (getline(file, line)) {
            if (config.showLineNumbers) { // Show line numbers
                cout << lineNumber << ": " << line << endl;
            } else {
                cout << line << endl;
            }
            lineNumber++;
        }
        file.close();
        
    } else if (config.mode == "write") {
        // Determine whether to use append mode or overwrite mode based on options
        ios_base::openmode openMode; // C++ standard library type
        if (config.overwriteMode) {
            openMode = ios::out;  // Overwrite mode from standard library
        } else {
            openMode = ios::app;  // Append mode from standard library (default)
        }
        
        ofstream file(config.filename, openMode);
        if (!file) { // File name error handling
            cerr << "Error: Cannot create/write to file " << config.filename << endl;
            return 1;
        }
        
        cout << "Enter content to write (empty line to finish):" << endl;
        string line;
        while (getline(cin, line) && !line.empty()) {
            file << line << endl;
        }
        file.close();
        
        // Display different success messages based on mode
        if (config.overwriteMode) {
            cout << "Content written to file (overwritten)" << endl;
        } else {
            cout << "Content appended to file" << endl;
        }
    }
    
    return 0;
}

```

Run the following command in the terminal to compile the program:

```terminal {fileName="terminal"}
make
```

Run the following command to view the help information:

```terminal {fileName="terminal"}
./output/main -h
```

The terminal output is as follows:

```terminal {fileName="terminal"}
Usage: ./output/main [OPTIONS] <filename> [mode]
A simple file read/write utility @ Aerosand

Arguments:
  filename          Name of the file to read/write
  mode              Operation mode: read or write (default: read)

Options:
  -h, --help        Display this help message
  -l                Show line numbers when reading
  -v, --version     Display version information
  -a, --append      Use append mode when writing (default)
  -o, --overwrite   Use overwrite mode when writing

Examples:
  ./output/main data.txt read
  ./output/main -l notes.txt
  ./output/main -o log.txt write
  ./output/main -a data.txt write
```

Run the following command to view the version information:

```terminal {fileName="terminal"}
./output/main --version
```

The terminal output is as follows:

```terminal {fileName="terminal"}
Version v1.0 @ Aerosand
This is a ofsp program from Aerosand
```

Run the following command to overwrite a file:

```terminal {fileName="terminal"}
./output/main data.txt write -o
Enter content to write (empty line to finish):
ofsp

Content written to file
```

Run the following command to read the file:

```terminal {fileName="terminal"}
./output/main -l data.txt
```

The terminal output is as follows:

```terminal {fileName="terminal"}
File content:
1: ofsp
```

Run the following command to append to a file:

```terminal {fileName="terminal"}
./output/main data.txt write -a
Enter content to write (empty line to finish):
This is a ofsp test from Aerosand.

Content appended to file
```

Run the following command to read the file:

```terminal {fileName="terminal"}
./output/main data.txt read -l
```

The terminal output is as follows:

```terminal {fileName="terminal"}
File content:
1: ofsp
2: This is a ofsp test from Aerosand.
```

It can be seen that this project can execute various command-line arguments and command-line options, such as displaying line numbers in the output and choosing how to write file content.

>[!tip]
>The above code does not consider architecture or optimization; it is intended to help readers simply understand the basic concepts of command-line arguments.

## 7. Summary

This project discussed the basic usage and implementation of command-line arguments in C++. I believe readers now have a better understanding of command-line arguments. Based on these discussions, the next section will transition to discussing command-line arguments in OpenFOAM.

This section has completed the following discussions:

- [x] Understanding the OpenFOAM command line
- [x] Understanding command-line arguments in C++
- [x] Understanding the functionality and options of command-line arguments
- [x] Compiling and running an arg project


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


