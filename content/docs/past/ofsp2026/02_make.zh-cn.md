---
uid: 20250723185036
title: 02_make
date: 2025-07-23
update: 2025-11-26
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
> 访问 https://aerosand.cn 以获取最近更新。


## 0. 前言

上一篇讨论的编译过程虽然清晰，但是一步一步的执行十分繁琐

为了简化项目编译的同时兼顾理解编译细节，我们采用 make 工具来管理我们的开发项目。很多项目也会使用 cmake 工具。cmake 更加简洁高效，不过本质上也是基于 makefile 的。

我们可以为项目提供 makefile 文件，在 makefile 中描述所有执行步骤。这样只需要简单执行 makefile 文件就可以编译整个项目，大大方便调试运行。此外，在之前的项目里，所有代码文件都放在一起，十分不方便，所以我们对代码进行架构管理。

本文主要讨论如下

- [ ] 理解 make 的使用
- [ ] 编写通用 makefile
- [ ] 编译运行 make 项目

## 1. 项目

终端输入命令，建立本文项目

```terminal {fileName="terminal"}
ofsp
mkdir ofsp_02_make
code ofsp_02_make
```

继续使用终端命令或者使用 vscode 界面创建其他文件，最终文件结构如下

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
>此时可以认为我们建立了一个库 Aerosand，其中包含一个同名的类 Aerosand

类 Aerosand 的**声明 Declaration** `Aerosand.h`，内容不变

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

类 Aerosand 的**定义 Definition** `Aerosand.cpp`，内容不变

```cpp {fileName="/Aerosand/Aerosand.cpp",linenos=table}
#include "Aerosand.h"

void Aerosand::setLocalTime(double t) {
    localTime_ = t;
}

double Aerosand::getLocalTime() const {
    return localTime_;
}
```

主源码 `ofsp_02_make.cpp` 需要修改头文件，其他内容不变

```cpp {fileName="/ofsp_02_make.cpp",linenos=table}
#include <iostream>

#include "Aerosand/Aerosand.h"   // 因为路径变化，需要修改此行

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

自动化构建工具 make 可以根据 makefile 中的规则自动完成源代码的编译、链接过程。

makefile 文件一般的基本格式为

```makefile {fileName="makefile"}
<target>:  <support>
	<command>
```

需要注意的是，当我们运行 `make` 命令的时候，它会尝试构建第一个目标以及它的依赖。所以原则上，makefile 除第一目标之外的其他目标顺序并不重要。如果 make 在构建的过程中发现某个目标缺少依赖，则会在 makefile 中寻求此依赖的构建。

当然，需要同时构建多个目标时，比较好的做法是使用 `all` 作为默认目标。

### 2.1. 库的 makefile

基于前文对 C++ 项目编译原理的过程的讨论，我们可以为库 Aerosand 提供 makefile 文件，直白的给出编译命令

注意，如果文件内容如下

```makefile {fileName="/Aerosand/makefile",linenos=table}
Aerosand.o: Aerosand.cpp
	g++ -c -fPIC Aerosand.cpp -o Aerosand.o

libAerosand.so: Aerosand.o
	g++ -shared -fPIC Aerosand.o -o libAerosand.so 
```

则第一目标错误，执行 `make` 后，仅 `Aerosand.o` 生成，而无法得到动态库。

调整脚本中命令的顺序，可以编译成功的写法为

```makefile {fileName="/Aerosand/makefile",linenos=table}
libAerosand.so: Aerosand.o
	g++ -shared -fPIC Aerosand.o -o libAerosand.so 

Aerosand.o: Aerosand.cpp
	g++ -c -fPIC Aerosand.cpp -o Aerosand.o
```

终端输入命令，成功生成动态库，

```terminal {fileName="terminal"}
cd Aerosand
make
```

当前文件夹的文件结构如下

```terminal {fileName="terminal"}
tree
.
├── Aerosand.cpp
├── Aerosand.h
├── Aerosand.o
├── libAerosand.so
└── makefile
```

可以看到动态库 `libAerosand.so` 已经成功生成。

### 2.2. 库编译的优化

如前讨论目标顺序问题，当构建目标繁多的时候，为了提高效率，便于维护，我们需要更好的组织 makefile 文件。优化 makefile 写法如下

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
> `.PHONY` 是 makefile 中的一个特殊声明，用于告诉 Make 某些目标**不是实际的文件名**，而是**伪目标**（Phony Targets）。这是 makefile 中非常重要且常用的功能，优点如下
> - 防止与真实文件冲突
> - 提高性能
> - 向其他阅读者明确表达意图
> 
> 使用 makefile 自动变量如下
> - `$^` 表示所有依赖项
> - `$@` 表示目标文件
> - `$<` 表示第一个依赖项

希望读者不要对 makefile 的写法感到担心，文件中大量采用了宏变量，可以让脚本更加具有通用性。陌生的语法大概知道可以这么使用即可，不需要花费更多的时间了解原因和原理。

这里给出必要的解释如下

- 第一段和第二段定义了一些宏变量，方便后续脚本的书写
- 第三段和第四段使用宏变量实现了编译命令，调换这两段的先后顺序并不影响编译结果
- 第五段明确了第一目标，定义了 `make all` 命令，可以实现完整编译
- 第六段定义了 `make clean` 命令，可以实现代码清理

终端输入命令，完成库的编译

```terminal {fileName="terminal"}
make clean
make all
```

与上一节的最终效果一样，动态库得到顺利编译生成。

### 2.3 项目的 makefile

有了前面讨论的经验，我们直接给出项目的 makefile 如下

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

相较于前一个 makefile 文件，这里多出了链接动态库的语句，其实也很是直白简单。

终端输入命令，完成代码编译

```terminal {fileName="terminal"}
cd ..
make clean
make all
make run
```

运行结果如下

```terminal {fileName="terminal"}
Hi, OpenFOAM! Here we are.
1 + 3.14159 = 4.14159
1 * 3.14159 = 3.14159

Current time step is : 0.2
```


## 3. vscode 插件

### 3.1. C/C++ Project Generator

对于一般的 C++ 项目，可以使用 vscode 的插件 `C/C++ Project Generator`，这是一个基于 makefile 的多文件项目模版。

操作和备注如下

{{% steps %}}

#### 新建项目

1. 使用 `F1` 打开快捷命令输入关键词，选择使用 `Create C++ project`
2. 输入项目名称
3. 在弹出窗口中选择新项目的父目录，并打开

#### 写代码

1. 在 `src/main.cpp` 中开发主函数代码
2. 在 `include/` 路径下开发头文件的声明
3. 在 `src/` 路径下开发头文件的定义

#### 编译运行

1. 终端使用命令 `make` 编译此项目，`make run` 编译并运行，`make clean` 清理项目
2. 终端使用命令 `./output/main` 直接运行该主程序

{{% /steps %}}

### 3.2. c cpp cmake project creator

这是一个基于 cmake 的多文件项目模板

操作和备注如下

{{% steps %}}

#### 新建项目

1. 使用 `F1` 打开快捷命令输入关键词，选择使用 `CMake Project: Create Project`
2. 在弹出窗口中选择到准备好的空白项目文件夹，并打开

#### 写代码

1. 在 `src/main.cpp` 中开发主函数代码
2. 在 `include/` 路径下开发头文件的声明
3. 在 `src/` 路径下开发头文件的定义

#### 编译运行

1. 终端使用命令 `cmake build/` 生成构建系统
2. 终端使用命令 `make -C build/` 进行项目编译
3. 终端使用命令 `./build/xxx` 运行该主程序

{{% /steps %}}


## 4. 小结

本文完成讨论

- [x] 理解 make 的使用
- [x] 编写通用 makefile
- [x] 编译运行 make 项目

## 支持我们

>[!tip]
>希望这里的分享可以对坚持、热爱又勇敢的您有所帮助。 
>
>如果这里的分享对您有帮助，您的评论、转发和赞助将对本系列以及后续其他系列的更新、勘误、迭代和完善都有很大的意义，这些行动也会为后来的新同学的学习有很大的助益。 
>
>赞助打赏时的信息和留言将用于展示和感谢。

{{< cards >}}
  {{< card link="/" title="支持我们" image="https://www.notion.so/image/attachment%3A3be6af9a-4829-4dfd-997e-641dfd055ba9%3Aalipay.jpg?table=block&id=22cd34b0-7c4c-8086-bdda-d558df1d9a11&t=22cd34b0-7c4c-8086-bdda-d558df1d9a11" subtitle="支付宝AliPay" >}}
{{< /cards >}}
