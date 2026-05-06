---
uid: 20250826161556
title: 03_wmake
date: 2025-08-26
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
weight: 4
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

在理解了 C++ 项目的 make 实现方式之后，我们现在可以更加容易的明白 OpenFOAM 提供的 wmake 实现方式。

本文主要

- [ ] 简单理解 wmake
- [ ] wmake 实现直接链接
- [ ] wmake 实现动态库链接
- [ ] 理解 Make 文件
- [ ] 编译运行 wmake 项目

## 1. 理解 wmake

OpenFOAM 中的 wmake 是一个基于 make 的构建工具，本质上是专门为 OpenFOAM 项目设计的一组脚本和配置文件。它简化了 OpenFOAM 的编译过程，并自动处理一些特定的依赖和路径设置。

 1. wmake 是一个包装了 make 的脚本，它设置了 OpenFOAM 编译所需的环境变量和规则
 2. wmake 会自动查找 OpenFOAM 的特定目录结构，并设置相应的编译选项（如包含路径、库路径等）
 3. wmake 提供了与 make 相似的接口，但针对 OpenFOAM 进行了优化

OpenFOAM 使用 `Make/files` 和 `Make/options` 实现 `wmake` 的编译管理。

> [!tip]
> 可以简单理解为，Make 文件夹下的 files 和 options 两个文件一起承担了 makefile 的功能。

OpenFOAM 约定源文件后缀为 `.C`，头文件后缀为 `.H` 。

为了让读者不要对文件架构感到困扰，这里作了不严谨的区分。OpenFOAM “**项目层面**” 的 `.H` 文件更多只是为了对主源码按功能进行拆分，方便代码阅读和维护，很多并不是类的头文件。这和 C++ 开发层面的头文件以及 OpenFOAM “**源码层面**”的头文件（如 `$FOAM_SRC/OpenFOAM/dimensionSet/dimensionSet.H`）有些不同。

> [!tip]
> OpenFOAM 中，应用（application）包括
> - 求解器（Solvers）
> - 工具程序（Utilities）
> - 预处理程序（Pre-processing Applications）
> - 后处理程序（Post-processing Applications）
> 
> 在本系列讨论中，我们所说的 “项目 Project” 是完整解决某个问题的所有内容，包括
> - 求解器
> - 调试算例
> - 调用库
> - 工具程序
> - 等等

本文下面讨论的是简单的“源码层面”头文件。

## 2. 项目

终端输入命令，建立本文项目

```terminal {fileName="terminal"}
ofsp
mkdir ofsp_03_wmake
code ofsp_03_wmake
```

继续使用终端命令或者使用 vscode 界面创建其他文件，最终文件结构如下

```terminal {fileName="terminal"}
tree
.
├── Aerosand
│   ├── Aerosand.C
│   └── Aerosand.H
├── Make
│   ├── files
│   └── options
└── ofsp_03_wmake.C
```

库 Aerosand 的声明 `Aerosand.H` 如下

```cpp {fileName="/Aerosand/Aerosand.H",linenos=table}
#pragma once

class Aerosand
{
public:
    void SetLocalTime(double t);
    double GetLocalTime() const;


private:
    double localTime_;
};

```

> [!warning]
> 注意每个代码文件的最后都需要留个一个空白行，否则编译时 OpenFOAM 会警告 `parse error`

库 Aerosand 的定义 `Aerosand.C` 如下

```cpp {fileName="/Aerosand/Aerosand.C",linenos=table}
#include "Aerosand.H"

void Aerosand::SetLocalTime(double t) {
    localTime_ = t;
}

double Aerosand::GetLocalTime() const {
    return localTime_;
}

```

应用主源码 `ofsp_03_wmake.C` 如下

```cpp {fileName="/ofsp_03_wmake.C",linenos=table}
#include <iostream>

#include "Aerosand.H"

using namespace std;

int main()
{
    int a = 1;
    double pi = 3.1415926;

    cout << "Hi, OpenFOAM!" << " Here we are." << endl;
    cout << a << " + " << pi << " = " << a + pi << endl;
    cout << a << " * " << pi << " = " << a * pi << endl;


    Aerosand mySolver;
    mySolver.SetLocalTime(0.2);
    cout << "\nCurrent time step is : " << mySolver.GetLocalTime() << endl;

    return 0;
}

```


OpenFOAM 提供了 Make 文件来帮助开发，其中

- `/Make/files` 文件用于指定要编译的源文件以及生成的目标文件的名称和位置
- `/Make/options` 文件用于指定编译和链接选项，包括头文件路径和需要链接的库

注意，`#include "Aerosand.H"` 无需指定路径，这是因为我们将在 Make 文件中处理路径问题。

下面将讨论不同连接方式以及 Make 文件的细节。

## 3. 直接链接

类似于 `02_helloWorld/3.4.链接` 文中讨论的直接链接，OpenFOAM 中也可以实现。

### 3.1. 配置 Make

文件 `ofsp_03_wmake/Make/files` 内容如下

```wmake {fileName="/Make/files" linenos=table}
Aerosand/Aerosand.C
ofsp_03_wmake.C

EXE = $(FOAM_USER_APPBIN)/ofsp_00_helloWorld_wmake
```

- 源文件为 `Aerosand.C` 和 `ofsp_03_wmake.C`
- 生成目标文件的位置为 `$(FOAM_USER_APPBIN)`
- 生成目标文件的名称为 `ofsp_03_wmake`

文件 `ofsp_03_wmake/Make/options` 内容如下

```wmake {fileName="/Make/options" linenos=table}
EXE_INC = \
	-IAerosand

EXE_LIBS = \
```

- 前缀 EXE 表示这是可执行文件的配置（前缀 LIB 表示是库文件的配置）
- 后缀 INC 表示包含路径
	- 使用 `-I` 标志指定头文件搜索路径
	- 通常指向 OpenFOAM 模块的 `lnInclude` 目录（链接包含目录）
	- 可以包含系统头文件路径或第三方库头文件路径
- 后缀 LIBS 表示库链接（直接链接无需链接任何库，可以留空不写）
	- 使用 `-l` 标志指定需要链接的库
	- 库名称不需要前缀 `lib` 和后缀 `.so`（如 `-lfiniteVolume` 对应 `libfiniteVolume.so`）
	- 使用 `-L` 标志指定库搜索路径

终端输入命令，执行编译

```terminal {fileName="terminal"}
wclean
wmake
```

### 3.2. 编译

终端输出信息有三段，对应着应用编译的三个过程。

第一段是自定义的 Aerosand 类编译得到目标文件 `Aerosand.o` （见输出信息的末尾处）

```terminal {fileName="terminal"}
g++ -std=c++14 -m64 -pthread -DOPENFOAM=2406 -DWM_DP -DWM_LABEL_SIZE=32 -Wall 
-Wextra -Wold-style-cast -Wnon-virtual-dtor -Wno-unused-parameter -Wno-invalid
-offsetof -Wno-attributes -Wno-unknown-pragmas -O3  -DNoRepository -ftemplate
-depth-100  -IAerosand -iquote. -IlnInclude 
-I/usr/lib/openfoam/openfoam2406/src/OpenFOAM/lnInclude 
-I/usr/lib/openfoam/openfoam2406/src/OSspecific/POSIX/lnInclude   -fPIC 
-c Aerosand/Aerosand.C -o Make/linux64GccDPInt32Opt/Aerosand/Aerosand.o
```

第二段是主源码编译得到目标文件 `ofsp_03_wmake.o` （见输出信息的末尾处）

```terminal {fileName="terminal"}
g++ -std=c++14 -m64 -pthread -DOPENFOAM=2406 -DWM_DP -DWM_LABEL_SIZE=32 -Wall 
-Wextra -Wold-style-cast -Wnon-virtual-dtor -Wno-unused-parameter -Wno-invalid
-offsetof -Wno-attributes -Wno-unknown-pragmas -O3  -DNoRepository -ftemplate
-depth-100  -IAerosand -iquote. -IlnInclude 
-I/usr/lib/openfoam/openfoam2406/src/OpenFOAM/lnInclude 
-I/usr/lib/openfoam/openfoam2406/src/OSspecific/POSIX/lnInclude   -fPIC 
-c ofsp_03_wmake.C -o Make/linux64GccDPInt32Opt/ofsp_03_wmake.o
```

第三段是链接的过程，此处就是上述两个目标文件的直接链接，最终生成可执行文件（见输出信息的末尾处）

```terminal {fileName="terminal"}
g++ -std=c++14 -m64 -pthread -DOPENFOAM=2406 -DWM_DP -DWM_LABEL_SIZE=32 -Wall 
-Wextra -Wold-style-cast -Wnon-virtual-dtor -Wno-unused-parameter -Wno-invalid
-offsetof -Wno-attributes -Wno-unknown-pragmas -O3  -DNoRepository -ftemplate
-depth-100  -IAerosand -iquote. -IlnInclude 
-I/usr/lib/openfoam/openfoam2406/src/OpenFOAM/lnInclude 
-I/usr/lib/openfoam/openfoam2406/src/OSspecific/POSIX/lnInclude   
-fPIC -Xlinker --add-needed -Xlinker --no-as-needed
Make/linux64GccDPInt32Opt/Aerosand/Aerosand.o 
Make/linux64GccDPInt32Opt/ofsp_03_wmake.o 
-L/usr/lib/openfoam/openfoam2406/platforms/linux64GccDPInt32Opt/lib \
     -lOpenFOAM -ldl  \
     -lm -o /home/aerosand/OpenFOAM/aerosand
 -v2406/platforms/linux64GccDPInt32Opt/bin/ofsp_03_wmake
```

编译的过程文件在 `ofsp_03_wmake/Make/linux64GccDPInt32Opt/` 文件夹下（根据平台可能会有所不同）。编译形成的可执行程序在 `$FOAM_USER_APPBIN` 文件夹下（上文在 `Make/files` 中指定）。

终端输入命令，可以找到所有生成成功的可执行程序

```terminal {fileName="terminal"}
tree $FOAM_USER_APPBIN
```

### 3.3. 运行

这个可执行程序是个独立的程序，不需要从任何外部文件读取参数。得益于 OpenFOAM 环境变量，我们可以在任何路径下，都可以通过终端命令直接运行编译成功的程序。

终端输入命令

```terminal {fileName="terminal"}
ofsp_03_wmake
```

可以看到运行结果

```terminal {fileName="terminal"}
Hi, OpenFOAM! Here we are.
1 + 3.14159 = 4.14159
1 * 3.14159 = 3.14159

Current time step is : 0.2
```

## 4. 动态库链接

在实际的开发中，我们还是要使用动态库来保证内存和效率。

调整文件结构如下

```terminal {fileName="terminal"}
.
├── Aerosand
│   ├── Aerosand.C
│   ├── Aerosand.H
│   └── Make
│       ├── files
│       └── options
├── Make
│   ├── files
│   └── options
└── ofsp_03_wmake.C
```

### 4.1. 库 Make

为库创建 Make 文件

文件 `/Aerosand/Make/files` 内容如下

```wmake {fileName="/Aerosand/Make/files"}
Aerosand.C

LIB = $(FOAM_USER_LIBBIN)/libAerosand
```

- 注意库使用 `LIB` 而不是 `EXE`
- 库的目标路径结尾是 `LIBBIN` 而不是 `APPBIN`
- 目标文件需要加 `lib`

因为这个 Aerosand 库的实现不再进一步需要链接其他库，所以 `/Aerosand/Make/options` 直接空置即可。

终端输入命令，在根目录下编译这个库

```terminal {fileName="terminal"}
wmake Aerosand
```

终端输出信息有两段，对应着编译的过程。

第一段是自定义的 Aerosand 库编译得到目标文件 `Aerosand.o` （见输出信息的末尾处）

```terminal {fileName="terminal"}
g++ -std=c++14 -m64 -pthread -DOPENFOAM=2406 -DWM_DP -DWM_LABEL_SIZE=32 -Wall 
-Wextra -Wold-style-cast -Wnon-virtual-dtor -Wno-unused-parameter -Wno-invalid
-offsetof -Wno-attributes -Wno-unknown-pragmas -O3  -DNoRepository -ftemplate
-depth-100   -iquote. -IlnInclude 
-I/usr/lib/openfoam/openfoam2406/src/OpenFOAM/lnInclude 
-I/usr/lib/openfoam/openfoam2406/src/OSspecific/POSIX/lnInclude   -fPIC 
-c Aerosand.C -o Make/linux64GccDPInt32Opt/Aerosand.o
```

第二段将目标文件 `Aerosand.o` 编译成动态库 `Aerosand.so` （见输出信息的末尾处）

```terminal {fileName="terminal"}
g++ -std=c++14 -m64 -pthread -DOPENFOAM=2406 -DWM_DP -DWM_LABEL_SIZE=32 -Wall 
-Wextra -Wold-style-cast -Wnon-virtual-dtor -Wno-unused-parameter -Wno-invalid
-offsetof -Wno-attributes -Wno-unknown-pragmas -O3  -DNoRepository -ftemplate
-depth-100   -iquote. -IlnInclude 
-I/usr/lib/openfoam/openfoam2406/src/OpenFOAM/lnInclude 
-I/usr/lib/openfoam/openfoam2406/src/OSspecific/POSIX/lnInclude   -fPIC -shared 
-Xlinker --add-needed -Xlinker 
--no-as-needed  Make/linux64GccDPInt32Opt/Aerosand.o 
-L/usr/lib/openfoam/openfoam2406/platforms/linux64GccDPInt32Opt/lib \
      -o /home/aerosand/OpenFOAM/aerosand
-v2406/platforms/linux64GccDPInt32Opt/lib/libAerosand.so
```

编译的过程文件在 `ofsp_03_wmake/Aerosand/Make/linux64GccDPInt32Opt/` 文件夹下（根据平台可能会有所不同）。编译形成的可执行程序在 `$FOAM_USER_LIBBIN` 文件夹下（上文在 `Make/files` 中指定）。

终端输入命令，可以找到所有生成成功的动态库

```terminal {fileName="terminal"}
tree $FOAM_USER_LIBBIN
```

库内的类可能会有多个，甚至还有其他子库。库编译后会同时在库的路径下生成 `lnInclude` 文件夹，`lnInclude` 包含了该库所有类（/子库）的声明（ `.H` 文件）或者实现 `.C文件` 的软链接目录（暂时可以简单理解成快捷方式），方便后续链接的时候可以提供统一简单路径。可以参考 OpenFOAM 的 `$FOAM_SRC/OpenFOAM` 库，可以看到根目录下有 `lnInclude` 文件夹，其中包含了此库内的所有类（/子库）的快捷方式。

终端输入命令，查看 Aerosand 库编译后的文件结构

```terminal {fileName="terminal"}
Aerosand
├── Aerosand.C
├── Aerosand.H
├── lnInclude
│   ├── Aerosand.C -> ../Aerosand.C
│   └── Aerosand.H -> ../Aerosand.H
└── Make
    ├── files
    ├── linux64GccDPInt32Opt
    │   ├── Aerosand.C.dep
    │   ├── Aerosand.o
    │   ├── options
    │   ├── sourceFiles
    │   └── variables
    └── options
```

### 4.2. 项目 Make

因为 Aerosand 库已经被编译成了动态库，所以我们需要在项目的 Make 文件中进行指定。

修改 `/Make/files` 文件

```wmake {fileName="/Make/files"}
ofsp_03_wmake.C

EXE = $(FOAM_USER_APPBIN)/ofsp_03_wmake
```

修改 `/Make/options` 文件

```wmake {fileName="/Make/options"}
EXE_INC = \
    -IAerosand/lnInclude

EXE_LIBS = \
    -L$(FOAM_USER_LIBBIN) \
    -lAerosand

```

- 前缀 EXE 表示这是可执行文件的配置（前缀 LIB 表示是库文件的配置）
- 后缀 INC 表示包含路径
	- 使用 `-I` 标志指定头文件搜索路径
	- 通常指向 OpenFOAM 模块的 `lnInclude` 目录（链接包含目录）
	- 可以包含系统头文件路径或第三方库头文件路径
- 后缀 LIBS 表示库链接（直接链接无需链接任何库，可以留空不写）
	- 使用 `-l` 标志指定需要链接的库
	- 库名称不需要前缀 `lib` 和后缀 `.so`（如 `-lAerosand` 对应 `libAerosand.so`）
	- 使用 `-L` 标志指定库搜索路径
- 注意最后空置一行，避免 OpenFOAM 编译警告


> [!tip]
> 我们常见的 OpenFOAM 求解器的 `Make/options` 中没有 `-L` 指定只有 `-l` 指定，这是因为那些求解器中使用的都是原生库，链接路径已经得到配置，无需进行 `-L` 路径指定，仅进行 `-l` 名称指定即可。如果是用户自定义库，编译结果在其他文件位置，当然就需要进行 `-L` 路径指定。
> 
一定要注意路径，路径对于成功编译非常重要。一般指定的路径都是从项目根目录出发的（可以从根目录出发，定义其他库的绝对路径，见下文演示）。

### 4.3. 编译

终端输入命令，执行编译过程

```terminal {fileName="terminal"}
wclean
wmake
```

终端输出信息有两段，对应着编译的过程。

第一段是主源码编译得到目标文件 `ofsp_03_wmake.o` （见输出信息的末尾处）

```terminal {fileName="terminal"}
g++ -std=c++14 -m64 -pthread -DOPENFOAM=2406 -DWM_DP -DWM_LABEL_SIZE=32 -Wall 
-Wextra -Wold-style-cast -Wnon-virtual-dtor -Wno-unused-parameter -Wno-invalid
-offsetof -Wno-attributes -Wno-unknown-pragmas -O3  -DNoRepository -ftemplate
-depth-100  -IAerosand/lnInclude -iquote. -IlnInclude 
-I/usr/lib/openfoam/openfoam2406/src/OpenFOAM/lnInclude 
-I/usr/lib/openfoam/openfoam2406/src/OSspecific/POSIX/lnInclude   -fPIC 
-c ofsp_03_wmake.C -o Make/linux64GccDPInt32Opt/ofsp_03_wmake.o
```

第二段链接动态库到应用，并生成可执行文件 `ofsp_03_wmake` （见输出信息的末尾处）

```terminal {fileName="terminal"}
g++ -std=c++14 -m64 -pthread -DOPENFOAM=2406 -DWM_DP -DWM_LABEL_SIZE=32 -Wall 
-Wextra -Wold-style-cast -Wnon-virtual-dtor -Wno-unused-parameter -Wno-invalid
-offsetof -Wno-attributes -Wno-unknown-pragmas -O3  -DNoRepository -ftemplate
-depth-100  -IAerosand/lnInclude -iquote. -IlnInclude 
-I/usr/lib/openfoam/openfoam2406/src/OpenFOAM/lnInclude 
-I/usr/lib/openfoam/openfoam2406/src/OSspecific/POSIX/lnInclude   -fPIC -Xlinker
--add-needed -Xlinker --no-as-needed  Make/linux64GccDPInt32Opt/ofsp_03_wmake.o 
-L/usr/lib/openfoam/openfoam2406/platforms/linux64GccDPInt32Opt/lib \
    -L/home/aerosand/OpenFOAM/aerosand-v2406/platforms/linux64GccDPInt32Opt/lib 
    -lAerosand -lOpenFOAM -ldl  \
     -lm -o /home/aerosand/OpenFOAM/aerosand
-v2406/platforms/linux64GccDPInt32Opt/bin/ofsp_03_wmake
```

同样，编译形成的可执行程序在 `$FOAM_USER_APPBIN` 文件夹下（上文在 `Make/files` 中指定）。

### 4.4. 运行

终端输入命令

```terminal {fileName="terminal"}
ofsp_03_wmake
```

可以看到运行结果

```terminal {fileName="terminal"}
Hi, OpenFOAM! Here we are.
1 + 3.14159 = 4.14159
1 * 3.14159 = 3.14159

Current time step is : 0.2
```

## 5. 小结

本文完成讨论

- [x] 简单理解 wmake
- [x] wmake 实现直接链接
- [x] wmake 编译动态库
- [x] 理解 Make 文件
- [x] 编译运行 wmake 项目

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
