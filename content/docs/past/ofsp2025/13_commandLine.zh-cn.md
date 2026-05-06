---
uid: 20251104124420
title: 13_commandLine
date: 2025-11-04
update: 2026-02-04
authors:
  - name: Aerosand
    link: https://github.com/aerosand
    image: https://github.com/aerosand.png
tags:
  - ofsp
  - ofsp2025
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
> 访问 https://aerosand.cn 以获取最近更新。


## 0. 前言

新同学很难不注意到求解器总是有几个固定的头文件，包括 `setRootCase.H` ，`createTime.H` 和 `createMesh.H` 。网络上找到的代码解析常常只有一句注释介绍功能，也许无法消除困惑感。该系列将分别讨论。

前文讨论了 C++ 中的命令行参数，本文将过渡到对 OpenFOAM 中命令行参数的讨论。在此之前，我们先看一下 `setRootCase.H` 文件。

本文主要讨论

- [ ] 理解 setRootCase.H 头文件
- [ ] 了解 argList 类
- [ ] 开发用户自定义命令行参数
- [ ] 编译运行 commandLine 项目

## 1. 项目准备

终端输入命令，建立项目

```terminal {fileName="terminal"}
ofsp
foamNewApp ofsp_13_commandLine
cd ofsp_13_commandLine
cp -r $FOAM_TUTORIALS/incompressible/icoFoam/cavity/cavity debug_case
code .
```

终端输入命令，测试初始求解器

```terminal {fileName="terminal"}
wmake
ofsp_13_commandLine -case debug_case
```

测试初始求解器没有问题，可以进一步开发。此步骤以后不再赘述。

终端输入命令，新建脚本和说明，并给脚本权限

```terminal {fileName="terminal"}
code caserun caseclean READEME.md
chmod +x caserun caseclean
```

脚本和说明参考前文讨论。除非有特别情况，不再赘述脚本和说明。

终端输入命令，查看文件结构如下

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

除非有特别情况，不再赘述简单的文件结构。

## 2. setRootCase.H

API 页面 https://api.openfoam.com/2506/setRootCase_8H.html

我们可以通过终端查找该文件

```terminal {fileName="terminal"}
find $FOAM_SRC -iname setRootCase.H
```

使用 vscode 可以直接点击终端输出的文件路径打开该文件。

在 vscode 中可以通过 OFextension 直接跳转打开该文件。

为了方便理解，我们查阅一下 OpenFOAM 2.0x 版本的代码，

Github 仓库文件链接如下 https://github.com/OpenFOAM/OpenFOAM-2.0.x/blob/master/src/OpenFOAM/include/setRootCase.H

代码如下

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

现代版本的代码具体如下

```cpp {fileName="setRootCase.H",linenos=table,linenostart=1}
// Construct from (int argc, char* argv[]),
// - use argList::argsMandatory() to decide on checking command arguments.
// - check validity of the options

Foam::argList args(argc, argv); // 将argList类实例化args对象
if (!args.checkRootCase()) 
{
    Foam::FatalError.exit();
}
// 基于算例根目录检查根目录和算例目录
// 1. 根目录是否存在
// 2. 算例目录是否存在
// 否则报错退出

// 基于命令行参数argc,argv构造对象args
// 所以此H文件需要放在所有和命令行参数相关的代码之后

// User can also perform checks on otherwise optional arguments.
// Eg,
//
//  if (!args.check(true, false))
//  {
//      Foam::FatalError.exit();
//  }
// 用户可以按照此模板添加额外的参数验证，暂不深究

// Force dlOpen of FOAM_DLOPEN_LIBS (principally for Windows applications)
#include "foamDlOpenLibs.H"
// 兼容相关，暂不深究
```

## 3. argList类

类 `Foam::argList`是 OpenFOAM 中的基础类，具有很多的成员数据和成员方法。我们大概挑几处代码作为切入点简单了解一下 `argList` 类。

API 页面 https://api.openfoam.com/2506/argList_8H.html

我们可以通过终端查找该文件

```terminal {fileName="terminal"}
find $FOAM_SRC -iname argList.H
```

打开该类的声明 `argList.H`，内容如下

```cpp {fileName="argList.H",linenos=table,linenostart=1}
...
class argList
{
	...
public:
	...
	// 基于命令行参数的构造函数
	argList
	(
		int& argc, // 主函数参数个数
		char**& argv, // 主函数参数的指针
		bool checkArgs = argList::argsMandatory(), // 是否必须检查参数
		bool checkOpts = true,
		bool initialise = true
	);
	...
	//- Check root path and case path // 检查根目录路径和算例路径
        bool checkRootCase() const;
    ...
...
```

关于成员函数 `checkRootCase()`的实现，需要去看同目录下的类的定义，即 `argList.C` 文件，摘取部分内容如下

```cpp {fileName="argList.C",linenos=table,linenostart=1}
...
bool Foam::argList::checkRootCase() const // 函数返回布尔类型
{
    if (!fileHandler().isDir(rootPath())) // 检查算例根目录是否存在
    {
        FatalError
            << executable_
            << ": cannot open root directory " << rootPath()
            << endl;

        return false;
    }
    // 如果算例的根目录（算例的父目录）不存在，则触发此报错
    // 如 simpleFoam -case wrongRoot/cavity

	// 检查算例目录是否错误
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
    // 如果算例目录不存在，则触发此报错
    // 如 simpleFoam -case rootPath/wrongCase

    return true;
}
...
```

讨论到此，读者应该了解并明白`setRootCase.H`到底是检查什么目录了。

>[!tip]
>对于现阶段来说，读者不宜深挖代码。只要做到心里有数，可以理解且不陌生排斥即可。
>
>更深入的代码讨论参见 `ofsc(openfoam sharing coding)` 系列。

## 4. 帮助信息

我们可以使用命令行参数实现一些功能。

在主函数上修改

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

终端输入命令，编译并查看帮助信息

```terminal {fileName="terminal"}
wmake
ofsp_13_commandLine -help
```

终端输出如下

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

可以看到其中有自定义的帮助信息

## 5. 强制参数

强制参数在项目运行时必须输入，否则会有错误提醒。

在主函数上修改

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
    // 将应用参数增补到主函数参数列表里
    // 在运行程序时，这两个参数是需要强制给定的

    #include "setRootCase.H"
    #include "createTime.H"

	// args的第0参数是程序名本身
    const word args1 = args[1]; // args第1参数就是增补的第1个参数
    const scalar args2 = args.get<scalar>(2); // 增补的第2个参数


	// 显示参数
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

重新编译此项目。

终端输入命令，运行该项目

```terminal {fileName="terminal"}
ofsp_13_commandLine -case debug_case/ o2 2
```

终端输出如下

```terminal {fileName="terminal"}
Create time

Solver setup: 
    Accuracy     : o2
    Acceleration : 2



ExecutionTime = 0 s  ClockTime = 0 s

End
```

可以想到，当我们拿到这些命令行参数，可以用来参与到项目的计算中。

## 6. 应用选项

应用选项在项目运行时可写可不写。

在主函数上修改

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
    // 将应用参数增补到主函数参数列表里
    // 在运行程序时，这两个参数是需要强制给定的

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

	// args的第0参数是程序名本身
    const word args1 = args[1]; // args第1参数就是增补的第1个参数
    const scalar args2 = args.get<scalar>(2); // 增补的第2个参数


	// 显示参数
    Info<< "Solver setup: " << nl
        << "    Accuracy     : " << args1 << nl
        << "    Acceleration : " << args2 << nl
        << nl << endl;


    // 确定字典文件路径
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

	// 自定义字典对象
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

    fileName dict_("./system/myDict"); // 默认字典路径
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

重新编译此项目。

为该项目提供自定义字典。不使用代码中设置的字典默认路径，创建字典 `/debug_case/constant/myDic`

自定义字典如下

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

终端输入命令，运行该项目

```terminal {fileName="terminal"}
ofsp_13_commandLine -case debug_case/ o2 2 -dict ./debug_case/constant/myDict -nPrecision 16 -debug
```

终端输出如下

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


## 7. 小结

本文一步一步讨论了 OpenFOAM 中命令行参数可能有的功能。虽然本文项目只是简单演示，相信读者可以在此讨论基础之上，理解 OpenFOAM 中的命令行参数，并为以后的自开发提供一些想法。

本文主要讨论

- [x] 理解 setRootCase.H 头文件
- [x] 了解 argList 类
- [x] 开发用户自定义命令行参数
- [x] 编译运行 commandLine 项目

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
