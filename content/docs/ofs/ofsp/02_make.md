---
uid: 20250723185036
title: 02_make
date: 2025-07-23
update: 2026-03-27
authors:
  - name: Aerosand
    link: https://github.com/aerosand
    image: https://github.com/aerosand.png
tags:
  - ofsp2026
  - OpenFOAM
  - ofsp
excludeSearch: false
toc: true
weight: 3
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

Although the compilation process discussed in the previous section is conceptually clear, executing it step by step is rather tedious and inefficient in practice.

In order to simplify project compilation while still preserving a clear understanding of the underlying compilation mechanisms, we introduce the `make` utility to manage the development workflow. Many modern projects also employ `cmake`, which offers a more concise and efficient interface; however, it is fundamentally still built upon `makefile`.

By providing a `makefile` for the project, all compilation steps can be explicitly described. Consequently, the entire project can be built through a single command, significantly improving both debugging efficiency and execution convenience. Furthermore, since all source files in previous examples were placed within a single
directory---resulting in poor organization---we now introduce a structured project architecture.

This section focuses on the following objectives:

- [ ] Understanding the usage of `make`
- [ ] Writing a general and reusable `makefile`
- [ ] Compiling and executing a project managed by `make`

## 1. Project

Execute the following commands in the terminal to create the project:

```terminal {fileName="terminal"}
ofsp
mkdir ofsp_02_make
code ofsp_02_make
```

Continue using either terminal commands or the VS Code interface to create the remaining files. The final project structure is as follows:

```terminal {fileName="terminal"}
tree
.
├── Aerosand
│   ├── Aerosand.cpp
│   ├── Aerosand.h
│   └── makefile
├── makefile
└── ofsp_02_make.cpp
```

>[!tip]
>At this stage, the directory `Aerosand` can be regarded as a library that contains a class of the same name.

The **declaration** of class `Aerosand` in `Aerosand.h` remains unchanged:

```cpp {fileName="/Aersoand/Aerosand.h",linenos=table}
#pragma once

class Aerosand
{
public:
    void setLocalTime(double t);
    double getLocalTime() const;


private:
    double localTime_;
};
```

The **definition** of class `Aerosand` in `Aerosand.cpp` remains unchanged:

```cpp {fileName="/Aerosand/Aerosand.cpp",linenos=table}
#include "Aerosand.h"

void Aerosand::setLocalTime(double t) {
    localTime_ = t;
}

double Aerosand::getLocalTime() const {
    return localTime_;
}
```

The main source file `ofsp_02_make.cpp` requires a modification to the header file path; the rest of the content remains unchanged:

```cpp {fileName="/ofsp_02_make.cpp",linenos=table}
#include <iostream>

#include "Aerosand/Aerosand.h"   // Modified due to path change

using namespace std;

int main()
{
    int a = 1;
    double pi = 3.1415926;

    cout << "Hi, OpenFOAM!" << " Here we are." << endl;
    cout << a << " + " << pi << " = " << a + pi << endl;
    cout << a << " * " << pi << " = " << a * pi << endl;


    Aerosand mySolver;
    mySolver.setLocalTime(0.2);
    cout << "\nCurrent time step is : " << mySolver.getLocalTime() << endl;

    return 0;
}
```

## 2. make

The automated build tool `make` can automatically perform the compilation and linking process based on the rules defined in a Makefile.

The basic format of a Makefile is as follows:

```makefile {fileName="makefile"}
<target>:  <support>
	<command>
```

It is important to note that when we run the `make` command, it attempts to build the first target and its dependencies. In principle, the order of targets other than the first does not matter. If `make` finds that a target lacks a dependency during the build process, it will look for a rule to build that dependency elsewhere in the Makefile.

When multiple targets need to be built simultaneously, it is good practice to use `all` as the default target.

### 2.1. Makefile for the Library

Based on the discussion of the C++ compilation process in the previous section, we can provide a Makefile for the `Aerosand` library, directly specifying the compilation commands.

Note that if the file content is as follows:

```makefile {fileName="/Aerosand/makefile",linenos=table}
Aerosand.o: Aerosand.cpp
	g++ -c -fPIC Aerosand.cpp -o Aerosand.o

libAerosand.so: Aerosand.o
	g++ -shared -fPIC Aerosand.o -o libAerosand.so 
```

The first target is incorrect; running `make` will only generate `Aerosand.o` and will not produce the dynamic library.

By adjusting the order of the rules, a successful compilation can be achieved:

```makefile {fileName="/Aerosand/makefile",linenos=table}
libAerosand.so: Aerosand.o
	g++ -shared -fPIC Aerosand.o -o libAerosand.so 

Aerosand.o: Aerosand.cpp
	g++ -c -fPIC Aerosand.cpp -o Aerosand.o
```

Run the following command in the terminal to successfully generate the dynamic library:

```terminal {fileName="terminal"}
cd Aerosand
make
```

The file structure of the current directory is as follows:

```terminal {fileName="terminal"}
tree
.
├── Aerosand.cpp
├── Aerosand.h
├── Aerosand.o
├── libAerosand.so
└── makefile
```

The dynamic library `libAerosand.so` has been successfully generated.

### 2.2. Optimizing the Library Makefile

As discussed regarding target order, when there are many build targets, it is necessary to better organize the Makefile to improve efficiency and maintainability. An optimized Makefile is as follows:

```makefile {fileName="/Aerosand/makefile",linenos=table}
# Compiler and flags
CXX         := g++
CXXFLAGS    := -std=c++17 -Wall -Wextra -g -O2
LDFLAGS     := -shared -fPIC

# Project information
NAME        := $(shell basename $(CURDIR))
SOURCES     := $(wildcard *.cpp)
OBJECTS     := $(SOURCES:.cpp=.o)
TARGET      := lib$(NAME).so

# Create shared library
$(TARGET): $(OBJECTS)
	$(CXX) $(LDFLAGS) $^ -o $@

# Compile source files to object files
%.o: %.cpp
	$(CXX) $(CXXFLAGS) -c $< -o $@

# Default target
.PHONY: all
all: $(TARGET)

# Clean up build artifacts
.PHONY: clean
clean:
	$(RM) $(TARGET) $(OBJECTS)
```

> [!tip]
> `.PHONY` is a special declaration in Makefiles used to tell `make` that a target is **not an actual file**, but rather a **phony target**. This is a very important and commonly used feature with the following advantages:
> - Prevents conflicts with real files
> - Improves performance
> - Clearly conveys intent to other readers
> 
> The following automatic variables are used in the Makefile:
> - `$^` represents all dependencies
> - `$@` represents the target file
> - `$<` represents the first dependency

Readers should not be overly concerned with the syntax of the Makefile. The extensive use of macros and variables makes the script more generic. For unfamiliar syntax, it is sufficient to know that it can be used in this way; there is no need to spend excessive time understanding the underlying reasons and principles.

A brief explanation is provided as follows:

- The first and second sections define macro variables to facilitate subsequent script writing.
- The third and fourth sections implement compilation commands using these macros; the order of these two sections does not affect the compilation result.
- The fifth section explicitly defines the first target, defining the `make all` command to perform a complete compilation.
- The sixth section defines the `make clean` command to clean up build artifacts.

Run the following command in the terminal to compile the library:

```terminal {fileName="terminal"}
make clean
make all
```

The final result is the same as in the previous section, and the dynamic library is successfully compiled and generated.

### 2.3. Makefile for the Project

Building on the previous discussion, we directly provide the Makefile for the project as follows:

```makefile {fileName="/makefile",linenos=table}
# Compiler and flags
CXX         := g++
CXXFLAGS    := -std=c++17 -Wall -Wextra -g -O2
LDFLAGS     := -shared -fPIC

# Project information
NAME        := $(shell basename $(CURDIR))
SOURCES     := $(wildcard *.cpp)
OBJECTS     := $(SOURCES:.cpp=.o)
OUTPUT      := $(NAME)

# Library paths
DLPATH      := -L./Aerosand -lAerosand

# Set library path for runtime
export LD_LIBRARY_PATH=./Aerosand

# Link executable
$(OUTPUT): $(OBJECTS)
	$(CXX) $(CXXFLAGS) $^ $(DLPATH) -o $@

# Compile source files to object files
%.o: %.cpp
	$(CXX) $(CXXFLAGS) -c $< -o $@

# Default target
.PHONY: all
all: $(OUTPUT)

# Run the program
.PHONY: run
run: $(OUTPUT)
	./$(OUTPUT)

# Clean up build artifacts
.PHONY: clean
clean:
	$(RM) $(OUTPUT) $(OBJECTS)
```

Compared to the previous Makefile, this one includes statements for linking the dynamic library, which is straightforward.

Run the following commands in the terminal to compile the code:

```terminal {fileName="terminal"}
cd ..
make clean
make all
make run
```

The output is as follows:

```terminal {fileName="terminal"}
Hi, OpenFOAM! Here we are.
1 + 3.14159 = 4.14159
1 * 3.14159 = 3.14159

Current time step is : 0.2
```


## 3. VS Code Extensions

### 3.1. C/C++ Project Generator

For general C++ projects, the VS Code extension `C/C++ Project Generator` can be used. It provides a multi-file project template based on Makefile.

Operations and notes are as follows:

{{% steps %}}

#### Create a New Project

1. Press `F1` to open the command palette, enter the keyword, and select `Create C++ project`.
2. Enter the project name.
3. In the pop-up window, select the parent directory for the new project, and open it.

#### Write Code

1. Develop the main function code in `src/main.cpp`.
2. Write header file declarations in the `include/` directory.
3. Write header file definitions in the `src/` directory.

#### Compile and Run

1. Use the command `make` in the terminal to compile the project; `make run` to compile and run; `make clean` to clean the project.
2. Use the command `./output/main` in the terminal to run the main program directly.

{{% /steps %}}

### 3.2. C Cpp CMake Project Creator

This is a multi-file project template based on CMake.

Operations and notes are as follows:

{{% steps %}}

#### Create a New Project

1. Press `F1` to open the command palette, enter the keyword, and select `CMake Project: Create Project`.
2. In the pop-up window, navigate to the prepared empty project folder and open it.

#### Write Code

1. Develop the main function code in `src/main.cpp`.
2. Write header file declarations in the `include/` directory.    
3. Write header file definitions in the `src/` directory.

#### Compile and Run

1. Use the command `cmake build/` in the terminal to generate the build system.
2. Use the command `make -C build/` in the terminal to compile the project.
3. Use the command `./build/xxx` in the terminal to run the main program.

{{% /steps %}}


## 4. Summary

This section has completed the following discussions:

- [x] Understanding the usage of `make`
- [x] Writing a general and reusable `makefile`
- [x] Compiling and executing a project managed by `make`


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

